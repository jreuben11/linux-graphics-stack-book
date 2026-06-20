# Chapter 149: GPU Hang Detection and Recovery — TDR, Scheduling Timeouts, and Reset Sequences

**Target audiences:** Kernel DRM driver developers implementing hang detection and reset sequences; GPU application developers diagnosing and handling `VK_ERROR_DEVICE_LOST`; CI/QA engineers debugging intermittent GPU hangs in test pipelines.

---

## Table of Contents

1. [Introduction — Why GPU Hangs Are Different](#1-introduction--why-gpu-hangs-are-different)
2. [Linux GPU Hang Taxonomy](#2-linux-gpu-hang-taxonomy)
3. [The DRM GPU Scheduler Timeout Mechanism](#3-the-drm-gpu-scheduler-timeout-mechanism)
4. [AMD amdgpu Hang Detection and Recovery](#4-amd-amdgpu-hang-detection-and-recovery)
5. [Intel i915 Hang Detection and Reset](#5-intel-i915-hang-detection-and-reset)
6. [Intel Xe Driver Reset Path](#6-intel-xe-driver-reset-path)
7. [NVIDIA Open Kernel Module (nouveau and GSP-RM)](#7-nvidia-open-kernel-module-nouveau-and-gsp-rm)
8. [Kernel Error Reporting and dmesg Patterns](#8-kernel-error-reporting-and-dmesg-patterns)
9. [IOMMU Faults from the GPU](#9-iommu-faults-from-the-gpu)
10. [Userspace Recovery — VK_ERROR_DEVICE_LOST and VK_EXT_device_fault](#10-userspace-recovery--vk_error_device_lost-and-vk_ext_device_fault)
11. [CI and Testing — Injecting and Verifying Recovery](#11-ci-and-testing--injecting-and-verifying-recovery)
12. [Tools and Monitoring](#12-tools-and-monitoring)
13. [Integrations](#13-integrations)

---

## 1. Introduction — Why GPU Hangs Are Different

A CPU crash delivers a well-defined signal. The hardware traps, the kernel receives an exception with a precise faulting instruction pointer, a core dump is written, and the offending process dies cleanly. The rest of the system continues. GPU hangs behave nothing like this.

When a GPU stops making forward progress — because a shader entered an infinite loop, a command buffer pointed to corrupted memory, or a microcontroller firmware fault silenced the interrupt line — the kernel receives *no notification at all*. The hardware simply becomes quiet. Pending fences never signal. The DRM scheduler's pending job list grows but never shrinks. The Wayland compositor waits for buffer release fences that will never arrive. The desktop freezes.

**On Windows**, the Timeout Detection and Recovery (TDR) subsystem is mandatory for WDDM drivers. A kernel watchdog timer fires after two seconds (by default, configurable via `TdrDelay`) and the display driver is forcibly reloaded. Applications receive `D3DDDIERR_DEVICEREMOVED`. The experience is jarring but defined.

**On Linux**, there is no kernel-wide mandatory TDR equivalent. Instead, each DRM driver registers a timeout with the DRM GPU scheduler (`drm_gpu_scheduler`), which fires a work queue item if any job exceeds the configured deadline. What happens next is entirely up to the driver's `timedout_job` callback: some drivers attempt a per-ring reset, some attempt a full GPU reset, some simply kill the offending context and let other clients continue.

The consequences of a GPU hang depend on its severity:

- **Soft hang, successfully recovered**: the GPU scheduler kills the hung job, resets the offending engine, and signals pending fences with `-ETIMEDOUT` or `-EIO`. Other clients continue. The affected application receives an error return.
- **Hard hang, full GPU reset**: all GPU clients lose their in-flight work. Applications receive `VK_ERROR_DEVICE_LOST` or `EGL_CONTEXT_LOST`. The compositor typically survives if it uses DRM direct scanout and can re-submit.
- **Unrecoverable hang**: the GPU never responds to reset commands. The kernel wedges the device (marks it unusable), userspace processes receive permanent errors, and a system reboot is required.

This chapter traces the complete hang detection and recovery path from the scheduler timeout through driver-specific reset sequences to the userspace API surface.

All source paths are pinned to kernel commit [`8c13415c8a43`](https://github.com/torvalds/linux/tree/8c13415c8a4383447c21ec832b20b3b283f0e01a) (Linux `master`, post-v6.19, targeting v6.20, June 2026).

---

## 2. Linux GPU Hang Taxonomy

GPU hangs form a spectrum. Understanding where a failure falls determines which recovery path is appropriate.

### 2.1 Soft Hang — Scheduler Timeout

The GPU is still alive and still generating interrupts, but a particular job has not completed within the configured timeout window. The GPU may be stuck in a shader loop (no preemption on most consumer hardware), waiting for a fence dependency that itself will never signal, or executing an unusually long compute dispatch.

The DRM GPU scheduler detects this: `work_tdr` (the delayed timeout-detection-recovery work item) fires when the pending-list head job has been waiting longer than `sched->timeout`. Because the GPU still responds to interrupts, a selective per-ring reset is often sufficient.

### 2.2 Hard Hang — GPU Stops Generating Interrupts

No completions arrive. The scheduler's TDR fires. The driver tries to communicate with the GPU via register reads and gets back `0xffffffff` or a PCIe link-down indicator. This is the worst single-chip scenario: the PCIe function may need to be reset (BACO or PCI FLR), which means all engines and all VRAM content are lost.

### 2.3 Firmware Crash

The GPU's onboard microcontroller — AMD uses the ME, PFP, CE, and RLC microcontrollers for the GFX ring, and the MES (Micro Engine Scheduler) microcontroller for hardware scheduling; NVIDIA uses the GSP-RM (GPU System Processor — Resource Manager) on Turing and later — crashes or stops responding. The firmware watchdog timer on the chip then triggers a GPU reset signal that the host driver receives as an interrupt.

For NVIDIA's open kernel module, `nvidia-open`, the GSP-RM runs the entire driver stack in firmware. A GSP-RM crash leaves the OS kernel driver with limited ability to recover; typically a full device unbind/rebind is required.

### 2.4 IOMMU Fault

The GPU's DMA engine accesses a GPU virtual address that is not mapped in the IOMMU page table. On x86 systems with the AMD IOMMU or Intel VT-d, and on ARM systems with the ARM SMMU, this generates a page-fault interrupt to the CPU. The driver's fault handler logs the faulting GPU VA, kills the offending process context, and resets the affected ring. See §9.

### 2.5 ECC / RAS Uncorrectable Error

On data-centre GPUs (NVIDIA A100/H100, AMD CDNA2/CDNA3 Instinct, AMD RDNA3 Pro) and optionally consumer GPUs with ECC enabled, single-bit errors are corrected silently by hardware ECC. Multi-bit (uncorrectable) errors generate an interrupt, log a RAS event, and trigger an immediate device-lost condition. VRAM contents cannot be trusted; all contexts must be destroyed and re-initialised.

---

## 3. The DRM GPU Scheduler Timeout Mechanism

The DRM GPU scheduler (`drivers/gpu/drm/scheduler/`) provides the generic timeout and recovery framework. Every driver that uses `drm_gpu_scheduler` gets hang detection for free; the driver only needs to implement `timedout_job`.

[Source: `include/drm/gpu_scheduler.h`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/include/drm/gpu_scheduler.h), [`drivers/gpu/drm/scheduler/sched_main.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/scheduler/sched_main.c)

### 3.1 Initialisation — `drm_sched_init`

A driver initialises a scheduler instance with `drm_sched_init()`, which now takes a consolidated `drm_sched_init_args` structure (refactored in the v6.12 timeframe to reduce argument count):

```c
/* include/drm/gpu_scheduler.h */
struct drm_sched_init_args {
    const struct drm_sched_backend_ops *ops;
    struct workqueue_struct            *submit_wq;    /* optional; created if NULL */
    struct workqueue_struct            *timeout_wq;   /* optional; uses system WQ if NULL */
    u32                                 credit_limit; /* max in-flight job credits */
    unsigned int                        hang_limit;   /* max resets before wedge */
    long                                timeout;      /* hang deadline in jiffies */
    atomic_t                           *score;        /* load metric for LB */
    const char                         *name;
    struct device                      *dev;
};

int drm_sched_init(struct drm_gpu_scheduler *sched,
                   const struct drm_sched_init_args *args);
```

The Panfrost driver (Mali GPU, `drivers/gpu/drm/panfrost/panfrost_job.c`) demonstrates a typical initialisation with a 500 ms timeout:

```c
/* drivers/gpu/drm/panfrost/panfrost_job.c — panfrost_jm_init() */
struct drm_sched_init_args args = {
    .ops         = &panfrost_sched_ops,
    .credit_limit = 2,
    .timeout     = msecs_to_jiffies(JOB_TIMEOUT_MS), /* 500 ms */
    .dev         = pfdev->base.dev,
};

ret = drm_sched_init(&pfdev->js->sched[i], &args);
```

[Source: `drivers/gpu/drm/panfrost/panfrost_job.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/panfrost/panfrost_job.c)

### 3.2 The `timedout_job` Callback

The driver registers its callback in `drm_sched_backend_ops`:

```c
/* include/drm/gpu_scheduler.h */
struct drm_sched_backend_ops {
    struct dma_fence *(*prepare_job)(struct drm_sched_job *,
                                     struct drm_sched_entity *);
    struct dma_fence *(*run_job)(struct drm_sched_job *);
    enum drm_gpu_sched_stat (*timedout_job)(struct drm_sched_job *);
    void (*free_job)(struct drm_sched_job *);
    void (*cancel_job)(struct drm_sched_job *);  /* optional */
};
```

The callback must return one of:

```c
enum drm_gpu_sched_stat {
    DRM_GPU_SCHED_STAT_NONE,     /* unused */
    DRM_GPU_SCHED_STAT_RESET,    /* deprecated alias */
    DRM_GPU_SCHED_STAT_ENODEV,   /* device gone; do not restart TDR timer */
    DRM_GPU_SCHED_STAT_NO_HANG,  /* false alarm; re-insert job, restart timer */
};
```

> **Note on `DRM_GPU_SCHED_STAT_RESET`**: In `drm_sched_job_timedout()`, `status` is initialised to `DRM_GPU_SCHED_STAT_RESET` as the default path (indicating a reset was performed). The semantically meaningful non-default returns are `DRM_GPU_SCHED_STAT_ENODEV` (device gone — do not restart the TDR timer) and `DRM_GPU_SCHED_STAT_NO_HANG` (spurious timeout — re-insert the job and restart the timer). Any other return (including `RESET`) causes the TDR timer to restart with the normal timeout, ready for the next pending job. [Source: mailing-list discussion](https://lkml.iu.edu/hypermail/linux/kernel/2501.2/03821.html)

### 3.3 `drm_sched_job_timedout` — The TDR Work Item

The TDR mechanism uses a `delayed_work` item (`work_tdr`) queued at `sched->timeout` jiffies after the job at the head of `pending_list` is dispatched. When it fires:

```c
/* drivers/gpu/drm/scheduler/sched_main.c — drm_sched_job_timedout() */
static void drm_sched_job_timedout(struct work_struct *work)
{
    struct drm_gpu_scheduler *sched =
        container_of(work, struct drm_gpu_scheduler, work_tdr.work);
    enum drm_gpu_sched_stat status = DRM_GPU_SCHED_STAT_RESET;
    struct drm_sched_job *job;

    spin_lock(&sched->job_list_lock);
    job = list_first_entry_or_null(&sched->pending_list,
                                   struct drm_sched_job, list);
    if (job) {
        list_del_init(&job->list);
        spin_unlock(&sched->job_list_lock);

        status = job->sched->ops->timedout_job(job);

        if (sched->free_guilty) {
            job->sched->ops->free_job(job);
            sched->free_guilty = false;
        }

        if (status == DRM_GPU_SCHED_STAT_NO_HANG)
            drm_sched_job_reinsert_on_false_timeout(sched, job);
    } else {
        spin_unlock(&sched->job_list_lock);
    }

    if (status != DRM_GPU_SCHED_STAT_ENODEV)
        drm_sched_start_timeout_unlocked(sched);
}
```

Key design choices visible here:

- The job is **removed from `pending_list`** before calling `timedout_job`. This prevents the job from being seen as pending during the reset, and the driver decides whether to re-insert it (via `DRM_GPU_SCHED_STAT_NO_HANG`) or discard it.
- The TDR timer is **restarted** after recovery (unless the device is gone), so a new timeout window is set for the next pending job.
- `sched->free_guilty`: if set by the driver inside `timedout_job`, the scheduler calls `free_job` on the timed-out job immediately after the callback returns, releasing its resources.

### 3.4 `drm_sched_fault` — Immediate TDR Trigger

When a driver's IRQ handler detects a hardware fault *without* waiting for the scheduler timeout (e.g., an IOMMU fault interrupt fires), it can call:

```c
void drm_sched_fault(struct drm_gpu_scheduler *sched);
```

This fires the `work_tdr` delayed work with a delay of zero (via `mod_delayed_work(..., 0)`), forcing immediate TDR execution. The path through `drm_sched_job_timedout` is identical to a natural timeout.

### 3.5 Per-Ring vs. Per-VM Hang Detection

Drivers with multiple independent engines (amdgpu's GFX, compute, SDMA, VCN rings) initialise a separate `drm_gpu_scheduler` per engine. A hang on the GFX ring fires only that ring's TDR; the compute and SDMA schedulers are unaffected. Only if the per-ring reset fails, or if the driver determines the hang is GPU-wide, does a full-device reset occur.

Some drivers implement per-VM (per address space) hang tracking on top of this: a context that triggers repeated TDR events gets marked guilty, its virtual address space is banned, and the GPU continues serving other VMs. This is AMD's `amdgpu_vm_task_info` guilty-context tracking.

---

## 4. AMD amdgpu Hang Detection and Recovery

The amdgpu driver (`drivers/gpu/drm/amd/amdgpu/`) implements one of the most complete hang recovery paths in the Linux kernel, supporting multiple reset modes (soft recovery, per-ring reset, BACO, Mode1, Mode2, PCI FLR) and integrating with the RAS (Reliability, Availability, Serviceability) subsystem.

[Source: `drivers/gpu/drm/amd/amdgpu/amdgpu_job.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/amd/amdgpu/amdgpu_job.c), [`drivers/gpu/drm/amd/amdgpu/amdgpu_device.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/amd/amdgpu/amdgpu_device.c)

### 4.1 `amdgpu_job_timedout` — Scheduler Callback

This is the `timedout_job` callback registered in `amdgpu_sched_ops`. The following is the verbatim function body from upstream:

```c
/* drivers/gpu/drm/amd/amdgpu/amdgpu_job.c (lines 88–189) */
static enum drm_gpu_sched_stat amdgpu_job_timedout(struct drm_sched_job *s_job)
{
	struct amdgpu_ring *ring = to_amdgpu_ring(s_job->sched);
	struct amdgpu_job *job = to_amdgpu_job(s_job);
	struct drm_wedge_task_info *info = NULL;
	struct amdgpu_task_info *ti = NULL;
	struct amdgpu_device *adev = ring->adev;
	int idx, r;

	if (!drm_dev_enter(adev_to_drm(adev), &idx)) {
		dev_info(adev->dev, "%s - device unplugged skipping recovery on scheduler:%s",
			 __func__, s_job->sched->name);
		return DRM_GPU_SCHED_STAT_ENODEV;
	}

	/*
	 * Do the coredump immediately after a job timeout to get a very
	 * close dump/snapshot/representation of GPU's current error status.
	 * Skip it for SRIOV, since VF FLR will be triggered by host driver
	 * before job timeout.
	 */
	if (!amdgpu_sriov_vf(adev))
		amdgpu_job_core_dump(adev, job);

	if (amdgpu_gpu_recovery &&
	    amdgpu_ring_is_reset_type_supported(ring, AMDGPU_RESET_TYPE_SOFT_RESET) &&
	    amdgpu_ring_soft_recovery(ring, job->vmid, s_job->s_fence->parent)) {
		dev_err(adev->dev, "ring %s timeout, but soft recovered\n",
			s_job->sched->name);
		goto exit;
	}

	dev_err(adev->dev, "ring %s timeout, signaled seq=%u, emitted seq=%u\n",
		job->base.sched->name, atomic_read(&ring->fence_drv.last_seq),
		ring->fence_drv.sync_seq);

	ti = amdgpu_vm_get_task_info_pasid(ring->adev, job->pasid);
	if (ti) {
		amdgpu_vm_print_task_info(adev, ti);
		info = &ti->task;
	}

	/* attempt a per ring reset */
	if (amdgpu_gpu_recovery &&
	    amdgpu_ring_is_reset_type_supported(ring, AMDGPU_RESET_TYPE_PER_QUEUE) &&
	    ring->funcs->reset) {
		dev_err(adev->dev, "Starting %s ring reset\n",
			s_job->sched->name);
		drm_sched_wqueue_stop(&ring->sched);
		r = amdgpu_ring_reset(ring, job->vmid, job->hw_fence);
		if (!r) {
			drm_sched_wqueue_start(&ring->sched);
			atomic_inc(&ring->adev->gpu_reset_counter);
			dev_err(adev->dev, "Ring %s reset succeeded\n",
				ring->sched.name);
			goto exit;
		}
		dev_err(adev->dev, "Ring %s reset failed\n", ring->sched.name);
	}

	if (dma_fence_get_status(&s_job->s_fence->finished) == 0)
		dma_fence_set_error(&s_job->s_fence->finished, -ETIME);

	if (amdgpu_device_should_recover_gpu(ring->adev)) {
		struct amdgpu_reset_context reset_context;
		memset(&reset_context, 0, sizeof(reset_context));

		reset_context.method = AMD_RESET_METHOD_NONE;
		reset_context.reset_req_dev = adev;
		reset_context.src = AMDGPU_RESET_SRC_JOB;
		clear_bit(AMDGPU_NEED_FULL_RESET, &reset_context.flags);
		set_bit(AMDGPU_SKIP_COREDUMP, &reset_context.flags);

		r = amdgpu_device_gpu_recover(ring->adev, job, &reset_context);
		if (r)
			dev_err(adev->dev, "GPU Recovery Failed: %d\n", r);
	} else {
		drm_sched_suspend_timeout(&ring->sched);
		if (amdgpu_sriov_vf(adev))
			adev->virt.tdr_debug = true;
	}

exit:
	amdgpu_vm_put_task_info(ti);
	drm_dev_exit(idx);
	return DRM_GPU_SCHED_STAT_NO_HANG;
}
```

[Source: `drivers/gpu/drm/amd/amdgpu/amdgpu_job.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/amd/amdgpu/amdgpu_job.c#L88)

The function always returns `DRM_GPU_SCHED_STAT_NO_HANG`. This re-inserts the job into the scheduler pending list after the callback; amdgpu has already signalled the guilty fence with an error code during the reset sequence, so the re-inserted job will be retired immediately with that error on the next scheduler pass.

### 4.2 Reset Mode Selection

amdgpu supports several reset methods, selected via the `reset_method` module parameter (default: `-1` for auto):

| Value | Name | Description |
|-------|------|-------------|
| 0 | Legacy | Reads a scratch register to check GPU liveness; in-band register write reset |
| 1 | Mode0 | Soft reset via CP/GFX register writes |
| 2 | Mode1 | Hard reset via PCIE config space `LINK_CTL2`; fastest in-band hard reset |
| 3 | Mode2 | Reset via MMIO SMN (System Management Network) bridge |
| 4 | BACO | Bus Active, Chip Off: PCIe link-down + power-gating cycle |
| 5 | PCI | OS-level PCIe FLR or secondary bus reset |

For everyday soft hangs on discrete GPUs, Mode1 is the default path on most RDNA/RDNA2/RDNA3 ASICs because it is fast and reliably resets the GFX IP without touching the display engine. BACO is used when Mode1 fails or when the GPU link is suspected dead.

### 4.3 BACO Reset — Bus Active, Chip Off

BACO (sometimes written BOCO for Brownout/Coldboot) performs a complete power-domain cycle on the GPU die while keeping the PCIe connection to the host alive:

1. **Pre-BACO**: Suspend all display pipes; flush GPU caches; disable IOMMU translation.
2. **BACO entry**: Assert `BACO_CNTL.BACO_EN`; the GPU's power domain powers off. All VRAM content is **lost**.
3. **PCIe link goes idle** for typically ~200 ms (ASIC-dependent).
4. **BACO exit**: Deassert `BACO_EN`; re-establish PCIe link; re-enumerate BARs.
5. **Re-init**: Full IP block reinitialisation sequence runs (equivalent to a driver `amdgpu_device_init` from the `AMDGPU_IP_BLOCK_TYPE_COMMON` level up).

BACO produces the cleanest possible reset state because every flip-flop in the GPU is reset by power removal. VRAM loss is the key consequence: command buffers, shaders, textures, and ring buffer contents must all be re-uploaded.

The `amdgpu_device_pre_asic_reset()` function (line 5112 in `amdgpu_device.c`) prepares for reset: it toggles the fence driver's ISR to flush in-progress completions, force-completes all ring fences to unblock waiting processes, then delegates to `amdgpu_reset_prepare_hwcontext()` which calls the reset handler's prepare method. If no specialised reset handler is registered (the typical bare-metal path), it checks whether a soft reset (IP-level reset without power cycling) suffices, falling back to a full reset if not.

The top-level `amdgpu_device_gpu_recover()` function (line 5795) coordinates XGMI hive-wide resets, RAS state cross-checks, and the three-phase sequence: halt → ASIC reset → scheduler resume. The high-level flow (simplified commentary, not verbatim):

```c
/* drivers/gpu/drm/amd/amdgpu/amdgpu_device.c — amdgpu_device_gpu_recover() flow */
int amdgpu_device_gpu_recover(struct amdgpu_device *adev,
                              struct amdgpu_job *job,
                              struct amdgpu_reset_context *reset_context)
{
    /* 1. Check if a concurrent RAS recovery already handles this */
    if (amdgpu_ras_is_err_state(adev, AMDGPU_RAS_BLOCK__ANY) &&
        reset_context->src != AMDGPU_RESET_SRC_RAS)
        return 0;  /* defer to RAS handler */

    /* 2. Prepare device list (may include XGMI peer devices) */
    amdgpu_device_recovery_prepare(adev, &device_list, hive);

    /* 3. Acquire the reset domain lock (serialises concurrent resets) */
    amdgpu_device_recovery_get_reset_lock(adev, &device_list);

    /* 4. Halt: drain schedulers, pre-asic-reset per device, unmap BARs */
    amdgpu_device_halt_activities(adev, job, reset_context, &device_list, ...);

    /* 5. Execute the ASIC reset (BACO / Mode1 / Mode2 / PCI FLR) */
    r = amdgpu_device_asic_reset(adev, &device_list, reset_context);

    /* 6. Resume schedulers: re-init IP blocks, restart GPU rings,
     *    signal orphaned job fences with -EIO */
    r = amdgpu_device_sched_resume(&device_list, reset_context, job_signaled);

    /* 7. Post-reset: GPU resume, RAS cleanup, release reset lock */
    amdgpu_device_gpu_resume(adev, &device_list, need_emergency_restart);
    amdgpu_device_recovery_put_reset_lock(adev, &device_list);
    ...
}
```

[Source: `drivers/gpu/drm/amd/amdgpu/amdgpu_device.c`, lines 5112 and 5795](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/amd/amdgpu/amdgpu_device.c)

### 4.4 RAS — ECC and Uncorrectable Errors

The AMD RAS subsystem (`drivers/gpu/drm/amd/amdgpu/amdgpu_ras.c`) handles hardware reliability events:

- **Correctable ECC errors** (single-bit): Counted via `amdgpu_ras_reset_error_count()`. Exposed through `/sys/bus/pci/drivers/amdgpu/.../ras/gfx_ce_count`. No GPU reset required.
- **Uncorrectable ECC errors** (multi-bit): Trigger `amdgpu_ras_interrupt_handler()` → `amdgpu_ras_reset_gpu()` → `amdgpu_device_gpu_recover()` with `AMDGPU_RAS_BLOCK__GFX` source. All in-flight jobs receive `-EIO`.

On CDNA2/CDNA3 (Instinct MI210/MI300) with ECC enabled, an uncorrectable error is treated as a fatal device event: the driver calls `amdgpu_ras_set_context_bad_page_cnt_threshold()` to track bad pages and, if the threshold is exceeded, wedges the device. The `amdgpu_ras_eeprom_*` functions write bad page records to the GPU's EEPROM so the information persists across reboots.

```bash
# Check ECC error counters (RDNA3 / CDNA with RAS enabled)
cat /sys/bus/pci/devices/0000:03:00.0/ras/gfx_ce_count
cat /sys/bus/pci/devices/0000:03:00.0/ras/umc_ue_count

# Trigger a manual GPU recovery for testing (write '1' triggers amdgpu_device_gpu_recover)
echo 1 | sudo tee /sys/kernel/debug/dri/0/amdgpu_gpu_recover
```

---

## 5. Intel i915 Hang Detection and Reset

The i915 driver maintains its own hang detection infrastructure layered on top of the DRM GPU scheduler, with additional per-engine watchdog timers and GuC-mediated reset for modern hardware.

[Source: `drivers/gpu/drm/i915/gt/intel_reset.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/i915/gt/intel_reset.c)

### 5.1 Engine Hang Detection — Heartbeat Requests

For non-GuC submission paths (legacy execlists), i915 uses a heartbeat mechanism: the driver periodically submits a "null" request onto each engine ring. If the heartbeat request does not retire within `sysctl_i915_engine_heartbeat_interval_ms` (configurable, default 2500 ms), the engine is considered hung.

The watchdog path in `intel_gt_requests.c` handles fence expiration:

```c
/* drivers/gpu/drm/i915/gt/intel_gt_requests.c (line 239) */
void intel_gt_watchdog_work(struct work_struct *work)
{
	struct intel_gt *gt =
		container_of(work, typeof(*gt), watchdog.work);
	struct i915_request *rq, *rn;
	struct llist_node *first;

	first = llist_del_all(&gt->watchdog.list);
	if (!first)
		return;

	llist_for_each_entry_safe(rq, rn, first, watchdog.link) {
		if (!i915_request_completed(rq)) {
			struct dma_fence *f = &rq->fence;
			const char __rcu *timeline;
			const char __rcu *driver;

			rcu_read_lock();
			driver = dma_fence_driver_name(f);
			timeline = dma_fence_timeline_name(f);
			pr_notice("Fence expiration time out i915-%s:%s:%llx!\n",
				  rcu_dereference(driver),
				  rcu_dereference(timeline),
				  f->seqno);
			rcu_read_unlock();
			i915_request_cancel(rq, -EINTR);
		}
		i915_request_put(rq);
	}
}
```

[Source: `drivers/gpu/drm/i915/gt/intel_gt_requests.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/i915/gt/intel_gt_requests.c#L239)

### 5.2 `__intel_gt_reset` — The Core Reset Orchestrator

When a reset is warranted (either from the scheduler TDR or from the heartbeat watchdog), the per-GT reset path executes:

```c
/* drivers/gpu/drm/i915/gt/intel_reset.c (line 766) */
static int __intel_gt_reset(struct intel_gt *gt, intel_engine_mask_t engine_mask)
{
	const int retries = engine_mask == ALL_ENGINES ? RESET_MAX_RETRIES : 1;
	reset_func reset;
	int ret = -ETIMEDOUT;
	int retry;

	reset = intel_get_gpu_reset(gt);
	if (!reset)
		return -ENODEV;

	/*
	 * If the power well sleeps during the reset, the reset
	 * request may be dropped and never completes (causing -EIO).
	 */
	intel_uncore_forcewake_get(gt->uncore, FORCEWAKE_ALL);
	for (retry = 0; ret == -ETIMEDOUT && retry < retries; retry++) {
		intel_engine_mask_t reset_mask;

		reset_mask = wa_14015076503_start(gt, engine_mask, !retry);

		GT_TRACE(gt, "engine_mask=%x\n", reset_mask);
		ret = reset(gt, reset_mask, retry);

		wa_14015076503_end(gt, reset_mask);
	}
	intel_uncore_forcewake_put(gt->uncore, FORCEWAKE_ALL);

	return ret;
}
```

[Source: `drivers/gpu/drm/i915/gt/intel_reset.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/i915/gt/intel_reset.c#L766)

`intel_get_gpu_reset()` (line 676) returns an architecture-specific reset function pointer: `i915_do_reset` (writes to the `I915_GDRST` PCI config register), `gen8_reset_engines`, or similar. The `wa_14015076503_*` wrappers handle a workaround for coordinating resets involving the GSC engine (present on Gen12.5 and later).

The callers of `__intel_gt_reset()` sequence additional steps around it:

1. **`reset_prepare()`** (line 875): If GuC submission is active, calls `intel_uc_reset_prepare()` to disable GuC submission first. Then iterates `for_each_engine()`, calling `reset_prepare_engine()` on each to stop the ring and acquire per-engine forcewake.
2. **`gt_reset()`** (line 906): Calls `__intel_engine_reset()` per engine (resets the engine's ring state: advances RING_HEAD, re-initialises context), then calls `intel_uc_reset()` to reset the microcontroller.
3. **Guilty/innocent marking** (lines 64–147): Requests on the hung engine are marked guilty (`-EIO`); requests on other engines are marked innocent (`-EAGAIN`, allowing retry). A per-context hang counter records repeated hangs; contexts exceeding a threshold are **banned** via `intel_context_ban()`.
4. **`reset_finish()`** (line 939): Re-enables submission per engine; calls `intel_engine_signal_breadcrumbs()` to wake up waiters; releases forcewake; calls `intel_uc_reset_finish()`.

### 5.3 GuC-Mediated Engine Reset

On Alder Lake and newer platforms with GuC firmware submission enabled (`intel_guc_submission_enable()`), engine resets are **delegated to the GuC firmware**. The kernel driver sends a Host-to-GuC (H2G) action:

```
INTEL_GUC_ACTION_REQUEST_ENGINE_RESET
```

The GuC firmware performs the engine reset in firmware context, then sends a GuC-to-Host (G2H) response indicating success or failure. If GuC reports failure, the driver falls back to a full GT reset (writing directly to the `GDRST` register).

This delegation is significant: it means the kernel driver **cannot reset an individual engine independently** when GuC is active. A GuC reset failure forces the whole chip to reset.

### 5.4 Error State Capture — `i915_error_state`

When a hang occurs, i915 captures a GPU crash dump into the `i915_error_state` debugfs entry:

```bash
# Read the GPU error state after a hang
cat /sys/kernel/debug/dri/0/i915_error_state > gpu_error.bin

# Parse with Intel's aubinator (part of mesa-utils or standalone)
intel_gpu_abrt gpu_error.bin
```

The captured state includes ring buffer contents (up to 4 KiB before and after the head/tail pointers at the time of hang), active batch buffer addresses, per-engine register dumps (RING_HEAD, RING_TAIL, RING_CTL, ACTHD/BB_ADDR), and the active context's GPU VA space layout.

---

## 6. Intel Xe Driver Reset Path

The Xe driver (`drivers/gpu/drm/xe/`) is the next-generation Intel GPU driver, targeting Xe-LP (Tiger Lake and later) and Xe-HPG (Arc/Battlemage) graphics. It uses the DRM GPU scheduler directly and has a cleaner reset architecture than i915.

[Source: `drivers/gpu/drm/xe/xe_gt.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/xe/xe_gt.c)

### 6.1 Execution Queue States

Xe models GPU contexts as **execution queues** (`xe_exec_queue`). An execution queue progresses through states tracked as atomic flags:

- `EXEC_QUEUE_STATE_RESET`: a reset is in progress for this queue.
- `EXEC_QUEUE_STATE_WEDGED`: the queue is permanently dead (device wedged).
- `EXEC_QUEUE_STATE_BANNED`: the queue was responsible for a hang; future submissions are rejected.

When a scheduler TDR fires, `xe_guc_exec_queue_trigger_cleanup()` is called, which wakes any waiters on the user fence and immediately queues the TDR work item with zero delay (equivalent to `drm_sched_fault()`).

### 6.2 `xe_gt_reset_async` — Asynchronous GT Reset

```c
/* drivers/gpu/drm/xe/xe_gt.c */
void xe_gt_reset_async(struct xe_gt *gt)
{
    xe_gt_info(gt, "trying reset from %ps\n", __builtin_return_address(0));

    /* Optional fault injection for testing */
    if (!xe_fault_inject_gt_reset() && xe_uc_reset_prepare(&gt->uc))
        return;

    xe_gt_info(gt, "reset queued\n");

    /* Grab a runtime PM reference so the device stays awake during reset */
    xe_pm_runtime_get_noresume(gt_to_xe(gt));

    /* Queue reset work onto the ordered workqueue (serialises concurrent resets) */
    if (!queue_work(gt->ordered_wq, &gt->reset.worker))
        xe_pm_runtime_put(gt_to_xe(gt));
}
```

The `gt->reset.worker` (`gt_reset_worker`) executes the actual reset:

1. **Sanitise**: Calls `xe_gt_sanitize()` — disables GuC submission, stops all engines.
2. **Forcewake**: Acquires all forcewake domains.
3. **Stop**: Calls `xe_uc_stop()` (microcontroller stop), `xe_gt_tlb_invalidation_reset()`.
4. **`do_gt_reset()`**: Writes to the `GDRST` register (GT reset) and polls for completion.
5. **`do_gt_restart()`**: Reloads GuC/HuC firmware via `xe_uc_init_hw()`; restores engine workarounds; re-enables submission.

If `do_gt_reset()` fails (GDRST never clears), the device is declared **wedged** (`xe_device_declare_wedged()`). All future IOCTLs return `-EIO`.

### 6.3 Device Wedging and Recovery

A wedged Xe device generates a `drm` uevent:

```
WEDGED=vendor-specific
```

The default recovery from a wedge is a PCIe bus-reset (rebind): the user or management software must unbind the PCIe function (`echo 0000:03:00.0 > /sys/bus/pci/drivers/xe/unbind`) and rebind it. On platforms with Runtime Survivability Mode (certain Arc Pro and Battlemage SKUs), the wedge event indicates that firmware flashing may be needed.

---

## 7. NVIDIA Open Kernel Module (nouveau and GSP-RM)

### 7.1 Nouveau — Classic Firmware-Free Path

Nouveau (`drivers/gpu/drm/nouveau/`) implements hang detection through the DRM GPU scheduler per engine. When a scheduler timeout fires:

1. `nouveau_sched_job_timedout()` (the `timedout_job` implementation) is called.
2. The driver calls `nouveau_channel_kill()`, which marks the DRM channel as dead and signals all pending fences with `-ENODEV` via `nouveau_fence_context_kill()`.
3. A full GPU reset is then attempted via the nvkm (Nouveau kernel module core) engine reset path.

The limitation with Nouveau is that hardware documentation gaps mean reset sequences for newer architectures (Turing/Ampere and later) are not fully implemented. Recovery on these GPUs is often unreliable.

### 7.2 NVIDIA Open Kernel Module — GSP-RM Mediation

The `nvidia-open` kernel module (open-gpu-kernel-modules) is architecturally different: almost all GPU management runs in the **GSP-RM** (GPU System Processor — Resource Manager), which is an ARM Cortex-M firmware that runs on a small co-processor integrated into every Turing+ NVIDIA GPU.

The host OS kernel driver (`kernel-open/nvidia/`) is a thin shim that communicates with GSP-RM via RPC over a shared memory ring buffer. This has profound implications for hang recovery:

- **Watchdog owned by GSP-RM**: GSP-RM runs its own watchdog timer over all GPU engines. If an engine hangs, GSP-RM detects it and initiates recovery internally, then notifies the host driver via an interrupt.
- **Host-initiated recovery**: The host driver can *request* a GPU reset via an RPC call, but the actual reset execution is performed by GSP-RM.
- **OS kernel has limited control**: If GSP-RM itself crashes or becomes non-responsive, the host driver has very limited options. It can attempt a full PCIe function-level reset (FLR), but this may not recover a dead GSP-RM. In practice, a system reboot is often required.

The `nvidia-smi` tool surfaces GSP-RM crash information:

```bash
nvidia-smi --query-gpu=ecc.errors.uncorrected.volatile.total \
           --format=csv,noheader,nounits

# For per-instance error detail (A100/H100 MIG mode)
nvidia-smi mig --gpu-instance-id 0 --query-gpu-instance=ecc.errors
```

For `nouveau`, the equivalent of driver-level hang diagnostics is:

```bash
# nouveau DRM debug — enable verbose hang reporting
echo 0x01f | sudo tee /sys/module/nouveau/parameters/debug
dmesg | grep -i "nouveau\|channel.*killed\|timeout"
```

---

## 8. Kernel Error Reporting and dmesg Patterns

GPU hangs leave characteristic patterns in `dmesg`. Knowing what to grep for is the first step in any hang investigation.

### 8.1 AMD amdgpu

```
[drm:amdgpu_job_timedout [amdgpu]] *ERROR* ring gfx_0.0.0 timeout, but soft recovered
[drm:amdgpu_job_timedout [amdgpu]] *ERROR* ring gfx_0.0.0 timeout
[drm] GPU recovery succeeded
amdgpu 0000:03:00.0: amdgpu: GPU reset succeeded
[drm] VRAM is lost due to GPU reset!
amdgpu 0000:03:00.0: amdgpu: RAS: uncorrectable error detected
amdgpu 0000:03:00.0: amdgpu: GPU recovery failed
```

The `VRAM is lost` message is printed whenever a BACO or Mode1 reset was performed; it means all allocated BO (buffer object) contents must be treated as undefined.

```bash
# Live hang monitoring for amdgpu
dmesg -wH | grep -iE "GPU HANG|ring.*timeout|gpu reset|VRAM is lost|recovery (failed|succeeded)"

# Manual GPU reset trigger for testing recovery code
echo 1 | sudo tee /sys/kernel/debug/dri/0/amdgpu_gpu_recover
```

### 8.2 Intel i915

```
i915 0000:00:02.0: [drm] GPU HANG: ecode 12:1:85dffffb, in process [pid] ...
i915 0000:00:02.0: [drm] Resetting chip for stopped heartbeat on rcs0
[drm] *ERROR* GT0: GUC: Engine reset failed; GPU HANG
i915 0000:00:02.0: [drm] GPU reset
```

The `ecode` field encodes: `gen:engine_class:additional_info`. Intel provides a decoding table in kernel source at `drivers/gpu/drm/i915/gt/intel_engine_cs.c`.

### 8.3 Panfrost (ARM Mali)

```
panfrost 13000000.gpu: gpu sched timeout, js=0, status=0x58, head=0x..., tail=0x...
panfrost 13000000.gpu: GPU fault detected
```

[Source: `drivers/gpu/drm/panfrost/panfrost_job.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/panfrost/panfrost_job.c)

### 8.4 Generic DRM Scheduler

```
sched_main: 0:0: ring timeout
drm_gpu_scheduler: scheduler timeout, signalling fences
```

### 8.5 Comprehensive Live Filter

```bash
# Catch all GPU hang events across any driver
dmesg -wH | grep -iE \
  "GPU HANG|ring.*timeout|gpu reset|IOMMU.*fault|drm.*sched.*timeout|\
  device.*lost|amdgpu.*recover|wedged|engine reset failed"
```

---

## 9. IOMMU Faults from the GPU

When a GPU DMA engine accesses a GPU virtual address that has no valid mapping in the IOMMU page table, the IOMMU (AMD IOMMU, Intel VT-d) or ARM SMMU generates a page-fault interrupt to the CPU.

### 9.1 AMD IOMMU Fault Path

The AMD IOMMU event log records the fault, and the kernel IOMMU fault handler calls any registered page-fault handlers. In amdgpu, the per-device fault handler:

1. Reads the faulting GPU VA from the IOMMU event log entry.
2. Logs it to `dmesg`:
   ```
   amdgpu 0000:03:00.0: amdgpu: GPU fault detected: PASID 0x8, addr 0x7fff00000000
   ```
3. Looks up the amdgpu VM (virtual memory context) owning the PASID.
4. Signals that VM's fences with an error code.
5. Calls `drm_sched_fault()` on the appropriate ring to trigger immediate TDR.

### 9.2 ARM SMMU Fault on Qualcomm Adreno (msm driver)

On mobile SoCs, the Adreno GPU's per-process address space is managed by the ARM SMMU. When a fault occurs, the ARM SMMU calls the registered fault handler, `msm_gpu_fault_handler()` in `msm_iommu.c`. This function retrieves extended fault information from the Adreno SMMU private interface and dispatches to the GPU-level handler via `iommu->base.handler`:

```c
/* drivers/gpu/drm/msm/msm_iommu.c (line 632) */
static int msm_gpu_fault_handler(struct iommu_domain *domain, struct device *dev,
		unsigned long iova, int flags, void *arg)
{
	struct msm_iommu *iommu = arg;
	struct adreno_smmu_priv *adreno_smmu = dev_get_drvdata(iommu->base.dev);
	struct adreno_smmu_fault_info info, *ptr = NULL;

	if (adreno_smmu->get_fault_info) {
		adreno_smmu->get_fault_info(adreno_smmu->cookie, &info);
		ptr = &info;
	}

	if (iommu->base.handler)
		return iommu->base.handler(iommu->base.arg, iova, flags, ptr);

	pr_warn_ratelimited("*** fault: iova=%16lx, flags=%d\n", iova, flags);
	return 0;
}
```

The `iommu->base.handler` is wired to `adreno_fault_handler()` in `adreno_gpu.c`, which logs the faulting IOVA (the GPU virtual address), decodes the ARM SMMU FSR (Fault Status Register) to identify TRANSLATION vs. PERMISSION vs. EXTERNAL fault type, and triggers a GPU crash state capture. Engine recovery then proceeds through the hangcheck timer and the scheduler TDR path.

[Source: `drivers/gpu/drm/msm/msm_iommu.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/msm/msm_iommu.c#L632), [`drivers/gpu/drm/msm/adreno/adreno_gpu.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/msm/adreno/adreno_gpu.c#L283)

### 9.3 Enabling IOMMU Debug Logging

```bash
# AMD IOMMU event log (where available)
echo 1 | sudo tee /sys/kernel/debug/iommu/amd/*/enable_event_log
dmesg | grep -i "iommu.*fault\|gpu.*fault"

# Kernel build option for verbose IOMMU fault reporting
# CONFIG_IOMMU_DEBUG=y (adds extra fault context to messages)

# ARM SMMU fault verbose (for Qualcomm/ARM platforms)
echo 0xff | sudo tee /sys/module/arm_smmu/parameters/debug_mask
```

---

## 10. Userspace Recovery — `VK_ERROR_DEVICE_LOST` and `VK_EXT_device_fault`

### 10.1 `VK_ERROR_DEVICE_LOST` — The Basic Signal

After a GPU hang and reset, any Vulkan call that touches the affected device returns `VK_ERROR_DEVICE_LOST`. This includes:

- `vkQueueSubmit` / `vkQueueSubmit2`
- `vkWaitForFences`
- `vkQueueWaitIdle`
- Any call that touches a resource whose backing memory was on the reset GPU

`VK_ERROR_DEVICE_LOST` is a **fatal device error**. The Vulkan specification states that no further Vulkan calls on the lost device are valid, except for destruction operations (`vkDestroyDevice`, `vkDestroyFence`, etc.) and the device fault query extension below.

### 10.2 `VK_EXT_device_fault` — Richer Diagnostics

The `VK_EXT_device_fault` extension ([`VK_EXT_device_fault`](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_device_fault.html)) allows drivers to report the specific GPU virtual address and vendor context of the fault, callable *after* `VK_ERROR_DEVICE_LOST` is received:

```c
/* Query fault counts first */
VkDeviceFaultCountsEXT counts = {
    .sType = VK_STRUCTURE_TYPE_DEVICE_FAULT_COUNTS_EXT,
};
vkGetDeviceFaultInfoEXT(device, &counts, NULL);

/* Allocate and fill the fault info */
VkDeviceFaultAddressInfoEXT *addr_infos =
    calloc(counts.addressInfoCount, sizeof(*addr_infos));
VkDeviceFaultVendorInfoEXT  *vendor_infos =
    calloc(counts.vendorInfoCount,  sizeof(*vendor_infos));

VkDeviceFaultInfoEXT fault_info = {
    .sType        = VK_STRUCTURE_TYPE_DEVICE_FAULT_INFO_EXT,
    .pAddressInfos = addr_infos,
    .pVendorInfos  = vendor_infos,
};
VkResult r = vkGetDeviceFaultInfoEXT(device, &counts, &fault_info);

/* fault_info.description: human-readable string from driver */
printf("GPU fault: %s\n", fault_info.description);

/* Each address entry reports the GPU VA that caused the fault */
for (uint32_t i = 0; i < counts.addressInfoCount; i++) {
    printf("  faulting VA: 0x%016llx (type: %u, precision: %llu bytes)\n",
           (unsigned long long)addr_infos[i].reportedAddress,
           addr_infos[i].addressType,
           (unsigned long long)addr_infos[i].addressPrecision);
}
```

The key struct definitions:

```c
typedef struct VkDeviceFaultInfoEXT {
    VkStructureType              sType;         /* VK_STRUCTURE_TYPE_DEVICE_FAULT_INFO_EXT */
    void                        *pNext;
    char                         description[VK_MAX_DESCRIPTION_SIZE]; /* human-readable */
    VkDeviceFaultAddressInfoEXT *pAddressInfos; /* array of faulting GPU VA records */
    VkDeviceFaultVendorInfoEXT  *pVendorInfos;  /* driver/vendor-specific fault detail */
    void                        *pVendorBinaryData; /* optional binary blob (e.g. GPU crash dump) */
} VkDeviceFaultInfoEXT;

typedef struct VkDeviceFaultAddressInfoEXT {
    VkDeviceFaultAddressTypeEXT  addressType;      /* READ/WRITE/EXECUTE/INSTRUCTION/etc. */
    VkDeviceAddress              reportedAddress;  /* GPU VA at fault time */
    VkDeviceSize                 addressPrecision; /* power-of-2 precision (bytes) */
} VkDeviceFaultAddressInfoEXT;

typedef struct VkDeviceFaultVendorInfoEXT {
    char     description[VK_MAX_DESCRIPTION_SIZE];
    uint64_t vendorFaultCode;   /* driver-defined error code */
    uint64_t vendorFaultData;   /* driver-defined auxiliary data */
} VkDeviceFaultVendorInfoEXT;
```

[Source: Vulkan Registry `VK_EXT_device_fault`](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_device_fault.html)

### 10.3 Application Recovery Pattern

The standard recovery pattern after `VK_ERROR_DEVICE_LOST`:

```c
void handle_device_lost(VulkanApp *app) {
    /* 1. Query fault info BEFORE destroying the device (while driver state is valid) */
    if (app->has_device_fault_ext) {
        VkDeviceFaultCountsEXT counts = { VK_STRUCTURE_TYPE_DEVICE_FAULT_COUNTS_EXT };
        vkGetDeviceFaultInfoEXT(app->device, &counts, NULL);
        /* ... allocate and query as above, log to crash reporter ... */
    }

    /* 2. Wait for all queues to quiesce (they return errors, not blocking) */
    vkDeviceWaitIdle(app->device);  /* may return VK_ERROR_DEVICE_LOST, ignore */

    /* 3. Destroy all Vulkan objects in reverse creation order */
    destroy_all_pipelines(app);
    destroy_all_descriptors(app);
    destroy_all_buffers_and_images(app);
    destroy_all_synchronization_objects(app);
    vkDestroyCommandPool(app->device, app->cmd_pool, NULL);
    vkDestroyDevice(app->device, NULL);

    /* 4. Optionally destroy and re-create the VkInstance (for clean slate) */
    /* Some validation layers recommend this; not strictly required */

    /* 5. Re-enumerate physical devices — the GPU may have re-initialised */
    app->device = create_device(app->instance, app->surface);
    if (!app->device) {
        fprintf(stderr, "GPU did not recover; exiting\n");
        exit(1);
    }

    /* 6. Re-allocate all GPU resources */
    recreate_swapchain(app);
    create_all_pipelines(app);
}
```

Key considerations:
- **Do not skip the destroy step**: Vulkan drivers track device-child objects. On many drivers, destroying the lost device without destroying its children causes resource leaks or assertion failures in the validation layer.
- **Re-enumerate physical devices**: After a full GPU reset, the physical device is re-probed by the kernel and may appear as a new enumeration. Correlate identity via `VkPhysicalDeviceProperties.deviceUUID` (from `VkPhysicalDeviceIDProperties`) or via `VK_KHR_external_memory_capabilities` if persistent cross-process identity matters.
- **Surface validity**: A `VkSurfaceKHR` is typically window-system state and survives device loss. Do not destroy and recreate the surface unless the window is also being recreated.
- **Steam runtime re-exec**: Steam games running under the Steam Linux Runtime (container/pressure-vessel) may trigger a full runtime re-exec on device lost rather than attempting in-process recovery. This is a valid strategy for games that do not have a device-recovery path.

---

## 11. CI and Testing — Injecting and Verifying Recovery

Testing GPU hang recovery is essential but risky on real hardware. Several kernel and Mesa mechanisms allow hang injection in controlled environments.

### 11.1 IGT `gem_exec_hang` Test Suite

The Intel GPU Tools (IGT) `gem_exec_hang_*` test family injects deliberate GPU hangs and verifies that recovery completes correctly:

```bash
# Run the basic hang-and-recover test on i915
sudo ./build/tests/gem_exec_hang --run-subtest basic-reset

# Run the full hang matrix including per-engine hangs
sudo ./build/tests/gem_exec_hang --run-subtest reset-engine
```

IGT injects hangs by submitting a batch buffer containing an infinite loop (`while(1) nop;` in shader terms, or a spinning busy-wait in the command stream). The kernel scheduler timeout fires, `timedout_job` is called, reset occurs, and IGT verifies that subsequent submissions succeed.

[Source: IGT repo `tests/intel/gem_exec_hang.c`](https://gitlab.freedesktop.org/drm/igt-gpu-tools/-/blob/main/tests/intel/gem_exec_hang.c)

### 11.2 amdgpu Manual Recovery Trigger

```bash
# Force a manual GPU recovery on amdgpu (useful to test the recovery path
# without needing a real hang — writes 1 to trigger amdgpu_device_gpu_recover)
echo 1 | sudo tee /sys/kernel/debug/dri/0/amdgpu_gpu_recover
dmesg | tail -50  # inspect recovery log
```

### 11.3 VKMS and lavapipe — Hang-Immune Environments

VKMS (Virtual Kernel Mode Setting, `drivers/gpu/drm/vkms/`) is a pure software KMS driver. It has no GPU command submission and therefore **cannot hang**. CI pipelines running graphics tests on headless servers can use VKMS as the display backend with zero risk of GPU hang failures.

Similarly, **lavapipe** (Mesa's LLVMpipe-based software Vulkan implementation) runs entirely on the CPU and cannot produce `VK_ERROR_DEVICE_LOST`. For conformance test suites (`dEQP-VK.*`), lavapipe provides a hang-free baseline that separates logic bugs from GPU-specific issues.

```bash
# Run dEQP Vulkan device-lost conformance tests against lavapipe
MESA_VK_DEVICE_SELECT=lavapipe deqp-vk --deqp-case=dEQP-VK.device_lost.*

# Use VKMS as the DRM display backend in CI
modprobe vkms
export WAYLAND_DISPLAY=wayland-0
weston --backend=drm-backend.so --drm-device=/dev/dri/card0
```

### 11.4 Shader Loop Injection for Hang Reproduction

When investigating a hang reported by a user, reproducing it deterministically requires injecting the specific shader that caused the hang. Tools:

- **`VK_LAYER_LUNARG_crash_diagnostic`** (LunarG SDK): Captures a checkpoint trace of GPU progress. On hang, reports the last executed command buffer and draw call index.
- **`VK_AMD_buffer_marker` / `VK_NV_device_diagnostic_checkpoints`**: Driver extensions that write progress markers into a GPU-accessible buffer. On hang, the last written marker identifies the hung draw/dispatch.
- **RenderDoc GPU crash diagnostics**: RenderDoc can capture a frame before a hang, store all command buffer contents, and replay them in isolation to narrow down the crashing command.

```c
/* Using VK_AMD_buffer_marker to narrow down a hang */
void vkCmdWriteBufferMarkerAMD(
    VkCommandBuffer          commandBuffer,
    VkPipelineStageFlagBits  pipelineStage,   /* VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT */
    VkBuffer                 dstBuffer,       /* CPU-visible marker buffer */
    VkDeviceSize             dstOffset,
    uint32_t                 marker);         /* sequential counter value */
```

### 11.5 `dEQP-VK.device_lost.*` Conformance

The `dEQP` (drawElements Quality Programme) Vulkan conformance tests include a `device_lost` test group that verifies:

- Correct `VK_ERROR_DEVICE_LOST` return from submission after device loss.
- Valid `VK_EXT_device_fault` query (returns `VK_SUCCESS` or `VK_INCOMPLETE`, not a crash).
- Ability to destroy and recreate the device (no kernel-level resource leak).

---

## 12. Tools and Monitoring

### 12.1 Live dmesg Monitoring

```bash
# Watch for any GPU hang or reset event in real time
dmesg -wH | grep -iE "GPU HANG|ring.*timeout|gpu reset|IOMMU.*fault|wedged|device lost"
```

### 12.2 `amdgpu_top` — AMD GPU Monitor

`amdgpu_top` (from the `amdgpu_top` crate / `lact` package) shows a GPU hang counter in its status line:

```bash
amdgpu_top --single-gpu
# STATUS: gpu_reset_count: 3
```

### 12.3 `intel_gpu_top` — Intel Engine Reset Counter

```bash
intel_gpu_top
# Columns include: "reset-count" per engine
# A non-zero reset-count indicates that engine has recovered from at least one hang
```

### 12.4 `nvidia-smi` — NVIDIA ECC Error Monitoring

```bash
# Poll ECC correctable and uncorrectable error counts (A100/H100)
watch -n 5 nvidia-smi --query-gpu=ecc.errors.corrected.volatile.total,\
ecc.errors.uncorrected.volatile.total --format=csv

# Detailed ECC error breakdown by memory region
nvidia-smi --query-gpu=ecc.errors.corrected.volatile.l1_cache,\
ecc.errors.corrected.volatile.l2_cache,\
ecc.errors.corrected.volatile.device_memory --format=csv
```

### 12.5 RenderDoc GPU Crash Capture

RenderDoc's crash diagnostics mode captures command buffer contents at submission time into a circular buffer. On hang:

```bash
# Launch application under RenderDoc crash capture
renderdoccmd capture --capture-file hang_frame.rdc ./my_vulkan_app

# On hang, inspect the captured commands
qrenderdoc hang_frame.rdc
```

### 12.6 Per-Driver Debugfs Knobs

```bash
# amdgpu: reset counter and hang statistics
cat /sys/kernel/debug/dri/0/amdgpu_gpu_recover  # shows last reset result

# amdgpu: view ring states (useful pre/post-hang)
cat /sys/kernel/debug/dri/0/amdgpu_ring_gfx_0.0.0

# i915: GPU error state (binary blob from last hang)
cat /sys/kernel/debug/dri/0/i915_error_state > /tmp/i915_error.bin
intel_gpu_abrt /tmp/i915_error.bin

# i915: engine info (head/tail/active context)
cat /sys/kernel/debug/dri/0/i915_engine_info
```

---

## Roadmap

### Near-term (6–12 months)

- **amdgpu pipe-reset support for compute workloads**: AMD posted a 42-patch series enabling pipe-level reset for AMDKFD compute queues, addressing corner cases where single-queue reset fails to recover a hung compute ring. Pipe reset resets all queues on the affected pipe but avoids a full device reset. [Source](https://www.phoronix.com/news/AMDGPU-Pipe-Reset-Support)
- **`DRM_GPU_SCHED_STAT_NO_HANG` — spurious-timeout fast path**: Merged in the v6.12–v6.13 cycle, this return code from `timedout_job` allows drivers to skip the reset sequence when the fence has already signalled by the time the TDR work item fires. Remaining drivers (vc4, lima, etnaviv) are expected to adopt the pattern. [Source](https://lwn.net/Articles/1029951/)
- **Preemption-aware hangcheck for MSM/Adreno A7xx**: Patches track per-ring progress via the `CP_ALWAYS_ON_CONTEXT` counter register; the hangcheck now distinguishes "context has not advanced" from "context has advanced but not finished", eliminating false positives during long preemptive compute dispatches. [Source](https://lkml.iu.edu/2509.3/07182.html)
- **Intel Xe2 hang debugging via GDB**: Intel GDB 2025 added Xe2-specific support for tracking kernel hangs that previously required hardware resets, exposing GPU state through the kernel debugger without invoking TDR. [Source](https://markaicode.com/intel-gdb-2025-xe2-gpu-debugging-kernel-hanging-fixes/)
- **MES hang-detection debugfs interface for amdgpu**: The `hang_hws` debugfs knob used by GPU reset tests currently crashes the kernel with a NULL pointer dereference on MES-based hardware (gfx11+); a CVE fix skips MES for now with a proper MES debugfs interface tracked for completion. [Source](https://osv.dev/vulnerability/CVE-2025-37853)

### Medium-term (1–3 years)

- **Hardware GPU preemption on consumer RDNA and Arc**: True mid-draw preemption (not just inter-draw) would allow the scheduler to interrupt a hung shader dispatch and recover without a full ring reset. AMD's RDNA4 and Intel's Battlemage expose finer-grained preemption points in hardware; driver plumbing to expose these to `drm_gpu_scheduler` is an active design discussion. Note: needs verification of specific timeline commitments.
- **Standardised `VK_EXT_device_fault` breadcrumb protocol**: The extension is ratified and implemented in RADV and ANV, but the kernel-side breadcrumb memory (GPU-written fault addresses flushed to system RAM before the hang) currently uses driver-private ABIs. A proposal to standardise the kernel uAPI for fault breadcrumbs via a new `DRM_IOCTL_SUBMIT_BREADCRUMB` ioctl has been discussed on the dri-devel mailing list. Note: needs verification.
- **Per-context reset isolation (no cross-context blast radius)**: Today, a full GPU reset kills all contexts sharing the device. Per-context reset — isolating the hang to a single VM — requires hardware Memory Protection Unit support (AMD's GFXHUB VM-level fault isolation on RDNA3+). Wider driver support for per-context reset without disturbing peer contexts is a medium-term goal tied to virtual machine and container workload isolation.
- **GSP-RM crash recovery without unbind/rebind (NVIDIA open)**: When the NVIDIA GSP-RM firmware crashes, the current open kernel module requires a full PCIe device unbind and rebind, which is disruptive in multi-tenant environments. NVIDIA's open-source kernel module roadmap (post-v595) includes GSP watchdog recovery that restarts the firmware in place without losing device registration. Note: needs verification of specific release target.
- **ECC RAS integration with kernel EDAC framework**: AMD CDNA and RDNA3 Pro expose RAS events via `amdgpu_ras.c`; integrating these with the kernel's `drivers/edac/` framework would allow standard EDAC tools and IPMI baseboard management controllers to consume GPU ECC errors alongside CPU/DRAM errors. This has been proposed but not yet merged.

### Long-term

- **Mandatory kernel-wide GPU TDR analogous to Windows WDDM**: A long-standing proposal would introduce a `drm_tdr` core layer that mandates a maximum hang dwell time across all DRM drivers, enforced by the core rather than individual driver timeouts. Drivers that do not implement `timedout_job` would receive a default kill-and-wedge behaviour. This would require all in-tree DRM drivers to be updated and is a multi-year effort. Note: needs verification.
- **GPU fault domains and IOMMU-based sandbox reset**: Using PCIe ACS (Access Control Services) and IOMMU fault domains, a future architecture could reset a single PCIe function (SR-IOV virtual function) hosting a hung guest without touching peer VFs or the physical function. This would make GPU virtualisation as resilient as CPU virtualisation for hung guest VMs.
- **Standardised cross-vendor GPU coredump format**: Each driver (`i915_error_state`, `amdgpu_gpu_recover`, MSM crashdump) emits its own binary format. A kernel-standard GPU coredump format (analogous to ELF core dumps) that captures ring contents, register state, and page table snapshots in a vendor-neutral structure would enable cross-vendor tooling and postmortem analysis pipelines. Discussions have occurred at XDC (X.Org Developers Conference) but no formal RFC exists yet.

---

## 13. Integrations

- **Ch01 — DRM Driver Architecture**: The DRM driver lifecycle (`drm_driver_register`, `drm_dev_exit`) is the context in which GPU reset runs. `drm_dev_enter()` / `drm_dev_exit()` are the SRCU-based guards that prevent reset from racing with driver unbind — `amdgpu_job_timedout` calls `drm_dev_enter()` first to bail cleanly if the PCIe device has been removed.

- **Ch04 — GPU Memory Management**: VRAM allocation state interacts deeply with GPU reset. After a BACO or Mode1 reset, all VRAM contents are undefined. amdgpu's `amdgpu_device_evict_resources()` (called from `amdgpu_device_pre_asic_reset`) migrates evictable BOs to system RAM before the reset; non-evictable BOs (those pinned by the display engine) must be re-uploaded afterwards. `drm_buddy` and `drm_mm` allocators survive the reset because they track metadata in system RAM, not VRAM.

- **Ch102 — The DRM GPU Scheduler**: This chapter is the complementary deep-dive into the `drm_gpu_scheduler` scheduling layer that *detects* hangs and dispatches `timedout_job`. Ch102 covers the CFS-style fairness algorithm, entity/job lifecycle, and the `work_tdr` delayed work mechanism that Ch149 depends on. Ch149 focuses on what happens *after* `timedout_job` is called — the driver-specific reset sequences.

- **Ch129 — GPU Firmware**: Understanding GPU firmware (GuC/HuC on Intel, GSP-RM on NVIDIA, MES/RLC on AMD) is essential context for firmware-mediated reset paths described in §5.3, §6, and §7. Firmware crashes are a qualitatively different hang type from hardware hangs; they require different recovery strategies and are harder to debug.

- **Ch159 — Panfrost and the Mali GPU Driver**: Panfrost's `panfrost_job_timedout()` is one of the cleanest reference implementations of the DRM scheduler `timedout_job` callback. The function checks for spurious timeouts (fence already signalled → `DRM_GPU_SCHED_STAT_NO_HANG`), generates a core dump, and calls `panfrost_reset()` for genuine hangs. The pattern transfers directly to any new driver author implementing hang recovery.

- **Ch160 — Freedreno, Turnip, and the Qualcomm Adreno Driver**: The Adreno GPU uses the Qualcomm Graphics Management Unit (GMU) microcontroller as a hardware watchdog. When the GMU detects a GPU hang, it generates an interrupt to the CPU; the MSM driver's `adreno_fault.c` fault handler then calls `drm_sched_fault()` to trigger immediate scheduler TDR. The IOMMU path described in §9.2 (`msm_iommu_fault_handler`) is the other major fault entry point for Adreno.

- **Ch163 — VKMS and Virtual Display Drivers**: VKMS is deliberately immune to GPU hangs — it has no hardware GPU command submission and its "rendering" is done by the CPU. This makes it the ideal CI baseline for separating GPU hang failures from rendering logic failures, as described in §11.3.

- **Ch164 — GPU Power Management: Runtime PM, DVFS, and Power Caps**: Power events can trigger GPU hangs and interact with recovery. An aggressive runtime PM suspend (GPU entering D3cold during a workload) can cause the GPU to stop responding, producing symptoms identical to a hard hang. The amdgpu driver uses `drm_dev_enter()` guards and runtime PM reference counting to prevent `amdgpu_device_gpu_recover()` from racing with `amdgpu_device_suspend()`. BACO reset itself is a power-cycle operation that relies on the same BACO infrastructure used for runtime power gating.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
