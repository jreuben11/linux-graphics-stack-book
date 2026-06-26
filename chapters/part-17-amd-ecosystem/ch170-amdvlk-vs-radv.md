# Chapter 170: AMDVLK vs. RADV — AMD's Two Open Vulkan Drivers

> **Part**: Part XVII — AMD Developer Ecosystem
> **Audience**: Graphics application developers targeting AMD GPUs on Linux; driver developers; system
> integrators who need to choose between driver stacks.
> **Status**: First draft — 2026-06-20

---

## Table of Contents

1. [Introduction: Why AMD Had Two Open Vulkan Drivers](#1-introduction-why-amd-had-two-open-vulkan-drivers)
2. [AMDVLK Architecture](#2-amdvlk-architecture)
   - [XGL: The Vulkan Front-End](#21-xgl-the-vulkan-front-end)
   - [PAL: Platform Abstraction Layer](#22-pal-platform-abstraction-layer)
   - [LLPC: LLVM-Based Pipeline Compiler](#23-llpc-llvm-based-pipeline-compiler)
   - [GPURT: GPU Ray Tracing Library](#24-gpurt-gpu-ray-tracing-library)
   - [ICD Installation and Discovery](#25-icd-installation-and-discovery)
3. [RADV Architecture (Contrast)](#3-radv-architecture-contrast)
   - [NIR-Centric Pipeline](#31-nir-centric-pipeline)
   - [ACO: Custom AMD Compiler Backend](#32-aco-custom-amd-compiler-backend)
   - [Kernel Interface via radeon_winsys](#33-kernel-interface-via-radeon_winsys)
4. [Compiler Pipeline Comparison](#4-compiler-pipeline-comparison)
   - [AMDVLK: SPIR-V → LLPC → LLVM → ISA](#41-amdvlk-spir-v--llpc--llvm--isa)
   - [RADV: SPIR-V → NIR → ACO → ISA](#42-radv-spir-v--nir--aco--isa)
   - [Compilation Speed and Shader Quality](#43-compilation-speed-and-shader-quality)
   - [Pipeline Cache Behaviour](#44-pipeline-cache-behaviour)
5. [Extension and Feature Support](#5-extension-and-feature-support)
   - [Vulkan Conformance Level](#51-vulkan-conformance-level)
   - [Historical VK_AMD_* Extensions](#52-historical-vk_amd_-extensions)
   - [Ray Tracing](#53-ray-tracing)
   - [Mesh Shaders and Descriptor Buffers](#54-mesh-shaders-and-descriptor-buffers)
   - [Video Decode](#55-video-decode)
6. [Distribution Packaging and Driver Selection](#6-distribution-packaging-and-driver-selection)
   - [Default Drivers by Distro](#61-default-drivers-by-distro)
   - [Selecting the Active Driver](#62-selecting-the-active-driver)
   - [Coexistence: Running Both Drivers](#63-coexistence-running-both-drivers)
7. [Performance Comparison](#7-performance-comparison)
   - [Gaming Workloads](#71-gaming-workloads)
   - [Ray Tracing Workloads](#72-ray-tracing-workloads)
   - [Compute Workloads](#73-compute-workloads)
8. [The End of AMDVLK: AMD Consolidates on RADV](#8-the-end-of-amdvlk-amd-consolidates-on-radv)
9. [Stability, Bug Report Paths, and CTS](#9-stability-bug-report-paths-and-cts)
10. [Choosing Between AMDVLK and RADV: Decision Guide](#10-choosing-between-amdvlk-and-radv-decision-guide)
11. [Integrations](#11-integrations)

---

## 1. Introduction: Why AMD Had Two Open Vulkan Drivers

For most of the period between 2018 and 2025, AMD Linux users faced a choice that had no equivalent
in the Windows world: two distinct, fully open-source Vulkan Installable Client Drivers (ICDs) for
the same GPU. Both were community-visible, both targeted modern AMD hardware, and both claimed
Vulkan conformance. Yet they diverged almost completely in codebase, compiler strategy, and design
philosophy.

**RADV** was born in the summer of 2016 when Bas Nieuwenhuizen (then an independent contributor,
later Google) and Dave Airlie (Red Hat, DRM subsystem maintainer) wrote an unofficial Radeon Vulkan
driver over a single summer and merged it into mainline Mesa. At the time, AMD had not released a
Linux Vulkan driver; the only option was the closed `amdgpu-pro` binary blob. RADV was created both
to fill the gap and to nudge AMD into releasing its own open driver sooner.
[Source](https://www.phoronix.com/review/radv-hits-mesa)

**AMDVLK** appeared on GitHub on December 22, 2017, as AMD's official open-source release of the
Vulkan front-end (`XGL`) it had been developing internally — the same code-base that powers AMD's
Windows Vulkan driver. Unlike RADV, AMDVLK was not written from scratch for Linux; it was a
publication of AMD's existing cross-platform driver built on the Platform Abstraction Library (PAL)
that AMD uses for DirectX 12, OpenCL, and ROCm as well.
[Source](https://github.com/GPUOpen-Drivers/AMDVLK)

The two drivers coexisted for seven years. Their parallel existence raised a question that every AMD
Linux developer faced: which one is installed on my system by default, and which one should I use for
my application? The answer changed significantly in 2025, when AMD announced the discontinuation of
AMDVLK in favour of full investment in RADV. Understanding what distinguished these drivers — and
why AMD eventually consolidated — remains practically and historically relevant to anyone writing
Vulkan software targeting AMD hardware on Linux.

This chapter covers the architecture of both drivers (Sections 2–3), their diverging compiler
strategies (Section 4), their extension and feature support matrices (Section 5), how distributions
package and users select between them (Section 6), the performance picture that led AMD to
consolidate (Section 7), the consolidation story itself (Section 8), and practical guidance for
anyone still maintaining or migrating code (Sections 9–10). Code examples focus on ICD selection,
driver identification, and cache management.

---

## 2. AMDVLK Architecture

AMDVLK is assembled from five source repositories that together form a self-contained Vulkan stack
sitting on top of the `amdgpu` kernel module
[Source](https://github.com/GPUOpen-Drivers/AMDVLK):

| Repository | Role |
|------------|------|
| `xgl` | Vulkan API front-end |
| `pal` | Platform Abstraction Layer |
| `llpc` | LLVM-based pipeline compiler |
| `gpurt` | GPU ray tracing library |
| `llvm` | Forked LLVM with AMD GCN/RDNA backend |

### 2.1 XGL: The Vulkan Front-End

`XGL` ([github.com/GPUOpen-Drivers/xgl](https://github.com/GPUOpen-Drivers/xgl)) is the Vulkan
"ICD layer": it exports the symbols required by the Vulkan loader (e.g., `vkCreateDevice`,
`vkCreateGraphicsPipelines`) and translates every Vulkan call into PAL API calls. XGL handles:

- Vulkan instance and device enumeration
- Memory management integration between `VkDeviceMemory` and PAL `IGpuMemory` objects
- Descriptor-set layout compilation into PAL-compatible tables
- Render pass management (including split render passes for deferred-tile-rendering modes on APUs)
- Synchronisation primitives: `VkSemaphore`, `VkFence`, and `VkEvent` mapped onto PAL queue fences
  and events

XGL does not compile shaders. All shader and pipeline work is delegated to LLPC.

### 2.2 PAL: Platform Abstraction Layer

PAL ([github.com/GPUOpen-Drivers/pal](https://github.com/GPUOpen-Drivers/pal)) is AMD's internal
cross-platform GPU abstraction library, shared between the Windows DX12 driver, the Linux Vulkan
driver, and ROCm. It presents a uniform C++ interface to GPU hardware regardless of the operating
system or graphics API. Key PAL responsibilities:

- **KMD bridge**: PAL's `Pal::Linux` module submits command buffers to the kernel using raw
  `amdgpu` DRM ioctls — `DRM_IOCTL_AMDGPU_CS`, `DRM_IOCTL_AMDGPU_GEM_CREATE`, and so on — without
  going through `libdrm`'s higher-level wrappers.
- **Command buffer encoding**: PAL's `CmdBuffer` class encodes packets in the PM4 command-processor
  format that the AMDGPU kernel module submits to the GPU's ring buffers.
- **Queue management**: PAL exposes graphics, compute, and DMA queues that map to the hardware
  scheduler queues exposed by `amdgpu`.
- **Memory management**: PAL handles VRAM and system-memory allocation, heap selection (VRAM local,
  VRAM invisible, GTT), and residency management.
- **Pipeline ABI**: PAL defines the ELF-based ABI that compiled pipeline objects must conform to.
  LLPC produces ELF objects in the PAL ABI format; PAL loads them and extracts the GPU binary.

PAL explicitly does **not** compile shaders — its ABI requires fully compiled pipelines as inputs,
not individual shader stages.

An important consequence of the PAL architecture: because PAL is shared with AMD's Windows DX12
driver and ROCm stack, bug fixes in PAL's command-buffer encoder or memory allocator propagate to
all of AMD's cross-platform drivers simultaneously. This was a strength for AMDVLK — Windows DX12
correctness fixes often appeared in AMDVLK shortly after — and also a constraint, because PAL's
abstractions were designed for a wide range of APIs and operating systems rather than being
optimised specifically for the Linux + Vulkan use case.

The Linux port of PAL (in PAL's `src/core/os/lnx/` subtree) provides thin wrappers around the
`amdgpu` DRM kernel interface. Command submission flows through PAL's `Queue::Submit()` →
`Device::Submit()` → `DrmLoaderFuncs::pfnAmdgpuCsSubmit()` call chain, ultimately invoking
`DRM_IOCTL_AMDGPU_CS`. Buffer object lifecycle (allocation, export, import, mmap for CPU access)
maps similarly to `DRM_IOCTL_AMDGPU_GEM_CREATE` and related ioctls.
[Source](https://docs.kernel.org/gpu/amdgpu/ring-buffer.html)

### 2.3 LLPC: LLVM-Based Pipeline Compiler

LLPC ([github.com/GPUOpen-Drivers/llpc](https://github.com/GPUOpen-Drivers/llpc)) is the shader
compiler that transforms SPIR-V binary into GPU machine code packaged as a PAL ABI-compliant ELF.
Its compilation pipeline has three phases
[Source](https://github.com/GPUOpen-Drivers/llpc/blob/dev/llpc/docs/LlpcOverview.md):

**Front-end** — translates SPIR-V to LLVM IR:
- The Khronos SPIR-V–to–LLVM translator converts SPIR-V instructions to LLVM IR enriched with
  `lgc.call.*` pseudo-calls that encode Vulkan-specific resource accesses (image samples, buffer
  loads, push constants).
- Vulkan-specific metadata (descriptor set layouts, pipeline layout, specialisation constants) is
  embedded into the LLVM module as metadata nodes.

**Middle-end (LGC — LLVM GPU Compiler)** — lowers the IR to hardware-conformant form:
- Runs the `BuilderReplayer` pass to materialise `lgc.call.*` calls into concrete LLVM IR operating
  on buffer and image descriptors.
- Performs target-specific shader merging: on GFX9+, vertex and hull shaders are merged (LS-HS);
  on GFX10+ with NGG enabled, the geometry stage merges with the pre-rasterisation stages (ES-GS).
- Inserts SGPR and VGPR argument setup so that the compiled shader conforms to the wave-dispatch
  calling convention expected by the CP micro-scheduler.
- Lowers resource accesses to `llvm.amdgcn.*` intrinsics understood by LLVM's AMDGPU back-end.

**Back-end (LLVM AMDGPU)** — standard LLVM passes applied to `amdgcn` target:
- Instruction selection, SIMD control-flow lowering, instruction scheduling, register allocation,
  and ISA emission.
- Produces an ELF `amdgpu` object embedding the GPU binary and PAL metadata sections.

LLPC also exposes a standalone binary tool `amdllpc` that can compile SPIR-V offline:

```bash
# Compile a vertex/fragment shader pair offline with LLPC
amdllpc --gfxip=10.3 \
        --spvgen-dir=/usr/share/spvgen \
        vertex.spv \
        fragment.spv \
        -o pipeline.elf
```

### 2.4 GPURT: GPU Ray Tracing Library

GPURT ([github.com/GPUOpen-Drivers/gpurt](https://github.com/GPUOpen-Drivers/gpurt)) is a static
library that implements BVH (Bounding Volume Hierarchy) construction and traversal for ray tracing
workloads. It is used by both the DX12 and Vulkan paths inside AMD's driver stack. GPURT provides:

- Acceleration structure building for `VkAccelerationStructureKHR` (bottom-level and top-level BVH)
- Traversal shader written in HLSL (compiled via DirectXShaderCompiler and embedded in the library)
  that implements `TraceRayKHR()` intrinsic semantics
- Ray history capture for debugging with AMD's Radeon Developer Tool Suite

GPURT was archived as part of AMD's consolidation when AMDVLK was discontinued in September 2025.

### 2.5 ICD Installation and Discovery

When AMDVLK is installed, it places two files on disk:

1. A shared library: `amdvlk64.so` (64-bit; `amdvlk32.so` for 32-bit if present)
2. An ICD JSON descriptor in `/etc/vulkan/icd.d/` (or `/usr/share/vulkan/icd.d/` on some distros)

```json
// /etc/vulkan/icd.d/amd_icd64.json
{
    "file_format_version": "1.0.0",
    "ICD": {
        "library_path": "/usr/lib/x86_64-linux-gnu/amdvlk64.so",
        "api_version": "1.4.313"
    }
}
```

The Vulkan loader discovers all JSON files in the `icd.d` directories on `VK_ICD_FILENAMES` and
`VK_DRIVER_FILES` (Vulkan 1.3.234+ name). When both AMDVLK and RADV ICDs are present, the loader
enumerates both physical devices, typically presenting them in JSON-directory-scan order.

---

## 3. RADV Architecture (Contrast)

RADV ([Mesa source at `src/amd/vulkan/`](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/amd/vulkan))
takes a wholly different approach. It is built on Mesa's shared Vulkan infrastructure
(`src/vulkan/`) and uses Mesa's common NIR intermediate representation rather than LLVM IR as its
primary middle-layer.

### 3.1 NIR-Centric Pipeline

SPIR-V enters RADV through Mesa's shared `spirv_to_nir()` translator, which produces NIR — a
structured SSA IR designed specifically for GPU shader compilation and shared across Mesa's OpenGL,
OpenCL, and Vulkan drivers. RADV then runs a long chain of NIR lowering passes via
`radv_optimize_nir()` that specialise the IR for AMD hardware: scalarisation of vector operations,
vectorisation of image address calculations, lowering of subgroup operations to GCN wave instructions,
and AMD-specific intrinsic substitution. The result feeds into the ACO backend.
[Source](https://deepwiki.com/bminor/mesa-mesa/2.1.3-radv-pipeline-and-shader-compilation)

RADV's Vulkan implementation — `radv_cmd_buffer`, `radv_pipeline`, `radv_descriptor_set` — is
implemented directly against Mesa's Vulkan common layer rather than going through a PAL equivalent.
This means RADV's kernel submission path uses `libdrm_amdgpu` in `src/amd/common/ac_drm.c` rather
than reimplementing raw ioctls.

### 3.2 ACO: Custom AMD Compiler Backend

The ACO compiler ([`src/amd/compiler/`](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/amd/compiler))
was contributed primarily by Valve engineers (Timothy Arceri, Daniel Schürmann, and others) and merged
in Mesa 20.1. ACO was designed from first principles as a shader compiler for AMD GCN and RDNA
ISAs, with four goals: fast compile time, high shader quality, wave-size flexibility (wave32 vs.
wave64), and full correctness under CTS. Key components:

- **Instruction selection**: `aco::select_program()` converts NIR instructions to ACO IR, a typed
  instruction-level IR with explicit SGPR/VGPR allocation classes.
- **Register allocator**: A linear-scan allocator with special handling for AMD's banked register
  file, split-register VGPR pairs (64-bit loads), and M0 register constraints.
- **Instruction scheduler**: A list scheduler that hides memory latency and respects SIMD execution
  hazards on GFX10.3 and later.
- **ISA emission**: Direct binary emission into the PM4 packet stream without going through LLVM's
  MCCodeEmitter.

The LLVM backend still exists in RADV for hardware bringup and automated testing, selectable via
`RADV_DEBUG=llvm`, but Mesa documentation explicitly recommends against it for production use.
[Source](https://docs.mesa3d.org/drivers/radv.html)

### 3.3 Kernel Interface via radeon_winsys

RADV submits command buffers to the `amdgpu` kernel module through the `radeon_winsys` abstraction
in `src/amd/common/`. This uses `libdrm_amdgpu`'s public API — `amdgpu_cs_submit()`,
`amdgpu_bo_alloc()` — so RADV benefits from any improvements to the `libdrm` layer without needing
its own ioctl implementations. The winsys layer handles BO (Buffer Object) residency, fence
management, and ring-buffer scheduling.

RADV's winsys has two codepaths: the traditional `amdgpu` path (used by default) and an
experimental "user queue" codepath that allows job submission without going through the kernel
driver's CS ioctl, reducing dispatch latency. The user queue path was being developed for RDNA4
class hardware as of 2025. **Note: verify user queue enablement status for Mesa 26.x.**

Unlike PAL, RADV does not maintain an independent BO residency manager — it relies on the `amdgpu`
kernel's implicit residency tracking and lets the kernel scheduler evict and restore BOs as needed.
This simplifies the RADV codebase but means RADV has less direct control over which buffers are
resident in VRAM at any given moment compared to PAL's explicit residency management.

RADV's source tree structure under `src/amd/vulkan/` follows Mesa conventions:

```
src/amd/vulkan/
├── radv_cmd_buffer.c      # Command buffer recording and patch-up
├── radv_device.c          # VkDevice creation and teardown
├── radv_descriptor_set.c  # Descriptor set and pool management
├── radv_pipeline.c        # Pipeline creation and state cache
├── radv_pipeline_rt.c     # Ray tracing pipeline
├── radv_shader.c          # Shader object lifecycle (ACO/LLVM backend)
├── radv_image.c           # Image and image view creation
├── radv_wsi.c             # WSI swapchain (Wayland, X11, headless)
└── nir/                   # RADV-specific NIR lowering passes
    ├── radv_nir_lower_vs_inputs.c
    └── radv_nir_apply_pipeline_layout.c
```

[Source](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/amd/vulkan)

---

## 4. Compiler Pipeline Comparison

The most substantive technical difference between the two drivers is their shader compilation
strategy.

### 4.1 AMDVLK: SPIR-V → LLPC → LLVM → ISA

```
SPIR-V binary
    │
    ▼
[LLPC Front-End]   ← Khronos SPIRV-LLVM translator
    │               ← lgc.call.* metadata injection
    ▼
LLVM IR (LGC dialect)
    │
    ▼
[LGC Middle-End]   ← BuilderReplayer pass
    │               ← Shader merging (LS-HS, NGG)
    │               ← SGPR/VGPR argument setup
    ▼
LLVM IR (amdgcn target)
    │
    ▼
[LLVM AMDGPU Back-End]  ← InstructionSelect, RegAlloc, Schedule
    │
    ▼
PAL ABI ELF (GCN/RDNA ISA + PAL metadata)
```

LLPC exploits the full LLVM optimisation pipeline: inlining, vectorisation, loop transformations,
GVN, LICM, and SCCP all run before instruction selection. For compute-heavy workloads with
complex arithmetic, this can produce high-quality machine code. The trade-off is compilation time:
the LLVM middle-end passes are not tuned for interactive shader compilation latency.

LLPC also supports partial pipeline caching. The pipeline state is split into vertex-processing and
fragment-processing halves; each half is hashed independently, allowing cache hits to skip
recompiling one half when only the other changes. This is similar in concept to Mesa's Graphics
Pipeline Library extension support but implemented inside LLPC rather than at the Vulkan API layer.

### 4.2 RADV: SPIR-V → NIR → ACO → ISA

```
SPIR-V binary
    │
    ▼
[spirv_to_nir()]   ← Mesa shared SPIR-V translator
    │
    ▼
NIR (SSA)
    │
    ▼
[radv_optimize_nir()]   ← ~15 optimisation passes
    │                    ← AMD-specific lowering
    ▼
NIR (lowered, hardware-specialised)
    │
    ▼
[aco::select_program()]  ← NIR → ACO IR
    │
    ▼
ACO IR
    │
    ▼
[ACO RegAlloc + Scheduler]
    │
    ▼
GCN/RDNA ISA binary (in radv_shader struct)
```

ACO was architected for latency above all else. Its register allocator and scheduler were written
specifically for AMD GCN and RDNA constraints rather than being general-purpose. The result is that
ACO compiles shaders significantly faster than LLPC's LLVM path, an advantage that matters
enormously in games where hundreds of pipelines may be compiled during load screens or at runtime.
[Source](https://www.phoronix.com/review/radv-aco-llvm)

### 4.3 Compilation Speed and Shader Quality

| Dimension | AMDVLK (LLPC) | RADV (ACO) |
|-----------|---------------|------------|
| Compile latency | High (full LLVM opt pipeline) | Low (purpose-built for speed) |
| Register allocation quality | LLVM's greedy RA | Custom linear-scan tuned for GCN/RDNA |
| Arithmetic-heavy compute | Strong (LLVM vectoriser, LICM) | Competitive; some gaps in edge cases |
| Graphics shader quality | Competitive | Generally matches or exceeds |
| Wave32 / Wave64 flexibility | Supported | Supported; dynamic per-workload |
| Partial pipeline caching | Yes (LGC hash split) | Yes (VK_EXT_graphics_pipeline_library) |

### 4.4 Pipeline Cache Behaviour

Both drivers implement `VkPipelineCache` serialisation to disk. AMDVLK stores PAL ABI ELF objects
keyed by a hash of the pipeline state. RADV stores ACO binary blobs inside Mesa's shader cache
database, with the cache location controlled by Mesa environment variables:

```bash
# RADV shader cache locations (in priority order)
# 1. MESA_SHADER_CACHE_DIR if set:
#    $MESA_SHADER_CACHE_DIR/mesa_shader_cache_sf/
# 2. XDG_CACHE_HOME if set:
#    $XDG_CACHE_HOME/mesa_shader_cache_sf/
# 3. Default:
#    ~/.cache/mesa_shader_cache_sf/

# Inspect the Mesa-DB single-file cache (Mesa 22.3+)
ls -lh ~/.cache/mesa_shader_cache_sf/
# mesa_shader_cache_sf.idx  mesa_shader_cache_sf.lck  mesa_shader_cache_sf.cache

# Disable the on-disk cache (forces recompilation every run)
MESA_SHADER_CACHE_DISABLE=true ./application

# Limit cache size to 1 GB
MESA_SHADER_CACHE_MAX_SIZE=1G ./application
```

[Source](https://docs.mesa3d.org/envvars.html)

RADV additionally supports the `VK_EXT_graphics_pipeline_library` (GPL) extension, which allows
applications to pre-compile vertex-processing and fragment-processing halves of a pipeline
independently and link them later — analogous to LLPC's internal split but now exposed at the Vulkan
API layer. Mesa 23.1 enabled GPL by default for RADV. This extension became the cornerstone of
Valve's reduced pipeline stutter strategy for Steam games.
[Source](https://www.gamingonlinux.com/2023/05/mesa-graphics-drivers-23-1-0-out-now-with-radv-gpl-enabled/)

A practical consequence of RADV's cache architecture: if two applications use the same shader
source but different `VkDevice` configurations, they can share compiled shaders in the Mesa cache.
AMDVLK's pipeline cache is per-application (stored where `vkCreatePipelineCache` data is saved) and
not globally shared between processes.

---

## 5. Extension and Feature Support

### 5.1 Vulkan Conformance Level

Both drivers achieved Vulkan 1.4 conformance. AMDVLK 2025.Q2.1 (the final release, April 2025) was
built against Vulkan API headers 1.4.313 and tested against CTS 1.4.1.3.
[Source](https://github.com/GPUOpen-Drivers/AMDVLK/releases/tag/v-2025.Q2.1)

Mesa RADV passed Vulkan 1.4 conformance as part of the Mesa Linux conformance submission covering
AMD, Intel, Apple, and Qualcomm hardware. The Steam Machine platform (Valve's next-generation
Steam hardware) was listed on the Khronos Vulkan 1.4 conformant products database in May 2026 using
`RADV_NAVI33` with CTS 1.4.5.3, showing RADV's conformance continues to be maintained after
AMDVLK's discontinuation.
[Source](https://videocardz.com/newz/steam-machine-appears-on-official-vulkan-1-4-conformant-products-list)

### 5.2 Historical VK_AMD_* Extensions

AMD-vendor extensions (`VK_AMD_*`) were historically where the AMDVLK/PAL stack had an early-mover
advantage: AMD's Windows driver team exposed hardware features through vendor extensions before the
Khronos process produced KHR equivalents. Key examples:

| Extension | First shipping in | KHR successor |
|-----------|-------------------|---------------|
| `VK_AMD_rasterization_order` | AMDVLK (2018) | No direct KHR; folded into `VK_EXT_rasterization_order_attachment_access` |
| `VK_AMD_gcn_shader` | AMDVLK (2018) | Superseded by SPIR-V 1.4 capabilities |
| `VK_AMD_shader_ballot` | AMDVLK (2018); RADV (2017 patch series) | `VK_EXT_shader_subgroup_ballot` → Vulkan 1.1 core |
| `VK_AMD_shader_info` | AMDVLK (2018) | `VK_KHR_pipeline_executable_properties` |
| `VK_AMD_device_coherent_memory` | AMDVLK | No equivalent |
| `VK_AMD_anti_lag` | AMDVLK (2024) | No KHR equivalent as of 2025 |
| `VK_AMDX_shader_enqueue` | AMDVLK (experimental, 2023) | Under Khronos consideration |

`VK_AMD_shader_ballot` is an interesting case: RADV actually received patches for it in August 2017
(before AMDVLK's public release) because the extension spec was published and DOOM used it to
select AMD-optimised code paths. Both drivers supported it by 2018.
[Source](https://www.phoronix.com/news/RADV-VK_AMD_shader_ballot)

The `VK_AMD_gcn_shader` extension exposes GCN-specific GLSL built-ins: `cubeFaceIndexAMD`,
`cubeFaceCoordAMD` (for shadow cube-map sampling), `timeAMD` (a 64-bit hardware clock counter), and
`fragmentMaskFetchAMD` / `fragmentFetchAMD` (for EQAA sample pattern access). These were primarily
useful for porting GCN-specific D3D extensions to Vulkan and have been largely superseded by
portable Vulkan 1.2+ extensions.

`VK_AMD_anti_lag` was registered in the Vulkan specification as extension number 476 and published
with the Vulkan 1.3.291 header in July 2024. The extension works by inserting a fine-grained
pacing mechanism that prevents the CPU from submitting frames too far ahead of the GPU, reducing
the latency between player input and the resulting frame reaching the display. The `antiLagUpdate()`
call hints to the driver where in the application's frame loop the input event was sampled, allowing
the driver to time GPU submission to minimise queuing delay.
[Source](https://www.phoronix.com/news/Vulkan-1.3.291-Released)

AMDVLK shipped `VK_AMD_anti_lag` as an AMD driver feature in 2024. RADV later added support.
**Note: needs verification for the exact RADV landing date for `VK_AMD_anti_lag`.**

`VK_AMDX_shader_enqueue` is an experimental extension for work graph pipelines — a GPU-driven
dispatch model where shaders can enqueue additional work for other shaders without returning to the
CPU. AMD added mesh nodes to this extension in later revisions. Work graphs are a significant
architectural direction for GPU-driven rendering and are under active standardisation discussion in
Khronos. Both AMDVLK and RADV received experimental implementations.

### 5.3 Ray Tracing

AMDVLK added ray tracing support (`VK_KHR_ray_tracing_pipeline`, `VK_KHR_acceleration_structure`)
in release 2022.Q3.4 (September 2022), targeting RDNA2 GPUs.
[Source](https://www.phoronix.com/news/AMDVLK-2022.Q3.4-Released)

RADV's ray tracing implementation was enabled by default in Mesa 23.2 (Q3 2023), following
significant investment from Valve's Linux driver team. Mesa 26.0 (February 2026) brought major
improvements: Wave32 execution for ray tracing shaders on RDNA3 and RDNA4, delivering 50–75%
performance improvements in titles like Silent Hill 2 Remake and Desordre.
[Source](https://www.gamingonlinux.com/2026/02/mesa-26-0-is-out-bringing-ray-tracing-performance-improvements-for-amd-radv/)

By late 2025, at the point of AMDVLK's discontinuation, RADV's ray tracing performance was largely
on par with or ahead of AMDVLK for most workloads.

### 5.4 Mesh Shaders and Descriptor Buffers

`VK_EXT_mesh_shader` was added to AMDVLK in 2023.Q2.1 alongside Mesh Shading for RDNA3 GPUs.
[Source](https://www.phoronix.com/news/AMDVLK-2023.Q2.1)

`VK_EXT_descriptor_buffer` (which moves descriptor data from driver-managed pools into
application-managed buffers, reducing CPU overhead for descriptor updates) was also added in AMDVLK
2023.Q2.1. RADV's implementation of `VK_EXT_descriptor_buffer` followed in Mesa 23.x.

Both extensions are present in both drivers. Their performance profiles differ: AMDVLK's descriptor
buffer implementation translates to PAL's internal descriptor format, while RADV writes descriptors
directly in the hardware format, potentially with lower CPU overhead.

### 5.5 Video Decode

`VK_KHR_video_decode_queue` support in AMDVLK was added for H.264 and H.265 decoding on supported
RDNA hardware. **Note: needs verification for the exact AMDVLK release that added this.**

RADV's video decode support (`VK_KHR_video_decode_h264` and `VK_KHR_video_decode_h265`) has been
maturing in Mesa since 2022 as part of the broader Vulkan Video ecosystem. For production video
decode on AMD Linux, most deployments use VA-API through Mesa's RadeonSI or RADV paths rather than
Vulkan Video directly. (See Chapter 35 for the VA-API stack on AMD.)

---

## 6. Distribution Packaging and Driver Selection

### 6.1 Default Drivers by Distro

RADV is the default AMD Vulkan ICD on all major Linux distributions because it ships as part of
the `mesa-vulkan-drivers` package (or equivalent), which is installed by default with any graphical
session:

| Distribution | Default AMD Vulkan ICD | AMDVLK packaging |
|--------------|------------------------|------------------|
| Ubuntu 22.04 / 24.04 | Mesa RADV (`mesa-vulkan-drivers`) | `amdvlk` (universe/optional) |
| Fedora 40 / 41 | Mesa RADV (`mesa-vulkan-drivers`) | Not in default repos; third-party |
| Arch Linux | Mesa RADV (`vulkan-radeon`) | `amdvlk` in `extra` repo |
| openSUSE Tumbleweed | Mesa RADV | `amdvlk` available |
| Steam Runtime (Deck) | Mesa RADV | Not installed |
| Radeon Software for Linux ≥ 25.20 | Mesa RADV (officially packaged by AMD) | No longer included |

The Steam Deck runs Mesa RADV. Valve has been a major contributor to RADV since 2019, and ACO was
developed primarily by Valve engineers specifically to improve RADV's compile-time performance for
gaming workloads. [Source](https://steamcommunity.com/app/1675200/discussions/0/3424446023726315215/)

Radeon Software for Linux 25.20 marked a watershed: AMD's own packaged Linux driver stack switched
from including AMDVLK to officially packaging and supporting Mesa RADV.
[Source](https://www.phoronix.com/news/Radeon-Software-25.20.3-Linux)

### 6.2 Selecting the Active Driver

When multiple Vulkan ICDs are installed, the loader reads all `.json` files from the ICD search
path and enumerates all discovered physical devices. Several environment variables control selection:

```bash
# Show all available Vulkan ICDs and active driver name
vulkaninfo 2>/dev/null | grep -E "driverName|driverID|apiVersion" | head -20
```

Example output with both RADV and AMDVLK installed:

```
        driverName     = radv
        driverID       = DRIVER_ID_MESA_RADV
        ...
        driverName     = AMD open-source driver
        driverID       = DRIVER_ID_AMD_OPEN_SOURCE
```

To force a specific ICD for a single application, set `VK_ICD_FILENAMES` (or the newer
`VK_DRIVER_FILES` for Vulkan 1.3.234+) to the path of the chosen ICD JSON file:

```bash
# Force RADV (Mesa)
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/radeon_icd.x86_64.json \
    vkmark --benchmark triangle

# Force AMDVLK (when installed)
VK_ICD_FILENAMES=/etc/vulkan/icd.d/amd_icd64.json \
    vkmark --benchmark triangle
```

On some Arch Linux setups where AMDVLK is installed, the `AMD_VULKAN_ICD` environment variable
is read by AMDVLK's own loader integration to select between RADV and AMDVLK when both are present:

```bash
# Select AMDVLK via AMD's own environment variable (Arch-specific convention)
AMD_VULKAN_ICD=AMDVLK some_application

# Select RADV
AMD_VULKAN_ICD=RADV some_application
```

`AMD_VULKAN_ICD` is a convention adopted by Arch Linux packaging and is not a Vulkan loader
standard; `VK_ICD_FILENAMES` is the portable mechanism.

The RADV ICD JSON installed by Mesa names itself:

```json
// /usr/share/vulkan/icd.d/radeon_icd.x86_64.json
{
    "file_format_version": "1.0.0",
    "ICD": {
        "library_path": "/usr/lib/x86_64-linux-gnu/libvulkan_radeon.so",
        "api_version": "1.4.0"
    }
}
```

### 6.3 Coexistence: Running Both Drivers

When developing or benchmarking, it is straightforward to install both drivers and switch between
them per-process. On Arch Linux, both packages are available:

```bash
# Install both drivers (Arch Linux)
sudo pacman -S vulkan-radeon   # Mesa RADV (part of mesa package)
sudo pacman -S amdvlk          # AMD official (now archived)

# Verify both ICDs are visible
vulkaninfo --summary
```

At the Vulkan API level, applications can identify the active driver at runtime using
`VkPhysicalDeviceDriverProperties` (Vulkan 1.2+):

```c
// Programmatically detect the active Vulkan driver
VkPhysicalDeviceDriverProperties driverProps = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_DRIVER_PROPERTIES,
};
VkPhysicalDeviceProperties2 props2 = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_PROPERTIES_2,
    .pNext = &driverProps,
};
vkGetPhysicalDeviceProperties2(physDev, &props2);

// For RADV:   driverProps.driverID == VK_DRIVER_ID_MESA_RADV
//             driverProps.driverName == "radv"
// For AMDVLK: driverProps.driverID == VK_DRIVER_ID_AMD_OPEN_SOURCE
//             driverProps.driverName == "AMD open-source driver"

printf("Driver: %s (%s)\n",
       driverProps.driverName,
       driverProps.driverInfo);   // Mesa version string or AMDVLK release tag
```

This pattern is useful for applications that need driver-specific workarounds or want to log
diagnostic information in crash reports.

For integration tests or CI pipelines that need to exercise both drivers, a common pattern is:

```bash
#!/bin/bash
# Run a Vulkan test suite against both AMD ICDs
RADV_ICD=/usr/share/vulkan/icd.d/radeon_icd.x86_64.json
AMDVLK_ICD=/etc/vulkan/icd.d/amd_icd64.json

echo "=== Testing with RADV ==="
VK_ICD_FILENAMES=$RADV_ICD ./run_vk_tests.sh

echo "=== Testing with AMDVLK ==="
VK_ICD_FILENAMES=$AMDVLK_ICD ./run_vk_tests.sh
```

---

## 7. Performance Comparison

### 7.1 Gaming Workloads

Benchmarking by Phoronix over 2024–2025 consistently showed that RADV matched or outperformed
AMDVLK across a broad range of gaming workloads. A December 2025 final comparative benchmark — run
just after AMD's AMDVLK discontinuation announcement — used the Radeon RX 9070 and RX 9070 XT as
the last RDNA4 GPUs supported by AMDVLK, comparing AMDVLK 2025.Q2.1 against Mesa 26.0-devel RADV.
The overall conclusion: RADV wins in the majority of tests, especially in rasterisation-heavy
gaming.
[Source](https://www.phoronix.com/review/radeon-radv-amdvlk-final)

An earlier mid-2025 Phoronix benchmark compared RADV and AMDVLK on the AMD Strix Halo platform
(Radeon 8060S integrated graphics, RDNA3.5 architecture) using the HP ZBook Ultra G1a with a Ryzen
AI Max+ PRO 395. The conclusion was clear: in the majority of benchmarks, the Mesa RADV driver was
faster than AMDVLK, even when using the Ubuntu 24.04.2 LTS default RADV rather than a development
build. The Strix Halo results also showed RADV ahead on Vulkan LLM inference (via llama.cpp's
Vulkan compute backend), with RADV achieving approximately 1097 tokens/s prompt processing against
AMDVLK's 777 tokens/s.
[Source](https://www.phoronix.com/forums/forum/linux-graphics-x-org-drivers/open-source-amd-linux/1556720-radv-vs-amdvlk-driver-performance-for-strix-halo-radeon-8060s-graphics)

Key factors in RADV's gaming advantage:

1. **ACO compile speed**: In games with runtime shader compilation (no shader pre-cache), RADV's
   faster ACO compiler produces less stutter. AMDVLK's LLPC path can block the CPU for longer on
   complex pipeline compilations. This difference is most visible in open-world games and titles
   using Unreal Engine 5, where pipeline creation occurs during traversal of new areas.
2. **Mesa integration**: RADV shares memory management, format handling, and synchronisation
   primitives with Mesa's broader driver ecosystem. Fixes for edge cases in Vulkan memory barriers
   and format compatibility benefit from the large community of Mesa contributors.
3. **Valve investment**: Valve's SteamOS team regularly profiles games on RADV and contributes
   game-specific workarounds and optimisations. Samuel Pitoiset (Valve), David Airlie (Red Hat),
   Timur Kristóf and Friedrich Vock (Valve) have been prolific contributors. AMDVLK did not have
   an equivalent consumer-facing tuning effort on Linux.
4. **VK_EXT_graphics_pipeline_library**: RADV's GPL support (enabled by default in Mesa 23.1)
   allows game engines to pre-link pipeline halves asynchronously, further reducing the stall that
   occurs when a new pipeline state is encountered mid-frame.
   [Source](https://www.gamingonlinux.com/2023/05/mesa-graphics-drivers-23-1-0-out-now-with-radv-gpl-enabled/)

### 7.2 Ray Tracing Workloads

Ray tracing was the last domain where AMDVLK held a meaningful advantage over RADV. AMDVLK's
GPURT library implements BVH traversal in HLSL compiled offline, with a more mature traversal loop
at the time. The Phoronix final benchmarks noted that "in some benchmarks the AMDVLK Vulkan
ray-tracing is still faster than Mesa RADV" but that the gap had narrowed substantially.
[Source](https://x.com/phoronix/status/2004213697259160018)

Mesa 26.0 (February 2026) addressed the remaining gap with Wave32 ray-tracing shaders on RDNA3 and
RDNA4, delivering 50–75% performance improvements over Mesa 25.x in ray-traced titles. With this
release, RADV's ray tracing performance moved to parity with or beyond AMDVLK's final state.
[Source](https://www.gamingonlinux.com/2026/02/mesa-26-0-is-out-bringing-ray-tracing-performance-improvements-for-amd-radv/)

### 7.3 Compute Workloads

For Vulkan compute (not ROCm), AMDVLK's LLVM-based compilation produced high-quality code for
compute-heavy kernels thanks to LLVM's vectorisation and loop optimisation passes. RADV's ACO was
originally optimised for graphics shader patterns and caught up to LLPC on compute quality over
subsequent Mesa releases.

A notable edge case is LLM inference via llama.cpp's Vulkan compute backend. The Strix Halo
benchmarks mentioned above showed RADV outperforming AMDVLK even on this compute-heavy workload,
suggesting that ACO's code quality for the matrix-multiply compute shaders used by llama.cpp had
surpassed LLPC by 2025.

ROCm (AMD's HPC/AI compute platform) uses neither AMDVLK nor RADV; it has its own kernel driver
path (`amdkfd`) and its own LLVM-based compiler pipeline (HIP Clang). Vulkan compute via RADV is
a separate path from ROCm used for graphics-adjacent compute tasks such as post-processing and
machine learning inference integrated into a Vulkan render graph. When choosing between ROCm and
Vulkan compute for a new AMD Linux application, the answer depends on the use case:

- **ROCm (HIP)**: For large-scale HPC, deep learning training, and inference with large models.
  ROCm has better interoperability with PyTorch, TensorFlow, and the broader ML framework ecosystem.
- **Vulkan compute via RADV**: For graphics-adjacent workloads, game post-processing effects,
  smaller neural network inference that needs to share resources with a Vulkan render pass, or any
  scenario where the application already has a Vulkan device and wants to avoid maintaining both a
  Vulkan and a ROCm context.

---

## 8. The End of AMDVLK: AMD Consolidates on RADV

The story of AMD's two Vulkan drivers ended definitively in 2025. The consolidation happened in
two steps:

**May 2025**: AMD announced that its packaged "Radeon Software for Linux" release (version 25.20)
would adopt Mesa RADV as the officially supported Vulkan driver, replacing the AMDVLK binary it had
previously bundled. The proprietary OpenGL and Vulkan driver components were dropped from the
package in favour of exclusively open-source Mesa components.
[Source](https://www.phoronix.com/news/Radeon-Software-25.20.3-Linux)

**September 15, 2025**: AMD formally announced on the AMDVLK GitHub Discussions page:

> "AMD is unifying its Linux Vulkan driver strategy and has decided to discontinue the AMDVLK
> open-source project, throwing our full support behind the RADV driver as the officially supported
> open-source Vulkan driver for Radeon graphics adapters. This consolidation allows us to focus our
> resources on a single, high-performance codebase that benefits from the incredible work of the
> entire open-source community."
> — AMD, [AMDVLK Discontinuation Announcement](https://github.com/GPUOpen-Drivers/AMDVLK/discussions/416)

AMD simultaneously archived the PAL, GPURT, and related repositories on GitHub. The final AMDVLK
release was **v-2025.Q2.1** (April 30, 2025), which supported:

- Radeon RX 9070 Series (RDNA4/GFX12 — the final generation supported)
- Radeon RX 7900/7800/7700/7600 Series (RDNA3/GFX11)
- Radeon RX 6900/6800/6700/6600/6500 Series (RDNA2/GFX10.3)
- Radeon RX 5700/5600/5500 Series (RDNA1/GFX10)
- Radeon Pro W5700/W5500 Series

Pre-GCN10 GPUs (Polaris, Vega, GCN 1–5) had already been dropped from AMDVLK in 2023 (final
version with GCN support: v-2023.Q3.3). RADV continues to support these architectures all the way
back to GFX6.
[Source](https://github.com/GPUOpen-Drivers/AMDVLK/blob/master/README.md)

The XGL, PAL, and LLPC repositories remain publicly readable on GitHub under the
[GPUOpen-Drivers organisation](https://github.com/GPUOpen-Drivers), but are archived and receive no
new commits. AMDVLK 2025.Q2.1 packages remain available in Arch Linux's `extra` repository and
from GitHub Releases, but no further GPU support or bug fixes will be added.

---

## 9. Stability, Bug Report Paths, and CTS

### RADV: Mesa GitLab

RADV bugs are filed at [gitlab.freedesktop.org/mesa/mesa](https://gitlab.freedesktop.org/mesa/mesa)
with the `~Driver/AMD/radv` label. The team of active RADV maintainers (including contributors from
Valve, Red Hat, Google, and independent developers) typically responds within days to regression
reports on recent hardware. RADV has a CI pipeline running the Vulkan CTS on multiple AMD GPU
generations for every merge request, making regressions relatively uncommon.

The bug report process for RADV is identical to Mesa broadly:

```bash
# Get a useful RADV debug trace for a bug report
RADV_DEBUG=errors,shaders \
MESA_LOG_FILE=/tmp/radv_debug.log \
    some_application 2>&1 | tee /tmp/app_output.log

# Collect system information for the bug report
vulkaninfo --summary 2>/dev/null | grep -E "driverName|driverVersion|apiVersion|deviceName"
uname -r
mesa --version 2>/dev/null || vulkaninfo 2>/dev/null | grep "Vulkan Instance Version"

# Force RADV to use the LLVM backend (for isolating ACO-specific bugs)
RADV_DEBUG=llvm ./repro_application

# Common RADV_DEBUG flags:
# errors          — report validation errors to stderr
# shaders         — dump compiled shaders to /tmp/
# spirv           — dump SPIR-V input for each shader
# nir             — dump NIR before/after optimisations
# aco             — dump ACO IR at each stage
# nodcc           — disable Delta Color Compression (isolates DCC bugs)
# nomemorycache   — disable pipeline binary cache (forces recompilation)
```

### AMDVLK: GitHub Issues (Archived)

AMDVLK bug reports were filed at [github.com/GPUOpen-Drivers/AMDVLK/issues](https://github.com/GPUOpen-Drivers/AMDVLK/issues).
AMD engineers engaged with driver-level bugs. With the project archived in September 2025, the
issue tracker remains readable but new issues will not receive AMD responses. Users experiencing
bugs in AMDVLK are encouraged to migrate to RADV and file there if the issue reproduces.

### CTS Status

Both drivers achieved Vulkan 1.4 conformance before AMDVLK's discontinuation. RADV's ongoing
CTS testing infrastructure — integrated into Mesa's GitLab CI and run against actual hardware via
the freedesktop.org CI farm — means CTS failures are caught and fixed on a per-merge-request basis.
AMDVLK's final CTS pass was against 1.4.1.3; the Khronos conformance database entry for AMDVLK
reflects the last passing test run before archival.

### Steam Deck

The Steam Deck ships Mesa RADV. It does not include AMDVLK. Valve's SteamOS is built on a Mesa
stack, and Valve engineers have been active RADV contributors for years. Any game certified for
Steam Deck compatibility is implicitly tested against RADV.
[Source](https://docs.mesa3d.org/drivers/radv.html)

---

## 10. Choosing Between AMDVLK and RADV: Decision Guide

Given that AMDVLK is archived as of September 2025, the choice for new projects is straightforward:
**use RADV**. However, understanding when each was preferable helps with maintaining existing code
and evaluating edge cases.

### When RADV Is Correct (Always for New Work)

- **Shipping a game or application**: All major Linux distributions ship RADV by default; targeting
  RADV means users have the driver installed without additional steps.
- **Steam Deck**: Only RADV is available on SteamOS. Applications that need to run on Steam Deck
  must work with RADV.
- **Mesa ecosystem integrations**: Zink (OpenGL-on-Vulkan via Mesa RADV), DXVK (uses any Vulkan
  ICD but optimised against RADV on Linux), Proton's D3D12 path via vkd3d-proton, and NineOneOne
  (Direct3D 9 via Gallium Nine + RADV in some configurations) all assume Mesa driver availability.
- **Old GPU support**: RADV supports GFX6/GFX7 (GCN1 and GCN2 hardware from 2012–2014) which
  AMDVLK dropped in 2023. For legacy hardware, RADV is the only open-source Vulkan option.
- **Driver contribution**: RADV development happens on Mesa GitLab with a fast community review
  cycle. Contributing to RADV has immediate upstream impact.

### When AMDVLK Had Advantages (Historical, Now Archived)

- **VK_AMD_* extension parity with Windows**: Applications that relied on AMD vendor extensions
  (e.g., `VK_AMD_anti_lag`, `VK_AMD_device_coherent_memory`) saw them in AMDVLK before RADV
  implemented them.
- **Matching Windows AMD driver behaviour**: AMDVLK and AMD's Windows Vulkan driver share the XGL
  and LLPC codebase, so undefined-behaviour edge cases often had the same result on both platforms.
  Applications ported from Windows that relied on AMDVLK-specific behaviour could be harder to
  debug on RADV.
- **Enterprise / ISV certification**: Workstation applications that needed an AMD-supported driver
  could point to AMD's engagement with AMDVLK GitHub issues.

### FidelityFX SDK

AMD's FidelityFX SDK (FSR, CACAO, SPD, and related post-processing effects) is Vulkan-agnostic: it
takes a `VkDevice` and is not tied to either ICD. FSR 3 and FSR 4 work correctly on RADV. See
Chapter 72 for FidelityFX integration details.

### Decision Flowchart

```
Is AMDVLK archived? (Yes, as of September 2025)
  │
  └── Do you need a specific VK_AMD_* extension not yet in RADV?
        │
        ├── Yes → Check if RADV has since added it (likely yes by 2026)
        │          If still missing → file an issue on Mesa GitLab
        │
        └── No → Use RADV. Full stop.
```

---

### Strategic Outlook: Why AMD Maintained Two Vulkan Drivers — and Which One Has Won

When AMD published the XGL and PAL source repositories on GitHub on 22 December 2017, the public rationale was a reference implementation: a Vulkan driver that matched the Windows AMD driver's code path exactly, built on the same cross-platform PAL layer used for DirectX 12 and ROCm. The unstated motivations were more layered. AMD's Windows driver team had been developing LLPC and PAL for years; releasing them as AMDVLK cost relatively little because the Linux port of PAL already existed internally. Publishing the code also gave AMD a hedge against RADV: if the Mesa community driver proved unstable or incomplete, AMD's own ICD provided a fallback that enterprise ISVs and workstation users could cite in support contracts. AMDVLK functioned simultaneously as an insurance policy, a reference against which to compare RADV behaviour, and an internal validation tool — AMD's own developers used it to check whether bugs were in the kernel driver, in XGL, or in LLPC before engaging the Mesa mailing list.

RADV won for reasons that were architectural and organisational rather than merely technical. The ACO compiler, contributed primarily by Valve engineers and merged in Mesa 20.1, was designed from first principles for AMD GCN and RDNA ISAs rather than being an adaptation of a general-purpose compiler. Its linear-scan register allocator and purpose-built instruction scheduler produced faster compile times than LLPC's full LLVM optimisation pipeline — an advantage that compounds in games where hundreds of pipeline states are compiled at runtime. LLPC's strength was arithmetic throughput for compute-heavy kernels; ACO caught up on that axis across successive Mesa releases while maintaining its compile-latency lead. Critically, Valve's investment in the Steam Deck foreclosed the outcome: SteamOS ships Mesa RADV and nothing else. Every game certified for Deck is tested on RADV. That consumer-facing validation loop — with game-specific workarounds landing in Mesa within days of a report — was something AMDVLK never had. Community momentum compounded the effect: contributors from Valve, Red Hat, Google, and independent developers submit RADV merge requests that are reviewed against a CI pipeline running the Vulkan CTS on actual hardware. AMDVLK's GitHub issue tracker was engaged but small.

The September 2025 archival is best read as AMD completing a strategic retreat that had been visible since May 2025, when Radeon Software for Linux 25.20 dropped AMDVLK in favour of officially packaging Mesa RADV. The announcement that AMD was "throwing full support behind RADV" [Source](https://github.com/GPUOpen-Drivers/AMDVLK/discussions/416) is precise: AMD is not leading RADV development, it is contributing to it. AMD engineers now submit patches to Mesa GitLab alongside Valve and Red Hat engineers, operating under the same review process as any external contributor.

What remains proprietary — and what the AMDVLK story should not obscure — is AMD's actual competitive moat on Linux: the ROCm compute stack. ROCm's HIP Clang compiler, `amdkfd` kernel interface, and machine learning libraries (MIOpen, rocBLAS, hipBLAS) are maintained separately and are not open in the same way Mesa is. The `amdkfd` driver and the HIP runtime are AMD's differentiator in the HPC and AI training market, where PyTorch and TensorFlow backends depend on ROCm support. AMDVLK was never part of that moat — it was a Vulkan graphics driver, and Vulkan graphics on Linux was already well-served by the Mesa ecosystem AMD did not control.

The forward trajectory is clear. RADV is the unambiguous primary path for Vulkan on AMD Linux hardware. AMD engineers contributing patches upstream is structurally healthier than a parallel driver: fixes for hardware errata, new hardware enablement for RDNA4 and beyond, and conformance against new Vulkan extension CTS tests all land in a single codebase that distributions package and users run. The XGL and LLPC source repositories remain archived and readable, and LLPC is expected to persist as an offline shader compilation tool for ROCm's `amdllpc` binary. The PAL abstractions that once seemed like a strength — shared correctness fixes across Windows DX12, Linux Vulkan, and OpenCL — are now a legacy concern. AMD's open-source Vulkan story is, from 2025 onward, Mesa's story.

---

## 11. Integrations

This chapter connects to several others in the book:

- **Chapter 143 (RADV Internals)**: The deep dive into RADV's ACO compiler, winsys abstraction,
  descriptor handling, ray tracing BVH management, and debugging environment. The present chapter
  provides the comparative context; Chapter 143 provides the implementation detail for RADV
  specifically.

- **Chapter 72 (AMD FidelityFX SDK)**: FidelityFX works transparently on top of RADV. FSR 3
  and FSR 4 performance data is collected on RADV hardware. Cross-reference Chapter 72 for
  FidelityFX integration patterns that are agnostic to the choice of AMD Vulkan ICD.

- **Chapter 5 / Chapter 31 (amdgpu kernel driver)**: Both AMDVLK and RADV submit work to the
  `amdgpu` DRM kernel module — PAL's Linux KMD layer issues raw ioctls; RADV uses `libdrm_amdgpu`.
  The kernel driver's command submission, fence management, and memory allocator are shared by both.

- **Chapter 15 (ACO Compiler)**: ACO is the heart of RADV's performance advantage over LLPC.
  Chapter 15 covers ACO's register allocator, instruction scheduler, and ISA emitter in depth —
  the architectural decisions that made RADV's compile-time superior to LLPC.

- **Chapter 9 (Mesa Architecture)**: RADV builds on Mesa's Vulkan common layer (`src/vulkan/`),
  shared NIR infrastructure, and winsys abstraction. Chapter 9 situates RADV within the broader
  Mesa driver architecture.

- **Chapter 35 (VA-API on AMD)**: For video decode on AMD Linux, the VA-API path via Mesa's
  RadeonSI and RADV is the standard approach. Vulkan Video (`VK_KHR_video_decode_queue`) is an
  alternative covered in this chapter's Section 5.5 and in Chapter 35.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*

## Roadmap

### Near-term (6–12 months)
- **RADV user-queue enablement for RDNA4**: The experimental user-queue submission path (`amdgpu_userq`), which bypasses the kernel CS ioctl and reduces dispatch latency, is being finalised for RDNA4 (GFX12) hardware; Mesa MRs tracking this work are expected to land in Mesa 26.x or 27.0.
- **VK_AMDX_shader_enqueue stabilisation**: AMD and Valve are collaborating to refine the work-graph pipeline extension beyond its experimental status; a non-`AMDX` KHR proposal is under Khronos Working Group discussion, with RADV likely to be the reference Linux implementation.
- **Vulkan Video encode on RADV**: H.264 and H.265 encode support via `VK_KHR_video_encode_queue` is in active development in Mesa; patches have been posted to the Mesa mailing list targeting RDNA2+ hardware, complementing the existing decode path.
- **ACO RDNA4 (GFX12) optimisation**: ACO's instruction scheduler and register allocator are being tuned for RDNA4's updated wave-front model and new dual-issue compute execution units, with profiling-driven patches expected throughout 2026.

### Medium-term (1–3 years)
- **Full ACO parity with LLPC on compute-heavy workloads**: As LLM inference via Vulkan compute grows (llama.cpp, ExecuTorch Vulkan backend), ACO's matrix-multiply and reduction kernel quality is being systematically improved; the medium-term goal is to match or exceed what LLPC achieved on compute-intensive shaders.
- **Sparse resources and opacity micromap (OMM) in RADV**: `VK_EXT_opacity_micromap` (for ray tracing against masked geometry, e.g., foliage) and improved `VK_EXT_image_sliced_view_of_3d` / sparse image support are on the RADV backlog following AMDVLK's archival; AMD's hardware supports these features and driver-level work is expected.
- **XGL/LLPC code reuse in downstream tools**: Although AMDVLK itself is archived, AMD has signalled that LLPC will live on as a compiler component for ROCm offline shader compilation (`amdllpc`) and the AMDGPU back-end in LLVM; further consolidation between the Mesa NIR/ACO path and LLPC's middle-end (LGC) remains a long-discussed possibility.
- **Descriptor buffer and bindless improvements**: `VK_EXT_descriptor_buffer` landed in both drivers; the next phase — full bindless rendering with `VK_EXT_device_generated_commands` — is under active development in RADV, following Vulkan 1.4 promotion of this extension.

### Long-term
- **Single AMD open-source driver for all APIs**: With AMDVLK archived and ROCm's HIP compiler increasingly sharing infrastructure with Mesa's LLVM/ACO path, AMD's stated long-term direction is a unified open-source software stack where Vulkan, OpenGL (RadeonSI), OpenCL (Clover/rusticl), and HIP share front-ends over a common ACO or LLVM back-end.
- **Ray tracing hardware specialisation on future RDNA architectures**: RDNA5 and beyond are expected to introduce dedicated ray tracing execution units; RADV's BVH builder (currently software-based, inherited from GPURT design concepts) will need hardware acceleration hooks analogous to what GPURT provided for AMDVLK, likely via new `amdgpu` kernel uAPI and RADV pipeline extensions.
- **Convergence of Vulkan and ROCm dispatch paths**: The `amdgpu` kernel module already serves both graphics (`amdgpu`) and compute (`amdkfd`) clients; long-horizon work targets a unified queue model where RADV Vulkan compute and ROCm HIP kernels can share command queues and memory residency management without context switches between the two scheduler paths.
