# The Ascend buff subsystem: XSMEM, cross-process shared memory, and Mbuf

Grounded in the tree at `~/ascend_stack` (driver + runtime, 910B/950 era). Analysis date: 2026-08-27.

**Provenance caveat.** This repo contains the **host-side kernel driver** and the **userspace HAL** only. The device-side driver is closed source. The buff HAL sources here are compiled with `DRV_HOST` (`driver/src/ascend_hal/buff/dc/CMakeLists.txt:35, 98`), so what we can read is the host build. Statements about the device-side backing are inferred from the shared UAPI header and from device-side *callers* that live in `runtime/src/aicpu_sched`; those are marked.

Companion document: [`queue_subsystem_host_driver_tdt_and_ub.md`](./queue_subsystem_host_driver_tdt_and_ub.md). buff and queue are a matched pair and are best read together.

Also in this set: [`dms_subsystem_urd_dispatch_and_control_plane.md`](./dms_subsystem_urd_dispatch_and_control_plane.md) — the control plane, where DMS's design is contrasted with this module's.

---

## 1. The one-sentence version

buff makes **one region of memory addressable at the same virtual address in several processes**, so that "sending" a payload between them costs the movement of a single pointer. It transports nothing itself; the queue subsystem does the handing over.

Two things are called "buff", exactly mirroring the queue split:

| Path | What it is |
|---|---|
| `driver/src/ascend_hal/buff/` | Userspace HAL (`libascend_hal`) — the `halBuff*` / `halGrp*` / `halMbuf*` API surface |
| `driver/src/sdk_driver/buff/` | Kernel module (`asdrv_buff`), internally named **XSMEM** |

`asdrv_buff` is built for both 910B and 950 (`driver/cmake/config/driver_config/driver_config_ascend910B.cmake:26`, and the 950 equivalent).

---

## 2. The kernel module manages bookkeeping, never memory

This is the single most important structural fact, and it is easy to get backwards.

```
dc/command/ioctl/buff_ioctl.h    shared UAPI (SPDX GPL-2.0 WITH Linux-syscall-note), 197 lines
dc/buff_host/                    the whole implementation, ~5.2 kLOC
  xsmem_framework.c              2430 ln  pool/task lifecycle, ioctl dispatch, fops
  xsmem_framework_host.c                  host entry, algo registration
  xsmem_algo_vma.c                422 ln  the offset allocator
  xsmem_algo_cache_vma.c                  same, for per-device cache pools
  multi_rbtree.c                          rbtree allowing duplicate keys
  xsmem_feature_check.c                   runtime feature bitmap
  xsmem_prop.c / xsmem_res_idr.c / xsmem_proc_fs.c / xsmem_ns_adapt.c
```

Built on the same `ka_*` kernel-adaptation shim as the queue module (`sdk_driver/kernel_adapt`).

### It refuses to map anything

```c
static int xsmem_mmap_not_support(ka_file_t *file, ka_vm_area_struct_t *vma)
{
    return -ENOTSUPP;
}

static ka_file_operations_t xsmem_fops = {
    .owner          = KA_THIS_MODULE,
    .open           = xsmem_open,
    .release        = xsmem_release,
    .unlocked_ioctl = xsmem_ioctl,
    .mmap           = xsmem_mmap_not_support,
};
```
`xsmem_framework.c:2331-2342`

The char device cannot be mmap'd. Whatever supplies the pages, it is not this module.

### The allocator hands out integers

`xsmem_algo_vma.c` is a pure offset allocator over an abstract range. Two rbtrees — `idle_size_tree` and `idle_va_tree` (`:30-31`) — support best-fit allocation and coalescing free. The ops table is small:

```c
static struct xsm_pool_algo vma_algo = {
    .num             = XSMEM_ALGO_VMA,
    .name            = "vma_algo",
    .xsm_pool_init   = vma_algo_pool_init,
    .xsm_pool_free   = vma_algo_pool_free,
    .xsm_pool_show   = vma_algo_pool_show,
    .xsm_block_alloc = vma_algo_block_alloc,
    .xsm_block_free  = vma_algo_block_free,
};
```
`xsmem_algo_vma.c:410-418`

No page allocation, no `alloc_pages`, no `vm_insert_page`. It is a range allocator and nothing more.

### Which is why block handles are offsets

```c
struct xsm_block_alloc_arg {
    unsigned long flag;   /* input */
    unsigned long size;   /* input */
    unsigned long offset; /* output */
    unsigned int blkid;
};
```
`buff_ioctl.h:26-31`

