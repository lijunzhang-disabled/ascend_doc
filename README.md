# ascend analysis docs

Notes on the Ascend driver + CANN sources under `~/ascend_stack`, written while reading the code.

| Document | Covers |
|---|---|
| [ub_h2d/README.md](./ub_h2d/README.md) | UB host-to-device summary and ordered deep dives: transport and memory, memcpy, TRS submission, queue/HDC, kernel control/boot, ACL/runtime dispatch, PCIe/UB mapped host memory with KV-cache tradeoffs, graph/software-SQ lifecycle, runtime 2D/batch bookkeeping, and batch correctness review |
| [ub_transfer_path_reference.md](./ub_transfer_path_reference.md) | Application content → runtime/driver API → UB interface: weights, inference inputs/results, operator launch, KV cache, and supporting driver traffic; also includes the PCIe-to-UB control/data reference with source links and unresolved mappings marked |
| [queue_subsystem_host_driver_tdt_and_ub.md](./queue_subsystem_host_driver_tdt_and_ub.md) | The `queue` subsystem: `asdrv_queue` host kernel module, the enqueue/DMA path, who calls it, TDT, and PCIe vs UnifiedBus |
| [buff_subsystem_xsmem_shared_memory.md](./buff_subsystem_xsmem_shared_memory.md) | The `buff` subsystem: XSMEM kernel module, memfd + SCM_RIGHTS shared pools, `Mbuf`, and why it never crosses PCIe |
| [dms_subsystem_urd_dispatch_and_control_plane.md](./dms_subsystem_urd_dispatch_and_control_plane.md) | The `dms` subsystem: one multiplexed ioctl, URD string-key dispatch, the per-command privilege model, device forwarding, and fault events |

**Reading order.** queue and buff are a matched pair (data plane) and are best read together. DMS is the control plane and stands alone; its §1 also collects the driver's acronym expansions (URD, DMC, DSMI, UDA, PBL, FMS, DPA, TRS) from `driver/README_en.md`.

All three carry the same provenance caveat: this tree has the **host-side** kernel driver and the userspace HAL only. The device-side driver is closed source, and inferences about it are marked as such.
