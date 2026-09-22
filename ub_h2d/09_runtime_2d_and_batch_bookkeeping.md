# 09 — Runtime 2D and batch memcpy bookkeeping over UB

[Summary](README.md) · Previous: [Graph and software-SQ lifecycle](08_graph_and_software_sq_lifecycle.md) · Next: [Batch correctness review](10_batch_correctness_review.md)

Analysis date: 2026-09-22. Source snapshots: `runtime` at `216912472` and
`driver` at `866e409`. This chapter connects the 950/v200 runtime's H2D/D2H
2D and batch APIs to the HAL mechanisms in [02](02_sync_and_async_memcpy.md)
and the software-SQ capture lifecycle in
[08](08_graph_and_software_sq_lifecycle.md). These are source observations;
the checkouts have not been verified as an installed pair, and no NPU test
was run.

Here an ordinary stream means an unbound stream outside capture. Non-software
task-sink and auto-split model variants retain their own lifetime rules.

**A runtime API call can prepare several memcpy tasks. Its progress counters
describe prepared work, not completed payload transfers. Ordinary UB streams
may synchronize between portions; capture records the portions for later
execution.** The trace also identifies two batch continuation concerns in
this snapshot, detailed in section 8. Connecting the source path does not
establish that every partial-progress case is correct.
[The follow-up review](10_batch_correctness_review.md) confirms both host-code
defects and identifies a third issue in empty-preparation credit ownership.

## 1. The counters have different units

| Field / context | Meaning in the selected path |
|---|---|
| 2D `width × height` | Total payload bytes, excluding gaps between rows |
| 2D `fixedSize`, returned runtime `realSize` | Cumulative prepared payload bytes from the original matrix start |
| Batch `count` / `remainCnt` | Number of entries still represented by the current local arrays |
| Batch `fixedCnt`, returned `realCnt` | Whole entries prepared by the preceding/current conversion call; not the total since API entry |
| Captured batch `fixedSize`, returned `realSize` | Bytes prepared within the next entry, after the completed entries |
| `MemcpyAsyncTaskInfo::size` | Overloaded: cumulative bytes for UB 2D; whole-entry count for UB batch |
| Ordinary 2D/batch `ubDma.pi` | HAL's absolute producer index for the prepared queue prefix |
| Captured task's patched `ubDma.pi` | Number of WQEs belonging to that task; patched during graph binding |
| WQE conversion `wqeCnt` | Descriptor count; neither an entry count nor a completion count |
| Public batch `failIndex` | Error-location output where a layer supplies one; not a progress/completion counter |

The HAL create API aliases `fixedSize` and `fixedCnt` in a union. Its
standalone WQE conversion API returns them as distinct fields. Runtime
must preserve that difference: batch **create** prepares whole entries;
batch **conversion** can also prepare part of the next entry.

