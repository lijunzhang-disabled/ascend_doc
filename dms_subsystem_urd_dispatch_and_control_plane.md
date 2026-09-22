# The Ascend DMS subsystem: URD dispatch, the control plane, and fault events

Grounded in the tree at `~/ascend_stack` (driver + runtime, 910B/950 era). Analysis date: 2026-08-27.

**Provenance caveat.** This repo contains the **host-side kernel driver** and the **userspace HAL** only. The device-side driver is closed source. DMS is unusual in that the same command format is forwarded verbatim to the device (§7), so a large part of the command space is answered by code we cannot read; those parts are marked.

Companion documents: [`queue_subsystem_host_driver_tdt_and_ub.md`](./queue_subsystem_host_driver_tdt_and_ub.md), [`buff_subsystem_xsmem_shared_memory.md`](./buff_subsystem_xsmem_shared_memory.md).

---

## 1. Names

The repo README expands the acronyms; worth recording because almost nothing else in the tree does.

| Acronym | Expansion | Source |
|---|---|---|
| **DMS** | Device Management System | `driver/README_en.md:66, 95` |
| **URD** | User Request Distribute ("user request forwarding") | `driver/README_en.md:75` |
| **DMC** | Device Maintenance Components | `driver/README_en.md:59, 94` |
| **DSMI** | Device System Management Interface | `driver/README_en.md:61` |
| **UDA** | Unified Device Access | `driver/README_en.md:74` |
| **PBL** | Public Base Lib | `driver/README_en.md:73` |
| **FMS** | Fault Management System | `driver/README_en.md:99` |
| **DPA** | Device Public Adapter | `driver/README_en.md:67, 96` |
| **TRS** | Task Resource Schedule | `driver/README_en.md:109` |

DMS is the **control and query plane** — everything you ask *about* the device, as opposed to work you send *to* it. It is the second-largest thing in `sdk_driver` after SVM: ~37 kLOC in `asdrv_dms`, plus a comparably wide HAL surface.

---

## 2. One ioctl, not twenty-four

```c
#define DMS_MAGIC 'V'
#define DMS_IOCTL_CMD _IO(DMS_MAGIC, 1)
```
`dms/command/ioctl/dms_cmd_def.h:19-20`

```c
struct dms_ioctl_arg {
    unsigned int msg_source;
    unsigned int main_cmd;
    unsigned int sub_cmd;
    const char *filter;
    unsigned int filter_len;
    void *input;
    unsigned int input_len;
    void *output;
    unsigned int output_len;
};
```
`dms_cmd_def.h:48-58`

Compare buff's 23 numbered ioctls and queue's 9. DMS has **one**, carrying a `(main_cmd, sub_cmd, filter)` triple. The command space is far too large for a fixed table — health, temperature, power, voltage, ECC statistics, PCIe VID/DID, board id, CPU utilisation, DDR/HBM usage, chip topology, flash, eMMC, sensors, BIST, fault injection, SR-IOV, vdevice info, MCU passthrough, TS patch update, URMA EIDs, hot reset — so the dispatch was made data-driven instead.

Main commands are literally ASCII letters:

```c
enum urd_main_cmd {                       /* pbl_urd_main_cmd_def.h:11-53 */
    ...
    DMS_MAIN_CMD_BASIC      = 'E',   /* 0x45 */
    DMS_MAIN_CMD_SOC        = 'F',
    DMS_MAIN_CMD_MEMORY     = 'G',
    DMS_MAIN_CMD_LPM        = 'H',
    DMS_MAIN_CMD_PRODUCT    = 'I',
    DMS_MAIN_CMD_SOFT_FAULT = 'J',
    DMS_MAIN_CMD_TRS        = 'K',
    DMS_MAIN_CMD_SENSORHUB  = 'L',
    DMS_MAIN_CMD_ISP        = 'M',
    DMS_MAIN_CMD_CAN        = 'N',
    DMS_MAIN_CMD_BBOX       = 'O',
    DMS_MAIN_CMD_EMMC       = 'P',
    DMS_MAIN_CMD_PCIE       = 'Q',
    DMS_MAIN_CMD_QOS        = 'R',
    DMS_MAIN_CMD_LOG        = 'S',
    DMS_MAIN_CMD_CAPA_GROUP = 'T',
    ASCEND_UB_CMD_BASIC     = 'U',
    DMS_MAIN_CMD_IMU        = 'V',
    DMS_MAIN_CMD_SILS       = 'w',
    DMS_MAIN_CMD_P2P_COM    = 'X',
    DMS_MAIN_CMD_FLASH      = 'Y',
    URD_TRS_MODE_CONFIG_CMD = 'Z',
    DMS_MAIN_CMD_HOTRESET   = ']',
    ...
};
```