An allocation returns an **offset**, not a pointer. Section 3 explains why that is sufficient.

### It registers as a davinci sub-module, not its own /dev node

```c
int xsmem_dev_reg_external(void)
{
    /* davinci_recycle can't call mg_sp_free, because davinci_recycle is kernel thread,
       which is not in the sp group */
    xsmem_fops.release = NULL;
    ret = drv_davinci_register_sub_module(DAVINCI_XSMEM_SUB_MODULE_NAME, &xsmem_fops);
    ...
    ret = drv_ascend_register_notify(DAVINCI_XSMEM_SUB_MODULE_NAME, &xsmem_notifier_ops);
```
`xsmem_framework.c:2367-2381`

Contrast the queue module, which owns `/dev/hi-queue-manage` outright. Two details matter here:

- `xsmem_fops.release` is deliberately **nulled** on this path. Cleanup instead arrives through a notifier (`xsmem_notifier_release`, `:2358`) so that it runs in the exiting task's own context. The comment states the reason: the generic `davinci_recycle` kernel thread is not a member of the share-pool group and therefore cannot call `mg_sp_free` on its behalf.
- A misc-device path (`xsmem_dev_reg_internal`, `:2345`) also exists, gated by `BUFF_FEATURE_EXTERNAL_CDEV`.

---

## 3. Where the memory actually comes from

Entirely userspace, in `driver/src/ascend_hal/buff/dc/share_fd_adp/` — the "share fd adapter". Three steps.

### 3.1 An anonymous in-RAM file

```c
static drvError_t create_shr_mem_fd(const char *name, unsigned long long max_mem_size)
{
    int fd = (int)syscall(SYS_memfd_create, name, 0);
    ...
    ret = ftruncate(fd, (long)max_mem_size);
    set_grp_shr_mem_fd(fd);
```
`grp_mng.c:751-767`

`memfd_create` produces a file with no filesystem path — it exists only in RAM and can only be reached through its descriptor. Compared to `shm_open` or a file under `/dev/shm`, this means no name for anyone to guess, no stale object to clean up, and no ambient permissions: **possession of the fd is the capability**.

### 3.2 The fd travels over a unix socket

```c
cmsg->cmsg_level = SOL_SOCKET;
cmsg->cmsg_type  = SCM_RIGHTS;
cmsg->cmsg_len   = CMSG_LEN(sizeof(int));
...
*(int *)CMSG_DATA(CMSG_FIRSTHDR(&msg)) = get_grp_shr_mem_fd();
sendmsg(sfd, &msg, 0);
```
`grp_mng.c:414-434`, received and installed at `:440-461`

`SCM_RIGHTS` is the kernel's descriptor-passing mechanism: the receiver gets a fresh fd number in its own table referring to the same underlying file object. Hence the directory name.

### 3.3 Every member maps it at the same fixed address

```c
if (is_authed_read(attr))  prot |= PROT_READ;
if (is_authed_write(attr)) prot |= PROT_WRITE;
if (prot == 0) { ... return DRV_ERROR_PARA_ERROR; }
...
addr = mmap((void *)BUFF_MEM_BASE, max_mem_size, (int)prot, MAP_SHARED, mem_fd, 0);
if (addr == MAP_FAILED) { ... }
if (addr != (void *)BUFF_MEM_BASE) {
    buff_err("pool_id %d mem_fd %d mmap addr %p error\n", pool_id, mem_fd, addr);
    (void)munmap(addr, max_mem_size);
    return DRV_ERROR_INNER_ERR;
}
(void)madvise(addr, max_mem_size, MADV_DONTDUMP);
```
`buff_mng.c:145-190` (the mmap at `:173`), with `#define BUFF_MEM_BASE 0x200000000000ULL` at `buff_mng.h:19`

Three things worth noting:

- The address is a **hint plus an assertion**, not `MAP_FIXED` — identical in spirit to the SVM master's mapping in `svm/v3/assign/va_allocator/va_reserve.c:408`. Fail loudly rather than silently landing somewhere else.
- The mapping's protection is derived from the kernel-arbitrated `GroupShareAttr`. A read-only member **physically cannot** write the pool; that is MMU enforcement, not convention.
- `MADV_DONTDUMP` keeps a potentially huge shared pool out of core dumps.

Because the base is identical in every member, `BUFF_MEM_BASE + offset` denotes the same byte everywhere. That is the whole trick, and it is what makes an offset a legitimate cross-process handle.

### The VA map is deliberately partitioned

