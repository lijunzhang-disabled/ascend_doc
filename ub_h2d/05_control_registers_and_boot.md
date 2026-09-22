# 05 — Kernel control, register access, and boot over UB

[Summary](README.md) · Previous: [queue and HDC](04_queue_and_hdc.md) · Next: [ACL/runtime dispatch](06_acl_runtime_dispatch.md)

Source snapshot and evidence boundaries: [series scope](README.md#scope-and-evidence).
Expanded on 2026-09-20 against the same `driver` commit, `866e409`. This chapter
covers P7–P10: remote events and kernel messages, register/shared-status access,
and boot delivery. The source trace ends at UBCORE, UMMU, or the firmware
interface where their implementations are outside this checkout.

## 1. Requests, register transfers, and boot have different completion models

Kernel control traffic includes both request/reply SENDs and direct READ/WRITE
operations. Boot delivery exposes a host buffer for firmware to read and uses
a separate handoff protocol. The following boundaries determine when the host
can use a response or refill a buffer.

| Path | Host action | Successful completion visible here |
|---|---|---|
| Common or dedicated kernel message | Copy a request into a registered descriptor and SEND it | SEND completion, matching protocol reply, reply status and copy checks |
| UB admin request | SEND a resource-management opcode and its parameters | SEND completion and matching admin reply |
| Kernel `devdrv_urma_copy` | Post WRITE or READ between supplied segments | Transport completion for that operation |
| RAO read/write | Post READ or WRITE using the client's registered ranges | Transport completion for that operation |
| BIOS mailbox loader | Fill the host window and publish address/TID/size and ready flags | Firmware flag handshake allows reuse; all-files success advances boot status |
| UBIOS/UVB callback loader | Return the host window's address/TID and refill on callbacks | Loading-state callbacks coordinate reuse; successful final-state callback frees the window |

Some clients check a further service-specific result inside the reply. Even a
successful service reply does not universally mean that a submitted event has
finished executing or that a device application has consumed its payload.
An ioctl or mapping operation can also finish entirely on the host; follow
its forwarding branch before counting H2D traffic.

## 2. Common and dedicated kernel message channels

The PBL message-channel APIs dispatch through the selected communication
operations. For UB, `g_ubdrv_ops` installs the UB implementations of common
messages, dedicated synchronous messages, RAO, URMA copy, and segment handling.
See [UB operation table](../../driver/src/sdk_driver/comm/ub/host/ascend_ub_main_adapt.c#L740),
[common-message wrapper](../../driver/src/sdk_driver/pbl/msg_chan/host/msg_chan_common_msg.c#L74),
and [synchronous-message dispatch](../../driver/src/sdk_driver/pbl/msg_chan/common/msg_chan_msg_comm.c#L16).

```text
Kernel module request
  -> devdrv_common_msg_send(device, ..., client_type)
     -> devdrv_ub_common_msg_send
        -> select common channel and encode client type
  or
  -> devdrv_sync_msg_send(channel, ...)
     -> devdrv_ub_sync_msg_send

Both UB routes
  -> devdrv_sync_msg_send_inner
     -> prepare registered send descriptor and request data
     -> ubdrv_post_send_wr
        -> ubcore_post_jfs_wr with SEND
     -> wait/check send completion
     -> wait for matching protocol reply
     -> copy reply, process status, repost receive buffer
```

Common messages multiplex client types on a common channel. Dedicated channels
give modules their own channel configuration. Both converge on the same
synchronous transaction machinery in the UB driver.
See [common-message adaptation](../../driver/src/sdk_driver/comm/ub/host/msg/ascend_ub_msg_adapt.c#L115),
[dedicated-message adaptation](../../driver/src/sdk_driver/comm/ub/host/msg/ascend_ub_non_trans_chan_adapt.c#L306),
and [SEND construction](../../driver/src/sdk_driver/comm/ub/common/ub_res/ascend_ub_jetty.c#L674).

“Non-trans” is a channel-family name in this code. These functions do transfer
request and reply bytes over UB; the name does not mean that their data stays
on the host. Similarly, the common-channel allocator's
`devdrv_pcimsg_alloc_non_trans_queue` name does not establish a PCIe payload
path when the selected operations are the UB table.

### Channel setup precedes individual sends

The host allocator reserves an idle channel, installs the client's receive
callback, and prepares its descriptor sizes and queue depths. The shared
allocation procedure then:

1. Creates the local communication resources.
2. Sends `UBDRV_CREATE_MSG_QUEUE` on the admin channel, exchanging resource
   information with the peer.
3. Initializes the local channel from the returned peer information.
4. Rearms completion notification.
5. Sends `UBDRV_ENABLE_MSG_QUEUE` for a normal non-trans channel. RAO channel
   setup skips this last enable message.

The error paths unwind the resources already created. Normal teardown takes
the channel's transmit mutex, requests peer deletion while the device is
live, and releases the local resources. Peer-delete errors are logged and
local cleanup continues; teardown success by itself is not confirmation that
a disconnected peer processed the deletion.

The common-message adapter stores one common-channel handle per device and
places the requested common client type in each message. Calls using that
handle share its transmit serialization. Dedicated channels have their own
transmit mutexes.

Evidence: [reservation and allocation](../../driver/src/sdk_driver/comm/ub/host/msg/ascend_ub_non_trans_chan_adapt.c#L19),
[create/enable exchange and unwind](../../driver/src/sdk_driver/comm/ub/host/msg/ascend_ub_non_trans_chan_adapt.c#L145),
[teardown](../../driver/src/sdk_driver/comm/ub/host/msg/ascend_ub_non_trans_chan_adapt.c#L241),
[common-channel selection and creation](../../driver/src/sdk_driver/comm/ub/host/msg/ascend_ub_msg_adapt.c#L115).

### One synchronous transaction owns a descriptor through both waits

`devdrv_sync_msg_send_inner` holds a device-status reference and the channel's
transmit mutex. It checks that the channel is enabled, increments its message
number, and borrows a send descriptor from the registered channel storage.
The request is CPU-copied into that descriptor. The caller's ordinary kernel
buffer is therefore not itself the SEND source.

The descriptor carries the message number, input length, requested reply
capacity, client type, and request bytes. The posted SEND uses the descriptor's
segment ID as `user_ctx`. These identifiers serve different purposes:

| Identifier | What it correlates |
|---|---|
| Send descriptor segment ID | Local SEND completion with the borrowed buffer |
| Message number | A received reply with the current request |
| Receive completion's segment ID | The posted receive buffer containing that reply |

After successful SEND completion, `ubdrv_wait_sync_msg_rqe` polls the reply
receive queue. A reply with another message number is reposted and the helper
continues waiting. A matching reply must carry a successful protocol status
before response bytes are copied. The copy helper checks the returned length
against both the caller's reply capacity and the receive descriptor's payload
capacity. The transaction reports the actual response length, processes the
protocol status, and attempts to repost the receive buffer.

Finally, it clears the send descriptor's payload, returns the descriptor to
the pool, unlocks the channel, and drops the device reference. Thus registered
storage lasts across transactions, while descriptor ownership spans the SEND
and reply phases of one transaction.

Evidence: [transaction and cleanup](../../driver/src/sdk_driver/comm/ub/common/msg/ascend_ub_non_trans_chan.c#L957),
[descriptor allocation and release](../../driver/src/sdk_driver/comm/ub/common/msg/ascend_ub_msg.c#L87),
[reply matching](../../driver/src/sdk_driver/comm/ub/common/ub_res/ascend_ub_jetty.c#L1140),
[reply copy bounds](../../driver/src/sdk_driver/comm/ub/common/msg/ascend_ub_msg.c#L63),
[protocol result mapping](../../driver/src/sdk_driver/comm/ub/common/msg/ascend_ub_msg.c#L275).

### Message capacity and timeout are properties of this protocol

The request payload limit is:

```text
min(channel send descriptor size, 4096) - sizeof(ascend_ub_msg_desc)
```

The SEND helper includes the descriptor header in its transfer length. A
4 KiB descriptor is therefore not a 4 KiB application payload. The common
channel's default descriptor size is 4 KiB, and admin requests have their own
header-adjusted limit. This limit belongs to these kernel messages; it is not
the limit for HDC payloads, queue receiver READs, or SVM memcpy.

An ACK-timeout completion can rebuild the sending JFS/JFC and repost the
request within the configured retry budget. The transaction keeps the same
message number. Other posting/polling failures, an unsuccessful completion,
or failure to obtain the matching reply return an error through cleanup.
The host resend branch and reply correlation do not establish exactly-once
service execution in the unavailable peer implementation.

Polling constants vary between ordinary and ESL/FPGA builds. Separate SEND
and reply waits, resource acquisition, retries, and scheduler delay mean that
the constants must not be treated as a measured end-to-end latency bound.

Evidence: [request preparation and capacity](../../driver/src/sdk_driver/comm/ub/common/msg/ascend_ub_non_trans_chan.c#L892),
[SEND header and opcode](../../driver/src/sdk_driver/comm/ub/common/ub_res/ascend_ub_jetty.c#L674),
[message limits](../../driver/src/sdk_driver/comm/ub/inc/ascend_ub_jetty.h#L31),
[common descriptor default](../../driver/src/sdk_driver/comm/ub/inc/ascend_ub_non_trans_chan.h#L27),
[retry branch](../../driver/src/sdk_driver/comm/ub/common/msg/ascend_ub_non_trans_chan.c#L1001),
[build-dependent polling constants](../../driver/src/sdk_driver/comm/ub/inc/ascend_ub_msg.h#L22).

## 3. Main clients and their additional result boundaries

| Client | Visible forwarding route | Interpretation |
|---|---|---|
| DMS/URD forwarded feature | `dms_urd_forward_send_to_device` → `devdrv_manager_h2d_sync_urd_forward` → dedicated `devdrv_sync_msg_send` | Management request and response; host-only features do not take this route |
| Remote ESCHED operation | `esched_host_msg_send` → `devdrv_common_msg_send(..., DEVDRV_COMMON_MSG_ESCHED)` | Event/control request dispatched by the remote service |
| SVM KMC | `svm_kmc_h2d_send` → SVM's dedicated channel → `devdrv_sync_msg_send` | Kernel memory-management coordination |
| SVM UMC | `svm_umc_send` → synchronous or asynchronous HAL ESCHED submission | Userspace memory-service protocol, including resource and segment exchanges |
| HDC session control | `HDCDRV_SEND_CTRL_MSG` → HDC kernel control channel | Session establishment/teardown, separate from normal payload SEND |
| TRS remote memory event | Copy request into device memory, then submit an event carrying its address | Combines the memcpy path with a control notification |

The DMS forwarder and its manager-channel sender are visible in
[URD forwarding](../../driver/src/sdk_driver/dms/devmng/dc/urd_forward/dms_urd_forward.c#L193)
and [manager request/reply](../../driver/src/sdk_driver/dms/devmng/drv_devmng/drv_devmng_host/ascend910/devdrv_manager_msg.c#L208).
The existing [DMS document](../dms_subsystem_urd_dispatch_and_control_plane.md)
covers the surrounding feature dispatcher.

For the forwarded DMS route, the manager wraps the URD request with device/VF
identity and a manager message header. On return it checks the transport call,
manager result, response magic, and response length, then extracts a separate
device command result before copying the command output. The outer URD helper
checks the output capacity again. A successful UB transaction is only one
layer of this management call's success conditions.

Remote ESCHED uses the common-message route in
[host message submission](../../driver/src/sdk_driver/esched/dc/host_adapt/esched_host_msg.c#L37)
and [remote event publishing](../../driver/src/sdk_driver/esched/dc/host_adapt/esched_host_msg.c#L74).
The sender checks `msg->head.error_code` after the common-message result.
The remote publishing helper packages event metadata and optional inline
message bytes. Its normal build rejects supplied event-ack/finish callbacks;
the synchronous control exchange must not be presented as such a callback.
Chapter 04 describes a different queue-notification route that builds a topic
SQE and reuses TRS. These are explicit alternatives in the source, so “all
events use kernel SEND” would be too broad.

SVM's two service layers are also distinct:
[UMC](../../driver/src/ascend_hal/svm/v3/umc/umc_client.c#L141) submits HAL events,
while [KMC H2D](../../driver/src/sdk_driver/svm/v3/kmc/h2d/kmc_h2d.c#L33) sends on
a kernel channel and interprets the returned message status. They coordinate
memory operations; they are not two additional bulk memcpy engines.

KMC obtains the channel from a referenced device context and calls
`devdrv_sync_msg_send`; when output is requested, it also returns the KMC
header's result. UMC builds an event and reply storage, selects
`halEschedSubmitEventSync` or `halEschedSubmitEvent`, and parses a service
reply only for the synchronous selection. The H2D UMC wrapper serializes
through a semaphore selected by device and subevent. “Async” here describes
the event submission choice, rather than a new DMA transfer operation.

HDC's UB kernel adapter calls `devdrv_sync_msg_send` with
`hdc_dev->ctrl_msg_chan`. This is the session-control route surrounding the
registered-buffer payload SENDs described in chapter 04.
See [HDC UB control sender](../../driver/src/sdk_driver/hdc/ub/host/hdcdrv_adapt_ub.c#L156).

`trs_svm_mem_event_sync` illustrates how one logical request can contain both
data and control phases. It allocates device memory, calls `drvMemcpy` to upload
the request, then submits an event containing the device address and waits for
a reply. SQ/CQ allocation uses its specialized synchronous event helper; other
subevents use `halEschedSubmitEventSync`. Consequently, remote async-WQE
preparation from chapter 02 can itself require a synchronous H2D request upload
before the device service prepares the eventual data-copy WQEs.
See [TRS request upload and event](../../driver/src/ascend_hal/trs/remote/master/trs_master_event.c#L325).

The TRS helper keeps the temporary device allocation until the synchronous
event helper returns and then frees it, including on submission/wait error.
It also validates the returned reply length. This documents the visible
host cleanup order; cancellation or quiescence of remote work after a timeout
requires the event service's contract.

## 4. UB admin messages support channel lifecycle

`ubdrv_admin_send_msg` validates the request and device state, obtains the admin
channel, and invokes the basic-channel send transaction. These messages support
the UB driver's resource-management protocol. For example, the URMA channel
code sends admin requests when creating or freeing its remote counterpart.
See [admin sender](../../driver/src/sdk_driver/comm/ub/common/msg/ascend_ub_admin_msg.c#L501)
and [remote URMA-channel management](../../driver/src/sdk_driver/comm/ub/common/msg/ascend_ub_urma_chan.c#L446).

The admin channel is a separate logical role from an application's queue or
HDC session. It is part of the setup and lifecycle cost supporting the later
traffic. It should not be counted as a separate user bulk-transfer API merely
because it also issues transport work.

The admin-channel getter acquires its mutex and checks whether the channel is
enabled. That mutex remains held until `ubdrv_put_admin_msg_chan` after the
transaction. `ubdrv_basic_chan_send` allocates a descriptor, stores an admin
opcode, SENDs it, checks transport completion, waits for a matching reply,
processes status/output, reposts the receive buffer, and returns the cleared
send descriptor. Its ACK-timeout branch can rebuild and repost.

For kernel URMA channels, allocation creates local JFC/JFR/JFS resources,
sends `UBDRV_CREATE_URMA_CHAN`, imports the peer JFR from the reply, and marks
the channel enabled. Freeing marks the channel disabled, unimports the peer
JFR, attempts `UBDRV_FREE_URMA_CHAN`, and releases local resources before
marking it idle. The admin sender rejects a dead device; local cleanup still
proceeds. The setup exchange establishes the channel, and later READ/WRITE
calls use it directly.

Evidence: [admin locking](../../driver/src/sdk_driver/comm/ub/common/msg/ascend_ub_admin_msg.c#L333),
[admin transaction](../../driver/src/sdk_driver/comm/ub/common/msg/ascend_ub_admin_msg.c#L431),
[URMA allocation and free](../../driver/src/sdk_driver/comm/ub/common/msg/ascend_ub_urma_chan.c#L461).

## 5. Kernel READ/WRITE and TRS register access

The PBL `devdrv_urma_copy` wrapper selects `ubdrv_urma_copy` on UB. The caller
supplies local and peer segment descriptors, offsets, lengths, a channel type,
and a direction. `LOCAL_TO_PEER` selects WRITE, while `PEER_TO_LOCAL` selects
READ. The implementation serializes the channel, constructs the WR, posts
through UBCORE, and polls its completion.
See [PBL dispatch](../../driver/src/sdk_driver/pbl/msg_chan/common/msg_chan_dev_info_comm.c#L321),
[direction selection](../../driver/src/sdk_driver/comm/ub/common/msg/ascend_ub_urma_chan.c#L130),
[kernel copy](../../driver/src/sdk_driver/comm/ub/common/msg/ascend_ub_urma_chan.c#L139),
and [READ/WRITE posting](../../driver/src/sdk_driver/comm/ub/common/ub_res/ascend_ub_jetty.c#L736).

TRS register operations are a concrete H2D use. `trs_ub_stars_reg_op` selects the
imported notification, counter-notification, or SQ-register segment. For a
write, it copies the value into a registered local staging area, then calls
`devdrv_urma_copy(..., URMA_CHAN_TSDRV, LOCAL_TO_PEER, ...)`. Reads perform the
reverse operation and copy the received value back to the caller.
See [TRS register access](../../driver/src/sdk_driver/trsdrv/trs/lba/near/sia/adapt/trs_host_ub/trs_ub_host_init.c#L744).

This is distinct from the fast userspace task upload: chapter 03 posts SQ
entry WRITEs and a tail WRITE from HAL, whereas this path is kernel
resource/register control. Direct mapped accesses are another possibility
where the relevant UB mapping supports them; chapter 01 describes the mapping
foundation.

### The copy accepts existing registrations and waits for transport

`ubdrv_urma_copy` checks the local/peer segment pointers, direction and channel
type, offsets and ranges, and equal transfer lengths. Under the channel's
transmit mutex it requires an enabled channel, builds source and destination
addresses from segment bases plus offsets, and posts one READ or WRITE with
completion enabled. It then polls the JFS completion.

There is no SEND-style service reply in this helper. The caller owns the
supplied segment registrations; this function neither registers an ordinary
caller buffer nor unregisters the supplied segments on return. The BBOX
channel disables the polling helper's device-status check, while other channel
types enable it. This does not skip the transport-completion status check.

On an ACK-timeout completion the helper invokes send-resource rebuilding and
returns an error. It does not repost the copy within this call. This differs
from the non-trans/admin and RAO resend loops.

Evidence: [copy validation and execution](../../driver/src/sdk_driver/comm/ub/common/msg/ascend_ub_urma_chan.c#L64),
[READ/WRITE construction](../../driver/src/sdk_driver/comm/ub/common/ub_res/ascend_ub_jetty.c#L736).

### TRS retains local staging and imports specific remote register ranges

TRS device initialization allocates/registers `reg_seg`. Its jetty-bind
exchange returns notification, counter-notification, and SQ-register segment
information; `trs_ub_import_reg_segment` imports those ranges using the peer
token. `trs_ub_stars_reg_op` chooses among them using the resource type.

```text
Register write:
  caller value -> CPU copy into local reg_seg
               -> URMA_CHAN_TSDRV WRITE
               -> transport completion -> return

Register read:
  imported device register segment -> URMA_CHAN_TSDRV READ
                                   -> transport completion
                                   -> CPU copy out of local reg_seg
```

Concrete callers reset notification resources and read/write SQ head/tail
registers. The SQ helper calculates a local staging offset separately from
the device register offset. Unbinding drops the imported register segments;
device adaptation teardown releases local staging.

Evidence: [local segment lifecycle](../../driver/src/sdk_driver/trsdrv/trs/lba/near/sia/adapt/trs_host_ub/trs_ub_host_init.c#L1021),
[register import/unimport](../../driver/src/sdk_driver/trsdrv/trs/lba/near/sia/adapt/trs_host_ub/trs_ub_host_init.c#L697),
[bind exchange](../../driver/src/sdk_driver/trsdrv/trs/lba/near/sia/adapt/trs_host_ub/trs_ub_host_init.c#L921),
[reset and SQ head/tail callers](../../driver/src/sdk_driver/trsdrv/trs/lba/near/sia/adapt/trs_host_ub/trs_ub_host_init.c#L802).

## 6. RAO transfers use registered client ranges

The RAO API uses a similar registered-buffer arrangement. A RAO WRITE reads
from the channel's local send segment and targets an imported peer segment;
a RAO READ reverses the source/destination roles. The shared helper posts the
chosen READ or WRITE and checks completion. The visible implementation does
not justify interpreting every RAO call as a fabric atomic instruction.
See [RAO wrapper](../../driver/src/sdk_driver/pbl/msg_chan/dc/host/msg_chan_rao.c#L34),
[source/destination setup](../../driver/src/sdk_driver/comm/ub/host/unit/ascend_ub_rao_adapt.c#L130),
and [RAO posting](../../driver/src/sdk_driver/comm/ub/host/unit/ascend_ub_rao_adapt.c#L184).

A management dump-control update uses `devdrv_rao_write`, illustrating H2D
control bytes in this family. By contrast, a diagnostic read that uses
`PEER_TO_LOCAL` has D2H payload even though the host initiated the request.
See [management write example](../../driver/src/sdk_driver/dms/devmng/dc/bbox_dump/dms_bbox_dump_host.c#L463).

Registration records a client type and its local address/length, then creates
a channel in `UBDRV_MSG_CHAN_FOR_RAO` mode through the non-trans resource
exchange. Individual read/write calls supply an offset and length rather than
an arbitrary new buffer. The helper applies that offset to both source and
destination segments.

The management write example first copies the caller's bytes into the local
registered region and then calls `devdrv_rao_write`. This local preparation
is part of the H2D update. A successful RAO transfer alone does not report that
the remote consumer has acted on the new status/control bytes.

RAO checks the requested range and enabled channel, holds the device read
semaphore and channel transmit mutex during the operation, and polls transport
completion. Its ACK-timeout branch can rebuild and repost within the retry
budget. Unregistering a RAO client releases its channel through the shared
channel-free machinery.

| Behavior | Kernel `devdrv_urma_copy` | RAO read/write |
|---|---|---|
| Buffer selection | Caller-supplied local/peer segments and separate offsets | Registered client ranges, one offset applied to both sides |
| Payload operation | READ or WRITE | READ or WRITE |
| Wait after posting | Transport completion | Transport completion |
| ACK-timeout branch | Rebuild, then return error | Rebuild and potentially repost |
| Service reply per transfer | None in this helper | None in this helper |

Evidence: [RAO registration and teardown](../../driver/src/sdk_driver/comm/ub/host/unit/ascend_ub_rao_adapt.c#L31),
[RAO offsets, validation, and retry](../../driver/src/sdk_driver/comm/ub/host/unit/ascend_ub_rao_adapt.c#L130),
[RAO locking and public adapters](../../driver/src/sdk_driver/comm/ub/host/unit/ascend_ub_rao_adapt.c#L234).

## 7. Boot delivery shares a host SVA window

Boot delivery uses the loader's UMMU/SVA setup. `ubdrv_load_sva_alloc` enables
kernel SVA, binds the device, obtains a TID, allocates the host buffer, and grants
`UMMU_DEV_READ`. The loader records the virtual buffer value in both its CPU
address and its `dma_addr` field. That field name does not make this a
conventional PCIe physical DMA-address allocation.
See [SVA allocation and grant](../../driver/src/sdk_driver/comm/ub/common/load_image/ascend_ub_load.c#L453)
and [load-buffer bookkeeping](../../driver/src/sdk_driver/comm/ub/common/load_image/ascend_ub_load.c#L510).

The transfer window is 48 MiB in this snapshot. A larger image is handled in
parts; 48 MiB is not the maximum total image length.
See [window-size definition](../../driver/src/sdk_driver/comm/ub/inc/ascend_ub_load.h#L42).

### Allocation grants device read access; it does not upload an image

`ubdrv_alloc_load_segment` first allocates a UMMU target device and block
bookkeeping, then calls the SVA allocator. The successful sequence is:

```text
allocate UMMU target device
  -> enable kernel SVA -> bind in table mode -> obtain TID
  -> allocate and clear host virtual buffer
  -> grant UMMU_DEV_READ for that buffer
  -> store one window's CPU address, published address, and capacity
```

The allocation is reused when `loader->blocks` already exists. Image bytes
arrive in this buffer through kernel file reads. Publishing its virtual
address and TID gives firmware the access information; the visible path does
not build the runtime SVM memcpy WQEs or call the kernel READ/WRITE helper
above to push the image.

`ubdrv_free_load_segment` attempts to revoke the grant, flushes the IOTLB
range, frees the host allocation, unbinds SVA, disables kernel SVA, and frees
the block bookkeeping and target device. The code logs several cleanup errors
and continues. This is the visible release sequence, not proof of an
unavailable firmware reader's behavior during error recovery.

Evidence: [allocation and error unwind](../../driver/src/sdk_driver/comm/ub/common/load_image/ascend_ub_load.c#L453),
[reuse and published address](../../driver/src/sdk_driver/comm/ub/common/load_image/ascend_ub_load.c#L510),
[revocation, IOTLB flush, and release](../../driver/src/sdk_driver/comm/ub/common/load_image/ascend_ub_load.c#L560).

### Two control interfaces advertise the same kind of buffer

| Control flow | How it advertises/refills the host buffer | Visible completion |
|---|---|---|
| BIOS mailbox loader | Publishes host identity/TID and block address/size in mapped BAR memory; updates ready flags; waits for BIOS flags before reusing the window | BIOS flag handshake, then boot-status update and resource cleanup |
| UBIOS/UVB callback loader | Returns buffer address, image size, file ID, TID as `token_id`, and section size through a CIS message callback; subsequent callbacks request another file offset | Loading/final-state callbacks; successful final state frees the buffer and updates boot status |

The mailbox probe code is within the host file's
`CFG_FEATURE_SLAVE_MODE` branch, and registering that UB driver is conditional
on the same feature. The UBIOS callback bodies and registration use
`CFG_FEATURE_CDMA_LOAD_IMAGE`, which the checked-in UB Makefile enables.
These source/build selectors do not establish which flow a particular
installed firmware invokes.

Evidence: [slave-mode branch](../../driver/src/sdk_driver/comm/ub/host/ascend_ub_main_adapt.c#L50),
[conditional UB driver registration](../../driver/src/sdk_driver/comm/ub/host/ascend_ub_main_adapt.c#L809),
[callback feature selection](../../driver/src/sdk_driver/comm/ub/host/load_image/ascend_ub_load_image_adapt.c#L364),
[UB Makefile](../../driver/src/sdk_driver/comm/ub/Makefile#L67).

## 8. BIOS mailbox flow: publish, wait, refill

For the mailbox flow, host probe schedules `ubdrv_load_file`. The worker sets
the mailbox base from `res.mem_bar.va`, allocates the shared window, publishes
identity and TID, and begins the BIOS handshake. `ubdrv_load_file_fill_blocks`
reads image bytes into the host window, writes address/size pairs into the
mailbox, and signals partial or final readiness. It waits for the BIOS before
reusing the buffer for a subsequent part.
See [work scheduling](../../driver/src/sdk_driver/comm/ub/host/ascend_ub_main_adapt.c#L204),
[mailbox setup](../../driver/src/sdk_driver/comm/ub/common/load_image/ascend_ub_load.c#L213),
[chunk publication](../../driver/src/sdk_driver/comm/ub/common/load_image/ascend_ub_load.c#L317),
and [worker lifecycle](../../driver/src/sdk_driver/comm/ub/common/load_image/ascend_ub_load.c#L766).

The normal protocol sequence is:

1. Host publishes EID/UPI/CNA and TID, then sets `UBDRV_LOAD_HOST_READY`.
2. BIOS sets `UBDRV_BIOS_REQUIRE_FILE` and a file ID.
3. Host resolves the configured image path, opens the file, and fills up to
   one window. It publishes block count and address/length pairs.
4. If more of that file remains, host sets `UBDRV_LOAD_FILE_PART_READY` and
   waits for `UBDRV_BIOS_LOAD_FILE_FINISH` before overwriting the window.
5. For the last portion of a file, host sets `UBDRV_LOAD_FILE_READY`.
   The outer file helper waits for another file request or all-files success.
6. BIOS `UBDRV_BIOS_LOAD_FILE_SUCCESS` finishes the file sequence. The host
   clears the flag, advances its recorded boot status to OS, and starts a
   timer that separately checks device startup progress.

The publication path contains write barriers around the metadata/flag
updates. That shows host publication ordering in this code; it does not
specify all cache-coherency behavior of the external firmware/UMMU stack.

`ubdrv_wait_for_flag_change` polls at 20 ms intervals, checks an abort flag,
and accepts either its expected flag or the all-files-success flag. Its
count is build-dependent. A timeout or unexpected flag returns an error;
abort during the wait returns cancellation.

Once the worker has allocated the window, it calls the release helper after
the loading attempt returns, on success or failure. Probe teardown sets
abort, removes the startup timer, and synchronously cancels/joins the work.
The timer's later startup check is separate from the BIOS file-consumption
handshake and from the host's recorded boot-status update.

Evidence: [flags and host identity](../../driver/src/sdk_driver/comm/ub/common/load_image/ascend_ub_load.c#L204),
[flag wait](../../driver/src/sdk_driver/comm/ub/common/load_image/ascend_ub_load.c#L288),
[file-to-file handoff](../../driver/src/sdk_driver/comm/ub/common/load_image/ascend_ub_load.c#L644),
[start and startup timer](../../driver/src/sdk_driver/comm/ub/common/load_image/ascend_ub_load.c#L702),
[probe teardown](../../driver/src/sdk_driver/comm/ub/host/ascend_ub_main_adapt.c#L217).

## 9. UBIOS/UVB flow: callbacks retain the window across file requests

For the callback flow, the file-info handler allocates the same kind of window,
loads its first part, and returns the address and access metadata. The
loading-state handler refills the window for a requested offset. The successful
final-state handler releases the resources. These functions are registered as
UBIOS message callbacks during loader initialization.
See [file-info response](../../driver/src/sdk_driver/comm/ub/host/load_image/ascend_ub_load_image_adapt.c#L209),
[window refill](../../driver/src/sdk_driver/comm/ub/host/load_image/ascend_ub_load_image_adapt.c#L72),
[state callbacks](../../driver/src/sdk_driver/comm/ub/host/load_image/ascend_ub_load_image_adapt.c#L107),
and [callback registration](../../driver/src/sdk_driver/comm/ub/host/load_image/ascend_ub_load_image_adapt.c#L364).

The callbacks have distinct responsibilities:

| Callback | Visible behavior |
|---|---|
| Get ability | Advertises `UBDRV_UB_MEMORY_MASK` |
| Get file info | Parses a numeric file ID, allocates/reuses the window, records BIOS status, resolves the file and loads offset zero |
| Get file loading state, state 1 | Refills the existing window from the requested file offset |
| Get file loading state, state 0 | Reports that file's transfer succeeded; retains the window |
| Get final loading state, state 0 | Frees the window and records OS boot status |

The file-info response contains `file_address`, `file_total_size`,
`file_id`, `token_id` (the loader's TID), and `section_size` (48 MiB).
The refill helper reads from the saved file path at the supplied offset,
using up to the window capacity, and requires the full requested read to
succeed. The callback interface therefore coordinates reuse of a persistent
window rather than allocating one buffer per image portion.

File-info failures after allocation call the release helper. The loading
state callback does not release the window on a per-file success. A nonzero
final state is rejected before the successful-final cleanup; it must not be
described as automatically freeing the window. Loader uninitialization
unregisters the callbacks and releases remaining loader windows.

Evidence: [refill](../../driver/src/sdk_driver/comm/ub/host/load_image/ascend_ub_load_image_adapt.c#L63),
[per-file and final states](../../driver/src/sdk_driver/comm/ub/host/load_image/ascend_ub_load_image_adapt.c#L107),
[file-info fields and failure cleanup](../../driver/src/sdk_driver/comm/ub/host/load_image/ascend_ub_load_image_adapt.c#L209),
[ability and callback table](../../driver/src/sdk_driver/comm/ub/host/load_image/ascend_ub_load_image_adapt.c#L303),
[loader uninitialization](../../driver/src/sdk_driver/comm/ub/host/load_image/ascend_ub_load_image_adapt.c#L382).

```text
Host image file
  -> kernel read into SVA buffer
  -> device-read permission + host identity/address/TID/size publication
  -> device/firmware consumes the buffer
  -> handshake or callback permits refill/cleanup
```

Device pull is the contract exposed by the host allocation and notification
code. The actual BIOS/device reader is outside this implementation. The two
control flows are both present in the tree; this analysis does not assert that
every deployment runs both, or that the boot reader issues runtime URMA JFS
operations. Their control notifications differ from the runtime copy APIs.

## 10. Ownership and completion across P7–P10

| Resource | When it is prepared | Visible reuse/release boundary |
|---|---|---|
| Kernel message send descriptor | Borrowed and populated per request | Cleared and returned on transaction exit after the SEND/reply attempt |
| Kernel message receive buffer | Posted from registered channel storage | Reposted after handling a matching reply, or discarding a mismatched reply |
| Dedicated/common channel | Admin-assisted resource exchange | Channel teardown, serialized against sending |
| TRS temporary device request | Allocated and uploaded before its event | Freed after the synchronous event helper returns, including errors |
| TRS register staging/imports | Device adaptation initialization and bind | Local segment teardown and peer-segment unimport |
| RAO client ranges/channel | Client registration | Client/channel teardown |
| Mailbox boot window | Loading worker | Released after loading attempt; intermediate refill follows BIOS handshake |
| Callback boot window | File-info callback, reused if already allocated | Successful final callback, file-info failure cleanup, or loader uninitialization |

These are source-level ownership boundaries. An error return after submission
does not, by itself, prove that the device never observed the request or that
all remote work has stopped. The missing service and firmware implementations
are needed to complete that recovery contract.

## 11. Inventory boundaries and follow-up

Shared mapping setup can expose pages for future access without copying bytes
at setup time. Device-local memset can send control without uploading the
requested number of payload bytes. Diagnostic collection is often D2H. These
distinctions prevent counting every memory-related API as another H2D data
engine.

The legacy PCI VNIC host initialization explicitly excludes UB connections.
It is therefore not an additional active UB H2D path in this snapshot.
See [VNIC connection check](../../driver/src/sdk_driver/vnic/host/pcivnic_host.c#L497).

[Document 06 — ACL/runtime dispatch](06_acl_runtime_dispatch.md) connects
selected application memcpy, kernel/argument launch, and TDT send actions
to the HAL paths, including stream completion and buffer ownership. Service
reply contracts in this chapter remain distinct from those task/queue
completion boundaries. Hardware timing, complete device execution, and
external transport/IOMMU internals require evidence beyond these checkouts.
