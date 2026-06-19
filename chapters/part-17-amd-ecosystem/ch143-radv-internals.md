# Chapter 143: RADV Internals: The Mesa AMD Vulkan Driver

This chapter targets **systems and driver developers** who want to understand how the Mesa AMD Vulkan
driver (RADV) is structured, how it maps the Vulkan API onto AMD GCN and RDNA hardware, and how its
compiler pipeline — from SPIR-V through NIR to the ACO backend — produces high-quality GPU machine
code at interactive compile times. **Graphics application developers** targeting AMD hardware on Linux
will also benefit from the sections on pipeline caching, descriptor handling, ray tracing, and the
debugging environment. RADV is the reference implementation for anyone studying an open-source Vulkan
ICD built directly on the Mesa Vulkan common infrastructure without a Gallium3D intermediary.

---

## Table of Contents

1. [History and Overview](#1-history-and-overview)
2. [AMD GPU Hardware Relevant to RADV](#2-amd-gpu-hardware-relevant-to-radv)
3. [Source Tree Structure](#3-source-tree-structure)
4. [Instance, Physical Device, and Logical Device](#4-instance-physical-device-and-logical-device)
5. [Memory Management](#5-memory-management)
6. [Command Buffer Recording](#6-command-buffer-recording)
7. [The Shader Compiler Pipeline](#7-the-shader-compiler-pipeline)
8. [Descriptor Handling](#8-descriptor-handling)
9. [Render Passes and Dynamic Rendering](#9-render-passes-and-dynamic-rendering)
10. [Ray Tracing in RADV](#10-ray-tracing-in-radv)
11. [Performance Features](#11-performance-features)
12. [Pipeline Cache and Shader Object](#12-pipeline-cache-and-shader-object)
13. [Debugging Infrastructure](#13-debugging-infrastructure)
14. [Integrations](#14-integrations)

---

## 1. History and Overview

RADV was written over the summer of 2016 by Bas Nieuwenhuizen (then an independent contributor,
later Google) and Dave Airlie (Red Hat, DRM subsystem maintainer). At the time, AMD had not yet
shipped an open-source Vulkan driver; the only available option was the binary-only `amdgpu-pro`
package. Nieuwenhuizen and Airlie set out to create a community-driven alternative, partially to
provide an open-source Vulkan ICD and partially to pressure AMD into releasing its own open driver
faster. The driver landed in mainline Mesa in October 2016.
[Source](https://www.phoronix.com/review/radv-hits-mesa)

RADV lives entirely inside `src/amd/vulkan/` in the Mesa repository and builds to
`libvulkan_radeon.so`. It is the default AMD Vulkan ICD on all major Linux distributions: Fedora,
Ubuntu, Arch, and the Steam Runtime. When multiple ICDs are present (RADV and the official `amdvlk`
or `amdvlk-pro`), Vulkan's loader selects the driver based on environment variable overrides or ICD
enumeration order.

A critical architectural decision was made early: RADV does **not** use Gallium3D. The RadeonSI
OpenGL driver uses the Gallium state-tracker abstraction, but RADV implements Vulkan directly
against the Mesa Vulkan common infrastructure in `src/vulkan/`. This gives RADV full control over
GPU command encoding, descriptor management, and pipeline compilation without the overhead of the
Gallium intermediary. It also means that Vulkan-specific features — bindless descriptors, dynamic
rendering, ray tracing — can be implemented without retrofitting the Gallium model.

The other foundational event was the ACO compiler backend, contributed principally by Valve engineers
and merged in Mesa 20.1 (2020). ACO replaced LLVM as the default RADV compilation backend,
delivering dramatically faster shader compile times and better code quality for common shader
patterns. The LLVM backend still exists in debug builds (`RADV_DEBUG=llvm`) but is no longer the
default code path.

### Vulkan Conformance by Hardware Generation

RADV supports AMD hardware from GFX6 through GFX12:

| GFX Level | Marketing Name | Example GPUs | Vulkan Version |
|-----------|---------------|--------------|----------------|
| GFX6 | GCN 1 | Tahiti, Pitcairn, Oland | Vulkan 1.3 |
| GFX7 | GCN 2 | Bonaire, Hawaii, Kaveri | Vulkan 1.3 |
| GFX8 | GCN 3–5 | Fiji, Polaris 10/11, Carrizo | Vulkan 1.4 |
| GFX9 | Vega / Raven | Vega10/20, Raven Ridge, Renoir | Vulkan 1.4 |
| GFX10 | RDNA1 | Navi10, Navi14 | Vulkan 1.4 |
| GFX10.3 | RDNA2 | Navi21 (RX 6800 XT), Navi23 | Vulkan 1.4 |
| GFX11 | RDNA3 | Navi31 (RX 7900 XTX), Navi33 | Vulkan 1.4 |
| GFX12 | RDNA4 | Navi48 (RX 9070 XT), Navi44 | Vulkan 1.4 |

GFX6 and GFX7 require the `amdgpu` kernel module, which is not the default for those GPUs; users
must pass `amdgpu.si_support=1 amdgpu.cik_support=1` on the kernel command line and disable the
older `radeon` driver. [Source](https://docs.mesa3d.org/drivers/radv.html)

---

## 2. AMD GPU Hardware Relevant to RADV

Understanding the driver requires familiarity with the hardware abstractions RADV targets. The AMD
GPU architecture evolved significantly from GCN to RDNA; many of RADV's conditional code paths are
gated on the `amd_gfx_level` enum defined in `src/amd/common/amd_family.h`.

### GCN Architecture (GFX6–GFX9)

GCN (Graphics Core Next) organises the GPU into **Shader Engines (SEs)** each containing multiple
**Compute Units (CUs)**. Each CU has:
- Four **SIMD16** units (executing 16 lanes simultaneously)
- One **Scalar Unit (SU)** for control flow and uniform data
- 64 KB of **LDS (Local Data Share)** per CU
- 256 × 64 **VGPRs** (Vector General-Purpose Registers, one per lane)
- 800 × 32-bit **SGPRs** (Scalar GPRs, shared by the wave)

GCN executes shaders in **wavefronts of 64 threads (Wave64)**. Four cycles are needed to dispatch a
single wavefront across all SIMD16 units. The scheduler can interleave up to 10 wavefronts per SIMD
to hide memory latency.

### RDNA Architecture (GFX10–GFX12)

RDNA merged pairs of GCN CUs into **Work Group Processors (WGPs)**. Each WGP has:
- Four **SIMD32** units (each executing 32 lanes per cycle)
- 128 KB of LDS per WGP (versus 64 KB per CU in GCN)
- A new **L1 vector cache (GL1)** shared within the WGP
- Native **Wave32** support: one SIMD32 dispatches a full wavefront in one cycle

Wave32 is the preferred mode on RDNA because it achieves the same throughput as Wave64 on GCN while
completing in half the registers. RADV selects wave size per shader stage at compile time based on
the workgroup size, subgroup requirements, and the device's preferred wave size for each stage.

**RDNA3 (GFX11)** added dual-issue: each SIMD32 can issue two VALU instructions per cycle to
separate pipelines. **RDNA4 (GFX12)** extended the register file to 192 KB per SIMD and introduced
dynamic VGPR allocation via the `s_alloc_vgpr` instruction, allowing shaders to request additional
registers at runtime. These hardware differences propagate into RADV through capability flags in
`struct radeon_info`.

### Hardware Compression Metadata

AMD GPUs use several on-chip compression schemes that RADV must manage:
- **DCC (Delta Color Compression)**: applied to colour render targets. On layout transitions to
  `SHADER_READ_ONLY_OPTIMAL`, RADV must decompress DCC if the texture sampler cannot consume
  compressed data. Handled via meta-passes in `radv_meta_decompress.c`.
- **HTILE**: hierarchical depth-stencil compression. Fast depth clears write to HTILE; the driver
  must flush HTILE before a depth image is read as a texture.
- **CMASK / FMASK**: coverage mask metadata for MSAA images. FMASK stores per-sample coverage,
  and RADV resolves FMASK during MSAA resolve operations.
- **SMASK**: stencil compression on GFX9+.

Cache hierarchy that RADV must invalidate:
- **L1K (scalar cache / K$)** — holds scalar data fetched via `SMEM`
- **GL1 (vector L1)** — per-WGP vector data (RDNA only)
- **GL2 (L2 TCC)** — shared L2 across all SEs; main target for cache flushes
- **CB/DB metadata caches** — hold DCC, HTILE, CMASK state

---

## 3. Source Tree Structure

RADV lives under `src/amd/vulkan/`. The directory has grown substantially since 2016 and is now
organised by subsystem:

```
src/amd/vulkan/
├── radv_instance.c/h         — VkInstance lifecycle, global state
├── radv_physical_device.c/h  — hardware capability enumeration
├── radv_device.c/h           — VkDevice, queue families, special BOs
├── radv_device_memory.c      — VkDeviceMemory allocation/mapping
├── radv_queue.c              — queue submission, timeline semaphores
├── radv_cmd_buffer.c/h       — command recording, dirty state tracking
├── radv_cs.h                 — inline PM4 packet helpers
├── radv_shader.c/h           — shader compilation orchestration
├── radv_shader_info.c/h      — shader analysis pass (radv_shader_info)
├── radv_shader_args.c/h      — SysVal → SGPR user-data mapping
├── radv_shader_object.c/h    — VK_EXT_shader_object implementation
├── radv_pipeline.c/h         — base radv_pipeline struct
├── radv_pipeline_graphics.c/h— graphics pipeline, prolog/epilog
├── radv_pipeline_compute.c/h — compute pipeline
├── radv_pipeline_rt.c        — ray tracing pipeline
├── radv_pipeline_cache.c     — VkPipelineCache and disk cache
├── radv_descriptor_set.c/h   — descriptor sets, pools, layouts
├── radv_image.c              — VkImage, tiling, DCC/HTILE tracking
├── radv_formats.c            — format table, DRM format support
├── radv_pass.c               — legacy VkRenderPass objects
├── radv_query.c              — occlusion/timestamp/stats queries
├── radv_video.c/h            — Vulkan Video decode (H.264/H.265/AV1)
├── radv_video_enc.c          — Vulkan Video encode
├── radv_debug.c/h            — RADV_DEBUG parsing, hang detection
├── radv_rmv.c                — Radeon Memory Visualizer integration
├── radv_dgc.c/h              — VK_EXT_device_generated_commands
├── radv_wsi.c                — WSI base dispatch
├── radv_wsi_wayland.c        — Wayland DMA-BUF presentation
├── radv_wsi_x11.c            — Xlib/XCB WSI
├── radv_android.c            — AHardwareBuffer import
├── nir/
│   ├── radv_nir.h
│   ├── radv_nir_lower_io.c
│   ├── radv_nir_lower_vs_inputs.c
│   ├── radv_nir_lower_fs_barycentric.c
│   ├── radv_nir_opt_fs_builtins.c
│   ├── radv_nir_rt_shader.c       — RT traversal loop, callable lowering
│   └── radv_nir_lower_hit_attrib_derefs.c
├── bvh/                       — compute-shader BVH builder
│   ├── build_interface.h
│   ├── lbvh.comp
│   ├── ploc.comp
│   ├── encode.comp
│   └── ... (GFX12 BVH8 variants)
├── layers/
│   ├── radv_sqtt_layer.c      — SQTT/RGP thread trace layer
│   └── radv_no_mans_sky.c     — game-specific workaround
└── winsys/amdgpu/
    ├── radv_amdgpu_winsys.c/h
    ├── radv_amdgpu_bo.c/h     — GEM BO allocation, DMA-BUF
    └── radv_amdgpu_cs.c       — IB submission via amdgpu_cs_submit
```

The `winsys/amdgpu/` layer abstracts the libdrm-amdgpu API behind a vtable (`struct radeon_winsys`)
so that the driver core never calls libdrm directly. This design makes it possible to write a
null/testing winsys without a real GPU.

The `nir/` subdirectory holds RADV-specific NIR lowering passes that run after the generic Mesa
`spirv_to_nir()` conversion and before ACO instruction selection. The `bvh/` subdirectory holds
GLSL compute shaders compiled offline into the RADV binary to implement the GPU-side BVH builder
for ray tracing.

---

## 4. Instance, Physical Device, and Logical Device

### `radv_instance`

`struct radv_instance` (in `radv_instance.c`) holds global state shared across all devices:
extension availability, layer hooks, the `RADV_DEBUG` and `RADV_PERFTEST` flag bitmasks parsed at
startup, and the list of enumerated physical devices. `radv_instance_init()` parses environment
variables early so that flag checks throughout the driver are simple bitmask tests.

### `radv_physical_device`

`struct radv_physical_device` (in `radv_physical_device.c/.h`) wraps `struct radeon_info`, the
hardware description struct populated by `amdgpu_query_gpu_info()` from libdrm-amdgpu. Key fields:

```c
/* src/amd/common/ac_gpu_info.h (simplified) */
struct radeon_info {
    enum amd_gfx_level   gfx_level;     /* GFX6 .. GFX12 */
    enum radeon_family   family;         /* CHIP_TAHITI .. RX 9070 */
    uint32_t             num_shader_engines;
    uint32_t             num_cu_per_sh;
    bool                 has_hw_raytracing;   /* GFX10.3+ */
    bool                 has_ngp;             /* NGG primitive shaders */
    bool                 has_vrs_hw;          /* VRS, GFX10.3+ */
    bool                 has_dedicated_vram;
    bool                 has_tmz_support;     /* protected memory */
    uint64_t             vram_size;
    uint64_t             gart_size;
    uint32_t             ps_wave_size;  /* preferred PS wave size */
    uint32_t             ge_wave_size;  /* preferred GE wave size */
};
```

`radv_physical_device_init_mem_types()` configures Vulkan memory heaps: one heap for VRAM
(`VK_MEMORY_HEAP_DEVICE_LOCAL_BIT`) and one for host-visible GTT, with appropriate
`VK_MEMORY_PROPERTY_*` flags. On APUs with UMA the two heaps collapse into a single coherent heap.

`radv_physical_device_init_cache_key()` generates a 20-byte cache key encoding the driver version,
chip family, and GFX level. This key invalidates the pipeline disk cache when the driver or hardware
configuration changes.

Hardware capability flags are checked throughout the driver using a helper pattern:

```c
/* Example: gate NGG culling on RDNA2+ */
if (pdev->rad_info.gfx_level >= GFX10_3 && !RADV_DEBUG_IS_SET(instance, NONGGC)) {
    /* enable NGG culling */
}
```

### `radv_device`

`struct radv_device` (in `radv_device.c/.h`) represents a `VkDevice`. It holds:
- `ws` — pointer to the `radeon_winsys` vtable (the amdgpu backend)
- Per-queue-family `radv_queue` objects backed by amdgpu hardware rings (GFX ring, ACE compute
  ring, SDMA transfer ring, VCN video decode/encode rings)
- A border-colour BO for `VK_BORDER_COLOR_FLOAT_CUSTOM_EXT`
- A GPU-side timestamp query pool for `vkCmdWriteTimestamp`
- The pipeline cache (`radv_pipeline_cache`)
- The slab allocator for GPU events

The winsys vtable decouples the device from libdrm:

```c
/* radv_device.c — submit using the vtable, never calling libdrm directly */
result = device->ws->cs_submit(device->ws, queue->hw_ctx,
                               submit_info, &fence_info);
```

### WSI Integration

Window system integration is handled by per-platform files. `radv_wsi_wayland.c` implements
`VK_KHR_wayland_surface`, creating swapchain images backed by wl_buffer objects imported via
DMA-BUF. Each swapchain image is a GEM BO exported with `amdgpu_bo_export()`, turned into a
DMA-BUF file descriptor, and imported into the compositor via the `wl_drm` or `linux-dmabuf-v1`
Wayland protocols. On Xorg, `radv_wsi_x11.c` uses DRI3 for GPU-to-GPU swapchain transfer.

---

## 5. Memory Management

### GEM Buffer Objects

Memory allocation in RADV flows through the winsys vtable:

```
vkAllocateMemory()
  → radv_alloc_memory()              [radv_device_memory.c]
    → radv_bo_create()
      → ws->buffer_create()          [vtable]
        → radv_amdgpu_winsys_bo_create()  [winsys/amdgpu/radv_amdgpu_bo.c]
          → amdgpu_bo_alloc()        [libdrm: DRM_IOCTL_AMDGPU_GEM_CREATE]
```

The `struct amdgpu_bo_alloc_request` carries GEM creation flags:

```c
/* winsys/amdgpu/radv_amdgpu_bo.c — domain and flags selection */
struct amdgpu_bo_alloc_request alloc_request = {
    .alloc_size    = size,
    .phys_alignment = alignment,
    .preferred_heap = AMDGPU_GEM_DOMAIN_VRAM,  /* device-local */
    .flags = AMDGPU_GEM_CREATE_NO_CPU_ACCESS    /* GPU-only, no CPU mapping */
              | AMDGPU_GEM_CREATE_VRAM_CLEARED, /* zero on alloc */
};
```

Key flags:
- `AMDGPU_GEM_CREATE_CPU_ACCESS_REQUIRED` — place in GTT (host-visible)
- `AMDGPU_GEM_CREATE_CPU_GTT_USWC` — write-combining map (fast CPU writes)
- `AMDGPU_GEM_CREATE_NO_CPU_ACCESS` — VRAM only, no CPU mapping possible
- `AMDGPU_GEM_CREATE_EXPLICIT_SYNC` — disable kernel implicit synchronisation (DRM 4.22+)
- `AMDGPU_GEM_CREATE_VM_ALWAYS_VALID` — keep BO resident in GPU VM always
- `AMDGPU_GEM_CREATE_ENCRYPTED` — TMZ protected memory (GFX9+)

### Memory Domains

RADV maps Vulkan memory property flags to amdgpu domains:

| Vulkan Memory Properties | amdgpu Domain | Use Case |
|--------------------------|---------------|----------|
| `DEVICE_LOCAL` | `AMDGPU_GEM_DOMAIN_VRAM` | Textures, render targets, shader BOs |
| `HOST_VISIBLE \| HOST_COHERENT` | `AMDGPU_GEM_DOMAIN_GTT` | Staging buffers, upload heaps |
| `HOST_VISIBLE \| HOST_CACHED` | GTT + `CPU_GTT_USWC` off | Readback buffers |
| `DEVICE_LOCAL \| HOST_VISIBLE` (SAM) | VRAM (CPU-visible on APU / Smart Access Memory) | Direct VRAM writes from CPU |

On discrete GPUs with Resizable BAR (Smart Access Memory / SAM) enabled, `DEVICE_LOCAL |
HOST_VISIBLE` memory maps directly into the PCIe aperture, exposing the full VRAM to the CPU.
RADV detects this via `rad_info.has_dedicated_vram` and whether the VRAM heap is CPU-accessible,
then enables `RADV_PERFTEST=sam` optimisations.

### CPU Mapping

```
vkMapMemory()
  → ws->buffer_map()
    → amdgpu_bo_cpu_map()   [libdrm: mmap on GEM fd → CPU virtual address]
```

### DMA-BUF Import and Export

DMA-BUF is the kernel-level mechanism for sharing GPU BOs across driver boundaries (e.g., sharing a
RADV render target with a V4L2 video encoder or a Wayland compositor):

```c
/* Export: radv_amdgpu_bo.c */
amdgpu_bo_export(bo_handle, amdgpu_bo_handle_type_dma_buf_fd, &dma_buf_fd);

/* Import: radv_amdgpu_bo.c */
amdgpu_bo_import(ws->dev, amdgpu_bo_handle_type_dma_buf_fd,
                 dma_buf_fd, &import_result);
```

Exported DMA-BUF file descriptors can be passed to GBM (`gbm_bo_import()`) for display or to
V4L2 for hardware video encode/decode. This is the mechanism that enables zero-copy game-to-wayland
compositor frame delivery.

### Sparse Memory

Sparse binding (`VK_EXT_sparse_residency`) is supported on GFX9+ as an experimental feature
(`RADV_EXPERIMENTAL=sparse`). Sparse BOs use GPU VM range reservations via `amdgpu_vm_reserve_va()`
and a dedicated sparse queue backed by the DMA engine.

---

## 6. Command Buffer Recording

### Data Structures

`struct radv_cmd_buffer` (in `radv_cmd_buffer.h`) is RADV's most complex structure:

```c
struct radv_cmd_buffer {
    /* The PM4 command stream under construction */
    struct radeon_cmdbuf    *cs;

    /* All mutable GPU state tracked for lazy emission */
    struct radv_cmd_state    state;

    /* Bitmask of RADV_CMD_DIRTY_* — high-level state changes */
    uint64_t                 dirty;

    /* Bitmask of RADV_DYNAMIC_* — Vulkan dynamic state */
    uint64_t                 dynamic_state_dirty;

    /* Which push constant stages are dirty */
    VkShaderStageFlags       push_constant_stages;
};
```

`struct radv_cmd_state` tracks the logical GPU state: the bound `radv_graphics_pipeline`,
currently active render attachments (`radv_rendering_state`), bound index buffer, bound vertex
buffers, viewport and scissor arrays, and the full dynamic state block for all `VK_EXT_extended_dynamic_state*` values.

### Lazy State Emission

A central performance strategy in RADV is **lazy PM4 register emission**. API calls like
`vkCmdSetViewport()`, `vkCmdBindPipeline()`, and `vkCmdSetDepthBias()` only set dirty flags;
they write no PM4 packets. The actual register writes are deferred until `vkCmdDraw*()` calls
`radv_emit_all_graphics_states()`, which checks each dirty bit and emits only the changed registers:

```c
/* radv_cmd_buffer.c — simplified draw preamble */
static void radv_emit_all_graphics_states(struct radv_cmd_buffer *cmd_buffer)
{
    if (cmd_buffer->dirty & RADV_CMD_DIRTY_PIPELINE)
        radv_emit_graphics_pipeline(cmd_buffer);

    if (cmd_buffer->dynamic_state_dirty & RADV_DYNAMIC_VIEWPORT)
        radv_emit_viewport(cmd_buffer);

    if (cmd_buffer->dynamic_state_dirty & RADV_DYNAMIC_SCISSOR)
        radv_emit_scissor(cmd_buffer);

    /* ... cache flush decisions ... */
    if (cmd_buffer->state.flush_bits)
        radv_emit_cache_flush(cmd_buffer);

    cmd_buffer->dirty = 0;
}
```

This batches register writes and avoids redundant PM4 packets when the same state is bound multiple
times between draw calls — a common pattern in games that repeatedly bind the same pipeline.

### PM4 Packet Emission

`radv_cs.h` provides inline helpers for the AMD PM4 packet format:

```c
/* radv_cs.h — set a context register (IT_SET_CONTEXT_REG) */
static inline void
radeon_set_context_reg(struct radeon_cmdbuf *cs, unsigned reg, unsigned value)
{
    radeon_emit(cs, PKT3(PKT3_SET_CONTEXT_REG, 1, 0));
    radeon_emit(cs, (reg - SI_CONTEXT_REG_OFFSET) >> 2);
    radeon_emit(cs, value);
}

/* Set a shader register (IT_SET_SH_REG) */
static inline void
radeon_set_sh_reg(struct radeon_cmdbuf *cs, unsigned reg, unsigned value)
{
    radeon_emit(cs, PKT3(PKT3_SET_SH_REG, 1, 0));
    radeon_emit(cs, (reg - SI_SH_REG_OFFSET) >> 2);
    radeon_emit(cs, value);
}
```

Draw calls emit `PKT3_DRAW_INDEX_2` (indexed) or `PKT3_DRAW_INDEX_AUTO` (non-indexed) packets
with vertex count and instance count fields.

### Cache Flush Bitmasks

`radv_emit_cache_flush()` translates the `RADV_CMD_FLAG_*` bitmask accumulated during the command
buffer into `PKT3_EVENT_WRITE` and `PKT3_ACQUIRE_MEM` packets:

```c
/* Selected cache flush flags (radv_cmd_buffer.h) */
RADV_CMD_FLAG_INV_ICACHE          /* instruction cache */
RADV_CMD_FLAG_INV_SCACHE          /* scalar data cache (L1K) */
RADV_CMD_FLAG_INV_VCACHE          /* vector L1 (GL0/GL1) */
RADV_CMD_FLAG_INV_L2              /* L2 TCC */
RADV_CMD_FLAG_FLUSH_AND_INV_CB_META   /* DCC/CMASK/FMASK flush */
RADV_CMD_FLAG_FLUSH_AND_INV_DB_META   /* HTILE flush */
RADV_CMD_FLAG_PS_PARTIAL_FLUSH    /* wait for PS to drain */
RADV_CMD_FLAG_CS_PARTIAL_FLUSH    /* wait for CS to drain */
```

The set of flush flags needed depends on the source and destination access patterns at a pipeline
barrier. RADV coalesces flush flags accumulated across a command buffer's pipeline barriers and emits
them as a single `ACQUIRE_MEM` packet when they are actually needed.

### Kernel Submission

Completed command buffers are assembled as **Indirect Buffers (IBs)** and submitted to the kernel
via `amdgpu_cs_submit()`. The winsys layer in `radv_amdgpu_cs.c` builds the IB list, attaches
syncobj wait and signal operations for Vulkan semaphores and fences, and issues the ioctl:

```c
/* radv_amdgpu_cs.c — simplified submission path */
struct amdgpu_cs_request cs_request = {
    .ip_type    = AMDGPU_HW_IP_GFX,
    .number_of_ibs = ib_count,
    .ibs        = ibs,
    .fence_info = { .handle = fence_syncobj },
};
amdgpu_cs_submit(ctx, 0, &cs_request, 1);
```

Timeline semaphores are implemented via DRM syncobjs with payload values
(`AMDGPU_CS_FENCE_TO_HANDLE`), matching the Vulkan timeline semaphore model.

---

## 7. The Shader Compiler Pipeline

RADV's shader compilation path transforms SPIR-V into AMD GPU ISA through several stages. The ACO
compiler backend handles the final NIR-to-ISA transformation (covered in depth in Chapter 15); this
section focuses on the RADV-specific stages that precede ACO.

### Stage 1: SPIR-V to NIR

The entry point is `radv_shader_spirv_to_nir()` in `radv_shader.c`. It calls the Mesa-common
`spirv_to_nir()` function from `src/compiler/spirv/` with RADV-specific options:

```c
/* radv_shader.c — SPIR-V to NIR translation */
nir_shader *nir = spirv_to_nir(
    spirv_words, spirv_word_count,
    spec_entries, num_spec_entries,
    stage, entrypoint_name,
    &spirv_options,       /* relaxed precision, robust buffer access, etc. */
    &nir_options);        /* RADV-specific nir_shader_compiler_options */
```

**NIR is Mesa's "New Intermediate Representation"** — a typed SSA IR for GPU shaders. The name is
unrelated to any NVIDIA IR; it predates all such confusion and was coined within the Mesa project.

### Stage 2: RADV NIR Optimisation Loop

`radv_optimize_nir()` runs a convergence loop of NIR optimisation passes:

```c
/* radv_shader.c — optimisation loop (simplified) */
bool progress;
do {
    progress = false;
    progress |= nir_split_array_vars(nir, nir_var_function_temp);
    progress |= nir_opt_copy_prop_vars(nir);
    progress |= nir_opt_dead_write_vars(nir);
    progress |= nir_lower_phis_to_scalar(nir, NULL);
    progress |= nir_opt_dce(nir);
    progress |= nir_opt_loop(nir);
    progress |= nir_opt_cse(nir);
    progress |= nir_opt_algebraic(nir);
    progress |= nir_opt_constant_folding(nir);
    progress |= nir_opt_undef(nir);
} while (progress);
```

### Stage 3: RADV-Specific NIR Lowering Passes

After general optimisation, RADV applies its own lowering passes in `src/amd/vulkan/nir/`:

- **`radv_nir_lower_vs_inputs.c`**: Lowers vertex shader inputs — maps `gl_VertexIndex` and
  `gl_InstanceIndex` to SGPRs, handles vertex buffer descriptors.
- **`radv_nir_lower_io.c`**: Lowers I/O variables for each stage to explicit loads/stores; handles
  the memory layout differences between GCN and RDNA parameter caches.
- **`radv_nir_lower_fs_barycentric.c`**: Lowers barycentric interpolation coordinates
  (`gl_BaryCoordNoPerspEXT`, etc.) to the hardware `persp_center / linear_center` SysVals.
- **`radv_nir_opt_fs_builtins.c`**: Removes redundant fragment shader built-in reads.

### Stage 4: Shader Info Gathering

`radv_nir_shader_info_pass()` (in `radv_shader_info.c`) walks the NIR IR to populate
`struct radv_shader_info`:

```c
struct radv_shader_info {
    uint32_t   wave_size;           /* 32 or 64 */
    uint32_t   num_input_clips;
    uint32_t   num_input_culls;
    bool       uses_ngg;            /* NGG primitive shader */
    bool       uses_streamout;      /* transform feedback */
    bool       uses_ray_launch_size;
    uint8_t    cs.block_size[3];    /* compute workgroup dimensions */
    /* ... output masks, system value usage flags, etc. */
};
```

This information drives SGPR user-data layout in `radv_shader_args.c`, which maps Vulkan system
values (descriptor set base pointers, push constant address, view index, etc.) to specific SGPR
user-data slots. The hardware requires these to be pre-specified in the shader binary header.

### Stage 5: Wave Size Selection

Wave size is chosen per stage at compile time:

```
Compute:      Wave32 if workgroup size ≤ 32 threads, else Wave64 (GFX10+)
Fragment (PS):pdev->ps_wave_size — typically Wave64 on GFX9/GFX10, Wave32 on GFX11+
GS / legacy:  Wave64 on GFX10 and GFX10.3 (hardware limit); Wave32 on GFX11+
VS/TES/Mesh:  pdev->ge_wave_size
Ray tracing:  Wave32 on all RDNA (Mesa 26.0+, previously Wave64 on GFX10/GFX10.3)
```

Per-stage env var overrides: `RADV_PERFTEST=cswave32` forces compute to Wave32 on GFX10+;
`RADV_PERFTEST=pswave32` forces fragment shaders; `RADV_PERFTEST=gewave32` forces GE stages.

### Stage 6: ACO Compilation

The final stage calls `aco_compile_shader()` in `src/amd/compiler/`, passing the lowered NIR and
`radv_shader_info`. ACO performs:
- Instruction selection (`aco::select_program()`): NIR ops → VOP2/VOP3/SOP2/MUBUF/SMEM/MIMG
  pseudo-instructions
- Linear-scan register allocation with spill/fill
- Instruction scheduling (hide latency between VALU and SALU)
- Binary emission: AMD ISA (GCN or RDNA-specific opcodes per `gfx_level`)

The LLVM backend is still available in debug builds via `RADV_DEBUG=llvm`, but ACO is the default.
Graphics Pipeline Library (`VK_EXT_graphics_pipeline_library`) is disabled when the LLVM backend is
active because LLVM's compilation model is incompatible with GPL's fast-link approach.
[Source](https://www.mail-archive.com/mesa-commit@lists.freedesktop.org/msg137522.html)

### ACO Function Calls for Ray Tracing (Mesa 26.0+)

Prior to Mesa 26.0, RADV compiled ray tracing pipelines by inlining all callable shaders (any-hit,
intersection, closest-hit) into a single traversal megashader. This caused severe stutter when a
game loaded a new scene with many distinct shader groups, and prevented parallel compilation.

Mesa 26.0 introduced **true function calls** in ACO via a new `p_call` pseudo-instruction and the
`s_swappc_b64` hardware instruction for GPU subroutine calls. The `radv_nir_lower_call_abi` NIR
pass handles divergent calls — because different lanes may want to call different functions, the
pass wraps each callable function body in a per-lane pointer comparison:

```
/* Conceptual lowering of a divergent call */
foreach lane in wave:
    if (lane_function_ptr == current_function_ptr):
        execute function body
    else:
        mask off lane until function returns
```

Each RT shader is now compiled independently. Stack management uses per-thread scratch memory in
VRAM, with the caller's frame pointer advanced before a call and restored after return. The ABI
divides registers into preserved blocks (maintained across calls) and clobbered blocks (caller must
save/restore). Traversal shaders can use most clobbered registers; any-hit shaders preserve the
lower registers used for traversal temporaries.
[Source](https://docs.mesa3d.org/drivers/amd/function-calls.html)

The result: Ghostwire Tokyo's ray tracing pass accelerated by over 2× in Mesa 26.0. Compilation
times for RT pipelines dropped significantly due to parallel per-shader compilation.
[Source](https://pixelcluster.github.io/Mesa-26/)

---

## 8. Descriptor Handling

### Raw GPU VA Descriptors

RADV implements Vulkan descriptors as **raw GPU virtual address (VA) pointers** packed into
descriptor set BOs. A descriptor set is a flat memory region where each binding occupies a fixed
size:

| Descriptor Type | Size |
|----------------|------|
| Uniform/storage buffer | 4 DWORDs (V# buffer resource descriptor) |
| Sampled image (T#) | 8 DWORDs |
| Storage image | 8 DWORDs |
| Sampler (S#) | 4 DWORDs |
| Combined image-sampler | 12 DWORDs |
| Acceleration structure | 4 DWORDs (64-bit GPU VA) |

Descriptor sets are suballocated from pool BOs created at `vkCreateDescriptorPool()` time. The
pool BO is mapped CPU-side for fast host descriptor writes (`vkUpdateDescriptorSets()`).

### Buffer Device Address

`VK_KHR_buffer_device_address` is implemented by returning the GEM BO's GPU VA directly.
`vkGetBufferDeviceAddress()` returns the 64-bit VA, which shaders then use via
`OpLoad`/`OpStore` through `PhysicalStorageBuffer` pointers. On GFX9+ the GPU supports a flat
(non-segmented) 48-bit VA space, enabling true 64-bit pointer semantics.

### Push Descriptors

`VK_KHR_push_descriptor` allows updating descriptor sets inline in the command buffer without
pool allocation. RADV stores push descriptors in a dedicated region of the command buffer's
embedded constant space, updated by `vkCmdPushDescriptorSetKHR()`.

### `VK_EXT_descriptor_buffer`

Implemented in Mesa 23.0 (Samuel Pitoiset / Bas Nieuwenhuizen), this extension gives applications
direct control over descriptor memory layout, eliminating the driver's descriptor pool abstraction.
The application allocates a `VkBuffer` for descriptors; RADV writes raw T#/S#/V# data to that
buffer via `vkGetDescriptorEXT()`. On newer AMD hardware, using descriptor buffer instead of the
traditional binding model reduces CPU overhead by eliminating pool management and descriptor set
copy operations.
[Source](https://www.phoronix.com/news/RADV-VK_EXT_descriptor_buffer)

### Inline Uniform Blocks

`VK_EXT_inline_uniform_block` stores small uniform blocks directly in the descriptor set BO rather
than as a separate buffer, eliminating one level of indirection for small constant data (e.g.,
material parameters).

---

## 9. Render Passes and Dynamic Rendering

### Legacy Render Passes

`radv_pass.c` implements `VkRenderPass` objects. Each pass stores attachment descriptions (format,
load/store ops, initial/final layouts) and subpass dependencies. On `vkCmdBeginRenderPass()`, RADV:
1. Performs load ops — fast clears if `LOAD_OP_CLEAR` is specified, or DCC/HTILE init
2. Transitions image layouts by emitting cache flush flags
3. Sets up CB (colour buffer) and DB (depth-buffer) hardware registers

`vkCmdEndRenderPass()` performs store ops — resolves for MSAA, DCC flushes for sampled images.

### Dynamic Rendering (Vulkan 1.3 Core)

`VK_KHR_dynamic_rendering` (promoted to Vulkan 1.3) eliminates `VkRenderPass` and `VkFramebuffer`
objects. `vkCmdBeginRendering()` passes attachment descriptors inline:

```c
VkRenderingInfo rendering_info = {
    .sType = VK_STRUCTURE_TYPE_RENDERING_INFO,
    .renderArea = { .offset = {0,0}, .extent = {width, height} },
    .layerCount = 1,
    .colorAttachmentCount = 1,
    .pColorAttachments = &color_attachment,
    .pDepthAttachment  = &depth_attachment,
};
vkCmdBeginRendering(cmd_buffer, &rendering_info);
```

RADV stores this in `radv_cmd_state.render` — a `radv_rendering_state` struct — and emits CB/DB
hardware registers directly. There is no render pass compilation step; all state is determined at
record time.

### Variable Rate Shading (GFX10.3+)

`VK_KHR_fragment_shading_rate` (Mesa 22.x+) allows rendering regions at reduced shading rate.
On RDNA2 (GFX10.3) and later, RADV exposes:
- Pipeline shading rate (set in `VkPipelineFragmentShadingRateStateCreateInfoKHR`)
- Primitive shading rate (set per-primitive in the vertex/geometry stage)
- Attachment shading rate (per-tile rate image)

Supported rates: 1×1, 1×2, 2×1, 2×2 and combiners (`KEEP`, `REPLACE`, `MIN`, `MAX`, `MUL`).
Force-override for testing: `RADV_FORCE_VRS=2x2` (quarter resolution shading). Performance gain
in compatible scenes can reach 30%.

---

## 10. Ray Tracing in RADV

### Overview

RADV implements `VK_KHR_ray_tracing_pipeline` and `VK_KHR_ray_query` using a mix of hardware
acceleration instructions (GFX10.3+) and compute shader BVH building. Ray tracing was enabled by
default starting in Mesa 23.2 (prior to that, it required `RADV_PERFTEST=rt`).

### BVH Builder (`src/amd/vulkan/bvh/`)

The BVH builder runs entirely on the GPU as a series of compute shader dispatches. The build
pipeline:

1. **Radix sort** — sort leaf primitives by Morton code (space-filling curve) into a linearised
   spatial order
2. **LBVH (Linear BVH)** — build a binary BVH using the sorted Morton codes in O(n) time; produces
   a tree with correct hierarchy but suboptimal quality
3. **PLOC (Parallel Locally-Ordered Clustering)** — post-process the LBVH by locally re-pairing
   nearby nodes to reduce SAH cost; requires two passes over the sorted leaf array
4. **Encode** — convert the internal representation to the hardware BVH node format for traversal

PLOC was introduced to RADV's BVH builder and delivered a **33% performance improvement in
Quake II RTX** compared to the prior LBVH-only builder.
[Source](https://wccftech.com/amd-boosts-radeon-vulkan-radv-ray-tracing-performance-with-ploc-bvh-builder-33-improvement-in-quake-ii-rtx/)

### GFX10.3–GFX11: BVH4 Hardware Traversal

On RDNA2 and RDNA3 GPUs with hardware RT support (`rad_info.has_hw_raytracing`), RADV emits the
`image_bvh_intersect_ray` instruction for box and triangle intersection tests. The traversal loop
itself remains in shader code — RADV does not use fixed-function traversal hardware. BVH nodes are
in BVH4 format (up to 4 children per internal node):

```glsl
/* Conceptual traversal loop (radv_nir_rt_shader.c lowers this from SPIR-V) */
while (!stack_empty) {
    node = stack_pop();
    /* BVH4 box test — 4 children tested in parallel via hardware */
    vec4 hits = imageLoad_bvh_intersect_ray(bvh, node, ray_origin, ray_dir, t_min, t_max);
    for each hit child in order:
        stack_push(child);
}
```

### RDNA4 (GFX12): BVH8 and LDS Stack

RDNA4 introduces dedicated BVH8 traversal hardware in Mesa 25.2+:

- **`IMAGE_BVH8_INTERSECT_RAY`**: tests 8 child bounding boxes in a single instruction,
  doubling the traversal throughput per instruction vs BVH4
- **`DS_BVH_STACK_PUSH8_POP1_RTN_B32`**: a new LDS instruction that pushes 8 child references
  in one cycle and pops the nearest candidate. The traversal stack lives in LDS rather than
  VRAM registers, saving VGPR pressure.
- **Triangle pair compression** (Mesa 25.3): two triangles packed per BVH leaf node, reducing
  tree depth and memory bandwidth. Updatable BVHs also benefit from pair compression.
- **OBBs (Oriented Bounding Boxes)**: tighter fitting bounding boxes for scene geometry reduce
  false intersections.

The `RADV_DEBUG=bvh4` flag forces BVH4 even on GFX12 for debugging.

### Software RT Emulation

`RADV_EXPERIMENTAL=emulate_rt` forces software BVH traversal (pure compute shader, no hardware
`image_bvh_intersect_ray` instruction) even on HW RT-capable GPUs. Used for testing and for
running RT on pre-GFX10.3 hardware.

### Dispatch Z-Order Optimisation (Mesa 26.0)

A performance bug in RT dispatch organisation: 1D dispatches like `1920×1080` (used for a full-
screen ray tracing pass) mapped linearly to `(x, y)` thread IDs, producing poor cache locality
because adjacent horizontal pixels are far apart in VRAM. Mesa 26.0 applies **Z-order curve
bit-deinterleaving** to reorganise linear dispatch IDs into spatially coherent 2D tiles, improving
L2 cache hit rates for texture fetches during traversal.
[Source](https://pixelcluster.github.io/Mesa-26/)

---

## 11. Performance Features

### NGG (Next Generation Geometry) — GFX10+

NGG replaces the traditional VS→GS hardware pipeline with a compute-like workgroup model on RDNA.
Instead of separate ES (export shader) and GS stages, an NGG primitive shader handles vertex
transformation, optional culling, and primitive export in one pass:

```
Traditional GCN pipeline:  LS → HS → ES → GS → VS → PS
RDNA NGG pipeline:         LS → HS → NGG → PS
```

Each NGG wave allocates parameter cache space via `s_sendmsg(GS_ALLOC_REQ, ...)` and exports
vertices and primitives directly to the scan converter. NGG is always enabled on GFX11+ and is
the default on GFX10.3. On GFX10 it must be explicitly tested (`RADV_PERFTEST=ngg`; now the
default is to enable NGG on GFX10 when not using the LLVM backend).

**NGG Culling** eliminates back-facing and view-frustum-outside primitives before rasterisation,
reducing scan converter load. Default on GFX10.3+; can be disabled with `RADV_DEBUG=nonggc`.
Enable on GFX11+ with `RADV_PERFTEST=nggc`.

### Mesh Shaders (GFX11+) and Task Shaders

`VK_EXT_mesh_shader` on RDNA3 replaces the entire vertex-processing pipeline with task and mesh
shader workgroups:

- **Task shaders** run on the ACE (async compute) queue and dispatch mesh shader workgroups via
  `DispatchMesh()`. The task→mesh handoff uses a draw indirect buffer (DISPATCH_MESH_INDIRECT).
- **Mesh shaders** output vertices and primitives into the RDNA3 **attribute ring** — a VRAM
  buffer that replaces the GCN parameter cache. Any mesh invocation can write any attribute to any
  output index in the ring, enabling complex geometry LOD and culling in the mesh wave.

On GFX11, each thread can export more than one vertex/primitive, unlike the GFX10.3 constraint
that limited each NGG thread to one vertex export. Disable mesh shader support with
`RADV_DEBUG=nomeshshader`.

### Transform Feedback

`VK_EXT_transform_feedback` uses GDS (Global Data Share) counters to track the number of
primitives written to the stream-out buffer. RADV emits `DS_ORDERED_COUNT` instructions to
atomically update the GDS counter across waves.

### Vulkan Video (VCN Hardware)

`radv_video.c` implements `VK_KHR_video_decode_h264`, `VK_KHR_video_decode_h265`, and
AV1 decode using the VCN (Video Core Next) fixed-function block on GFX9+ discrete GPUs.
H.264 and H.265 decode was enabled in Mesa 23.1; encode support followed.

In Mesa 26.1, the AMD video decode implementation was **unified between RadeonSI and RADV** — both
drivers now share a single video decode implementation rather than maintaining duplicate code.
[Source](https://www.techedubyte.com/amd-video-decode-unified-mesa-26-1-radeonsi-radv/)

Video hardware is exposed via separate queue families:
- `VK_QUEUE_VIDEO_DECODE_BIT_KHR` → VCN decode ring
- `VK_QUEUE_VIDEO_ENCODE_BIT_KHR` → VCN encode ring

Experimental support for GFX6–GFX9 video paths is gated behind
`RADV_EXPERIMENTAL=video_decode` / `video_encode`.

### Dynamic VGPR (GFX12 Only)

RDNA4's `s_alloc_vgpr` instruction enables a wave to request additional VGPR allocation beyond
its static reservation, up to the 192 KB per-SIMD limit. RADV's ACO backend emits
`s_alloc_vgpr` for Wave32 compute shaders that benefit from expanded register files (e.g.,
large ray tracing payloads). This is only available for Wave32 and only on GFX12.

---

## 12. Pipeline Cache and Shader Object

### `VkPipelineCache` and Disk Cache

`radv_pipeline_cache.c` implements the two-level caching strategy:

1. **In-memory `VkPipelineCache`**: applications can pass a pipeline cache object at creation.
   RADV checks this first; on a hit, no compilation occurs.
2. **On-disk cache** (`$XDG_CACHE_HOME/mesa_shader_cache` or `~/.cache/mesa_shader_cache`): if
   the in-memory cache misses, RADV queries the Mesa disk cache keyed by the shader's SPIR-V hash
   combined with compilation options. On a hit, the compiled binary is loaded without invoking ACO.

The pipeline cache UUID encodes driver version and chip family, ensuring stale entries from a
previous driver version are never loaded. Setting `RADV_DEBUG=nocache` disables the disk cache for
debugging (forces recompilation every time).

### `VK_EXT_graphics_pipeline_library` (GPL)

Enabled by default in Mesa 23.1 (and disabled by `RADV_DEBUG=nogpl` or when `RADV_DEBUG=llvm`
is active). GPL splits a graphics pipeline into four independently compilable libraries:
- **Vertex input** — vertex attribute layout
- **Pre-rasterisation** — VS, TCS, TES, GS (or NGG), prolog/epilog
- **Fragment shader** — PS stage
- **Fragment output** — colour blend, depth test configuration

The **fast link** step merges pre-compiled libraries at pipeline creation time without re-invoking
ACO. Reports of 50,000% speedup in the final link step in some benchmarks reflect the cost of
the pre-compilation being amortised across many pipelines.

### `VK_KHR_pipeline_binary` (Mesa 24.3)

RADV was the first Mesa driver to implement this extension. `VkPipelineBinary` objects store
compiled per-stage binaries independently of `VkPipelineCache`, allowing applications to serialise
and restore GPU-ready binaries without the driver-opaque `VkPipelineCache` format.
`radv_pipeline_binary` holds per-stage code sizes and the binary data.

### `VK_EXT_shader_object` (Mesa 24.1+)

Implemented in `radv_shader_object.c`, this extension decouples shaders from pipeline objects.
Applications create `VkShaderEXT` handles and bind individual stages via `vkCmdBindShadersEXT()`.
RADV compiles each shader object independently, using the same ACO pipeline described in §7.
Disable with `RADV_DEBUG=noeso`.

---

## 13. Debugging Infrastructure

### `RADV_DEBUG` Environment Variable

A comma-separated list of debug flags (parsed in `radv_instance.c`):

**Compilation and shader dumps:**

| Flag | Effect |
|------|--------|
| `llvm` | Use LLVM backend instead of ACO (debug builds only) |
| `spirv` | Dump SPIR-V bytecode before compilation |
| `nir` | Dump NIR for selected stages |
| `preoptir` | Dump ACO/LLVM IR before optimisations |
| `shaders` | Dump final compiled ISA |
| `shaderstats` | Print register usage statistics |
| `checkir` | Validate NIR and ACO IR (slow) |
| `vs/ps/cs/gs/...` | Dump specific stage only |

**Rendering behaviour:**

| Flag | Effect |
|------|--------|
| `nodcc` | Disable Delta Color Compression |
| `nohiz` | Disable HTILE hierarchical Z |
| `nofmask` | Disable FMASK on MSAA images |
| `nobinning` | Disable primitive binning (GFX9 Vega) |
| `fullsync` | Insert full GPU sync after every draw/dispatch |
| `syncshaders` | Insert `ALL_COMMANDS → ALL_COMMANDS` barrier between commands |
| `nongg` | Disable NGG on GFX10/GFX10.3 |
| `nonggc` | Disable NGG culling |
| `nomeshshader` | Disable mesh shader support |
| `nocache` | Disable shader disk cache |
| `zerovram` | Zero all VRAM allocations on creation |
| `noeso` | Disable VK_EXT_shader_object |
| `nogpl` | Disable VK_EXT_graphics_pipeline_library |
| `bvh4` | Force BVH4 on GPUs that support BVH8 (GFX12) |

**A note on `syncshaders`**: this inserts a `VK_PIPELINE_STAGE_ALL_COMMANDS → ALL_COMMANDS`
barrier between every command, forcing serial GPU execution. This is the minimum required to
isolate which draw or dispatch is responsible for a GPU hang. It is different from `fullsync`,
which uses a hard `PIPE_SYNC` fence flush.

### GPU Hang Detection (`RADV_DEBUG=hang`)

RADV annotates command buffers with **breadcrumb values** — 32-bit counters written to a
BO at the start and end of each draw/dispatch. On GPU hang detection, the driver reads back the
last written breadcrumb (the last command that completed before the hang) and writes a dump
directory to `$HOME/radv_dumps_<pid>_<timestamp>/`:

```
radv_dumps_12345_20260619_143000/
├── *.spv              — SPIR-V for all shaders active at hang
├── pipeline.log       — ACO ISA, wave program counters, descriptor state
├── bo_history.log     — BO allocation/free history
├── bo_ranges.log      — active BO VA ranges
├── vm_fault.log       — GPU virtual memory fault addresses
├── trace.log          — annotated CP trace with draw markers
├── registers.log      — GPU register snapshot
├── umr_waves.log      — per-wave state via UMR
├── umr_ring.log       — ring buffer content via UMR
└── dmesg.log          — kernel dmesg capture
```

UMR (Userspace Management Ring) is an AMD developer tool for reading GPU register state and ring
buffer contents. RADV integrates with it automatically on hang; disable with `RADV_DEBUG=noumr`.
UMR requires `chmod +s $(which umr)` for setuid access.

### RGP / SQTT Profiling (`layers/radv_sqtt_layer.c`)

The AMD Radeon GPU Profiler (RGP) captures frame-level GPU timelines and per-instruction timing
via **SQTT (SQ Thread Trace)**, which samples the instruction issue history of every active wave.
Capture is triggered by:
- `MESA_VK_TRACE=rgp` (generic Mesa trace env var)
- `RADV_THREAD_TRACE_TRIGGER_FILE` (write to this path at the frame you want to capture)

Configuration options:
- `RADV_THREAD_TRACE_BUFFER_SIZE` — SQTT buffer size (default 32 MiB per SE)
- `RADV_THREAD_TRACE_INSTRUCTION_TIMING` — enable instruction timing
- `RADV_THREAD_TRACE_CACHE_COUNTERS` — L0/L1/L2 hit rates per wavefront (GFX10+)

RGP works on GFX8+ and is the standard profiling tool for AMD GPU performance work on Linux.
[Source](https://gpuopen.com/rgp/)

### Radeon Memory Visualizer (`radv_rmv.c`)

Enable with `MESA_VK_TRACE=rmv`. RMV captures a timeline of all BO allocations and frees with
size, type, and heap information. Use the Radeon Memory Visualizer GUI tool to inspect VRAM
fragmentation and identify memory leaks.
[Source](https://docs.mesa3d.org/envvars.html#radv-env-vars)

### `RADV_PERFTEST` Performance Experiments

Selected flags for application developers:

| Flag | Effect |
|------|--------|
| `cswave32` | Force Wave32 for compute (GFX10+) |
| `pswave32` | Force Wave32 for pixel shaders (GFX10+) |
| `gewave32` | Force Wave32 for vertex/tess/geometry (GFX10+) |
| `nggc` | Enable NGG culling on GFX11+ |
| `rtwave64` | Force Wave64 for ray tracing shaders |
| `rtcps` | Use CPS lowering mode for RT instead of function calls |
| `dccmsaa` | Enable DCC compression on MSAA render targets |
| `dmashaders` | Upload shaders to invisible VRAM (bandwidth test) |
| `localbos` | Enable local BO placement heuristic |
| `sam` | Enable Smart Access Memory (ReBAR) optimisations |
| `nosam` | Disable SAM optimisations |
| `nircache` | Cache per-stage NIR for graphics pipelines |

---

## 14. Integrations

**Chapter 15 — ACO: AMD's Optimising Compiler.** ACO is the direct compilation backend for
RADV. Everything described in §7 above — instruction selection from `aco::select_program()`,
linear-scan register allocation, instruction scheduling, and AMD ISA binary emission — is covered
in depth in Chapter 15, including ACO's correctness guarantees, its register allocator invariants,
and how `--enable-checks` verification works. Chapter 15 is the essential companion to §7 of this
chapter.

**Chapter 12 — amdgpu Kernel Driver.** RADV's winsys layer (`radv_amdgpu_bo.c`,
`radv_amdgpu_cs.c`) wraps the libdrm-amdgpu API, which in turn issues ioctls to the `amdgpu`
kernel driver: `DRM_IOCTL_AMDGPU_GEM_CREATE` (BO allocation), `DRM_IOCTL_AMDGPU_CS` (command
submission), and `DRM_IOCTL_PRIME_HANDLE_TO_FD` (DMA-BUF export). Chapter 12 covers the kernel
implementation of these ioctls, the amdgpu command processor, and the GEM memory management model
from the kernel perspective.

**Chapter 18 — Mesa Vulkan Drivers / Vulkan Clients of amdgpu.** Chapter 18 covers the Vulkan
ICD loader mechanism, the `VK_LAYER_*` intercept chain, and how `libvulkan_radeon.so` is registered
and selected as the active ICD. RADV is the AMD Vulkan ICD in this model. Chapter 18 also surveys
how RADV fits alongside ANV (Intel), NVK (NVIDIA), and Turnip (Qualcomm) in the Mesa Vulkan driver
family and the common infrastructure they all share in `src/vulkan/`.

**Chapter 48 — ROCm and the KFD.** RADV and ROCm both run on the `amdgpu` kernel driver, but
via different user-space paths. RADV uses the GEM/CS path (via libdrm-amdgpu); ROCm uses the
KFD (Kernel Fusion Driver) queue path with HSA-style AQL packets. The two stacks share the amdgpu
kernel module but do not otherwise communicate at the user-space level. Interoperability between
ROCm compute and RADV graphics is achieved via DMA-BUF buffer sharing and peer memory mapping.
Chapter 48 covers the KFD architecture and ROCm runtime in depth.

**Chapter 104 — DXVK and VKD3D-Proton.** DXVK (D3D9/D3D10/D3D11 → Vulkan) and VKD3D-Proton
(D3D12 → Vulkan) are RADV's largest application clients on Steam. The RADV features most important
to these translation layers are: `VK_EXT_graphics_pipeline_library` (eliminates pipeline
compilation stutter), `VK_EXT_descriptor_buffer` (reduces descriptor management overhead),
mesh shaders (Battlefront/UE5 games), ray tracing (DXR via VKD3D-Proton), and VRS (checkerboard
rendering). Chapter 104 explains how each translation layer maps D3D concepts to Vulkan and which
RADV capabilities it depends on.

**Chapter 135 — Ray Tracing on RADV.** Chapter 135 goes deeper on the BVH builder implementation
in `src/amd/vulkan/bvh/`: the PLOC algorithm in detail, the LBVH Morton-code sort, BVH node
encoding for GFX10.3 (BVH4) vs GFX12 (BVH8), callable shader ABI specifics, the traversal loop
NIR lowering in `radv_nir_rt_shader.c`, and dEQP-VK ray tracing conformance test results.
Chapter 135 is the natural extension of §10 in this chapter.

**Chapter 14 — NIR: Mesa's Shader Intermediate Representation.** All RADV shader compilation
begins with NIR. Chapter 14 describes the NIR data model (SSA values, basic blocks, intrinsics,
deref chains), the full set of core NIR optimisation passes, and how NIR is extended with new
intrinsics when hardware support requires it. The RADV-specific NIR lowering passes described in
§7.3 above build directly on the foundation in Chapter 14.

**Chapter 73 — Asahi Apple Silicon Driver.** The Asahi Linux Vulkan driver (AGX) and RADV both
derive from the Mesa Vulkan common infrastructure in `src/vulkan/`, using the same `vk_device`,
`vk_physical_device`, `vk_queue`, and `vk_command_buffer` base types. Comparing the two drivers
is instructive for understanding what the common infrastructure provides versus what each driver
implements for its own hardware. Chapter 73 covers the Asahi driver in detail.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