Lower ranges are legacy device-monitor codes composed with `MAKE_UP_COMMON_COMMAND` / `MAKE_UP_DAVINCI_COMMAND`, e.g. `DMS_GET_CHIP_TEMP_CMD` (0x3), `DMS_GET_ECC_STAT_CMD` (0x1F), `DMS_GET_D_INFO_CMD` (0x60005).

---

## 3. URD: dispatch by string key

The routing lives in `sdk_driver/pbl/dev_urd/` (~1.8 kLOC), a generic dispatcher that DMS builds on and other modules reuse (TRS, DPA/UDIS, FMS all register URD handlers).

A command becomes a **string**, which is looked up in a hash table:

```c
int dms_feature_make_key(u32 main_cmd, u32 sub_cmd, const char* filter, char* key, u32 len)
{
    ...
    if (filter != NULL) {
        ret = snprintf_s(key, len, len - 1, "fun_0x%x_0x%x_%s", main_cmd, sub_cmd, filter);
    } else {
        ret = snprintf_s(key, len, len - 1, "fun_0x%x_0x%x", main_cmd, sub_cmd);
    }
```
`urd_feature.c:421-438`

The store is a djb2-style ("hash 33") bucketed table:

```c
static inline unsigned int dms_kv_hash_33(const char* tag)
{
    unsigned int hash = 0;
    while (*tag != '\0') {
        hash = (hash << 5) + hash + *tag++; /* 5 hash left offset */
    }
    return hash;
}
```
`urd_kv.c:50-61`, with `KV_KEY_MAX_LEN 256` (`urd_kv.h:25`)

The optional `filter` string is what makes the scheme extensible without burning sub-command numbers. `DMS_FILTER_TS_INFO "main_cmd=0xb"` (`dms_cmd_def.h:305`) and the macro

```c
#define DMS_MAKE_UP_FILTER_HAL_DEV_INFO_EX(f, module, info) do { \
    (f)->filter_len = (unsigned int)sprintf_s((f)->filter, sizeof((f)->filter), \
        "module=0x%x,info=0x%x", (unsigned int)(module), (unsigned int)(info)); \
} while (0)
```
`dms_cmd_def.h:12-15` (used e.g. by DSMI device-info queries)

show the pattern: a `key=value` string appended to the key, so one `(main, sub)` pair can fan out to many handlers.

---

## 4. The feature registry

```c
typedef s32 (*feature_handler)(void *feature, char *in, u32 in_len, char *out, u32 out_len);

typedef struct tag_dms_feature {
    const char *owner_name;
    u32 main_cmd;
    u32 sub_cmd;
    const char *filter;
    const char *proc_ctrl_str;   /* process whitelist, NULL = no whitelist */
    u32 privilege;
    u32 handler_type;            /* URD_HANDLER or URD_DEV_HANDLER */
    union {
        feature_handler handler;
        dev_feature_handler dev_handler;
        void *call_fun;
    };
} DMS_FEATURE_S;

int dms_feature_register(DMS_FEATURE_S *feature);
int dms_feature_unregister(DMS_FEATURE_S *feature);
```
`inc/pbl/pbl_urd.h:25-48`

Modules declare tables through macros rather than calling register directly:

```c
#define BEGIN_FEATURE_COMMAND() static DMS_FEATURE_S _feature_def[] = {
#define ADD_FEATURE_COMMAND(o, m, s, f, l, p, handler) \
    {o, m, s, f, l, p, URD_HANDLER, .call_fun = handler},
#define ADD_DEV_FEATURE_COMMAND(o, m, s, f, l, p, dev_handler) \
    {o, m, s, f, l, p, URD_DEV_HANDLER, .call_fun = dev_handler},
#define END_FEATURE_COMMAND() }; \
    static void _init_feature_table(void) { \
        _feature_table = _feature_def; \
        _feature_table_size = sizeof(_feature_def) / sizeof(_feature_def[0]); \
    }
```
`pbl_urd.h:102-110`

