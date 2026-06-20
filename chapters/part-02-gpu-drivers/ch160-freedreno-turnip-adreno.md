# Chapter 160: Freedreno and Turnip: Open-Source Qualcomm Adreno Driver

**Target audiences**: Developers building Linux-on-ARM systems with Qualcomm SoCs (Snapdragon X Elite, RB5, Dragonboard); driver developers studying Turnip as the most capable open-source mobile Vulkan driver; and embedded engineers running Wayland compositors on Adreno hardware.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Qualcomm Adreno GPU Architecture](#qualcomm-adreno-gpu-architecture)
3. [Freedreno: The OpenGL Gallium Driver](#freedreno-the-opengl-gallium-driver)
4. [Turnip: The Vulkan Driver](#turnip-the-vulkan-driver)
5. [Compiler: ir3 and the Adreno ISA](#compiler-ir3-and-the-adreno-isa)
6. [Kernel Driver: msm_drm](#kernel-driver-msm_drm)
7. [GPU Memory: IOMMU and SMMU](#gpu-memory-iommu-and-smmu)
8. [Qualcomm Snapdragon X Elite on Linux](#qualcomm-snapdragon-x-elite-on-linux)
9. [Performance and Debugging](#performance-and-debugging)
10. [Integrations](#integrations)

---

## Introduction

Qualcomm Adreno GPUs power the dominant Android platform (Snapdragon SoCs) and, with the Snapdragon X series, are entering the Linux laptop market. The open-source Adreno driver stack consists of:

- **Freedreno** — Mesa Gallium3D driver (OpenGL, OpenGL ES)
- **Turnip** — Mesa Vulkan driver, the highest-performing open-source mobile Vulkan driver
- **ir3** — the shared shader compiler (NIR → Adreno IR → a3xx/a5xx/a6xx ISA)
- **msm_drm** — Linux kernel DRM driver for Qualcomm display and GPU

Turnip is particularly impressive: it supports Vulkan 1.3, raytracing on A7xx (Adreno 700 series), and mesh shaders. It is used in production on postmarketOS, Arch Linux ARM, and Qualcomm's own reference Linux images.

Sources: [freedreno/turnip Mesa](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/freedreno) | [msm_drm kernel](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/msm)

---

## Qualcomm Adreno GPU Architecture

### GPU Families

| Series | SoC | Architecture | Vulkan |
|---|---|---|---|
| A3xx | SD 800/801 | Scalar ISA | No |
| A5xx | SD 820/835 | a5xx ISA | 1.0 (Turnip) |
| A6xx | SD 845–888 | Newer ISA, UCHE | 1.1–1.3 (Turnip) |
| A7xx | SD 8 Gen 3, X Elite | Ray tracing, mesh shaders | 1.3 + RT (Turnip) |

### Unified Compute Architecture

Adreno A6xx+ uses a SIMD architecture (called "warps" internally) of 128 threads per wave on A6xx. Unlike Mali's quad-based Bifrost, Adreno is warp-based like NVIDIA/AMD.

Key A6xx hardware blocks:
- **SP (Shader Processor)**: runs vertex/fragment/compute shaders
- **UCHE (Unified Cache Hierarchy)**: L2 cache shared between SP and TP
- **TP (Texture Processor)**: texture filtering
- **RB (Render Backend)**: render output, depth/stencil
- **CP (Command Processor)**: processes GPU command packets

### Tile-Based Rendering (GMEM)

Adreno uses GMEM (on-chip Graphics Memory) for tile rendering:

```
Render pass start: clear/load tiles from main memory → GMEM
Draw calls: all rendering in GMEM (fast on-chip)
Render pass end: store tiles GMEM → main memory (DRAM)
```

This is Qualcomm's version of TBDR. Turnip is aware of GMEM and emits `gmem_store`/`gmem_load` instructions for render pass load/store operations.

---

## Freedreno: The OpenGL Gallium Driver

### Source Structure

```
mesa/src/gallium/drivers/freedreno/
├── freedreno_context.c  # pipe_context implementation
├── freedreno_resource.c # BO allocation, tiling
├── freedreno_screen.c   # pipe_screen, capabilities
├── a6xx/
│   ├── fd6_draw.c       # draw call emission for A6xx
│   ├── fd6_gmem.c       # GMEM (tile) management
│   ├── fd6_texture.c    # texture descriptors
│   └── fd6_zsa.c        # depth/stencil state
```

### GMEM Binning Pass

Freedreno implements the Adreno binning (tiling) pass:

```c
/* freedreno/a6xx/fd6_gmem.c: emit binning pass */
static void fd6_emit_tile_init(struct fd_batch *batch)
{
    /* Emit VS-only binning pass: */
    fd6_vsc_update_sizes(batch);
    OUT_PKT4(ring, REG_A6XX_VPC_SO_OVERRIDE, 1);
    OUT_RING(ring, A6XX_VPC_SO_OVERRIDE_SO_DISABLE);
    /* Emit binning draw: */
    fd6_draw_emit(batch, ring, DI_PT_POINTLIST_PSIZE,
        IGNORE_VISIBILITY, 0, batch->draw_count);
}
```

The binning pass is a vertex-only render that records which tiles each primitive touches, enabling per-tile fragment processing.

---

## Turnip: The Vulkan Driver

### Architecture Overview

```
mesa/src/freedreno/vulkan/
├── tu_device.c      # VkDevice, VkPhysicalDevice
├── tu_cmd_buffer.c  # VkCommandBuffer recording
├── tu_pipeline.c    # VkPipeline compilation
├── tu_renderpass.c  # VkRenderPass, GMEM load/store
├── tu_image.c       # VkImage, tiling
├── tu_descriptor_set.c  # VkDescriptorSet
└── tu_shader.c      # shader compilation (NIR → ir3 binary)
```

### Vulkan Version Support

Turnip achieves near-complete Vulkan 1.3 on A6xx+ and is the most capable open-source mobile Vulkan driver:

```bash
# Check Turnip Vulkan support:
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/freedreno_icd.aarch64.json vulkaninfo --summary
# Output: Vulkan 1.3 on Adreno (TM) 730 ...
```

```c
/* tu_physical_device.c: capability flags */
VKFEATURE(vk12, bufferDeviceAddress);
VKFEATURE(vk12, descriptorIndexing);
VKFEATURE(vk12, timelineSemaphore);
VKFEATURE(vk13, dynamicRendering);
VKFEATURE(vk13, synchronization2);
VKFEATURE(vk13, maintenance4);
```

### GMEM-Aware Render Pass

Turnip's render pass implementation is tightly coupled to GMEM. A Vulkan render pass with `loadOp=DONT_CARE` and `storeOp=DONT_CARE` avoids all DRAM traffic:

```c
/* tu_renderpass.c: decide GMEM vs SYSMEM */
static enum tu_gmem_layout tu_pick_gmem_layout(
    const struct tu_render_pass *pass,
    const struct tu_framebuffer *fb)
{
    uint32_t gmem_bytes_needed = 0;
    for (uint32_t i = 0; i < pass->attachment_count; i++) {
        gmem_bytes_needed += pass->attachments[i].cpp * fb->width * fb->height / tile_count;
    }
    if (gmem_bytes_needed > phys_dev->gmem_size)
        return TU_GMEM_LAYOUT_BYPASS;  /* too big: fall back to SYSMEM */
    return TU_GMEM_LAYOUT_FULL_COLOR;
}
```

SYSMEM mode (bypass GMEM) is used when attachments are too large for GMEM. Performance degrades significantly — prefer to keep render target sizes within GMEM capacity.

### Command Stream Packets

Adreno's command processor uses type-4 and type-7 packets:

```c
/* freedreno/common/freedreno_pm4.h */
#define CP_TYPE4_PKT  0x40000000
#define CP_TYPE7_PKT  0x70000000

/* Emit a register write (type-4 packet): */
#define OUT_PKT4(ring, reg, cnt) \
    OUT_RING(ring, CP_TYPE4_PKT | ((cnt) << 0) | \
             (((reg) >> 2) << 8))

/* Draw call (type-7 packet): */
#define OUT_PKT7(ring, opcode, cnt) \
    OUT_RING(ring, CP_TYPE7_PKT | ((cnt) << 0) | ((opcode) << 8))
```

---

## Compiler: ir3 and the Adreno ISA

### ir3 Compiler Pipeline

```
GLSL/WGSL/SPIR-V → NIR → ir3 NIR passes → ir3 IR → register allocation → a6xx binary
```

```c
/* freedreno/ir3/ir3_compiler_nir.c */
struct ir3_shader_variant *ir3_shader_create_variant(
    struct ir3_shader *shader,
    const struct ir3_shader_key *key,
    bool keep_ir)
{
    struct ir3_context *ctx = ir3_context_init(shader->compiler, shader, key);
    ir3_nir_lower_variant(ctx->v, shader->nir);
    emit_instructions(ctx);             /* NIR → ir3 IR */
    ir3_optimize_loop(ctx->compiler, ctx->ir);
    ir3_ra(ctx->v);                     /* register allocation */
    assemble(ctx->v);                   /* emit binary */
    return ctx->v;
}
```

### ir3 Instruction Set

ir3 (Adreno IR) is a RISC-like 64-bit instruction set:

```
; Example a6xx fragment shader instructions:
; vec4 = texture(sampler, uv):
(sy)(ss)sam.s2en.mode6.base0 (f32)(xyzw)r0.x, r0.x, r1.x, 0, 0

; mul: r0.x = r1.x * r2.x (float):
mul.f r0.x, r1.x, r2.x

; add: r0.x = r1.x + c0.x (with const):
add.f r0.x, r1.x, c0.x

; kill (discard fragment):
kill r0.x
```

### Half-Precision (fp16)

A6xx natively executes fp16 instructions at 2× throughput. Freedreno promotes fp16 where safe:

```bash
# Enable aggressive fp16:
FD_MESA_DEBUG=perfcntrs MESA_GL_VERSION_OVERRIDE=3.2 app
# Or via NIR pass: nir_lower_alu_to_scalar for fp16
```

---

## Kernel Driver: msm_drm

### MSM DRM Architecture

```
drivers/gpu/drm/msm/
├── msm_drv.c         # DRM driver registration
├── adreno/
│   ├── a6xx_gpu.c    # A6xx GPU ops
│   ├── a6xx_gmu.c    # Graphics Management Unit (power/clock)
│   └── a6xx_ringbuffer.c  # command ring
├── disp/
│   ├── dpu1/         # Display Processing Unit (Snapdragon DPU)
│   └── mdp5/         # older MDP5 display
└── msm_gem.c         # GEM BO management
```

### Ring Buffer Command Submission

MSM uses a ring buffer (not a job queue) for GPU command submission:

```c
/* msm/adreno/a6xx_ringbuffer.c */
void a6xx_rb_submit(struct msm_gpu *gpu, struct msm_gem_submit *submit,
    struct msm_ringbuffer *ring)
{
    /* Write CP_INDIRECT_BUFFER packet pointing to submit's IB: */
    OUT_PKT7(ring, CP_INDIRECT_BUFFER_PFE, 3);
    OUT_RING(ring, lower_32_bits(submit->iova));
    OUT_RING(ring, upper_32_bits(submit->iova));
    OUT_RING(ring, submit->nr_cmds);

    /* Write fence value to scratch register on completion: */
    OUT_PKT7(ring, CP_EVENT_WRITE, 4);
    OUT_RING(ring, CACHE_FLUSH_TS | CP_EVENT_WRITE_0_IRQ);
    OUT_RING(ring, lower_32_bits(rbmemptr(ring, fence)));
    OUT_RING(ring, upper_32_bits(rbmemptr(ring, fence)));
    OUT_RING(ring, submit->seqno);
}
```

### GMU (Graphics Management Unit)

A6xx+ GPUs have a GMU — a separate microcontroller that manages GPU power states and clock scaling independently of the CPU:

```c
/* msm/adreno/a6xx_gmu.c */
static int a6xx_gmu_start(struct a6xx_gmu *gmu)
{
    /* Load GMU firmware: /lib/firmware/qcom/a660_gmu.bin */
    request_firmware_direct(&fw, gmu->fw_name, gmu->dev);
    a6xx_gmu_load_fw(gmu, fw);
    a6xx_gmu_power_on(gmu);
    /* GMU takes over GPU power management: */
    a6xx_gmu_send_nmi(gmu, false);
    return a6xx_gmu_hfi_start(gmu);  /* HFI: Host Firmware Interface */
}
```

---

## GPU Memory: IOMMU and SMMU

### SMMU (System MMU)

Qualcomm SoCs use the ARM SMMU v2/v3 for GPU IOVA (I/O Virtual Address) mapping:

```c
/* msm/msm_iommu.c */
struct msm_iommu {
    struct msm_mmu base;
    struct iommu_domain *domain;
};

static int msm_iommu_map(struct msm_mmu *mmu, uint64_t iova,
    struct sg_table *sgt, size_t len, int prot)
{
    struct msm_iommu *iommu = to_msm_iommu(mmu);
    return iommu_map_sgtable(iommu->domain, iova, sgt, prot);
}
```

All GPU buffers (BOs) get an IOVA through the SMMU. GPU commands reference IOVAs, not physical addresses — enabling full memory isolation between GPU contexts.

### Protected Mode (Secure World)

A6xx supports a protected execution mode for DRM content protection (video decode in the secure world). The GPU's SMMU is configured to restrict access to secure BOs from non-secure contexts.

---

## Qualcomm Snapdragon X Elite on Linux

### Platform Context

The Snapdragon X Elite (QCM6490, X1E80100) is a Windows-on-ARM laptop platform that shipped in 2024. Linux support via the `linux-next` tree includes:

- **msm_drm** — kernel DRM driver for the display pipeline
- **Freedreno/Turnip** — Mesa drivers for the Adreno 830 GPU
- **FastRPC** — DSP offload (for camera/AI)

```bash
# Check Adreno on Snapdragon X:
lspci | grep -i qualcomm   # or lsmod | grep msm
vulkaninfo --summary | grep -i adreno
```

### Current Status (2025)

As of kernel 6.11 and Mesa 24.3:
- Adreno 830 on X Elite: Vulkan 1.3 via Turnip (working)
- Display output via DPU1 (working)
- Hardware video decode (iris) — in progress
- Suspend/resume — in progress

---

## Performance and Debugging

### Debug Environment Variables

```bash
# Freedreno:
FD_MESA_DEBUG=perf,log app

# Turnip:
TU_DEBUG=perf,cs app

# ir3 compiler:
IR3_SHADER_DEBUG=verbose,disasm app 2>&1 | head -100

# MSM kernel debug:
echo 'module msm +p' | sudo tee /sys/kernel/debug/dynamic_debug/control
dmesg | grep msm_gpu
```

### GPU Profiler: adreno-gpu-profiler

Qualcomm's GPU profiler (open-source tool set):

```bash
# GPU performance counters via debugfs:
cat /sys/kernel/debug/dri/0/gpu_busy   # GPU utilisation %
cat /sys/kernel/debug/dri/0/freq       # current frequency

# Or via MSM ring buffer stats:
cat /sys/kernel/debug/dri/0/ringbuffer
```

### GMEM Performance Analysis

```bash
# Count GMEM sysmem fallbacks (performance regression):
TU_DEBUG=perf vulkan_app 2>&1 | grep -i "sysmem\|gmem"
# Ideally all render passes stay in GMEM (faster than SYSMEM)
```

---

## Roadmap

### Near-term (6–12 months)

- **Adreno Gen 8 / Snapdragon X2 Elite support in Mesa 26.0**: Turnip Vulkan driver support for the Adreno Gen 8 GPU (used in the Snapdragon X2 Elite and Snapdragon 8 Elite Gen 5) was merged into Mesa 26.0, paired with kernel enablement in Linux 6.19's MSM driver — enabling accelerated graphics on the latest Qualcomm laptop platform with no proprietary blobs. [Source](https://www.phoronix.com/news/Mesa-26.0-Adreno-Gen-8-Graphics)
- **Variable Rate Shading (VRS) on A8xx**: Initial Gen 8 support ships with VRS disabled while developers resolve stability issues; enabling it is an active near-term goal. [Source](https://news.lavx.hu/article/adreno-gen-8-vulkan-support-merged-into-mesa-26-0-for-snapdragon-x2-elite-linux-graphics)
- **Qualcomm iris video decoder upstream**: The `iris` V4L2 video decoder driver (for H.264/H.265 decode offload on Snapdragon SoCs) completed its v10 patch series in early 2025 and landed for Linux 6.15, replacing the older `venus` driver path on newer hardware and unblocking VA-API hardware video decode for Freedreno/Turnip users. [Source](https://www.phoronix.com/news/Linux-6.15-Media-Subsystem)
- **Suspend/resume on Snapdragon X Elite**: Power management for MSM-based laptops (suspend-to-RAM, runtime PM) is in active upstream patch series; expected to stabilize as hardware becomes more widely deployed. Note: needs verification of exact landing kernel version.
- **Vulkan 1.4 conformance for Turnip**: With Turnip already supporting Vulkan 1.3 on A6xx/A7xx, the driver is tracking the Vulkan 1.4 specification (ratified in early 2024); conformance submission is expected as extension coverage is completed. Note: needs verification of official conformance timeline.

### Medium-term (1–3 years)

- **GMEM tile-based rendering on A7xx**: Optimisation of the GMEM tiling path specifically for the A7xx (Adreno 730/740/830) micro-architecture is ongoing; correct GMEM handling on A7xx was a significant 2024–2025 work item and further bandwidth-reduction tuning is expected. [Source](https://pocket-gaming.org/2026/06/15/the-definitive-guide-to-android-turnip-drivers-hardware-compatibility-2026/)
- **Ray tracing maturation on A7xx/A8xx**: Turnip gained initial ray-tracing support on A7xx; the medium-term goal is full `VK_KHR_ray_tracing_pipeline` and `VK_KHR_ray_query` conformance and enabling RT on A8xx. Note: needs verification of exact extension status per generation.
- **Mesh shaders and task shaders**: `VK_EXT_mesh_shader` enablement on A7xx+ is in progress; full support requires ir3 compiler work for the mesh/task shader stages. Note: needs verification of current implementation state.
- **FastRPC and DSP offload integration**: Qualcomm's FastRPC mechanism (for DSP-accelerated AI/ML and camera pipelines) needs deeper Linux integration to support workloads that interleave GPU and Hexagon DSP compute. Note: needs verification of upstream driver plans.
- **ir3 compiler performance**: Ongoing NIR-level optimisations (instruction scheduling, register allocation tuning for A7xx wave sizes) are expected to close the performance gap with the proprietary Qualcomm driver on SPEC and gaming benchmarks.

### Long-term

- **Unified msm_drm → DRM upstream consolidation**: Longer-term architectural goal to further consolidate display (DPU), GPU, and camera (CSI/ISP) subsystems within the MSM DRM driver tree, aligning with the kernel's ongoing work to improve ARM SoC graphics coherence. Note: needs verification.
- **OpenCL/Compute on Turnip**: Adding a Clover/RustiCL compute path using the ir3 compiler backend to enable OpenCL workloads on Adreno GPUs — relevant for AI inference and GPGPU on Snapdragon Linux laptops. Note: needs verification of feasibility and upstream interest.
- **Vulkan video extensions (VK_KHR_video_decode_*)**: Integration of the Vulkan video decode extensions with the `iris` firmware interface would enable GPU-accelerated decode directly through the Vulkan API rather than V4L2/VA-API, reducing pipeline complexity for media applications. Note: needs verification.
- **Rust-language contributions to ir3/Turnip**: Following the trend set by the `nova` Rust NVIDIA kernel driver (Ch10), there is speculative interest in introducing Rust for safety-critical parts of the Turnip or ir3 codebase; no official RFC exists yet. Note: speculative.

---

## Integrations

- **Ch01 (DRM Architecture)** — msm_drm registers as a DRM driver; it uses `drm_sched` for GPU job scheduling and implements DRM KMS for the Qualcomm DPU display
- **Ch11 (DMA-BUF)** — Freedreno/Turnip export GEM BOs as DMA-BUF FDs; camera capture (V4L2 on Qualcomm ISP) shares zero-copy buffers with the GPU
- **Ch14 (Gallium3D)** — Freedreno is a Gallium3D driver; shares the `pipe_context`/`pipe_screen` interface with RadeonSI, Iris, Panfrost
- **Ch15 (NIR)** — ir3 compiles from NIR; all NIR optimisation passes apply to Adreno as well as AMD/Intel
- **Ch19 (Vulkan)** — Turnip is a Mesa Vulkan driver using the `vk_common` framework (`vk_device`, `vk_command_buffer`) shared with RADV, ANV, Panvk
- **Ch159 (Panfrost/Mali)** — Compare ARM Mali (Panfrost/Panthor) vs Qualcomm Adreno (Freedreno/Turnip) open-source maturity; both use ir3-style ISA compilation into NIR
