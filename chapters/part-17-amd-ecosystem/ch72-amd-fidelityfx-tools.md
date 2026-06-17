# Chapter 72: AMD FidelityFX SDK and Radeon Developer Tools

**Target audiences**: Graphics application developers integrating AMD upscaling and post-processing
effects on Linux; systems developers profiling and debugging Vulkan workloads on AMD RDNA
hardware; browser and game engine engineers working with the open AMD GPUOpen toolchain.

---

## Table of Contents

1. [AMD Developer Ecosystem Overview](#1-amd-developer-ecosystem-overview)
2. [FidelityFX SDK Architecture](#2-fidelityfx-sdk-architecture)
3. [FSR 4 — Neural Upscaling on RDNA 4](#3-fsr-4--neural-upscaling-on-rdna-4)
4. [Other FidelityFX Effects](#4-other-fidelityfx-effects)
5. [AMF — Advanced Media Framework](#5-amf--advanced-media-framework)
6. [Radeon GPU Profiler (RGP)](#6-radeon-gpu-profiler-rgp)
7. [Radeon Memory Visualizer (RMV)](#7-radeon-memory-visualizer-rmv)
8. [RenderDoc — Cross-Vendor Frame Debugger](#8-renderdoc--cross-vendor-frame-debugger)
9. [Integration with the Open-Source Stack](#9-integration-with-the-open-source-stack)
10. [Integrations](#10-integrations)

---

## 1. AMD Developer Ecosystem Overview

AMD publishes the bulk of its developer-facing SDK work under the
[GPUOpen](https://gpuopen.com) initiative, a programme launched in 2016 to deliver
open-source GPU libraries, effects, and tools under permissive licences (predominantly MIT).
This stands in deliberate contrast to NVIDIA's NGX/DLSS and CUDA ecosystems, which mix
proprietary binaries with open headers. GPUOpen's three main pillars are:

- **FidelityFX SDK** — a library of image-quality and rendering-technique effects (upscaling,
  sharpening, global illumination, screen-space reflections, and more) that run on any
  DirectX 12 or Vulkan GPU, with RDNA-optimised shader paths for AMD hardware.
  [Source](https://github.com/GPUOpen-LibrariesAndSDKs/FidelityFX-SDK)

- **Advanced Media Framework (AMF)** — AMD's hardware video encode/decode API, historically
  backed by the proprietary `amf-amdgpu-pro` component on Linux, and since driver 25.20
  transitioning to an open stack that delegates to VA-API and Mesa Multimedia.
  [Source](https://github.com/GPUOpen-LibrariesAndSDKs/AMF)

- **Radeon Developer Tools** — a suite of profiling, memory-visualisation, and debugging tools:
  Radeon GPU Profiler (RGP), Radeon Memory Visualizer (RMV), and Radeon Developer Panel
  (RDP), all available for Linux (Ubuntu 24.04+, Vulkan-only, Vulkan and OpenCL/HIP).
  [Source](https://gpuopen.com/tools/)

Additionally, **RenderDoc** — while technically cross-vendor and independent — was created by
Baldur Karlsson with significant AMD engagement and is the de-facto open-source frame debugger
for Vulkan, Direct3D, and OpenGL on Linux. [Source](https://renderdoc.org)

This chapter covers the public APIs and internal architecture of each of these tools, placing
them in the context of the Mesa/RADV driver stack described in Ch15 (ACO) and Ch18 (RADV).

---

## 2. FidelityFX SDK Architecture

### 2.1 Repository Layout

The FidelityFX SDK lives at
[github.com/GPUOpen-LibrariesAndSDKs/FidelityFX-SDK](https://github.com/GPUOpen-LibrariesAndSDKs/FidelityFX-SDK).
The repository separates cleanly into three layers that mirror the host/GPU/backend split in
the SDK design documentation:

```
FidelityFX-SDK/
├── sdk/                     # Core SDK — the only part that ships in games
│   ├── include/FidelityFX/  # Public C headers for each effect
│   ├── src/components/      # Effect implementation (host-side C++ and GPU shaders)
│   │   ├── fsr3upscaler/    # FSR 3/4 upscaling context
│   │   ├── cas/             # Contrast Adaptive Sharpening
│   │   ├── spd/             # Single Pass Downsampler
│   │   ├── sssr/            # Stochastic Screen Space Reflections
│   │   ├── brixelizer/      # Sparse voxel GI
│   │   └── ...
│   └── src/backends/        # DX12, Vulkan, GDK backend implementations
├── ffx-api/                 # New "Upgradable FSR API" (SDK 2.x, FSR 4)
│   └── include/ffx_api/     # ffx_upscale.h, ffx_framegeneration.h
├── samples/                 # Cauldron-based sample applications
├── tools/FidelityFX_SC/     # Offline shader compiler
└── docs/
```

[Source: FidelityFX-SDK SDK structure docs](https://github.com/AzagraMac/FidelityFX-SDK-FSR3/blob/master/docs/getting-started/sdk-structure.md)

Version 1.1.4 (the SDK 1.x series) bundles 20+ post-processing and rendering techniques.
FidelityFX SDK 2.0 introduces a separate "Upgradable API" (`ffx-api/`) specifically for
FSR 4 and future ML-based neural rendering technologies.
[Source](https://gpuopen.com/learn/amd-fidelityfx-sdk-2-0/)

### 2.2 The `FfxInterface` Abstraction

Every FidelityFX effect uses a single backend abstraction struct, `FfxInterface`, defined in
`sdk/include/FidelityFX/host/ffx_interface.h`. The struct carries a table of function
pointers that decouple effect logic from any specific graphics API:

```c
// sdk/include/FidelityFX/host/ffx_interface.h (simplified — function pointer names
// verified against the FfxInterface API reference; struct layout is illustrative)
typedef struct FfxInterface {
    // Version and capability queries
    FfxGetSDKVersionFunc            fpGetSDKVersion;
    FfxGetDeviceCapabilitiesFunc    fpGetDeviceCapabilities;
    FfxGetEffectGpuMemoryUsageFunc  fpGetEffectGpuMemoryUsage;

    // Backend context lifecycle
    FfxCreateBackendContextFunc     fpCreateBackendContext;
    FfxDestroyBackendContextFunc    fpDestroyBackendContext;

    // Resource management
    FfxCreateResourceFunc           fpCreateResource;
    FfxRegisterResourceFunc         fpRegisterResource;
    FfxGetResourceFunc              fpGetResource;
    FfxMapResourceFunc              fpMapResource;
    FfxUnmapResourceFunc            fpUnmapResource;
    FfxDestroyResourceFunc          fpDestroyResource;

    // Pipeline management (shader programs)
    FfxCreatePipelineFunc           fpCreatePipeline;
    FfxDestroyPipelineFunc          fpDestroyPipeline;

    // Command scheduling
    FfxScheduleGpuJobFunc           fpScheduleGpuJob;
    FfxExecuteGpuJobsFunc           fpExecuteGpuJobs;

    // Crash diagnostics (Breadcrumbs effect)
    FfxBreadcrumbsAllocBlockFunc    fpBreadcrumbsAllocBlock;
    FfxBreadcrumbsWriteFunc         fpBreadcrumbsWrite;

    void*   scratchBuffer;
    size_t  scratchBufferSize;
} FfxInterface;
```

[Source: FfxInterface API reference](https://gpuopen.com/manuals/fidelityfx_sdk/fidelityfx_sdk-group__ffxinterface/)

The `fpCreatePipeline` and `fpScheduleGpuJob` function pointers are the critical seam:
effects build an abstract job graph, and the backend translates those jobs to actual
`VkCmdDispatch` / `ID3D12GraphicsCommandList::Dispatch` calls. This design means all HLSL and
GLSL shader permutations can be compiled offline by the **FidelityFX Shader Compiler
(FFX-SC)**, packaged as blobs, and embedded in the SDK binary — eliminating all runtime
shader compilation overhead that plagues naive integrations.

### 2.3 Vulkan Backend Initialisation

The Vulkan backend ships in `sdk/src/backends/vk/`. Initialising it follows a
scratch-memory-then-interface pattern:

```c
// File: sdk/src/backends/vk/ffx_vk.cpp (simplified usage)
#include <FidelityFX/host/backends/vk/ffx_vk.h>

// 1. Query scratch memory needed for N simultaneous effect contexts
const size_t scratchSize = ffxGetScratchMemorySizeVK(physicalDevice, /*maxContexts=*/4);
void* scratch = malloc(scratchSize);

// 2. Populate the FfxInterface for the Vulkan backend
FfxInterface backendInterface;
ffxGetInterfaceVK(&backendInterface,
                  ffxGetDeviceVK(vkDevice),
                  scratch, scratchSize,
                  /*maxContexts=*/4);

// Native Vulkan types map to FidelityFX opaque handles:
//   VkDevice          → FfxDevice   (via ffxGetDeviceVK)
//   VkCommandBuffer   → FfxCommandList
//   VkBuffer/VkImage  → FfxResource
//   VkSwapchainKHR    → FfxSwapchain
```

[Source: Vulkan Backend API](https://gpuopen.com/manuals/fidelityfx_sdk/fidelityfx_sdk-group__vkbackend/)

After backend initialisation each effect follows a consistent three-phase lifecycle:
**Create** (allocate GPU resources, compile/load shader pipelines), **Dispatch** (record one
frame's compute work into the command list), and **Destroy** (free all GPU resources). This
pattern is identical across CAS, SPD, FSR, SSSR, Brixelizer, and every other technique in the
SDK. [Source: FidelityFX SDK 1.1.4 Getting Started](https://gpuopen.com/manuals/fidelityfx_sdk/)

### 2.4 SDK 2.x and the Upgradable API

Starting with FidelityFX SDK 2.0 (released alongside FSR 4 in early 2025), AMD introduced a
separate `ffx-api/` layer that wraps ML-based upscalers. Its key architectural properties are:

- **Driver-level upgradability**: the `amd_fidelityfx_loader.dll` / `.so` shim loads the
  actual `amd_fidelityfx_upscaler` and `amd_fidelityfx_framegeneration` libraries at runtime,
  so AMD Adrenalin driver releases can transparently upgrade FSR 4's neural model weights
  without game patches.
- **DirectX 12 only at launch**: the SDK 2.0 release explicitly lists no Vulkan support. The
  open SDK 1.1.x Vulkan backend remains the integration path for Linux Vulkan applications
  wishing to use FSR 3 temporal upscaling.
- **Separate DLLs per effect category**: `amd_fidelityfx_upscaler`, `amd_fidelityfx_framegeneration`.

[Source: AMD FidelityFX SDK 2.0 launch blog](https://gpuopen.com/learn/amd-fidelityfx-sdk-2-0/)

Note: on Linux, FSR 4 via the SDK 2.x Upgradable API requires either a Wine/Proton D3D12
translation layer or a future Vulkan backend. At the time of writing (June 2026), Vulkan
support in SDK 2.x is on AMD's roadmap but not yet shipped. Linux Vulkan games therefore use
FSR 3.1 via SDK 1.1.x.

---

## 3. FSR 4 — Neural Upscaling on RDNA 4

### 3.1 Evolution of AMD Super Resolution

AMD has shipped three distinct generations of super-resolution technology:

| Generation | Algorithm | Hardware | API struct |
|-----------|-----------|----------|------------|
| FSR 1 (spatial) | Edge-Adaptive Spatial Upsampling (EASU) + RCAS sharpening | Any GPU with shader support | `FfxFsr1Context` |
| FSR 2 / 3 (temporal) | Temporal reprojection + reactive mask + RCAS | Any GPU | `ffxCreateContextDescUpscale` |
| FSR 4 (neural) | ML inference on RDNA 4 Matrix Cores (WMMA) | RDNA 4 (RX 9000+) | `ffxCreateContextDescUpscale` + version wrapper |

**FSR 1** worked purely in screen space: the EASU pass used a 12-tap Catmull-Rom filter with
per-tap luma weighting to detect edges and sharpen them during upsampling, followed by
Robust Contrast Adaptive Sharpening (RCAS) as a post-process. No temporal information, no
motion vectors. The entire algorithm ran in a single compute dispatch, making it trivially
portable and cheap (well under 1 ms on any modern GPU).

**FSR 2/3** (temporal) requires motion vectors, depth, and per-frame jitter to accumulate
detail across multiple frames. The reprojection stage uses motion vectors to warp the previous
high-resolution accumulation buffer onto the current frame, a reactive mask reduces reliance on
historical data for translucent or VFX-heavy pixels, and RCAS sharpening runs as the final
pass. Internally, motion vectors are stored at 16-bit precision, so providing FP32 vectors
offers no additional benefit.
[Source: FSR 2 manual](https://gpuopen.com/manuals/fidelityfx_sdk2/techniques/super-resolution-temporal/)

**FSR 4** replaces the analytical temporal accumulation with a trained neural network that
uses AMD RDNA 4's **3rd-generation Matrix Cores** — dedicated matrix multiply-accumulate
hardware programmed via **WMMA** (Wave Matrix Multiply-Accumulate) ISA instructions. The
`__builtin_amdgcn_wmma_f32_16x16x16_f16_w32_gfx12` intrinsic (HLSL SM6 `wmma` ops) is the
low-level interface; FSR 4's shaders use HLSL CS_6_4 targeting this hardware.
[Source: Using Matrix Cores on AMD RDNA 4 — GPUOpen](https://gpuopen.com/learn/using_matrix_core_amd_rdna4/)
The network takes the same inputs as FSR 2/3 (color, depth, motion vectors, jitter) but
infers the high-resolution output through learned weights trained on large datasets of game
footage. FSR 4 falls back automatically to FSR 3.1 on pre-RDNA-4 hardware.
[Source: AMD FSR 4 GPUOpen](https://gpuopen.com/amd-fsr-upscaling/)

### 3.2 FSR 4 Input Requirements

| Resource | Resolution | Format | Notes |
|----------|------------|--------|-------|
| Color buffer | Render resolution | Application-defined | Must be **linear** colorspace; apply tonemapping *after* FSR |
| Depth buffer | Render resolution | R32_FLOAT | Inverted depth recommended (`FFX_UPSCALE_ENABLE_DEPTH_INVERTED`) |
| Motion vectors | Render resolution | RG16_FLOAT | Screen-space range [−width, −height] to [+width, +height] |
| Exposure (optional) | 1×1 | R32_FLOAT | Required unless `FFX_UPSCALE_ENABLE_AUTO_EXPOSURE` is set |
| Reactive mask | Render resolution | R8_UNORM | Optional in FSR 4; recommended for VFX with FSR 3 |

Camera jitter must use a Halton(2,3) sequence queried via `ffxQueryDescUpscaleGetJitterOffset`.
[Source: AMD FSR Upscaling 4.1.0 Manual](https://gpuopen.com/manuals/fidelityfx_sdk2/techniques/super-resolution-ml/)

### 3.3 FSR 4 API

FSR 4 ships through the `ffx-api` layer, using C99-compatible structs with linked-list
extension headers. The key integration points are:

> **Note**: The struct definitions below are reconstructed from the AMD FSR Upscaling 4.1.0
> manual and SDK 2.x API documentation. Field names match the documented API but should be
> cross-checked against the actual `ffx-api/include/ffx_api/ffx_upscale.h` header before
> production use.
> [Source: AMD FSR Upscaling 4.1.0 Manual](https://gpuopen.com/manuals/fidelityfx_sdk2/techniques/super-resolution-ml/)

```c
// Reconstructed from: ffx-api/include/ffx_api/ffx_upscale.h (FidelityFX SDK 2.x)

// Version wrapper (mandatory as of SDK 2.1)
struct ffxCreateContextDescUpscaleVersion {
    ffxCreateContextDescHeader header;  // .type = FFX_API_CREATE_CONTEXT_DESC_TYPE_UPSCALE_VERSION
    uint64_t version;                   // Set to FFX_UPSCALER_VERSION
};

// Primary context descriptor
struct ffxCreateContextDescUpscale {
    ffxCreateContextDescHeader header;  // .type = FFX_API_CREATE_CONTEXT_DESC_TYPE_UPSCALE
    uint32_t  flags;                    // FFX_UPSCALE_ENABLE_* bitmask
    uint32_t  maxRenderSize[2];         // Maximum render resolution
    uint32_t  maxUpscaleSize[2];        // Display resolution
    ffxContext backendCtx;              // Populated via ffxGetContextDescUpscaleBackend
};

// Dispatch descriptor (one per frame)
struct ffxDispatchDescUpscale {
    ffxDispatchDescHeader header;       // .type = FFX_API_DISPATCH_DESC_TYPE_UPSCALE
    ffxCommandList  commandList;
    ffxResource     color;              // IN: pre-tonemapped render colour
    ffxResource     depth;              // IN: depth buffer
    ffxResource     motionVectors;      // IN: screen-space motion vectors
    ffxResource     exposure;           // IN: optional 1×1 exposure
    ffxResource     reactiveMap;        // IN: optional reactive mask
    ffxResource     output;             // OUT: upscaled colour
    float           jitterOffset[2];    // Sub-pixel jitter from Halton sequence
    float           motionVectorScale[2];
    uint32_t        renderSize[2];
    uint32_t        upscaleSize[2];
    float           frameTimeDelta;     // Milliseconds; ~16.6 at 60 Hz
    bool            reset;             // Set true on scene cuts
};
```

[Source: AMD FSR Upscaling 4.1.0 API manual](https://gpuopen.com/manuals/fidelityfx_sdk2/techniques/super-resolution-ml/)

A minimal integration sequence:

```c
// 1. Create context
ffxCreateContextDescUpscaleVersion versionDesc = {
    .header = { .type = FFX_API_CREATE_CONTEXT_DESC_TYPE_UPSCALE_VERSION },
    .version = FFX_UPSCALER_VERSION,
};
ffxCreateContextDescUpscale upscaleDesc = {
    .header = {
        .type = FFX_API_CREATE_CONTEXT_DESC_TYPE_UPSCALE,
        .pNext = &versionDesc,          // Chained version struct
    },
    .flags = FFX_UPSCALE_ENABLE_DEPTH_INVERTED |
             FFX_UPSCALE_ENABLE_AUTO_EXPOSURE   |
             FFX_UPSCALE_ENABLE_HIGH_DYNAMIC_RANGE,
    .maxRenderSize  = { renderW, renderH },
    .maxUpscaleSize = { displayW, displayH },
};
ffxContext ctx;
ffxCreateContext(&ctx, &upscaleDesc.header, NULL);

// 2. Query jitter each frame
ffxQueryDescUpscaleGetJitterOffset jitterQuery = {
    .header  = { .type = FFX_API_QUERY_DESC_TYPE_UPSCALE_GET_JITTER_OFFSET },
    .index   = frameIndex,
    .phaseCount = 0,  // filled by query
};
ffxQuery(ctx, &jitterQuery.header);
float jitterX = jitterQuery.jitterX;
float jitterY = jitterQuery.jitterY;

// 3. Dispatch upscaling each frame
ffxDispatchDescUpscale dispatchDesc = {
    .header       = { .type = FFX_API_DISPATCH_DESC_TYPE_UPSCALE },
    .commandList  = ffxCmdList,
    .color        = colorResource,
    .depth        = depthResource,
    .motionVectors= motionResource,
    .output       = outputResource,
    .jitterOffset = { jitterX, jitterY },
    .renderSize   = { renderW, renderH },
    .upscaleSize  = { displayW, displayH },
    .frameTimeDelta = 16.6f,
    .reset        = isSceneCut,
};
ffxDispatch(ctx, &dispatchDesc.header);

// 4. Destroy on shutdown
ffxDestroyContext(&ctx, NULL);
```

### 3.4 Performance Characteristics

On an AMD Radeon RX 9070 XT (RDNA 4), FSR 4 in Performance mode (2× scale) measures
approximately 352 µs at 1920×1080 and 1,316 µs at 3840×2160. GPU memory usage for a 4K
session is roughly 318 MB for the internal accumulation and weight buffers.

[Source: AMD FSR Upscaling 4.1.0 Manual](https://gpuopen.com/manuals/fidelityfx_sdk2/techniques/super-resolution-ml/)

> **Note**: FSR 4 requires RDNA 4 (RX 9000 series or newer) for neural inference. On RDNA 3
> and earlier, calling `ffxCreateContext` with `FFX_UPSCALER_VERSION` automatically selects
> FSR 3.1 temporal upscaling using the same `ffxDispatchDescUpscale` API — this is the
> "Upgradable API" compatibility guarantee.

### 3.5 Placement in the Frame Pipeline

FSR 4 must be placed **after** screen-space effects (SSAO, SSR, motion blur on opaque
geometry) but **before** post-processing (tonemapping, film grain, chromatic aberration). Film
grain or noise added before FSR will be amplified during temporal accumulation or neural
reconstruction. The output is in the same colorspace as the input (linear HDR unless
`FFX_UPSCALE_ENABLE_NON_LINEAR_COLORSPACE` is set).

---

## 4. Other FidelityFX Effects

The SDK 1.1.x bundles 20+ techniques. Four are highlighted here for their algorithmic
distinctiveness or their relevance to the Linux/Vulkan path.

### 4.1 CAS — Contrast Adaptive Sharpening

CAS is AMD's simplest post-process sharpener and the oldest FidelityFX technique (first
published at GDC 2019 in "A Survey of Temporal Antialiasing Techniques" by Timothy Lottes).
Unlike fixed unsharp-mask filters, CAS computes a per-pixel sharpening weight based on local
contrast: pixels in high-contrast regions receive lighter sharpening to avoid ringing, while
flat regions receive stronger sharpening to recover detail. The algorithm runs in a single
compute pass. CAS can also lightly upscale (up to 1.5× linear, producing output larger than
its input) as a lightweight alternative to full FSR.

[Source: AMD FidelityFX CAS on GPUOpen](https://gpuopen.com/fidelityfx-cas/)

CAS integration:

```c
// sdk/src/components/cas/ lifecycle pattern
FfxCasContextDescription casDesc = {
    .backendInterface = backendInterface,
    .colorSpaceConversion = FFX_CAS_COLOR_SPACE_LINEAR,
    .displaySize = { displayW, displayH },
    .mode = FFX_CAS_SHARPEN_ONLY,
};
FfxCasContext casCtx;
ffxCasContextCreate(&casCtx, &casDesc);

FfxCasDispatchDescription casDispatch = {
    .commandList = ffxCmdList,
    .color = colorResource,       // input
    .output = outputResource,     // output (can alias input for in-place)
    .renderSize  = { renderW, renderH },
    .displaySize = { displayW, displayH },
    .sharpness = 0.8f,            // 0.0 = minimum, 1.0 = maximum
};
ffxCasContextDispatch(&casCtx, &casDispatch);
```

### 4.2 SPD — Single Pass Downsampler

A ubiquitous utility that generates up to 12 mip levels of a texture in a single compute
dispatch. The standard multi-pass approach requires a full GPU pipeline flush between each
mip level because each pass reads the output of the previous one. SPD avoids this by
processing 64×64 tiles: within a tile, a thread group uses LDS (Local Data Share / shared
memory) for intra-tile reduction, and a single GPU-wide atomic counter tracks when the last
tile of a given mip level completes, allowing the next mip's thread groups to begin without a
full barrier.
[Source](https://gpuopen.com/fidelityfx-spd/)

```c
// sdk/src/components/spd/ integration
FfxSpdContextDescription spdDesc = {
    .backendInterface = backendInterface,
    .flags = FFX_SPD_MATH_PACKED,    // Use FP16 intermediate math (RDNA wave ops)
    .downsampleFilter = FFX_SPD_DOWNSAMPLE_FILTER_MEAN,
};
FfxSpdContext spdCtx;
ffxSpdContextCreate(&spdCtx, &spdDesc);

FfxSpdDispatchDescription spdDispatch = {
    .commandList = ffxCmdList,
    .resource = sourceTexture,   // SRV to mip 0; UAVs to mips 1–12 must be bound
};
// Returns FFX_OK; FfxSpdContext tracks atomic counter in internal GPU buffer
FfxErrorCode err = ffxSpdContextDispatch(&spdCtx, &spdDispatch);
```

[Source: FidelityFX SPD manual](https://gpuopen.com/manuals/fidelityfx_sdk/fidelityfx_sdk-page_techniques_single-pass-downsampler/)

The thread group size is `[numthreads(256, 1, 1)]`. With `FFX_SPD_MATH_PACKED`, the SPD
shaders use SM 6.0 wave operations (HLSL `WaveActiveMin`, `WaveActiveMax`) that map to
RDNA `v_min_f16_f16` wave instructions, making it significantly faster on AMD hardware than
scalar mip generation loops.

### 4.3 SSSR — Stochastic Screen Space Reflections

SSSR combines hierarchical depth-buffer ray traversal with blue-noise stochastic sampling and
temporal denoising to produce glossy and specular screen-space reflections without requiring
hardware ray tracing. The algorithm:

1. **Hierarchical depth traversal**: rays march through a depth mip pyramid generated by SPD,
   taking large steps in open space and small steps near surfaces, analogous to GPU ray
   marching through a signed distance field.

2. **Blue-noise sampling**: reflection ray direction is perturbed by a blue-noise offset (Eric
   Heitz's blue-noise sampling paper) according to surface roughness. Rough surfaces get wider
   cone jitter; mirror surfaces get no jitter.

3. **Temporal denoiser**: the stochastic noise is resolved by a spatiotemporal denoiser that
   blends current and previous frame reflections using per-pixel motion vectors. One rendered
   frame's worth of ray budget produces a temporally stable result.

[Source](https://gpuopen.com/fidelityfx-sssr/)

SSSR requires both the SPD context (to build the depth mip hierarchy each frame) and an SSSR
context:

```c
FfxSssrContextDescription sssrDesc = {
    .backendInterface    = backendInterface,
    .renderSize          = { renderW, renderH },
    .normalsHistoryBufferFormat = FFX_SURFACE_FORMAT_R8G8B8A8_UNORM,
};
FfxSssrContext sssrCtx;
ffxSssrContextCreate(&sssrCtx, &sssrDesc);
```

### 4.4 Brixelizer GI — Sparse Voxel Global Illumination

Brixelizer GI provides compute-only (no hardware ray tracing required) real-time indirect
diffuse and specular global illumination using sparse voxel distance fields.
[Source](https://gpuopen.com/fidelityfx-brixelizer/)

The GI algorithm runs in two layers:

- **Brixelizer** (lower layer): maintains a set of cascades, each a 64×64×64 grid of voxels
  centred around the camera. When geometry intersects a voxel cell, a local brick (a small
  3D SDF patch within that cell) is created or updated. Brixelizer streams geometry additions
  and removals on the GPU without stalling the render thread.

- **Brixelizer GI** (upper layer): traces rays against the sparse SDF cascade to find
  incident radiance. It maintains a radiance cache populated from previous frames'
  lighting, stored as spherical harmonics per valid brick, and uses screen probes to sample
  this cache. Outputs are denoised indirect diffuse and specular at render resolution,
  ready to composite into the final lighting pass.

On RDNA hardware, Brixelizer GI selects SM 6.6 wave operations when available, falling back
to SM 6.0 wave ops for portability on older hardware and non-AMD GPUs:

```c
FfxBrixelizerContextDescription brixDesc = {
    .backendInterface = backendInterface,
    .flags = FFX_BRIXELIZER_CONTEXT_FLAG_ALL_DEBUG,
    .numCascades = 8,
};
FfxBrixelizerContext brixCtx;
ffxBrixelizerContextCreate(&brixCtx, &brixDesc);
```

[Source: Brixelizer GI docs](https://github.com/GPUOpen-LibrariesAndSDKs/FidelityFX-SDK/blob/main/docs/techniques/brixelizer-gi.md)

---

## 5. AMF — Advanced Media Framework

### 5.1 Overview and Linux History

The Advanced Media Framework (AMF) is AMD's hardware-accelerated multimedia encoding and
decoding API. On Windows it wraps the VCE (Video Coding Engine) and VCN (Video Core Next)
hardware units inside RDNA and CDNA GPUs, providing H.264, HEVC, AV1, and AVC encode/decode.
[Source](https://github.com/GPUOpen-LibrariesAndSDKs/AMF)

On Linux, AMF's history is more complex:

- **Pre-driver 25.20**: AMF was bundled with the proprietary `amf-amdgpu-pro` component.
  RDNA 3 and newer supported a semi-open path using `amf-amdgpu` with the Mesa Vulkan
  (`vulkan-radeon/RADV`) driver. RDNA 2 and earlier required the full proprietary stack.

- **Driver 25.20+ (2025)**: AMD removed all proprietary components from the Linux graphics
  driver stack. AMF is no longer shipped as part of the graphics driver. AMD's official
  recommendation is to use **VA-API via Mesa Multimedia** for hardware video acceleration on
  Linux. The AMF runtime is released separately from the graphics driver starting with this
  version.
  [Source: AMD Linux driver 25.20 release notes](https://www.amd.com/en/resources/support-articles/release-notes/RN-AMDGPU-UNIFIED-LINUX-25-20-3.html)

The strategic shift aligns with AMD's broader commitment to the open-source stack: VA-API
acceleration through Mesa's `radeonsi` (OpenGL/compute) and RADV (Vulkan) provides
equivalent encode/decode quality via the `iHD` and `radeonsi_drv_video.so` VA-API drivers.

### 5.2 AMF Core Architecture (Windows/Cross-Platform)

Although AMF is no longer the primary Linux path, its architecture is documented here because
AMF remains important on Windows and for streaming applications (OBS, HandBrake) that target
multiple platforms.

AMF is structured around a factory/component model implemented as a COM-like interface in C++:

```cpp
// amf/public/include/core/Factory.h
class AMFFactory {
public:
    virtual AMFResult AMF_STD_CALL Init(amf_uint64 version, AMFTrace** ppTrace) = 0;
    virtual AMFResult AMF_STD_CALL CreateContext(AMFContext** ppContext) = 0;
    virtual AMFResult AMF_STD_CALL GetTrace(AMFTrace** ppTrace) = 0;
    virtual AMFResult AMF_STD_CALL GetDebug(AMFDebug** ppDebug) = 0;
};

// Access via DLL/DSO entry point
typedef AMFResult (AMF_CDECL_CALL * AMFQueryVersion_Fn)(amf_uint64* pVersion);
typedef AMFResult (AMF_CDECL_CALL * AMFInit_Fn)(amf_uint64 version, AMFFactory** ppFactory);
```

[Source: AMF GitHub](https://github.com/GPUOpen-LibrariesAndSDKs/AMF)

An `AMFContext` wraps the GPU device and allocator. Video components — encoder, decoder,
pre-processor — are `AMFComponent` instances created from the context:

```cpp
// Initialize factory
// Note: the exact DSO name depends on the AMF runtime package;
// on recent AMD Linux stacks it may be "libamfrt64.so" or accessed
// via the AMF runtime loader — verify against the installed package.
void* amfLib = dlopen("libamfrt64.so", RTLD_LAZY);
AMFInit_Fn amfInit = (AMFInit_Fn)dlsym(amfLib, AMF_INIT_FUNCTION_NAME);
AMFFactory* factory = nullptr;
amfInit(AMF_FULL_VERSION, &factory);

// Create device context (Vulkan device path on Linux)
AMFContext* context = nullptr;
factory->CreateContext(&context);
context->InitVulkan(vkDevice, nullptr);

// Create H.265/HEVC encoder component
AMFComponent* encoder = nullptr;
factory->CreateComponent(context, AMFVideoEncoderHW_HEVC, &encoder);

// Configure encoder properties
encoder->SetProperty(AMF_VIDEO_ENCODER_HEVC_USAGE,       AMF_VIDEO_ENCODER_HEVC_USAGE_TRANSCODING);
encoder->SetProperty(AMF_VIDEO_ENCODER_HEVC_QUALITY_PRESET, AMF_VIDEO_ENCODER_HEVC_QUALITY_PRESET_QUALITY);
encoder->SetProperty(AMF_VIDEO_ENCODER_HEVC_TARGET_BITRATE, (amf_int64)8000000);  // 8 Mbps
encoder->Init(AMF_SURFACE_NV12, width, height);

// Submit a surface for encoding
AMFSurface* surface = nullptr;
context->AllocSurface(AMF_MEMORY_VULKAN, AMF_SURFACE_NV12, width, height, &surface);
// ... fill surface planes ...
encoder->SubmitInput(surface);

// Retrieve encoded bitstream
AMFData* data = nullptr;
encoder->QueryOutput(&data);
AMFBuffer* buffer = static_cast<AMFBuffer*>(data);
void* encoded = buffer->GetNative();   // Pointer to raw H.265 NAL units
```

The `AMFSurface` / `AMFBuffer` pair maps to GPU memory surfaces and linear output buffers
respectively. On the Linux VA-API path, `AMFContext::InitVulkan` configures AMF to use the
RADV Vulkan device, and the encode/decode passes translate through Mesa's `VK_KHR_video_encode_queue`
and `VK_KHR_video_decode_queue` Vulkan Video extensions (see Ch26 and Ch50).

### 5.3 Comparison with NVENC

| Feature | AMD AMF (Windows) | AMD VA-API/Mesa (Linux) | NVIDIA NVENC |
|---------|-------------------|------------------------|-------------|
| Codecs | H.264, HEVC, AV1 | H.264, HEVC, AV1 | H.264, HEVC, AV1, H.266 |
| API style | COM-like C++ | C + libva | CUDA-based C |
| Linux support | Separate runtime (post-25.20) | Full open-source | Proprietary |
| Zero-copy interop | Vulkan surface → AMF | DMA-BUF / VA surface | CUDA → NVENC |
| Rate control | CBR, VBR, CQP | CBR, VBR, CQP, CRF | CBR, VBR, CQP, lookahead |

For Linux-native applications (FFmpeg, GStreamer, OBS), the recommended path is
`ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD128` or
`gst-launch-1.0 ... ! vaapih265enc`, which routes to Mesa's VA-API backend and the VCN
hardware unit via the amdgpu kernel driver.

---

## 6. Radeon GPU Profiler (RGP)

### 6.1 Architecture

The Radeon GPU Profiler is AMD's frame-level GPU timeline profiler with instruction-level
timing granularity. It is open source, hosted at
[github.com/GPUOpen-Tools/radeon_gpu_profiler](https://github.com/GPUOpen-Tools/radeon_gpu_profiler).

RGP consists of two separate tools that communicate via the **Radeon Developer Panel (RDP)**:

```
┌──────────────────────────────────────────────────────────────────┐
│  Application (Vulkan / DX12 / OpenCL / HIP)                     │
│    │                                                              │
│    ▼                                                              │
│  Driver (amdgpu UMD / AMDVLK / RADV)                            │
│    │  ◄──── RDP capture trigger (SIGPIPE/socket) ────────────────┤
│    │        writes SQTT token stream to GPU-resident buffer       │
│    ▼                                                              │
│  .rgp capture file (SQTT data + counter blocks + timing)         │
└──────────────────────────────────────────────────────────────────┘
                    │
                    ▼
        RGP GUI (Qt5, reads .rgp file, renders timeline)
```

On Linux, RDP connects to the driver via the **Radeon Developer Service (RDS)** daemon which
uses a local socket. The user starts RDS before launching the application, connects RDP to it,
configures capture parameters, then triggers capture either by frame number or pressing
Shift+F11.

[Source: AMD Radeon GPU Profiler on GPUOpen](https://gpuopen.com/rgp/)

### 6.2 SQTT — Shader Queue Thread Trace

The low-level hardware mechanism behind RGP is **SQTT** (Shader Queue Thread Trace), exposed
to software via the Vulkan extension **`VK_AMD_gpa_interface`**.
[Source: VK_AMD_gpa_interface spec](https://docs.vulkan.org/features/latest/features/proposals/VK_AMD_gpa_interface.html)

SQTT uses dedicated on-chip hardware in the RDNA SQ (Shader Queue) block to record
instruction-level execution tokens for each wavefront. These tokens capture:

- Which instruction was issued in each clock cycle per SIMD lane
- Stall cycles waiting for memory (L1/L2/VRAM latency)
- Wavefront occupancy and scheduling decisions
- Cache hit/miss events per instruction

The `VK_AMD_gpa_interface` extension defines three session sample types:

```c
// From VK_AMD_gpa_interface spec
typedef enum VkGpaSampleTypeAMD {
    VK_GPA_SAMPLE_TYPE_CUMULATIVE_AMD = 0,  // Per-counter delta values
    VK_GPA_SAMPLE_TYPE_TRACE_AMD      = 1,  // SQTT + SPM data → RGP file format
    VK_GPA_SAMPLE_TYPE_TIMING_AMD     = 2,  // Timestamp pairs at pipeline stages
} VkGpaSampleTypeAMD;

// SQTT configuration inside VkGpaSampleBeginInfoAMD
typedef struct VkGpaSqThreadTraceCreateInfoAMD {
    VkBool32  sqThreadTraceEnable;
    VkBool32  sqThreadTraceSuppressInstructionTokens; // Reduce data volume
    uint32_t  sqShaderMask;      // VK_GPA_SHADER_MASK_PS | _VS | _CS etc.
    uint64_t  sqThreadTraceDeviceMemoryLimit;         // Buffer size cap
} VkGpaSqThreadTraceCreateInfoAMD;
```

The `VK_GPA_SAMPLE_TYPE_TRACE_AMD` sample type fills a GPU memory buffer with SQTT token
data in the proprietary **RGP file format** (`.rgp`). This format encodes:

- **SQTT data chunks**: one chunk per shader engine, containing wavefront token streams.
- **Performance counter blocks**: cumulative counter values from hardware blocks including
  CPF (command processor frontend), IA (input assembler), VGT (vertex grouper/tessellator),
  SPI (shader processor input), SQ (shader queue), CB (colour buffer), DB (depth buffer),
  and memory subsystem blocks.
- **Event timing**: timestamps at vkCmdDraw / vkCmdDispatch boundaries, enabling per-draw
  GPU duration measurement.
- **Barrier information**: synchronisation events with flush/invalidation bitmasks.

### 6.3 Barrier and Pipeline Stall Analysis

RGP's barrier analysis tab presents every `VkCmdPipelineBarrier` / `vkCmdMemoryBarrier` call
with:

- Which caches were flushed (L1 shader data cache, L2 cache, VRAM)
- Whether an internal "resolve barrier" shader was needed (some barrier configurations on
  RDNA hardware require a small driver-generated shader to flush metadata)
- The GPU time lost to the barrier (stall cycles between the last wave before the barrier and
  the first wave after)

Pipeline stall analysis correlates the SQTT wavefront token stream with the hardware
occupancy data to identify bottlenecks: a high ratio of VALU (vector ALU) stall tokens
indicates shader occupancy limits or instruction-level latency; a high ratio of VMEM
(vector memory) stall tokens indicates VRAM bandwidth saturation.

The **Wavefront Occupancy** pane shows active wavefronts per SIMD unit over time. RDNA uses
Wave32 by default, with maximum wavefront counts per SIMD depending on register file usage
(up to ~20 waves per SIMD32 in resource-light compute shaders; consult the RDNA architecture
programmer's guide for exact limits per GPU generation). A frame that consistently shows only
2–4 active wavefronts per SIMD is occupancy-limited and needs either larger workgroups or
reduced register pressure.

> **Note: needs verification** — exact maximum wavefront-per-SIMD figures for RDNA 4 should
> be confirmed against the AMD RDNA 4 Architecture Programmer's Reference Manual, which AMD
> publishes at [gpuopen.com](https://gpuopen.com/amd-rdna4-architecture-technical-reference-guide/).

### 6.4 Linux Capture Workflow

```bash
# 1. Start Radeon Developer Service (runs as a daemon)
$ RadeonDeveloperService &

# 2. Launch the application with the RDP backend active
$ AMDGPU_ENABLE_RGP=1 AMDGPU_RGP_OUTPUT_DIR=/tmp/captures ./my_vulkan_app

# 3. Open Radeon Developer Panel, connect to RDS at 127.0.0.1:27300
# 4. Navigate to "Profiling" → set SQTT buffer size if needed
# 5. Press Shift+F11 in the app or trigger from RDP GUI
# 6. Open the .rgp file in the RGP GUI
```

RGP on Linux supports Ubuntu 24.04 LTS with Vulkan, OpenCL, and HIP workloads.
[Source: Radeon GPU Profiler release notes](https://github.com/GPUOpen-Tools/radeon_gpu_profiler/releases)

---

## 7. Radeon Memory Visualizer (RMV)

### 7.1 Purpose and Architecture

The Radeon Memory Visualizer (RMV) is a GPU memory profiler that captures the full timeline
of VRAM allocations, deallocations, and resource bindings over the lifetime of an application
session. It is open source at
[github.com/GPUOpen-Tools/radeon_memory_visualizer](https://github.com/GPUOpen-Tools/radeon_memory_visualizer).

RMV captures are generated by the Radeon Developer Service (RDS) daemon — the same service
used by RGP. When a capture is requested, RDS installs a callback into the AMD UMD (User
Mode Driver) that receives every memory management event:

- `ALLOCATION_CREATE`: heap type, size, alignment, usage flags
- `ALLOCATION_DESTROY`: resource lifetime end
- `RESOURCE_BIND`: a `VkBuffer` or `VkImage` bound to a VkDeviceMemory allocation
- `RESOURCE_MAKE_RESIDENT` / `RESOURCE_EVICT`: residency state changes

These events are logged to a `.rmv` trace file with nanosecond timestamps from the GPU
timeline. The heap types correspond directly to Vulkan memory property flags:

| RMV Heap | Vulkan Memory Properties | Notes |
|----------|--------------------------|-------|
| Local (VRAM) | `VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT` | Fastest GPU access |
| Invisible | Device-local, not host-visible | Large VRAM on discrete GPUs |
| Host-visible | `VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT \| DEVICE_LOCAL` | 256 MB BAR or smart access memory |
| System (GTT) | Host-coherent, not device-local | CPU RAM mapped for GPU |

[Source: RMV Getting Started](https://gpuopen.com/learn/getting-started-with-radeon-memory-visualizer-rmv/)

### 7.2 Analysis Views

The RMV UI presents several analysis perspectives:

**Virtual Memory Heap Timeline** (the default timeline view): a horizontal scrollable timeline
where each row is a heap, and allocation events appear as coloured bars. Overlapping bars
indicate fragmentation — VRAM ranges that are allocated but interleaved with free space,
preventing larger contiguous allocations. This view is particularly useful for diagnosing
`VK_ERROR_OUT_OF_DEVICE_MEMORY` failures that occur even when `VkPhysicalDeviceMemoryProperties`
reports sufficient total VRAM.

**Resource Usage Timeline**: shows per-resource-type (texture, buffer, RTV/DSV) memory usage
over time. Useful for finding textures that persist in VRAM longer than necessary (streaming
asset not unloaded, descriptor set holding a reference preventing deallocation).

**Snapshots**: right-clicking on the timeline at any point creates a snapshot of the exact
VRAM state at that moment. Two snapshots can be compared to identify memory that was
allocated between them — the standard technique for locating memory leaks in Vulkan
applications.

**VRAM Fragmentation Analysis**: RMV 1.8+ added a dedicated fragmentation score visible in
the heap summary. A high fragmentation score means many small free blocks between large
allocations, indicating the application should use a suballocator (such as AMD's
`VulkanMemoryAllocator` library) rather than creating one `VkDeviceMemory` per resource.

### 7.3 AMD Smart Access Memory (SAM) / Resizable BAR

RMV is particularly useful for profiling AMD APU and discrete GPU configurations with Smart
Access Memory (SAM / `VK_AMD_device_coherent_memory`). When SAM is active, the host-visible
device-local heap (the 256 MB BAR default becomes full VRAM) appears in RMV as a much larger
"Host-visible" heap. Applications can track whether large textures are benefiting from
direct CPU → VRAM placement, or whether they are falling through to system RAM and incurring
PCIe DMA overhead.

```bash
# Linux capture workflow
$ RadeonDeveloperService &
$ RDP_CAPTURE=1 ./my_vulkan_app  # or trigger via RDP GUI
# → produces app.rmv in the configured output directory
$ RadeonMemoryVisualizer app.rmv
```

---

## 8. RenderDoc — Cross-Vendor Frame Debugger

### 8.1 Architecture and Origins

RenderDoc is an open-source frame debugger and graphics API inspector created by Baldur
Karlsson, with the source code at
[github.com/baldurk/renderdoc](https://github.com/baldurk/renderdoc). Despite being
cross-vendor and cross-API (Vulkan, D3D11, D3D12, OpenGL), RenderDoc was largely developed
with AMD hardware as the primary testing environment and AMD engineers contributing backend
improvements.

[Source: RenderDoc Wikipedia](https://en.wikipedia.org/wiki/RenderDoc)

RenderDoc uses an **in-process capture layer** model. On Linux with Vulkan:

1. RenderDoc is injected via `LD_PRELOAD` or the RenderDoc launcher, which installs a Vulkan
   layer (`VK_LAYER_RENDERDOC_Capture`) by writing to the implicit layer JSON search path.

2. The layer intercepts all Vulkan entry points by acting as a "man in the middle": the
   application calls `vkQueueSubmit` → RenderDoc layer → real `vkQueueSubmit`. In background
   mode the layer records only resource creation/destruction. When capture is triggered,
   it enters **active capture mode** and serialises every API call.

3. The capture trigger can come from the hotkey (F12 / PrintScreen), the RenderDoc GUI, or
   the in-application API (see §8.3). Capture is framed by `vkQueuePresentKHR` calls: one
   frame = one present.

4. The capture is serialised to an `.rdc` file using a chunk-based binary format: each chunk
   has an integer type, byte length, and serialised data. Chunk types correspond to API
   calls (e.g., `vkCreateImage`, `vkCmdDraw`, `vkCmdDispatch`). Initial resource contents
   (texture data, buffer data) are stored as separate chunks in the "initial contents"
   section.

[Source: RenderDoc architecture documentation](https://renderdoc.org/docs/behind_scenes/how_works.html)

### 8.2 Vulkan Capture Path

For Vulkan, RenderDoc's capture layer hooks the three critical submission points:

- **`vkQueueSubmit`**: serialises all command buffers being submitted. Each recorded
  `VkCommandBuffer` is replayed by re-recording it through the RenderDoc tracking layer.
- **`vkQueuePresentKHR`**: marks frame boundaries and triggers the save-to-disk of the
  captured frame data.
- **`vkAllocateMemory` / `vkBindBufferMemory` / `vkBindImageMemory`**: track resource
  lifetime and backing allocations for the initial contents section.

During replay, RenderDoc re-creates all resources from the `.rdc` file and replays command
buffer recording in order, enabling the user to step through individual draw calls and inspect
GPU state (textures, buffers, render targets, descriptor sets) at each step.

### 8.3 In-Application Capture API

RenderDoc exposes a C API for programmatic capture triggering, loaded at runtime via
`dlopen`:

```c
// Link: renderdoc.h (shipped with RenderDoc)
#include "renderdoc_app.h"

RENDERDOC_API_1_1_2* rdoc = NULL;

// Load RenderDoc at runtime (inject before device creation)
void* rdoc_lib = dlopen("librenderdoc.so", RTLD_NOW | RTLD_NOLOAD);
if (rdoc_lib) {
    pRENDERDOC_GetAPI RENDERDOC_GetAPI =
        (pRENDERDOC_GetAPI)dlsym(rdoc_lib, "RENDERDOC_GetAPI");
    int ret = RENDERDOC_GetAPI(eRENDERDOC_API_Version_1_1_2, (void**)&rdoc);
    assert(ret == 1);
}

// In render loop: trigger programmatic capture
if (rdoc && captureThisFrame) {
    rdoc->StartFrameCapture(NULL, NULL);  // NULL = capture on any device/window
}

// ... render frame ...

if (rdoc && captureThisFrame) {
    rdoc->EndFrameCapture(NULL, NULL);
}
```

[Source: RenderDoc in-application API](https://renderdoc.org/docs/in_application_api.html)

The in-application API also provides `TriggerCapture()` (equivalent to the hotkey),
`TriggerMultiFrameCapture(N)` for capturing N consecutive frames, `IsFrameCapturing()` to
avoid redundant work during capture, and `SetCaptureTitle()` for labelling captures in the UI.

### 8.4 Python API for Automated Analysis

RenderDoc exposes its full replay engine to Python, enabling headless automated analysis of
`.rdc` captures. This is valuable for CI pipelines (regression testing shader outputs),
automated frame statistics extraction, or batch resource inspection.

```python
# RenderDoc Python API — run via: renderdoccmd script my_analysis.py capture.rdc
import renderdoc as rd

def analyse_capture(controller):
    """Iterate all draw calls and report resource usage."""
    actions = controller.GetRootActions()

    def walk(action_list, depth=0):
        for action in action_list:
            if action.flags & rd.ActionFlags.Drawcall:
                # Move replay to this draw call's event ID
                controller.SetFrameEvent(action.eventId, True)
                state = controller.GetPipelineState()
                # Inspect bound index and vertex buffers
                ib  = state.GetIBuffer()   # index buffer (singular)
                vbs = state.GetVBuffers()  # vertex buffers (plural)
                print(f"{'  ' * depth}Draw EID={action.eventId} "
                      f"indices={action.numIndices} IB={ib.resourceId} "
                      f"VBs={[v.resourceId for v in vbs]}")
            walk(action.children, depth + 1)

    walk(actions)

    # Access textures at a specific draw call
    controller.SetFrameEvent(actions[0].eventId, True)
    textures = controller.GetTextures()
    for tex in textures[:5]:
        print(f"  Texture: {tex.name} {tex.width}x{tex.height} fmt={tex.format.Name()}")

# Entry point when run via renderdoccmd script
if 'pyrenderdoc' in dir():
    pyrenderdoc.Replay().BlockInvoke(analyse_capture)
```

[Source: RenderDoc Python API](https://renderdoc.org/docs/python_api/index.html)
[Source: RenderDoc basic Python examples](https://renderdoc.org/docs/python_api/examples/basics.html)

The `renderdoc` Python module exposes:

- `ReplayController.GetRootActions()` → list of `ActionDescription` (draw calls, dispatches, clears)
- `ReplayController.SetFrameEvent(eid, force)` → seek to a specific event for state inspection
- `ReplayController.GetPipelineState()` → full bound pipeline and resource state snapshot
- `ReplayController.GetBufferData(buffer, offset, length)` → raw buffer bytes
- `ReplayController.GetTextureData(tex, sub)` → pixel data for any mip/array slice

The `qrenderdoc` module extends this with UI integration for scripts running inside the
RenderDoc Qt GUI, enabling panel automation and custom analysis views.

---

## 9. Integration with the Open-Source Stack

### 9.1 FidelityFX on RADV

FidelityFX effects that target Vulkan run on RADV (the Mesa open-source AMD Vulkan driver)
without any special configuration. The SDK's pre-compiled SPIR-V shader blobs are standard
SPIR-V 1.6, processed by RADV's shader compiler pipeline:

```
FidelityFX SPIR-V blob
    → RADV vkCreateShaderModule
    → nir_from_spirv() (Mesa NIR frontend)
    → ACO shader compiler (aco::select_program, Ch15)
    → RDNA ISA binary
    → amdgpu kernel driver (executes via PM4 packets)
```

The ACO compiler's instruction scheduling and register allocation are generally well-tuned for
the FidelityFX shader patterns (compute-heavy, wave-wide, with significant LDS usage in SPD
and CAS). Users can verify shader compilation with:

```bash
AMD_DEBUG=preoptir,shaders RADV_DEBUG=shaders ./my_fidelityfx_app 2>&1 | grep -A 20 "SPD"
```

### 9.2 RGP and the AMDGPU Performance Counter Interface

RGP's SQTT capture depends on the amdgpu kernel driver's performance counter subsystem,
exposed via `VK_AMD_gpa_interface` through AMDVLK (AMD's open-source official Vulkan
driver) and partially via RADV. The kernel-side interface is in
`drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c` and the PM4 register programming for SQ_THREAD_TRACE
is in the per-ASIC GFX pipeline headers.

> **Note: needs verification** — `VK_AMD_gpa_interface` support in RADV (as opposed to
> AMDVLK) is partial. The performance counter registers require `CAP_SYS_ADMIN` or the
> `perf_event_paranoid` sysctl to be relaxed to 0 or below:
>
> ```bash
> # Required for RGP capture on Linux
> echo 0 | sudo tee /proc/sys/kernel/perf_event_paranoid
> ```
>
> Without this, the SQTT buffer cannot be allocated from GPU-reserved memory, and captures
> produce empty SQTT sections (timing data is still present).

### 9.3 RMV and the UMD Allocation Callback

RMV's capture mechanism hooks into the AMD UMD memory management callbacks. In the open-source
stack (Mesa/RADV), allocations flow through:

```
vkAllocateMemory
  → radv_alloc_memory() (src/amd/vulkan/radv_device_memory.c)
  → amdgpu_bo_alloc() (libdrm_amdgpu)
  → DRM_AMDGPU_GEM_CREATE ioctl (kernel)
```

RMV via RDS intercepts at the libdrm/UMD boundary, so it captures allocations from both
RADV and `radeonsi` (OpenGL), `radeon_drm` (legacy), and compute APIs (ROCm/HIP) on the
same GPU. The `/dev/dri/renderD128` render node is the single choke point.

### 9.4 RenderDoc and the Mesa RADV Backend

RenderDoc has deep integration with RADV. When capturing Vulkan on AMD hardware through RADV:

- RenderDoc disables RADV's internal shader cache (`RADV_DEBUG=nocache`) during capture to
  ensure deterministic replay.
- RenderDoc uses `VK_EXT_debug_utils` labels from RADV's internal tracking to correlate
  Mesa NIR shader names with RenderDoc pipeline stages.
- `VK_AMD_buffer_marker` (supported by RADV) can be used in RenderDoc's GPU crash reporting
  to identify which command in a buffer was executing when `VK_ERROR_DEVICE_LOST` occurred.

The RenderDoc Vulkan layer is compatible with all Mesa Vulkan drivers (RADV, ANV, NVK,
freedreno, panfrost) since it operates purely at the `VkInstance`/`VkDevice` API boundary.

---

## 10. Integrations

This chapter connects to numerous other chapters across the book:

- **Ch5 (amdgpu Kernel Driver)**: FidelityFX effects, RGP SQTT, and RMV all ultimately use
  `DRM_AMDGPU_GEM_CREATE`, PM4 command streams, and the amdgpu performance counter interface
  exposed through the kernel DRM driver.

- **Ch15 (ACO Compiler)**: FidelityFX SPIR-V shaders are compiled by ACO when running on
  RADV. ACO's wave-level operation support is critical for SPD and CAS performance; the
  `WAVE32` mode on RDNA (32-wide wavefronts vs GCN's 64-wide) changes thread group
  efficiency.

- **Ch18 (RADV — Vulkan Driver)**: FidelityFX SDK's Vulkan backend communicates with RADV
  via standard `VkDevice` operations. RGP's `VK_AMD_gpa_interface` is more complete in
  AMDVLK but partially available in RADV; RenderDoc works fully on RADV.

- **Ch25 (GPU Compute)**: Brixelizer GI and SPD are pure compute workloads. AMF on Linux
  now delegates to the same VCN hardware via VA-API that Mesa's compute path uses. ROCm
  workloads and FidelityFX effects can share the same `amdgpu` render node.

- **Ch26 (Hardware Video Acceleration)**: AMF's encoding pipeline now delegates to VA-API
  (libva + `radeonsi_drv_video.so`). FFmpeg's `-hwaccel vaapi` path is the recommended
  Linux replacement for AMF encode in non-Windows workflows.

- **Ch29 (Upscaling, Overlays, and Frame Pacing)**: FSR 4 is the AMD counterpart to DLSS 4
  and XeSS. The `vkBasalt` Vulkan layer can inject CAS or FSR 1 into any Vulkan game without
  source code access. MangoHud exposes FSR scaling mode status.

- **Ch30 (Debugging and Profiling Tools)**: RenderDoc and RGP are the two primary
  GPU debugging tools described in that chapter's overview. The `umr` tool (User Mode
  Register debugger) provides complementary MMIO-level inspection alongside RGP's
  high-level timeline view.

- **Ch48 (ROCm and HIP)**: RGP supports HIP workloads in addition to Vulkan graphics.
  Brixelizer GI's compute dispatches are structurally equivalent to HIP kernels using the
  same `amdgpu` queue infrastructure. ROCm's profiling (`rocprof`) is the HIP-centric
  complement to RGP's Vulkan-centric approach.

- **Ch56 (Vulkan Ray Tracing)**: SSSR provides a compute-only alternative to
  `VK_KHR_ray_query` reflections. Brixelizer GI similarly avoids `VK_KHR_acceleration_structure`,
  making it compatible with pre-RDNA2 hardware that lacks hardware ray tracing units.

- **Ch76 (Modern Vulkan Extensions)**: `VK_AMD_gpa_interface`, `VK_AMD_buffer_marker`,
  `VK_AMD_device_coherent_memory` (SAM), and `VK_KHR_video_encode_queue` are all AMD-driven
  Vulkan extensions that underpin the tools described in this chapter.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