A concrete table:

```c
BEGIN_DMS_MODULE_DECLARATION(DMS_CHIP_DEV_CMD_NAME)
BEGIN_FEATURE_COMMAND()
ADD_FEATURE_COMMAND(DMS_CHIP_DEV_CMD_NAME, DMS_MAIN_CMD_BASIC, DMS_SUBCMD_GET_CHIP_COUNT, NULL, NULL,
                    DMS_SUPPORT_ALL_USER, dms_ioctl_get_chip_count)
ADD_FEATURE_COMMAND(DMS_CHIP_DEV_CMD_NAME, DMS_MAIN_CMD_BASIC, DMS_SUBCMD_GET_CHIP_LIST, NULL, NULL,
                    DMS_SUPPORT_ALL_USER, dms_ioctl_get_chip_list)
ADD_FEATURE_COMMAND(DMS_CHIP_DEV_CMD_NAME, DMS_MAIN_CMD_BASIC, DMS_SUBCMD_GET_DEVICE_FROM_CHIP, NULL, NULL,
                    DMS_SUPPORT_ALL_USER, dms_ioctl_get_device_from_chip)
ADD_FEATURE_COMMAND(DMS_CHIP_DEV_CMD_NAME, DMS_MAIN_CMD_BASIC, DMS_SUBCMD_GET_CHIP_FROM_DEVICE, NULL, NULL,
                    DMS_SUPPORT_ALL_USER, dms_ioctl_get_chip_from_device)
END_FEATURE_COMMAND()
END_MODULE_DECLARATION()
```
`dms/devmng/dc/chip_dev/dms_chip_dev.c:28-39`

Registration order is driven by a staged loader — `DECLAER_FEATURE_AUTO_INIT(dms_urd_forward_init, FEATURE_LOADER_STAGE_5)` and the per-device variant `DECLAER_FEATURE_AUTO_INIT_DEV` (`dc/urd_forward/dms_urd_forward.c:112, 129`). The net effect is a plugin registry: any subsystem in the driver publishes queries without touching a central switch.

### Per-feature bookkeeping

Each registered handler gets a node with a refcount, a state, a whitelist, and statistics:

```c
typedef struct tag_feature_statistic {
    u64 used;  u64 failed;
    u64 time_max;  u64 time_min;  u64 time_last;
    s32 last_ret;
} FEATURE_STATISTIC_S;                      /* urd_feature.h:30-37 */

typedef struct tag_dms_feature_node {
    struct tag_dms_feature *feature;
    char **proc_ctrl;  char *proc_buf;
    ka_atomic_t count;
    u32 proc_num;  u32 state;
    FEATURE_STATISTIC_S s;
} DMS_FEATURE_NODE_S;                       /* urd_feature.h:39-47 */
```

Every call is timed with `dms_get_cur_cpu_tick()` — `CNTVCT_EL0` on device, `ka_system_ktime_get_raw_ns()` on host (`urd_feature.h:65-76`) — and folded into the statistics. Unregistration waits for in-flight calls to drain, up to `FEATURE_WAIT_MAX_TIME (6000)` × `FEATURE_WAIT_EACH_TIME (10)` ms = 60 s (`urd_feature.h:51, 53`). `dms_feature_print_feature_list` exposes the table.

---

## 5. Authorization is part of the command definition

The `privilege` field is three orthogonal axes packed into one word.

