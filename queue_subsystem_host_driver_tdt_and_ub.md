# The Ascend queue subsystem: host kernel driver, TDT, and PCIe vs UB

Grounded in the tree at `~/ascend_stack` (driver + runtime, 910B/950 era). Analysis date: 2026-08-19.

**Provenance caveat.** This repo contains the **host-side kernel driver** and the **userspace HAL** only. The device-side driver is closed source. Several conclusions below are inferred from a shared UAPI header whose other half we cannot read; those are marked.

---

## 1. Two different things are called "queue"

| Path | What it is |
|---|---|
| `driver/src/ascend_hal/queue/` | Userspace HAL (`libascend_hal`) — the `halQueue*` API surface |
| `driver/src/sdk_driver/queue/` | Host **kernel module** (`asdrv_queue`), char device `/dev/hi-queue-manage` |

The HAL is the portable API. The kernel module implements exactly **one** transport for it: host as a remote client of a device-resident queue over **PCIe**.

---

## 2. The host kernel module

```
command/ioctl/queue_ioctl.h         shared UAPI (SPDX GPL-2.0 WITH Linux-syscall-note)
command/msg/queue_h2d_kernel_msg.h  host->device wire structs
host/                               the host implementation (all that exists here)
host/common/                        channel, dma, context, feature-check, procfs
```

Standalone GPL module built on a `ka_*` kernel-adaptation shim (`sdk_driver/kernel_adapt`) rather than raw Linux APIs — `ka_mm_get_user_pages_fast`, `ka_file_operations_t`, `ka_module_init`. Typical out-of-tree portability layer.

### Only 3 of 9 ioctls are handled here

`drv_queue_ioctl_handlers` (`host/queue_fops.c:678`) populates:

- `QUEUE_HOST_COMMON_OP_CMD` — init/uninit, establishes the HDC session
- `QUEUE_ENQUEUE_CMD` — the real work
- `QUEUE_CTRL_MSG_SEND_CMD` — 4KB control messages

The other six declared in the UAPI — `COPY`, `REPLY_CLIENT`, `SESSION_INIT`, `CTRL_MSG_RECV`, `GET_IOVEC_NUM` — have no handler in this tree. *(Inferred: they are dispatched by the closed device-side driver against the same header.)*

### The enqueue path

`queue_drv_enqueue` (`host/queue_fops.c:404`):

1. Require an HDC session for that devid (`ctx_private->hdc_session[devid]`), set up by `QUEUE_INIT`.
2. Allocate a global serial number.
3. Create a per-transfer "queue channel" (`queue_chan_create`).
4. Walk the user's `buff_iovec` — context base + N data buffers — into the channel (`queue_drv_vector_add:349`).
5. `queue_make_dma_list` (`host/common/queue_dma.c:342`): pin user pages, convert to physical, merge adjacent blocks, `dma_map_page` for the device.
6. Send the scatter-gather descriptor list over **HDC** with a published scheduler event so the device-side CP process wakes (`queue_chan_send` -> `queue_hdc_msg_send`).
7. **Block** on `queue_chan_wait(que_chan, QUEUE_HOST_WAIT_MAX_TIME)`.
8. On reply: unmap, unpin, destroy.

**The host never copies the payload.** It pins its own pages in place, hands over descriptors, and the **device is the DMA master** — it pulls when its consumer is ready. The reply (`queue_reply_complete_msg`) returns `dma_node_num` confirming what was consumed. `QUEUE_ENQUEUE` is fully synchronous to the caller.

### Three DMA variants

`queue_make_dma_list` branches three ways:

| Variant | Trigger | Behaviour |
|---|---|---|
| Normal | default | pin -> phys -> `dma_map_page` |
| SVA | `queue_get_dma_sva_enable_platform()` | no pinning; one block holding the page-aligned **VA** (`queue_fill_dma_blks_sva:165`), no `dma_map_page`. Device SMMU resolves the host VA directly. |
| HCCS-VM passthrough | `queue_is_hccs_vm_through_scene()` | translate to "pm pa" via `queue_pa_blks_to_pm_pa_blks`, also skips `dma_map_page` |

### SVM interaction (cross-module dependency)

`queue_get_user_pages` (`queue_dma.c:230`) cannot treat SVM pointers as ordinary memory:

