# 13 — Transparent encryption for D2rH and rH2D

[Summary](README.md) · Previous: [D2rH/rH2D traffic](12_remote_host_memory_d2rh_rh2d.md) · Next: [HcommWriteOnThread over UB](14_hcomm_write_on_thread.md)

Analysis date: 2026-10-13. **Status: source-backed design proposal, not an
implemented or hardware-validated remote encryption feature.**

## 1. Objective and scope

Apply the transparent encryption model in `~/ub/runtime_enc` to remote
host-memory transfers. Applications keep their existing HIXL/HCOMM calls,
logical buffer addresses, lengths, and synchronization operations. Both
application endpoints see plaintext; the payload crossing the interconnect
is ciphertext. In particular, the remote host decrypts D2rH data before
publishing application-visible completion.

Let A contain the initiating NPU and B own the remote host buffer:

- **D2rH:** A writes its device data to B's host buffer.
- **rH2D:** A reads B's host data into its device buffer.

The initial scope is HIXL's AICPU execution path over UB_CTP, using HCOMM
one-sided primitives. Host-side RoCE is a separately counted extension.
An application making valid use of the original API should not need to add
crypto calls, staging allocations, remote workers, or new completion calls.
Both peers do need compatible updated library implementations.

This is source/API transparency, not a claim of zero overhead. Encryption,
extra memory accesses, control exchanges, and receiver processing add work.
Existing buffer lifetime and producer/consumer synchronization obligations
still apply.

Inspected snapshots:

| Component | Revision | Role |
|---|---|---|
| HCOMM | `f9f91c08434fffac4c0bd754a93d10f6665fc1a8` | Resource, transfer, and completion APIs |
| HCCL | `e6c8122480151b4f154ceb489a49ce05ecdc1769` | Other consumers of the shared primitives |
| HIXL | `49bd2a256b728544c1e12df555c545670f9ab51a` | Concrete remote-memory caller |
| `~/ub/runtime_enc/AscendCCv2-Runtime` | `1ec3ba3eda87d2ac95a576039e749708f9a9ae34` | Existing transparent local-copy orchestration |
| `~/ub/runtime_enc/AscendCCv2-AICPU` | `0dbf653477a380c2ba510461de75e481b1b5d17b` | Existing device crypto operator |

HCOMM links assume the sibling checkout used by this documentation. HIXL
links are pinned upstream. Links to the local secure Runtime/AICPU trees
are absolute because those trees are outside `ascend_stack`; their paths
and revisions are recorded above. The snapshots are not a verified installed
software combination.

## 2. What the existing local implementation already hides

The relevant example is the secure Runtime implementation, rather than the
older torch_npu-only hook design.

| Existing component | Observed behavior | Remote-transfer analogue |
|---|---|---|
| `TryRouteAsync` | Routes H2D/D2H into a context-owned engine, with eager/capture handling | HCOMM routes selected logical READ/WRITE operations into a per-peer secure engine |
| `FixedRequest::Allocate` | Allocates hidden host/device slots and a control block | Allocate and register local/remote ciphertext slots and request state |
| `FixedRequest::StepLocked` | Host worker encrypts H2D input or decrypts D2H output | B's library worker prepares rH2D ciphertext or delivers D2rH plaintext |
| `RunRequest` in the AICPU operator | Encrypts device source data or decrypts into device destination | Crypto on A before WRITE or after READ |
| `HostCompletion` integration | Stream query/synchronization includes host work | Communication completion includes the destination's crypto/delivery |

Sources: [routing][runtime-route], [allocation and worker][runtime-fixed],
[AICPU crypto execution][device-fixed], [stream completion][runtime-stream].

The current `SecureSeal`/`SecureUnseal` implementation uses AES-256-CTR with
a slot header. The inspected functions do **not** generate or verify an
authentication tag. Reusing their orchestration must not be described as
already providing authenticated encryption. A wire-integrity requirement
needs an appropriate crypto format and verification step in addition to the
transparent scheduling design. [Crypto implementation][runtime-crypto].

The existing `RuntimeTransport::ToDevice/FromDevice` uses local driver
`MemCopySync`. That transport adapter cannot simply interpret an address in
another OS as a local host/device pointer. The remote version needs HCOMM's
peer/channel and registered-memory machinery. [Local transport][runtime-transport].

## 3. Proposed internal flows

### 3.1 D2rH: encrypted WRITE, plaintext delivery on B

