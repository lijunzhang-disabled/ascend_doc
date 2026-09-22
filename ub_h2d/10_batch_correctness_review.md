# 10 — Batch continuation correctness review

[Summary](README.md) · Previous: [Runtime 2D and batch bookkeeping](09_runtime_2d_and_batch_bookkeeping.md)

Review date: 2026-09-22. Source snapshots: `runtime` at `216912472` and
`driver` at `866e409`. Scope: the UB H2D/D2H batch implementation selected
by `ApiImplDavid`, including ordinary streams and software-SQ capture.
This is a static correctness review. Runtime and driver sources were not
modified, and native unit tests and device workloads were not run.

## 1. Review result

The two concerns from chapter 09 are confirmed violations of the host-side
progress contract. Following their cleanup paths exposes an additional
credit-ownership defect. “Confirmed” here means that the visible producer
and consumer code disagree; it does not mean a device-level failure was
observed or that every deployment selects this implementation.

| Finding | Priority | Source-level consequence |
|---|---|---|
| B1: batch submission compares successive per-call counts | P1 | Valid newly prepared work can bypass its task submission while the outer loop consumes the reported entry count |
| B2: captured byte-only continuation is discarded | P1 | The next conversion can retain an already-prepared prefix instead of advancing the remaining entry |
| B3: empty preparation can return credits belonging to earlier work | P1 | Task unwind can assign the shared jetty CI without observing completion of the work represented by that PI |

P1 denotes a correctness issue to resolve before relying on the affected
partial-progress path. It is not a security severity assessment. Fixes and
behavioral validation remain outstanding.

## 2. Contracts checked before classifying the findings

The batch path is selected only after runtime chip-feature checks and the
driver-feature/connection selector described in chapter 09. The selected
UB path validates all batch entries, and unregistered-memory cases use a
separate synchronous fallback. The findings below concern work that reaches
the asynchronous preparation loop.

The key facts checked in both checkouts are:

1. `BatchMemcpyAsync` keeps private arrays and passes the preceding
   `fixedCnt`/`fixedSize` into the next attempt. It subtracts each returned
   `realCnt` from `remainCnt`.
2. HAL batch preparation resets its whole-entry count on every call. It
   does not return a cumulative count from the original API request.
3. Ordinary conversion shifts the arrays itself and passes the remaining
   arrays to HAL. Neither it nor the driver wrapper turns the returned
   count into a cumulative count.
4. Standalone WQE conversion has a separate partial-byte output. It can
   prepare bytes without completing a whole entry.
5. Ordinary 2D/batch destruction assigns the supplied CI to shared
   `batch_2d_async_ctx` state. That function does not poll completion.

These checks rule out an alternative interpretation in which the counter
comparison or byte-offset omission would be justified by a lower layer.