```c
#define DMS_ACC_ROOT           0x0001U
#define DMS_ACC_ALL            (DMS_ACC_ROOT | DMS_ACC_DM_USER | DMS_ACC_OPERATE | DMS_ACC_USER | DMS_ACC_LIMIT_USER)
#define DMS_ACC_NOT_LIMIT_USER (DMS_ACC_ROOT | DMS_ACC_DM_USER | DMS_ACC_OPERATE | DMS_ACC_USER)
#define DMS_ACC_MANAGE_USER    (DMS_ACC_ROOT | DMS_ACC_DM_USER)
#define DMS_ACC_ROOT_ONLY      (DMS_ACC_ROOT)

#define DMS_ENV_PHYSICAL       0x0010U
#define DMS_ENV_ALL            (DMS_ENV_PHYSICAL | DMS_ENV_VIRTUAL | DMS_ENV_DOCKER | DMS_ENV_ADMIN_DOCKER)
#define DMS_ENV_NOT_VIRTUAL    (DMS_ENV_PHYSICAL | DMS_ENV_DOCKER | DMS_ENV_ADMIN_DOCKER)
#define DMS_PHYSICAL_ONLY      (DMS_ENV_PHYSICAL)

#define DMS_VDEV_NOTSUPPORT    (0)

#define DMS_SUPPORT_ALL_USER   (DMS_ACC_ALL | DMS_ENV_ALL | DMS_VDEV_ALL)
#define DMS_SUPPORT_ROOT_ONLY  (DMS_ACC_ROOT_ONLY | DMS_ENV_ALL | DMS_VDEV_ALL)
#define DMS_SUPPORT_ROOT_PHY   (DMS_ACC_ROOT_ONLY | DMS_PHYSICAL_ONLY)
```
`pbl/dev_urd/command/ioctl/pbl_urd_common.h:23-88`

- **Who** — root / device-manager user / operator / ordinary user / limited user
- **Where** — bare metal / VM / normal docker / admin docker
- **Whether a vdevice may issue it at all**

Checked in that order:

```c
s32 dms_feature_access_identify(u32 feature_prof, u32 msg_source)
{
    user_acc = dms_feature_access_get_acc(msg_source);
    ret = vdevice_access_check(user_acc, feature_prof);  if (ret != 0) return ret;
    ret = runtime_env_check(user_acc, feature_prof);     if (ret != 0) return ret;
    ret = user_group_identify(user_acc, feature_prof);   if (ret != 0) return ret;
    return 0;
}
```
`urd_acc_ctrl.c:211-232`

A real declaration:

```c
DMS_ACC_ROOT | DMS_ENV_PHYSICAL | DMS_VDEV_NOTSUPPORT, urd_forward_ubmem_dev_repair
```
`dc/urd_forward/dms_urd_forward.c:94`

Memory repair: root only, bare metal only, never from a vdevice.

On top of that sits an optional **process-name whitelist** (`proc_ctrl_str`, parsed by `dms_feature_parse_proc_ctrl:297`, enforced by `dms_feature_whitelist_check:353`).

This machinery exists because DMS is the one interface a container or tenant VM can reach that touches physical device state. Note the provenance tagging:

```c
enum cmd_source { FROM_DSMI = 0, FROM_HAL = 1, FROM_KERNEL = 2, INVALID_SOURCE = 0xFF };
```
`dms_cmd_def.h:100-105`

with `MSG_FROM_USER_REST_ACC` distinguished for restricted-access file descriptors (`urd_init.c:385`).

---

## 6. The request path

```c
STATIC long dms_ioctl(ka_file_t* filep, unsigned int ioctl_cmd, unsigned long arg)
{
    ...
    ret = ka_base_copy_from_user(&ctl_arg, (void*)((uintptr_t)arg), sizeof(struct urd_ioctl_arg));
    ...
    if ((ioctl_cmd == URD_IOCTL_CMD) || (check_cmd_arg_valid(ioctl_cmd, ctl_arg))) {
        msg_source = (ascend_intf_is_restrict_access(filep) ? MSG_FROM_USER_REST_ACC : MSG_FROM_USER);
        ret = dms_cmd_process(ctl_arg.devid, &ctl_arg.cmd, &ctl_arg.cmd_para, msg_source);
    } else {
        ...
        ret = -ENODEV;
    }
    return ret;
}
```
`urd_init.c:367-392`

`check_cmd_arg_valid` cross-checks that the ioctl number itself encodes the same `(main_cmd, sub_cmd)` the payload claims (`urd_init.c:346-354`) — the reason `dms_cmd_def.h` also defines named aliases like `DMS_GET_FAULT_EVENT _IO(DMS_MAIN_CMD_BASIC, DMS_SUBCMD_GET_FAULT_EVENT)`.

Then:

