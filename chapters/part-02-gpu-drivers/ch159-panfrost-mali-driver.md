# Chapter 159: Panfrost and Valhall: Open-Source ARM Mali Driver

**Target audiences**: Embedded Linux developers targeting ARM Mali GPUs (Raspberry Pi, Rockchip, MediaTek, Amlogic SoCs); driver developers studying Panfrost as an example of reverse-engineered open-source GPU driver; and systems engineers optimising graphics on ARM SBCs (single-board computers).

---

## Table of Contents

1. [Introduction](#introduction)
2. [ARM Mali GPU Architecture](#arm-mali-gpu-architecture)
3. [Panfrost: Midgard and Bifrost Driver](#panfrost-midgard-and-bifrost-driver)
4. [Panfrost Kernel Driver](#panfrost-kernel-driver)
5. [Panfrost Mesa Gallium Driver](#panfrost-mesa-gallium-driver)
6. [Shader Compilation: bifrost-compiler and valhall-compiler](#shader-compilation-bifrost-compiler-and-valhall-compiler)
7. [Panthor: Valhall CSF Driver](#panthor-valhall-csf-driver)
8. [Vulkan on Mali: Panvk](#vulkan-on-mali-panvk)
9. [Performance and Debugging](#performance-and-debugging)
10. [Integrations](#integrations)

---

## Introduction

ARM Mali GPUs are ubiquitous in embedded Linux: Raspberry Pi 5 (VideoCore VII is not Mali, but RK3588, Amlogic S922X, MediaTek MT8195 use Mali-G series), Android phones, and TV SoCs. The open-source Mali driver ecosystem on Linux consists of:

- **Panfrost** — DRM kernel driver + Mesa Gallium driver for Midgard (T600–T880) and Bifrost (G31–G92) Mali GPUs
- **Panthor** — DRM kernel driver for Valhall (G510, G610, G715) and newer CSF (Command Stream Frontend) GPUs
- **Panvk** — Vulkan driver (Mesa) for Bifrost and Valhall

ARM's proprietary Mali driver (`libmali.so`) is also available but not open-source. This chapter focuses on the open-source stack.

Sources: [panfrost kernel docs](https://www.kernel.org/doc/html/latest/gpu/panfrost.html) | [Mesa panfrost](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/drivers/panfrost) | [panthor](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/drivers/panthor)

---

## ARM Mali GPU Architecture

### GPU Families

| Family | Examples | Architecture | Shader ISA |
|---|---|---|---|
| Utgard | Mali-400, Mali-450 | Pre-unified shader | Lima driver |
| Midgard | T604–T880 | Unified shader, vector ISA | Panfrost |
| Bifrost | G31–G92 | Quad-based, scalar ISA | Panfrost |
| Valhall | G57, G68, G78, G510, G610 | Superscalar, CSF | Panthor |
| 5th Gen | G720, G925 | Deferred vertex shading | Panthor (WIP) |

### Tile-Based Deferred Rendering (TBDR)

All Mali GPUs use TBDR — the same approach as Imagination PowerVR and Apple GPU (Ch134):

```
Vertex processing → geometry binned into tiles (32×32 px)
  → Tiler writes per-tile primitive lists
  → Fragment (pixel) shading per tile
  → Tile buffer → CRC write-back to main memory
```

The tile buffer (on-chip) holds one tile's worth of colour/depth. Memory bandwidth savings over immediate-mode: ~5–10× for typical scenes with overdraw.

### Bifrost Shader Core

Bifrost shaders execute in quads of 4 threads (not warps of 32). Each quad shares a sampler unit. The instruction set is a 32-bit scalar ISA:

```
+----------------------------------+
| Arithmetic Units (per 2 threads) |
|   FMA (fused multiply-add)       |
|   CVT (convert)                  |
|   SFU (sin/cos/recip/sqrt)       |
+----------------------------------+
| Texture Unit (per quad)          |
| Load/Store Unit                  |
+----------------------------------+
```

---

## Panfrost: Midgard and Bifrost Driver

### Reverse Engineering History

Panfrost was written by Lyude Paul and Alyssa Rosenzweig (Red Hat) starting in 2018, building on prior work by Connor Abbott and others. The driver is based on reverse engineering — ARM does not provide public documentation for Mali shader ISA or GPU commands.

Key reverse-engineering tools:
- **IGT GPU tools** (render submissions to proprietary driver → replay on open driver)
- **panfrost_dump** (capture command streams from proprietary driver)
- ARM's own documentation for TBDR behavior (partially public)

### Supported Hardware

```bash
# Check if your Mali GPU is supported by Panfrost:
grep -r "panfrost" /sys/bus/platform/drivers/
# Or:
dmesg | grep -i "panfrost\|mali"
# Typical output: "panfrost: loaded for Mali-G52"
```

| SoC | Mali GPU | Driver |
|---|---|---|
| Rockchip RK3399 | Mali-T860 | Panfrost (Midgard) |
| Rockchip RK3568 | Mali-G52 | Panfrost (Bifrost) |
| Rockchip RK3588 | Mali-G610 | Panthor (Valhall) |
| Amlogic S922X | Mali-G52 | Panfrost (Bifrost) |
| MediaTek MT8183 | Mali-G72 | Panfrost (Bifrost) |

---

## Panfrost Kernel Driver

### Driver Registration

```c
/* drivers/gpu/drm/panfrost/panfrost_drv.c */
static const struct of_device_id dt_match[] = {
    { .compatible = "arm,mali-t604",  .data = &panfrost_model_list[0] },
    { .compatible = "arm,mali-t860",  .data = &panfrost_model_list[4] },
    { .compatible = "arm,mali-bifrost", .data = &panfrost_model_list[6] },
    { .compatible = "arm,mali-g52",   .data = &panfrost_model_list[8] },
    /* ... */
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

### Job Scheduler

Panfrost uses the DRM GPU scheduler (`drm_sched`) for job submission:

```c
/* panfrost/panfrost_job.c */
struct panfrost_job {
    struct drm_sched_job base;
    struct panfrost_file_priv *file;
    struct dma_fence *render_done_fence;
    struct panfrost_gem_mapping **mappings;
    /* Job descriptor (GPU command): */
    u64 jc;          /* job chain pointer (GPU VA of first descriptor) */
    uint32_t requirements;
};

/* Job is pushed to hardware via panfrost_job_run(): */
static struct dma_fence *panfrost_job_run(struct drm_sched_job *sched_job)
{
    struct panfrost_job *job = to_panfrost_job(sched_job);
    panfrost_job_write_affinity(pfdev, job->requirements, slot);
    /* Write job chain address to GPU registers: */
    job_write(pfdev, JS_HEAD_LO(slot), lower_32_bits(job->jc));
    job_write(pfdev, JS_HEAD_HI(slot), upper_32_bits(job->jc));
    job_write(pfdev, JS_COMMAND(slot), JS_COMMAND_START);
    return dma_fence_get(job->render_done_fence);
}
```

### Memory Management

Panfrost uses a custom IOMMU-backed allocator:

```c
/* panfrost/panfrost_mmu.c */
int panfrost_mmu_map(struct panfrost_mmu *mmu,
    struct panfrost_gem_mapping *mapping)
{
    /* Map GPU VA → physical pages via IOMMU: */
    iommu_map_sg(mmu->domain, mapping->mmnode.start << PAGE_SHIFT,
                 sgt->sgl, sgt->nents, IOMMU_READ | IOMMU_WRITE);
}
```

Mali GPUs have their own MMU (distinct from the ARM system MMU). Panfrost configures it via GPU registers.

---

## Panfrost Mesa Gallium Driver

### Driver Structure

```
mesa/src/gallium/drivers/panfrost/
├── pan_context.c       # pipe_context: draw calls, state management
├── pan_resource.c      # pipe_resource: textures, buffers
├── pan_screen.c        # pipe_screen: capabilities, format support
├── pan_cmdstream.c     # command stream assembly (job descriptors)
├── pan_blend.c         # blend state → Midgard/Bifrost blend descriptors
├── pan_texture.c       # texture descriptors
└── pan_job.c           # job submission to kernel
```

### Gallium Pipe Context

```c
/* pan_context.c */
static void panfrost_draw_vbo(struct pipe_context *pipe,
    const struct pipe_draw_info *info,
    unsigned drawid_offset,
    const struct pipe_draw_indirect_info *indirect,
    const struct pipe_draw_start_count_bias *draws,
    unsigned num_draws)
{
    struct panfrost_context *ctx = pan_context(pipe);
    struct panfrost_batch *batch = panfrost_get_batch_for_fbo(ctx);

    /* Emit vertex job descriptor: */
    panfrost_emit_vertex_data(batch, ctx);
    panfrost_emit_vertex_job(batch, ctx, info, draws);

    /* Emit tiler job descriptor: */
    panfrost_emit_tiler_job(batch, ctx, info, draws);

    /* If batch is full, submit: */
    if (batch->scoreboard.job_index > MAX_JOBS)
        panfrost_flush_all_batches(ctx, "Batch overflow");
}
```

### Batch System

Panfrost accumulates draw calls into **batches** (analogous to Vulkan render passes). A batch corresponds to one render target (FBO). When the FBO changes or a flush is needed, the batch is submitted:

```c
struct panfrost_batch {
    struct panfrost_context *ctx;
    struct pan_fb_info key;         /* render target config */

    /* Job descriptors: */
    struct panfrost_pool pool;      /* GPU-side memory pool */
    struct panfrost_scoreboard scoreboard;

    /* Resource tracking: */
    struct set *bos;                /* referenced BOs */
    struct set *resources;

    uint64_t jc;    /* job chain: first vertex job VA */
};
```

---

## Shader Compilation: bifrost-compiler and valhall-compiler

### NIR → Bifrost ISA

Panfrost's shader compiler translates NIR to Bifrost ISA via:

```
NIR (from GLSL/SPIR-V)
  → panfrost_lower_nir (Panfrost-specific lowering)
  → bi_from_nir (NIR → BI IR, Bifrost Intermediate Representation)
  → bi_opt_* (constant folding, DCE, etc.)
  → bi_register_allocate (linear scan)
  → bi_pack (emit binary)
```

```c
/* mesa/src/panfrost/bifrost/bifrost_compile.c */
void bifrost_compile_shader_nir(nir_shader *nir,
    const struct panfrost_compile_inputs *inputs,
    struct util_dynarray *binary,
    struct pan_shader_info *info)
{
    /* Bifrost-specific NIR lowering: */
    NIR_PASS(_, nir, bi_lower_uniforms);
    NIR_PASS(_, nir, bi_lower_swizzles);
    NIR_PASS(_, nir, nir_lower_tex, &tex_options);

    /* Convert to Bifrost IR: */
    bi_context *ctx = bi_from_nir(nir, inputs, info);
    bi_optimize(ctx);
    bi_register_allocate(ctx);
    bi_pack(ctx, binary);
}
```

### Instruction Format

Bifrost uses 128-bit instruction bundles (two 64-bit instructions packed):

```
Bits [127:64]: FMA instruction (fused multiply-add)
Bits [ 63: 0]: ADD instruction (add / compare / type convert)
```

Registers are 32-bit; FP16 uses paired 16-bit halves. The ISA was fully reverse-engineered with a disassembler in `src/panfrost/bifrost/disassemble.c`.

---

## Panthor: Valhall CSF Driver

### CSF vs JSM

Valhall GPUs use a **Command Stream Frontend (CSF)** instead of the Job Scheduler Model (JSM) used by Midgard/Bifrost. In CSF:

- CPU writes commands to a **CS buffer** (circular ring buffer)
- A **firmware** (Mali firmware blob) reads the CS and schedules work
- Multiple **CS groups** and **CS queues** replace job slots

```c
/* drivers/gpu/drm/panthor/panthor_sched.c */
struct panthor_group {
    /* Scheduling group: multiple CS queues */
    u32 group_id;
    struct panthor_queue *queues[MAX_CS_PER_GROUP];
};

struct panthor_queue {
    struct panthor_circular_buffer cs_buffer; /* CS ring buffer */
    struct panthor_circular_buffer ringbuf;   /* IRQ ring buffer */
    u32 priority;
};
```

### Panthor Kernel Driver

Panthor was upstreamed in Linux 6.8:

```c
/* drivers/gpu/drm/panthor/panthor_drv.c */
static const struct of_device_id panthor_of_match_table[] = {
    { .compatible = "arm,mali-g610", },
    { .compatible = "arm,mali-g715", },
    { .compatible = "arm,mali-g720", },
    {}
};
```

---

## Vulkan on Mali: Panvk

### Panvk Architecture

`panvk` is the Mesa Vulkan driver for Bifrost and Valhall, implemented in `src/panfrost/vulkan/`:

```bash
# Build panvk:
meson setup build -Dvulkan-drivers=panfrost
# Check:
vulkaninfo --summary | grep panfrost
# Output: "panfrost (Bifrost)"
```

Panvk targets Vulkan 1.1 on Bifrost and Vulkan 1.2 on Valhall. It is production-quality for Bifrost (MediaTek, Rockchip) as of Mesa 24.x.

```c
/* src/panfrost/vulkan/panvk_physical_device.c */
static void panvk_physical_device_init_features(
    struct panvk_physical_device *pdev)
{
    VkPhysicalDeviceVulkan11Features *f11 = &pdev->vk.supported_features.v11;
    f11->storageBuffer16BitAccess = true;
    f11->uniformAndStorageBuffer16BitAccess = true;
    /* ... */

    VkPhysicalDeviceVulkan12Features *f12 = &pdev->vk.supported_features.v12;
    f12->descriptorIndexing = pdev->model->gpu_id >= 0x7000; /* Valhall+ */
}
```

---

## Performance and Debugging

### Enabling Panfrost Debug

```bash
# Panfrost debug environment variables:
PAN_MESA_DEBUG=perf,trace  mesa_app
# Values: perf, trace, dirty, sync, nofp16, bifrost, ...

# Kernel debug:
echo 'module panfrost +p' | sudo tee /sys/kernel/debug/dynamic_debug/control
dmesg | grep panfrost

# GPU frequency scaling:
cat /sys/class/devfreq/*/cur_freq
cat /sys/class/devfreq/*/available_frequencies
```

### Performance Counters

Mali GPUs expose performance counters via the kernel:

```bash
# Mali performance counters via hwmon/HWCPIPE:
# Or via the proprietary gator daemon (ARM Streamline)
cat /sys/devices/platform/mali/gpuinfo  # basic info
```

### Common Performance Issues on SBCs

- **Memory bandwidth**: Mali GPUs on SBCs share LPDDR with CPU; framebuffer updates consume significant bandwidth. Use AFBC (ARM Frame Buffer Compression) when supported.
- **Thermal throttling**: Small SBCs have limited cooling; `cat /sys/class/thermal/thermal_zone*/temp` monitors temperature.
- **DVFS**: GPU frequency scaling. Set `performance` governor for benchmarking: `echo performance > /sys/class/devfreq/*/governor`.

---

## Integrations

- **Ch01 (DRM Architecture)** — Panfrost and Panthor are DRM drivers registered via `platform_driver_register`; they use `drm_sched` for job submission
- **Ch11 (DMA-BUF)** — Panfrost exports GEM BOs as DMA-BUF FDs; V4L2 camera capture and display composition share buffers via DMA-BUF zero-copy
- **Ch14 (Gallium3D)** — Panfrost is a Gallium driver; it implements `pipe_context`, `pipe_screen`, `pipe_resource` exactly as RadeonSI and Iris do
- **Ch15 (NIR)** — Panfrost's shader compiler (bifrost-compiler, valhall-compiler) takes NIR as input — the same IR produced by all Mesa shader frontends
- **Ch134 (Asahi/Apple GPU)** — Asahi and Panfrost both use TBDR architecture and reverse-engineered ISAs; compare their NIR→ISA compiler approaches
- **Ch160 (Turnip/Adreno)** — Turnip is the open-source Qualcomm Adreno driver, built similarly to Panvk; compare ARM vs Qualcomm open-source driver maturity