```text
Application uses its existing WRITE/TransferAsync call
  → A resolves the logical destination to B's registered buffer and offset
  → A reserves an internal ciphertext slot on B
  → A orders device encryption after the source producer
  → A encrypts into a registered device ciphertext buffer
  → raw HCOMM WRITE transfers ciphertext to B's ciphertext slot
  → an ordered internal ready indication identifies the completed payload
  → B's library-owned host worker decrypts into the original host destination
  → B publishes delivery status / acknowledgment
  → A's secure operation satisfies its original completion dependencies
```

The worker belongs to B's communication library. The application does not
create it or call a decrypt API. The acknowledgment is internal and binds
the operation to its request identity and buffer range.

An ordinary incoming one-sided WRITE does not invoke B's
`HcommWriteOnThread`. Consequently, changing A's function body alone cannot
deliver plaintext on B. B's worker and request processing must be installed
through resource setup.

Payload completion must precede the ready indication. A local send/write
completion is not automatically a remote worker notification. Polling an
internal state or using a supported message/notification mechanism are
possible implementations; the chosen provider's ordering and visibility
must be established. The existing local JFC proposal also distinguishes
these concepts. [JFC design proposal][jfc-proposal].

### 3.2 rH2D: prepare ciphertext on B before READ

```text
Application uses its existing READ/TransferAsync call
  → A sends an internal prepare-read request for B's logical buffer range
  → B's library worker encrypts that range into a registered ciphertext slot
  → B returns ciphertext-ready metadata and a slot lease
  → A issues a raw HCOMM READ into local device ciphertext staging
  → after READ completion, A decrypts into the original device destination
  → the library retires the slot lease and reports operation completion
```

Reading plaintext first and decrypting afterward would not encrypt the
wire. Nor is encrypting the host region once at `HcommMemReg` sufficient:
the application may update it between transfers. Registration establishes
access and lifetime, not a permanent snapshot of its contents. The worker
must prepare the requested version under the application's existing source
stability contract.

The library may choose a different internal push/pull mechanism, provided
the original read result, ordering, errors, and buffer lifetimes are preserved.

## 4. Can `HcommWriteOnThread` be the interception point?

**Yes.** Adapt it to recognize a secure operation and submit the composite
crypto/transport/delivery sequence. Keep the API signature and asynchronous
submission behavior. Put reusable state and scheduling in an internal engine
rather than implementing registration, remote processing, and crypto inline
in each API wrapper.

The following is architectural pseudocode, not an existing or compilable API:

```cpp
HcommWriteOnThread(thread, channel, dst, src, len)
{
    ValidateExistingApiArguments(...);
    auto request = ResolveLogicalRegisteredRange(channel, dst, src, len);
    if (request.requiresEncryption) {
        return SecureTransferEngine(channel).SubmitWrite(thread, request);
    }
    return RawWriteOnThread(thread, channel, dst, src, len);
}
```

The secure branch must precede construction of the ordinary payload WQE.
The real function currently builds a local RMA buffer and calls transport
`Write`; this is where unmodified plaintext would become the hardware source.
[Scalar WRITE implementation][hcomm-write].

`RawWriteOnThread` above is a proposed internal helper. It moves already
prepared ciphertext and internal protocol data without recursively entering
the secure policy. It must not create a caller-selectable bypass for a
protected application transfer.

Address translation is part of the engine: the application's logical remote
address identifies the original buffer, while hardware uses the allocated
ciphertext slot and its registration/token information. Resolve bounds and
offsets before replacing addresses; preserve the caller's original byte
count even when the internal wire format has extra bytes.

On the inspected AICPU path this entry point executes in device code.
Intercepting only a host shared-library symbol is insufficient. HCOMM builds
`cpu_primitives_c_adpt.cc` into the host object target and
`aicpu_ts_primitives_c_adpt.cc` into `ccl_kernel`. Resource creation and host
progress still belong on the host side. [Build selection][primitive-build].

## 5. Payload interception counts

Counts below are **distinct public function names**, not source files,
generated WQEs, new helper functions, or total implementation edits.

| Coverage target | Payload entry points | Count |
|---|---|---:|
| Only calls explicitly made through scalar WRITE | `HcommWriteOnThread` | **1** |
| HIXL's inspected UB device-side D2rH WRITE path, including batch and fallback | `HcommWriteOnThread`, `HcommBatchTransferOnThread` | **2** |
| HIXL's inspected UB device-side D2rH and rH2D paths | Previous two plus `HcommReadOnThread` | **3** |
| Previous row plus HIXL's host-side RoCE path | Previous three plus `HcommWriteNbiOnThread`, `HcommReadNbiOnThread` | **5** |
| Broader non-reduction transfer API family in `hcomm_primitives.h` | All ten names listed below | **10** |

