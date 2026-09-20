# Kernel control, register access, and boot over UB

[Summary](README.md) · Previous: [queue and HDC](04_queue_and_hdc.md)

This chapter covers P7–P10. These paths carry requests, register values, and
boot images. They complement the userspace payload submissions in the earlier
chapters. An ioctl, service request, or mapping operation does not necessarily
cross the interconnect: the relevant forwarding and connection branches must
be followed.

## Common and dedicated kernel message channels

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

`devdrv_sync_msg_send_inner` serializes a channel's sending transaction, allocates
a descriptor, posts SEND, polls its transport completion, then separately waits
for the reply corresponding to the message number. It copies the response to
the caller, handles the remote processing status, and replenishes the receive
resource. Transport success alone is therefore insufficient for this API's
successful request/reply result. Acknowledgment-timeout handling can rebuild
the sending resource and retry; the code should not be read as guaranteeing
application-level exactly-once effects for every arbitrary request.
See [complete synchronous transaction](../../driver/src/sdk_driver/comm/ub/common/msg/ascend_ub_non_trans_chan.c#L957).

“Non-trans” is a channel-family name in this code. These functions do transfer
request and reply bytes over UB; the name does not mean that their data stays
on the host.

## Main clients of the control paths

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

Remote ESCHED uses the common-message route in
[host message submission](../../driver/src/sdk_driver/esched/dc/host_adapt/esched_host_msg.c#L37)
and [remote event publishing](../../driver/src/sdk_driver/esched/dc/host_adapt/esched_host_msg.c#L74).
Chapter 04 describes a different queue-notification route that builds a topic
SQE and reuses TRS. These are explicit alternatives in the source, so “all
events use kernel SEND” would be too broad.

SVM's two service layers are also distinct:
[UMC](../../driver/src/ascend_hal/svm/v3/umc/umc_client.c#L141) submits HAL events,
while [KMC H2D](../../driver/src/sdk_driver/svm/v3/kmc/h2d/kmc_h2d.c#L33) sends on
a kernel channel and interprets the returned message status. They coordinate
memory operations; they are not two additional bulk memcpy engines.

`trs_svm_mem_event_sync` illustrates how one logical request can contain both
data and control phases. It allocates device memory, calls `drvMemcpy` to upload
the request, then submits an event containing the device address and waits for
a reply. SQ/CQ allocation uses its specialized synchronous event helper; other
subevents use `halEschedSubmitEventSync`. Consequently, remote async-WQE
preparation from chapter 02 can itself require a synchronous H2D request upload
before the device service prepares the eventual data-copy WQEs.
See [TRS request upload and event](../../driver/src/ascend_hal/trs/remote/master/trs_master_event.c#L325).

## UB admin messages

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

## Kernel READ/WRITE and register access

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

## Boot/image delivery: host buffer and two control flows

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

| Control flow | How it advertises/refills the host buffer | Visible completion |
|---|---|---|
| BIOS mailbox loader | Publishes host identity/TID and block address/size in mapped BAR memory; updates ready flags; waits for BIOS flags before reusing the window | BIOS flag handshake, then boot-status update and resource cleanup |
| UBIOS/UVB callback loader | Returns buffer address, image size, file ID, TID as `token_id`, and section size through a CIS message callback; subsequent callbacks request another file offset | Loading/final-state callbacks; successful final state frees the buffer and updates boot status |

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

For the callback flow, the file-info handler allocates the same kind of window,
loads its first part, and returns the address and access metadata. The
loading-state handler refills the window for a requested offset. The successful
final-state handler releases the resources. These functions are registered as
UBIOS message callbacks during loader initialization.
See [file-info response](../../driver/src/sdk_driver/comm/ub/host/load_image/ascend_ub_load_image_adapt.c#L209),
[window refill](../../driver/src/sdk_driver/comm/ub/host/load_image/ascend_ub_load_image_adapt.c#L72),
[state callbacks](../../driver/src/sdk_driver/comm/ub/host/load_image/ascend_ub_load_image_adapt.c#L107),
and [callback registration](../../driver/src/sdk_driver/comm/ub/host/load_image/ascend_ub_load_image_adapt.c#L364).

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

## Inventory boundaries and follow-up

Shared mapping setup can expose pages for future access without copying bytes
at setup time. Device-local memset can send control without uploading the
requested number of payload bytes. Diagnostic collection is often D2H. These
distinctions prevent counting every memory-related API as another H2D data
engine.

The legacy PCI VNIC host initialization explicitly excludes UB connections.
It is therefore not an additional active UB H2D path in this snapshot.
See [VNIC connection check](../../driver/src/sdk_driver/vnic/host/pcivnic_host.c#L497).

The next layer of investigation should connect a chosen application action to
these HAL/kernel paths, then match transport completion, service reply, task
completion, and buffer ownership. Hardware timing, complete device execution,
and external transport/IOMMU internals require evidence beyond this checkout.