| Base | Owner | Source |
|---|---|---|
| `0x100000000000` | SVM device default | `svm/v3/assign/va_allocator/va_dev_default_allocator.c:21` |
| `0x200000000000` | buff pool | `buff/dc/share_fd_adp/buff_mng.h:19` |
| `0x300000000000 + 16 TB` | SVM non-device default | `svm/v3/assign/va_allocator/va_non_dev_default_allocator.c:19` |

Three subsystems each claim a fixed-address region, chosen far enough apart that they never collide.

---

## 4. Why any of this needs a kernel module

Given that userspace does the memfd, the fd passing and the mmap, it is reasonable to ask what the kernel is for. Five things userspace cannot do for itself:

**1. Rendezvous between processes that never talk.** A process asks *"which pools is pid N in?"* — `QueryProcBuffInfo(pid, grpInfos)` (`aicpusd_queue_event_process.cpp:767`, `hwts_kernel_queue.cpp:102`), which is `XSMEM_POOL_TASK_QUERY`. The kernel is the naming service because it is the only party that already knows about every pid. A userspace scheme would need a rendezvous daemon and a trust story for it.

**2. Permission arbitration.** `GroupShareAttr {admin, read, write, alloc}` — see `ALL_ATTR_FOR_GROUP = {1U, 1U, 1U, 1U, 0U}` at `aicpusd_queue_event_process.cpp:22`. A member may be allowed to read a pool but not allocate from it. The participants are mutually untrusting (distinct jobs on a shared accelerator), so this cannot be an honor system — and as shown above, the decision is cashed out in the `prot` argument of the joiner's own mmap.

**3. Cross-process refcounting.** `XSMEM_BLOCK_GET` / `XSMEM_BLOCK_PUT` are separate ioctls from `ALLOC` / `FREE`. A block travelling producer → scheduler → consumer must be freed exactly once, and the count has to survive any one participant dying mid-flight.

**4. Cleanup on crash.** If a member segfaults holding blocks, the pool must be reclaimed. That is the notifier path in §2, and the `mg_sp_free` comment is the scar tissue from getting it right.

**5. Quota.** `GroupCfg.maxMemSize` caps a group's total (`driver/pkg_inc/ascend_hal_define.h:349`), with defaults and a floor applied in `get_grp_max_mem_size_from_cfg` (`grp_mng.c:735-749`).

Note that none of the five is memory management. That is exactly consistent with a module that refuses `mmap` and whose allocator only manipulates integers.

---

## 5. The ioctl surface

23 commands, magic `'X'`, `XSMEM_MAX_CMD 24` (`buff_ioctl.h:142-189`).

| Group | Commands |
|---|---|
| Pool lifecycle | `POOL_REGISTER` (1), `POOL_UNREGISTER` (2) |
| Membership | `POOL_TASK_ADD` (3), `POOL_TASK_DEL` (4), `POOL_ATTACH` (5), `POOL_DETACH` (6) |
| Blocks | `BLOCK_ALLOC` (7), `BLOCK_FREE` (8), `BLOCK_GET` (9), `BLOCK_PUT` (10) |
| Discovery | `POOL_ID_QUERY` (11), `POOL_NAME_QUERY` (12), `POOL_TASK_QUERY` (13), `POOL_TASK_ATTR_QUERY` (14), `TASK_POOL_QUERY` (15) |
| Properties | `PROP_OP` (16) — set/get/del, owner is task or task-group |
| Exit handling | `POLL_EXIT_TASK` (17) |
| Device cache | `CACHE_CREATE` (18), `CACHE_DESTROY` (19), `CACHE_QUERY` (20) |
| Misc | `POOL_FLAG_QUERY` (21), `VADDR_CHECK` (22), `FEATURE_GET` (23) |

Pools are registered **by key**, shmget-style:

```c
struct xsm_reg_arg {
    int algo;
    unsigned int priv_flag;
    unsigned long pool_size;
    unsigned int block_size; /* used for blk algorithm */
    unsigned int key_len;
    const char *key;
};
```
`buff_ioctl.h:7-14`

`XSMEM_VADDR_CHECK` (`struct xsm_check_va_arg`, `:112-116`) answers "is this VA inside pool N?" — the kernel-side counterpart of the userspace range check against `buff_base_addr` / `buff_end_addr`.

### Four algorithms, two registered on host

```c
#define XSMEM_ALGO_EMPTY     (-1)
#define XSMEM_ALGO_VMA         0
#define XSMEM_ALGO_SP          1
#define XSMEM_ALGO_CACHE_VMA   2
#define XSMEM_ALGO_CACHE_SP    3
```
`buff_ioctl.h:184-189`