Evidence: [outer loop](../../runtime/src/runtime/api/impl/api_impl_david.cc#L860),
[ordinary conversion](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L273),
[driver wrapper](../../runtime/src/runtime/driver/npu_driver_standard_soc.cc#L162),
[HAL per-call progress](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L555),
[HAL partial-byte output](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L1599),
[HAL CI assignment](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L1170).

## 3. B1 — Preserve submission for every positive preparation result

The ordinary-stream no-progress branch in `MemcopyBatchAsync` compares the
preceding attempt's `batchInfo.fixedCnt` with the current task's `size`.
Both are per-call whole-entry counts. Equality between them says nothing
about whether the current call prepared work.

The branch returns before `DavidSendTask` and before writing the output
parameters. Its scope guard still calls task uninitialization and slot
rollback. Meanwhile the outer loop retains the preceding `realCnt` and
continues consuming it as the result of the current successful attempt.
For equal positive results, that counts prepared entries without submitting
their required doorbell task. The exact effect on eventual device execution
depends on later submissions; the missing submission is already visible in
the host code.

There is no hidden cumulative-count correction: `ShiftBatchArrays` moves
the arrays but leaves `batchInfo.fixedCnt` unchanged, and HAL starts its
new count at zero. Nor is this a 2D-style cursor: the neighboring 2D path
compares absolute byte positions, which is a different contract.

**Required fix behavior:** base the ordinary batch no-work decision on the
current preparation result alone. Publish the current `realCnt` and
`realSize` on every successful return, including an empty result. Changing
only the comparison is insufficient: an early return that leaves the old
output count intact can still make the outer loop skip unprepared entries.
Capture must continue to distinguish zero completed entries from zero
prepared bytes/WQEs.

Evidence: [submission guard and output assignment](../../runtime/src/runtime/core/src/launch/memcpy_starsv2.cc#L135),
[array shift preserves the input cursor fields](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L206),
[outer count consumption](../../runtime/src/runtime/api/impl/api_impl_david.cc#L864),
[per-call HAL count initialization](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L566).

## 4. B2 — Apply the partial-byte cursor independently of array compaction

`ConvertAsyncDmaBatchForSoftWareSq` requests byte-offset handling with
`ShiftBatchArrays(batchInfo, true)`. However, that helper returns immediately
when the preceding `fixedCnt` is zero. The return occurs before its separate
source/destination offset and size reduction.

HAL standalone conversion can supply a nonzero partial-byte result with
zero completed entries. It computes progress from the space remaining in
the supplied host WQE buffer. That space is only the remainder of the
current shared chunk, so this case does not require an entry larger than
the graph's entire WQE storage capacity.

Consequently, the next conversion can start from the same addresses and
length despite the already-recorded prefix. The persistent task/WQE records
from the earlier conversion are still retained by `StreamJettyHandler`.
This is a cursor-consumption defect; metadata compaction alone cannot fix it.
The visible path does not establish one universal device symptom: duplicate
recorded work and loss of progress are different possible consequences of
the surrounding conversion sequence.

**Required fix behavior:** make whole-entry compaction conditional on a
nonzero completed-entry count, but apply a valid partial-byte offset
independently. Preserve bounds checks and reduce the first remaining entry's
length consistently with both adjusted addresses. A successful byte-only
result is progress and must not be discarded as an empty conversion.

Evidence: [early return and byte adjustment](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L206),
[software-SQ caller](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L237),
[remaining chunk space and retained records](../../runtime/src/runtime/feature/jetty/stream_jetty_handler.cc#L98),
[HAL partial-entry calculation](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L566).

## 5. B3 — An empty preparation does not own the queue's current PI

HAL create can succeed with no new WQEs while returning the async context's
existing PI. That PI can include earlier prepared work; it is not proof of
that work's completion.

On the runtime side, successful batch conversion sets
`dmaKernelConvertFlag = true` even for an empty result. If the task helper
takes its no-work return, its active scope guard calls `TaskUnInitProc`.
The memcpy cleanup then selects `AsyncDmaWqeBatchProc` and passes the saved
PI to `DestroyAsyncDmaWqeBatch`. HAL stores it as CI without a completion
check. Task-slot rollback does not wait for payload execution either.

The outer loop's synchronization happens after this unwind, so it cannot
justify the earlier CI assignment. If CI was already equal to PI, the
assignment is harmless; if earlier work remains outstanding, the empty
attempt has no completion evidence authorizing that credit return.

This issue is distinct from B1. Correcting the count comparison still leaves
the genuine empty-result cleanup path to handle. B1 can also lead the same
cleanup machinery to release newly prepared work that was never submitted.

**Required fix behavior:** distinguish task-metadata cleanup from ownership
of DMA completion credit. An empty preparation must still release its local
task resources and slot, but must not acknowledge the shared queue's PI as
completed. Positive prepared work that fails before submission requires a
separate cancellation/rollback contract; ordinary completion cleanup is not
automatically that contract.

The neighboring ordinary UB 2D no-progress branch uses the same scope-guard
pattern and a method-specific CI-return helper. Include it when correcting
empty-preparation ownership, even though its cumulative-byte comparison is
itself appropriate.

Evidence: [HAL empty-result output](../../driver/src/ascend_hal/trs/core/urma/master/trs_master_async.c#L617),
[conversion marks cleanup enabled](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L318),
[batch scope guard](../../runtime/src/runtime/core/src/launch/memcpy_starsv2.cc#L135),
[uninitialization dispatch](../../runtime/src/runtime/core/src/task/task_info/task_manager.cc#L411),
[memcpy cleanup](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L419),
[method-specific CI return](../../runtime/src/runtime/core/src/task/task_info/memory/memory_task_v200_base.cc#L336),
[local task-slot rollback](../../runtime/src/runtime/core/src/task/task_submit/v200/task_david.cc#L54),
[2D no-progress branch](../../runtime/src/runtime/core/src/launch/memcpy_starsv2.cc#L79).

## 6. Existing tests do not close these findings

The inspected 950 tests cover API routing, task-init failures, and individual
field propagation, but their boundaries omit the relevant contracts:

| Existing test | Coverage limitation |
|---|---|
| `test_memcpy_batch_async_batch_path` | Replaces `MemcopyBatchAsync` with a successful whole-batch result, bypassing submission and continuation logic |
| `TestMemcopyBatchAsync` | Stubs initialization and submission; it does not check successive preparation results, current output counters, or credit ownership |
| `TestMemcopyBatchAsyncFail` | Covers an initialization error, not a successful empty preparation or a later submission failure |
| `memcpy_async_batch_ub_dma_test` | Checks selected conversion behavior with mocks; its successful mock reports a count larger than its input count, so it is not a valid continuation oracle |
| Software-SQ 2D `FixedCntOne` test | Checks normalization of a complete matrix result, not partial batch continuation |

Evidence: [API route test](../../runtime/tests/ut/runtime/runtime/test/platform/950/rt_utest_api_david.cc#L10407),
[task helper tests](../../runtime/tests/ut/runtime/runtime/test/platform/950/rt_utest_david_task.cc#L523),
[conversion mock and test](../../runtime/tests/ut/runtime/runtime/test/platform/950/rt_utest_david_task.cc#L2167),
[2D normalization test](../../runtime/tests/ut/runtime/runtime/test/platform/950/rt_utest_aclgraph_ub.cc#L2294).

The task tests are part of `runtime_utest_task_david`; API tests are in
`runtime_utest_api_david`. The checkout has no configured build, and the
usual local CANN SDK, GTest, and MockCPP installation paths checked for this
review were absent. No native build was attempted. This limits validation;
it does not turn mocked test presence into evidence that the paths work.
[Native test targets](../../runtime/tests/ut/runtime/runtime/test/platform/950/CMakeLists.txt#L12)

## 7. Acceptance criteria for a corrective patch

The fix should preserve these observable properties, with lower-level
transport mocked where appropriate and real runtime continuation logic
left in the test:

| Area | Required regression property |
|---|---|
| Ordinary positive progress | Every successfully prepared positive portion gets its required task submission, irrespective of previous portion size |
| Ordinary empty progress | Returned progress is current and empty; remaining entries are preserved; local task resources are reclaimed without a DMA CI update |
| Captured partial progress | The next conversion consumes exactly the remaining byte range, including byte-only continuation |
| Captured mixed progress | Whole-entry compaction and the partial next-entry adjustment compose without duplicating or omitting payload ranges |
| Normal completion | Credit return occurs through the completed task's ownership; it is not disabled by the empty-result fix |
| Error paths | Preparation failures, failed task submission, and postprocessing errors have explicitly distinguished cleanup behavior |
| 2D shared cleanup | Empty 2D preparation preserves its absolute byte cursor and does not acknowledge earlier queue work |
| Caller metadata | Runtime continuation leaves the application's descriptor arrays unchanged |

Tests should check task submission, output counters, the remaining ranges,
and cleanup side effects together. Stubbing the entire task helper at the
API boundary would miss B1 and B3; stubbing the shift/conversion layer would
miss B2. Device validation must subsequently confirm payload correctness and
completion ordering with a verified runtime/driver pair.

## 8. Disposition and remaining work

The review is complete for B1–B3 at the host-source contract level. The next
implementation should address ordinary batch result publication/submission,
captured byte-cursor consumption, and empty-preparation credit ownership as
one related correction, with the regression properties above.

Separate open items remain: cancellation of positive prepared-but-unsubmitted
work, delayed recycling and progress under sustained pressure, graph failure
unwind, and firmware/device ordering. The existing 2048-entry HAL input
limit and incomplete `failIndex` coverage from chapter 09 are API/diagnostic
limitations, not additional findings claimed fixed by this review.

Only documentation was changed. Validation consists of source and test-code
inspection plus local link/anchor and whitespace checks. Runtime fixes,
native regression tests, and hardware validation remain outstanding.
