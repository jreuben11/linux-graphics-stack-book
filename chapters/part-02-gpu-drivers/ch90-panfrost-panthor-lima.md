# Chapter 90: Open ARM GPU Drivers — Lima, Panfrost, and Panthor

**Target audiences**: Embedded Linux developers working with ARM SoC platforms, GPU driver developers interested in open-source reverse-engineering methodology, mobile Linux device developers (PinePhone, Librem 5, Pinebook Pro), and SoC platform engineers integrating open-source graphics stacks.

---

## Table of Contents

1. [Introduction: ARM's Mali GPU Family and the Open-Driver Landscape](#1-introduction-arms-mali-gpu-family-and-the-open-driver-landscape)
   - [1.1 What is the ARM Mali GPU Family?](#11-what-is-the-arm-mali-gpu-family)
   - [1.2 What is Tile-Based Deferred Rendering (TBDR)?](#12-what-is-tile-based-deferred-rendering-tbdr)
   - [1.3 What is the Command Stream Frontend (CSF)?](#13-what-is-the-command-stream-frontend-csf)
2. [Lima: Mali Utgard Driver](#2-lima-mali-utgard-driver)
3. [Panfrost: Midgard and Bifrost](#3-panfrost-midgard-and-bifrost)
4. [Bifrost Shader Compilation Deep Dive](#4-bifrost-shader-compilation-deep-dive)
5. [Panthor: Valhall and the CSF Model](#5-panthor-valhall-and-the-csf-model)
6. [Mesa Panfrost Vulkan (PanVK)](#6-mesa-panfrost-vulkan-panvk)
7. [Real-World Platforms](#7-real-world-platforms)
8. [Lima/Panfrost in Mesa CI](#8-limapanfrost-in-mesa-ci)
9. [Comparison with Asahi AGX](#9-comparison-with-asahi-agx)
10. [Contributing to Lima/Panfrost/Panthor](#10-contributing-to-limapanfrostpanthor)
11. [Integrations](#11-integrations)

---

## 1. Introduction: ARM's Mali GPU Family and the Open-Driver Landscape

ARM's Mali GPU product line spans roughly two decades and five distinct microarchitectures, each requiring a different driver strategy. Understanding the generational boundaries is essential before choosing which kernel driver or **Mesa** backend to work with.

**Utgard (Mali-200/300/400/450)** — the first-generation Mali design, deployed from approximately 2008 onward. The Utgard family uses separate **Geometry Processor (GP)** and **Pixel Processor (PP)** units, a physical separation that determines how the **Lima** kernel driver models job submission — two independent job types (**GP** jobs and **PP** jobs) that can run concurrently. The **Mali-400** MP4 — four pixel processor cores clocked at 500 MHz — appears in dozens of Allwinner A-series and H-series SoCs. The Lima **Mesa** **Gallium** driver implements two custom IR tiers — **GPIR** (targeting the **GP**'s scalar **VLIW** engine) and **PPIR** (targeting the **PP**'s fragment shader engine) — lowered from **NIR**. The **Mali-470** is a later variant that is structurally similar but not currently supported by Lima.

**Midgard (Mali-T6xx/T7xx/T8xx)** — the second-generation design, shipping from 2012 onward. **Midgard** introduced a unified shader processor model and a **VLIW**-like bundle instruction set with texture, load/store, and math slots packed into 128-bit words. The **Mali-T760** is found in **Rockchip RK3288**; the **Mali-T860/T880** appears in **RK3399** (used in the **PinePhone Pro** and **Pinebook Pro**). **Panfrost** supports the T600 through T880 range and manages eight hardware **MMU** address spaces (**AS0–AS7**), submitting work via *job chains* through hardware **Job Slots (JS)**.

**Bifrost (Mali-G31/G51/G52/G57/G72/G76)** — the third-generation scalar architecture, introduced with the **Mali-G71** in 2016. **Bifrost** reorganised execution into fixed-latency *clauses* — groups of up to eight instruction tuples — which offload scheduling complexity from hardware to the compiler. The **Mali-G52** is widely deployed in mid-range SoCs (**Amlogic G12B**, **Rockchip RK3566/RK3568**). **Panfrost** covers **G31** through **G76**, and **Mesa** 26.x begins extending **Panfrost** into v9 **Valhall** territory. The **Bifrost** compiler backend in `src/panfrost/compiler/` is one of the most sophisticated reverse-engineered **ISA** implementations in the open-source graphics world, encompassing **NIR** lowering, **BI IR** instruction selection, **SSA**-based register allocation, clause scheduling, binary emission via **bi_pack.c**, and the **pan_shader_*** descriptor interface. As of Mesa 26.2, the new **KRAID** Rust-language shader compiler (developed at **Collabora**) targets Mali v9+ architectures as a replacement for the aging C-based **Bifrost** compiler for those generations. The **Mesa Panfrost Gallium** driver (`src/gallium/drivers/panfrost/`) uses a stateless descriptor programming model with dirty-state tracking to minimise CPU overhead per draw call.

**Valhall (Mali-G57/G68/G77/G78/G710/G715)** — the fourth-generation design. The critical architectural change is the introduction of the **Command Stream Frontend (CSF)**: a dedicated microcontroller inside the GPU that accepts a firmware-defined command stream, replacing the hardware job-chain submission model of older generations. Firmware (`mali_csffw.bin`) is mandatory. The **Panthor** kernel driver, mainlined in **Linux 6.10**, targets third-generation Valhall (v10) GPUs including the **Mali-G310**, **G510**, **G610**, and **G710**. **Panthor** introduces a **VM** model based on **drm_gpuvm**, a group/queue submission model via **DRM_IOCTL_PANTHOR_GROUP_SUBMIT**, dynamic tiler heap growth via **panthor_heap_grow()**, device frequency scaling via the **devfreq** framework, and performance profiling through **DRM** client usage stats. The **Mesa Panfrost Vulkan** driver (**PanVK**) in `src/panfrost/vulkan/` targets **Bifrost** v7 through fifth-generation **Valhall** and introduces the **Valhall** software-defined descriptor model (using explicit **LD_ATTR**/**LD_VAR** loads and **FAU** registers for push constants), a userspace **CS Builder API** (`cs_builder.h`) for constructing **CSF** command streams, and per-render-pass tiler heap management via **DRM_IOCTL_PANTHOR_TILER_HEAP_CREATE**.

**Fifth Generation (v12–v14, Immortalis-G715, Mali-G725, etc.)** — the current generation as of 2026. **Panthor** and **Mesa**'s **Panfrost** userspace now support these too, extending the same **CSF** model with additional performance features.

**Architecture quick-reference**:

| Family | Examples | Shader ISA | Open driver |
|---|---|---|---|
| Utgard | Mali-400, Mali-450 | Pre-unified (GP+PP) | Lima |
| Midgard | T604–T880 | Unified vector VLIW | Panfrost |
| Bifrost | G31–G92 | Scalar clause-based | Panfrost |
| Valhall | G57, G68, G510, G610 | Superscalar + CSF | Panthor |
| 5th Gen | G720, G925, Immortalis-G715 | CSF v12–v14 | Panthor (in progress) |

### The Closed-Driver Problem

ARM ships a proprietary GPU driver kit (**Mali DDK**, also called the **Binary Driver**). These blobs are deeply tied to specific Android or vendor kernel versions, carry no source code, and have historically been incompatible with upstream kernels. As of DDK r52 (2024), ARM dropped **Bifrost** and older support from the proprietary stack, leaving embedded Linux users of those GPUs entirely dependent on the open-source drivers. The reverse-engineering methodology used to build these open drivers spans **ioctl** and **mmap** interception (the **Panwrap** tool and its **Pandecode** offline decoder at `src/panfrost/decode/`), ISA disassembly, and — for comparison — macOS IOKit interception (**agxdecode**) used by the **Asahi AGX** team. The **Midgard** and **Bifrost** ISAs were reverse-engineered primarily by **Alyssa Rosenzweig** and contributors at **Collabora**; the **Valhall** ISA was extended from the same work.

The open-source driver ecosystem today covers the full Mali product line from **Utgard** to fifth-generation **Valhall**:

| Driver | GPU Generations | Kernel entry | Mesa entry |
|--------|----------------|-------------|-----------|
| **Lima** | Utgard (Mali-400/450) | `drivers/gpu/drm/lima/` (kernel 5.2+) | `src/gallium/drivers/lima/` (Mesa 19.1+) |
| **Panfrost** | Midgard + Bifrost (+ early Valhall) | `drivers/gpu/drm/panfrost/` (kernel 5.2+) | `src/gallium/drivers/panfrost/`, `src/panfrost/vulkan/` |
| **Panthor** | Valhall v10+ (CSF-based) | `drivers/gpu/drm/panthor/` (kernel 6.10+) | `src/panfrost/` (shared with Panfrost userspace) |

Key deployment targets include: **PinePhone**/**PinePhone Pro** (**Rockchip RK3399**, **Mali-T860**), **Pinebook Pro** (**RK3399**, **Mali-T860 MP4**), **Librem 5** (**NXP i.MX8MQ**, **Vivante GC7000Lite** — though that uses the **etnaviv** driver, not **Panfrost**), **Rock Pi** and **Orange Pi 5** (**RK3588**, **Mali-G610** MC4, served by **Panthor**), and **NXP i.MX8M Plus** evaluation boards. An in-progress Rust reimplementation of the **Panthor** kernel driver, **Tyr**, mirrors the pattern of other Rust **DRM** driver efforts. The drivers are tested in **Mesa**'s CI via **LAVA** (Linaro Automated Validation Architecture), with **dEQP** conformance results tracked by **deqp-runner** against expected-failure lists, shader compilation quality tracked via **shader-db**, and a **drm-shim** at `src/panfrost/drm-shim/` for hardware-free regression testing. Contributions flow through the **dri-devel** mailing list for kernel patches and **GitLab** merge requests for **Mesa** changes.

### 1.1 What is the ARM Mali GPU Family?

The ARM Mali GPU family is a series of tile-based mobile GPU designs licensed by ARM Holdings to SoC vendors for integration into embedded and mobile processors. Mali GPUs are among the most widely deployed graphics processors in the world, appearing in Allwinner, Rockchip, Amlogic, Samsung Exynos, NXP, and MediaTek system-on-chip designs that power smartphones, single-board computers, and embedded Linux platforms.

The Mali product line spans five distinct microarchitectural generations — Utgard, Midgard, Bifrost, Valhall, and the fifth generation (v12–v14) — each introducing different execution models, instruction set architectures, and hardware scheduling mechanisms. This generational fragmentation is central to the open-source driver situation: each generation requires a separate kernel driver and Mesa backend, because the programming interfaces differ fundamentally rather than evolving incrementally.

On Linux, the DRM (Direct Rendering Manager) subsystem hosts three open-source Mali kernel drivers — Lima, Panfrost, and Panthor — while the Mesa project provides the corresponding userspace Gallium and Vulkan drivers. These drivers emerged from reverse-engineering because ARM's proprietary Mali DDK is tied to specific Android kernel versions and is incompatible with upstream Linux. As of DDK r52 (2024), ARM dropped Bifrost and older support from the proprietary stack, leaving embedded Linux users of those GPUs entirely dependent on the open drivers. The open driver ecosystem now covers the full Mali range from Utgard through fifth-generation Valhall, enabling mainstream Linux distributions and compositor stacks on Mali-equipped hardware without proprietary blobs for all but the newest generations.

### 1.2 What is Tile-Based Deferred Rendering (TBDR)?

Tile-Based Deferred Rendering is the rendering architecture used by all Mali GPU generations. Instead of processing a frame as a single stream of primitives that immediately write to the framebuffer — the immediate-mode model used by many desktop GPUs — a TBDR GPU divides the render target into small rectangular tiles, typically 16×16 or 32×32 pixels, and processes each tile independently in two phases.

In the binning phase, the GPU runs vertex shaders and records which triangles overlap each tile into a per-tile polygon list held in a compact tiler heap buffer. In the fragment phase, each tile is processed independently: geometry is re-fetched from the tile list, fragment shaders execute, and the completed tile is written to memory. Because only one tile is live in the GPU's on-chip memory at a time, depth testing, blending, and hidden-surface removal all occur without touching DRAM, yielding significant memory bandwidth savings compared to immediate-mode rendering.

This architecture explains several driver-visible details covered in this chapter. The Mali-400's physical separation into a Geometry Processor and a Pixel Processor directly reflects the two TBDR phases. The job chain model in Panfrost submits vertex/tiler jobs to slot 1 and fragment jobs to slot 0 as distinct stages. The tiler heap — the buffer that holds per-tile polygon lists — appears as an explicit kernel-managed object in Panthor via `panthor_heap_grow()` and in PanVK via `DRM_IOCTL_PANTHOR_TILER_HEAP_CREATE`. Understanding TBDR is prerequisite to understanding why the driver models job submission as two sequenced stages rather than a single dispatch.

### 1.3 What is the Command Stream Frontend (CSF)?

The Command Stream Frontend is a dedicated microcontroller embedded within Valhall-generation and later Mali GPUs that replaces the hardware-managed job-slot submission model used by Midgard and Bifrost. In pre-CSF GPUs, the kernel driver directly writes job descriptor addresses into hardware Job Slot registers, and the hardware fetches and executes the job chain with no firmware involvement. With CSF, the GPU hosts a small firmware processor — loaded from `mali_csffw.bin` at driver initialization — that reads a structured command stream from a shared ring buffer and schedules work across the shader cores on behalf of the driver.

This architectural change has several concrete consequences for the Panthor driver and PanVK. The kernel driver no longer writes directly to job slot registers; instead it enqueues commands into per-queue ring buffers managed via `DRM_IOCTL_PANTHOR_GROUP_SUBMIT`. The userspace driver constructs those command streams using the CS Builder API (`cs_builder.h`). Firmware is mandatory — the GPU cannot process any work without `mali_csffw.bin` — which means Panthor cannot operate on systems where the firmware file is absent, unlike Lima and Panfrost which require no firmware for older generations.

The CSF model also enables finer-grained scheduling: the firmware can context-switch between GPU queues within a job rather than only at job boundaries. This underpins the group/queue abstraction in Panthor, where a group represents a collection of queues that share a set of shader cores and can be scheduled together or preempted atomically. The CSF firmware interface is documented in part through ARM's publicly released architecture reference manuals, supplemented by reverse-engineering incorporated into the open driver.

---

## 2. Lima: Mali Utgard Driver

### Hardware Architecture: GP and PP

The Mali-400 (and its derivatives) expose two types of processing unit to software:

- **GP (Geometry Processor)**: Executes vertex shaders and the tile-list building phase. The GP is a scalar VLIW engine. Its ISA is exposed directly through registers — there is no firmware layer.
- **PP (Pixel Processor)**: Executes fragment shaders and performs tile-based rendering. Mali-400 MP4 contains four PP cores; Mali-450 can have up to eight.

This physical separation between vertex and fragment execution is unique to Utgard and determines how the kernel driver models job submission: two independent job types that can run concurrently.

### Kernel Driver (`drivers/gpu/drm/lima/`)

Lima was the first open-source Mali kernel driver to reach mainline Linux, landing in kernel 5.2. The key source files under `drivers/gpu/drm/lima/` are:

- `lima_device.c` / `lima_device.h` — `struct lima_device`, the top-level device object holding clocks, regulators, power domains, and per-IP references to the GP and PP clusters.
- `lima_gem.c` / `lima_gem.h` — `struct lima_bo`, the GEM buffer object backed by `drm_gem_shmem_object`. Memory is allocated with `drm_gem_shmem_create` and pinned at allocation time. GPU virtual address mapping is performed in kernel space during buffer creation, differing from the more complex lazy-mapping strategy in newer drivers.
- `lima_sched.c` — job scheduling. Lima wraps DRM's `drm_gpu_scheduler` (also called `drm_sched`) with two instances: one for the GP and one for each PP. Each OpenGL context produces a `lima_ctx` object; `drm_sched` pulls from each context's queue in round-robin fashion to provide fair scheduling across processes.
- `lima_pp.c` / `lima_gp.c` — hardware engine management. These files contain the `lima_pp_*` and `lima_gp_*` functions that directly write registers to initiate and monitor jobs. Because Utgard has no command stream frontend firmware, job parameters (job descriptor address, MMU address space, thread count) are written by the driver via `lima_write32()` register accessors.
- `lima_mmu.c` — each processor (GP, each PP core) has its own MMU. On a context switch, the driver writes the new page table base to `LIMA_MMU_DTE_ADDR` and flushes TLBs. The MMU supports 32-bit physical and virtual addresses.

A typical GP job submission proceeds as follows:

```c
/* From lima_sched.c — simplified */
static void lima_sched_run_job(struct drm_sched_job *job)
{
    struct lima_sched_pipe *pipe = to_lima_pipe(job->sched);
    struct lima_job *ljob = to_lima_job(job);

    pipe->task_run(pipe, ljob);   /* calls lima_gp_task_run() or lima_pp_task_run() */
}
```

```c
/* From lima_gp.c — writing the job head pointer */
void lima_gp_task_run(struct lima_sched_pipe *pipe, struct lima_sched_task *task)
{
    struct lima_device *dev = pipe->ldev;
    u32 frame = task->frame;

    lima_write32(dev, LIMA_GP_VSCL_START_ADDR, frame);
    lima_write32(dev, LIMA_GP_COMMAND, LIMA_GP_CMD_START_VS);
}
```

[Source: `drivers/gpu/drm/lima/lima_gp.c` in `torvalds/linux`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/lima/)

### Mesa Lima Gallium Driver (`src/gallium/drivers/lima/`)

The Mesa side implements the Gallium3D pipe driver interface. The primary objects are:

- `lima_screen` — wraps the DRM device fd, queries hardware capabilities via kernel ioctls, and instantiates the shader compiler.
- `lima_context` — per-OpenGL-context state, including current vertex buffer bindings, render target, and the descriptor buffers uploaded for each draw call.

The compiler uses two custom IR tiers:

- **GPIR** (Geometry Processor IR): targets the GP's scalar VLIW engine. Vertex shader NIR is lowered into GPIR and then register-allocated and packed into the GP binary format.
- **PPIR** (Pixel Processor IR): targets the PP's fragment shader engine. Fragment shader NIR is lowered into PPIR. Unlike Bifrost's clause model, Utgard exposes pipeline details directly in its ISA, so the compiler must be aware of which pipeline stages can overlap.

Because the Utgard hardware lacks integer ALUs, integer operations in shaders are lowered to FP16 arithmetic. This causes precision artifacts in applications that depend on integer logic. Similarly, the hardware's FP16 texture rendering path clamps results to the [0.0, 1.0] range, which affects HDR render targets.

**Supported APIs and conformance status**: Lima achieves approximately 97% pass rate on the OpenGL ES 2.0 conformance suite. OpenGL 2.1 is partially supported through Mesa's Gallium GL state tracker. OpenGL ES 3.0 and above are not supported due to hard hardware limitations. [Source: Mesa Lima documentation](https://docs.mesa3d.org/drivers/lima.html)

**Deployment**: Allwinner H5 (NanoPi NEO Plus2, Orange Pi Zero Plus), Allwinner H6 (Orange Pi 3), and Samsung Exynos 4210/4412 all ship Mali-400 MP4 and are supported by Lima. The Raspberry Pi Zero 2W uses a Broadcom VideoCore VI (not Mali), so it falls under the V3D driver (see Ch92).

---

## 3. Panfrost: Midgard and Bifrost

### Kernel Driver (`drivers/gpu/drm/panfrost/`)

Panfrost was developed in parallel with Lima and also reached mainline Linux in kernel 5.2. It handles the substantially more complex Midgard and Bifrost generations.

**Device Tree registration** — Panfrost binds to GPU nodes via `of_device_id` compatible strings:

```c
/* drivers/gpu/drm/panfrost/panfrost_drv.c */
static const struct of_device_id dt_match[] = {
    { .compatible = "arm,mali-t604",    .data = &panfrost_model_list[0] },
    { .compatible = "arm,mali-t860",    .data = &panfrost_model_list[4] },
    { .compatible = "arm,mali-bifrost", .data = &panfrost_model_list[6] },
    { .compatible = "arm,mali-g52",     .data = &panfrost_model_list[8] },
    {}
};

static struct drm_driver panfrost_drm_driver = {
    .driver_features    = DRIVER_RENDER | DRIVER_GEM | DRIVER_SYNCOBJ,
    .open               = panfrost_open,
    .postclose          = panfrost_postclose,
    .ioctls             = panfrost_drm_driver_ioctls,
    .num_ioctls         = ARRAY_SIZE(panfrost_drm_driver_ioctls),
    .fops               = &panfrost_drm_driver_fops,
    .gem_create_object  = panfrost_gem_create_object,
    .prime_handle_to_fd = drm_gem_prime_handle_to_fd,
    .prime_fd_to_handle = drm_gem_prime_fd_to_handle,
};
```

Key objects:

**`struct panfrost_device`** (`panfrost_device.h`):

```c
struct panfrost_device {
    struct device *dev;
    struct drm_device ddev;
    void __iomem *iomem;
    struct clk *clock;
    struct panfrost_features features;
    struct panfrost_mmu *mmu;
    struct panfrost_job_slot *js;
    struct panfrost_perfcnt *perfcnt;
    /* ... */
};
```

**`struct panfrost_gem_object`** (`panfrost_gem.h`): The GEM buffer object. Panfrost uses `drm_gem_shmem_object` as the base and adds a radix tree of `panfrost_mmu_map_entry` structs to track which address spaces (AS) the buffer is mapped into.

**`struct panfrost_job`** (`panfrost_job.h`): Represents one GPU job, carrying a list of referenced BOs, the job descriptor GPU address, and synchronization state.

### Address Space (AS) and MMU Model

Midgard and Bifrost GPUs contain eight hardware MMU address spaces (AS0–AS7). Each GPU context is assigned one AS; the hardware translates GPU virtual addresses through that AS's page table while the job runs. The Panfrost driver manages AS assignment in `panfrost_mmu.c`:

```c
/* panfrost_mmu_as_get() allocates an AS for a context */
int panfrost_mmu_as_get(struct panfrost_device *pfdev, struct panfrost_mmu *mmu)
{
    int as;

    mutex_lock(&pfdev->sched_lock);
    as = panfrost_mmu_as_alloc(pfdev, mmu);
    if (as < 0) {
        /* Evict a context whose job has completed */
        as = panfrost_mmu_as_evict(pfdev, mmu);
    }
    mutex_unlock(&pfdev->sched_lock);
    return as;
}
```

[Source: `drivers/gpu/drm/panfrost/panfrost_mmu.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/panfrost/)

The page table format for Midgard uses a two-level structure (PGD → PTE) with 4 KiB granularity. `panfrost_mmu_map()` walks the GEM object's pages and inserts PTEs; `panfrost_mmu_unmap()` removes them and issues TLB invalidation via `TLBI_ALL`.

### Job Chain Submission

Panfrost submits *job chains* — linked lists of job descriptors in GPU memory — via the Job Slots (JS) hardware. Each job slot can hold one active job and one queued job. The Panfrost driver implements two job slot types: vertex/tiler (slot 1) and fragment (slot 0).

```c
/* panfrost_job_hw_submit() — simplified */
static void panfrost_job_hw_submit(struct panfrost_job *job, int js)
{
    struct panfrost_device *pfdev = job->pfdev;
    u64 cfg = panfrost_job_get_affinity(pfdev, js);

    /* Write the job head pointer */
    job_write(pfdev, JS_HEAD_NEXT_LO(js), lower_32_bits(job->jc));
    job_write(pfdev, JS_HEAD_NEXT_HI(js), upper_32_bits(job->jc));

    /* Configure job slot: thread priority, cache flush policy */
    job_write(pfdev, JS_CONFIG_NEXT(js), cfg | JS_CONFIG_THREAD_PRI(8));

    /* Kick the slot */
    job_write(pfdev, JS_COMMAND_NEXT(js), JS_COMMAND_START);
}
```

[Source: `drivers/gpu/drm/panfrost/panfrost_job.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/panfrost/)

The kernel's `drm_gpu_scheduler` is used for dependency tracking and fair scheduling across contexts. Each `panfrost_job` is a `drm_sched_job`; the scheduler runs the `panfrost_sched_run_job` callback when dependencies (expressed as `dma_fence` objects) have been resolved.

### Midgard ISA

Midgard uses a VLIW-like bundle format where each 128-bit *word* contains up to three instruction slots:

- **Texture slot**: samples from a texture sampler or reads from a special function unit.
- **Load/store slot**: reads or writes varyings, attributes, or uniform buffers.
- **Math slot**: scalar or vector arithmetic (multiply-add, special functions, 16-bit SIMD).

Not every slot need be filled; unused slots are encoded with a no-op. The assembler and disassembler for Midgard (`src/panfrost/midgard/`) expose an arcane assembly syntax reflecting this per-slot encoding. The compiler's task is to fill slots with independent instructions to maximise ILP while honouring pipeline latency constraints.

The Midgard compiler (`src/panfrost/midgard/`) lowers NIR into a Midgard-specific IR, schedules instructions into VLIW bundles, and performs register allocation. Mali's vertex shaders must also handle perspective division and viewport transformation in software (the GPU does not do this in hardware), so the compiler emits epilogue code for vertex shaders.

### Bifrost ISA

Bifrost (Mali-G31 through G76) replaced the VLIW bundle model with a *clause* model:

- Instructions are paired into 64-bit **tuples** combining a multiplier-unit operation and an adder-unit operation.
- Up to eight tuples form a **clause**, a fixed-latency sequence that executes with no pipeline stalls between tuples.
- Clauses are chained together in the instruction stream; inter-clause data dependencies are expressed via register hazards checked at compile time.

This design moves complexity from the GPU hardware scheduler into the compiler. If the compiler emits a clause that violates any of dozens of architectural invariants, the GPU raises an *Invalid Instruction Encoding* exception and aborts execution. Panfrost's Bifrost compiler (`src/panfrost/compiler/`) implements a formal constraint model to verify clause legality before emission.

### Mesa Panfrost Gallium Driver

The Gallium driver lives in `src/gallium/drivers/panfrost/`. Key structures:

- `panfrost_screen` — per-screen (per-device) state, hardware feature queries, format support tables.
- `pan_context` — per-context draw state, dirty-state bitmask, bound resources.

**`panfrost_draw_vbo`** is the Gallium entry point — it emits vertex and tiler job descriptors:

```c
/* src/gallium/drivers/panfrost/pan_context.c */
static void panfrost_draw_vbo(struct pipe_context *pipe,
    const struct pipe_draw_info *info,
    unsigned drawid_offset,
    const struct pipe_draw_indirect_info *indirect,
    const struct pipe_draw_start_count_bias *draws,
    unsigned num_draws)
{
    struct panfrost_context *ctx = pan_context(pipe);
    struct panfrost_batch *batch = panfrost_get_batch_for_fbo(ctx);

    panfrost_emit_vertex_data(batch, ctx);
    panfrost_emit_vertex_job(batch, ctx, info, draws);
    panfrost_emit_tiler_job(batch, ctx, info, draws);

    if (batch->scoreboard.job_index > MAX_JOBS)
        panfrost_flush_all_batches(ctx, "Batch overflow");
}
```

**`panfrost_batch`** accumulates draw calls for a single render target (FBO). When the FBO changes or a flush is triggered, the batch is submitted as a job chain:

```c
struct panfrost_batch {
    struct panfrost_context *ctx;
    struct pan_fb_info key;      /* render target config */
    struct panfrost_pool pool;   /* GPU-side memory pool */
    struct panfrost_scoreboard scoreboard;
    struct set *bos;             /* referenced BOs */
    uint64_t jc;                 /* job chain: first vertex job GPU VA */
};
```

A key performance optimization is **dirty-state tracking**: Mali GPUs use a *stateless programming model* in which each draw call requires a complete set of descriptor bundles describing all graphics state (vertex format, rasterizer, blend, depth/stencil, textures). The driver tracks which state has changed since the last descriptor upload and reuses unchanged descriptors, reducing CPU overhead. This optimization improved synthetic benchmark scores by approximately 400% when first introduced. [Source: Collabora blog, Open Source OpenGL ES 3.1 on Mali GPUs with Panfrost](https://www.collabora.com/news-and-blog/blog/2021/06/11/open-source-opengl-es-3.1-on-mali-gpus-with-panfrost/)

**Supported APIs**: OpenGL ES 3.1 is conformant on Mali-G52, Mali-G57, and Mali-G610. OpenGL 2.1–3.1 is supported depending on architecture. Vulkan 1.0–1.4 is available via PanVK (see §6). Midgard and Bifrost support OpenCL is not available.

---

## 4. Bifrost Shader Compilation Deep Dive

The Bifrost compiler in Mesa (`src/panfrost/compiler/`) represents one of the most technically sophisticated reverse-engineered ISA implementations in the open-source graphics world. This section traces a shader from NIR entry to binary emission.

### NIR Entry and Lowering

NIR enters the Panfrost Bifrost backend via `bifrost_compile()` in `bifrost_compile.c`. Before instruction selection, several NIR lowering passes run:

1. **System value mapping**: `VertexID`, `InstanceID`, and `FragCoord` system values are mapped to preloaded registers — the Bifrost ABI delivers these via a fixed register convention rather than special instructions.
2. **Descriptor lowering** (`panvk_vX_nir_lower_descriptors.c`): Vulkan descriptor set accesses are converted to explicit memory loads. Reserved UBO slots (`RESERVED_UBO_COUNT = 6`) hold push constants and system values.
3. **Format lowering**: OpenGL ES 3.1 atomic operations and image formats are lowered to sequences of Bifrost instructions.

### BI IR and Instruction Selection

The compiler defines a Bifrost intermediate representation (BI IR) where each BI IR instruction maps one-to-one to a single hardware instruction, but without register allocation or clause packing. This keeps optimisation passes simple and separates concerns cleanly.

Instruction selection uses the `bi_builder` struct:

```c
/* Simplified NIR-to-BI lowering (bifrost_nir_lower_algebraic.c) */
static void bi_emit_fadd(bi_builder *b, nir_alu_instr *instr)
{
    bi_index src0 = bi_src_index(&instr->src[0].src);
    bi_index src1 = bi_src_index(&instr->src[1].src);
    bi_index dst  = bi_dest_index(&instr->dest.dest);

    bi_fadd_f32_to(b, dst, src0, src1, BI_ROUND_NONE);
}
```

The `bi_emit_split_i32()` and `bi_emit_collect_to()` helpers handle vector component splitting and recombination, caching intermediate results in an `allocated_vec` map to eliminate redundant moves.

### SSA-Based Register Allocation

Bifrost's register file is 32-bit with 64 general-purpose registers per thread. The compiler's register allocator works in SSA form rather than using classic graph-colouring, instead using a constraint solver that models the register file explicitly. This approach handles the Bifrost-specific requirement that clause inputs and outputs must follow specific port conventions (port 0 and port 1 encoding occupies space in the instruction encoding, so the "ordering of ports is used as an implicit extra bit" for encoding efficiency). Per-component liveness tracking enables effective 16-bit packing: adjacent 16-bit values share a 32-bit register slot, halving register pressure for 16-bit shaders.

### Clause Scheduling

Clause scheduling is the most architecturally distinctive phase. The scheduler must group BI IR instructions into legal clauses satisfying all hardware constraints:

- No read-after-write hazard within the fixed-latency inter-tuple pipeline.
- Texture lookups must appear in a dedicated slot within a clause.
- The clause must specify the correct "message" type (texture, varying load, store, etc.) so the GPU's fixed-function units can be configured.

Panfrost's solution is to formally model clause legality as a predicate:

```
can_schedule(instr, clause_position, current_clause) → bool
```

The scheduler iterates candidate instructions and greedily selects those that pass the predicate, using heuristics to balance immediate fill rate against future scheduling flexibility. Instructions that cannot fit in the current clause initiate a new one. [Source: Collabora Bifrost deep-dive blog post](https://www.collabora.com/news-and-blog/blog/2020/04/23/from-bifrost-to-panfrost-deep-dive-into-the-first-render/)

### Binary Emission

The packing stage (`bi_pack.c`) takes a scheduled, register-allocated BI IR program and assembles the final binary:

- Each tuple is packed into 64 bits according to the Bifrost instruction encoding (fields for opcode, source ports, destination, modifier bits, and the implicit extra port-ordering bit).
- Up to eight tuples are assembled into a 128-byte clause header + body.
- Clauses are chained via a 32-bit header encoding the clause type, dependency on the previous clause's outputs, and end-of-shader flag.
- 16-bit arithmetic uses a distinct encoding path that halves instruction bandwidth by packing two 16-bit operations into one 32-bit slot.

### The `pan_shader_*` Interface

The boundary between the kernel driver and the Mesa userspace compiler is the `pan_shader_desc` structure, which carries:

- Compiled binary blob and size.
- Attribute, varying, UBO, and texture binding maps.
- `threads_per_warp` and `register_count` for hardware thread configuration.
- Blend equation descriptors for fragment shaders.

This interface is versioned with per-architecture XML descriptors in `src/panfrost/genxml/` (e.g., `gen6.xml` for Midgard v6, `gen7.xml` for Bifrost v7), which are parsed at build time to generate C structs for hardware registers and descriptors via the `pan_pack` macro system.

### KRAID: The Rust Shader Compiler (2026)

As of June 2026, Mesa's development tree received **KRAID**, the first Rust-written shader compiler in Mesa's history, targeting ARM Mali v9 (Valhall) and newer architectures. Developed by Faith Ekstrand at Collabora and merged into Mesa 26.2, KRAID translates NIR into Mali v9+ machine code, replacing the aging Bifrost-era C compiler for these newer GPU generations. KRAID targets the same Panfrost and PanVK driver stack, with NIR as the shared intermediate representation between the two compiler backends. [Source: Tech Times, June 2026](https://www.techtimes.com/articles/317763/20260604/arm-mali-open-source-driver-gets-first-rust-shader-compiler-mesa-history.htm)

---

## 5. Panthor: Valhall and the CSF Model

**What is Panthor?** Panthor is the upstream Linux DRM kernel driver for **ARM Mali Valhall and later GPU generations** — specifically those using the **Command Stream Frontend (CSF)** submission model, which was introduced with third-generation Valhall (v10) hardware. It is the successor to Panfrost for newer Mali silicon. Panthor targets Mali-G310, G510, G610, G710, G720, G925 and Immortalis-G715 cores, which are found in mid-range and flagship SoCs (Rockchip RK3588 / Orange Pi 5, MediaTek Dimensity 9000-series, Samsung Exynos 2200). The driver was mainlined in **Linux 6.10** (mid-2024). Unlike Panfrost — which programs GPU job descriptors directly — Panthor communicates with a firmware microcontroller (`mali_csffw.bin`) that mediates all GPU work submission, making the firmware blob a hard runtime requirement. The Mesa userspace counterpart is **PanVK** (`src/panfrost/vulkan/`), which targets Vulkan on Bifrost through fifth-generation Valhall.

### Why a New Driver?

Starting with third-generation Valhall (v10) hardware such as the Mali-G310, G510, G610, and G710, ARM changed the GPU's submission model fundamentally. Earlier generations used a *Job Manager* (JM): the kernel driver wrote job-chain head pointers into hardware registers, and the GPU's internal job manager fetched and executed descriptors directly from GPU-accessible memory. This approach required the kernel to deeply understand GPU descriptor formats.

Third-generation Valhall replaces the Job Manager with the **Command Stream Frontend (CSF)**: a firmware-resident microcontroller that reads a high-level command stream from ring buffers in shared memory. The GPU's hardware still executes the same ISA, but the *interface* between kernel and GPU is now mediated by firmware. This requires the kernel driver to load and communicate with the Mali firmware blob (`mali_csffw.bin`) rather than writing job descriptors directly.

This architectural discontinuity motivated a separate driver rather than extending Panfrost: the kernel ABI, memory management model, and synchronisation primitives are different enough that a clean split made sense. The driver was initially developed under the name "pancsf" before being renamed "Panthor" [Source: `drivers/gpu/drm/panthor/`, LWN article on Panthor RFC](https://lwn.net/Articles/953784/).

### Panthor Kernel Driver (`drivers/gpu/drm/panthor/`)

The driver was merged into `drm-misc` in early 2024 and reached mainline in **Linux 6.10** [Source: Phoronix, Panthor Driver Queued For Linux 6.10](https://www.phoronix.com/news/Panthor-Driver-Linux-6.10). It is organised into several logical blocks:

**Firmware (`panthor_fw_*`)**: The firmware binary is loaded from `/lib/firmware/arm/mali/arch10.8/mali_csffw.bin`. ARM provided this firmware blob into the linux-firmware repository in February 2024 [Source: Phoronix, Arm Lands Mali Gen10 Panthor Firmware](https://www.phoronix.com/news/Arm-Mali-Panthor-Firmware-Git)]. The `panthor_fw_init()` function maps the firmware into a GPU-accessible memory region, sets up the CSF interface page (a shared memory area for kernel–firmware communication), and boots the CSF microcontroller. The firmware communicates status back to the kernel via doorbell interrupts and mailbox registers.

**Buffer Objects (`panthor_bo_*`)**: Panthor's GEM objects use `drm_gem_shmem_object` as the base but add support for the Panthor *virtual machine* (VM) model. The VM model (based on `drm_gpuvm`) associates a set of buffer objects with a GPU virtual address space. Two VM models exist:

- **Exclusive VM**: one BOs-to-one-VM, used for most rendering; simpler, lower-overhead.
- **Shared VM**: supports sparse binding via `VM_BIND` operations for Vulkan sparse resources.

The `DRM_IOCTL_PANTHOR_BO_CREATE` ioctl creates a new BO; `DRM_IOCTL_PANTHOR_VM_BIND` maps or unmaps BOs in the VM's address space.

**Queue Submission**: User space submits work via `DRM_IOCTL_PANTHOR_GROUP_SUBMIT`. The submission model uses *groups* and *queues*:

- A **group** (analogous to a Vulkan queue family) contains one or more *queues*.
- Each queue targets a specific CSF engine (vertex/tiler, fragment, or compute).
- User space populates a command stream buffer — a sequence of CSF instructions encoding draw calls, compute dispatches, and synchronisation operations.
- The kernel validates the group/queue submission, updates the VM's GPU page tables if new BOs are mapped, and signals the CSF firmware via a doorbell to begin processing.

```c
/* Simplified submission path */
int panthor_ioctl_group_submit(struct drm_device *dev,
                               void *data, struct drm_file *file)
{
    struct drm_panthor_group_submit *args = data;
    struct panthor_group *group;
    int ret;

    group = panthor_group_get(file, args->group_handle);
    if (IS_ERR(group))
        return PTR_ERR(group);

    ret = panthor_group_submit(group, args);
    panthor_group_put(group);
    return ret;
}
```

[Source: `drivers/gpu/drm/panthor/panthor_drv.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/panthor/)

**Heap Management (`panthor_heap_*`)**: The tiler unit in Valhall GPUs uses a *tiler heap* — dynamically grown GPU memory — to accumulate intermediate geometry during tiling. Unlike Midgard/Bifrost where the kernel could size the heap at job dispatch time, the CSF model requires the kernel to service heap-exhausted interrupts at runtime. The `panthor_heap_grow()` callback extends the heap in response to CSF firmware requests, allocating new heap chunks and informing the firmware via the CSF interface page.

**MMU/VM**: Panthor uses `drm_gpuvm` (a new DRM subsystem for GPU virtual memory management that was merged alongside Panthor) to manage the GPU's two-level page tables (PGD → PMD → PTE, 4 KiB granularity). The driver depends on `drm_gpuvm` and `drm_sched` with single-entity scheduling support — both of which were also merged in the Linux 6.10 timeframe.

**Device frequency scaling (`panthor_devfreq_*`)**: Panthor integrates with the kernel's `devfreq` framework for dynamic voltage and frequency scaling (DVFS), enabling power-proportional GPU operation on battery-powered devices.

### Key ioctls

```
DRM_IOCTL_PANTHOR_DEV_QUERY       — query device capabilities
DRM_IOCTL_PANTHOR_VM_CREATE       — create a GPU virtual address space
DRM_IOCTL_PANTHOR_VM_DESTROY      — destroy a GPU VM
DRM_IOCTL_PANTHOR_VM_BIND         — map/unmap BOs in a VM (sync or async)
DRM_IOCTL_PANTHOR_VM_GET_STATE    — query VM state (for sparse binding)
DRM_IOCTL_PANTHOR_BO_CREATE       — allocate a GEM buffer object
DRM_IOCTL_PANTHOR_BO_MMAP_OFFSET  — get mmap offset for CPU access
DRM_IOCTL_PANTHOR_GROUP_CREATE    — create a submission group
DRM_IOCTL_PANTHOR_GROUP_SUBMIT    — submit command streams to a group
DRM_IOCTL_PANTHOR_GROUP_DESTROY   — destroy a group
DRM_IOCTL_PANTHOR_TILER_HEAP_CREATE  — allocate tiler heap
DRM_IOCTL_PANTHOR_TILER_HEAP_DESTROY — free tiler heap
```

[Source: `include/uapi/drm/panthor_drm.h`](https://github.com/torvalds/linux/blob/master/include/uapi/drm/panthor_drm.h)

### Reference Platform: Rockchip RK3568

The Rockchip RK3568 SoC (used in, e.g., the Radxa Rock3A and NanoPi R5C) integrates a Mali-G52 MC2 (two shader cores, Bifrost v7 / actually serves as a bridge generation). The Rockchip RK3588 (Orange Pi 5, Rock 5B) features Mali-G610 MC4 — four cores of third-generation Valhall — and is the primary reference platform for Panthor development and conformance testing.

### Performance Profiling

Panthor exposes engine utilisation via the DRM client usage stats interface. The driver reports:

```
drm-engine-panthor: 111110952750 ns
drm-cycles-panthor: 94439687187
drm-maxfreq-panthor: 1000000000 Hz
```

Hardware performance counters (cycle counts, cache hit rates, etc.) require the `panthor_perfcnt_*` subsystem, which provides a `sysfs` interface to enable profiling:

```bash
echo 1 > /sys/bus/platform/drivers/panthor/fc000000.gpu/profiling
```

[Source: Linux kernel documentation for drm/Panthor CSF driver](https://docs.kernel.org/gpu/panthor.html)

---

## 6. Mesa Panfrost Vulkan (PanVK)

### Architecture of PanVK

The Panfrost Vulkan driver (**PanVK**) lives in `src/panfrost/vulkan/` and targets Bifrost v7 (Midgard-era Vulkan is not planned) through fifth-generation Valhall GPUs. The driver uses Mesa's Vulkan common layer (`src/vulkan/`) for objects like pipeline cache, render pass lowering, and descriptor set management (see Ch16 for the full common layer description).

Key top-level structs follow a naming convention parallel to RADV and other Mesa Vulkan drivers:

- `panvk_physical_device` — per-GPU-device state: hardware feature detection via `pan_kmod_dev_create`, core mask, instruction cache sizes.
- `panvk_device` — per-logical-device state: GEM allocator, VM, firmware handle (for Panthor), shader cache.
- `panvk_cmd_buffer` — command buffer recording state.
- `panvk_pipeline` — compiled and linked shader binaries + hardware descriptor layout.

The driver supports multiple hardware generations through `PER_ARCH_FUNCS` dispatch macros:

```c
/* per-arch dispatch — v7 (Bifrost), v9 (Valhall), v10 (3rd-gen Valhall) */
#define PER_ARCH_FUNCS(fn) \
    [6] = panvk_per_arch(fn, v6), \
    [7] = panvk_per_arch(fn, v7), \
    [9] = panvk_per_arch(fn, v9), \
    [10] = panvk_per_arch(fn, v10),
```

Functions for each generation are compiled into separate translation units (`panvk_vX_cmd_dispatch.c`, `panvk_vX_descriptor_set.c`, etc.) to avoid ifdefs cluttering shared code.

### Valhall Descriptor Model

Valhall GPUs replace the hardware descriptor-table model of older generations with an entirely software-defined descriptor model: descriptor sets are laid out in ordinary GPU memory and accessed via ordinary load instructions. There is no hardware descriptor cache or hardware-managed descriptor heap. Instead:

- Descriptors (texture descriptors, sampler descriptors, UBO addresses) are packed into contiguous GPU memory buffers.
- The shader compiler (`bi_emit_*`, or KRAID for v9+) emits explicit `LD_ATTR` / `LD_VAR` / `LOAD.i128` instructions to fetch descriptor data.
- **FAU (Fragment Append Unit)** registers on Valhall provide a fast path for small push constant blocks, avoiding a full descriptor load for the most common uniform data.

The descriptor lowering pass in `panvk_vX_nir_lower_descriptors.c` transforms Vulkan `OpLoad`/`OpAccessChain` SPIR-V patterns into this explicit-load model. Six UBO slots are reserved (`RESERVED_UBO_COUNT = 6`) for push constants and system values, with the remaining slots available for application descriptors.

### Command Stream Frontend in Userspace

For Valhall v10+ (Panthor kernel driver), PanVK constructs a CSF command stream in userspace using the CS Builder API (`cs_builder.h`):

```c
/* Encode a compute dispatch into the CSF command stream */
cs_load32_to(b, cs_reg32(b, EXEC_ID_REG), exec_id_buf, 0);
cs_store32(b, cs_reg32(b, EXEC_ID_REG), ctrl_reg_addr, 0);
cs_run_compute(b, num_groups_x, num_groups_y, num_groups_z);
```

The CS Builder tracks register state and generates compact binary instruction sequences that the Panthor kernel driver dispatches to the CSF firmware. This is the reverse of the Job Manager model: instead of the kernel encoding GPU descriptors, userspace encodes the *control flow* and the kernel validates and submits it.

### Tiler Heap in Vulkan

Vulkan's tile-based deferred rendering model on Mali requires the driver to allocate a tiler heap per render pass. PanVK calls `DRM_IOCTL_PANTHOR_TILER_HEAP_CREATE` before beginning a render pass and passes the heap base address to the GPU via the render pass command buffer. Mid-render heap growth is handled transparently by the kernel driver as described in §5.

### Conformance Status

- **Mali-G52 (Bifrost v7)**: PanVK conformant for Vulkan 1.0. [Source: Mesa Panfrost documentation](https://docs.mesa3d.org/drivers/panfrost.html)
- **Mali-G57 (Valhall v9)**: PanVK conformant for Vulkan 1.0.
- **Mali-G610 (Valhall v10)**: PanVK in active development; OpenGL ES 3.1 conformance achieved as of Mesa 24.1 / Linux 6.10 [Source: Collabora, Taming the Panthor](https://www.collabora.com/news-and-blog/news-and-events/taming-the-panthor-opengl-es-31-conformance-achived-mali-g610.html); Vulkan conformance targeted for subsequent release cycles.

---

## 7. Real-World Platforms

### PinePhone Pro (Rockchip RK3399, Mali-T860 MP4)

The PinePhone Pro is a community-maintained Linux smartphone using the Rockchip RK3399S — a power-optimised RK3399 variant — paired with four Mali-T860 shader cores clocked at 500 MHz. Software stack:

- **Kernel driver**: Panfrost (`CONFIG_DRM_PANFROST`), mainlined since Linux 5.2.
- **Mesa driver**: Panfrost Gallium, OpenGL ES 3.1.
- **Display**: DSI panel attached via Rockchip VOPB, driven by the `rockchip-drm` KMS driver (which relies on the DRM bridge framework connecting to KMS — see Ch2).
- **Compositor**: Phosh (a GNOME Mobile shell) running on Wayland via wlroots.
- **Performance**: The Mali-T860 is a mid-range 2016-era GPU. GLmark2 scores on PinePhone Pro with Panfrost are in the 40–80 range (versus 300–500 on a desktop discrete GPU), adequate for smooth 60 fps UI compositing but not demanding 3D games.

As of GTK 4.18 (GNOME 48, early 2025), the GTK rendering pipeline dropped its legacy GL renderer and now requires OpenGL ES 3.0+, Vulkan, or software rendering. Panfrost on the T860 meets the GLES 3.0 threshold, so Phosh and GTK4 applications continue to receive GPU acceleration. [Source: LINMOB.net, GTK 4.18 and PinePhone](https://linmob.net/gtk-418-the-pinephone-and-megapixels/)

### Librem 5 (NXP i.MX8MQ, Vivante GC7000Lite)

The Purism Librem 5 uses a Vivante GC7000Lite GPU, *not* a Mali GPU. The open-source **etnaviv** driver handles this hardware [Source: etnaviv project, `drivers/gpu/drm/etnaviv/`]. Purism chose the i.MX8M specifically because the etnaviv driver existed and was supported upstream. The GC7000Lite nominally supports OpenGL ES 3.1, but the etnaviv driver currently implements OpenGL 2.1 / OpenGL ES 2.0. Higher API support is a long-term etnaviv development goal. [Source: Librem 5 Wikipedia](https://en.wikipedia.org/wiki/Librem_5)

The etnaviv driver is architecturally similar to Panfrost — a reverse-engineered kernel driver paired with a Gallium3D userspace driver — but is not covered in this chapter; it demonstrates that the broader embedded-Linux open-driver ecosystem applies the same patterns.

### Pinebook Pro (Rockchip RK3399, Mali-T860 MP4)

The Pinebook Pro shares the RK3399 SoC with the PinePhone Pro and therefore uses the same Panfrost driver and Mali-T860 GPU. On a Pinebook Pro running Arch Linux ARM or Manjaro ARM with Wayland (Sway or GNOME), Panfrost provides:

- OpenGL ES 3.1 for compositing and applications.
- Hardware-accelerated `wlroots`-based compositing.
- Approximately 95–120 GLmark2 score in typical configurations (the higher clocked T860 MP4 on the Pinebook Pro slightly outperforms the T860 in the PinePhone Pro due to thermal headroom).

### Orange Pi 5 / Rock 5B (Rockchip RK3588, Mali-G610 MC4)

The RK3588 SoC integrates a four-core Mali-G610 — third-generation Valhall. The full open-source software stack is:

- **Kernel driver**: Panthor (kernel 6.10+, or vendor-patched 6.1 trees for earlier availability).
- **Firmware**: `mali_csffw.bin` from the linux-firmware repository.
- **Mesa driver**: Panfrost (Gallium) + PanVK (Vulkan), both targeting the v10 CSF model.
- **Achieved conformance**: OpenGL ES 3.1 (official Khronos conformance, July 2024, Mesa 24.1.1, kernel 6.10.0-rc1) [Source: CNX Software, Panthor OpenGL ES 3.1 conformance](https://www.cnx-software.com/2024/07/18/panthor-open-source-driver-achieves-opengl-es-3-1-conformance-with-arm-mali-g610-gpu-rk3588-soc/)

The RK3588 is the most capable openly-driven ARM SoC platform available as of 2026 and is increasingly used in hobbyist ML inference, media servers, and desktop replacement applications.

### Tyr: Rust Driver for Panthor (2026)

The **Tyr** project is an in-progress Rust implementation of the Panthor kernel driver. It mirrors the pattern of Nova (the Rust NVIDIA driver, Ch10d) — bringing Rust's memory-safety guarantees to GPU driver code. Tyr is in active demonstration but is not yet merged into the mainline kernel as of mid-2026. [Source: sbcwiki.com Mali GPU Architecture](https://sbcwiki.com/docs/soc-manufacturers/arm/mali-gpu/)]

---

## 8. Lima/Panfrost in Mesa CI

### LAVA: Linaro Automated Validation Architecture

Mesa's CI for ARM hardware drivers uses **LAVA** (Linaro Automated Validation Architecture), an open-source automated board testing system. LAVA can deploy custom bootloaders and kernels, boot target boards remotely, and execute test payloads. This is essential for Panfrost CI because:

1. Changes to the Panfrost kernel driver often require a kernel rebuild and reflash — something LAVA automates.
2. GPU workloads can crash boards or lock up the GPU, requiring automated board reset.
3. The test lab must cover multiple boards (RK3288 for Midgard, RK3399 for Bifrost-era Midgard, RK3568/RK3588 for Bifrost/Valhall).

### CI Jobs and Expected-Failure Files

In Mesa's `.gitlab-ci/` directory, Panfrost CI jobs are configured as YAML pipelines targeting specific boards. Expected-failure lists (`xfails`) are maintained per board:

```
.gitlab-ci/
  lava/
    panfrost-rk3288-fails.txt   # Midgard expected failures
    panfrost-rk3399-fails.txt   # T860 Bifrost expected failures
    panthor-g610-fails.txt      # Valhall expected failures
```

The `deqp-runner` tool runs dEQP (the Khronos conformance test suite) and compares results against the xfails list. A CI job fails only if a *new* test fails (regression) or a previously-passing test disappears (improvement in the xfails list that wasn't reflected). [Source: Mesa CI documentation](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/panfrost/)

### Running dEQP Locally on a PinePhone

To run the OpenGL ES conformance suite locally on a PinePhone Pro or Pinebook Pro:

```bash
# Install Mesa with Panfrost (Arch Linux ARM example)
pacman -S mesa

# Run dEQP-GLES2 using the drm-shim (no display required)
export LIBGL_DRIVERS_PATH=/usr/lib/dri
export LD_PRELOAD=/usr/lib/libpanfrost_noop_drm.so
deqp-gles2 -n dEQP-GLES2.functional.shaders.loops --deqp-log-filename=results.qpa

# Or with a real display (Wayland/KMS):
deqp-gles2 --deqp-gl-config-name=rgba8888d24s8ms0 \
            --deqp-log-filename=results.qpa \
            -n dEQP-GLES2
```

The **drm-shim** (`src/panfrost/drm-shim/`) is a userspace shim that intercepts DRM ioctls and replays them without real hardware, useful for shader-db regression testing and CI on non-Mali build machines.

### Shader-db Support

Shader-db (`https://gitlab.freedesktop.org/mesa/shader-db`) measures shader compilation statistics (instruction counts, register usage, spill rates). Panfrost supports shader-db via the Bifrost disassembler (`src/panfrost/compiler/disassemble.c`), enabling developers to track compiler quality regressions across Mesa commits.

To run shader-db against the Bifrost backend:

```bash
cd shader-db
./run -p panfrost shaders/games/
```

### Debugging Environment Variables

For driver and shader debugging:

```bash
# Mesa userspace (Gallium/Panfrost)
PAN_MESA_DEBUG=trace    # Pretty-print GPU data structures; disassemble shaders to ~/pan_trace_*
PAN_MESA_DEBUG=dump     # Dump raw GPU memory contents
PAN_MESA_DEBUG=nofp16   # Disable 16-bit optimisations (useful for precision debugging)
PAN_MESA_DEBUG=linear   # Force linear image layout (disable AFBC/AFRC compression)

# PanVK Vulkan
PANVK_DEBUG=startup,cmd,sync   # Verbose Vulkan object lifecycle and submission logging

# Kernel driver
echo 0x1 > /sys/kernel/debug/dri/1/panfrost_gpu_info   # Dump GPU register state
```

[Source: Mesa Panfrost documentation](https://docs.mesa3d.org/drivers/panfrost.html)

---

## 9. Comparison with Asahi AGX

The Asahi AGX driver (Apple GPU, see Ch73) and the Panfrost/Lima family share a common heritage as community reverse-engineering efforts but differ significantly in methodology and constraints.

### Reverse Engineering Methodology

**Lima/Panfrost approach — register sniffing and command-stream interception**:

The original technique, pioneered for Mali by the Lima project and later applied by Panfrost, involves writing a small `LD_PRELOAD` shim that intercepts `ioctl()` and `mmap()` calls to the kernel driver. When the proprietary Mali blob submits a job, the shim captures all GPU-mapped memory at submission time and saves it for offline analysis. This technique is similar to what Rob Clark used for `freedreno` (the open Adreno driver). The **Panwrap** tool was the specific interceptor; **Pandecode** (`src/panfrost/decode/`) is the offline decoder that reconstructs GPU descriptors and disassembles shaders from the captured dumps. [Source: Panfrost Pantrace and Pandecode](https://www.phoronix.com/news/Panfrost-Pantrace-Pandecode)

This methodology works well when a Linux-driver proprietary blob is available to intercept. The challenge is that proprietary Mali blobs are tied to specific kernel versions and Android userspace libraries, requiring careful environment construction.

**Asahi AGX approach — on-device JIT interception on macOS**:

Apple's AGX driver lives entirely in macOS kernel space (IOKit). Asahi developed a macOS-side interception library (`agxdecode`) that wraps the IOKit calls the proprietary Metal driver makes, capturing command buffers and GPU memory in-process. Because Apple's shader compiler runs on-device (unlike most desktop GPUs), the team also needed to reverse the ISA by observing output from different Metal shader inputs — a significantly harder problem. [Source: Alyssa Rosenzweig, Dissecting the Apple M1 GPU](https://alyssarosenzweig.ca/blog/asahi-gpu-part-1.html)

### Firmware Dependencies

| Driver | Firmware required? | Notes |
|--------|-------------------|-------|
| Lima (Utgard) | No | Direct register programming |
| Panfrost (Midgard) | No | Direct register programming |
| Panfrost (Bifrost) | No | Direct register programming |
| Panthor (Valhall v10+) | Yes — `mali_csffw.bin` | ARM provides blob in linux-firmware |
| Asahi AGX | Yes — DCP firmware (display), GPU microsequencer | Partially open-sourced by Apple |

### ISA Reverse Engineering

The Midgard ISA was reverse-engineered primarily by Alyssa Rosenzweig, producing the Midgard assembler and disassembler in `src/panfrost/midgard/`. The Bifrost ISA was reverse-engineered collaboratively by Alyssa Rosenzweig, Lyude Paul, and others at Collabora, producing the Bifrost encoder/decoder in `src/panfrost/compiler/`. The Valhall ISA is closely related to Bifrost and was extended from the same work.

The AGX ISA was reverse-engineered by the Asahi team using the `agx` framework, which keeps "canonical meaning and raw hardware encoding side by side" in a structural disassembler that produces control-flow graphs from captured shaders.

### Cross-Pollination

The PanVK Vulkan driver benefits from the Vulkan common layer (`src/vulkan/`) that AGX and Asahi also use. Boris Brezillon (Collabora, Panthor maintainer) collaborated with the Asahi team on `drm_gpuvm` and the explicit-sync DRM APIs that both drivers required. Alyssa Rosenzweig contributed to both Panfrost and Asahi, carrying reverse-engineering intuitions and code patterns between the two projects.

---

## 10. Contributing to Lima/Panfrost/Panthor

### Kernel Driver Contributions

Kernel patches for Lima, Panfrost, and Panthor go to the **dri-devel** mailing list (`dri-devel@lists.freedesktop.org`) and are reviewed by the respective subsystem maintainers:

- Lima: maintained by Qiang Yu and Lima community maintainers.
- Panfrost: Boris Brezillon (Collabora) and Steven Price (ARM).
- Panthor: Boris Brezillon (Collabora) and Liviu Dudau (ARM) as co-maintainers.

Patch format follows the standard Linux kernel convention:

```bash
git format-patch -1 --subject-prefix="PATCH drm/panfrost"
git send-email --to=dri-devel@lists.freedesktop.org \
               --cc=alyssa.rosenzweig@collabora.com \
               --cc=boris.brezillon@collabora.com \
               patch.patch
```

Hardware access is essentially required for kernel driver work: you need a board with the target GPU to test your changes. For CI, the Mesa LAVA lab (operated by Collabora and Linaro) provides remote access to test boards for registered contributors.

### Mesa Contributions

Mesa merge requests are submitted via [GitLab](https://gitlab.freedesktop.org/mesa/mesa). For Panfrost:

- Gallium driver: `src/gallium/drivers/panfrost/`
- Compiler: `src/panfrost/compiler/`
- PanVK: `src/panfrost/vulkan/`
- Shared libraries: `src/panfrost/lib/`

New Mesa contributors should read the Mesa contribution guide (`docs/submitting-patches.rst`) and join `#panfrost` on the OFTC IRC network. The Mesa CI pipeline runs Panfrost tests on real hardware via LAVA automatically on every MR.

### Key Contributors

- **Alyssa Rosenzweig** (Collabora → independent): lead reverse-engineer for Midgard and Bifrost ISAs; designed the Bifrost compiler; initiated the Asahi AGX driver work. Currently maintains the AGX Mesa driver.
- **Boris Brezillon** (Collabora): primary maintainer of the Panthor kernel driver and PanVK; co-designer of `drm_gpuvm` and the Panthor uAPI; coordinated the OpenGL ES 3.1 conformance effort.
- **Tomeu Vizoso** (Collabora, now independent): contributed LAVA CI integration, deqp-runner, and Panfrost performance counter support.
- **Steven Price** (ARM): ARM's upstream representative for Panfrost and Panthor; contributed hardware documentation and bug fixes.
- **Faith Ekstrand** (Collabora): KRAID Rust shader compiler.

### Debugging Tools Summary

```bash
# Kernel-side: check if Panfrost loaded correctly
dmesg | grep -i panfrost
ls /dev/dri/   # Should show renderD128 for the Mali GPU

# Verify OpenGL ES works
glxinfo -B | grep renderer   # Should show "panfrost" (X11)
eglinfo | grep renderer      # Wayland

# Debug Panfrost rendering issues
PAN_MESA_DEBUG=trace glmark2-es2-wayland 2>&1 | head -100

# Check Panthor firmware load
dmesg | grep -i panthor
cat /proc/driver/panthor/fw_version   # Note: path may vary by kernel version

# Profile GPU with Perfetto (Panfrost performance counters)
# Requires enabling in sysfs first (see §5)
record_android_trace -o trace.pb -t 5 -c ... gpu
```

---

## Roadmap

### Near-term (6–12 months)

- **Tyr driver full functional parity**: The Rust-based Tyr DRM driver (submitted for Linux 6.18) is being built out incrementally to implement the complete Panthor uAPI, including full job submission, VM management, and tiler heap support. It aims to become a drop-in replacement for Panthor for use with PanVK. [Source](https://www.phoronix.com/news/Rust-DRM-Drivers-Linux-6.18-Tyr)
- **KRAID Rust compiler maturation**: The KRAID Rust shader compiler, which landed in Mesa 26.2 behind a `-Dpanfrost-rust` Meson flag in June 2026, is being actively developed toward production quality for Mali v9+ (Valhall/5th-gen) targets. It replaces the C-based Bifrost-era compiler for these architectures. [Source](https://www.phoronix.com/news/Mesa-Arm-Mali-KRAID)
- **PanVK Vulkan 1.3/1.4 conformance and Roadmap 2022 profile**: Following Vulkan 1.2 conformance on Mali-G610, the team is working toward Vulkan 1.3 and 1.4 feature coverage and the Khronos Roadmap 2022 profile requirements (including descriptor indexing, synchronization2, dynamic rendering). [Source](https://www.collabora.com/news-and-blog/news-and-events/panvk-reaches-vulkan-12-conformance-on-mali-g610.html)
- **AFBC YUV 4:2:2 video decode support**: Mesa's Panfrost video decode path added AFBC-compressed YUV 4:2:0 textures in Mesa 25.2; AFBC for YUV 4:2:2 is in progress for higher-quality video output on ARM SoC platforms. [Source](https://www.collabora.com/news-and-blog/news-and-events/improvements-to-mesa-video-decoding-for-panfrost.html)
- **Performance counter and coredump support in Panthor**: The Panthor maintainers have called out performance counter support (for GPU profiling tools such as Perfetto and `perf`) and device coredump (for firmware/driver crash debugging) as near-term development items. Note: specific kernel version targets need verification.

### Medium-term (1–3 years)

- **5th-generation Mali (v12–v14) full enablement**: Mesa 25.1 added initial Panfrost/PanVK support for Mali 5th-gen v12/v13 (G720, G725). Full feature parity — including all Vulkan extensions, video decode, and performance counters — for these and forthcoming v14 parts is ongoing. [Source](https://www.phoronix.com/news/Mesa-25.1-Newer-Mali-5th-Gen)
- **Khronos Roadmap 2024 Vulkan profile**: The Collabora PanVK team has stated intent to eventually target the Roadmap 2024 profile, which requires mesh shading, ray query, maintenance5, and other extensions not yet implemented. [Source](https://www.collabora.com/news-and-blog/news-and-events/mesa-25-panvk-moves-towards-production-quality.html)
- **Tyr as Panthor replacement in mainline**: Over several kernel release cycles, Tyr is intended to reach feature parity with the C Panthor driver and potentially replace it as the primary upstream driver for CSF-based Mali GPUs. The timeline depends on Rust DRM bindings maturity. [Source](https://www.collabora.com/news-and-blog/news-and-events/kernel-618-tyr-advances-rust-in-linux.html)
- **Lima Vulkan (speculative)**: Lima currently only supports OpenGL ES via the Gallium driver; there has been no announced plan for a Vulkan backend for Utgard hardware, but community interest exists given the widespread Mali-400/450 installed base. Note: needs verification from mailing list discussion.
- **`drm_gpuvm` enhancements for heterogeneous compute**: `drm_gpuvm` (co-developed for Panthor) is being extended by multiple driver teams to support sparse VM, large BAR mappings, and heterogeneous compute workloads; Panthor and Tyr would inherit these improvements automatically.

### Long-term

- **KRAID coverage expansion to Bifrost**: KRAID currently targets Mali v9+ Valhall architectures. Long-term, a Rust compiler path covering Bifrost (v7) would unify the compiler stack and could retire the C-based Bifrost compiler entirely. Note: no public commitment yet; needs verification.
- **Conformance for OpenGL ES 3.2 on Bifrost/Valhall**: The current conformance achievement is OpenGL ES 3.1 on Mali-G610 (Panthor). Achieving ES 3.2 conformance (adding geometry shaders, ASTC decode modes, advanced blend) is a plausible long-term target as PanVK and the Gallium driver mature. Note: needs verification.
- **Upstream ARM hardware documentation**: Greater official documentation from ARM (beyond the existing public ISA specs and driver stubs) could accelerate feature completeness and reduce maintenance burden for the Panfrost/Panthor teams. This is a recurring community aspiration without a concrete commitment. Note: speculative.

---

## 11. Integrations

This chapter's drivers and compiler infrastructure touch many other parts of the Linux graphics stack:

**Ch1 — DRM Architecture**: Lima, Panfrost, and Panthor all implement the DRM driver interface (`drm_driver`, `drm_gem_object_funcs`, `drm_sched`). They register render nodes at `/dev/dri/renderD128` and expose ioctls following the DRM UAPI conventions described in Ch1. The `drm_gpu_scheduler` used by Lima and Panfrost is the same scheduler used by amdgpu, i915, and nouveau.

**Ch2 — KMS and Display**: ARM SoC display engines (Rockchip VOP/VOPB, Allwinner TCON, NXP LCDIF) are separate KMS drivers paired with Lima/Panfrost/Panthor for rendering. The DRM bridge framework chains display controller → MIPI DSI transmitter → panel, all within the KMS connector/encoder/CRTC model. Lima and Panfrost use the `kmsro` (KMS render-only) framework, which allows a render-only driver (no display outputs) to share a `drm_device` with the display controller driver.

**Ch5 — Mesa Gallium3D**: Lima and Panfrost implement the `pipe_screen`/`pipe_context` Gallium pipe driver interface (Ch5/Ch13), sharing the same NIR shader front-end, state tracker, and disk cache infrastructure as radeonsi, iris, and other Gallium drivers.

**Ch14 — NIR**: All three driver families receive shaders as NIR. Lima's GPIR/PPIR backends, Panfrost's Midgard and Bifrost compilers, and PanVK's descriptor lowering passes all consume NIR. NIR lowering passes (integer lowering, load/store lowering) are shared with other Mesa drivers.

**Ch16 — Vulkan Common Layer**: PanVK is built on Mesa's shared Vulkan infrastructure: descriptor set layout, render pass conversion, pipeline cache, and WSI (Window System Integration) are all shared. PanVK's per-architecture dispatch mirrors the pattern used by RADV, ANV, and NVK.

**Ch21 — wlroots Compositors**: Phosh (PinePhone Pro) and Sway (Pinebook Pro) are wlroots-based Wayland compositors. wlroots' DRM backend allocates GBM buffers backed by Panfrost GEM objects, composites via OpenGL ES through the Panfrost Gallium driver, and presents frames via KMS atomic commits to the Rockchip display engine.

**Ch73 — Asahi/AGX**: The AGX driver and Panfrost share reverse-engineering heritage, key contributors (Alyssa Rosenzweig), and Vulkan common layer infrastructure. Both drivers confront firmware dependency issues for their respective newer GPU generations (Panthor/Mali firmware; AGX/DCP firmware). The `drm_gpuvm` subsystem, developed collaboratively for Panthor, is also used by the AGX driver.

**Ch92 — Raspberry Pi V3D**: V3D (used in the Raspberry Pi 4/5) follows the same embedded-driver pattern as Lima/Panfrost: reverse-engineered, Gallium3D userspace, kernel DRM driver, deployed on a popular hobbyist SBC. V3D and Panfrost both use kmsro for their display driver integration and share the same Mesa NIR pipeline.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
