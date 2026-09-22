# 08 — Graph and software-SQ lifecycle over UB

[Summary](README.md) · Previous: [Mapped host memory and KV cache](07_host_memory_access_and_kv_cache.md) · Next: [Runtime 2D and batch bookkeeping](09_runtime_2d_and_batch_bookkeeping.md)

Analysis date: 2026-09-22. Source snapshots: `runtime` at `216912472` and
`driver` at `866e409`. This chapter connects capture, finalization, first
execution, replay, updates, resource reclamation, and destruction for the
950/v200 **UB-connected software-SQ capture-model path**. It also identifies
the separate auto-split path. These are source-level findings, not measured
device behavior or confirmation of an installed runtime/driver pair.

The main example is an eligible H2D async copy captured alongside compute
tasks. The host allocation must satisfy the memory/segment requirements in
[06](06_acl_runtime_dispatch.md#4-async-memory-classification-comes-before-task-creation).
Pageable-memory fallback and explicit host staging must keep their own
selectors. Runtime 2D/batch progress is traced separately in
[09](09_runtime_2d_and_batch_bookkeeping.md). Nested conditional graphs and
recovery after device failure are not silently included in this example.

## 1. What changes from an ordinary stream

**Software-SQ capture records task SQEs and UB WQEs in separate host buffers.
First execution binds device resources and uploads those buffers. Replay
can reuse the uploaded descriptors, but still performs the recorded payload
transfers.**

| Stage | Task SQEs | UB WQEs / jetty | User H2D payload |
|---|---|---|---|
| Capture | Constructed and retained in the capture stream's host SQE buffer | Converted into host WQE chunks; standard jetties can be preallocated into a pool | Recording this copy does not execute it |
| Capture end | Capture bookkeeping and external-event finalization | Pad unused WQE slots with NOPs; round capacity | No execution of the recorded copy |
| First execution | Allocate/bind SQ/CQ, patch SQEs, fill device SQ, configure tail | Bind a jetty, upload all WQE chunks, record jetty identity and padding count | Device executes the recorded H2D READ work |
| Replay with resources retained | Reuse recorded SQ storage and submit another model-execution task | Submit the padding-count prelude; reuse uploaded WQEs | Payload is transferred again |
| Idle resource reclamation | SQ/CQ identities can be returned separately | Standard jetty can return to the pool; host WQEs remain | No new copy requested by the reclamation operation |
| Destruction | Recycle persistent tasks and release stream storage | Release remaining bindings and host WQE contexts | Application allocations are not owned by this descriptor machinery |

Capture is not necessarily free of device traffic: resource preallocation,
argument loading, registration, and external-event setup can perform work.
The deferred action here is execution of the captured memcpy payload.

## 2. Entry points and feature gates

The ACL entries are `aclmdlRICaptureBegin`, `aclmdlRICaptureEnd`,
`aclmdlRIExecuteAsync`, and `aclmdlRIDestroy`. In this implementation,
`aclmdlRIExecuteAsyncImpl` calls `rtModelExecute`, which reaches the model's
virtual `Execute` method. The async public name does not require the internal
method itself to be named `ExecuteAsync`.

Evidence: [ACL entries](../../runtime/src/acl/aclrt_impl/model_ri.cpp#L26),
[capture end entry](../../runtime/src/acl/aclrt_impl/model_ri.cpp#L93),
[runtime model dispatch](../../runtime/src/runtime/api/impl/api_impl.cc#L4430).

`Context::StreamBeginCapture` enables software SQ only when all three checks
succeed:

- Runtime device feature `RT_FEATURE_MODEL_ACL_GRAPH_SOFTWARE_ENABLE`.
- Firmware feature `TS_FEATURE_SOFTWARE_SQ_ENABLE`.
- Driver feature `FEATURE_TRSDRV_SQ_SUPPORT_DYNAMIC_BIND`.

It creates an internal persistent capture stream. In software-SQ mode the
stream is added to the model; the non-software path uses its earlier binding
flow. User submissions are redirected to that capture stream.

Evidence: [three feature gates](../../runtime/src/runtime/feature/aclgraph/context_aclgraph.cc#L309),
[internal stream and association](../../runtime/src/runtime/feature/aclgraph/context_aclgraph.cc#L178),
[persistent flag](../../runtime/src/runtime/feature/aclgraph/v200/capture_adapt_v200.cc#L35).
The v200 build includes the real jetty implementation and v200 capture
adapter; the v100 adapter and jetty stubs are not interchangeable evidence.
[Build selection](../../runtime/src/runtime/cmake/v200.cmake#L163)

## 3. Capture retains two different descriptor buffers

### UB WQEs

```text
Eligible captured async H2D task
  -> ConvertAsyncDma
  -> ConvertAsyncDmaForSoftWareSqUb
  -> StreamJettyHandler::HandleUbDmaTask
  -> GetOrCreateStreamJettyContext
  -> CreateAndAppendWqe
  -> NpuDriver::AsyncDmaWqeConvert
  -> halAsyncDmaWqeConvert
```

The driver converts source/destination addresses into WQEs in caller-owned
host storage. H2D uses the device-READ preparation described in document 02.
This conversion is not WQE submission and not payload completion.

A `StreamJettyContext` is keyed by `(stream ID, jetty type)` and records:

| Field | Role |
|---|---|
| `wqeBuffers` | Host allocations, each holding 2048 WQEs × 64 bytes = 128 KiB |
| `filledWqeCount` | Number of valid converted WQEs, excluding padding |
| `capacity` | Total allocated WQE slots, later rounded to supported depth |
| `taskWqeCounts` | Pairs of persistent `TaskInfo*` and that task's WQE count |
| `jettyHandle` | Current device-jetty binding; initially zero |
| `isLargeDepth` | Selects dedicated large-depth allocation rather than standard pool reuse |

H2D and D2H share the `JETTY_TYPE_H2D` resource family. Cross-device UB copies
use `JETTY_TYPE_D2D`; that name does not change the H2D focus of this chapter.

Context creation calls `PreAllocJetty`, which can create a standard-depth
device jetty in the pool's FREE state. It does not assign that jetty handle
to this stream yet. WQE buffers use `HostMemAlloc` and grow in 2048-entry
chunks up to the source limit of 32768 entries per context.

Evidence: [conversion selector](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L868),
[append and per-task counts](../../runtime/src/runtime/feature/jetty/stream_jetty_handler.cc#L98),
[context layout](../../runtime/src/runtime/feature/jetty/stream_jetty_context.h#L24),
[host allocation and growth](../../runtime/src/runtime/feature/jetty/stream_jetty_context.cc#L34),
[context creation and preallocation](../../runtime/src/runtime/feature/jetty/jetty_manager.cc#L174),
[driver conversion wrapper](../../runtime/src/runtime/driver/npu_driver_standard_soc.cc#L800).

### Task SQEs and task objects

`DavidSendTask` constructs task SQEs and updates host task bookkeeping. Its
software-SQ branch copies the SQEs into `stream->GetSqeBuffer()` at
`task->pos × SQE_SIZE_UNIT`, then returns before ordinary `halSqTaskSend`.
The graph therefore does not submit each recorded task through the ordinary
stream's immediate SQ-write path.

Bound/persistent tasks are placed in `delayRecycleTaskid_`. They must remain
available for execution, updates, and jetty patching; successful execution
does not recycle every captured task object.

Evidence: [deferred SQE storage](../../runtime/src/runtime/core/src/task/task_submit/v200/task_david.cc#L545),
[persistent task bookkeeping](../../runtime/src/runtime/core/src/stream/stream_david.cc#L781).

## 4. Capture end pads the WQE queue

For software SQ on UB, `Context::StreamEndCapture` calls
`FillNopWqeOnCaptureEnd` for both jetty types on each capture stream. It first
fills the unused part of the current allocation with NOP WQEs, then rounds
larger capacities upward to a power of two and fills any added chunks with
NOPs. The valid count is not increased by this padding.

Let `V = filledWqeCount`, `C = final capacity`, and `P = C - V`:

| Valid WQEs V | Chunk capacity before rounding | Final C | NOP entries P | Uploaded WQE bytes |
|---|---|---|---|---|
| 3 | 2048 | 2048 | 2045 | 128 KiB |
| 4500 | 6144 | 8192 | 3692 | 512 KiB |

These examples follow buffer arithmetic; they are not throughput estimates.
With zero WQEs, the fill helper returns without creating padding work.

Unlike the non-software capture branch, this branch does not call the same
immediate `ModelEndGraph`/`LoadComplete` pair at capture end. The software
model prepares its execution resources later in `BuildSqCq`.
`EndCaptureAdapterProc` separately finalizes the external-event refresh table.

Evidence: [capture-end branches](../../runtime/src/runtime/feature/aclgraph/context_aclgraph.cc#L605),
[partial-buffer NOPs](../../runtime/src/runtime/feature/jetty/stream_jetty_handler.cc#L171),
[capacity rounding](../../runtime/src/runtime/feature/jetty/stream_jetty_context.cc#L79),
[external-event finalization](../../runtime/src/runtime/feature/aclgraph/capture_model.cc#L1977).

## 5. First execution binds, uploads, and patches

```text
aclmdlRIExecuteAsync(model, executionStream)
  -> rtModelExecute -> ApiImpl::ModelExecute
  -> CaptureModel::Execute -> ExecuteCommon
  -> PreModelExecute
       -> SetNotifyBeforeExecute
       -> BuildSqCq
            -> BindJettyForUbdma
                 -> bind each required stream jetty
                 -> FillWqeToDevice
                 -> UpdateUbdmaSqeWithJettyInfo
                 -> RefreshJettyInfoList
            -> prepare end-graph/load-complete notification resources
            -> AllocAllSqCq
                 -> AllocSqCqAndBindInternal
                 -> AllocSqAddr
                 -> BindSqCqAndSendSqe
                      -> BindSqCq
                      -> RebuildAllExternalTaskSqes
                      -> SendSqe
                      -> BindStreamToModel
                      -> ConfigSqTail
            -> notification/load-complete/activation fixups
            -> increment running refCount
       -> prepare and submit external-event refresh information
  -> ExecuteModel -> Model::Execute
       -> submit model execution and end-graph wait on executionStream
  -> PostModelExecute
```

Evidence: [execution wrapper](../../runtime/src/runtime/feature/aclgraph/capture_model.cc#L329),
[BuildSqCq ordering](../../runtime/src/runtime/feature/aclgraph/capture_model.cc#L1012),
[SQ allocation](../../runtime/src/runtime/feature/aclgraph/capture_model.cc#L1715),
[v200 bind/fill order](../../runtime/src/runtime/feature/aclgraph/v200/capture_model_adapt_v200.cc#L27).

### WQE upload and task patching

Standard contexts obtain a FREE jetty from the device pool. Large contexts
create a jetty with their rounded depth. `FillWqeToDevice` uploads every
128-KiB host chunk, including padding, using sequential offsets:

```text
NpuDriver::AsyncDmaWqeFill
  -> halAsyncDmaJettyWqeFill
  -> trs_async_dma_jetty_fill
  -> halSvmAccess(deviceJettyBase + offset, hostChunk, size, WRITE)
```

The HAL checks the fill against the jetty queue length. `halSvmAccess` can
use an eligible direct mapping or its synchronous DMA access path; the
latter reaches `svm_sync_copy`. Do not label every WQE fill an identical
transport operation without checking that selector. This upload transfers
**descriptors**, not the user's memcpy payload.

After the fill, `UpdateUbdmaSqeWithJettyInfo` sets each recorded memcpy task's
`dieId`, `functionId`, `jettyId`, and `pi = task WQE count`. It reconstructs
a UBDMA doorbell-mode SQE and updates the host SQE buffer. If the model's
SQEs are already uploaded, it also performs a synchronous patch of that
device SQE. The SQE uses `RT_UBDMA_SOURCE_MODEL_ASYNC`.

The value written here is the individual task's count, not a cumulative
host JFS PI. The host JFS doorbell from document 03 and this device UBDMA
doorbell-mode task have different roles.

Evidence: [bind, fill, and patch](../../runtime/src/runtime/feature/jetty/stream_jetty_handler.cc#L259),
[runtime/HAL fill](../../runtime/src/runtime/driver/npu_driver_standard_soc.cc#L857),
[device queue destination](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L1650),
[SVM access selector](../../driver/src/ascend_hal/svm/v3/api/master/svm_register_access.c#L597),
[synchronous access copy](../../driver/src/ascend_hal/svm/v3/api/master/svm_register_access.c#L444),
[doorbell SQE fields](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200.cc#L113).

### Task-SQ upload

`Model::SendSqe` invokes `StreamTaskFill` with the retained task-SQE count,
then marks `IsSendSqe = true`. Runtime reaches `halStreamTaskFill`. In the
HAL high-performance send mode, this fills device task storage through
`drvMemcpy`; the alternate mode uses `TRS_STREAM_TASK_FILL` ioctl.
`ConfigSqTail` separately publishes the configured task tail.

WQE fill, task-SQ fill, tail configuration, model activation, and payload
completion are therefore separate stages. Successful preparation alone
does not mean the captured copy has run.

Evidence: [SendSqe and tail](../../runtime/src/runtime/feature/model/model.cc#L757),
[task-fill mode selector](../../driver/src/ascend_hal/trs/core/trs_sqcq.c#L1752).

## 6. Replay reuses WQEs and advances over the NOP padding

`BuildSqCq` calls `BindJettyForUbdma` before testing whether SQ/CQ resources
already exist. This ordering matters: an idle graph can lose a pooled jetty
without losing all its other resources.

If the model remains jetty-bound, `BindJettyForUbdma` sets
`needUpdateUbPi = true` and skips WQE upload. If a new binding was needed,
it refreshes the jetty list and sets that flag to false. The list contains
the physical jetty identity and `piValue = C - V`.

Before submitting the model-execution task, `ModelSubmitExecuteTask` calls
`AsyncJettyToHead`. For a software-SQ capture model it submits the extra UB
doorbell tasks only when `needUpdateUbPi` is true. Each carries `P = C - V`
with source `RT_UBDMA_SOURCE_MODEL_EXE`. The recorded memcpy tasks then
carry their individual valid-WQE counts as before.

The host arithmetic expresses the intended queue-cycle behavior:

```text
Device WQE queue: [ V valid WQEs ][ P NOP WQEs ], depth C = V + P

First run after binding:        execute V valid entries
Next run with binding retained: advance P NOP entries, then execute V again
```

For the three-WQE example, the replay prelude carries 2045 and the captured
copies collectively carry 3. A retained graph does not regenerate and
upload these three WQEs on every replay. It still requests the H2D payload
again. Capturing a direct copy records addresses/descriptors, not an immutable
snapshot of the application's data.

For KV-cache offload, this can amortize descriptor preparation and upload
across repeated executions. It does not remove the KV payload bytes from
the host link or establish a bandwidth improvement. The transfer-volume
analysis in document 07 still applies.

The NOP traversal is the intent supported by the padding and count code.
The precise device interpretation of the two source values, counter width,
and completion ordering remains a firmware/hardware contract. It would be
incorrect to describe this as a host CPU simply writing a CI register to zero.

Evidence: [binding/replay flags and padding list](../../runtime/src/runtime/feature/aclgraph/v200/capture_model_adapt_v200.cc#L49),
[replay prelude and model submission](../../runtime/src/runtime/feature/model/model_c.cc#L231),
[standalone UB doorbell SQE](../../runtime/src/runtime/core/src/task/task_info/dma/ubdma_task_v200_base.cc#L274).

```mermaid
sequenceDiagram
    participant A as Application
    participant R as Runtime capture model
    participant D as Device queues
    A->>R: Capture copy and compute tasks
    R->>R: Retain task SQEs and converted WQEs
    A->>R: Capture end
    R->>R: Add NOP padding
    A->>R: First execute
    R->>D: Bind jetty, upload WQEs, upload patched SQEs
    R->>D: Submit model execution and end-graph wait
    D-->>R: Completion observed through runtime contract
    A->>R: Replay with resources retained
    R->>D: Padding-count doorbell, then model execution
    Note over D: Reuse descriptors; execute payload transfers again
```

## 7. Completion and resource reclamation are separate

For the Stars non-AICPU path, model execution submits an end-graph notify
wait on the execution stream. In software-SQ capture mode that wait is
associated with the capture model. Notify-task cleanup calls
`DeleteEndGraphNotifyInfo`; device SQ-head polling can also observe that
the tracked wait was passed. Both paths reach `CaptureModelExecuteFinish`.

That method decrements `refCount_`. A successful finish does not release
all jetties, WQE buffers, or persistent tasks. If the count reaches zero
with an error, it instead attempts `ReleaseAllJetty`.

Evidence: [end-graph wait](../../runtime/src/runtime/feature/model/model.cc#L1308),
[notify cleanup](../../runtime/src/runtime/core/src/task/task_info/event/notify_task.cc#L436),
[completion bookkeeping](../../runtime/src/runtime/core/src/device/raw_device.cc#L2462),
[SQ-head polling](../../runtime/src/runtime/core/src/device/raw_device.cc#L2548),
[refcount and error release](../../runtime/src/runtime/feature/aclgraph/capture_model.cc#L1343).

### Standard-depth jetty pressure

`AllocJettyWithRetry` tries the pool, attempts preallocation, and can ask the
context to recycle another capture model's resources. The recycler skips
the model currently being bound, non-software models, and models reported
as running, and uses the model resource lock before recycling.

For each bound jetty, `RecycleJetty` first calls `ResetJettyCi`. Despite the
name, this sends a padding-count UB doorbell on the control stream and
**synchronizes that stream**. It then:

- Returns a standard-depth jetty to the pool's FREE state and clears the
  stream binding. Host WQE chunks and task/count records remain.
- Keeps a large-depth jetty bound; the normal pressure-reclamation path
  does not return it as one of the freed standard resources.

The model clears its aggregate binding flag and jetty lists. On a later
execution, standard contexts bind and upload again, patching the new jetty
identity into task SQEs. A large context whose handle remains set skips
its upload. `RefreshJettyInfoList` reconstructs the model's current list.

Evidence: [pool retry](../../runtime/src/runtime/feature/jetty/jetty_manager.cc#L33),
[idle-model selection](../../runtime/src/runtime/feature/aclgraph/context_standard_soc_aclgraph.cc#L56),
[reset, control-stream wait, and recycle](../../runtime/src/runtime/feature/jetty/stream_jetty_handler.cc#L398),
[model-level recycle](../../runtime/src/runtime/feature/aclgraph/v200/capture_model_adapt_v200.cc#L112).

### SQ/CQ pressure is a different pool

`ReleaseSqCqAndNotifyId` requires no outstanding model references, unbinds
SQ/CQ resources, returns them lazily to the device pool, and releases notify
IDs. It does not call `ReleaseAllJetty`. The next `BuildSqCq` can rebuild
those resources while reusing other model-owned state. Do not equate an
SQ/CQ identity, its backing SQ memory, a jetty, and a host WQE allocation.

Evidence: [SQ/CQ reclamation](../../runtime/src/runtime/feature/aclgraph/capture_model.cc#L1112).

## 8. Updates have distinct mechanisms and supported task types

### A. Task-group kernel update submits an SQE patch

`aclmdlRICaptureTaskUpdateBegin/End` select a captured task group. In v200,
task-group sampling supports AIC/AIV kernels and the accepted fusion-kernel
case. During an update, `AllocTaskInfoForCapture` accepts a supported kernel
launch and rejects other task kinds. `Stream::UpdateTask` finds the existing
task and marks it `isUpdateSinkSqe`.

The kernel launch rebuilds its task/arguments and calls
`UpdateDavidKernelTaskSubmit`:

1. Construct replacement SQE bytes in a runtime-owned host allocation.
2. If software-SQ task storage has not yet been uploaded, also update the
   host SQE mirror in `UpdateDavidKernelPrepare`.
3. Create an internal H2D memcpy task targeting the captured device SQE.
4. Convert and submit that patch on the update stream.

For a software-SQ target, the destination is its device SQ base plus the
recorded SQE position; runtime allocates that storage if necessary. The
kernel-update validation requires the issuing stream to be a single-operator
stream with no model association. Its ordinary nonpersistent UB path uses
`CreateAsyncDmaWqe` for the patch. The conversion helper also contains a
persistent-stream branch that appends WQEs through the stream jetty context;
that branch alone does not establish a supported public update sequence.
The issuing stream and the persistent stream owning the target SQE have
different roles.

The patch task retains the temporary host SQE bytes and the old argument
handle. Task uninitialization releases them. Ending task-group update checks
the number of matched tasks and resets update bookkeeping; that function
contains no stream synchronization. A later execution must be ordered after
the patch on the relevant stream or by an appropriate dependency.

Evidence: [supported group tasks](../../runtime/src/runtime/feature/aclgraph/v200/capture_adapt_v200.cc#L91),
[update allocation selector](../../runtime/src/runtime/core/src/task/task_submit/v200/task_david.cc#L352),
[existing task selection](../../runtime/src/runtime/core/src/stream/stream.cc#L4724),
[SQE preparation and patch task](../../runtime/src/runtime/core/src/launch/aix_starsv2.cc#L163),
[update-stream validation](../../runtime/src/runtime/core/src/launch/aix_starsv2.cc#L268),
[software-SQ target](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L908),
[persistent/nonpersistent UB patch](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L797),
[patch resource release](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L419),
[update-end bookkeeping](../../runtime/src/runtime/feature/aclgraph/context_aclgraph.cc#L878).

This is not an API for arbitrarily changing a captured memcpy's source,
destination, length, or WQE count. In particular, the existence of an internal
memcpy used to patch a kernel SQE does not make memcpy a supported task-group
update target. Also, the host-mirror write above is conditional on
`!IsSendSqe()`; it must not be described as an unconditional mirror refresh.

### B. Task-parameter edits plus `aclmdlRIUpdate` rebuild the model SQEs

The task-parameter API checks that the software-SQ capture model can be
updated, polls completion if needed, and rejects a still-running/capturing
model. Supported parameter categories in this implementation are kernel,
value-write, and value-wait. The model enters UPDATING and tasks receive
update/keep/disable flags.

`aclmdlRIUpdate -> rtModelUpdate -> CaptureModel::Update` releases eligible
SQ/CQ/notify bindings, rebuilds driver stream resources using the same stream
ID, and calls `UpdateAllPersistentTask`. That routine resets positional
bookkeeping, returns existing device SQ storage to its pool, reconstructs
or retains each task's host SQE according to its flag, and drops disabled
tasks. The model marks `IsSendSqe = false`; a successful API update restores
READY, and a failure marks FAULT. The next execution builds/uploads task
storage again.

This path does not contain a general regeneration of all captured memcpy
WQEs. Do not infer arbitrary memcpy retargeting from model rebuild support.
UB task WQE contexts and SQ task storage have separate ownership.

Evidence: [update preconditions](../../runtime/src/runtime/feature/aclgraph/capture_model_utils.cc#L241),
[parameter categories](../../runtime/src/runtime/api/impl/api_impl.cc#L9647),
[RI update state handling](../../runtime/src/runtime/api/impl/api_impl.cc#L9699),
[model rebuild](../../runtime/src/runtime/feature/aclgraph/capture_model.cc#L1381),
[persistent task rebuild](../../runtime/src/runtime/core/src/stream/stream.cc#L3624),
[v200 SQE reconstruction](../../runtime/src/runtime/core/src/stream/stream_david.cc#L1683),
[same stream-ID reallocation](../../runtime/src/runtime/core/src/stream/stream.cc#L1332).

## 9. Destruction and ownership

The normal `Context::ModelDestroy` path rejects a model still being captured.
For a running capture model it polls end-graph progress while checking the
running reference count, with a bounded retry loop; it can return
`RT_ERROR_MODEL_RUNNING` rather than deleting it. After that check it removes
the model from the context, calls `TearDown`, and deletes it.

The capture-model destructor releases external refresh state and capture
events, calls `ReleaseAllJetty`, deconstructs SQ/CQ resources, removes model
stream associations, and destroys the internal capture streams. Persistent
stream teardown recycles retained tasks. Stream destruction frees host SQE
storage and returns software-SQ backing memory to its allocator.

| Resource | Owner and ordinary release behavior |
|---|---|
| User copy source/destination | Application or its allocator; descriptor retention does not transfer allocation ownership |
| Captured `TaskInfo` objects | Persistent capture streams; retained across replay, recycled during task removal/stream teardown |
| Host task-SQE buffer | Stream; retained for preparation/rebuild, freed by stream destruction |
| Device task-SQ backing memory | Stream through SQ-address allocator; can survive identity reclamation, returned on rebuild/destruction |
| Host WQE chunks and task/count list | Stream jetty context managed by device `JettyManager`; retained after normal pool reclamation, released by `ReleaseJetty` |
| Bound standard jetty | Device pool; lazy FREE on idle reclamation, actual destroy if still bound during final release |
| Large jetty | Dedicated large-depth pool entry; retained by normal idle reclamation, destroyed at final release |
| SQE patch bytes / old argument handle | Internal patch task; released through task cleanup |

`ReleaseJetty` destroys/unbinds the applicable handle, frees its host WQE
buffers with `HostMemFree`, then deletes the stream jetty context. A standard
jetty already returned to the pool has no stream-owned handle to destroy at
that point. HAL jetty destruction unregisters the queue region and requests
remote free; the external service supplies the remote part of that contract.

Evidence: [normal destroy gate](../../runtime/src/runtime/core/src/context/context.cc#L1960),
[capture destructor](../../runtime/src/runtime/feature/aclgraph/capture_model.cc#L81),
[jetty/context release](../../runtime/src/runtime/feature/jetty/stream_jetty_handler.cc#L462),
[persistent stream teardown](../../runtime/src/runtime/core/src/stream/stream_david.cc#L568),
[task recycling](../../runtime/src/runtime/core/src/stream/stream_david.cc#L1531),
[stream storage release](../../runtime/src/runtime/core/src/stream/stream.cc#L232),
[HAL remote-free path](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L1388).

Error and process-exit paths are distinct. For example, the capture destructor
has an early process-exit branch, and an error finish can release jetties.
These host actions are not proof that every failed remote operation was
quiesced or that replay after failure is safe. The source also returns early
from intermediate bind/fill/patch failures; this chapter does not assume an
atomic rollback of all preparation stages.

## 10. Auto-split shares deferred SQEs but is a separate path

`IsAutoSplitSq()` is checked separately from `IsSoftwareSqEnable()`.
Auto-split setup allocates its host SQE buffer, split context, and SQ/CQ
resources. When the current stream approaches capacity, task allocation
creates a slave stream, adds it to the same model, and records a stream
activation from the preceding stream. `DavidSendTask` writes the SQE into
the split stream's host buffer rather than calling ordinary task send.

At `Model::LoadComplete`, `BuildSqCqForAutoSplit` allocates device SQ storage,
constructs stream/SQ switch descriptions, calls `SqSwitchStreamBatch`, fills
the SQEs through `SendSqe`, binds model streams, and configures their tails.
Its upload boundary is therefore load-complete, whereas software-SQ capture
performs the resource-building sequence in `BuildSqCq` before execution.

Auto-split alone does not select `StreamJettyHandler`: normal UB conversion
selects that helper using `IsSoftwareSqEnable`. Non-software models use the
driver-created async contexts and the load-complete jetty-query path from
`GetAndSaveJettyInfo`; that function explicitly skips software-SQ capture
models. Their replay prelude uses first-execution state rather than the
software-capture `needUpdateUbPi` selector.

Evidence: [auto-split setup](../../runtime/src/runtime/core/src/stream/stream.cc#L1034),
[slave creation/activation](../../runtime/src/runtime/core/src/task/task_submit/v200/task_david.cc#L154),
[slave ownership](../../runtime/src/runtime/core/src/context/context.cc#L1656),
[host SQE write](../../runtime/src/runtime/core/src/task/task_submit/v200/task_david.cc#L498),
[load-complete build and fill](../../runtime/src/runtime/feature/model/model.cc#L807),
[normal UB conversion selector](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L868),
[non-software jetty query](../../runtime/src/runtime/feature/model/model_c.cc#L408).

This comparison closes the selector/upload distinction. It is not a full
audit of every auto-split slave cleanup/error path, split-boundary update,
or device stream-activation implementation. Those should remain a separate
bounded follow-up rather than inheriting the capture-model ownership rules.

## 11. Coverage and remaining evidence

The source trace now covers software-SQ capture recording, NOP finalization,
first bind/fill/patch/upload, retained replay, completion references,
standard-versus-large jetty reclamation, the two update mechanisms, and
normal destruction. Ordinary-stream behavior remains in document 06.

The remaining work is narrower:

- Correction and validation of the findings confirmed in
  [10](10_batch_correctness_review.md), following the 2D/batch lifecycle
  trace in 09.
- Complete auto-split slave/update/error ownership beyond the comparison
  above, and nested conditional/external-event edge cases.
- Device interpretation of doorbell counts, scheduling/report ordering,
  cache visibility, and failure quiescence; these require implementation or
  interface evidence beyond the host trace.
- Hardware validation of first execution versus replay, jetty-pressure
  rebind, update ordering, and teardown, using a verified runtime/driver pair.

Validation is source inspection, local link/line-anchor checking, and the
explicit WQE-capacity arithmetic. No runtime library was built and no NPU
workload was run. The repository also contains mocked jetty tests, including
retained binding and standard/large recycling; their presence is not a
hardware result and they were not executed for this documentation change.
[Existing tests](../../runtime/tests/ut/runtime/runtime/test/platform/950/rt_utest_aclgraph_ub.cc#L826)