```c
static void xsmem_algo_register(void)
{
    xsmem_register_algo(xsm_get_vma_algo());
    xsmem_register_algo(xsm_get_cache_vma_algo());
}
```
`xsmem_framework_host.c:29-33`

The two SP variants are the Huawei kernel **share pool** (`mg_sp_alloc` / `mg_sp_free`). The comment immediately above the function records the migration: *"not included because their availability is determined at runtime via the feature check system (buff_has_feature / xsmem_feature_check). The CFG_FEATURE_SUPPORT_SP compile-time switch has been replaced by the runtime feature check mechanism."*

### Feature bitmap

```c
enum buff_feature_id {
    BUFF_FEATURE_EXTERNAL_CDEV = 0,
    BUFF_FEATURE_SUPPORT_VMA,
    BUFF_FEATURE_SHARE_LOG,
    BUFF_FEATURE_KA_ALLOC_INTERFACE,
    BUFF_FEATURE_HOST_LOG,
    BUFF_FEATURE_SUPPORT_SP,
    BUFF_FEATURE_SUPPORT_HUGE_PAGE,
    BUFF_FEATURE_NO_SUPPORT_DVPP_MZ,
    BUFF_FEATURE_MAX
};
```
`buff_ioctl.h:122-132`, queried per device via `XSMEM_FEATURE_GET` (`struct xsm_feature_get_arg`, `BUFF_FEATURE_DEV_NUM 1140`)

---

## 6. Mbuf: the object that travels

The user-visible unit is an `Mbuf`, defined at `driver/src/ascend_hal/buff/dc/comm/common/drv_buff_common.h:261` — `total_len`, `data_len`, `datablock`, `data`, `buff_type`, `lock`, `prev`/`next`, `user_data[]`, plus tracing fields (`timestamp`, `record_pid[]`, `record_opt[]`, `record_cur_idx`).

Every allocation carries a **unified buffer header** with guard words, implemented in `drv_buff_unibuff.c` (438 ln):

| Symbol | Meaning |
|---|---|
| `UNI_HEAD_IMAGE 0xAABCDEFF` | head magic |
| `UNI_TAIL_IMAGE 0xFFBCDEAA` | tail magic |
| `UNI_STATUS_IDLE / ALLOC / RELEASE` | lifecycle state |
| `UNI_TYPE_ZONE / MP / LARGE` | which sub-allocator owns it |

with `buff_verify_and_get_head`, `mbuf_verify_and_get_head` and `buff_trace` as the checked accessors. In shared memory that any member can scribble on, a corrupted header must be detected rather than followed.

> **Naming trap.** The "uni" in `drv_buff_unibuff.c` means **unified buffer header**. It is *not* UnifiedBus. Unlike the queue subsystem, which has a genuine UB transport in `queue/dc/agent/client/remote/que_clt_ub/`, buff has no UB client anywhere in this tree.

An mbuf also carries a **private header** alongside its payload, retrieved with `halMbufGetPrivInfo`, enabled per group by `GroupCfg.privMbufFlag = BUFF_ENABLE_PRIVATE_MBUF` (`0x5A5A5B5B`, `ascend_hal_define.h:332`). This is how metadata rides with the data in the same shared block — see the consumer at `operator_kernel_dequeue_base.cpp:72-81`, which pulls shape/end-of-sequence information out of the private head before running the operator.

---

## 7. Who uses it

Across the entire tree, the non-test callers of `halGrpCreate` / `halBuffInit` are in exactly two places:

| Caller | Side | Reference |
|---|---|---|
| `runtime/src/aicpu_sched/**` — the AICPU scheduler | device | `aicpusd_queue_event_process.cpp:690-709`, `hwts_kernel_queue.cpp:119-140`, `aicpusd_resource_manager.cpp:56`, `aicpu_cust_schedule/core/aicpusd_interface_process.cpp:334` |
| `runtime/src/queue_schedule/dgwclient/dgw_client.cpp:198` — the DGW data-gateway client | host | attaches a queue, then `halBuffInit` |

That is the whole list. Not GE, not the host runtime's main path, not PyTorch.

### Typical device-side setup