```c
svm_flag = is_svm_addr(vma, dma_list->va);
/* memory remap by remap_pfn_rang, get user page fast can not get page addr */
if (svm_flag == true) {
    ret = devmm_get_pages_list(...);      /* into the devmm kernel module */
} else {
    ret = queue_get_user_pages_fast(...); /* ordinary GUP */
}
```

SVM VAs come from `remap_pfn_range` on the SVM char device, so no `struct page` is reachable by a normal GUP walk. This is a **hard runtime dependency of `asdrv_queue` on the devmm/SVM kernel module**.

### Built-in cross-boundary instrumentation

`queue_status_time_type` (`command/ioctl/queue_ioctl.h:13`) is a 14-point timeline spanning both sides:

- host: `START_MAKE_DMA_LIST` -> `END_MAKE_DMA_LIST` -> `END_HDC_SEND` -> `END_WAIT_REPLY`
- device: `START_SUBMIT_EVENT` -> `END_MAKE_DMA_LIST` -> `END_DMA_COPY` -> `END_REPLY`

Device timestamps ride back in `queue_reply_complete_msg.time_record[]`; the whole thing is exposed through procfs (`host/common/queue_proc_fs.c`). Ready-made latency attribution across PCIe.

---

## 3. Who calls it

Only the **`*Buff` variants** reach this module. `halQueueEnQueueBuff`/`DeQueueBuff` take `struct buff_iovec*` + timeout, matching `struct queue_ioctl_enqueue`. The mbuf variants (`halQueueEnQueue(devId, qid, mbuf)`) take an opaque `Mbuf*` and go elsewhere.

### Path A — ACL -> Runtime (the normal path)

```
acltdtSendTensor / acltdtSendTensorV2 / acltdtReceiveTensor
    runtime/src/acl/acl_tdt_channel/tensor_data_transfer.cpp:642, 736
acltdtEnqueueData / dequeue
    runtime/src/acl/acl_tdt_queue/queue_process.cpp:608, 657
        |
rtMemQueueEnQueueBuff / rtMemQueueDeQueueBuff    api_c.cc:2773, 2785
        |   (declared pkg_inc/runtime/rt_external_mem.h:1023)
ApiImpl::MemQueueEnQueueBuff                     api_impl.cc:6531
        |
NpuDriver::MemQueueEnQueueBuff                   npu_driver_queue.cc:699
        |
halQueueEnQueueBuff                              (libascend_hal)
        |
ioctl(QUEUE_ENQUEUE_CMD)   que_clt_pcie/queue_client_kernel_api.c:167
```

Session setup takes the same route: `NpuDriver::MemQueueInit` -> `halQueueInit` (`dc/interface/queue_interface.c:215`) -> `queue_host_common_queue_init` -> `QUEUE_HOST_COMMON_OP_CMD/QUEUE_INIT`.

### Path B — Queue Schedule / DGW, bypassing runtime

`runtime/src/queue_schedule/` calls the HAL **directly**, with no `rt*` layer:

- `server/entity_manager/client_entity.cpp:280` (enqueue), `:160` (dequeue)
- `dgwclient/dgw_client.cpp:661, 693`

A standalone service, not part of `libruntime`. Easy to miss when auditing the ioctl contract from the runtime side alone.

### NOT on this path

| Component | Why not |
|---|---|
| AI CPU scheduler (`runtime/src/aicpu_sched/.../operator_kernel/{enqueue,dequeue}/`) | Calls the same HAL API, but runs on the **device** ARM cores — hits the device half of the UAPI, never `/dev/hi-queue-manage` |
| mbuf path (`queue_manager.cpp:449`, `channel_entity.cpp:849`, `dgw_client.cpp:546`) | `halQueueEnQueue` with `Mbuf*` — buff-pool memory, no user-page pinning |
| torch_npu / vLLM-Ascend | Only a link-time stub `third_party/acl/libs/acl_tdt.cpp`; **no real acltdt call** anywhere in `torch_npu/`. A normal PyTorch train/infer run never touches this driver |
| GE | No acltdt usage found |

---

## 4. Why TDT needs its own channel (and not `aclrtMemcpy`)

`acltdtSendTensorV2` (`tensor_data_transfer.cpp:617`) serializes a dataset into a `vector<rtMemQueueBuffInfo>` where **each tensor contributes two entries**:

```c
// TensorDataitemSerialize, :560
qBufVec.push_back(qItem);                       // control: ItemInfo + dims[], 64B-aligned
if (dataLen > 0) qBufVec.push_back(tmpQItem);   // raw tensor bytes
```