HIXL first attempts `HcommBatchTransferOnThread` and falls back to scalar
READ/WRITE on `HCCL_E_NOT_SUPPORT`. The batch API can submit directly to
`ExecuteBatchTransfer`; it does not universally funnel through scalar
`HcommWriteOnThread`. Therefore, intercepting the scalar WRITE alone misses
the successful batch path. [HIXL batch/fallback][hixl-kernel],
[HCOMM batch implementation][hcomm-batch].

The inspected HIXL host path calls the two `*NbiOnThread` functions separately.
These must be included when that RoCE backend is enabled. [Host dispatch][hixl-host].

The ten ordinary-transfer API names are:

1. `HcommWriteOnThread`
2. `HcommReadOnThread`
3. `HcommBatchTransferOnThread`
4. `HcommWriteWithNotifyOnThread`
5. `HcommWriteNbiOnThread`
6. `HcommReadNbiOnThread`
7. `HcommWriteNbi`
8. `HcommReadNbi`
9. `HcommWriteWithNotifyNbiOnThread`
10. `HcommWriteWithNotifyNbi`

This is an API inventory, not a claim that every name is supported by every
engine or protocol. For example, the AICPU implementation rejects the NBI
entries, while the host implementation dispatches them through NIC operations.
The notify-bearing WRITE has its own transport call and is not guaranteed
to pass through scalar WRITE. [Public APIs][primitive-header],
[notify WRITE][hcomm-write-notify], [host NBI dispatch][host-nbi].

Ordinary ciphertext cannot be passed unchanged to plaintext arithmetic.
`HcommWriteReduceOnThread`, `HcommReadReduceOnThread`, and
`HcommWriteReduceWithNotifyOnThread` are **three additional scalar reduction
entries**, outside this copy design. The batch API also contains reduction
and notification descriptor kinds; classify each descriptor and reject
unsupported protected operations rather than silently treating every batch
as a byte copy. [Batch descriptor dispatch][batch-types].

## 6. Supporting integration: a concrete initial audit inventory

For the initial **eager HIXL UB READ/WRITE scope**, start with the following
**18 named HCOMM API surfaces to review**. This is a bounded implementation
inventory, not a statement that 18 independent wrappers are necessary or
that these functions cover every HCOMM client.

| Area | API surfaces | Count | Integration responsibility |
|---|---|---:|---|
| Payload submission | `HcommWriteOnThread`, `HcommReadOnThread`, `HcommBatchTransferOnThread` | 3 | Route selected logical transfers to secure requests |
| Endpoint lifecycle | `HcommEndpointCreate`, `HcommEndpointDestroy` | 2 | Own per-endpoint host progress, session state, and shutdown |
| Memory lifecycle and exchange | `HcommMemReg`, `HcommMemUnreg`, `HcommMemExport`, `HcommMemImport`, `HcommMemUnimport` | 5 | Track logical regions, ciphertext registrations, peer metadata, and lifetime |
| Channel lifecycle | `HcommChannelCreate`, `HcommChannelGetStatus`, `HcommChannelDestroy` | 3 | Establish secure peer readiness and clean up outstanding requests |
| Thread lifecycle | `HcommThreadAlloc`, `HcommThreadFree` | 2 | Associate ordering and pending secure work with the execution thread |
| Channel ordering | `HcommChannelFenceOnThread` | 1 | Include secure delivery dependencies in the applicable ordering contract |
| Submission batching | `HcommBatchModeStart`, `HcommBatchModeEnd` | 2 | Preserve launch batching while inserting crypto and protocol work |
| **Total initial audit inventory** | **3 payload + 12 resource/lifecycle + 3 ordering/batching** | **18** | **Not the total number of source functions that will change** |

Sources: [HIXL adapter surface][hixl-proxy], [resource declarations][resource-header],
[channel declarations][channel-header], [memory implementation][memory-adapter].

Implement common behavior in resource managers, transport state, and the
secure engine. For example, unregister and endpoint destruction may already
converge on one cleanup implementation; do not duplicate it merely because
two public APIs are in the inventory. Likewise, a composite task that gates
all later work may preserve a fence without changing the wrapper itself.

The following are extensions to that initial inventory, not hidden in the
number 18:

- Other callers can use `HcommChannelCreateWithConfig`, communicator-backed
  `Hccl*` resource APIs, configured/thread-from-stream allocation variants,
  `HcommChannelFence`, or `HcommChannelDrainOnThread`. Integrate the shared
  internals they reach or cover the additional entry point explicitly.