```
aicpusd starts
  QueryProcBuffInfo(self)                      -> empty, so become master
  halGrpCreate("Aicpusd<ts>", {privMbufFlag = BUFF_ENABLE_PRIVATE_MBUF})
  halGrpAddProc(grp, self, {admin, read, write, alloc})
  halGrpAttach(grp, 0)                         <- memfd mmap'd at BUFF_MEM_BASE here
  halBuffInit(&buffCfg)

a second process (custom AICPU .so, HCCP, model executor)
  aicpusd calls halGrpAddProc(grp, thatPid, ALL_ATTR)   [hwts_kernel_cust_so.cpp:130]
  that process calls halGrpAttach(grp)                  -> same base, same pages

producer:  halMbufAlloc(size, &mbuf); write payload; halQueueEnQueue(dev, qid, mbuf)
consumer:  halQueueDeQueue(dev, qid, &mbuf); halMbufGetPrivInfo(...); run op; halMbufFree(mbuf)
```

Master/slave selection is `GetOrCreateGroup` (`aicpusd_queue_event_process.cpp:748-780`): query own groups, create if none, otherwise attach as slave — and a slave is required to own exactly one group (`AttachGroupForSlave:718-746`).

### API layering

```
acltdtAllocBuf / acltdtFreeBuf / acltdtGetBufData          runtime/include/external/acl/acl_tdt_queue.h:233+
  -> QueueProcessor{Ccpu,Sp}::acltdtAllocBuf               queue_process_ccpu.cpp:185, queue_process_sp.cpp:195
     -> QueueProcessor::acltdtAllocBufData                 queue_process.cpp:469
        -> rtMbufAllocEx / rtMbufAlloc                     api_c_mbuf.cc:49, :62
rtBuffAlloc / rtMemGrpCreate / rtMemGrpAddProc /
rtMemGrpAttach / rtBuffGetInfo                             api_c.cc:2976, :3021, :3057, :3069, :3033
  -> NpuDriver::BuffAlloc / MemGrpCreate / ...             npu_driver_queue.cc:38-345
     -> halBuffAlloc / halGrpCreate / ...                  ascend_hal_base.h:3864+
        -> ioctl(magic 'X')                                buff_manage_kernel_api.c:231, 285, 340, 380
```

`NpuDriver::BuffAlloc` and `NpuDriver::MemQueueEnQueueBuff` live in the **same file** (`npu_driver_queue.cc`). That is not filing sloppiness — it reflects that buff and queue are one feature seen from two angles.

### The ACL-level API is device-only

```c
aclError QueueProcessorHost::acltdtAllocBuf(const size_t size, const uint32_t type, acltdtBuf* const buf)
{
    (void)(size); (void)(type); (void)(buf);
    ACL_LOG_ERROR("[Unsupported][Feature]acltdtAllocBuf is not supported in this version. Please check.");
    ...
    return ACL_ERROR_FEATURE_UNSUPPORTED;
}
```
`queue_process_host.cpp:344-354`

The processor is selected by run mode (`acl_tdt_queue_manager.cpp:27-56`):

| Run mode | Environment | Selected by |
|---|---|---|
| `ACL_HOST` | `ACL_ENV_HOST` | — |
| `ACL_DEVICE` | `ACL_ENV_DEVICE_CCPU` | `MM_ENV_SHAREGROUP_PRECONFIG` unset |
| `ACL_DEVICE` | `ACL_ENV_DEVICE_SP` | `MM_ENV_SHAREGROUP_PRECONFIG` set — a scheduler pre-created the group |

This is an ACL policy decision, **not** a driver capability limit: `halBuffAlloc` and friends exist in the host build, and `dgw_client.cpp` uses them.

---

## 8. Scope boundary: same-side only, never host to device

buff works host↔host and device↔device. It never crosses the PCIe boundary, and the code says so explicitly.

`halQueueEnQueue` dispatches on deployment type (`queue/dc/interface/queue_interface.c:442-453`):

```c
typedef enum queue_deployment_type {
    CLIENT_QUEUE = CLIENT_QUEUE_DEPLOY,   /* queue lives on the remote side */
    LOCAL_QUEUE  = LOCAL_QUEUE_DEPLOY,    /* queue lives on this side */
    DEPLOYMENT_EXTER_MAX_TYPE,
    INTER_DEV_QUEUE = DEPLOYMENT_EXTER_MAX_TYPE,
    INVALID_INTER_DEV_QUEUE,
    DEPLOYMENT_MAX_TYPE,
} QUEUE_DEPLOYMENT_TYPE;
```
`queue/dc/interface/queue_interface.h:22-29`