all handed to a single `rtMemQueueEnQueueBuff`. Six reasons that shape cannot collapse into a memcpy:

1. **Self-describing data.** `ItemInfo` (`tensor_data_transfer.h:105`, `#pragma pack(1)`) carries `version`, `dataType`, `tensorType`, `dimNum`, `dataLen`, `curCnt`/`cnt`, `sliceNum`/`sliceId`. Shape and dtype travel with the bytes.
2. **No destination address exists.** Memcpy needs a known destination of known size. The device-side `GetNext` consumes from a queue whose head moves; the producer does not know where or how much room is free. That is what a qid abstracts.
3. **Backpressure is the point.** `acltdtCreateChannelWithCapacity` sets `attr.depth`, `flowCtrlFlag`, `flowCtrlDropTime`, `overWriteFlag` (`:1052`); `acltdtSendTensor` returns `ACL_ERROR_RT_QUEUE_FULL` (`:643`). Memcpy has no full/empty/blocking.
4. **In-band control signals.** `acltdtTensorType` (`acl_tdt.h:20`) = `TENSOR`, `END_OF_SEQUENCE`, `ABNORMAL`, `SLICE_TENSOR`, `END_TENSOR`. Epoch end and feeder errors must arrive in order with the data.
5. **One round trip for N tensors.** N tensors -> up to 2N iovec entries -> **one** DMA descriptor list, **one** HDC message. N separate memcpys would be N PCIe round trips. This is exactly why the ioctl takes an iovec.
6. **Named, cross-process, addressable.** Channels resolve by name (`halQueueGetQidbyName`), grant to other pids (`halQueueGrant`), export/import across devices. Memcpy targets one address in one address space.

Summary: TDT is a **typed, bounded, named, flow-controlled stream**; `aclrtMemcpy` is an unstructured byte move between two pre-agreed addresses.

The only naming convention in the code is the `TF_RECEIVE_` prefix (`tensor_data_transfer.h:80`) — TensorFlow's receive channel. TDT's origin is the framework dataset input pipeline.

### Not every TDT call reaches the driver

Two generations sit behind the same ACL API, and only one is queue-backed. Every entry
point branches on `handle->isTdtProcess` (`tensor_data_transfer.cpp:1103, 1127, 1155, 1168, 1207, 1252`):

| | `acltdtCreateChannel(dev, name)` | `acltdtCreateChannelWithCapacity(dev, name, cap)` |
|---|---|---|
| `isTdtProcess` | `true` (ctor default) | `false` (`:1051`) |
| Transport | `libtsdclient.so` — `TdtHostInit` / `TdtHostPushData` | `rtMemQueueCreate` -> `rtMemQueueEnQueueBuff` |
| TF-specific | yes — `TF_RECEIVE_` prefix triggers `TdtHostPreparePopData()` | none |
| Reaches `/dev/hi-queue-manage` | **no** | **yes** |

The legacy TF path goes through `libtsdclient.so`, which is not in this tree. Only the
capacity/queue-backed variant reaches the kernel module analysed in this document. The
runtime's own samples use the modern variant with generic names
(`runtime/example/2_advanced_features/tdt_channel/0_simple_channel/main.cpp:68`).

### PyTorch does not use TDT

In the torch_npu checkout at `~/ascend_stack/pytorch` there are **zero** acltdt call sites.
What exists is non-functional: `third_party/acl/libs/acl_tdt.cpp` (a CI link stub compiled
into a fake `libacl_tdt_channel.so` by `build_stub.sh:17`) and an auditwheel `--exclude`
entry at `setup.py:623`. It appears in no `target_link_libraries`.

Structural reason: TDT feeds a **device-resident graph**. GE registers `GetNext` /
`DynamicGetNext` / `DynamicGetNextV2` (`ge/base/common/types.cc:289`) and carries
`ATTR_GETNEXT_SINK_DYNMAIC` for sink mode (`om2_model_adapter.cc:195`) — in TF/MindSpore
graph sink mode the training loop runs on device, so the host pipeline must push into a
queue the device pulls from. Eager PyTorch has no such consumer: the DataLoader runs in
host Python and each iteration is an ordinary H2D copy onto a stream, host-driven every
step.

### What TDT is NOT