- HIXL's `CheckStatusDevice`, synchronous wait, and published completion flag
  must be reviewed end to end. These are outside the HCOMM API count. If the
  secure operation remains pending in the underlying execution sequence
  until plaintext delivery, existing upper completion code may remain intact;
  otherwise it needs an explicit completion extension.
- Notifications after a secure copy, including the HIXL completion protocol,
  must not overtake it. Count their dependencies, even when their function
  bodies need no interception.
- Graph/task-cache support adds recording, execution, update, and cleanup
  paths. It is not established by the eager inventory.

Do not implement every fence as a blocking host wait. In this snapshot,
the AICPU `HcommChannelFenceOnThread` calls transport `Fence`; it is not
itself a remote plaintext-delivery acknowledgment. Other fence functions
return `HCCL_E_NOT_SUPPORT` on that build. Preserve each supported API's
contract and express dependencies at the execution layer.
[AICPU fences][aicpu-fences], [host fence/drain][host-fences].

## 7. State, completion, and classification requirements

The proposed engine should own per-peer crypto context, registered staging
pools, request IDs/epochs, logical-region mappings, host progress, and device
task dependencies. A and B must agree on the crypto context; a key provisioned
only for A's existing local host/NPU pair does not automatically establish a
session with B. Keys are not exported as ordinary memory descriptors.

Completion has three distinct stages: ciphertext ready, transport complete,
and destination plaintext ready. D2rH's final stage is on B's host CPU;
rH2D's final stage is on A's device. Public completion and consumer notification
must correspond to the appropriate final stage, while internal events can
represent earlier stages. Preserve asynchronous submission; do not add an
unconditional synchronization to each intercepted call.

The inspected HIXL device path publishes a small local D2H completion flag
after its kernel/notify sequence. That flag must be ordered after the new
secure operation, or it could report completion while B is still decrypting.
[Existing completion publication][hixl-completion].

Select operations using explicit registered-region/session metadata. A
function name or pointer value alone does not establish D2rH/rH2D. Mapped
host memory may be represented by a device address. HIXL also uses scalar
READ for a control flag; encrypting that access as application payload would
break the protocol. A control path needs its own treatment and must not be
mistaken for a missed payload. [HIXL control READ][hixl-control].

Retain ciphertext slots until their transport users have finished, and retain
logical buffers until their library users finish. On timeout or partial
submission, retire or quarantine resources according to proven completion;
do not free a buffer merely because the API returned an error. For concurrent
operations, identify each slot and buffer range rather than relying on one
shared FULL/EMPTY bit with no request identity.

## 8. Capture, other engines, and coverage boundaries

An API hook can run at recording time while cached work executes repeatedly
without re-entering that function. Fresh data preparation, nonce allocation,
remote slot leases, and completion must occur on every execution. A graph
must not replay a stale prepared ciphertext buffer or reuse per-execution
state accidentally. The existing Runtime fixed path provides a useful
resource-lifetime pattern, not a ready-made cross-process replay protocol.

HCOMM separately exposes `HcommAicpuTsTaskCacheLookup`, `Start`, `End`,
`Execute`, and `Clear`. Their execution path must be covered before claiming
task-cache support. [Task-cache implementation][task-cache].

These C primitive hooks are not a universal interception layer for every
communication engine. In particular:

- HIXL's separate UB_MEM engine has a distinct submission branch.
- AIV and CCU execution require their own coverage analysis; CCU has
  separate READ/WRITE primitives.
- Reduction semantics require a separate design.
- Modifying a host symbol does not modify the AICPU binary with the same
  exported function name.
- Protecting remote application payloads is not the same claim as protecting
  every control descriptor, task argument, or local H2D/D2H copy in the stack.

Sources: [HIXL backend split][hixl-dispatch], [CCU primitives][ccu-primitives],
[host/device build split][primitive-build].

## 9. Suggested implementation sequence and validation

1. Implement eager UB D2rH with the scalar and batch WRITE entries, hidden
   ciphertext registrations, B's internal worker, and delivery completion.
2. Add scalar/batch rH2D using a prepare-read exchange and device decryption.
3. Validate the three-entry UB path, including scalar fallback, batching,
   bounds, offsets, concurrent requests, and resource teardown.
4. Add the two host-side RoCE entries if that backend is required; keep the
   same logical request model and adapt the transport/completion backend.
5. Extend notify-bearing APIs, configured resource routes, and graph/task
   replay only with their own ordering and lifecycle validation.

