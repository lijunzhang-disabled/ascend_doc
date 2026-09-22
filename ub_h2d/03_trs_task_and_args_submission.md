# 03 — TRS task and argument submission over UB

[Summary](README.md) · Previous: [memcpy](02_sync_and_async_memcpy.md) · Next: [queue and HDC](04_queue_and_hdc.md)

Source snapshot and evidence boundaries: [series scope](README.md#scope-and-evidence).
This chapter expands P3 and P4: moving task descriptors and arguments from host
memory into device memory. It also follows the host task-report return path
far enough to separate submission, transport reporting, and task completion.
The complete device scheduler remains outside this trace. The upper runtime
caller is connected in [document 06](06_acl_runtime_dispatch.md).

## 1. The submission contract

**Argument upload posts host-issued WRITEs without ringing the transport
doorbell. Normal task submission posts SQ-entry WRITEs followed by an SQ-tail
WRITE, then rings that doorbell.** Using the same SQ-associated JFS gives the
host code a way to order arguments before the task that consumes them.

| Stage | Host action | What successful return establishes |
|---|---|---|
| Argument preparation | Caller provides argument addresses and usable segment handles | Memory/segment preparation only |
| `halSqTaskArgsAsyncCopy` | Post argument WRITE chain on the SQ's JFS; advance JFS PI | Transport work was posted; this call does not ring the doorbell or wait for the copy |
| `halSqTaskSend` | Populate local SQ if needed; post SQ data and tail WRITEs; ring JFS doorbell | Submission succeeded and the software SQ tail advanced |
| JFC processing | Poll transport completion records that are reported | Transport status handling, without a successful CQE for every task/argument WRITE |
| Task-report receive | Receive CQ reports through TRS's kernel/report machinery | The returned reports are available for the caller to interpret and match to work |

The device READ WQEs in [document 02](02_sync_and_async_memcpy.md) are a
separate kind of work. Preparing an async memcpy WQE does not itself upload
an SQ entry. Conversely, uploading a task descriptor does not mean the task
has executed. [Document 06](06_acl_runtime_dispatch.md) connects the ordinary
runtime ASYNCDMA format and launch sequence; these are not established by the
host posting code alone.

Evidence: [argument posting](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L1071),
[task posting and return](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L937),
[report receive entry](../../driver/src/ascend_hal/trs/core/trs_sqcq.c#L1795).

### Which public calls select this path

`halSqTaskSend` looks up the requested SQ, checks that it supports sending and
has SQ memory, and selects `trs_sq_task_send_urma` for `DRV_NORMAL_TYPE` over
UB. Other SQ types go through generic task handling. Likewise,
`halSqTaskArgsAsyncCopy` requires a normal SQ, a UB connection, nonzero source,
destination, and size, and an existing SQ. Its lower helper additionally
requires non-null source and destination segment handles.

The host build includes `trs_urma.c`, `trs_master_urma.c`, and
`trs_master_async.c` for Ascend 950 when `ENABLE_UBE` is enabled. The general
interface file also contains weak fallbacks in other build configurations.
The presence of a public symbol therefore does not establish which transport
implementation was linked.

Evidence: [task dispatch](../../driver/src/ascend_hal/trs/core/trs_interface.c#L1019),
[argument dispatch](../../driver/src/ascend_hal/trs/core/trs_interface.c#L472),
[host build selection](../../driver/src/ascend_hal/trs/dc/CMakeLists.txt#L31).

## 2. Resources established before sending

### Process resources and SQ resources have different lifetimes

The TRS process context is indexed by device. It holds a URMA context, EID,
token, JFCE, local JFR, and a lazily imported remote receive endpoint. SQ
contexts borrow those shared resources while retaining their own JFS, JFC,
producer index, queue-segment imports, and host buffers.

The process initialization path obtains UB device information, creates the
URMA context and receive resources, allocates a bank of eight segment token
IDs, and starts/registers transport event polling. A local SQ registration
selects a token ID using `sq_id % 8`; it does not allocate one token ID for
every SQ. Details of registration versus memory mappings are in
[document 01](01_transport_and_memory.md#5-registering-local-and-remote-memory).

| Resource | Scope | Purpose |
|---|---|---|
| URMA context, EID, token, local JFR, JFCE | Process/device | Shared transport identity, receive endpoint, and event source |
| Imported remote JFR (`tjetty`) | Process/device, reused by SQ contexts | Target endpoint for the posted WRITEs |
| Host SQ mapping and registered `local_tseg` | SQ | Source ring containing SQ-entry bytes |
| Imported `sq_que_tseg` | SQ | Device SQ-entry destination ring |
| Host notification buffer and local segment description | SQ | Source storage for eight-byte SQ-tail updates |
| Imported `sq_tail_tseg` | SQ | Device destination of the tail WRITE |
| JFS, JFC, doorbell address, and JFS PI | SQ | Post and activate ordered transport work; process transport reports |
| Reusable argument WR/SGE array | SQ | Construct argument-copy chains |
| Async-copy contexts | SQ | Device READ preparation from document 02; distinct from this host JFS's PI |

Evidence: [process initialization](../../driver/src/ascend_hal/trs/core/urma/trs_urma.c#L495),
[token bank and polling setup](../../driver/src/ascend_hal/trs/core/urma/trs_urma.c#L566),
[SQ context sharing](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L339),
[remote endpoint reuse](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L252),
[resource structures](../../driver/src/ascend_hal/trs/core/urma/trs_urma.h#L40).

### Allocation handshake

For the normal locally managed UB SQ/CQ case, `halSqCqAllocate` selects
`trs_sqcq_urma_alloc`. Its successful sequence is:

```text
allocate SQ URMA context referencing the process resources
  -> request remote SQ/CQ allocation and receive resource descriptions
  -> import the remote endpoint, SQ segment, and SQ-tail segment
  -> create the SQ's host JFS/JFC and obtain its doorbell address
  -> allocate local SQ/CQ using the returned IDs
  -> attach the SQ URMA context and register the host SQ buffer
  -> allocate notification storage and obtain its SVM segment
```

The remote request contains the SQ/CQ input parameters, host JFR ID, token,
and optional extended information. It uses
`DRV_SUBEVENT_TRS_ALLOC_SQCQ_WITH_URMA_MSG` through
`trs_svm_mem_event_sync`. As traced in document 02, that helper first copies
the request into temporary device memory with synchronous `drvMemcpy`, then
submits the event and waits for a reply. Allocation is therefore a control
exchange that can itself contain H2D copying.

The reply supplies SQ/CQ IDs, remote receive-resource information, and device
SQ/tail segment descriptions. Local allocation then forces the specified
IDs so that both sides refer to the same resources. Failure branches unwind
resources already acquired; they do not make later payload posting part of
the allocation transaction.

SQs created without SQ memory skip the queue-memory registration/import work.
The normal `halSqTaskSend` entry rejects that mode, so the WRITE sequence below
assumes a fully initialized SQ with memory.

Evidence: [public allocation dispatch](../../driver/src/ascend_hal/trs/core/trs_interface.c#L253),
[allocation exchange](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L402),
[local allocation with specified IDs](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L531),
[allocation orchestration and unwind](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L594),
[event upload](../../driver/src/ascend_hal/trs/remote/master/trs_master_event.c#L325).

### Local registration and notification storage

`trs_master_register_segment` registers the mapped local SQ directly with
URMA. It supplies the process token and selected token ID, non-cacheable
READ/WRITE/ATOMIC access, and `non_pin = 1`; the source comment ties that flag
to memory allocated as pages. This registration is separate from looking up
an existing managed-host allocation.

The notification buffer is allocated as host SVM memory with
`halMemAlloc`, sized as **SQ depth × 8 bytes**. `halMemGetSeg` supplies its
segment description, which TRS stores in a local target-segment wrapper.
For each submission, the new tail value is written into the buffer slot at
`new_tail * 8`. The tail WRITE reads eight bytes from that slot. The buffer
is an array of tail-value slots, rather than one eight-byte variable reused
for every submission.

Remote SQ and tail segments are imported with the common non-cacheable,
READ/WRITE/ATOMIC, `URMA_SEG_NOMAP` import flags. Their addresses are transport
destinations; these imports do not establish CPU load/store mappings.

Evidence: [SQ and notification registration](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L184),
[notification-slot addressing](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L846),
[remote imports](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L293),
[import flags](../../driver/src/ascend_hal/trs/core/urma/trs_urma.h#L221).

### JFS posting and activation are deliberately separate

The host JFS is configured in reliable-message mode with high priority,
transport retry/error settings, and error suspension. `_trs_create_jfs` sets
`URMA_JFS_DB_STATUS` using the value annotated “disable db,” then activates
the JFS. `trs_create_jfs` retrieves `URMA_JFS_DB_ADDR` and allocates argument
WR/SGE scratch storage.

| Quantity in this snapshot | Value | Interpretation |
|---|---|---|
| Host JFS configured depth | 8,192 | Transport queue configuration; not the number of task SQ entries |
| Host JFC configured depth | 5,000 | Transport report capacity |
| JFS PI modulus | 65,536 | Software producer-index domain |
| Argument WR/SGE array length | 16 | Maximum WRs constructed for one argument-upload call |
| Argument WR chunk size | 256 MiB | Maximum bytes assigned to one argument WRITE |
| Task SQ depth and entry size | Supplied during allocation | Determine task-ring addressing and task credits |

Posting uses the external `urma_post_jfs_wr`. The explicit doorbell store in
TRS activates the posted work when requested. The provider's final posting,
queue-capacity, and hardware behavior are outside this source tree; the
host-side separation is visible in both JFS setup and call sites.

Evidence: [depth constants](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L168),
[JFS configuration](../../driver/src/ascend_hal/trs/core/urma/trs_urma.c#L101),
[doorbell lookup and scratch allocation](../../driver/src/ascend_hal/trs/core/urma/trs_urma.c#L154),
[argument constants](../../driver/src/ascend_hal/trs/core/urma/trs_urma.h#L23),
[SQ geometry initialization](../../driver/src/ascend_hal/trs/core/trs_sqcq.c#L719).

## 3. Normal task submission

### Task-ring credits and the source of SQ entries

```text
halSqTaskSend
  -> look up SQ and select normal-SQ UB implementation
  -> trs_sq_task_send_urma
       -> check SQ status
       -> trs_sq_task_send_check
            -> check in-ring source position when caller filled the SQ
            -> check SQ credits; refresh head once if needed
       -> trs_sq_task_fill when source is outside the local SQ
       -> set info.pos to the current software tail
       -> trs_sq_task_send_with_urma
            -> post SQ-data WRITE(s) and tail WRITE; ring JFS doorbell
       -> advance software SQ tail and send statistics
```

Task credits use the local `head`, `tail`, and `depth`, leaving one SQ entry
unused to distinguish a full ring from an empty one. If cached credit is
insufficient, the check refreshes the head through the mapped control field
or `halSqCqQuery`, then checks again. It returns `NO_RESOURCES` if the requested
entry count still does not fit.

This is separate from JFS capacity. Free task entries do not prove the
transport post will succeed. Although `halTaskSendInfo` has a `timeout` field,
the UB send helper has no queue-space wait loop using that field. It does not
turn an SQ-credit or JFS-full result into a timed retry operation.

TRS decides who populated the SQ by the address of `info->sqe_addr`:

| Source address | Preparation before transport posting |
|---|---|
| Outside the local SQ range | TRS copies each entry into the local SQ at the current tail, wrapping by SQ depth |
| Inside the local SQ range | Caller has filled the ring; address must be entry-aligned and point at the current tail |

The actual URMA source is the registered local SQ ring in both cases. Thus an
external descriptor array can incur a CPU copy even though the subsequent
H2D upload uses URMA. `info->pos` is the first entry's ring position; it is
assigned before transport posting and is not a completion token.

Evidence: [credit refresh and check](../../driver/src/ascend_hal/trs/core/trs_sqcq.c#L1508),
[source-position validation and CPU fill](../../driver/src/ascend_hal/trs/core/trs_sqcq.c#L1587),
[source-range classification](../../driver/src/ascend_hal/trs/core/trs_sqcq.h#L169),
[UB send state changes](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L1010),
[public send fields](../../driver/pkg_inc/hal_pkg/trs_pkg.h#L155).

### A submission produces two or three WRITEs

Let `T` be the old software tail, `N` the number of SQ entries, `D` the SQ
depth, and `E` the entry size. The new tail is `(T + N) % D`.

| Ring placement | Linked transport work |
|---|---|
| `T + N <= D` | WRITE `N * E` SQ bytes starting at entry `T`; WRITE the eight-byte new-tail value |
| `T + N > D` | WRITE `(D - T) * E` bytes at the end of the ring; WRITE the remaining entries at the start; WRITE the eight-byte new-tail value |

Ending exactly at the boundary takes the two-WR branch and publishes tail
zero. Crossing the boundary takes three WRs. These WR counts describe one
batched submission, not one WR per task descriptor. The task-data helper
splits at the ring boundary; it does not use the argument-copy chunking loop.

Each WR has one source SGE and one destination SGE. SQ data targets
`sq_que_tseg` plus the entry offset; the final WRITE targets the base of
`sq_tail_tseg`. The tail update is a separate WRITE, not an inline trailer
appended to the SQ-entry bytes.

Evidence: [source/destination SGE construction](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L846),
[contiguous and wrapping chains](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L937).

### Ordering, doorbell, and successful return

Both SQ-data and tail WRs request `place_order = 2`, annotated strong order,
and `comp_order = 1`, annotated completion ordering with the previous WR.
`trs_post_send_wr` posts the chain, advances the SQ's JFS PI by two or three
modulo 65,536, and, when `is_db_en` is true, performs a barrier followed by a
CPU store of the new PI to the JFS doorbell address.

On ARM/AArch64, the local `trs_smp_wmb` macro expands to `dmb st` with a compiler
memory clobber. It is empty in this file's other architecture branch. The
ordering discussion therefore describes the explicit source requests and the
intended UB platform; it is not a proof of fabric behavior on every build.

After that helper succeeds, TRS advances its software task-SQ tail by `N`
modulo `D` and updates send statistics. No transfer or task-execution wait
occurs on this successful path.

| Notification/state | Updated by | Meaning |
|---|---|---|
| JFS PI | Host posting helper | Number of posted transport WRs in its modulo domain |
| Local JFS doorbell | Host CPU store | Activate the transport work through that PI |
| Remote SQ tail | Final ordered URMA WRITE | Publish the device task-queue position after SQ data |
| Software SQ tail | Host after successful posting helper | Position for the next host task submission |

The local doorbell and remote SQ-tail WRITE have different destinations and
purposes. Neither is a task-completion report.

Evidence: [ordering and completion flags](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L895),
[post, PI, and doorbell](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L912),
[barrier definition](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L29),
[software tail update](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L1010).

## 4. Argument upload and its dependency on submission

### Preparing segment handles

`halSqTaskArgsInfo` carries source/destination addresses, a 32-bit byte size,
SQ identity, and pointers to two `halTsegInfo` objects. Argument posting uses
the supplied `tseg_ptr` values directly. It does not allocate a staging buffer,
CPU-copy the arguments, or temporarily register ordinary host memory.

The associated handle APIs are `halGetTsegInfoByVa` and `halPutTsegInfo`:

| Handle kind | Get behavior | Put behavior |
|---|---|---|
| Local, `flag == 1` | Obtain segment/token through `halMemGetSeg`; allocate a local target-segment wrapper and copy the segment description | Free the wrapper |
| Remote, flag described by the header as 0 | Obtain segment/token through `halMemGetSeg`; import the segment into the TRS process URMA context | Unimport the segment |

`halMemGetSeg` requires a suitable SVM range and consistent segment/token/owner
metadata. Its lookup does not create an extra in-flight allocation reference.
The local wrapper is not a fresh pin/registration of arbitrary memory, and
importing the remote segment does not allocate the destination. The caller
supplies memory whose registration and ownership already support the transfer.
See [the SVM/TRS bridge](01_transport_and_memory.md#managed-host-memory-and-the-trs-bridge).

Evidence: [public argument and segment fields](../../driver/pkg_inc/ascend_hal_define.h#L1304),
[public handle dispatch](../../driver/src/ascend_hal/trs/core/trs_interface.c#L1070),
[handle get/put](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L1123),
[argument handle validation](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L1052).

### Constructing and posting the argument chain

`trs_sq_task_args_async_copy` computes `ceil(size / 256 MiB)` WRs and checks
that they fit the 16-entry scratch array. It fills one source and destination
SGE per WR, advancing both addresses by the chunk offset and retaining the
supplied segment handles. Each WR is a WRITE with the same placement and
completion ordering flags as task upload.

The size is bounded by both the public 32-bit field and the WR construction
rules; the array length is not an unlimited streaming-copy mechanism. Also,
this helper does not walk different allocation properties and replace segment
handles between chunks. The handles must describe the requested transfer
ranges, or the caller must divide the operation at the appropriate boundary.

It then calls `trs_post_send_wr(..., is_db_en=false)`. A successful call
advances **JFS PI only**. It does not advance the task-SQ tail and does not
write the JFS doorbell. Argument copying also does not consume task-SQ entries
through `trs_sq_credit_check`; transport queue capacity is a separate concern.

Evidence: [argument construction and post](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L1071),
[shared posting helper](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L912),
[chunk and scratch limits](../../driver/src/ascend_hal/trs/core/urma/trs_urma.h#L23).

### Argument-before-task sequencing

A later `halSqTaskSend` on the same SQ posts its SQ-entry/tail chain to the
same JFS and rings the doorbell using the accumulated PI. The visible intended
sequence is therefore argument WRITEs, SQ-entry WRITE(s), then the device-tail
WRITE. Multiple successful argument posts can be included before that trigger,
subject to provider capacity.

```mermaid
sequenceDiagram
    participant C as Caller
    participant H as TRS HAL
    participant J as SQ-associated host JFS
    participant D as Device memory and SQ
    C->>H: halSqTaskArgsAsyncCopy
    H->>J: Post argument WRITE chain; advance PI; no doorbell
    H-->>C: Posting result
    C->>H: halSqTaskSend
    H->>J: Post SQ WRITE(s), then SQ-tail WRITE
    H->>J: Barrier and CPU write of accumulated PI to JFS doorbell
    H-->>C: Submission result
    J->>D: Ordered argument bytes
    J->>D: Ordered SQ-entry bytes
    J->>D: New SQ tail
    Note over H,D: Submission return can overlap transport progress
    Note over D: Task execution and task reports follow a separate contract
```

An isolated argument-post call does not perform this trigger. The helpers do
not return a standalone argument-copy completion object, nor do they roll
back earlier successful argument posts when a later task send fails.

The ordering dependency is scoped to the **same SQ-associated JFS**. It is
not a global order across SQs or across the SVM copy-channel pool. The visible
UB send and argument helpers do not wrap the argument-plus-task pair in one
critical section. Their mutable PI, task tail, and scratch storage make the
caller's ordering and serialization policy part of the interface contract.
[Document 06](06_acl_runtime_dispatch.md#8-kernel-launch-connects-host-argument-staging-to-ordered-task-upload)
shows that the selected ordinary Stars v2 kernel launch holds its stream
lock across argument loading and task submission. Locks elsewhere in generic
TRS paths do not by themselves establish this caller policy.

Evidence: [argument post without doorbell](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L1071),
[task post with doorbell](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L1002),
[UB send entry](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L1010),
[generic send locking for comparison](../../driver/src/ascend_hal/trs/core/trs_sqcq.c#L1679).

## 5. Transport reports and task reports

### Userspace JFC processing is transport reporting

Task, tail, and argument WRITEs all set `complete_enable = 0`. The adjacent
comment says a failed WQE still reports a CQE. Consequently, these posts do
not request a successful CQE for every uploaded SQ batch or argument chunk.
The existence of a JFC should not be interpreted as such a per-upload fence.

The process polling thread waits on device JFCE file descriptors through
`epoll`. For a ready device it obtains reported JFCs, uses the JFC's stored SQ
ID to locate the SQ, checks that the SQ/JFC association is still valid under
a read lock, polls completion records, and acknowledges/rearms the JFC.

`trs_jfs_wqe_credit_poll` checks returned statuses and logs abnormal records,
including a JFC context dump. Despite its name, this source does not implement
a task-SQ credit update or an application-visible completion result per WR.
Provider-side queue reclamation is behind the external URMA API. Task-ring
head updates and async-copy CI updates belong to other mechanisms.

Evidence: [completion-report flag](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L907),
[JFC poll and status processing](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L705),
[SQ association and JFC acknowledgement](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L766),
[polling thread and epoll registration](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L1173).

### Kernel UB receive resources carry task reports back

The host UB kernel adapter has separate UBCORE JFR/JFC resources and registered
receive buffers for task CQ data. It preposts the receive-buffer segments.
A receive JFC callback schedules `trs_ub_cq_dispatch_task`; this is distinct
from the userspace JFS/JFC used to upload SQ entries and arguments.

The tasklet's visible processing is:

1. Poll UBCORE completion records for the receive resources.
2. Take the task CQ ID from completion immediate data and the receive-buffer
   index from `user_ctx`.
3. Copy the received report into the selected host CQ through
   `trs_ub_fill_cqe`.
4. Repost the receive buffer, collect the affected CQ IDs, and invoke the
   registered CQ-update handler.
5. Rearm the receive JFC.

The CQ insertion helper updates the report's phase, checks ring capacity,
copies the CQ entry, executes a write barrier, advances the host CQ tail,
and tracks wraparound. The registered CQ-update handler connects these reports
to the channel/report machinery. The complete device-side generation and
sending of each report is not present in this trace.

This shows why two different uses of “CQE” must be distinguished: a UBCORE
receive completion identifies a buffer containing a **task report**, while
the task CQ entry inside that buffer is what the TRS report path processes.
Neither is the userspace send JFC's report for an H2D WRITE.

Evidence: [kernel receive-resource setup](../../driver/src/sdk_driver/trsdrv/trs/lba/near/sia/adapt/trs_host_ub/trs_ub_host_init.c#L626),
[receive-buffer posting](../../driver/src/sdk_driver/trsdrv/trs/lba/near/sia/adapt/trs_host_ub/trs_ub_host_init.c#L399),
[receive dispatch tasklet](../../driver/src/sdk_driver/trsdrv/trs/lba/near/sia/adapt/trs_host_ub/trs_ub_host_init.c#L465),
[CQ insertion and phase](../../driver/src/sdk_driver/trsdrv/trs/lba/near/sia/adapt/trs_host_ub/trs_ub_host_init.c#L161),
[handler registration and callback](../../driver/src/sdk_driver/trsdrv/trs/lba/near/sia/adapt/trs_host_ub/trs_ub_host_init.c#L533).

### Report receive reaches a separate kernel path

For normal CQ receive, the visible call chain is:

```text
halCqReportRecv
  -> trs_cq_report_recv
  -> TRS_SQCQ_RECV ioctl
  -> ioctl_trs_sqcq_recv
  -> trs_hw_sqcq_recv
  -> trs_chan_recv / trs_chan_recv_ex
       -> fetch valid CQ entries into caller storage
       -> advance CQ head and publish the consumed head
       -> return report_cqe_num
```

The kernel checks the process's CQ ownership. The channel fetch path tests
entry validity/phase through its configured callbacks, uses a read barrier,
and waits when no valid report is available. It copies a report to the caller
and advances the CQ head. Where configured, report processing also updates SQ
head state; the hardware-CQ callback can extract SQ ID/head and route reports
to logical CQs.

For the normal channel receive loop, the requested timeout applies while
obtaining the first report. After one is fetched, later fetches use timeout
zero, draining reports already available rather than waiting to fill the
entire requested count. A successful return can therefore contain fewer
reports than requested; `report_cqe_num` is the actual count.

`DRV_LOGIC_TYPE` dispatches to `trs_logic_cq_recv` instead. Logical-CQ matching
and waiting use their own path and must not be assigned the normal channel's
precise timeout/count behavior. Runtime's choice of normal versus logical CQ
and its interpretation of task/stream identifiers remain caller-level details.

The older split report API also remains visible: `halCqReportIrqWait` receives
reports into TRS's buffer, `halCqReportGet` returns that buffer and count, and
`halReportRelease` clears the held count after validation. Releasing those
report bytes does not itself free argument buffers, imported segments, or
async WQE tracking objects.

Evidence: [HAL receive and CQ checks](../../driver/src/ascend_hal/trs/core/trs_sqcq.c#L1795),
[kernel receive dispatch](../../driver/src/sdk_driver/trsdrv/trs/trs_core/trs_fops.c#L438),
[normal CQ ownership and channel call](../../driver/src/sdk_driver/trsdrv/trs/trs_core/trs_hw_sqcq.c#L1708),
[fetch and receive loop](../../driver/src/sdk_driver/trsdrv/trsbase/chan/chan_rxtx.c#L451),
[SQ progress and logical-CQ routing](../../driver/src/sdk_driver/trsdrv/trs/trs_core/trs_hw_sqcq.c#L315),
[logical receive](../../driver/src/sdk_driver/trsdrv/trs/trs_core/trs_logic_cq.c#L621),
[split report APIs](../../driver/src/ascend_hal/trs/core/trs_sqcq.c#L1940).

## 6. Lifetime, failure, and teardown

### Buffer lifetime follows the consumer

| Storage/resource | Consumer | Lifetime implication from this trace |
|---|---|---|
| External SQ-entry array, when TRS copies it | Host CPU during `trs_sq_task_fill` | Original array is not the subsequent URMA source; local SQ storage retains the descriptor bytes |
| Caller-filled local SQ entries | Host URMA engine, then device task queue | Ring positions must follow SQ head/tail ownership rather than be reused on send return |
| Host argument bytes | Host URMA engine after posting/activation | Posting return does not make source storage reusable |
| Device argument bytes | Task that consumes the uploaded arguments | Transfer completion alone need not end the task's use of the destination |
| Argument segment wrappers/imports | Transport work referencing their ranges | Keep appropriate segment and allocation lifetime until the dependent transfer is complete |
| Tail notification buffer | Host URMA engine | Owned by SQ resources; new-tail-indexed slots supply the WRITE sources |
| Returned task reports | Report caller | Caller interprets completion; report release only manages report-buffer state |
| Async-copy WQE objects and CI state | Async-copy caller from document 02 | Cleanup is not automatically performed by SQ upload or CQ report release |

Source segment lookup adds no allocation reference, and the argument-copy
helper has no private lifetime tracker. The runtime must connect its completion
evidence to buffer and segment reclamation. The conservative task dependency
is visible; the earliest safe recycling point for every object is not proven
by a successful posting return.

Evidence: [SQ CPU fill](../../driver/src/ascend_hal/trs/core/trs_sqcq.c#L1609),
[argument SGEs and segment release](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L1071),
[SVM lookup lifetime](01_transport_and_memory.md#managed-host-memory-and-the-trs-bridge),
[async object cleanup](02_sync_and_async_memcpy.md#5-completion-and-resource-lifetime).

### Errors identify the failed stage

| Stage | Visible outcome |
|---|---|
| SQ disabled/unavailable for sending | Status/support error before the task WRITE chain |
| Insufficient task-SQ credit after head refresh | `DRV_ERROR_NO_RESOURCES`; no task chain posted by this call |
| URMA post returns `URMA_ENOMEM` | `DRV_ERROR_NO_RESOURCES`, reported as JFS full |
| Other URMA post failure | `DRV_ERROR_INNER_ERR` |
| Transport error reported later | JFC status processing/logging; it cannot retroactively turn the original successful return into a synchronous error |
| No task report available within receive waiting rules | Receive-path timeout/error or type-specific empty result; separate from submission |

JFS PI advances only after a successful `urma_post_jfs_wr`; software SQ tail
advances only after the task posting helper succeeds. The local SQ may already
have been filled and `info->pos` assigned before a posting failure. Earlier
argument calls may also have posted work. These bookkeeping rules do not
establish atomic rollback of a WR chain or cancel an earlier argument upload.
The wrapper receives the provider's `bad_wr` pointer but does not implement
partial-chain recovery here.

Evidence: [post errors and PI update](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L912),
[task preparation before posting](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L1010),
[credit error](../../driver/src/ascend_hal/trs/core/trs_sqcq.c#L1567).

### SQ teardown is resource teardown

`trs_sqcq_urma_free` first frees/unregisters the local SQ/CQ resources and
notification storage, then destroys the host JFS/JFC and scratch array,
unimports the remote SQ/tail segments, and optionally requests remote SQ/CQ
free. It then uninitializes async contexts and releases the SQ URMA context.
The local-free failure path attempts to restore local registration before
returning an error.

The imported remote receive endpoint is shared at process/device scope:
uninitializing one SQ clears its reference, while process teardown performs
the endpoint unimport and destroys shared receive/event/context resources.
Argument handles created separately with `halGetTsegInfoByVa` remain a distinct
caller-managed lifecycle.

These helpers do not contain a per-argument or per-task completion wait before
each release. External URMA deactivation/free semantics and the caller's
quiescence policy therefore matter. A free API should not be used as evidence
that the preceding submission path synchronously waited for execution.

Evidence: [local resource release](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L570),
[SQ teardown sequence](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_urma.c#L648),
[JFS/JFC destruction](../../driver/src/ascend_hal/trs/core/urma/trs_urma.c#L189),
[process resource release](../../driver/src/ascend_hal/trs/core/urma/trs_urma.c#L432).

## 7. Related task-storage interfaces

Some task-storage writes reuse other H2D paths and should be distinguished
from the normal SQ JFS chain:

| Interface/scenario | Visible path | Relationship to this chapter |
|---|---|---|
| `halStreamTaskFill`, when the send mode is `TRS_MODE_TYPE_SQ_SEND_HIGH_PERFORMANCE` | Validate the registered stream range; synchronously `drvMemcpy` task bytes into stream memory | On a UB-backed copy this reuses P1; the fill helper does not append the normal SQ-tail WRITE or ring this SQ's JFS doorbell |
| `halStreamTaskFill`, other send mode | `TRS_STREAM_TASK_FILL` ioctl | Separate kernel dispatch; do not assume the high-performance branch was selected |
| Async create with `DRV_ASYNC_DMA_TYPE_SQE_UPDATE` | Resolve target SQ entry address; prepare device READ | P2, traced in document 02; destination is SQ storage but transfer initiation differs from P3 |
| Legacy `halSqMemGet` / `halSqMsgSend` | Obtain TRS buffer; `halSqMsgSend` calls generic `trs_sq_task_send` | Does not pass through the explicit UB dispatch in `halSqTaskSend`; the API names alone do not justify substituting the P3 call chain |

The stream-fill public wrapper uses the device stream mutex. Stream switching
updates the bound SQ's depth and stream-memory resource information in the
high-performance mode. The normal UB submission sequence above should not be
used to infer the complete launch behavior of such a stream-binding workflow.

Evidence: [public stream fill and locking](../../driver/src/ascend_hal/trs/core/trs_interface.c#L209),
[stream binding and conditional copy](../../driver/src/ascend_hal/trs/core/trs_sqcq.c#L1703),
[async SQE update](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L679),
[legacy buffer/send wrappers](../../driver/src/ascend_hal/trs/core/trs_sqcq.c#L1842).

## 8. Worked examples and remaining evidence

These are illustrations of source arithmetic and control flow, not supported
hardware geometry claims or performance measurements.

| Example | Result |
|---|---|
| SQ depth 8, tail 2, send 3 entries, sufficient credit | Upload entries 2–4 in one WRITE, then publish tail 5; JFS PI advances by 2 and SQ tail by 3 |
| Same depth, tail 6, send 3 entries | Upload entries 6–7, then entry 0, then publish tail 1; JFS PI advances by 3 |
| Same depth, tail 6, send 2 entries | One contiguous SQ WRITE ending at the ring boundary, then tail 0; two WRs |
| 300 MiB argument upload followed by a nonwrapping task send | Argument call posts 256 MiB + 44 MiB without a doorbell; task call posts SQ data + tail and rings the accumulated PI, covering four WRs |
| SQ has room but URMA reports JFS full | Send returns `NO_RESOURCES`; free task entries do not imply transport capacity |
| Normal CQ receive asks for eight entries and three are available after its first wait | It can return three reports successfully; the caller must use the returned count |

The structural costs are CPU copying into the local SQ when necessary,
argument WR construction, the number of SQ batches, an extra WR on ring wrap,
registration/import amortized over resource lifetime, provider queue pressure,
and report handling. Sharing one activation between arguments and task
submission can reduce doorbell stores. These are source-derived cost
categories; this analysis does not measure throughput, latency, or overlap.

[Document 06](06_acl_runtime_dispatch.md) connects the ordinary runtime SQE
format, argument/task serialization, stream waiting, and resource recycling.
[Document 08](08_graph_and_software_sq_lifecycle.md) traces the software-SQ
capture-model lifecycle and compares the separate auto-split upload path.
Host ordering flags describe the intended transfer order, while final device
scheduling and report generation need the corresponding implementation or
interface contract.

The next chapter in the series is
[04 — Queue/TDT and HDC](04_queue_and_hdc.md), comparing receiver-pull queue
payload delivery with buffered HDC SEND and their control/completion paths.