```c
STATIC int dms_cmd_process(u32 devid, struct urd_cmd *cmd, struct urd_cmd_para *cmd_para, u32 msg_source)
{
    ...
    ret = dms_make_up_msg_arg(cmd, cmd_para, &arg);   /* builds the key, copies input in */
    ret = dms_feature_process(&arg);
    ret = dms_proc_feature_output(cmd_para, &arg);    /* copies output back to user */
out:
    dms_free_msg(&arg);
```
`urd_init.c:310-344`

and the core:

```c
int dms_feature_process(DMS_FEATURE_ARG_S* arg)
{
    node = feature_get_key_node_ex(arg->key);
    if (node == NULL || (node->feature == NULL)) {
        dms_debug("Not support feature. (key=\"%s\")\n", arg->key);
        return -EOPNOTSUPP;
    }
    feature = node->feature;

    if (arg->msg_source == MSG_FROM_USER || arg->msg_source == MSG_FROM_USER_REST_ACC) {
        ret = dms_feature_whitelist_check((const char**)node->proc_ctrl, node->proc_num);
        if (ret != 0) { ... return DRV_ERROR_OPER_NOT_PERMITTED; }
        ret = dms_feature_access_identify(feature->privilege, arg->msg_source);
        if (ret != 0) { ... return ret; }
    }

    start = dms_get_cur_cpu_tick();
    ret = urd_feature_handle(arg, feature);
    end = dms_get_cur_cpu_tick();
    dms_update_static(&node->s, ret, start, end);
    feature_dec_work(node);
    return ret;
}
```
`urd_feature.c:375-419`

Two things to notice: an unknown key returns `-EOPNOTSUPP` rather than an error (unsupported features degrade gracefully), and **`MSG_FROM_KERNEL` skips both checks** — in-driver callers via `dms_cmd_process_from_kernel` (`urd_init.c:356-365`, `KA_EXPORT_SYMBOL`) are trusted.

---

## 7. The same triple forwards to the device

```c
#define FILTER_LEN_MAX 128
#define PAYLOAD_LEN_MAX 340
struct urd_forward_msg {
    u32 main_cmd;
    u32 sub_cmd;
    char filter[FILTER_LEN_MAX];
    unsigned int filter_len;
    u32 output_len;
    u32 payload_len;
    char payload[PAYLOAD_LEN_MAX];
};
```
`dms/command/msg/dms_msg.h:41-50`

sent by `dms_urd_forward_send_to_device(u32 phy_id, u32 vfid, struct urd_forward_msg *urd_msg, char *out, u32 out_len)` (`dc/urd_forward/dms_urd_forward.c:193`), built by `dms_set_urd_msg` (`:160`).

A query the host driver cannot answer locally is forwarded **verbatim** to the device's own URD, which resolves it against *its* registry. One command format, two hash tables, transparent across PCIe. *(Inferred: the device-side registry is closed source; the mechanism is visible only from the sending end.)*

This is why the command space can be this large without a host-side handler for every entry — and why an unknown key returning `-EOPNOTSUPP` is the normal case rather than a bug.

The generic device message envelope alongside it:

```c
struct dms_h2d_msg_head { u32 dev_id; u32 msg_id; u16 valid; /* 0x5A5A */ u16 result; };
#define DMS_INFO_PAYLOAD_LEN 512UL
struct dms_h2d_msg { struct dms_h2d_msg_head header; u8 payload[DMS_H2D_MSG_PAYLOAD_LEN - sizeof(head)]; };
```
`dms_msg.h:19-31`

---

## 8. The async half: fault events

DMS is not purely request/response. The fops include `poll`:

```c
const ka_file_operations_t g_dms_file_operations = {
    ka_fs_init_f_owner(KA_THIS_MODULE)
    ka_fs_init_f_open(dms_open)
    ka_fs_init_f_release(dms_release)
    ka_fs_init_f_poll(dms_msg_poll)
    ka_fs_init_f_unlocked_ioctl(dms_ioctl)
};
```
`urd_init.c:394-400`

Userspace blocks on device-originated fault events:

```c
struct dms_event_para {
    unsigned int event_code;
    int pid;
    unsigned int event_id;
    unsigned short deviceid;
    unsigned short node_type;   unsigned char node_id;
    unsigned short sub_node_type; unsigned char sub_node_id;
    unsigned char severity;
    unsigned char assertion;
    unsigned short sensor_num;
    int event_serial_num;
    int notify_serial_num;
    unsigned long long alarm_raised_time;
    char event_name[DMS_MAX_EVENT_NAME_LENGTH];      /* 256 */
    char additional_info[DMS_MAX_EVENT_DATA_LENGTH]; /* 32 */
    unsigned char event_info[DMS_MAX_EVENT_INFO_NUM];
};
```
`dms_cmd_def.h:65-84`, batched up to `DMS_MAX_EVENT_ARRAY_LENGTH (128)` in `struct devdrv_event_obj_para`

Readers subscribe with a filter:

```c
struct dms_event_filter {
    unsigned long long filter_flag; /* bit0: event_id; bit1: severity; bit2: node_type; bit3: current tgid */
    unsigned int event_id;
    unsigned char severity;
    unsigned short node_type;
    unsigned char resv[MAX_EVENT_RESV_LENGTH];
};

struct dms_read_event_ioctl {
    enum cmd_source cmd_src;
    int timeout;
    int dev_id;
    struct dms_event_filter filter;
};
```
`dms_cmd_def.h:92-111`

`bit3: current tgid` is the interesting one — a process can ask for only the faults attributed to itself. This is the plumbing behind health monitoring, ECC/HBM fault reporting, and the FMS soft-fault path, which registers a URD notifier so that DMS nodes are cleaned up when a process exits (`sdk_driver/fms/soft_fault/soft_fault.c:721-726`).

---

## 9. Device nodes

No `/dev` node of its own. Two davinci sub-modules (the same registration style as buff/XSMEM):

| Name | Registered at | What |
|---|---|---|
| `DAVINCI_INTF_MODULE_URD` (`"URD"`) | `pbl/dev_urd/urd_init.c:427` | the dispatcher |
| `DAVINCI_INTF_MODULE_DEVMNG` | `dms/devmng/drv_devmng/drv_devmng_host/ascend910/devdrv_manager.c:2562` | the older devdrv_manager path |

Names in `sdk_driver/inc/davinci_interface.h:28` and `ascend_hal/inc/davinci_interface.h:23`.

---

## 10. What is inside

### Kernel (`sdk_driver/dms/devmng/`, ~36.8 kLOC)

| Subdir | LOC | Contents |
|---|---|---|
| `drv_devmng/` | 18.2k | the legacy device manager (`ascend910`, `common`, `dc`, `feature`, `inc`) |
| `dc/` | 12.7k | feature modules: `bbox`, `bbox_dump`, `chip_dev`, `core`, `custom`, `hccs`, `heart_beat`, `hotreset`, `log`, `status`, `time`, `ub`, `urd_forward` |
| `status/` | 1.6k | device status queries |
| `core/` | 1.4k | |
| `include/` | 1.2k | |
| `adapter/`, `event/`, `config/`, `ts/`, `product/` | ~1.6k | |
| `command/` | 1.0k | the UAPI + H2D message headers |

Plus the dispatcher itself in `sdk_driver/pbl/dev_urd/` (`urd_init.c`, `urd_feature.c`, `urd_acc_ctrl.c`, `urd_kv.c`, `urd_container.c`, `urd_notifier.c`, ~1.8 kLOC).

### HAL (`ascend_hal/dms/`)

A wide, thin fan-out: **27 files, 79 call sites**, all funnelling through one function.

```c
int DmsIoctl(int cmd, struct dms_ioctl_arg *ioarg);      /* dms/common/dms_user_common.h:55 */
```
implemented at `dms/common/dms_user_common.c:175`, which opens the interface, translates `dms_ioctl_arg` → `urd_ioctl_arg`, and issues the ioctl.

Organised by subject rather than by mechanism:

```
bbox/  board/  chip/{can,dvpp,flash,hccs,host_aicpu,imu,isp,memory,sensorhub,sio,soc,ts}
communication/  emmc/  fault/  fault_inject/  flash/  hbm/  lpm/  p2p/  pcie/
power/  qos/  sils/  time/  ub/  udis/  vdev/  drv_devmng/
```

---

## 11. Who calls it