The verification must establish more than correct final bytes: confirm that
wire-facing buffers contain ciphertext, that no plaintext payload route is
selected for a protected request, and that delayed receiver decryption keeps
public completion pending. Check source immutability, changing host input
between reads, multiple chunks, partial submission, failure propagation,
and safe reclamation. If authenticated encryption is added, include altered
ciphertext/metadata and rejected authentication in the failure cases.

No source implementation, build, or hardware experiment was performed for
this document. The interception counts were checked against the inspected
public declarations and HIXL call sites.

## Source references

[runtime-route]: /home/zlj/ub/runtime_enc/AscendCCv2-Runtime/src/runtime/core/src/memory/secure_dma/secure_dma_route.cc#L80
[runtime-fixed]: /home/zlj/ub/runtime_enc/AscendCCv2-Runtime/src/runtime/core/src/memory/secure_dma/secure_dma_fixed.cc#L27
[device-fixed]: /home/zlj/ub/runtime_enc/AscendCCv2-AICPU/ms_kernels/src/secure_dma/secure_dma_fixed_device.cc#L39
[runtime-stream]: /home/zlj/ub/runtime_enc/AscendCCv2-Runtime/src/runtime/core/src/stream/stream.cc#L1963
[runtime-crypto]: /home/zlj/ub/runtime_enc/AscendCCv2-Runtime/src/runtime/core/src/memory/secure_dma/secure_dma_crypto.cc#L118
[runtime-transport]: /home/zlj/ub/runtime_enc/AscendCCv2-Runtime/src/runtime/core/src/memory/secure_dma/secure_dma_memory.cc#L17
[jfc-proposal]: /home/zlj/ub/runtime_enc/iter-007-jfc-completion-notification-design.md
[hcomm-write]: ../../hcomm/src/base_comm/primitives/api_c_adpt/aicpu_ts_primitives_c_adpt.cc#L433
[hcomm-batch]: ../../hcomm/src/base_comm/primitives/api_c_adpt/aicpu_ts_primitives_c_adpt.cc#L847
[hcomm-write-notify]: ../../hcomm/src/base_comm/primitives/api_c_adpt/aicpu_ts_primitives_c_adpt.cc#L590
[host-nbi]: ../../hcomm/src/base_comm/primitives/api_c_adpt/cpu_primitives_c_adpt.cc#L575
[primitive-header]: ../../hcomm/include/hcomm_primitives.h#L231
[primitive-build]: ../../hcomm/src/base_comm/primitives/api_c_adpt/CMakeLists.txt#L25
[batch-types]: ../../hcomm/src/legacy/ascend950/unified_platform/resource/transport/aicpu/ub_transport_lite_impl.cc#L1011
[resource-header]: ../../hcomm/include/hcomm_res.h#L21
[channel-header]: ../../hcomm/include/hcomm_channel.h#L157
[memory-adapter]: ../../hcomm/src/base_comm/primitives/api_c_adpt/hcomm_mem_c_adpt.cc#L35
[aicpu-fences]: ../../hcomm/src/base_comm/primitives/api_c_adpt/aicpu_ts_primitives_c_adpt.cc#L1099
[host-fences]: ../../hcomm/src/base_comm/primitives/api_c_adpt/cpu_primitives_c_adpt.cc#L760
[task-cache]: ../../hcomm/src/base_comm/primitives/aicpu/aicpu_task_cache_c_adpt.cc#L22
[ccu-primitives]: ../../hcomm/include/ccu/ccu_primitives.hpp#L185
[hixl-kernel]: https://gitcode.com/cann/hixl/blob/49bd2a256b728544c1e12df555c545670f9ab51a/src/ops/hixl_kernel/hixl_batch_transfer.cc#L36
[hixl-host]: https://gitcode.com/cann/hixl/blob/49bd2a256b728544c1e12df555c545670f9ab51a/src/hixl/cs/hixl_cs_client.cc#L481
[hixl-proxy]: https://gitcode.com/cann/hixl/blob/49bd2a256b728544c1e12df555c545670f9ab51a/src/hixl/proxy/hcomm_proxy.h#L20
[hixl-completion]: https://gitcode.com/cann/hixl/blob/49bd2a256b728544c1e12df555c545670f9ab51a/src/hixl/cs/hixl_cs_client.cc#L895
[hixl-control]: https://gitcode.com/cann/hixl/blob/49bd2a256b728544c1e12df555c545670f9ab51a/src/ops/hixl_kernel/hixl_batch_transfer.cc#L105
[hixl-dispatch]: https://gitcode.com/cann/hixl/blob/49bd2a256b728544c1e12df555c545670f9ab51a/src/hixl/cs/hixl_cs_client.cc#L1028