Registrars: `LOCAL_QUEUE` <- `core/queue.c:2377`; `CLIENT_QUEUE` <- `que_clt_pcie/queue_client.c:1776` or `que_clt_ub/que_clt_ub.c:894`; `INTER_DEV_QUEUE` <- `agent/common/que_inter_dev_ini.c:1099`.

And on the remote client path the mbuf flavours are stubbed out unconditionally:

```c
STATIC drvError_t queue_en_queue_client(unsigned int dev_id, unsigned int qid, void *mbuf)
{
    (void)dev_id; (void)qid; (void)mbuf;
    return DRV_ERROR_NOT_SUPPORT;
}
STATIC drvError_t queue_de_queue_client(unsigned int dev_id, unsigned int qid, void **mbuf)
{
    (void)dev_id; (void)qid; (void)mbuf;
    return DRV_ERROR_NOT_SUPPORT;
}
```
`que_clt_pcie/queue_client.c:815-829`

An mbuf pointer physically cannot cross PCIe: the memfd is host RAM, the device cannot map it, and `BUFF_MEM_BASE + offset` would denote nothing on the far side.

| Participants | Mechanism | Enqueue API | Deployment |
|---|---|---|---|
| host proc ↔ host proc | buff group: memfd + SCM_RIGHTS + fixed-VA mmap | `halQueueEnQueue(mbuf)` | `LOCAL_QUEUE` |
| device proc ↔ device proc | buff group, device-side backing *(closed; the SP / `mg_sp_*` algorithm is the hint)* | `halQueueEnQueue(mbuf)` | `LOCAL_QUEUE` |
| host ↔ device | **not buff** — kernel pins pages and DMAs | `halQueueEnQueueBuff(iovec)` | `CLIENT_QUEUE` |

The third row is the host kernel driver work documented in the queue companion: `queue_get_user_pages`, `queue_make_dma_list`, the three DMA variants. It exists precisely because buff cannot reach across.

---

## 9. buff vs SVM vs queue

The three subsystems are frequently confused because all three deal in addresses.

| | Problem solved | Mechanism | Boundary crossed |
|---|---|---|---|
| **SVM** | one address means the same thing to host **and** device | fixed base (v2) or negotiated VA (v3), page tables on both sides | host ↔ device |
| **buff** | one address means the same thing to two processes on the **same** side | memfd + SCM_RIGHTS + fixed-VA `MAP_SHARED` | process ↔ process |
| **queue** | notify a peer and hand over ownership | local: pointer pass; remote: HDC + DMA descriptor list | either, depending on deployment |

buff and SVM claim deliberately disjoint VA regions (§3) so they can coexist in one process.

The buff/queue division of labour in one line: **buff sets up the stage; queue is the handoff.** A 100 MB tensor "sent" through a local queue costs one 8-byte pointer, because buff already made those bytes addressable at both ends.

---

## 10. Quick reference

| Question | Answer |
|---|---|
| Kernel module name | `asdrv_buff`, internally XSMEM |
| Char device | none of its own — davinci sub-module `DAVINCI_XSMEM_SUB_MODULE_NAME`; misc device behind `BUFF_FEATURE_EXTERNAL_CDEV` |
| ioctl magic / count | `'X'`, 23 commands (`XSMEM_MAX_CMD 24`) |
| Does the module allocate pages? | No. `xsmem_mmap_not_support`; the allocator only produces offsets |
| Where do the pages come from? | userspace `memfd_create` + `ftruncate`, shared via `SCM_RIGHTS` |
| Shared VA base | `BUFF_MEM_BASE 0x200000000000` |
| Block handle | `unsigned long offset` from the pool base |
| Permission model | `GroupShareAttr {admin, read, write, alloc}`, enforced via mmap `prot` |
| Algorithms | VMA, SP, CACHE_VMA, CACHE_SP; host registers VMA + CACHE_VMA, SP is runtime feature-checked |
| Payload object | `Mbuf` with unified header (`UNI_HEAD_IMAGE` / `UNI_TAIL_IMAGE`) and optional private head |
| Crash cleanup | notifier, not `fops.release` (which is nulled on the davinci path) |
| Real consumers in tree | `runtime/src/aicpu_sched/**` (device), `queue_schedule/dgwclient` (host) |
| ACL entry points | `acltdtAllocBuf` / `acltdtFreeBuf` / `acltdtGetBufData` — device run mode only |
| Crosses PCIe? | Never. `queue_en_queue_client` / `queue_de_queue_client` return `DRV_ERROR_NOT_SUPPORT` |
| "unibuff" means | unified buffer **header** — not UnifiedBus |