```
npu-smi / dcmi / container runtimes
  -> libdrvdsmi_host, dcmi            ascend_hal/dmc/dsmi/          (DMC sits ON TOP of DMS)
     -> ascend_hal/dms/**             27 files, 79 DmsIoctl call sites
        -> DmsIoctl(DMS_IOCTL_CMD, &ioarg)      dms_user_common.c:175
           -> ioctl on the URD davinci sub-module
              -> dms_ioctl -> dms_cmd_process -> dms_feature_process -> handler
                 (or urd_forward -> device-side URD)
```

Note the direction between the two management subsystems: DMC's `dsmi` is a *consumer* of DMS, which is why `ascend_hal/dms/CMakeLists.txt:17` includes `dmc/device_monitor/include`.

CANN runtime consumers, by call-site count:

| File | Sites |
|---|---|
| `msprof/collector/dvvp/driver/devmgmt/ai_drv_dev_api.cpp` | 33 |
| `aicpu_sched/aicpu_cust_schedule/core/aicpusd_drv_manager.cpp` | 21 |
| `runtime/core/src/runtime.cc` | 20 |
| `runtime/driver/npu_driver_res.cc` | 16 |
| `queue_schedule/server/bind_cpu_utils.cpp` | 14 |
| `runtime/driver/npu_driver.cc` | 12 |

Other driver subsystems reach it in-kernel through `dms_cmd_process_from_kernel`, and one crossover already appeared in the buff analysis: `buff_is_support()` calls `halGetDeviceSplitMode(0, &split_mode)` (`ascend_hal/buff/dc/share_fd_adp/grp_mng.c:785`), which is a DMS query.

---

## 12. The four subsystems compared

| | queue | buff | dmc | **dms** |
|---|---|---|---|---|
| Plane | data | data | observability | **control / query** |
| Kernel module here | `asdrv_queue` | `asdrv_buff` | none | `asdrv_dms` |
| `.c` files in `sdk_driver` | 11 | 12 | 0 | **73** |
| Node | `/dev/hi-queue-manage` | davinci sub-module | — | davinci sub-modules (URD + DEVMNG) |
| ioctl style | 9 numbered, magic `'Q'` | 23 numbered, magic `'X'` | (device-side only) | **1, magic `'V'`, multiplexed** |
| Dispatch | static handler table | static handler table | — | **runtime string-key hash registry** |
| Authorization | queue share attrs | `GroupShareAttr` via mmap `prot` | — | **per-command 3-axis privilege + process whitelist** |
| Crosses to device | HDC + DMA descriptors | never | HDC / URMA from the HAL | **URD forward, same message shape** |
| Async path | event multicast | buf event subscribe | channel poll | **`poll()` on fault events** |

The design differences track the workload. The data-plane modules have few operations on huge payloads, so their ioctl tables are small and their machinery is about pinning and mapping. DMS has hundreds of operations on tiny payloads, so its machinery is about routing and authorization instead.

---

## 13. Quick reference

| Question | Answer |
|---|---|
| Kernel module | `asdrv_dms` |
| Char device | none of its own — davinci sub-modules `"URD"` and DEVMNG |
| ioctl | one, `DMS_IOCTL_CMD _IO('V', 1)`, multiplexed on `(main_cmd, sub_cmd, filter)` |
| Dispatch key | string `"fun_0x<main>_0x<sub>[_<filter>]"`, max 256 chars |
| Lookup | hash-33 bucketed table (`urd_kv.c`) |
| Handler registration | `dms_feature_register` via `ADD_FEATURE_COMMAND` tables + staged auto-init |
| Unknown command | `-EOPNOTSUPP`, logged at debug level — a normal outcome |
| Authorization | user class × runtime environment × vdevice support, plus process whitelist |
| Kernel-internal callers | `dms_cmd_process_from_kernel` — skips whitelist and privilege checks |
| Device forwarding | `struct urd_forward_msg`, same triple, resolved by the device's own registry |
| Async events | `poll()` + `dms_event_para` / `dms_event_filter`, filterable by id / severity / node type / own tgid |
| HAL funnel | `DmsIoctl` — 79 call sites across 27 files |
| Sits below | DSMI / DCMI / `npu-smi`, and the CANN runtime's device queries |
