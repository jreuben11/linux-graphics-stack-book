# Chapter 179: The Linux `accel` Subsystem: NPU and AI Accelerator Drivers

**Target audiences:** Systems and driver developers porting or maintaining NPU or dedicated AI accelerator kernel drivers; application developers integrating AI inference runtimes with Linux hardware accelerators; engineers evaluating the `/dev/accel` device model relative to GPU compute paths (`/dev/dri/renderD*`, `/dev/kfd`).

---

## Table of Contents

1. [Motivation: Why a Separate Subsystem?](#motivation)
2. [The DRM Foundation: Built On, Not Separated From](#drm-foundation)
3. [Core Machinery: `ACCEL_MAJOR`, Device Nodes, and `accel_open`](#core-machinery)
4. [The `DRIVER_COMPUTE_ACCEL` Flag and `drm_driver` Integration](#driver-flag)
5. [Intel NPU Driver: `ivpu`](#ivpu)
6. [HabanaLabs Gaudi Driver](#habanalabs)
7. [Qualcomm Cloud AI 100: `qaic`](#qaic)
8. [AMD XDNA: Phoenix and Hawk Point NPUs](#amdxdna)
9. [ARM Ethos-U: Embedded Microcontroller NPU](#ethosu)
10. [Rockchip RKNPU: The `rocket` Driver](#rocket)
11. [Writing an Accel Driver: Key Steps](#writing-driver)
12. [Memory Management: GEM Without Display](#gem-memory)
13. [DRM GPU Scheduler Integration](#drm-sched)
14. [Security Model and Access Control](#security)
15. [Userspace Integration: Runtimes and Inference Engines](#userspace)
16. [When to Use `/dev/accel` vs `/dev/dri/renderD*` vs `/dev/kfd`](#device-choice)
17. [Integrations](#integrations)

---

## 1. Motivation: Why a Separate Subsystem? {#motivation}

The Linux kernel has long had two primary tracks for hardware accelerators: the DRM/KMS subsystem for GPU display and rendering, and the `misc` character device framework for everything else. By 2022 this division had become untenable. Multiple AI accelerator vendors — Intel (Habana Labs), Qualcomm, and ARM — had production hardware deploying at cloud scale, and each was writing a different flavour of `misc` device driver with incompatible IOCTL conventions, inconsistent sysfs layouts, and no shared infrastructure for memory management.

Oded Gabbay (Intel, Habana Labs), who had written the production `habanalabs` driver deployed at AWS, proposed a dedicated subsystem. His rationale, laid out in the [public LKML thread](https://lkml.kernel.org/lkml/CAFCwf13WU3ZEjurEaEnVC56zorwKr-uuQn-ec10r301Fh+XEtA@mail.gmail.com/T/) in August 2022, was threefold:

1. **DRM's common IOCTLs are irrelevant for dedicated accelerators.** DRM's shared interface set is designed around GPU rendering: KMS modesetting, GEM buffer objects exposed to display, and render-node access for shader compilation. Dedicated inference NPUs expose none of these. Reusing `struct drm_device` while implementing only a thin DRM shim would mislead both drivers and tooling.

2. **Avoiding forced upstreaming of compiler toolchains.** The DRM subsystem requires that compiler stacks for programmable shaders be upstreamed to mainline LLVM or Mesa before a driver can be accepted. Dedicated inference accelerators commonly run closed firmware with a narrow ABI; requiring full compiler upstreaming would block drivers for shipping hardware indefinitely.

3. **Establishing community standards.** A new subsystem under `drivers/accel/` gives maintainers a place to build shared conventions — consistent device naming, debugfs layout, DMA-BUF integration, sysfs attributes — that would otherwise accumulate as copy-paste between `misc` drivers.

The subsystem was merged into the DRM tree for **Linux 6.2** (released February 2023), via Linus Torvalds' acceptance of Dave Airlie's `drm-next` pull. The merge included the core `drivers/accel/drm_accel.c`, documentation under `Documentation/accel/`, and the initial migration of `habanalabs` from `drivers/misc/` to `drivers/accel/`. [Source: Phoronix coverage](https://www.phoronix.com/news/Linux-6.2-Compute-Next)

---

## 2. The DRM Foundation: Built On, Not Separated From {#drm-foundation}

A critical design decision distinguishes the `accel` subsystem from a truly independent framework: **it is built directly on the DRM core**, not parallel to it. Accelerator drivers use `struct drm_device`, `struct drm_driver`, `struct drm_file`, GEM object infrastructure, DRM scheduler, DMA-BUF helpers, debugfs integration, and the same device lifetime management (`devm_drm_dev_alloc`, `drm_dev_register`) as GPU drivers. What the subsystem adds is a separate character device major number, separate sysfs class, and a separate `xarray` of registered devices — sufficient to keep accelerator nodes entirely distinct from `/dev/dri/` entries in udev rules, device enumeration, and access policy.

The kernel documentation states this explicitly: "The accel core code integrates with the DRM subsystem, allowing accelerator devices to function as a new DRM device type." [Source: kernel.org accel docs](https://www.kernel.org/doc/html/latest/accel/introduction.html)

The practical implication for driver authors is that existing knowledge of DRM driver structure transfers almost entirely to `accel` drivers. The differences are:

| Aspect | DRM GPU Driver | Accel Driver |
|--------|---------------|-------------|
| `driver_features` flag | `DRIVER_RENDER` (or `DRIVER_MODESET`) | `DRIVER_COMPUTE_ACCEL` |
| File open entry point | `drm_open` | `accel_open` |
| Device node path | `/dev/dri/renderD128` | `/dev/accel/accel0` |
| Character device major | 226 (DRM) | 261 (ACCEL) |
| Minor xarray | `drm_minors_xa` | `accel_minors_xa` |
| `drm_minor` type | `DRM_MINOR_RENDER` | used via `dev->accel` |
| Modeset capability | Possible | Forbidden |
| Display pipeline | Possible | Forbidden |

The exclusive-flag relationship is enforced at registration time: `DRIVER_COMPUTE_ACCEL` is mutually exclusive with `DRIVER_RENDER` and `DRIVER_MODESET`. A device that exposes both graphics and compute (such as a GPU with an embedded neural engine) must use two separate kernel drivers connected via the auxiliary bus, each presenting its own device node to userspace. This mirrors the Intel NPU situation on Meteor Lake: the `i915`/`Xe` driver handles the GPU, while `ivpu` handles the NPU block via a completely independent PCI function.

---

## 3. Core Machinery: `ACCEL_MAJOR`, Device Nodes, and `accel_open` {#core-machinery}

The accel core is implemented in `drivers/accel/drm_accel.c` with the header at `include/drm/drm_accel.h`. [Source: github.com/torvalds/linux](https://github.com/torvalds/linux/blob/master/include/drm/drm_accel.h)

The major device number is a fixed constant:

```c
/* include/drm/drm_accel.h */
#define ACCEL_MAJOR         261
#define ACCEL_MAX_MINORS    256
```

The kernel registers this major at boot through `accel_core_init()`, called from `drm_core_init()`:

```c
/* drivers/accel/drm_accel.c */
int __init accel_core_init(void)
{
    int ret;

    ret = accel_sysfs_init();
    if (ret < 0) {
        DRM_ERROR("Cannot create ACCEL class: %d\n", ret);
        goto error;
    }

    ret = register_chrdev(ACCEL_MAJOR, "accel", &accel_stub_fops);
    if (ret < 0)
        DRM_ERROR("Cannot register ACCEL major: %d\n", ret);
    /* ... */
}
```

The `accel_class` structure controls device node naming:

```c
/* drivers/accel/drm_accel.c */
static char *accel_devnode(const struct device *dev, umode_t *mode)
{
    return kasprintf(GFP_KERNEL, "accel/%s", dev_name(dev));
}

static const struct class accel_class = {
    .name = "accel",
    .devnode = accel_devnode,
};
```

This causes udev to create `/dev/accel/accel0`, `/dev/accel/accel1`, etc. The sysfs class appears under `/sys/class/accel/`. Debugfs lands under `/sys/kernel/debug/accel/`.

The `accel_set_device_instance_params()` function assigns `ACCEL_MAJOR` and the `accel_class` to a device instance during driver registration:

```c
void accel_set_device_instance_params(struct device *kdev, int index)
{
    kdev->devt = MKDEV(ACCEL_MAJOR, index);
    kdev->class = &accel_class;
    kdev->type = &accel_sysfs_device_minor;
}
```

The `accel_open()` function is the entry point when a process opens `/dev/accel/accel0`. It mirrors `drm_open()` but uses the separate `accel_minors_xa` xarray to look up the correct device:

```c
/* drivers/accel/drm_accel.c */
int accel_open(struct inode *inode, struct file *filp)
{
    struct drm_device *dev;
    struct drm_minor *minor;
    int retcode;

    minor = drm_minor_acquire(&accel_minors_xa, iminor(inode));
    if (IS_ERR(minor))
        return PTR_ERR(minor);

    dev = minor->dev;
    atomic_fetch_inc(&dev->open_count);

    /* share address_space across all char-devs of a single device */
    filp->f_mapping = dev->anon_inode->i_mapping;

    retcode = drm_open_helper(filp, minor);
    /* ... */
}
EXPORT_SYMBOL_GPL(accel_open);
```

Drivers declare their file operations using the `DEFINE_DRM_ACCEL_FOPS` macro, which wires `accel_open` as the open handler while reusing the standard DRM IOCTL dispatch, poll, read, and mmap implementations:

```c
/* include/drm/drm_accel.h — simplified */
#define DRM_ACCEL_FOPS                          \
    .open           = accel_open,               \
    .release        = drm_release,              \
    .unlocked_ioctl = drm_ioctl,                \
    .compat_ioctl   = drm_compat_ioctl,         \
    .poll           = drm_poll,                 \
    .read           = drm_read,                 \
    .llseek         = noop_llseek,              \
    .mmap           = drm_gem_mmap,             \
    .fop_flags      = FOP_UNSIGNED_OFFSET

#define DEFINE_DRM_ACCEL_FOPS(name)             \
    static const struct file_operations name = {\
        .owner = THIS_MODULE,                   \
        DRM_ACCEL_FOPS,                         \
    }
```

---

## 4. The `DRIVER_COMPUTE_ACCEL` Flag and `drm_driver` Integration {#driver-flag}

There is no separate `struct accel_driver` or `struct accel_device`. Accelerator drivers use `struct drm_driver` and `struct drm_device` directly, embedding the latter in their own device structure. The sole registration distinction is the `DRIVER_COMPUTE_ACCEL` feature flag, defined in `include/drm/drm_drv.h`:

```c
/* include/drm/drm_drv.h */
enum drm_driver_feature {
    DRIVER_GEM              = BIT(0),
    DRIVER_MODESET          = BIT(1),
    DRIVER_RENDER           = BIT(3),
    /* ... */

    /**
     * @DRIVER_COMPUTE_ACCEL:
     *
     * Driver supports compute acceleration devices. This flag is mutually
     * exclusive with @DRIVER_RENDER and @DRIVER_MODESET. Devices that
     * support both graphics and compute acceleration should be handled by
     * two drivers connected using auxiliary bus.
     */
    DRIVER_COMPUTE_ACCEL    = BIT(7),
    /* ... */
};
```

When `drm_dev_register()` sees `DRIVER_COMPUTE_ACCEL` set, the DRM core allocates an `accel` minor (using `accel_minors_xa`) rather than a `render` or `primary` minor, and calls `accel_set_device_instance_params()` to configure the correct major and sysfs class.

A minimal accelerator driver skeleton looks like this:

```c
/* Skeleton accel driver — illustrative, not real upstream code */
#include <drm/drm_accel.h>
#include <drm/drm_drv.h>
#include <drm/drm_gem.h>

struct my_accel_device {
    struct drm_device drm;   /* must be first or use container_of */
    void __iomem *bar;
    /* ... hardware-specific fields ... */
};

DEFINE_DRM_ACCEL_FOPS(my_accel_fops);

static const struct drm_ioctl_desc my_accel_ioctls[] = {
    DRM_IOCTL_DEF_DRV(MY_SUBMIT, my_submit_ioctl, DRM_RENDER_ALLOW),
};

static const struct drm_driver my_accel_driver = {
    .driver_features = DRIVER_GEM | DRIVER_COMPUTE_ACCEL,
    .name            = "my_accel",
    .desc            = "Example accelerator driver",
    .fops            = &my_accel_fops,
    .ioctls          = my_accel_ioctls,
    .num_ioctls      = ARRAY_SIZE(my_accel_ioctls),
    .open            = my_accel_open_cb,
    .postclose       = my_accel_postclose_cb,
};
```

Note the use of `DRM_RENDER_ALLOW` in the IOCTL descriptor: accel drivers reuse DRM's render-node permission model for IOCTL access control, even though there is no `DRIVER_RENDER` flag. The flag name is slightly misleading for accel drivers, but its semantic — allow access without DRM master status — is exactly what inference accelerators need.

---

## 5. Intel NPU Driver: `ivpu` {#ivpu}

The `ivpu` driver (`drivers/accel/ivpu/`) supports Intel's Vision Processing Unit (VPU) integrated into Meteor Lake (MTL, IP generation 37xx), Arrow Lake (ARL, also 37xx), Lunar Lake (LNL, IP 40xx), Panther Lake (PTL, IP 50xx), Wolf Canyon Lake (WCL, IP 50xx), and Nova Lake (NVL, IP 60xx). [Source: github.com/torvalds/linux drivers/accel/ivpu/ivpu_drv.h](https://github.com/torvalds/linux/blob/master/drivers/accel/ivpu/ivpu_drv.h)

### Device Structure

The driver defines `struct ivpu_device`, which embeds `struct drm_device` as its first member:

```c
/* drivers/accel/ivpu/ivpu_drv.h */
struct ivpu_device {
    struct drm_device   drm;        /* must be first — used by container_of */
    void __iomem       *regb;       /* buttress register BAR */
    void __iomem       *regv;       /* VPU register BAR */
    u32                 platform;   /* SILICON / SIMICS / FPGA */
    u32                 irq;

    struct ivpu_wa_table    wa;     /* workaround flags per silicon revision */
    struct ivpu_hw_info    *hw;
    struct ivpu_mmu_info   *mmu;    /* IOMMU/MMU context */
    struct ivpu_fw_info    *fw;     /* loaded firmware image */
    struct ivpu_ipc_info   *ipc;    /* IPC channel to firmware */
    struct ivpu_pm_info    *pm;

    struct ivpu_mmu_context gctx;   /* global (driver) MMU context */
    struct ivpu_mmu_context rctx;   /* reserved context */
    struct mutex            context_list_lock;
    struct xarray           context_xa;  /* per-fd user contexts */
    struct xa_limit         context_xa_limit;

    struct xarray           submitted_jobs_xa;
    struct ivpu_ipc_consumer job_done_consumer;
    /* ... timeout config, busy time accounting, etc. ... */
};
```

The `to_ivpu_device()` helper uses `container_of` to recover the driver struct from a `struct drm_device *`:

```c
static inline struct ivpu_device *to_ivpu_device(struct drm_device *dev)
{
    return container_of(dev, struct ivpu_device, drm);
}
```

### PCI Enumeration and IP Generation

The NPU is a separate PCI function on Meteor Lake and later platforms (distinct from the GPU function handled by `i915`/`Xe`). The driver enumerates PCI device IDs and maps them to hardware IP generations:

```c
/* drivers/accel/ivpu/ivpu_drv.h */
#define PCI_DEVICE_ID_MTL    0x7d1d   /* Meteor Lake — NPU 3720, HW_IP_37XX */
#define PCI_DEVICE_ID_ARL    0xad1d   /* Arrow Lake */
#define PCI_DEVICE_ID_LNL    0x643e   /* Lunar Lake — HW_IP_40XX */
#define PCI_DEVICE_ID_PTL_P  0xb03e   /* Panther Lake — HW_IP_50XX */
#define PCI_DEVICE_ID_NVL    0xd71d   /* Nova Lake — HW_IP_60XX */

static inline int ivpu_hw_ip_gen(struct ivpu_device *vdev)
{
    switch (ivpu_device_id(vdev)) {
    case PCI_DEVICE_ID_MTL:
    case PCI_DEVICE_ID_ARL:
        return IVPU_HW_IP_37XX;
    case PCI_DEVICE_ID_LNL:
        return IVPU_HW_IP_40XX;
    case PCI_DEVICE_ID_PTL_P:
    case PCI_DEVICE_ID_WCL:
        return IVPU_HW_IP_50XX;
    case PCI_DEVICE_ID_NVL:
        return IVPU_HW_IP_60XX;
    /* ... */
    }
}
```

### Firmware Loading

`ivpu_fw_load()` (declared in `drivers/accel/ivpu/ivpu_fw.h`) loads the NPU firmware image using the kernel's `request_firmware()` mechanism and maps it into VPU memory. The `struct ivpu_fw_info` tracks the firmware image, boot parameters, cold/warm boot entry points, preemption buffer sizes, and runtime log buffers:

```c
/* drivers/accel/ivpu/ivpu_fw.h */
struct ivpu_fw_info {
    const struct firmware  *file;
    const char             *name;
    char                    version[FW_VERSION_STR_SIZE];
    struct ivpu_bo         *mem;          /* BO holding the firmware */
    struct ivpu_bo         *mem_log_crit; /* critical log buffer */
    u64                     cold_boot_entry_point;
    u64                     warm_boot_entry_point;
    u32                     primary_preempt_buf_size;
    u32                     secondary_preempt_buf_size;
    u32                     sched_mode;
    /* ... */
};

int  ivpu_fw_init(struct ivpu_device *vdev);
void ivpu_fw_load(struct ivpu_device *vdev);
void ivpu_fw_boot_params_setup(struct ivpu_device *vdev,
                               struct vpu_boot_params *boot_params);
```

### Job and Command Queue Model

The submission model uses `struct ivpu_job` and `struct ivpu_cmdq`. Each `drm_file` (user context) can have multiple command queues identified by `cmdq_id`, each with a dedicated doorbell register. A job carries a VPU-side command buffer address and a count of associated buffer objects:

```c
/* drivers/accel/ivpu/ivpu_job.h */
struct ivpu_cmdq {
    struct vpu_job_queue   *jobq;          /* shared memory ring with device */
    struct ivpu_bo         *mem;           /* BO backing the queue */
    u32                     entry_count;
    u32                     id;            /* command queue ID */
    u32                     db_id;         /* doorbell register index */
    u8                      priority;
};

struct ivpu_job {
    struct ivpu_device     *vdev;
    struct ivpu_file_priv  *file_priv;
    struct dma_fence       *done_fence;    /* signalled when firmware completes */
    u64                     cmd_buf_vpu_addr;
    u32                     cmdq_id;
    u32                     job_id;
    u32                     engine_idx;
    u32                     job_status;
    size_t                  bo_count;
    struct ivpu_bo         *bos[] __counted_by(bo_count);
};

int ivpu_submit_ioctl(struct drm_device *dev, void *data, struct drm_file *file);
int ivpu_cmdq_submit_ioctl(struct drm_device *dev, void *data, struct drm_file *file);
```

### Userspace Stack

Userspace communicates with `ivpu` through IOCTLs defined in `include/uapi/drm/ivpu_accel.h`. The primary consumers are:

- **OpenVINO** via the `intel_npu_driver` execution provider plugin, which opens `/dev/accel/accel0`, allocates GEM BOs for model weights and activations, and submits inference jobs via `DRM_IOCTL_IVPU_SUBMIT`.
- **ONNX Runtime** via the `OpenVINO Execution Provider`, which delegates to OpenVINO's NPU backend.
- The `intel-npu-driver` userspace library (open source at [github.com/intel/linux-npu-driver](https://github.com/intel/linux-npu-driver)) provides the `ze_intel_npu` Level Zero driver and a lower-level C API for direct accel node access.

The NPU's IOMMU context is managed through a per-process MMU SSID (Stream Set ID). Each open file descriptor gets its own SSID in the range `IVPU_USER_CONTEXT_MIN_SSID` to `IVPU_USER_CONTEXT_MAX_SSID` (2–130), providing hardware-enforced address space isolation between concurrent users.

---

## 6. HabanaLabs Gaudi Driver {#habanalabs}

The `habanalabs` driver (`drivers/accel/habanalabs/`) supports Intel Gaudi (first generation, datacenter AI training/inference cards), Gaudi 2, and Gaudi 3 accelerators. It was the first driver migrated to the `accel` subsystem and was Oded Gabbay's primary motivation for creating the framework. [Source: github.com/torvalds/linux drivers/accel/habanalabs](https://github.com/torvalds/linux/tree/master/drivers/accel/habanalabs)

The driver is substantially larger than `ivpu` — the `common/` subdirectory alone covers memory manager, command submission, debugfs, interrupt handling, heartbeat, and device reset. The main device struct `hl_device` (defined in `drivers/accel/habanalabs/common/habanalabs.h`) tracks hundreds of fields including ASIC-specific operations vtable, command queues array, collective operations state, MMU context, power management, and health monitoring.

### Command Submission Interface

The userspace ABI is defined in `include/uapi/drm/habanalabs_accel.h`. Command submission uses `DRM_IOCTL_HL_CS` with the `union hl_cs_args` structure. The `hl_cs_in` struct controls the submission:

```c
/* include/uapi/drm/habanalabs_accel.h — simplified */
struct hl_cs_chunk {
    __u64 cb_handle;       /* handle of the command buffer GEM object */
    __u32 queue_index;     /* which internal queue to route to */
    __u32 cb_size;         /* size of commands in bytes */
    /* ... */
};

struct hl_cs_in {
    __u64 chunks_restore;  /* pointer to restore-phase chunk array */
    __u64 chunks_execute;  /* pointer to execute-phase chunk array */
    __u64 seq;             /* returned: submission sequence number */
    __u32 cs_flags;        /* HL_CS_FLAGS_* */
    __u32 ctx_id;          /* per-context identifier */
    __u32 num_chunks_restore;
    __u32 num_chunks_execute;
    /* ... */
};
```

The `cs_flags` field accepts a rich set of operational flags:

- `HL_CS_FLAGS_TIMESTAMP` (0x20): request a hardware timestamp on the command stream
- `HL_CS_FLAGS_STAGED_SUBMISSION` (0x40): multi-phase submission for large graphs
- `HL_CS_FLAGS_STAGED_SUBMISSION_FIRST` (0x80) / `HL_CS_FLAGS_STAGED_SUBMISSION_LAST` (0x100): bookend a multi-phase submission
- `HL_CS_FLAGS_ENCAP_SIGNALS` (0x800): encapsulated signal handling for collective operations
- `HL_CS_FLAGS_ENGINE_CORE_COMMAND` (0x4000): control command routed to the engine core

### Gaudi Architecture Highlights

Gaudi 2 and Gaudi 3 are PCIe-attached training accelerators. Each device has:

- Multiple TPC (Tensor Processor Core) compute engines
- An MME (Matrix Multiplication Engine) for GEMM operations
- NIC (Network Interface Card) engines for RDMA-based collective operations (all-reduce, all-gather) across training clusters — unique among `accel` drivers in that the NIC is managed by the same driver
- HBM (High Bandwidth Memory) attached on-die
- A dedicated DMA engine cluster for host-to-device and device-to-device transfers

The `habanalabs` driver manages all of these via ASIC-specific operation tables (`struct hl_asic_funcs`) registered per hardware generation (`gaudi2_funcs`, etc.).

### Userspace Runtime

The `hlthunk` userspace library ([github.com/HabanaAI/HabanaAI-Intel-Gaudi-Base-Docker](https://github.com/HabanaAI)) provides the C API consumed by:

- PyTorch Habana plugin (`habana_frameworks.torch`) for training workloads
- DeepSpeed with Habana integration for distributed training
- TensorFlow Intel Habana plugin

The library opens `/dev/accel/accelN`, allocates command buffers as GEM objects, maps them into userspace with `mmap`, writes Gaudi-specific commands, and submits via `DRM_IOCTL_HL_CS`. Completion is polled with `DRM_IOCTL_HL_WAIT_CS`, which blocks until the sequence number returned by `HL_CS` has been processed.

---

## 7. Qualcomm Cloud AI 100: `qaic` {#qaic}

The `qaic` driver (`drivers/accel/qaic/`) supports the Qualcomm Cloud AI 100 series of PCIe inference cards (AIC100, AIC200), designed for datacenter inference workloads. [Source: github.com/torvalds/linux drivers/accel/qaic/qaic.h](https://github.com/torvalds/linux/blob/master/drivers/accel/qaic/qaic.h)

### Architecture: MHI Control + DMA Bridge Channels

Unlike `ivpu` and `habanalabs`, which use traditional PCI BAR-mapped registers for command submission, `qaic` uses the **MHI (Modem Host Interface)** protocol over PCIe for control-plane communication and **DMA Bridge Channels (DBC)** for data-plane inference execution.

The device model is split between two C structs:

```c
/* drivers/accel/qaic/qaic.h */
struct qaic_device {
    struct pci_dev         *pdev;
    void __iomem           *bar_mhi;   /* MHI BAR for control */
    void __iomem           *bar_dbc;   /* DBC BAR for data */
    struct mhi_controller  *mhi_cntrl;
    struct mhi_device      *cntl_ch;   /* MHI control channel */
    u32                     num_dbc;
    struct dma_bridge_chan  dbc[] __counted_by(num_dbc);
};

struct qaic_drm_device {
    struct drm_device       drm;        /* embedded drm_device */
    struct qaic_device     *qdev;
    s32                     partition_id; /* QAIC_NO_PARTITION or logical partition */
    struct list_head        users;
    struct mutex            users_mutex;
};
```

The `to_qaic_drm_device` and `to_qaic_device` macros provide the standard `container_of` traversal between these layers. Note that `qaic_drm_device` embeds `drm_device` directly and the macro `to_accel_kdev(qddev)` accesses `(to_drm(qddev))->accel->kdev` to retrieve the kernel device of the `/dev/accel/accelN` node.

### DMA Bridge Channels

Each DBC is a bidirectional ring-buffer pair (request queue and response queue) mapped via DMA coherent memory. The request queue accepts `dbc_req` descriptors that point to pre-allocated scatter/gather buffers; the response queue carries completion status. The model is:

1. User allocates a `qaic_bo` (GEM object) via `DRM_IOCTL_QAIC_CREATE_BO`
2. User attaches slicing information (`DRM_IOCTL_QAIC_ATTACH_SLICE_BO`) describing how the buffer is segmented across DBC elements
3. User executes the BO (`DRM_IOCTL_QAIC_EXECUTE_BO`), which enqueues `dbc_req` elements into the DBC ring
4. User waits for completion (`DRM_IOCTL_QAIC_WAIT_BO`) and collects performance statistics

### Control Plane: MHI and Network Model

The control plane uses the `QAIC_MANAGE` MHI channel to send commands for model programming — uploading compiled model artifacts to device DRAM, configuring DBC parameters, and querying device properties. The `qaic_manage_ioctl` function (`DRM_IOCTL_QAIC_MANAGE`) serialises these control messages through the `struct mhi_device *cntl_ch`.

### Distinction from Qualcomm's Mobile NPU

The Cloud AI 100 (`qaic`) is a discrete PCIe inference card aimed at datacenter deployments, not the Hexagon DSP/HTP NPU integrated into Snapdragon SoCs. The Snapdragon NPU is targeted by the ONNX Runtime QNN (Qualcomm Neural Networks) Execution Provider, which uses Qualcomm's proprietary `libQnnHtp` library on Android/Windows; its Linux driver path is separate from `qaic` and is not yet in mainline as of Linux 6.x.

---

## 8. AMD XDNA: Phoenix and Hawk Point NPUs {#amdxdna}

The `amdxdna` driver (`drivers/accel/amdxdna/`) supports AMD's XDNA NPU integrated into Ryzen AI (Phoenix, Hawk Point, Strix Point, and subsequent) APUs. [Source: github.com/torvalds/linux drivers/accel/amdxdna](https://github.com/torvalds/linux/tree/master/drivers/accel/amdxdna)

The driver appeared in mainline with Linux 6.10. Its hardware is the AMD AI Engine 2 (AIE2) array — a spatially programmable SIMD architecture with a mesh of "tiles" each containing a programmable core and local memory, connected by a network-on-chip. This is distinct from AMD's GPU compute (RDNA/CDNA covered by `amdgpu`).

### Device Structure

```c
/* drivers/accel/amdxdna/amdxdna_pci_drv.h */
struct amdxdna_dev {
    struct drm_device            ddev;      /* embedded drm_device */
    struct amdxdna_dev_hdl      *dev_handle;
    const struct amdxdna_dev_info *dev_info;
    void                        *xrs_hdl;  /* XRT runtime scheduler handle */

    struct mutex                 dev_lock;
    struct list_head             client_list;
    struct amdxdna_fw_ver        fw_ver;

    struct iommu_group          *group;
    struct iommu_domain         *domain;
    struct iova_domain           iovad;     /* per-device IOVA allocator */
    const char                  *vbnv;      /* board name from firmware */
};
```

The device operations table `struct amdxdna_dev_ops` abstracts hardware generation differences (AIE2 vs. AIE4) and provides callbacks for hardware context initialisation (`hwctx_init`), command submission (`cmd_submit`), and AIE array query (`get_aie_info`).

The supported NPU generations are registered as `dev_npu1_info` (Phoenix), `dev_npu3_*_info` (Hawk Point with SR-IOV), `dev_npu4_info`, `dev_npu5_info`, and `dev_npu6_info` — each paired with hardware-specific register tables (`npu1_regs.c`, `npu4_regs.c`, etc.).

### Userspace Runtime

AMD exposes the XDNA NPU through the **XRT (Xilinx Runtime)** library, which on Linux opens `/dev/accel/accelN` for the XDNA device. Inference workloads are dispatched through ONNX Runtime's QNN EP (when targeting AMD's Ryzen AI implementation) or through AMD's own `xrt-smi` and `xrtkernels` APIs.

The driver uses IOMMU SVA (Shared Virtual Addressing) via `iommu_sva_bind_device()` to give the NPU access to process virtual addresses directly, eliminating explicit buffer pinning for inference inputs and outputs. SR-IOV support (present in `aie4_sriov.c`) enables multi-tenant cloud deployments where virtual functions each see a partition of the NPU array.

---

## 9. ARM Ethos-U: Embedded Microcontroller NPU {#ethosu}

The `ethosu` driver (`drivers/accel/ethosu/`) represents the embedded end of the accel spectrum: ARM's Ethos-U microcontroller NPU series (U55, U65, U85), integrated into SoCs for edge inference on microcontrollers. Unlike the datacenter and laptop NPU drivers, Ethos-U targets deeply embedded systems running on Cortex-M processor cores.

The driver is considerably simpler — no GEM, no DMA-BUF, no IOMMU, no PCI. It communicates with the NPU through a shared memory message queue and relies on the SoC's existing MPU (Memory Protection Unit) for isolation. Job submission is modelled on inference request descriptors carried over the shared memory interface.

This driver demonstrates that the `accel` subsystem is designed to accommodate a wide range of deployment contexts, from microcontrollers running CMSIS-NN inference to 1000-chip Gaudi training clusters.

---

## 10. Rockchip RKNPU: The `rocket` Driver {#rocket}

The `rocket` driver (`drivers/accel/rocket/`) supports the Rockchip RKNN (also called RKNPU) neural processing unit found in Rockchip SoCs including the RK3588. [Source: github.com/torvalds/linux drivers/accel/rocket/Kconfig](https://github.com/torvalds/linux/blob/master/drivers/accel/rocket/Kconfig)

The Kconfig entry describes the target hardware:

```kconfig
config DRM_ACCEL_ROCKET  # drivers/accel/rocket/Kconfig
    tristate "Rocket (support for Rockchip NPUs)"
    depends on DRM_ACCEL
    depends on (ARCH_ROCKCHIP && ARM64) || COMPILE_TEST
    depends on ROCKCHIP_IOMMU || COMPILE_TEST
    depends on MMU
    select DRM_SCHED
    select DRM_GEM_SHMEM_HELPER
    help
      Choose this option if you have a Rockchip SoC that contains a
      compatible Neural Processing Unit (NPU), such as the RK3588.
      The interface exposed to userspace is described in
      include/uapi/drm/rocket_accel.h and is used by the Rocket
      userspace driver in Mesa3D.
```

Unlike most other `accel` drivers, `rocket` targets the **Mesa3D** project for its userspace implementation — specifically the Mesa `rocket` Gallium driver. This is significant: the accel subsystem's KConfig explicitly does not require userspace to ship with mainline Mesa for all drivers, but Rockchip has chosen the fully open-source Mesa route.

The `rocket` driver architecture follows the standard accel pattern closely:

```c
/* drivers/accel/rocket/rocket_drv.h */
struct rocket_file_priv {
    struct rocket_device       *rdev;
    struct rocket_iommu_domain *domain;  /* per-fd IOMMU domain */
    struct drm_mm               mm;      /* VA space allocator */
    struct mutex                mm_lock;
    struct drm_sched_entity     sched_entity; /* DRM GPU scheduler entity */
};
```

Each open file descriptor gets its own IOMMU domain (via `rocket_iommu_domain`), with a separate DRM memory manager (`drm_mm`) for GPU VA allocation within that domain, and a `drm_sched_entity` for DRM GPU scheduler integration. This is textbook accel driver design: per-fd isolation via IOMMU, GEM shmem backing via `DRM_GEM_SHMEM_HELPER`, and job ordering via `DRM_SCHED`.

The RK3588's NPU is a dual-core design (primary and secondary cores) capable of up to 6 TOPS (INT8) inference performance. It is widely used in embedded AI products, SBCs (Single Board Computers) such as the Orange Pi 5 and Rock 5B, and is the NPU for which the most community-maintained Linux accel driver development outside of Intel/AMD/Qualcomm occurs.

---

## 11. Writing an Accel Driver: Key Steps {#writing-driver}

For a driver developer porting new NPU or inference accelerator hardware to Linux, the `accel` subsystem provides a clear path. This section outlines the key implementation steps, using verified patterns from existing drivers.

### Step 1: Choose Your Memory Management Approach

The three common approaches in current `accel` drivers:

- **GEM shmem** (`DRM_GEM_SHMEM_HELPER`, used by `rocket`, `ivpu`): backs BOs with anonymous shmem pages that can be pinned, mmap'd to userspace, and exported as DMA-BUF. Best for SoC-integrated NPUs where DRAM is shared.
- **DMA coherent** (used by `qaic` for DBC rings): pre-allocates coherent DMA memory at device init. Best for fixed-size ring buffers with hard real-time latency requirements.
- **Custom allocator** (used by `habanalabs` and `amdxdna`): device has dedicated HBM or device DRAM not visible to the CPU; the driver manages an IOVA allocator and device-side memory map. Necessary for discrete accelerators with on-device memory.

### Step 2: Define Driver Feature Flags

```c
/* Example driver registration */
static const struct drm_driver my_npu_driver = {
    .driver_features = DRIVER_GEM | DRIVER_COMPUTE_ACCEL,
    /* DRIVER_SYNCOBJ if using dma_fence / drm_syncobj for explicit sync */
    .name     = "my_npu",
    .desc     = "My NPU Accelerator",
    .major    = 1,
    .minor    = 0,
    .fops     = &my_npu_fops,  /* DEFINE_DRM_ACCEL_FOPS(my_npu_fops) */
    .ioctls   = my_npu_ioctls,
    .num_ioctls = ARRAY_SIZE(my_npu_ioctls),
    .open     = my_npu_open,
    .postclose = my_npu_postclose,
};
```

### Step 3: Implement Per-fd Context Management

Every DRM file in an accel driver represents an isolated user context. The `open` callback allocates per-fd resources:

```c
/* Pattern from ivpu: allocate MMU context on open */
static int my_npu_open(struct drm_device *dev, struct drm_file *file)
{
    struct my_npu_device *ndev = to_my_npu_device(dev);
    struct my_npu_file_priv *file_priv;

    file_priv = kzalloc(sizeof(*file_priv), GFP_KERNEL);
    if (!file_priv)
        return -ENOMEM;

    /* Allocate an IOMMU SSID for this context */
    file_priv->ctx_id = my_npu_alloc_context(ndev);
    if (file_priv->ctx_id < 0) {
        kfree(file_priv);
        return file_priv->ctx_id;
    }

    file->driver_priv = file_priv;
    return 0;
}
```

The `postclose` callback must release the IOMMU context and cancel any in-flight jobs.

### Step 4: Implement Buffer Object IOCTLs

At minimum, an accel driver needs IOCTLs for:
- **BO creation** (`DRM_IOCTL_*_BO_CREATE`): allocates a GEM object of given size with specified flags (cached, uncached, write-combined; device-visible; host-visible)
- **BO mmap** (`DRM_IOCTL_*_BO_MMAP` or `DRM_IOCTL_*_BO_MMAP_OFFSET`): returns a mmap offset for userspace mapping
- **BO info** (`DRM_IOCTL_*_BO_INFO`): queries buffer properties (VPU address, size, flags)
- **BO wait** (`DRM_IOCTL_*_WAIT_BO`): blocks until the last job using the BO has completed

### Step 5: Implement Job Submission

The job submission IOCTL receives a command buffer handle (GEM handle of the pre-filled command buffer) plus ancillary BO handles, validates them, maps them into the device's IOMMU domain, and rings the hardware doorbell. If using the DRM scheduler, the submission creates a `drm_sched_job` and enqueues it; the `run_job` callback executes when dependencies are satisfied.

### Step 6: Handle Firmware Loading

Most NPUs require a firmware blob loaded via `request_firmware()`. The firmware is placed in a special BO (typically allocated in a pinned, physically contiguous region) and the device's boot sequence is triggered:

```c
/* Pattern from ivpu_fw_load() */
void my_npu_fw_load(struct my_npu_device *ndev)
{
    const struct firmware *fw = ndev->fw->file;

    /* Copy firmware to the pre-allocated BO */
    memcpy_toio(ndev->fw_bo->vaddr, fw->data, fw->size);

    /* Set the cold boot entry point in hardware register */
    writel(ndev->fw->cold_boot_entry_point,
           ndev->regv + MY_NPU_REG_COLD_BOOT_ADDR);

    /* Trigger boot sequence */
    my_npu_hw_boot(ndev);
}
```

### Step 7: Register with the accel Major

Device registration uses `devm_drm_dev_alloc()` followed by `drm_dev_register()`:

```c
static int my_npu_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
    struct my_npu_device *ndev;
    int ret;

    ndev = devm_drm_dev_alloc(&pdev->dev, &my_npu_driver,
                               struct my_npu_device, drm);
    if (IS_ERR(ndev))
        return PTR_ERR(ndev);

    /* ... hardware initialisation ... */

    ret = drm_dev_register(&ndev->drm, 0);
    if (ret)
        goto err_hw_fini;

    /* DRM core sees DRIVER_COMPUTE_ACCEL, allocates an accel minor,
     * calls accel_set_device_instance_params() to assign ACCEL_MAJOR,
     * and creates /dev/accel/accelN */
    return 0;
}
```

---

## 12. Memory Management: GEM Without Display {#gem-memory}

All `accel` drivers that handle substantial data (all except `ethosu`) use the **GEM (Graphics Execution Manager)** memory manager. GEM was designed for GPU driver use, but its core infrastructure — object lifetime management via `kref`, mmap offset allocation, DMA-BUF export/import (PRIME), fence tracking — is entirely independent of display pipelines.

### GEM Object Lifecycle in Accel Drivers

Buffer object creation uses the same `drm_gem_object_init()` / `drm_gem_shmem_create()` paths as GPU drivers. The `ivpu` driver defines `struct ivpu_bo` (in `drivers/accel/ivpu/ivpu_gem.h`) extending `drm_gem_shmem_object`:

```c
/* drivers/accel/ivpu/ivpu_gem.h */
struct ivpu_bo {
    struct drm_gem_shmem_object  base;      /* DRM shmem-backed GEM object */
    struct ivpu_mmu_context     *ctx;       /* IOMMU context for this BO */
    struct list_head             bo_list_node;
    struct drm_mm_node           mm_node;   /* node in VPU VA space allocator */

    u64                          vpu_addr;  /* device-side virtual address */
    u32                          flags;     /* DRM_IVPU_BO_CACHED / READ_ONLY / etc. */
    u32                          job_status;
    u32                          ctx_id;
    bool                         mmu_mapped;
};

/* Three-level container_of chain */
static inline struct ivpu_bo *to_ivpu_bo(struct drm_gem_object *obj)
{
    return container_of(obj, struct ivpu_bo, base.base);
}

/* Retrieve the device-side virtual address */
static inline u64 ivpu_bo_vpu_addr(struct ivpu_bo *bo) { return bo->vpu_addr; }

/* Translate VPU address → CPU virtual address (within the BO) */
static inline void *ivpu_to_cpu_addr(struct ivpu_bo *bo, u32 vpu_addr)
{
    if (vpu_addr < bo->vpu_addr)
        return NULL;
    if (vpu_addr >= (bo->vpu_addr + ivpu_bo_size(bo)))
        return NULL;
    return ivpu_bo_vaddr(bo) + (vpu_addr - bo->vpu_addr);
}
```

The `ivpu_bo_bind()` function maps the BO's pages into the per-context MMU domain and assigns the `vpu_addr`. `ivpu_bo_unbind_all_bos_from_context()` unmaps all BOs when a user context is torn down. Driver-provided IOCTLs:

- `ivpu_bo_create_ioctl` → `DRM_IOCTL_IVPU_BO_CREATE`
- `ivpu_bo_info_ioctl` → `DRM_IOCTL_IVPU_BO_INFO` (returns `vpu_addr`, size, flags)
- `ivpu_bo_wait_ioctl` → `DRM_IOCTL_IVPU_BO_WAIT` (waits for the done fence)
- `ivpu_bo_create_from_userptr_ioctl` → zero-copy registration of pre-existing userspace memory

For `qaic`, a `qaic_bo` extends `drm_gem_object`:

```c
/* drivers/accel/qaic/qaic.h */
struct qaic_bo {
    struct drm_gem_object   base;       /* must be first */
    struct sg_table        *sgt;        /* scatter-gather table */
    struct list_head        slices;     /* sub-buffer regions */
    int                     dir;        /* DMA_TO_DEVICE or DMA_FROM_DEVICE */
    struct dma_bridge_chan  *dbc;       /* associated DBC channel */
    u32                     nr_slice;
    bool                    sliced;
    u16                     req_id;
    struct completion       xfer_done;
    /* performance statistics */
    struct {
        u64 req_received_ts;
        u64 req_submit_ts;
        u64 req_processed_ts;
    } perf_stats;
};
```

The `to_qaic_bo()` macro (`container_of(obj, struct qaic_bo, base)`) recovers the driver struct from a `drm_gem_object *`.

### DMA-BUF: Zero-Copy Interop

DMA-BUF (PRIME) import and export lets accel devices share buffers with other components without copying:

- **Camera pipeline to NPU**: a V4L2 camera driver exports a DMA-BUF fd; the NPU driver imports it via `gem_prime_import`, mapping the same physical pages into the NPU's IOMMU address space. The `qaic` driver implements `qaic_gem_prime_import()` for exactly this pattern.
- **GPU to NPU**: a GPU render result (e.g., a frame from `amdgpu`) can be imported into `amdxdna` for post-processing inference without a kernel-to-user-to-kernel copy round trip.
- **NPU to display**: an NPU inference result (e.g., a segmentation mask overlaid on a video frame) can be exported to a Wayland compositor's KMS plane via the same DMA-BUF mechanism.

The DMA-BUF fence model extends naturally: an accel driver can attach a `dma_fence` to a buffer at export time, and the consumer (GPU, display, V4L2 encoder) can wait on the fence using `dma_fence_wait()` or the explicit sync IOCTL path.

### IOMMU and Address Isolation

`ivpu` uses a dedicated IOMMU domain per user context, assigning each a unique MMU SSID. `amdxdna` uses IOMMU SVA for direct process VA mapping. `habanalabs` manages its own MMU with a software page table. `qaic` relies on DMA coherent memory (pre-allocated rings) and scatter-gather for inference data.

The common theme is that GEM object backing pages are mapped into the device's IOMMU address space on submit and unmapped on completion (or pinned for the session lifetime), with no user involvement beyond the GEM handle.

---

## 13. DRM GPU Scheduler Integration {#drm-sched}

The DRM GPU scheduler (`drivers/gpu/drm/scheduler/`, header `include/drm/gpu_scheduler.h`) is optional for `accel` drivers but available. It provides job queuing, multi-queue round-robin scheduling, dependency resolution via `dma_fence` chains, and timeout/reset handling. The `ivpu` driver uses it via `struct drm_sched_job` and `struct drm_sched_entity`.

The scheduler's role in `ivpu` submission flow:

1. Userspace submits via `DRM_IOCTL_IVPU_SUBMIT`.
2. The driver creates a `struct ivpu_job` and calls `drm_sched_job_init()`, associating it with the per-context `drm_sched_entity`.
3. `drm_sched_entity_push_job()` enqueues the job; the scheduler's run-queue thread calls the driver's `run_job` callback when the job's input fences are satisfied.
4. `run_job` writes the VPU command buffer address to the shared job queue ring and rings the doorbell.
5. On firmware completion interrupt, `ivpu_job_done_consumer` signals the `done_fence` attached to the job, completing the scheduler cycle.

For drivers that forgo the scheduler (like `habanalabs` for some submission modes), the driver manages its own per-queue locking and serialisation. The scheduler is most valuable when multiple user contexts compete for the same hardware queue and ordered completion matters.

---

## 14. Security Model and Access Control {#security}

### Device Permissions

`/dev/accel/accel0` is created with mode `0600` by default (readable/writable only by root) unless modified by a udev rule. In practice, distributions ship udev rules that add members of the `render` or `accel` group:

```bash
# Typical udev rule for accel devices
KERNEL=="accel[0-9]*", SUBSYSTEM=="accel", TAG+="uaccess", MODE="0660", GROUP="render"
```

The `TAG+="uaccess"` rule integrates with systemd's `pam_systemd` to grant the logged-in console user transient access. This is the same mechanism used for `/dev/dri/renderD*`, making the accel permission model familiar to system administrators already managing GPU access.

### Render-Node Analogy

Accel device nodes operate analogously to DRM render nodes: any process that opens the fd and passes `DRM_RENDER_ALLOW`-flagged IOCTLs can submit work without requiring DRM master status. There is no concept of display ownership, master lease, or atomic modesetting privileges — those DRM mechanisms simply do not exist for `DRIVER_COMPUTE_ACCEL` drivers.

`IOCTL_DRM_HL_*` and `IOCTL_DRM_IVPU_*` calls all carry `DRM_RENDER_ALLOW`, meaning any process with read/write permission on the device node can submit inference workloads. Privilege separation beyond file permission is left to the hardware: per-context IOMMU domains (ivpu), per-process PASID (amdxdna), or per-DBC user locking (qaic).

### Multi-Tenant Isolation

Datacenter deployments serving multiple tenants simultaneously rely on hardware isolation mechanisms:

- **ivpu**: each open fd gets a unique MMU SSID (up to 128 user contexts per device). The NPU firmware enforces that context N cannot read or write SSID M's memory. A buggy or malicious user cannot exfiltrate another user's model weights or activations.
- **amdxdna**: IOMMU SVA with per-process PASID. The IOMMU hardware enforces that the NPU can only access pages belonging to the submitting process's address space, under the assumption that the process's own memory is protected by the OS.
- **qaic**: per-DBC user locking (`dbc->usr`) ensures that once a user activates a DBC, no other user can submit requests to that channel until it is released.

### cgroup Device Controller

In containerised deployments (Docker, containerd, Kubernetes), access to `/dev/accel/accel0` is controlled by the **cgroup v2 device controller** (`devices.allow` / `devices.deny`). The Kubernetes GPU device plugin model extends naturally: a device plugin for NPUs can advertise `intel.com/npu: 1` resources, allocate specific `/dev/accel/accelN` devices to pods, and use CDI (Container Device Interface) to inject the device path and required udev context into the container namespace. See Chapter 55 (GPU Containers and Cloud Compute) for the general container device framework.

---

## 15. Userspace Integration: Runtimes and Inference Engines {#userspace}

### ONNX Runtime Execution Providers

The **ONNX Runtime** (ORT) uses an *execution provider* (EP) model: a loaded model graph is partitioned into subgraphs, each dispatched to a hardware-specific EP backend. The relevant EPs for Linux `accel` devices:

- **OpenVINO EP** (`OrtSessionOptionsAppendExecutionProvider_OpenVINO`): delegates to Intel's OpenVINO toolkit, which in turn drives `ivpu` on Meteor Lake and later. The EP opens `/dev/accel/accel0`, compiles the ONNX graph to a VPU-compatible internal representation, allocates GEM BOs for the compiled model blob, and submits inference via `DRM_IOCTL_IVPU_SUBMIT`.
- **QNN EP** (`OrtSessionOptionsAppendExecutionProvider_QNN`): for Qualcomm Hexagon HTP on Snapdragon (mobile/Windows); the Cloud AI 100 (`qaic`) uses its own `qaicrt` runtime rather than ONNX Runtime's QNN EP.

### OpenVINO and the Intel NPU

OpenVINO is Intel's primary inference toolkit and the dominant userspace layer above `ivpu`. Its compilation flow:

1. Model loading: ONNX, TensorFlow, PaddlePaddle models are ingested via the OpenVINO Model Optimizer.
2. Graph compilation: the `intel_npu_driver` plugin (`ze_intel_npu`) compiles the graph to a binary blob using the NPU's closed firmware ABI. The compiler toolchain is part of the `intel-npu-driver` package.
3. Buffer allocation: activations and weights are allocated as GEM BOs via `DRM_IOCTL_IVPU_BO_CREATE`.
4. Inference: the compiled blob is submitted as a command buffer via `DRM_IOCTL_IVPU_SUBMIT`.

### PyTorch and Gaudi

For `habanalabs` Gaudi, the PyTorch Habana plugin (`habana_frameworks.torch`) intercepts `torch` operations and maps them to Gaudi command buffers. The plugin uses `hlthunk` to manage GEM BO allocation, command buffer construction, and `HL_CS` submission. Training throughput on Gaudi 3 is a primary use case — the driver's support for staged submissions and encapsulated collective signals enables the distributed training communication patterns (all-reduce over NIC engines) that large model training requires.

### Comparison with CUDA and ROCm

| Aspect | CUDA (`/dev/nvidiactl`, `/dev/nvidia0`) | ROCm (`/dev/kfd`) | accel (`/dev/accel/accel0`) |
|--------|----------------------------------------|--------------------|-----------------------------|
| Framework | `libcuda.so` → CUDA driver | `libhsa-runtime64.so` → KFD | `hlthunk`/OpenVINO/qaicrt |
| Memory | `cuMemAlloc` → UVM | `hsa_memory_allocate` | GEM `drm_ioctl_*_bo_create` |
| Submission | Ring buffer via `nvidia-modeset` | HSA AQL packets to KFD | Driver-specific IOCTLs via DRM |
| Fence | `cuStreamSynchronize` | HSA signal | `DMA_FENCE` / IOCTL wait |
| Programmability | Full CUDA C++ compilation | Full HIP/ROCm stack | Fixed firmware; model compilation done off-device |

The key distinction is **programmability**: CUDA and ROCm drivers support fully general-purpose shader-like kernels with custom compiled GPU programs. Accel devices (with the possible exception of Gaudi's TPC engines, which are programmable but with a specialised ISA and a closed firmware interface) operate on pre-compiled model artifacts. This is why the `accel` subsystem does not require LLVM or Mesa integration — there is no kernel-side shader compiler to host.

---

## 16. When to Use `/dev/accel` vs `/dev/dri/renderD*` vs `/dev/kfd` {#device-choice}

Systems designers frequently face the question of which device node to target for AI inference workloads on Linux.

### `/dev/dri/renderD128` (GPU Compute via DRM render node)

Use when:
- The target hardware is a GPU (NVIDIA, AMD, Intel Arc, Qualcomm Adreno GPU)
- The workload can be expressed in Vulkan compute, OpenCL, or CUDA shaders
- Training (not just inference) is required
- The inference engine uses Vulkan compute (llama.cpp with `vulkan` backend, MLC LLM, ExecuTorch Vulkan)
- GPU graphics and compute sharing is needed (e.g., ML post-processing on rendered frames)

### `/dev/kfd` (AMD ROCm)

Use when:
- Hardware is AMD RDNA/CDNA GPU or APU
- Workload uses ROCm/HIP (PyTorch ROCm, JAX/ROCm, TensorFlow ROCm)
- Access to ROCm-specific CDNA features (HBM, FP8, XGMI) is required

### `/dev/accel/accelN` (accel subsystem)

Use when:
- Hardware is a dedicated NPU or inference accelerator (Intel Meteor Lake+ NPU, AMD Ryzen AI XDNA, Qualcomm Cloud AI 100, Gaudi)
- Workload is fixed inference (ONNX model, pre-compiled artifact)
- The inference runtime is OpenVINO, hlthunk, qaicrt, or XRT
- Low power consumption relative to GPU inference is a priority (embedded/laptop inference)
- The hardware offers no general shader programmability

### The Grey Area: GPU NPU Blocks

Modern laptop SoCs blur these boundaries. Intel Meteor Lake has both:
- An integrated GPU (Xe-LP) managed by `i915`/`Xe` → `/dev/dri/renderD128`
- An NPU 3720 managed by `ivpu` → `/dev/accel/accel0`

AMD Phoenix/Hawk Point has:
- An iGPU (RDNA3) managed by `amdgpu` → `/dev/dri/renderD128` and `/dev/kfd`
- An XDNA NPU managed by `amdxdna` → `/dev/accel/accel0`

Inference engines must select between these based on model type and latency requirements. INT8 transformer inference at low batch size is often more efficient on the NPU (lower power); FP16 or BF16 inference at higher batch size may be faster on the GPU. OpenVINO, ONNX Runtime, and ROCm's Vitis AI inference stack each provide profiling tools to guide this selection.

---

## Roadmap

### Near-term (6–12 months)

- **Intel NPU 5 (Panther Lake) full support**: Linux 6.13 merged kernel-side `ivpu` support for Panther Lake's NPU 5, and the Intel user-space NPU driver v1.28.0 added Panther Lake userspace support. Continued firmware and performance work targeting Nova Lake (Core Ultra Series 4, late 2026) is underway. [Source: Phoronix — Intel NPU 5 Panther Lake Linux](https://www.phoronix.com/news/Intel-NPU-5-Panther-Lake-Linux)
- **NXP Neutron NPU upstream inclusion**: NXP posted a v2 patch series (`accel: New driver for NXP's Neutron NPU`) in March 2026 for the Neutron NPU in the i.MX95 SoC, accompanied by an open-source userspace library and LiteRT delegate. The series is under review for a future mainline kernel window. [Source: LKML patch v2](https://lkml.iu.edu/hypermail/linux/kernel/2603.0/11337.html)
- **AMD XDNA power metrics and Strix Halo support**: Patches to expose AMD Ryzen AI NPU power and thermal metrics via the `amdxdna` driver are in review, alongside continued support additions for Strix Halo (Ryzen AI Max) and later XDNA generations. [Source: Phoronix — Ryzen AI NPU Power Metrics](https://www.phoronix.com/news/Ryzen-AI-NPU-Linux-Power-Metric)
- **Two additional unnamed accel drivers expected in 2026**: Phoronix reported that at least two new open-source NPU accelerator drivers were expected to land in the Linux kernel during 2026, based on developer statements from Tomeu Vizoso (author of the `rocket` Rockchip NPU driver). [Source: Phoronix](https://www.phoronix.com/news/Two-NPU-Accel-Drivers-2026)
- **Arm Ethos-U85 open-source driver review**: ARM's open-source kernel accelerator driver for Ethos-U65/U85 NPUs continues to undergo mailing-list review. A Mesa Gallium3D companion driver for Ethos NPUs was also merged, establishing a fuller open stack. [Source: Phoronix — Arm Ethos Gallium Merged](https://www.phoronix.com/news/Arm-Ethos-Gallium-Merged)

### Medium-term (1–3 years)

- **Arm China Zhouyi (Compass) NPU upstreaming**: Arm China initiated a mailing-list discussion in March 2024 about upstreaming their Zhouyi NPU driver (with both open kernel and userspace stacks) into the mainline `accel` subsystem. Maintainer feedback established that it must fit within standard `accel` interfaces; the path to inclusion remains open but requires further review cycles. [Source: dri-devel thread](https://lore.kernel.org/dri-devel/20240328103205.seht2hbog3o4giv5@bogus/T/)
- **Shared Virtual Memory (SVM) interop between NPU and GPU**: Intel's Xe driver merged multi-device SVM support (Project Battlematrix) targeting Linux 6.20/7.0, enabling unified virtual address spaces across multiple Intel Arc and AI inference accelerator cards. Extending similar SVM semantics into the `accel` subsystem — so that NPU and GPU can share virtual address ranges without copy — is a natural next step for heterogeneous inference pipelines. [Source: Phoronix — Intel Multi-Device SVM](https://www.phoronix.com/news/Intel-Multi-Device-SVM-Linux-7)
- **Standardised sysfs/debugfs conventions across accel drivers**: Current drivers (`ivpu`, `amdxdna`, `habanalabs`, `qaic`, `rocket`) expose performance counters, firmware version, and hardware telemetry through inconsistent debugfs layouts. Pressure from system-management tools (e.g., `xrt-smi` for XDNA) and container runtimes is expected to drive a common sysfs schema for accel devices analogous to the DRM GPU stats interface. Note: needs verification as a formal kernel proposal.
- **CDI (Container Device Interface) standardisation for `/dev/accel/`**: Kubernetes device plugins and CDI injection already support `/dev/accel/accelN` devices in practice, but formal CDI spec coverage and `nvidia-container-toolkit`-style wrappers for Intel and AMD NPUs are nascent. Gaudi 3 cloud deployments are driving this work. [Source: kernel docs — accel introduction](https://www.kernel.org/doc/html/latest/accel/introduction.html)
- **ONNX Runtime and OpenVINO accel backend stabilisation**: As NPU hardware proliferates in laptop SoCs, the OpenVINO NPU execution provider for ONNX Runtime and the llama.cpp OpenVINO backend are expected to stabilise against a more consistent kernel ABI surface. Note: needs verification against specific ONNX RT release schedules.

### Long-term

- **Unified accel memory management layer**: The DRM GEM layer was designed for GPU memory with display export in mind; dedicated inference accelerators with physically separate DRAM (Gaudi, Cloud AI 100) or tightly coupled SRAM (Ethos-U) need different eviction and placement policies. A long-term architectural goal is a common memory manager within the `accel` subsystem that abstracts device-local DRAM, host-pinned memory, and SVM regions without layering on GPU-centric GEM assumptions. Note: needs verification as a formal RFC.
- **Cross-vendor inference scheduling**: With multiple NPUs coexisting on a single SoC (Intel iGPU + Intel NPU, AMD RDNA iGPU + XDNA NPU) or in a server node (GPU + Gaudi), a kernel-level heterogeneous scheduler that can migrate or pipeline inference tasks across `/dev/dri/renderD*` and `/dev/accel/accelN` is a speculative but technically motivated direction. Note: no formal proposal known as of mid-2026.
- **Mainlining of closed-firmware accelerators from additional vendors**: Following the precedent set by `habanalabs` and `qaic`, further data-centre AI accelerators (from vendors such as Graphcore, Cerebras, or emerging Chinese AI chip vendors subject to export control constraints) may seek `drivers/accel/` inclusion if they can satisfy the open-firmware or stable-ABI requirements the subsystem maintainers have articulated. [Source: LWN — new subsystem for compute accelerator devices](https://lwn.net/Articles/915509/)
- **OpenCL or Vulkan compute shim for fixed-function NPUs**: For NPUs that expose a narrow tile-based execution model (Ethos-U, XDNA at low-level), a thin Vulkan compute or OpenCL shim translating standard dispatch calls into NPU command streams would enable standard ML frameworks to target `/dev/accel/` devices without vendor-specific runtimes. This mirrors what the Mesa Ethos Gallium3D driver has begun exploring. [Source: Phoronix — Arm Ethos Gallium Merged](https://www.phoronix.com/news/Arm-Ethos-Gallium-Merged)

---

## 17. Integrations {#integrations}

**Chapter 5 — x86 GPU Drivers**: The Intel `ivpu` driver for the NPU sits alongside `i915`/`Xe` (Intel GPU driver) on Meteor Lake systems. Both are independent PCI functions; `ivpu` uses `DRIVER_COMPUTE_ACCEL` while `i915`/`Xe` use `DRIVER_RENDER` + `DRIVER_MODESET`. The auxiliary bus pattern for combined GPU+NPU devices is discussed there in the context of Intel Xe.

**Chapter 25 — GPU Compute**: GPU compute via DRM render nodes (`DRIVER_RENDER`), OpenCL (rusticl, intel-compute-runtime), and Vulkan compute represents the GPU side of the AI inference landscape. Chapter 25 covers the DRM scheduler, GEM BOs for compute, and DMA-BUF interop — all infrastructure that `accel` drivers share by virtue of being built on DRM.

**Chapter 48 — ROCm and Machine Learning on Linux GPUs**: AMD's ROCm stack uses `/dev/kfd` (the KFD compute queue via `amdkfd`) for GPU training and inference. The `amdxdna` NPU driver operates independently via `/dev/accel/accel0`. The chapter on ROCm explains the KFD/HSA model and the PyTorch/TensorFlow ROCm paths.

**Chapter 55 — GPU Containers and Cloud Compute**: cgroup v2 device controller, Kubernetes device plugins, and CDI device injection apply equally to `/dev/accel/accelN` devices as to GPU render nodes. Gaudi cloud deployments (AWS DL1 instances use Gaudi) are a primary production use case for the `habanalabs` driver in containerised environments.

**Chapter 88 — NPU and AI Accelerator Integration on Linux**: the user-facing view of the stack built on `drivers/accel/`. Chapter 88 covers OpenVINO toolkit configuration, ONNX Runtime EP selection, power management for NPU workloads, and benchmarking methodology. The present chapter covers the kernel driver internals that Chapter 88's userspace stack sits above.

**Chapter 108 — ROCm and HIP: AMD's GPU Compute Stack**: the AMD GPU side of the inference landscape. `amdxdna` and `amdgpu` are separate drivers for distinct AMD silicon on the same APU; Chapter 108 explains the GPU side while this chapter covers the NPU.

**Chapter 124 — Local LLM Inference on Linux GPUs**: covers llama.cpp, Ollama, and MLC LLM running on GPU and NPU backends. The backend selection logic (Vulkan, OpenCL, Metal, CUDA, NPU) maps directly to the device node choices described in Section 14 of this chapter. OpenVINO-backed llama.cpp inference on Intel NPU uses the `ivpu` driver described here.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