There is **no print/dump or device-log use of TDT anywhere in this tree.** Device logs are a separate subsystem:

- `driver/src/ascend_hal/msnpureport/` with `adcore/{hdc,log,device,protocol}` — log collection over HDC
- `driver/src/ascend_hal/bbox/` — black-box exception/dump manager

Neither touches the queue driver. The `QUEUE_FEATURE_SHARE_LOG` / `QUEUE_FEATURE_HOST_LOG` flags in `queue_chip_feature_map.h` concern the queue module's **own** logging, and are not referenced by any `.c` in this tree.

---

## 5. PCIe vs UB — is queue used on UB?

**Yes, queues are used on UB. No, this kernel module is not.**

The queue abstraction (qid, enqueue/dequeue, subscribe, grant) is transport-independent. `halQueueEnQueueBuff` (`dc/agent/client/remote/que_clt_api.c:63`) does parameter checks then dispatches through a vtable:

```c
ret = g_que_clt_api->api_enque_buf(devId, qid, vector, timeout);
```

`g_que_clt_api` is bound once at library load by a constructor (`que_clt_module_init:200`) from `que_clt_get_con_type()`:

```c
case QUE_CON_PCIE: g_que_clt_api = que_clt_pci_get_api(); break;   // :164
case QUE_CON_UB:   g_que_clt_api = que_clt_ub_get_api();  break;   // :168
```

### The two implementations

| | PCIe | UB |
|---|---|---|
| Source | `que_clt_pcie/` (1946 lines) | `que_clt_ub/` (2283 lines) |
| Enqueue | `queue_enqueue_cmd` -> `ioctl(QUEUE_ENQUEUE_CMD)` | `que_ctx_enque` -> `que_chan_pkt_send` -> `que_ini_pkt_send` |
| Transport | HDC message + kernel-built DMA descriptor list | **URMA jetty** (`que_jetty.h`, jfs/jfr indices, `que_ini_proc`/`que_tgt_proc`) |
| Kernel module | `/dev/hi-queue-manage` required | **zero ioctls** — `grep -rn ioctl que_clt_ub/*.c` returns nothing |

The UB path is userspace RMA end to end: initiator/target jetty pairs, no `asdrv_queue` involvement.

### Build gating

`src/ascend_hal/queue/dc/CMakeLists.txt:10-16` — UB support requires **all three**:

```cmake
if (${PRODUCT} STREQUAL ascend950)
  if (${CMAKE_HOST_SYSTEM_PROCESSOR} STREQUAL aarch64)
    if(${ENABLE_UBE})
      set(support_ub "ascend950")
```

So UB is 950-only, aarch64-host-only, and opt-in via `ENABLE_UBE`. Everything else compiles `que_clt_get_con_type()` down to a hard `return QUE_CON_PCIE` (`que_clt_api.c:154`).

### The kernel module is still built for both

`asdrv_queue` appears in `driver_config_ascend910B.cmake:34` **and** `driver_config_ascend950.cmake:32`. That is consistent: a 950 host on x86, or a 950 build with `ENABLE_UBE` off, still takes the PCIe path.

The kernel module even knows about UB — `queue_event_try_mcast` (`queue_fops.c:692`) early-returns when `devdrv_get_connect_protocol(devid) == CONNECT_PROTOCOL_UB`, skipping event multicast. So the module can be loaded in a UB-capable system and defers on the parts UB handles itself.

**Bottom line:** on a UB deployment the queue *feature* is fully alive; `sdk_driver/queue` is dead code on that path.

---

## 6. Quick reference

| Question | Answer |
|---|---|
| Char device | `/dev/hi-queue-manage` |
| Module target | `asdrv_queue` (910B + 950) |
| Handled ioctls | `HOST_COMMON_OP`, `ENQUEUE`, `CTRL_MSG_SEND` (3 of 9) |
| Who DMAs | The **device** pulls; host only pins + describes |
| Enqueue semantics | Synchronous — blocks until device replies |
| SVM pointers | Routed to `devmm_get_pages_list`, not GUP |
| Runtime entry | `NpuDriver::MemQueueEnQueueBuff` (`npu_driver_queue.cc:699`) |
| Public API | `acltdtSendTensor` / `acltdtEnqueueData`, plus QS/DGW direct HAL calls |
| UB | Uses URMA jetty in userspace; never this module |
| PyTorch / vLLM | Do not use it at all |