Evidence: [runtime batch cursor](../../runtime/src/runtime/core/inc/drv/driver_types.hpp#L113),
[HAL create output](../../driver/pkg_inc/ascend_hal_define.h#L1352),
[HAL preparation arithmetic](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L555),
[conversion output](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L1599).

## 2. ACL/runtime dispatch and selection

The v200 build uses `ApiImplDavid`, `memcpy_starsv2.cc`, and the shared
`memory_task.cc` 2D converter. The similarly named arch5162 implementation
is not the implementation used for this trace.

```text
aclrtMemcpy2dAsync
  -> aclrtMemcpy2dAsyncImpl -> rtMemcpy2dAsync
  -> ApiErrorDecorator::MemCopy2DAsync
  -> ApiImplDavid::MemCopy2DAsync
  -> Memcpy2DAsync                         [one preparation/submission attempt]

aclrtMemcpyBatchAsync / aclrtMemcpyBatchAsyncV2
  -> MemcpyBatchImpl -> rtsMemcpyBatchAsync
  -> ApiErrorDecorator::MemcpyBatchAsync
  -> ApiImplDavid::MemcpyBatchAsync
  -> BatchMemcpyAsync                      [selected UB fallback]
  -> MemcopyBatchAsync                     [one preparation/submission attempt]
```

Evidence: [ACL 2D entry](../../runtime/src/acl/aclrt_impl/memory.cpp#L1018),
[runtime 2D entry](../../runtime/src/runtime/api/api_c.cc#L1293),
[shared ACL batch dispatch](../../runtime/src/acl/aclrt_impl/memory.cpp#L1735),
[batch API variants](../../runtime/src/acl/aclrt_impl/memory.cpp#L1837),
[runtime batch feature gate](../../runtime/src/runtime/api/api_c_memory.cc#L765),
[v200 API implementation](../../runtime/src/runtime/api/impl/v200/api_impl_creator_c.cc#L17),
[v200 source selection](../../runtime/src/runtime/cmake/v200.cmake#L55).

### Validation and pageable-memory fallback

For nonzero 2D work, ACL checks pointers and `width <= spitch, dpitch`.
Runtime performs its geometry and memory-location checks and can revise the
copy direction. If an H2D/D2H operation involves unregistered pageable host
memory, the decorator rejects capture; outside capture it synchronizes the
stream and calls synchronous 2D copy. A zero width or height returns success
at the runtime C entry without task creation.

For batch work, the selected UB path validates **all entries before creating
the first batch task**. Attribute-index ranges select each entry's memory
attributes. Runtime verifies actual pointer locations/device IDs and permits
H2D/D2H pairs in this path. If any entry involves unregistered memory, runtime
rejects a capturing stream. On an ordinary stream, that same memory condition
converts the whole batch to stream synchronization followed by synchronous
`MemcpyBatch`.

The public ACL batch helper treats an all-zero-size batch as a successful
no-op and sets a supplied `failIndex` to `SIZE_MAX`. A zero-size entry mixed
with nonzero entries reaches ordinary validation; this UB runtime path
rejects it. The all-zero shortcut is not a promise that arbitrary zero-size
entries are filtered from a nonempty batch.

Evidence: [ACL geometry checks](../../runtime/src/acl/aclrt_impl/memory.cpp#L211),
[2D memory classification and fallback](../../runtime/src/runtime/api/impl/api_error.cc#L3057),
[batch prevalidation and fallback](../../runtime/src/runtime/api/impl/api_impl_david.cc#L806),
[per-entry validation](../../runtime/src/runtime/api/impl/api_impl.cc#L8850),
[batch attribute-index checks](../../runtime/src/runtime/api/impl/api_error.cc#L7342).

### The batch feature branch is explicit

`rtsMemcpyBatchAsync` first checks the runtime chip feature
`RT_FEATURE_TASK_MEMORY_BATCH_COPY`. Inside `ApiImplDavid::MemcpyBatchAsync`,
the branch is:

| Driver `FEATURE_MEMCPY_BATCH_ASYNC` query | Connection | Selected implementation |
|---|---|---|
| False | UB | `BatchMemcpyAsync`, the grouped preparation loop below |
| False | Non-UB | `LoopMemcpyAsync`, validating/submitting entries individually |
| True | Either | Returns `RT_ERROR_DRV_NOT_SUPPORT` in this snapshot |

The false-query UB branch is an implemented fallback. Do not reverse this
selector based on the feature's name or assume that every installed driver
will select it.

Evidence: [David selector](../../runtime/src/runtime/api/impl/api_impl_david.cc#L881),
[per-entry fallback](../../runtime/src/runtime/api/impl/api_impl.cc#L8904).

### Runtime validation is not the final HAL limit

Both `halAsyncDmaCreateBatch` and batch `halAsyncDmaWqeConvert` validate an
input count of at most **2048 entries**. The runtime UB loop passes the full
remaining count; it does not first clamp that count to 2048. Its later
partial-progress loop therefore does not automatically make an oversized
initial batch acceptable to these HAL APIs.

The HAL also requires the sources to share an owner and the destinations to
share an owner, and checks the executing device. Per-entry runtime validation
does not make mixed H2D/D2H or mixed-device entries a single supported UB
batch. The standalone converter and create path both derive the actual
direction from address attributes.

For 2D, the shared HAL geometry validator additionally limits height to
`5 × 1024 × 1024` and checks its cumulative cursor against the payload size.
These are snapshot limits, separate from WQE credit and graph queue depth.

Evidence: [batch count limit](../../driver/src/ascend_hal/trs/core/trs_interface.c#L710),
[conversion uses the same validator](../../driver/src/ascend_hal/trs/core/trs_interface.c#L925),
[owner consistency](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L214),
[direction derivation](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L256),
[2D geometry limits](../../driver/src/ascend_hal/trs/core/trs_interface.c#L604).

## 3. UB 2D continuation uses an absolute payload cursor

For H2D/D2H, `ApiImplDavid::MemCopy2DAsync` keeps the original pointers,
pitches, width, and height for every attempt. Its relevant loop is:

```text
total = width * height
fixedSize = 0
while fixedSize < total:
    prepare/submit a 2D task using original geometry and fixedSize
    fixedSize = returned realSize
    if fixedSize < total and the stream is not capturing:
        synchronize the stream
```

`fixedSize` is assigned, not incremented by the returned value. HAL resumes
at row `fixedSize / width`, byte `fixedSize % width`, applying `spitch` and
`dpitch` to the corresponding row bases. Pitch gaps consume address space
but do not contribute to this progress counter.

`Memcpy2DAsync` locks the user stream, allocates a task through
`AllocTaskInfoForCapture`, and initializes it on the selected destination
stream. `MemcpyAsyncTaskInitV2` selects the copy type and invokes
`ConvertAsyncDma2D` for UB DMA. The converter has two paths:

| Destination stream | Preparation | Progress returned to runtime |
|---|---|---|
| Ordinary, software SQ disabled | `CreateAsyncDmaWqe2D -> halAsyncDmaCreate2D` | HAL cumulative `fixedSize`; also retain jetty identity and PI |
| Software-SQ capture | `ConvertAsyncDma2DForSoftWareSq -> StreamJettyHandler -> halAsyncDmaWqeConvert` | If `fixedCnt == 1`, normalize to `width × height`; otherwise use cumulative `fixedSize` |

The normalization matters because the standalone converter returns
`fixedSize = 0` on full completion. Passing that zero directly to the outer
loop would lose the completion indication.

Evidence: [outer 2D loop](../../runtime/src/runtime/api/impl/api_impl_david.cc#L749),
[task allocation and submission](../../runtime/src/runtime/core/src/launch/memcpy_starsv2.cc#L79),
[copy-type initialization](../../runtime/src/runtime/core/src/task/task_info/memory/memory_memcpy_async_task.cc#L458),
[ordinary/captured converters](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task.cc#L37),
[HAL wrapper](../../runtime/src/runtime/driver/npu_driver_standard_soc.cc#L100).

If ordinary UB preparation leaves the cumulative size unchanged, the task
helper returns without `DavidSendTask`. Its scope guard cleans up the
temporary task, rolls back the task slot, and unlocks the stream. The caller's
`realSize` retains its preceding value, and the outer loop synchronizes
before another attempt. This is a retry/progress path, not full-copy success.

The outer loop adds no local retry budget for unchanged progress. Eventual
progress depends on reclaimable queue resources and successful stream waits.
There is no unconditional final synchronization when the last portion has
been prepared and submitted.

## 4. Batch continuation compacts private metadata arrays

After validation, `BatchMemcpyAsync` copies the caller's destination,
source, and size arrays into three local vectors. It changes these metadata
copies during continuation; it does not copy the user payload into those
vectors.

The outer state is:

```text
remainCnt = original count
fixedCnt = 0; fixedSize = 0
while remainCnt > 0:
    batchInfo = {local arrays, remainCnt, fixedCnt, fixedSize}
    prepare/submit one task -> realCnt, realSize
    fixedCnt = realCnt
    fixedSize = realSize
    remainCnt -= fixedCnt
    if remainCnt > 0 and the stream is not capturing:
        synchronize the stream
```

Here `fixedCnt` carries the **preceding call's** whole-entry count into the
next preparation attempt. It is not cumulative. `ShiftBatchArrays` moves
the remaining `count` entries left by that prefix length before the next
conversion. The vectors remain local to the API call; payload allocations
and their registrations must outlive execution of the submitted work.

Evidence: [local vectors and loop](../../runtime/src/runtime/api/impl/api_impl_david.cc#L860),
[array compaction](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L206).

### Ordinary UB batch: whole entries per preparation

`ConvertAsyncDmaBatch` calls `ShiftBatchArrays(batchInfo)` and passes the
compacted arrays to `CreateAsyncDmaWqeBatch -> halAsyncDmaCreateBatch`.
The HAL input is zero-initialized; runtime does not also pass the old prefix
as a nonzero HAL `fixedCnt`. The arrays already represent the remaining work.

HAL fits whole entries into available WQE credit. An entry may require more
than one WQE; its whole-entry count is still one. If the next whole entry
does not fit, create leaves it for a later attempt instead of partially
preparing it. Returned `fixedCnt` becomes the task's `size` and then runtime's
`realCnt`. This ordinary branch does not use the captured batch's byte-offset
continuation mechanism.

Evidence: [ordinary batch conversion](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L273),
[HAL batch wrapper](../../runtime/src/runtime/driver/npu_driver_standard_soc.cc#L162),
[whole-entry credit calculation](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L566).

### Captured UB batch: whole entries plus a partial next entry

`ConvertAsyncDmaBatchForSoftWareSq` calls
`ShiftBatchArrays(batchInfo, true)`, then passes the remaining arrays to
`StreamJettyHandler` using `DRV_ASYNC_DMA_TYPE_BATCH`.

For a preceding result with a nonzero completed prefix and a nonzero byte
offset, the shift helper removes the completed entries, advances the new
first source and destination by `fixedSize`, and reduces its remaining
length by the same amount. The converter returns a new whole-entry count
and an optional partial-byte count relative to these adjusted arrays.

The runtime records each generated WQE portion with its persistent task.
The host WQE buffer supplies only the unused space in its current 2048-entry
chunk to a conversion call; later attempts can allocate another chunk.
Graph capture end, jetty binding, upload, replay, and destruction then follow
document 08. No intermediate stream synchronization executes these captured
copies while they are being recorded.

The zero-whole-entry case has a different outcome in the current shift
helper and is an unresolved concern in section 8.

Evidence: [captured batch conversion](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L237),
[WQE chunk capacity and task records](../../runtime/src/runtime/feature/jetty/stream_jetty_handler.cc#L98),
[graph buffer growth](../../runtime/src/runtime/feature/jetty/stream_jetty_context.cc#L56).

## 5. One task SQE can represent many payload WQEs

Both helpers allocate one SQE for the UB 2D/batch method. They use
`TS_TASK_TYPE_MEMCPY` with `copyMethod = RT_ASYNC_CPY_2D` or
`RT_ASYNC_CPY_BATCH`. Successful UB conversion sets
`dmaKernelConvertFlag`, enabling the corresponding cleanup path.

`ConstructDavidSqeForMemcpyAsyncTask` selects a UBDMA doorbell-mode SQE for
these methods even on an ordinary stream. That SQE carries the jetty
identity and saved PI. This differs from the ordinary normal-copy path
that uses an ASYNCDMA SQE plus an inline direct WQE.

The shared doorbell constructor uses `RT_UBDMA_SOURCE_MODEL_ASYNC` for
2D/batch too; the source constant's name alone does not prove graph capture.
The ordinary HAL create result supplies an absolute PI, whereas software-SQ
graph binding later patches each task with its WQE count as described in 08.

After initialization, the helpers call `DavidSendTask`, release their
rollback guard, unlock, update thread task/stream identifiers, and perform
submission postprocessing. Capture redirects task ownership and retains
SQEs in host storage. Ordinary submission reaches the normal task-SQ send
path. Submission postprocessing can recycle earlier work or wake the recycle
thread; these calls do not request an unconditional wait for the new task.

Evidence: [one-SQE sizing](../../runtime/src/runtime/core/src/task/task_info/memory/memory_memcpy_async_task.cc#L338),
[2D/batch task helpers](../../runtime/src/runtime/core/src/launch/memcpy_starsv2.cc#L79),
[SQE selector](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200.cc#L170),
[doorbell fields](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200.cc#L113),
[submission postprocessing](../../runtime/src/runtime/core/src/task/task_submit/v200/task_david.cc#L670).

## 6. Completion returns credits through task cleanup

Ordinary 2D and batch preparation share the SQ's `batch_2d_async_ctx`.
For their remote WQE mode, HAL computes available credit from PI/CI modulo
the standard jetty depth. Preparing work advances PI; it does not advance
CI to assert that execution has finished.

```text
HAL prepares WQEs, returns PI and payload/entry progress
  -> runtime saves PI in the memcpy task
  -> submit that task's UBDMA doorbell SQE
  -> stream wait / normal task-report and SQ-progress reclamation
  -> TaskUnInitProc -> StarsV2MemcpyAsyncTaskUnInit
  -> AsyncDmaWqeProc
       2D:    AsyncDmaWqe2DProc    -> DestroyAsyncDmaWqe2D
       batch: AsyncDmaWqeBatchProc -> DestroyAsyncDmaWqeBatch
  -> halAsyncDmaDestroy2D / halAsyncDmaDestroyBatch
  -> batch_2d_async_ctx.ci = saved task PI
```

This closes the caller-side credit bookkeeping left open by the HAL trace.
The HAL destroy calls validate the CI and store it; they do not themselves
wait for device completion. Runtime associates that credit update with its
task lifecycle. WQE credit, task SQ credit, batch entry progress, and matrix
byte progress are four different quantities.
The follow-up review identifies an exception to correct ownership in
[empty-preparation cleanup](10_batch_correctness_review.md#5-b3--an-empty-preparation-does-not-own-the-queues-current-pi).

Evidence: [shared context selection](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L133),
[credit and PI update](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L545),
[stream wait selection](../../runtime/src/runtime/core/src/stream/stream.cc#L2054),
[completed-task cleanup](../../runtime/src/runtime/core/src/task/task_recycle/v200/task_recycle_common_base.cc#L169),
[task cleanup registration](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200.cc#L210),
[method-specific cleanup](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L336),
[HAL CI assignment](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L1170).

A successful intermediate stream wait is not an unconditional guarantee
that every WQE credit has already been returned. With separate submission
and recycling, `SynchronizeImpl` can finish after observing execution and
wake the recycle thread when no task requires a reclamation wait. CI advances
when task cleanup actually runs. This distinction explains why preparation
may still encounter no available credit immediately after a successful wait.
[Execution versus reclamation wait](../../runtime/src/runtime/core/src/stream/stream.cc#L2030)

For software-SQ capture, `AsyncDmaWqeProc` returns before these per-task
destroy calls. Persistent task objects and host WQE chunks survive successful
replay; graph-level reclamation/destruction owns their lifetime. Replaying
the graph still performs the payload transfers again.

On a preparation/submission failure, the task helper's active scope guard
also calls uninitialization and task-slot rollback. This is an error unwind,
not an observed successful device completion. Previously submitted portions
are not rolled back as one atomic API transaction. A postprocessing error
occurs after the guard has been released and the task submitted. Neither an
API failure nor a stream timeout proves remote quiescence.

Evidence: [software-SQ cleanup bypass](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L404),
[task submission guards](../../runtime/src/runtime/core/src/launch/memcpy_starsv2.cc#L79),
[graph ownership](08_graph_and_software_sq_lifecycle.md#9-destruction-and-ownership).

## 7. Synchronous and non-UB branches retain their own behavior

| Branch | Runtime behavior |
|---|---|
| Synchronous 2D, including pageable async fallback | `ApiImpl::MemCopy2DSync` invokes driver `MemCopy2D` with `DEVMM_MEMCPY2D_SYNC`; no runtime async-task progress loop |
| Synchronous batch, including whole-batch pageable fallback | Validate attributes/directions, then `NpuDriver::MemcpyBatch`; no runtime batch task or per-task jetty CI |
| Non-UB async batch fallback | `LoopMemcpyAsync` processes entries individually; eligible entries call ordinary `MemcpyAsync`, unregistered entries synchronize and use synchronous copy |
| Non-UB H2D/D2H async 2D | Descriptor conversion through driver `MemCopy2D(...ASYNC_CONVERT...)`; the outer loop accumulates per-attempt byte sizes |
| D2D row path using SDMA/HCCS/PCIe copy types | Task initialization sets one row's width as task size; outer loop advances source/destination by their pitches |

The main source trace here is UB H2D/D2H. Cross-device D2D geometry and
device execution should not inherit the H2D cumulative-cursor interpretation
without their own selectors. A generic D2D API branch is not evidence that
the dedicated `halAsyncDmaCreate2D` accepts every D2D topology.

Evidence: [synchronous 2D](../../runtime/src/runtime/api/impl/api_impl.cc#L6785),
[synchronous batch](../../runtime/src/runtime/api/impl/api_impl.cc#L8777),
[non-UB per-entry batch](../../runtime/src/runtime/api/impl/api_impl.cc#L8904),
[2D copy-type branches](../../runtime/src/runtime/core/src/task/task_info/memory/memory_memcpy_async_task.cc#L458),
[outer-loop cursor branches](../../runtime/src/runtime/api/impl/api_impl_david.cc#L749).

## 8. Bookkeeping concerns and error-location limits

These initial concerns have now been reviewed against the surrounding
contracts in [10](10_batch_correctness_review.md). Both progress defects
are confirmed at source level; correction and behavioral validation remain.

### Ordinary batch compares a per-call count with the preceding count

The no-progress guard in `MemcopyBatchAsync` compares
`batchInfo.fixedCnt == taskAsyncBatch->u.memcpyAsyncTaskInfo.size` outside
capture. The left side is the preceding attempt's whole-entry count; the
right side is the newly returned whole-entry count. HAL resets its batch
count for each preparation call, and array compaction does not reset the
left-side field.

Consequently, equality is not a reliable test for zero new work: successive
positive counts can be equal. The equality branch returns before
`DavidSendTask` and before writing `realCnt/realSize`, while its scope guard
runs cleanup and rollback. This is a source-level contract mismatch requiring
correctness review of submission, cursor updates, and credit release.

The adjacent 2D equality check operates on cumulative byte positions, so its
meaning cannot be copied directly into the batch path. No hardware symptom
or complete failure-recovery behavior has been established here.

Evidence: [batch equality guard](../../runtime/src/runtime/core/src/launch/memcpy_starsv2.cc#L162),
[preceding count passed by the outer loop](../../runtime/src/runtime/api/impl/api_impl_david.cc#L864),
[HAL starts each batch count at zero](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L566).

### Captured batch skips byte-only continuation when `fixedCnt == 0`

The standalone converter can return zero completed entries and a nonzero
partial-byte count. `ShiftBatchArrays`, however, returns immediately when
`batchInfo.fixedCnt == 0`, before reaching its `handleFixedSize` adjustment.
Thus the software-SQ caller's requested byte-offset adjustment is skipped
in that case even though conversion can have prepared payload WQEs.

This leaves the same entry start and size for the next conversion instead
of consuming the reported partial prefix. The resulting progress and graph
descriptor sequence need correctness review. A repair should preserve the
byte cursor independently of whether a whole entry was completed; this
documentation change does not modify runtime behavior.

Evidence: [early return and byte adjustment](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L206),
[captured caller enables byte adjustment](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L250),
[HAL partial-entry output](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L1555).

### `failIndex` does not identify every later failure

The runtime decorator initializes a supplied `failIndex` to `SIZE_MAX`.
Selected per-entry validation failures overwrite it with the original entry
index. The later UB preparation/task-submission loop returns errors without
mapping them back to an original entry index, and its task structure does
not retain the public `failIndex` pointer for asynchronous completion.

The create wrappers also map HAL preparation failures to `RT_ERROR_DRV_ERR`.
Therefore, `failIndex == SIZE_MAX` is not evidence that the batch executed
successfully, or that all earlier portions were undone. Inspect the API error
and subsequent stream completion/error result separately.

Evidence: [initial error index](../../runtime/src/runtime/api/impl/api_error.cc#L7342),
[validation index and later loop returns](../../runtime/src/runtime/api/impl/api_impl_david.cc#L836),
[create error mapping](../../runtime/src/runtime/driver/npu_driver_standard_soc.cc#L162).

## 9. Worked bookkeeping examples and validation

These examples check units and cursor arithmetic; they are not device
workloads or predictions of how much credit a live queue will expose.

| Example | Correct interpretation of the reported progress |
|---|---|
| 2D width 8, height 3, both pitches 16; returned prefixes 8 then 24 | Payload total is 24 bytes; resume the second attempt at row 1. Assign the second result 24, rather than adding it to 8. Each touched memory span is 40 bytes. |
| Batch entries A–E; whole-entry results 2, 1, 2 | Remaining counts are 3, 2, 0. Compacted arrays start at C, then D. Results are counts for each call, not cumulative totals. |
| Captured conversion completes A and 16 bytes of B, where B has length 64 | Shift past A, advance B's source/destination by 16, and reduce B's next length to 48. A byte-only result without a completed entry must also advance the cursor; section 8 records the current gap. |
| Final ordinary preparation consumes all remaining payload/entries | No outer-loop intermediate wait remains; API return does not itself establish completion of that final submitted task. |
| Batch input exceeds 2048 entries | The first selected HAL create/conversion rejects the input count; the runtime continuation loop does not pre-split it to fit. |

The source trace now connects API selection, private metadata, progress
units, task SQEs, intermediate waits, completion-driven CI return, and graph
ownership for the selected UB paths. The remaining correctness work is the
implementation and validation of the three findings in
[10](10_batch_correctness_review.md), progress under sustained resource
pressure, and error/recovery behavior. Firmware scheduling, coherency, and performance
still require separate implementation/interface or hardware evidence.

Validation consists of source inspection, local Markdown link/line-anchor
checks, and the example arithmetic. Existing mocked tests cover selected
API routes, task initialization failures, and conversion outputs. Some of
those tests stub out the task helper or converter; they do not establish
correctness of the complete continuation loop. They were inspected, not run.

Evidence: [mocked API routes](../../runtime/tests/ut/runtime/runtime/test/platform/950/rt_utest_api_david.cc#L10407),
[task helper tests](../../runtime/tests/ut/runtime/runtime/test/platform/950/rt_utest_david_task.cc#L488),
[mocked batch conversion](../../runtime/tests/ut/runtime/runtime/test/platform/950/rt_utest_david_task.cc#L2167),
[software-SQ 2D conversion test](../../runtime/tests/ut/runtime/runtime/test/platform/950/rt_utest_aclgraph_ub.cc#L2294).
