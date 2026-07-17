# Chapter 104: DXVK and VKD3D-Proton — D3D-to-Vulkan Translation

> **Part**: Part VIII — The Gaming Layer
> **Audience**: Linux gaming developers, Proton contributors, and graphics engineers who need to understand how the Direct3D API surface maps onto Vulkan, how shader bytecode is translated at runtime, and how the translation layer ecosystem has become the dominant Vulkan consumer on the Linux desktop
> **Status**: First draft — 2026-06-19

## Table of Contents

- [Overview](#overview)
- [1. Introduction — Why D3D-to-Vulkan Translation Matters](#1-introduction--why-d3d-to-vulkan-translation-matters)
- [2. Direct3D Architecture Primer](#2-direct3d-architecture-primer)
- [3. DXVK Architecture (D3D9/10/11 → Vulkan)](#3-dxvk-architecture-d3d91011--vulkan)
- [4. The DXVK State Cache and Shader Compilation](#4-the-dxvk-state-cache-and-shader-compilation)
- [5. VKD3D-Proton Architecture (D3D12 → Vulkan)](#5-vkd3d-proton-architecture-d3d12--vulkan)
- [6. Shader Translation: DXBC/DXIL → SPIR-V](#6-shader-translation-dxbcdxil--spir-v)
- [7. Proton Integration](#7-proton-integration)
- [8. Performance Considerations](#8-performance-considerations)
- [9. Ray Tracing Translation](#9-ray-tracing-translation)
- [10. Debugging and Development](#10-debugging-and-development)
- [11. Fossilize and the Steam Shader Pre-Caching Pipeline](#11-fossilize-and-the-steam-shader-pre-caching-pipeline)
- [Integrations](#integrations)
- [References](#references)

---

## Overview

DXVK and VKD3D-Proton are the two translation layers that allow the overwhelming majority of Windows PC games to run on Linux without modification. Together they implement the full Direct3D 9, 10, 11, and 12 API surfaces in terms of Vulkan, making them the most widely used Vulkan consumers on the Linux platform — more than any game engine or application framework. This chapter examines:

- **Internal architecture** — how DXVK and VKD3D-Proton are structured internally
- **Shader translation pipeline** — from DirectX bytecode to SPIR-V
- **Synchronisation challenges** — bridging D3D's implicit resource tracking model to Vulkan's explicit model
- **Proton integration** — how the translation layers are packaged for Steam

It targets three overlapping audiences:

- **Linux gaming developers** — who ship Proton-compatible titles
- **Proton and Mesa contributors** — who tune the translation layers themselves
- **Graphics engineers** — who want to understand how a production-scale D3D-to-Vulkan translation layer works as a reference for their own API bridging work

---

## 1. Introduction — Why D3D-to-Vulkan Translation Matters

### The Market Reality

The Linux gaming ecosystem faces an asymmetry: the overwhelming majority of commercial PC game titles are authored against Direct3D, not Vulkan or OpenGL. As of 2026, ProtonDB reports that approximately 89.7% of Windows titles on Steam launch successfully on Linux, with 42% of new releases earning Platinum ratings — all made possible by DXVK and VKD3D-Proton. [Source](https://commandlinux.com/statistics/steam-linux-runtime/) In the March 2026 Steam hardware survey, Linux reached 5.33% of all Steam users for the first time, representing roughly 7 million Linux gamers, the vast majority relying on Proton. [Source](https://windowsforum.com/threads/linux-gaming-hits-5-33-on-steam-mar-2026-steam-deck-proton-windows-10-end.413359/)

The alternative to translation layers would be native Vulkan ports or OpenGL ports — neither of which game publishers have historically provided at scale. Translation layers turn the existing Windows library of ~117,000 Steam titles into a de facto Linux library, and they do so at performance levels that often match or approach the Windows reference.

### The Projects

**DXVK** implements Direct3D 8, 9, 10, and 11 on top of Vulkan. It was created by Philip "doitsujin" Rebohle; in 2018 Valve began sponsoring its development full-time. The project is hosted at [github.com/doitsujin/dxvk](https://github.com/doitsujin/dxvk). As of version 2.7.1 (the latest stable release at time of writing), DXVK requires Vulkan 1.3 and `VK_KHR_maintenance5` as mandatory extensions. It is implemented in C++ with a Meson build system and cross-compiled for Windows PE targets using Mingw-w64.

**VKD3D-Proton** implements Direct3D 12 on top of Vulkan. It is a fork of the upstream `vkd3d` project (maintained by WineHQ), created by Hans-Kristian Arntzen ("HansKristian-Work") specifically for Proton. The fork aggressively uses modern Vulkan extensions and explicitly rejects backwards compatibility with older driver versions. It is hosted at [github.com/HansKristian-Work/vkd3d-proton](https://github.com/HansKristian-Work/vkd3d-proton). Version 3.0.1, released May 2026, is the current stable release.

Both projects ship as part of Valve's **Proton** compatibility tool for Steam. Outside of Proton, DXVK is also widely used standalone in Lutris, Bottles, and Heroic Games Launcher for running non-Steam Windows games.

Multiple translation layers coexist because no single layer covers the entire D3D API surface with equal fidelity: DXVK targets D3D9–11 via Vulkan with full state-cache support, while VKD3D-Proton exclusively targets D3D12 and exploits bleeding-edge Vulkan extensions that the older API generations cannot benefit from. WineD3D and Mesa Nine fill complementary niches — WineD3D as the universal fallback that works even without Vulkan support, and Mesa Nine as a zero-overhead native Gallium path for D3D9 workloads on Mesa drivers that bypasses translation entirely.

| Layer | D3D target | Output API | Shader translator | State cache | Ray tracing | Integration | Maintained by |
|---|---|---|---|---|---|---|---|
| **DXVK** | D3D9, D3D10, D3D11 | Vulkan 1.3 | DXBC→SPIR-V (dxbc2spirv) | Yes (DXVK state cache, `.dxvk-cache`) | No (D3D11 has no RT) | Proton, Wine (WineD3D replacement) | Philip Rebohle (doitsujin); community |
| **VKD3D-Proton** | D3D12 | Vulkan 1.3 | DXIL→SPIR-V (vkd3d-shader) | Yes (pipeline library via `VK_EXT_pipeline_library`) | Yes (`VK_KHR_ray_tracing_pipeline` on RDNA2+/RTX) | Proton only (Valve fork of vkd3d) | Valve / Hans-Kristian Arntzen |
| **Mesa Nine (D3D9)** | D3D9 only | Gallium3D (native pipe driver interface) | No translation (native Gallium state) | Inherits Mesa disk cache | No | Wine (`--with-nine`; native path, not DXVK) | Mesa community; Axel Davy |
| **WineD3D** | D3D8, D3D9, D3D10, D3D11 | OpenGL 4.x / Vulkan (partial) | Vertex/pixel shader→GLSL | No | No | Wine upstream (default fallback) | Wine project |

---

## 2. Direct3D Architecture Primer

Understanding what DXVK and VKD3D-Proton translate requires understanding what they are translating from. The Direct3D API family spans three distinct generations with very different device models.

### D3D9: IDirect3DDevice9

Direct3D 9 is the oldest API family DXVK supports. Its key interfaces are `IDirect3DDevice9` (the main rendering device), `IDirect3DVertexBuffer9`, `IDirect3DTexture9`, and shader objects created via `IDirect3DDevice9::CreateVertexShader` / `CreatePixelShader`. Draw calls are `DrawPrimitive` and `DrawIndexedPrimitive`. D3D9 has a fixed-function pipeline remnant (the `D3DRS_*` render state enumeration, the `IDirect3DVertexDeclaration9` vertex format), although in practice most D3D9 games use programmable shaders in Shader Model 2.0 or 3.0. Multisampling, render targets, and depth/stencil surfaces are managed as typed `IDirect3DSurface9` objects. The resource model is implicit: the driver tracks which resources are in use and handles hazards automatically.

### D3D10/11: The Device Context Model

D3D11 introduced the device context model that separates resource creation (via `ID3D11Device`) from command recording (via `ID3D11DeviceContext`). The immediate context (`D3D11_DEVICE_CONTEXT_IMMEDIATE`) records and submits commands synchronously; deferred contexts (`D3D11_DEVICE_CONTEXT_DEFERRED`) record into command lists for later execution. Resource types gained explicit view objects: Shader Resource Views (`ID3D11ShaderResourceView`, SRV), Unordered Access Views (`ID3D11UnorderedAccessView`, UAV), Render Target Views (`ID3D11RenderTargetView`, RTV), and Depth Stencil Views (`ID3D11DepthStencilView`, DSV). Compute shaders (`ID3D11ComputeShader`) with UAVs enabled GPU compute via D3D11. D3D11 retains implicit synchronisation: the driver deduces all resource hazards without the application specifying barriers.

### D3D12: Explicit Everything

D3D12 represents a fundamental philosophical shift toward explicit GPU management. The application is responsible for:

- **Memory management**: `ID3D12Heap` for explicit heap creation; `CreateCommittedResource`, `CreatePlacedResource`, and `CreateReservedResource` for the three resource creation paths.
- **Command recording**: `ID3D12GraphicsCommandList` records draw and dispatch commands; lists are submitted to `ID3D12CommandQueue` (Direct, Compute, or Copy queue types). Command lists are not reusable once submitted.
- **Synchronisation**: `ID3D12ResourceBarrier` with `D3D12_RESOURCE_BARRIER_TYPE_TRANSITION` marks layout changes; `ID3D12Fence` with `Signal`/`Wait` synchronises between CPU and GPU or between queues.
- **Binding**: Root signatures (`ID3D12RootSignature`) define the binding contract between shaders and resources; descriptor heaps (`ID3D12DescriptorHeap`) are GPU-visible arrays of resource descriptors.

### Conceptual Mapping to Vulkan

The Vulkan concepts that correspond most closely to D3D12 are:

| D3D12 | Vulkan |
|---|---|
| `ID3D12Device` | `VkDevice` |
| `ID3D12CommandList` + `ID3D12CommandQueue` | `VkCommandBuffer` + `VkQueue` |
| `ID3D12RootSignature` | `VkPipelineLayout` + `VkDescriptorSetLayout` |
| `ID3D12DescriptorHeap` | `VkDescriptorPool` + `VkDescriptorSet` (or `VkBuffer` with `VK_EXT_descriptor_buffer`) |
| `ID3D12Fence` | `VkTimelineSemaphore` (via `VK_KHR_timeline_semaphore`) |
| `ID3D12Heap` | `VkDeviceMemory` |
| `D3D12_RESOURCE_BARRIER_TYPE_TRANSITION` | `VkImageMemoryBarrier2` / `VkBufferMemoryBarrier2` |
| DXR `D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE_*` | `VkAccelerationStructureKHR` |

For D3D11, the mapping is less direct because D3D11 is implicit and Vulkan is explicit:

| D3D11 | Vulkan |
|---|---|
| `ID3D11DeviceContext` (immediate) | `VkCommandBuffer` recording + `vkQueueSubmit` |
| `ID3D11DeviceContext` (deferred) | Recorded `VkCommandBuffer` |
| D3D11 implicit hazard tracking | Manual `VkPipelineBarrier` insertion |
| `ID3D11*View` (SRV/UAV/RTV/DSV) | `VkImageView` + `VkBufferView` + descriptor writes |

---

## 3. DXVK Architecture (D3D9/10/11 → Vulkan)

### Project Layout

The DXVK source tree is organised to reflect its multi-API scope. Under `src/`:

- `d3d8/` — D3D8 implementation (thin wrapper over D3D9)
- `d3d9/` — D3D9 COM interface implementation
- `d3d10/` — D3D10 COM interface implementation (thin wrapper over D3D11)
- `d3d11/` — D3D11 COM interface implementation
- `dxgi/` — DXGI swapchain and adapter enumeration
- `dxvk/` — The shared Vulkan backend used by all D3D versions
- `spirv/` — The SPIR-V module builder (`SpirvModule`)
- `vulkan/` — Vulkan loader and extension management
- `wsi/` — Window System Integration (Win32 HWND → `VkSurfaceKHR`)

The architecture is a layered stack: the COM interface layer in `d3d9/` / `d3d11/` receives D3D calls and translates them into calls on the `dxvk/` shared backend, which records Vulkan commands.

### Core Backend Objects

**`DxvkDevice`** wraps a `VkDevice`. It manages the physical device selection, queue family selection, and extension probing. It creates all `DxvkContext` instances and owns the `DxvkMemoryAllocator` and `DxvkPipelineManager`.

**`DxvkContext`** is the central recording object. It maintains the current render state (bound pipelines, vertex/index buffers, viewports, scissors, descriptor bindings) and translates D3D draw calls into Vulkan command buffer recording. D3D11's `ID3D11DeviceContext` (immediate mode) translates into direct calls on a `DxvkContext` associated with the graphics queue. D3D11 deferred contexts record into `DxvkCsChunk` objects that are replayed on the submission thread. [Source](https://deepwiki.com/doitsujin/dxvk/2.1-command-system)

`DxvkContext` uses a dirty-flag system with over 40 `DxvkContextFlag` values to defer Vulkan state updates until the next draw call. Lazy binding is critical to performance: a D3D game that changes a single sampler between draws should not flush and rebuild all descriptor sets if nothing else changed.

**`DxvkCommandList`** wraps multiple `VkCommandBuffer` objects with specialised roles:

- `ExecBuffer` — Primary graphics and compute commands on the graphics queue
- `InitBuffer` — Resource initialization commands (clears, layout transitions) on the graphics queue
- `SdmaBuffer` — Asynchronous transfer operations on a dedicated transfer (`SDMA`) queue
- `SdmaBarriers` — Pre-transfer barriers for the SDMA path

The separation of `SdmaBuffer` enables concurrent DMA transfers while the GPU executes rendering, matching what Windows drivers do transparently.

### Memory Management

DXVK uses a **custom memory allocator** (`DxvkMemoryAllocator`) rather than the AMD Vulkan Memory Allocator (VMA) library. The allocator implements a two-tier hierarchy:

- `DxvkPageAllocator` — Page-based allocation from `VkDeviceMemory` blocks
- `DxvkPoolAllocator` — Pool allocator sitting above the page layer, providing per-size-class pools for small allocations
- `DxvkLocalAllocationCache` — Thread-local cache for power-of-two allocation sizes
- `DxvkSharedAllocationCache` — Thread-safe cache for small shared allocations

Each active `VkDeviceMemory` block is tracked as a `DxvkMemoryChunk`. Resources are represented by `DxvkResourceAllocation`, a reference-counted object storing the Vulkan resource handle together with the memory sub-allocation that backs it. Version 2.5 (November 2024) overhauled this subsystem to add defragmentation, eviction, and relocation support, yielding VRAM savings of up to 1 GB in memory-heavy titles. [Source](https://en.wikipedia.org/wiki/DXVK)

GPU resources accessible to shaders are exposed as `DxvkBuffer` and `DxvkImage` objects that encapsulate the `VkBuffer`/`VkImage` handle together with a `DxvkResourceAllocation`.

### Pipeline Architecture

**`DxvkGraphicsPipeline`** manages all pipeline variants for a given set of shaders. Because D3D's pipeline state is not fully known until draw time (the render target format, blend state, depth-stencil state, and many other items can change between draws), DXVK must create multiple `VkPipeline` variants from the same shaders.

On drivers that support `VK_EXT_graphics_pipeline_library` (with the `IndependentInterpolationDecoration` feature — available on current AMD RADV, NVIDIA, and Intel ANV), DXVK compiles shaders into library stages at the time the D3D game calls `CreatePixelShader`/`CreateVertexShader`, rather than waiting for the first draw call that uses them. This decomposes the pipeline into:

- `DxvkGraphicsPipelineVertexInputLibrary` — Vertex input assembly state
- Pre-rasterization library stage (vertex + geometry/tessellation shaders)
- Fragment shader library stage
- `DxvkGraphicsPipelineFragmentOutputLibrary` — Render target and blend state

At draw time, the final pipeline is linked from pre-compiled library stages via `vkCreateGraphicsPipelines` with the `VK_PIPELINE_CREATE_LINK_TIME_OPTIMIZATION_BIT_EXT` or `VK_PIPELINE_CREATE_RETAIN_LINK_TIME_OPTIMIZATION_INFO_BIT_EXT` flags. Linking a pre-compiled library pipeline takes microseconds rather than the hundreds of milliseconds required for a full `VkPipeline` compile from SPIR-V. [Source](https://github.com/doitsujin/dxvk/issues/2798)

On drivers that do not support graphics pipeline libraries (older hardware), DXVK falls back to creating monolithic `VkPipeline` objects; in that path the state cache (Section 4) takes on more importance.

**`DxvkComputePipeline`** handles compute workloads from D3D11 `ID3D11ComputeShader` and D3D9 compute emulation.

### Resource Barrier Tracking

D3D11 and D3D9 have fully implicit synchronisation. DXVK must infer which pipeline barriers are needed. This is the responsibility of `DxvkBarrierTracker`, which maintains a two-part hash table backed by red-black trees (`DxvkBarrierTreeNode`) to track pending read and write access ranges on buffers and images. When `DxvkContext` binds a resource for a new access type that is incompatible with a pending access, it emits the required `vkCmdPipelineBarrier2` calls through `DxvkBarrierBatch` before the draw. [Source](https://raw.githubusercontent.com/doitsujin/dxvk/master/src/dxvk/dxvk_barrier.h)

---

## 4. The DXVK State Cache and Shader Compilation

### The Shader Stutter Problem

The fundamental challenge for any D3D-to-Vulkan translation layer is the mismatch between when pipeline state is known and when `VkPipeline` objects must be created. In Vulkan, a `VkPipeline` is an expensive, opaque GPU-compiled object whose creation can take anywhere from tens to hundreds of milliseconds. D3D applications do not expose this cost: a D3D11 game calls `CreatePixelShader` with DXBC bytecode, then draws with whatever state happens to be set at draw time. The first draw that exercises a novel state combination triggers a `VkPipeline` compile, causing a stutter or frame hitch visible to the user.

The D3D12 path (VKD3D-Proton) has a related but different problem: D3D12 applications call `CreatePipelineState` or use pipeline state objects, which are expensive but scheduled; the challenge there is the translation from DXIL to SPIR-V plus the Vulkan compile.

### The DXVK State Cache

For the D3D9/10/11 path without graphics pipeline library support, DXVK maintains a persistent on-disk state cache. The cache key is a hash of the shader bytecode identifiers plus the render state vector. Cached pipeline state is serialized to disk in:

- **Steam games**: `~/.steam/steam/steamapps/shadercache/<APPID>/DXVK_state_cache/` (when shader pre-caching is enabled)
- **Direct path**: Next to the game's `.exe` in the Wine prefix
- **Override**: Set `DXVK_STATE_CACHE_PATH=/some/directory` to redirect

On subsequent runs, DXVK reads the cache and pre-compiles pipelines on background threads before they are needed by the GPU, converting the stutter on first encounter into a background CPU load on subsequent runs. This was the primary anti-stutter mechanism in DXVK 1.x. [Source](https://github.com/doitsujin/dxvk/issues/3400)

### Graphics Pipeline Library: The Modern Solution

Since DXVK 2.0 (November 2022), the preferred anti-stutter mechanism is `VK_EXT_graphics_pipeline_library`. When this extension is available, DXVK compiles shader library stages eagerly when the game calls `Create*Shader`. The final pipeline link at draw time is fast enough that it does not cause visible stutter. On drivers supporting this extension:

- State cache files for new games are much smaller (tens to hundreds of entries, rather than thousands), because only pipelines that cannot use the library path (e.g., tessellation shaders in some cases) are cached.
- The `dxvk-async` patch (a third-party modification that compiled pipelines asynchronously on a worker thread, causing visual corruption with placeholder pipelines) is entirely superseded and unnecessary.

DXVK 2.7 (July 2025) further improved this path by rewriting the descriptor management layer using `VK_EXT_descriptor_buffer` on AMD RDNA and NVIDIA hardware, reducing CPU overhead in descriptor-heavy games like Final Fantasy XIV and God of War. [Source](https://www.linuxcompatible.org/story/dxvk-27-released/)

### Valve's Shader Pre-Caching

Valve's Steam platform provides a complementary mechanism: the Steam backend distributes pre-compiled shader caches for popular games. When a user first launches a title via Steam Play, Steam downloads a "shader cache archive" that pre-populates the DXVK state cache directory. This means the stutter reduction of the state cache applies from the very first play session, even before the user has personally encountered the shader combinations in that cache. This is separate from DXVK's internal mechanisms and is managed by the Steam client itself.

---

## 5. VKD3D-Proton Architecture (D3D12 → Vulkan)

### Divergence from Upstream vkd3d

The upstream `vkd3d` project (hosted by WineHQ at `https://source.winehq.org/git/vkd3d.git`) is the reference D3D12-on-Vulkan implementation used in standard Wine. VKD3D-Proton is a fork, begun by Hans-Kristian Arntzen, that prioritises performance and game compatibility over API stability and broad driver support. The explicit goals of the fork are stated in its README: "Backwards compatibility with the vkd3d standalone API is not a goal of this project." Developers are expected to use current GPU drivers. [Source](https://github.com/HansKristian-Work/vkd3d-proton)

Key differences from upstream include:

- Mandatory Vulkan 1.3 (upstream supports 1.1)
- Aggressive use of optional Vulkan extensions (descriptor buffer, ray tracing, mesh shaders) as they become available
- A dedicated `dxil-spirv` shader translation library rather than the `vkd3d-shader` library
- Production-grade game workarounds not appropriate for a reference implementation
- A unified DXBC frontend shared with DXVK (as of version 3.0, November 2025)

### Required Vulkan Extensions

VKD3D-Proton mandates:

- Vulkan 1.3 core (which includes `VK_KHR_dynamic_rendering`, `VK_KHR_synchronization2`, `VK_KHR_copy_commands2`, etc.)
- Descriptor indexing with at least 1,000,000 `UpdateAfterBind` descriptors for most descriptor types (except `UniformBuffer`)
- `samplerMirrorClampToEdge` and `shaderDrawParameters` device features
- `VK_EXT_robustness2` (for null descriptor support)
- `VK_KHR_push_descriptor`

Strongly recommended (and effectively required for current games):

- `VK_EXT_descriptor_buffer` — Replaces descriptor sets with buffer-backed descriptors, enabling the bindless heap model; adopted in version 2.8
- `VK_EXT_mutable_descriptor_type` — Allows reinterpreting descriptors across types
- `VK_KHR_buffer_device_address` — Required for root descriptor GPU virtual address mapping
- `VK_KHR_ray_tracing_pipeline` + `VK_KHR_acceleration_structure` — DXR support
- `VK_EXT_mesh_shader` — D3D12 mesh and amplification shader support, added in v2.7

Driver requirements at the time of writing:
- **AMD (RADV)**: Mesa 22.0 or newer
- **NVIDIA**: Series 535+ drivers (or Vulkan beta track for latest features)
- **Intel (ANV)**: Supported; recommended to use recent Mesa releases

### Root Signature → VkPipelineLayout

The `ID3D12RootSignature` defines the GPU binding contract for a D3D12 pipeline. VKD3D-Proton translates it into a `VkPipelineLayout` with associated `VkDescriptorSetLayout` objects. The mapping within the push constant block follows a three-zone layout:

1. **Root descriptors (CBV/SRV/UAV bound directly)**: One 64-bit buffer device address per root descriptor, using `VK_KHR_buffer_device_address` to represent the D3D12 GPU virtual address directly.
2. **Descriptor tables**: One 32-bit heap offset per descriptor table, indexing into the global descriptor heap buffer.
3. **Root constants**: Direct 32-bit values.

Static samplers from the root signature become immutable samplers in the `VkDescriptorSetLayout`. [Source](https://deepwiki.com/HansKristian-Work/vkd3d-proton/2.1-d3d12-to-vulkan-translation)

This layout means that `CmdSetGraphicsRootDescriptorTable` calls in D3D12 translate to cheap push constant updates rather than `vkUpdateDescriptorSets` calls, which is critical to low-latency command recording.

### Descriptor Heaps and Bindless Rendering

D3D12's descriptor heaps (`ID3D12DescriptorHeap`) are large GPU-visible arrays of resource descriptors. VKD3D-Proton implements these using `VK_EXT_descriptor_buffer` (when available). In this mode, the heap becomes a `VkBuffer` in GPU-accessible memory containing packed descriptor data laid out in the format the driver requires. Shader-visible descriptor access is entirely bindless: shaders index into the descriptor buffer using heap offsets carried in push constants, rather than through `VkDescriptorSet` binding. This matches D3D12's model very closely and achieves substantially lower CPU overhead than the legacy `VkDescriptorSet`-based path. [Source](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/proposals/VK_EXT_descriptor_buffer.adoc)

### Command Queue Mapping and Synchronisation

D3D12 exposes three queue types: `D3D12_COMMAND_LIST_TYPE_DIRECT` (Direct), `D3D12_COMMAND_LIST_TYPE_COMPUTE` (Compute), and `D3D12_COMMAND_LIST_TYPE_COPY` (Copy). VKD3D-Proton maps these to Vulkan queues from the appropriate queue families:

- Direct → Graphics queue (`VK_QUEUE_GRAPHICS_BIT`)
- Compute → Async compute queue (`VK_QUEUE_COMPUTE_BIT` from a separate family, when available)
- Copy → Transfer queue (`VK_QUEUE_TRANSFER_BIT`; NVIDIA uses a dedicated TRANSFER queue family by default in VKD3D-Proton 2.x+)

The `single_queue` configuration flag disables async compute and transfer queues for debugging or driver workarounds.

`ID3D12Fence` maps to Vulkan timeline semaphores (`VK_KHR_timeline_semaphore`, promoted to Vulkan 1.2 core). `Signal` on the fence becomes a `vkQueueSubmit` with a signal operation on the timeline semaphore; `Wait` becomes a `vkWaitSemaphores` call on the CPU or a submit-time wait on the GPU. Timeline semaphores model D3D12 fences exactly because they carry monotonically increasing counter values, unlike binary semaphores which model a more restricted signalling protocol.

A background worker thread runs `vkWaitSemaphores` on a submission timeline semaphore and releases resources (command pools, staging buffers) once their GPU work completes, implementing D3D12's reference-counting semantics without per-frame synchronisation points. [Source](https://deepwiki.com/HansKristian-Work/vkd3d-proton)

---

## 6. Shader Translation: DXBC/DXIL → SPIR-V

### D3D11 Shaders: DXBC

Direct3D 10/11 shaders are compiled offline by the `fxc.exe` compiler into DXBC (DirectX Bytecode). (The `dxc.exe` compiler produces DXIL, not DXBC; it is used for Shader Model 6.0+ targets as described in the DXIL section below.) DXBC is a container format with tagged chunks:

- `SHDR` / `SHEX` — The shader bytecode stream
- `RDEF` — Resource definition (constant buffers, texture bindings)
- `ISGN` / `OSGN` — Input/output signature tables
- `STAT` — Statistics (instruction count, etc.)

DXVK implements a complete DXBC → SPIR-V compiler internally. The translation path is:

```
D3D bytecode (DXBC container)
    → DxbcModule (chunk parsing, src/dxbc/dxbc_reader.h)
    → DxbcCompiler (opcode-by-opcode translation, src/dxbc/dxbc_compiler.cpp)
    → SpirvModule (SPIR-V binary builder, src/spirv/spirv_module.cpp)
    → VkShaderModule (via vkCreateShaderModule)
```

`DxbcCompiler` iterates over DXBC instructions and emits corresponding SPIR-V opcodes into a `SpirvModule`. Each DXBC instruction maps to one or more SPIR-V instructions. `DxbcCompiler` handles all shader types (vertex, hull, domain, geometry, pixel, compute) and Shader Models 4.0 through 5.1. [Source](https://codeberg.org/redstrate/dxbc)

DXVK's DXBC compiler does **not** use SPIRV-Cross. SPIRV-Cross performs the inverse transformation (SPIR-V → GLSL/HLSL/MSL) and is not in this path. [Source](https://github.com/KhronosGroup/SPIRV-Cross)

Starting with VKD3D-Proton 3.0 (November 2025), VKD3D-Proton shares the same DXBC frontend as DXVK, providing a clean IR representation of DXBC instructions before translation to SPIR-V. This unified frontend replaced the legacy `vkd3d-shader` DXBC path and immediately fixed compatibility with titles that were completely broken under the old backend, most notably Red Dead Redemption 2 in D3D12 mode. [Source](https://github.com/HansKristian-Work/vkd3d-proton/releases/tag/v3.0)

### D3D12 Shaders: DXIL

D3D12 shaders (Shader Model 6.0+) are compiled by the `dxc.exe` compiler into DXIL (DirectX Intermediate Language). DXIL is LLVM 3.7 bitcode with a specialised resource system:

- DXIL encodes GPU resource access as `dx.op` intrinsic calls: more than 200 opcodes encoded as the first constant argument in calls to external functions.
- Vectors are decomposed into scalars (unlike SPIR-V's native vector types), which makes reconstruction of vectorised operations during translation non-trivial.
- Stage I/O, resource bindings, and entry points are described via LLVM metadata nodes (`!dx.valver`, `!dx.shaderModel`, etc.) rather than instructions.
- The DXIL container adds a `DXIL` chunk containing the LLVM bitcode, plus `RDAT` (resource and dependency data) and `ILDB` (debug information) chunks.

VKD3D-Proton uses the **dxil-spirv** library for DXIL → SPIR-V translation. `dxil-spirv` is a separate project by Hans-Kristian Arntzen at [github.com/HansKristian-Work/dxil-spirv](https://github.com/HansKristian-Work/dxil-spirv). It implements a custom LLVM 3.7 bitcode parser (avoiding a 40+ MB vendored LLVM dependency) and converts `dx.op` intrinsic calls to their SPIR-V equivalents. [Source](https://themaister.net/blog/2021/09/05/my-personal-hell-of-translating-dxil-to-spir-v-part-1/)

The key translation challenges in dxil-spirv are:

1. **Unstructured control flow**: DXIL uses SSA form with unrestricted `br` instructions, while SPIR-V requires structured control flow (explicit merge blocks for `OpSelectionMerge` / `OpLoopMerge`). Recovering structure from arbitrary LLVM CFGs is the most difficult problem, with new edge cases encountered regularly during compatibility testing against AAA titles.

2. **Virtualized resources**: HLSL's resource access model allows about eight different codegen strategies depending on local and global root signatures (in the D3D12 ray tracing path). Each must be handled correctly.

3. **Scalar-to-vector reconstruction**: DXIL decomposes vectors; SPIR-V works natively with vectors. dxil-spirv must reconstruct `OpFAdd` (vector) from multiple scalar `dx.op` additions.

### Shader Model 6.x Features

D3D12 with Shader Model 6.x introduces several GPU programming primitives that DXIL → SPIR-V must translate to Vulkan equivalents:

| DXIL / SM6.x Feature | SPIR-V / Vulkan Mapping |
|---|---|
| `WaveActiveSum`, `WaveActiveBallot` | `OpGroupFAdd`, `OpGroupBallotKHR` via `GL_KHR_shader_subgroup_*` |
| `WaveGetLaneIndex`, `WaveGetLaneCount` | `SubgroupLocalInvocationId`, `SubgroupSize` |
| `CallShader`, `TraceRay` (DXR) | `OpExecuteCallableKHR`, `OpTraceRayKHR` |
| Mesh shader `DispatchMesh` | `OpEmitMeshTasksEXT` via `VK_EXT_mesh_shader` |
| `SM6.6` `ResourceDescriptorHeap[]` | Bindless descriptor indexing via `VK_EXT_descriptor_buffer` |
| `SM6.x` cooperative matrix (FSR4) | `VK_KHR_cooperative_matrix` + `VK_KHR_shader_float8` (VKD3D-Proton 3.0) |

Wave intrinsics (`WaveActive*`, `WavePrefixSum`) map to Vulkan subgroup operations controlled by `VK_EXT_subgroup_size_control` and `VK_KHR_shader_subgroup_extended_types`. The subgroup size can matter for correctness when games assume specific wave widths (32 on AMD RDNA, 32 or 64 depending on workload; 32 on NVIDIA; 8–32 on Intel Xe). [Source](https://github.com/HansKristian-Work/dxil-spirv)

---

## 7. Proton Integration

### What Proton Is

Proton is Valve's open-source compatibility tool for Steam Play. It packages:

1. **Wine** — the Windows API compatibility layer (kernel32.dll, user32.dll, ntdll.dll, etc.)
2. **DXVK** — D3D8/9/10/11 → Vulkan translation (ships as `d3d8.dll`, `d3d9.dll`, `d3d10core.dll`, `d3d11.dll`, `dxgi.dll` — Windows PE DLLs loaded by Wine)
3. **VKD3D-Proton** — D3D12 → Vulkan translation (ships as `d3d12.dll` and `d3d12core.dll` — Windows PE DLLs)
4. **Steam Linux Runtime** — A container environment providing a consistent Linux library stack

Proton is hosted at [github.com/ValveSoftware/Proton](https://github.com/ValveSoftware/Proton). Each Proton version (Proton 8, 9, 10, Experimental) bundles specific DXVK and VKD3D-Proton releases.

### The Steam Linux Runtime Container

Rather than relying on the user's host distribution libraries, Proton games run inside a Steam Linux Runtime container. Container versions are:

- **Steam Linux Runtime 1.0 ("Scout")** — Ubuntu 12.04 library stack; legacy, used for old native Linux games
- **Steam Linux Runtime 2.0 ("Soldier")** — Debian 10 library stack; used for Proton 5.x–7.x
- **Steam Linux Runtime 3.0 ("Sniper")** — Debian 11 (Bullseye) library stack; used for Proton 8.0 through 10.0 and current native Linux games; App ID 1628350
- **Steam Linux Runtime 4.0** — Debian 13 library stack; launched November 2025, skipping Debian 12

The container uses Linux namespaces (mount namespace) to replace `/usr` with the runtime's library tree, ensuring that `libvulkan.so`, `libGL.so`, and other graphics libraries come from the Steam Runtime rather than the host. This means the ICD loader, the Vulkan layers, and the Mesa or NVIDIA Vulkan ICDs loaded inside the container are from a known, tested environment. [Source](https://steamcommunity.com/app/221410/discussions/8/4038104984938159075/)

### DLL Override Mechanism

Wine implements Windows DLL loading semantics. When a D3D11 game calls `LoadLibrary("d3d11.dll")`, Wine searches the DLL load path according to the `WINEDLLOVERRIDES` environment variable. The override syntax is:

```bash
# Format: DLL_NAME=load_order[,load_order...]
# n = native (load the actual PE DLL from the Wine prefix or path)
# b = builtin (use Wine's built-in implementation)

# Use DXVK's native d3d11.dll instead of Wine's built-in
export WINEDLLOVERRIDES="d3d9=n,b;d3d11=n,b;dxgi=n,b"
```

Proton sets these overrides automatically. When DXVK's `d3d11.dll` (a Windows PE binary compiled with Mingw-w64) is placed in the Wine prefix's `system32` directory, the `n` (native) load order causes Wine to load DXVK's implementation instead of its own built-in `d3d11` translation. VKD3D-Proton's `d3d12.dll` is loaded by the same mechanism.

### Environment Variables and Debugging

Key runtime controls:

```bash
# DXVK overlay: show fps, GPU info, etc.
DXVK_HUD=fps,devinfo,frametimes,memory,drawcalls,pipelines

# DXVK logging
DXVK_LOG_LEVEL=debug          # error|warn|info|debug
DXVK_LOG_PATH=/tmp/dxvk-logs  # directory for log files

# VKD3D-Proton debug logging
VKD3D_DEBUG=warn               # none|err|warn|info|trace

# VKD3D-Proton feature flags
VKD3D_CONFIG=dxr               # force-enable DXR (now default)
VKD3D_CONFIG=nodxr             # disable DXR
VKD3D_CONFIG=single_queue      # disable async compute/transfer queues
VKD3D_CONFIG=no_upload_hvv     # avoid host-visible VRAM for upload heap

# General Proton debug logging
PROTON_LOG=1                   # enable Wine+Proton logging
WINEDEBUG=+d3d12,+vkd3d        # Wine debug channels
```

`DXVK_HUD` is the quickest way to confirm DXVK is active: the HUD displays the API in use (`D3D11` or `D3D9`), the GPU name, and the pipeline compiler status icon.

### Steam Play and Per-Game Proton Selection

Steam Play allows users to select a specific Proton version per game via the game's Properties → Compatibility menu. The ProtonDB community database at [protondb.com](https://www.protondb.com) records community-sourced compatibility reports with per-game Proton version recommendations, launch options, and known issues. It is the primary reference for "which Proton version works best for this game."

---

## 8. Performance Considerations

### Synchronisation Translation: D3D11

D3D11's implicit synchronisation model (the driver tracks all hazards) translates to explicit Vulkan barriers via `DxvkBarrierTracker`. Because the tracker must conservatively insert barriers whenever a resource transitions between read and write access, the barrier overhead can be higher than what a native Vulkan application would produce. DXVK mitigates this with several strategies:

- **Lazy barrier insertion**: Barriers are batched and inserted only immediately before a draw or dispatch that requires them, never speculatively.
- **Access type merging**: If the same resource is read by multiple shader stages between writes, only one barrier is inserted for the full combined source/destination stage mask, via `VkDependencyInfo` with pipeline stage masks that cover all reader stages.
- **UAV barrier relaxation**: By default, DXVK inserts UAV write barriers between consecutive UAV accesses. The `dxvk.conf` file allows disabling this for titles where it's safe (`relaxedBarriers = True`), at the risk of rendering artifacts.

### Synchronisation Translation: D3D12

D3D12's explicit `ID3D12ResourceBarrier` transitions map directly to Vulkan image and buffer memory barriers. However, D3D12 describes barriers in terms of resource states (`D3D12_RESOURCE_STATE_RENDER_TARGET`, `D3D12_RESOURCE_STATE_UNORDERED_ACCESS`, etc.) while Vulkan requires explicit image layouts and access masks. VKD3D-Proton maintains a per-resource state machine that tracks the current D3D12 resource state and maps transitions to Vulkan layout transitions and memory barriers using `VkImageMemoryBarrier2` / `VkBufferMemoryBarrier2` (via `VK_KHR_synchronization2`, core in Vulkan 1.3). VKD3D-Proton 3.0 added `VK_KHR_unified_image_layouts` support to reduce redundant barrier overhead on drivers that support it. [Source](https://github.com/HansKristian-Work/vkd3d-proton/releases/tag/v3.0)

### Async Compute Queue Usage

D3D12 applications that use `ID3D12CommandQueue` with `D3D12_COMMAND_LIST_TYPE_COMPUTE` expect true concurrent execution with the Direct queue. VKD3D-Proton maps this to a separate Vulkan compute queue from a distinct queue family, enabling hardware-level async compute overlap. On AMD RDNA hardware with RADV, async compute is used for tasks like shadow map generation and particle simulation that are interleaved with rasterization workloads. Copy queues use a TRANSFER family queue (`VK_QUEUE_TRANSFER_BIT`) — on NVIDIA this is the dedicated DMA engine exposed as a `TRANSFER`-only queue family, which VKD3D-Proton uses by default to avoid blocking the compute queue with asset streaming transfers. [Source](https://gamingonlinux.com/2026/05/vkd3d-proton-3-0-1-brings-many-linux-gaming-enhancements-for-direct3d-12-via-vulkan/)

### Memory Management and ResizableBAR

D3D12's three heap types (`D3D12_HEAP_TYPE_DEFAULT`, `D3D12_HEAP_TYPE_UPLOAD`, `D3D12_HEAP_TYPE_READBACK`) map to Vulkan memory types based on the `VkMemoryPropertyFlags`:

- `DEFAULT` (GPU-only) → `VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT`
- `UPLOAD` (CPU→GPU streaming) → `VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`
- `READBACK` (GPU→CPU results) → `VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_CACHED_BIT`

When Resizable BAR (ReBAR) / AMD Smart Access Memory (SAM) is enabled, the GPU exposes all of its VRAM as CPU-accessible (`VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT | VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`). VKD3D-Proton can use this memory for `UPLOAD` heaps (`VKD3D_CONFIG=upload_hvv`) to eliminate CPU-to-GPU copy bandwidth for streaming assets. However, using ReBAR for upload heaps can increase CPU cache pressure on some workloads; `VKD3D_CONFIG=no_upload_hvv` disables this if needed. [Source](https://github.com/HansKristian-Work/vkd3d-proton/issues/1406)

VKD3D-Proton uses its own custom memory allocator (`vkd3d_memory_allocator`) that suballocates from `VkDeviceMemory` blocks tracked as `vkd3d_memory_chunk` structures, avoiding per-resource `VkDeviceMemory` allocations (which have a per-device limit on most drivers). [Source](https://deepwiki.com/HansKristian-Work/vkd3d-proton/3.3-resources-and-heaps)

### Driver Performance Characteristics

- **DXVK + RADV**: The combination that most DXVK development is tuned against. RADV's ACO compiler (Chapter 15) provides fast shader compilation times that make the pipeline library approach effective. RADV's aggressive async compute support benefits VKD3D-Proton workloads.
- **DXVK + ANV (Intel)**: Well-supported; Intel's graphics pipeline library implementation is mature. Lower VRAM budgets on integrated graphics require careful memory management.
- **DXVK + NVK (NVIDIA open-source)**: NVK — the Mesa open-source Vulkan driver for NVIDIA GPUs (Chapter 18) — is a supported DXVK target. For gaming, users more commonly use NVIDIA's proprietary Vulkan driver, which has its own well-optimised pipeline library path.

---

## 9. Ray Tracing Translation

### DXR in VKD3D-Proton

DirectX Raytracing (DXR) is a D3D12 feature that adds hardware-accelerated ray traversal. VKD3D-Proton maps DXR to `VK_KHR_ray_tracing_pipeline` (for ray tracing pipelines) and `VK_KHR_acceleration_structure` (for BVH management). DXR support was first added experimentally in VKD3D-Proton 2.3 and became enabled by default in VKD3D-Proton 2.11. [Source](https://gamingonlinux.com/2023/11/vkd3d-proton-211-released-with-directx-raytracing-enabled-by-default/)

### Acceleration Structure Translation

D3D12 acceleration structures (TLAS and BLAS) are managed through `ID3D12GraphicsCommandList4::BuildRaytracingAccelerationStructure`. This call specifies geometry (triangle or procedural), source/destination scratch buffers (all as `D3D12_GPU_VIRTUAL_ADDRESS`), and build flags. VKD3D-Proton translates this to:

```cpp
// D3D12:
commandList->BuildRaytracingAccelerationStructure(&desc, 0, nullptr);

// VKD3D-Proton translates to (approximately):
vkCmdBuildAccelerationStructuresKHR(
    vkCommandBuffer,
    1,                        // infoCount
    &accelerationStructureBuildGeometryInfo,
    &ppBuildRangeInfos
);
```

D3D12 acceleration structures are `D3D12_GPU_VIRTUAL_ADDRESS` values pointing into user-allocated buffers. In VKD3D-Proton, the corresponding `VkAccelerationStructureKHR` is created from a sub-range of the translated `VkBuffer` that backs the D3D12 resource. `VK_KHR_buffer_device_address` is essential here: D3D12's GPU virtual address model maps directly to Vulkan buffer device addresses. [Source](https://github.com/HansKristian-Work/vkd3d-proton/pull/486)

VKD3D-Proton 3.0 added experimental DXR 1.2 support when `VK_EXT_opacity_micromap` is available, enabling per-microgeometry opacity for more accurate transparent surface ray tests.

### Shader Binding Table Translation

DXR defines a **shader binding table** (SBT) as a GPU buffer containing records for each shader type:

- Ray generation shader record
- Miss shader records (one per miss shader)
- Hit group records (one per geometry, containing closest-hit, any-hit, and intersection shaders)
- Callable shader records

D3D12 passes the SBT to `ID3D12GraphicsCommandList4::DispatchRays` as `D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE` structs — base address, stride, and size for each section.

The Vulkan equivalent is the Shader Binding Table passed to `vkCmdTraceRaysKHR` as `VkStridedDeviceAddressRegionKHR` structs. The structure is identical conceptually:

```cpp
// VKD3D-Proton translates DispatchRays to:
vkCmdTraceRaysKHR(
    vkCommandBuffer,
    &raygenShaderBindingTable,    // VkStridedDeviceAddressRegionKHR
    &missShaderBindingTable,
    &hitShaderBindingTable,
    &callableShaderBindingTable,
    width, height, depth          // dispatch dimensions
);
```

The main translation complexity is in the shader handles: D3D12 uses 32-byte opaque shader identifiers (from `ID3D12StateObjectProperties::GetShaderIdentifier`); Vulkan uses `VK_KHR_ray_tracing_pipeline`'s `vkGetRayTracingShaderGroupHandlesKHR` to obtain handles of `shaderGroupHandleSize` bytes. VKD3D-Proton maps the D3D12 handle space to Vulkan handles through the pipeline state object translation. [Source](https://developer.nvidia.com/blog/bringing-hlsl-ray-tracing-to-vulkan/)

### Hardware Support and Game Compatibility

At time of writing, `VK_KHR_ray_tracing_pipeline` is supported on:

- AMD RDNA2+ (RX 6000 series and later) via RADV
- NVIDIA Turing+ (RTX 2000 series and later) via the NVIDIA proprietary Vulkan driver and (in development) NVK
- Intel Arc DG2+ (Alchemist) via ANV

Games known to use DXR via VKD3D-Proton include:

- **Cyberpunk 2077** — Hardware ray tracing reflections, global illumination (path tracing mode with RT Overdrive)
- **Control** — DXR reflections and shadows via Remedy's Northlight engine
- **Metro Exodus Enhanced Edition** — Full path tracing mode (the Enhanced Edition requires DXR)
- **Dying Light 2** — DXR global illumination
- **Microsoft Flight Simulator** — DXR reflections and lighting

DXR 1.1 indirect ray dispatch (`DispatchRays` indirect via `ExecuteIndirect`) has limitations in mapping to `vkCmdTraceRaysIndirectKHR` in some configurations; `VKD3D_CONFIG=nodxr12` disables DXR 1.2-specific features for games that expose compatibility issues.

---

## 10. Debugging and Development

### RenderDoc Integration

RenderDoc ([renderdoc.org](https://renderdoc.org)) can capture frames from DXVK and VKD3D-Proton applications because both operate at the Vulkan level below the D3D API surface. RenderDoc captures `VkCommandBuffer` submissions, so the captured frame shows Vulkan calls, not D3D calls. This is useful for diagnosing rendering artifacts at the Vulkan level.

To capture a Proton game with RenderDoc on Linux:

```bash
# Launch with RenderDoc environment injection
PROTON_USE_RENDERDOC=1 %command%
```

This flag causes Proton to load the RenderDoc Vulkan layer inside the container. The RenderDoc UI can then connect to the running process or receive captures via the in-process API.

For non-Steam Wine environments, RenderDoc requires that `DXVK_LOG_LEVEL=none` is not set (logging must be at a non-null level) and that the RenderDoc Vulkan layer is in the layer search path via `VK_INSTANCE_LAYERS=VK_LAYER_RENDERDOC_Capture` or `VK_LAYER_PATH`. The `RENDERDOC_HOOK_EGL=0` workaround mentioned in some documentation is relevant on Linux EGL paths (for example ANGLE-based renderers); for DXVK's WSI path it is generally not needed.

### Apitrace Capture (D3D-Level)

To capture D3D API calls (rather than Vulkan calls), use `apitrace`:

```bash
# Determine game architecture
file "game.exe"
# "PE32 executable" = 32-bit, use x32 DLLs
# "PE32+ executable" = 64-bit, use x64 DLLs

# Place apitrace DLL wrappers (d3d9.dll, d3d11.dll, dxgi.dll)
# in the same directory as the game executable.
# The wrapper intercepts D3D calls before DXVK sees them.

# The .trace file appears on the Wine prefix Desktop.
```

The trace file captures all D3D API calls at the game's level, before DXVK translation. This is essential for diagnosing bugs where the game's D3D usage is incorrect rather than DXVK's translation being wrong. Trace files are often several gigabytes. [Source](https://github.com/doitsujin/dxvk/wiki/Using-Apitrace)

### Building DXVK from Source

```bash
# Prerequisites: meson, mingw-w64, glslang
git clone https://github.com/doitsujin/dxvk.git
cd dxvk

# Cross-compile for Windows (producing PE .dll files)
./package-release.sh master /tmp/dxvk-build

# The output appears in /tmp/dxvk-build/dxvk-master/
# Containing x64/ and x32/ subdirectories with the PE DLLs
```

DXVK is built as Windows PE DLLs using the Mingw-w64 cross-compiler. This is required because Wine loads them as if they were native Windows DLLs. The result is `.dll` files (true Windows PE format) rather than Linux `.so` files.

### Building VKD3D-Proton from Source

```bash
git clone --recursive https://github.com/HansKristian-Work/vkd3d-proton.git
cd vkd3d-proton

# Build requires wine-development or mingw-w64, meson, and LLVM for dxil-spirv
./package-release.sh master /tmp/vkd3d-build --no-package

# Output: x64/ directory with d3d12.dll and d3d12core.dll
```

VKD3D-Proton links the `dxil-spirv` submodule, which contains its own LLVM 3.7 bitcode parser — the build does not require a system LLVM installation.

### Contribution and Community

**DXVK**:
- Issues and patches: [github.com/doitsujin/dxvk/issues](https://github.com/doitsujin/dxvk/issues)
- Development discussion: `#dxvk` on Libera.Chat IRC

**VKD3D-Proton**:
- Issues: [github.com/HansKristian-Work/vkd3d-proton/issues](https://github.com/HansKristian-Work/vkd3d-proton/issues)
- Community: `#proton-developers` on the Proton Discord server
- Phoronix and GamingOnLinux regularly cover releases with technical summaries

When reporting bugs to either project, the key inputs are:
1. DXVK/VKD3D-Proton version (from `DXVK_HUD=version` or build tag)
2. GPU vendor and driver version (`VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation` can identify the driver)
3. `DXVK_LOG_LEVEL=debug` or `VKD3D_DEBUG=trace` output
4. If possible, an apitrace capture (for DXVK) or a VKD3D debug trace covering the reproduction

---

## 11. Fossilize and the Steam Shader Pre-Caching Pipeline

Section 4 of this chapter describes the per-game `.dxvk-cache` file and how `VK_EXT_graphics_pipeline_library` (GPL) eliminates most shader stutter by compiling shader stages at `Create*Shader` time rather than at draw time. Those mechanisms operate entirely within a single game session on a single machine. Fossilize and Steam's pre-caching pipeline solve an orthogonal problem: making the Vulkan-level pipeline compilation happen **before** the game is launched for the first time, across the entire Linux Steam user population. This section describes that system from capture layer to driver replay.

### The Residual Stutter Problem

Even with GPL, a game's first launch on a new machine incurs Vulkan pipeline link and backend-compile costs for every unique pipeline state. On the D3D12 path (VKD3D-Proton), a single complex title can require thousands of `vkCreateGraphicsPipelines` calls during loading screens; even with GPL's fast linking, the aggregate compile time can reach seconds to minutes of CPU saturation. The DXVK state cache and VKD3D-Proton's pipeline library state (described in Sections 4 and 5) accumulate as a user plays, but they do nothing on a pristine install. The problem is therefore fundamentally a "cold start" problem: a player's first session is always worst-case.

The DXVK `.dxvk-cache` stores D3D-to-SPIR-V state: it records which shader bytecode identifiers were combined with which render-state vectors, so that `DxvkPipelineManager` can start background `VkPipeline` creation before the GPU needs them on the next run. It does **not** store driver-compiled GPU binaries; those live in the driver's own disk cache (Mesa's shader disk cache, or NVIDIA's pipeline cache). The DXVK cache is therefore driver-version-agnostic — it is equally useful whether Mesa 22 or Mesa 25 is installed — but it requires at least one full play session to populate. A new player who installs a game and expects smooth first-run performance gets no benefit from another player's DXVK cache.

Fossilize is the answer to this gap. It captures Vulkan-level pipeline state (not D3D-level state) and allows Steam to distribute that state and pre-compile it for every player's specific GPU driver before the game starts. The two systems are complementary: the DXVK cache handles the D3D→SPIR-V translation layer; Fossilize handles the SPIR-V→driver-binary layer.

### The Fossilize Library and Capture Layer

Fossilize is an open-source Valve project hosted at [github.com/ValveSoftware/Fossilize](https://github.com/ValveSoftware/Fossilize). It provides two things: a **serialization library** and a **Vulkan implicit layer** named `VK_LAYER_fossilize`.

The layer intercepts six categories of Vulkan creation calls by hooking into the Vulkan dispatch table:

- `vkCreateSampler` → serializes `VkSamplerCreateInfo`
- `vkCreateDescriptorSetLayout` → serializes `VkDescriptorSetLayoutCreateInfo`
- `vkCreatePipelineLayout` → serializes `VkPipelineLayoutCreateInfo`
- `vkCreateRenderPass` / `vkCreateRenderPass2` → serializes `VkRenderPassCreateInfo`
- `vkCreateShaderModule` → serializes the SPIR-V bytecode blob
- `vkCreateGraphicsPipelines` / `vkCreateComputePipelines` / `vkCreateRayTracingPipelinesKHR` → serializes the full `Vk*PipelineCreateInfo` with all handle references replaced by their Fossilize hashes

The critical design point is that a `VkPipeline` references many other Vulkan objects — a `VkPipelineLayout` that itself references `VkDescriptorSetLayout` objects, `VkRenderPass`, and `VkShaderModule` objects. In the serialized database, each of these objects is identified by a **Fossilize hash**: a 64-bit content-addressed hash computed from the complete `CreateInfo` chain of that object and all objects it transitively depends on. A `VkGraphicsPipelineCreateInfo` is therefore stored as a single record whose dependencies are referenced only by their hashes; replaying that record on a different machine requires first replaying all depended-upon records, which the Fossilize replayer resolves automatically. [Source](https://github.com/ValveSoftware/Fossilize/blob/master/README.md)

The layer is enabled either by the standard Vulkan implicit layer mechanism (the layer JSON registers it globally) or, within Proton, by Valve's Steam integration code which arranges the layer's presence before the Wine process starts.

### The .foz Database Format

Fossilize records are stored in a bespoke binary container with the `.foz` extension. The format is designed for robustness in the face of abrupt process termination (which is common when capturing production games through a background layer):

- A database file consists of a stream of fixed-size **header blocks** interleaved with variable-length **data blobs**.
- Each entry in the database is identified by a resource tag (the Vulkan object type, encoded as an enum: sampler, descriptor set layout, pipeline layout, render pass, shader module, or pipeline) and its Fossilize hash.
- The payload for `VkShaderModule` entries is varint-encoded and deflate-compressed SPIR-V. The payload for pipeline and other `CreateInfo` entries is deflate-compressed JSON representing the complete `Vk*CreateInfo` structure with handle references replaced by their hashes.
- An accompanying **index file** (`filename_idx.foz`) records the byte offset and size of each entry within the data file, enabling O(1) lookup by (tag, hash) without scanning the entire blob. The pair `(filename.foz, filename_idx.foz)` constitutes a complete Fossilize database.

The database format supports concurrent writers (multiple game sessions appending to the same database) through a locking protocol and is designed to never lose previously-written entries even if a write is cut short. [Source](https://github.com/ValveSoftware/Fossilize/blob/master/README.md)

A Fossilize database captured from a running game via `VK_LAYER_fossilize` grows without bound during a session; Valve's distribution databases are curated extracts of a representative set of pipeline states that cover the title's typical rendering paths.

### The Steam Pre-Compilation Workflow

Steam's use of Fossilize follows a four-stage pipeline:

**1. Developer capture.** The game developer (or Valve, using internal test runs) runs the game with `VK_LAYER_fossilize` active and captures a representative Fossilize database — one that covers the shader states exercised by the game's levels, cutscenes, and common gameplay scenarios. The result is a `.foz` file containing the `VkShaderModule` SPIR-V blobs and `VkGraphicsPipelineCreateInfo` records for all captured pipelines.

**2. Valve distribution upload.** The developer submits the Fossilize database to Valve via the Steam partner portal. Valve hosts these databases on its CDN. The database is game-specific and keyed to a particular Proton/DXVK/VKD3D-Proton version (since a change in the translation layer changes the SPIR-V emitted for a given D3D shader, invalidating old databases).

**3. Steam pre-compilation on the user's machine.** When a user installs or updates a game that has a Fossilize database available, the Steam client downloads it into:

```
~/.steam/steam/steamapps/shadercache/<APPID>/fozpipelinesv6/
```

This directory contains the downloaded database pairs (`.foz` + `_idx.foz`). Steam then invokes `fossilize-replay` — its bundled Fossilize replay binary — to drive the local GPU driver to compile every pipeline in the database. This is the "Preparing Shader Pre-Caches" step visible in the Steam game download queue. The replay runs `fossilize-replay` with the local Vulkan device, which calls `vkCreateGraphicsPipelines` for each record. The driver's response to those calls is to compile the SPIR-V into GPU-specific machine code and store it in the Mesa disk cache (or the NVIDIA/Intel proprietary cache).

**4. Driver cache population and per-driver invalidation.** The driver identifies compiled pipeline binaries via the cache key defined by `vkCreatePipelineCache`: a header containing `vendorID`, `deviceID`, `driverVersion`, and `pipelineCacheUUID` from `VkPhysicalDeviceProperties`. When the user installs a new Mesa release (changing `driverVersion` and `pipelineCacheUUID`), the compiled binaries are no longer valid — they will not be found in the driver cache on lookup — so the pre-compilation step must be re-run against the new driver. Steam detects driver version changes and triggers a re-run of `fossilize-replay` automatically. [Source](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/11485)

The Mesa side of the driver cache uses `MESA_DISK_CACHE_SINGLE_FILE=1` to consolidate compiled shader binaries into a single-file Fossilize DB format internally (Mesa's own `.foz`-format disk cache), and `MESA_DISK_CACHE_READ_ONLY_FOZ_DBS` to mount additional pre-compiled databases as read-only overlays. Steam sets these variables in the Proton environment so that the pre-compiled cache from `shadercache/fozpipelinesv6/` is visible to the driver when the game actually launches. [Source](https://www.phoronix.com/news/Mesa-Single-File-Cache)

### RADV Integration with Fossilize

On AMD hardware with the RADV driver, Fossilize replay exercises RADV's full pipeline compilation path. When `fossilize-replay` calls `vkCreateGraphicsPipelines`, RADV:

1. Deserializes the `VkGraphicsPipelineCreateInfo` and resolves shader modules to their SPIR-V.
2. Runs the SPIR-V through the Mesa NIR translation frontend (`spirv_to_nir`).
3. Compiles the NIR through the ACO backend (RADV's default shader compiler since Mesa 20.2) to produce RDNA or GCN machine code. The legacy LLVM backend remains available but is not the default path.
4. Stores the compiled binary in the Mesa disk cache, keyed by the pipeline hash.

When the game subsequently calls `vkCreateGraphicsPipelines` for the same pipeline during a real game session, the driver finds the pre-compiled binary in the disk cache, avoiding a full compile. The end result is that the first-launch compile cost is paid during the "Preparing Shader Pre-Caches" step before the game loads, rather than during gameplay.

RADV's `VK_EXT_graphics_pipeline_library` support (enabled by default since Mesa 23.1 [Source](https://www.gamingonlinux.com/2023/05/mesa-graphics-drivers-23-1-0-out-now-with-radv-gpl-enabled/)) interacts with Fossilize replay in an important way: Fossilize can capture and replay GPL pipeline libraries (pre-rasterization shader stages, fragment shader stages) as independent entries, not only monolithic pipelines. This means the replay can pre-compile the shader-compilation-expensive GPL stages independently of the final fast link step, yielding a large reduction in the number of distinct GPU binary compilations required for a given game. The final pipeline-link step at game runtime is then a cheap sub-millisecond operation regardless of whether the pipelines were pre-cached.

One concrete performance example: for a Ghostwire Tokyo Fossilize capture containing ray-tracing pipelines from an Unreal Engine 4 title, RADV optimizations that allowed any-hit and intersection shaders to be compiled separately (eliminating forced inlining into one large megashader) reduced the Fossilize replay time from four minutes and twenty seconds to twenty seconds on the same hardware. [Source](https://www.phoronix.com/news/RADV-10x-Fast-RT-Pipeline-Comp) The user-visible benefit is that the "Preparing Shader Pre-Caches" wait on Steam is dramatically shorter for ray-tracing titles on RDNA hardware.

### DXVK State Cache vs. Fossilize: Complementary Layers

The two caching systems operate at different levels of the translation stack and serve different purposes. Understanding their relationship prevents confusion about which file to share or delete when diagnosing stutter issues.

**DXVK state cache (`.dxvk-cache`)**:
- Records which D3D11/D3D9 shader hash + render-state vector combinations have been encountered during gameplay.
- The cache content is SPIR-V, produced by DXVK's DXBC→SPIR-V compiler (`DxbcCompiler` / `SpirvModule`).
- DXVK's `dxvk_pipelineWorker` thread reads the cache on startup and submits `vkCreateGraphicsPipelines` calls in the background before those pipelines are needed at draw time.
- The cache is **driver-version-agnostic**: it caches the D3D→SPIR-V translation output, not driver-compiled GPU code.
- Since DXVK 2.0 and GPL support, the state cache is far less critical for anti-stutter on modern drivers; its primary value is on older hardware or drivers without GPL support.
- Location: `~/.steam/steam/steamapps/shadercache/<APPID>/DXVK_state_cache/` (Steam) or next to the game `.exe` in the Wine prefix.

**Fossilize database (`.foz` in `fozpipelinesv6/`)**:
- Records complete Vulkan-level `VkGraphicsPipelineCreateInfo` objects with all SPIR-V already embedded as `VkShaderModule` payloads.
- The pre-compilation step drives the GPU driver to compile these pipelines and stores the resulting GPU binaries in the Mesa disk cache.
- The database is **driver-version-sensitive**: the Mesa disk cache entries it populates are keyed by `pipelineCacheUUID` and become stale when the driver is updated.
- Populated and maintained by Valve/developers via Steam distribution, not by per-user gameplay sessions.
- Covers the Vulkan-level pipeline state produced by DXVK (D3D11→Vulkan) and VKD3D-Proton (D3D12→Vulkan) for games that have developer-submitted Fossilize databases.

The two caches can coexist and do not interfere. A title with both systems active benefits from: (a) DXVK state cache reducing the D3D→SPIR-V translation overhead on background threads, and (b) Fossilize pre-cache ensuring the SPIR-V→GPU-binary compilation has already occurred in the driver cache before the game launches.

### Verifying and Debugging Fossilize Pre-Caching

**The `fossilize-replay` CLI tool** (built from `cli/fossilize_replay.cpp` in the Fossilize source tree) is the primary utility for both Steam's pre-compilation automation and manual debugging:

```bash
# Replay all pipelines in a .foz database against the first Vulkan device (index 0)
fossilize-replay --device-index 0 game_pipelines.foz

# Replay with multi-threaded compilation (8 worker threads)
fossilize-replay --device-index 0 --num-threads 8 game_pipelines.foz

# Validate all SPIR-V modules before replay (useful for catching translation bugs)
fossilize-replay --device-index 0 --spirv-val game_pipelines.foz

# Replay only a specific range of graphics pipelines (for debugging individual pipelines)
fossilize-replay --device-index 0 \
    --start-graphics-index 100 \
    --end-graphics-index 110 \
    game_pipelines.foz

# Replay a single pipeline by its Fossilize hash
fossilize-replay --device-index 0 --pipeline-hash 0xdeadbeefcafebabe game_pipelines.foz
```

The `--spirv-val` flag invokes the SPIR-V validator (`spirv-val`) against every `VkShaderModule` payload before submitting it to the driver; modules that fail validation are skipped. This is essential when debugging DXVK or VKD3D-Proton translation bugs that produce invalid SPIR-V — the Fossilize database provides a reproducible input without requiring the full game. [Source](https://github.com/ValveSoftware/Fossilize/blob/master/cli/fossilize_replay.cpp)

**Counting pipelines in a database.** The Fossilize project ships `fossilize-opt` (in `cli/fossilize_opt.cpp`) as a database manipulation utility. To inspect database contents:

```bash
# Merge and count entries in a Fossilize database
fossilize-opt --input-db game_pipelines.foz --output-db /dev/null 2>&1 | grep -i pipeline
```

For a quick count of pipelines in the `fozpipelinesv6` directory for a given Steam game (Steam App ID 1234560):

```bash
ls -la ~/.steam/steam/steamapps/shadercache/1234560/fozpipelinesv6/*.foz 2>/dev/null
```

**RADV debugging flags for cache verification.** Two RADV options are particularly useful when verifying that Fossilize pre-compilation is working:

```bash
# Show PSO cache hit/miss statistics: confirms whether pre-compiled pipelines
# are being found in the Mesa disk cache at game runtime
RADV_DEBUG=psocachestats %command%

# Disable the shader cache entirely, forcing full recompilation on every vkCreatePipeline call
# Use this to measure baseline compile time vs. pre-cached runtime
RADV_DEBUG=nocache %command%
```

`RADV_DEBUG=psocachestats` is the recommended diagnostic tool: if the hit rate is high at game startup, Fossilize pre-caching succeeded and the driver found the pre-compiled binaries. A low hit rate despite pre-caching usually indicates a driver version mismatch (the cache was compiled against a different Mesa version) or a `pipelineCacheUUID` change. [Source](https://docs.mesa3d.org/envvars.html)

For cases where a shader compiler fix needs to be back-ported to a game that is already pre-cached against an old version, RADV exposes per-game drirc overrides:

```ini
# In ~/.drirc or /etc/drirc: force a shader version bump for a specific game,
# causing RADV to recompile all pipelines for that game once with the new compiler
[application name="mygame.exe"]
  radv_override_graphics_shader_version = 1
```

This mechanism is how SteamOS manages shader compiler bug fixes on the Steam Deck without requiring a user-visible "Preparing Shader Pre-Caches" re-run of the entire database. [Source](https://www.phoronix.com/news/RADV-Force-Shader-Re-Comp)

---

## Roadmap

### Near-term (6–12 months)

- **VK_EXT_descriptor_heap in VKD3D-Proton**: The VKD3D-Proton 3.0.1 release notes explicitly state that 3.0.1 is likely the last release before `VK_EXT_descriptor_heap` support lands, which will fundamentally improve how D3D12 descriptor heaps map to Vulkan — eliminating the need to emulate GPU-visible descriptor heaps via `VkDescriptorSet` copies. [Source](https://www.phoronix.com/news/VKD3D-Proton-3.0.1)
- **NVIDIA DLSS4 integration for Proton**: While AMD FSR4 (via `VK_KHR_cooperative_matrix` / `VK_KHR_shader_float8`) landed in VKD3D-Proton 3.0/3.0.1, DLSS4 has no native Proton integration as of mid-2026. NVIDIA DLSS4 support is the most-requested outstanding feature for VKD3D-Proton; enabling it would require either NVAPI bridge work or a dedicated VKD3D-Proton extension interface. [Source](https://www.tomshardware.com/video-games/pc-gaming/vulkan-to-directx-12-translation-tool-used-in-valves-proton-now-supports-amds-fsr4-and-anti-lag-while-nvidias-dlss4-remains-unsupported-fsr4-now-also-works-on-older-gpus-vkd3d-proton-v3-0-brings-other-performance-improvements)
- **DXVK-Sarek legacy GPU improvements**: The community fork DXVK-Sarek v1.12 (April 2026) added dyasync (async shader compilation) enabled by default for older hardware; near-term work continues on expanding Vulkan 1.1 compatibility shims for GPUs that cannot meet DXVK mainline's Vulkan 1.3 requirement. [Source](https://www.gamingonlinux.com/2026/04/gaming-on-linux-with-an-older-gpu-levels-up-with-dxvk-sarek-v1-12-bringing-major-new-features/)
- **D3D12 Work Graphs promotion to stable**: Work graph support (`D3D12_WORK_GRAPHS`) landed in VKD3D-Proton 3.0 as experimental; stabilisation and enabling it by default for drivers that expose `VK_KHR_work_graphs` (or equivalent compute graph extensions) is expected in a near-term point release. Note: needs verification of exact extension name.
- **Vulkan present timing integration**: VKD3D-Proton 3.0.1 added `VK_EXT_present_timing` support for smoother frame pacing; DXVK mainline is expected to adopt the same extension for D3D9/11 swap-chain paths. [Source](https://www.gamingonlinux.com/2026/05/vkd3d-proton-3-0-1-brings-many-linux-gaming-enhancements-for-direct3d-12-via-vulkan/)

### Medium-term (1–3 years)

- **Upstream vkd3d merge**: The DXBC shader backend rewrite in VKD3D-Proton 3.0 unified the DXBC frontend with DXVK's, creating a shared shader compiler path. A longer-term goal is to upstream enough of VKD3D-Proton's D3D12 improvements back into WineHQ's `vkd3d` so the two projects can share maintenance. [Source](https://9to5linux.com/vkd3d-proton-3-0-released-with-fsr4-support-dxbc-shader-backend-rewrite)
- **Opacity micromap and ray tracing extensions**: `VK_EXT_opacity_micromap` (for compressed foliage and vegetation geometry) was exposed experimentally in VKD3D-Proton 3.0 for DXR titles; stabilisation as a production feature depends on driver support maturing across RADV, ANV, and NVK. [Source](https://www.phoronix.com/news/VKD3D-Proton-3.0)
- **D3D12 view instancing and layered rendering**: View instancing (`D3D12_VIEW_INSTANCING_*`) was added experimentally in VKD3D-Proton 3.0.1; full stable support requires VR title validation and broader Vulkan multiview interaction testing. [Source](https://www.gamingonlinux.com/2026/05/vkd3d-proton-3-0-1-brings-many-linux-gaming-enhancements-for-direct3d-12-via-vulkan/)
- **DXVK D3D8 completeness**: D3D8 support in DXVK is implemented as a thin shim over D3D9; medium-term work targets closing remaining fixed-function pipeline gaps in D3D8 that affect a subset of pre-2002 games not yet passing ProtonDB's Platinum bar.  Note: needs verification against open issue tracker.
- **Mobile and Steam Deck GPU optimisations**: VKD3D-Proton 3.0.1 introduced deferred clears/discards and render-pass suspend-resume specifically for mobile GPU tile architectures (Steam Deck's AMD RDNA 2 APU). As Steam Deck 2 and next-generation handheld hardware arrives, both projects are expected to expand low-power-GPU tuning paths. [Source](https://www.phoronix.com/news/VKD3D-Proton-3.0.1)

### Long-term

- **DirectX 13 / next-generation D3D support**: Microsoft has not announced D3D13 publicly, but the VKD3D-Proton architecture has been structured to absorb new D3D12 feature levels incrementally (shader model 6.x tiers, enhanced barrier model). Any future D3D API generation would require new VKD3D-Proton feature tiers mapping to corresponding Vulkan extensions. Note: needs verification once D3D13 is formally announced.
- **Full NVK (Mesa open-source NVIDIA) support**: NVK is an emerging Vulkan driver for NVIDIA hardware within Mesa; DXVK and VKD3D-Proton running on NVK is a stated use case. Long-term, as NVK matures toward feature-complete Vulkan 1.3 + extension coverage, it may become a supported first-class target alongside RADV and ANV. [Source](https://github.com/HansKristian-Work/vkd3d-proton)
- **AI/ML upscaling ecosystem on Linux**: Beyond FSR4, the long-term roadmap for Proton gaming involves integrating neural upscaling (XeSS 2, DLSS 4's transformer model) through vendor API bridges. The architectural challenge is that these depend on vendor-specific inference backends (NVIDIA TensorRT, Intel OpenVINO) that have no Vulkan-native equivalent yet.
- **Convergence with Wine's translation approach**: As Wine's own Vulkan and D3D infrastructure matures, there is community interest in whether DXVK and VKD3D-Proton's compile-time and runtime optimisations can be progressively adopted by the Wine project's default stack, reducing the need for Proton-specific forks. Note: needs verification of current Wine upstream discussions.

---

## Integrations

- **Chapter 28 (Windows Compatibility — Wine, DXVK, VKD3D)**: Chapter 28 provides an overview of the full Windows compatibility stack including Wine's architecture, DXVK's role within it, and practical usage. This chapter (104) is the deep-dive technical complement — covering DXVK and VKD3D-Proton architecture, shader translation internals, and Vulkan extension mechanics at a level of detail beyond what Chapter 28's overview can address. Readers encountering DXVK for the first time should start with Chapter 28; readers building on the translation layers should read both.
- **Chapter 18 (Vulkan Drivers — RADV, ANV, NVK)**: RADV is the primary Vulkan driver target for both DXVK and VKD3D-Proton on AMD hardware. RADV's ACO compiler (Chapter 15) provides fast pipeline compilation that makes the graphics pipeline library approach viable. ANV provides the Intel path. NVK (the Mesa open-source Vulkan driver for NVIDIA GPUs) is an emerging target; DXVK and VKD3D-Proton running on NVK is a described use case in NVK's integration notes.
- **Chapter 24 (Vulkan and EGL for Application Developers)**: Every Vulkan extension that DXVK and VKD3D-Proton rely on — `VK_KHR_timeline_semaphore`, `VK_EXT_descriptor_buffer`, `VK_EXT_graphics_pipeline_library`, `VK_KHR_buffer_device_address` — is part of the Vulkan application-visible API described in Chapter 24.
- **Chapter 56 (Ray Tracing on Linux)**: DXR-to-Vulkan translation (Section 9 of this chapter) is the most complex consumer of `VK_KHR_ray_tracing_pipeline` and `VK_KHR_acceleration_structure` described in Chapter 56. The SBT translation and TLAS/BLAS management discussed here complement the native Vulkan ray tracing pipeline discussion there.
- **Chapter 61 (Modern Vulkan Extensions)**: `VK_EXT_descriptor_buffer` (bindless descriptors), `VK_EXT_graphics_pipeline_library` (pipeline linking), `VK_EXT_mesh_shader` (mesh shaders), and `VK_KHR_cooperative_matrix` (FSR4 via VKD3D-Proton 3.0) are all covered in Chapter 61; this chapter is one of their primary production consumers.
- **Chapter 77 (Shader Source-to-ISA: The Complete Compilation Toolchain)**: The DXBC → SPIR-V → NIR → ISA path (via `DxbcCompiler`, `SpirvModule`, and the Mesa NIR front end) and the DXIL → SPIR-V → NIR → ISA path (via `dxil-spirv`) are key compilation pipelines in the overall shader toolchain taxonomy. Chapter 77 covers the ISA backend; this chapter covers the frontend.
- **Chapter 78 (Gamescope and the Steam Deck)**: On SteamOS 3, Proton games (DXVK + VKD3D-Proton) run as nested Wayland clients of Gamescope. Gamescope provides the fullscreen Wayland compositor, applies AMD FSR upscaling as a final pass, and handles frame pacing. Chapter 78 describes the full Steam Deck graphics stack that DXVK games run on top of.
- **Chapter 97 (Unreal Engine 5 on Linux)**: Windows builds of UE5 games on Linux go through VKD3D-Proton (for D3D12 targets) or DXVK (for D3D11 targets). Native UE5 Linux builds bypass this entirely and use Vulkan directly. Chapter 97 describes both paths from the engine perspective.

---

## References

- DXVK project: [github.com/doitsujin/dxvk](https://github.com/doitsujin/dxvk)
- VKD3D-Proton project: [github.com/HansKristian-Work/vkd3d-proton](https://github.com/HansKristian-Work/vkd3d-proton)
- dxil-spirv: [github.com/HansKristian-Work/dxil-spirv](https://github.com/HansKristian-Work/dxil-spirv)
- Arntzen, H.-K., "My personal hell of translating DXIL to SPIR-V — part 1" (2021): [themaister.net](https://themaister.net/blog/2021/09/05/my-personal-hell-of-translating-dxil-to-spir-v-part-1/)
- DXVK 2.7 release: [linuxcompatible.org/story/dxvk-27-released](https://www.linuxcompatible.org/story/dxvk-27-released/)
- VKD3D-Proton 3.0 release notes: [github.com/HansKristian-Work/vkd3d-proton/releases/tag/v3.0](https://github.com/HansKristian-Work/vkd3d-proton/releases/tag/v3.0)
- VKD3D-Proton 2.11 DXR default announcement: [gamingonlinux.com](https://gamingonlinux.com/2023/11/vkd3d-proton-211-released-with-directx-raytracing-enabled-by-default/)
- VK_EXT_descriptor_buffer proposal: [github.com/KhronosGroup/Vulkan-Docs](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/proposals/VK_EXT_descriptor_buffer.adoc)
- Steam Linux Runtime versions: [steamcommunity.com](https://steamcommunity.com/app/221410/discussions/8/4038104984938159075/)
- DXVK apitrace guide: [github.com/doitsujin/dxvk/wiki/Using-Apitrace](https://github.com/doitsujin/dxvk/wiki/Using-Apitrace)
- DeepWiki DXVK command system: [deepwiki.com/doitsujin/dxvk/2.1-command-system](https://deepwiki.com/doitsujin/dxvk/2.1-command-system)
- DeepWiki VKD3D-Proton D3D12 translation: [deepwiki.com/HansKristian-Work/vkd3d-proton/2.1-d3d12-to-vulkan-translation](https://deepwiki.com/HansKristian-Work/vkd3d-proton/2.1-d3d12-to-vulkan-translation)
- DeepWiki VKD3D-Proton resources and heaps: [deepwiki.com/HansKristian-Work/vkd3d-proton/3.3-resources-and-heaps](https://deepwiki.com/HansKristian-Work/vkd3d-proton/3.3-resources-and-heaps)
- Steam Linux usage statistics 2026: [commandlinux.com/statistics/steam-linux-runtime](https://commandlinux.com/statistics/steam-linux-runtime/)
- NVIDIA blog, "Bringing HLSL Ray Tracing to Vulkan": [developer.nvidia.com/blog/bringing-hlsl-ray-tracing-to-vulkan](https://developer.nvidia.com/blog/bringing-hlsl-ray-tracing-to-vulkan/)
- Fossilize project: [github.com/ValveSoftware/Fossilize](https://github.com/ValveSoftware/Fossilize)
- Fossilize README (format, layer, CLI tools): [github.com/ValveSoftware/Fossilize/blob/master/README.md](https://github.com/ValveSoftware/Fossilize/blob/master/README.md)
- fossilize-replay source (CLI options): [github.com/ValveSoftware/Fossilize/blob/master/cli/fossilize_replay.cpp](https://github.com/ValveSoftware/Fossilize/blob/master/cli/fossilize_replay.cpp)
- Mesa single-file Fossilize disk cache: [phoronix.com/news/Mesa-Single-File-Cache](https://www.phoronix.com/news/Mesa-Single-File-Cache)
- Mesa util/disk_cache combined foz MR 11485: [gitlab.freedesktop.org/mesa/mesa/-/merge_requests/11485](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/11485)
- RADV GPL enabled by default in Mesa 23.1: [gamingonlinux.com/2023/05/mesa-graphics-drivers-23-1-0-out-now-with-radv-gpl-enabled](https://www.gamingonlinux.com/2023/05/mesa-graphics-drivers-23-1-0-out-now-with-radv-gpl-enabled/)
- RADV 10x faster RT pipeline compilation (Fossilize capture): [phoronix.com/news/RADV-10x-Fast-RT-Pipeline-Comp](https://www.phoronix.com/news/RADV-10x-Fast-RT-Pipeline-Comp)
- RADV forced shader re-compilation (drirc knobs): [phoronix.com/news/RADV-Force-Shader-Re-Comp](https://www.phoronix.com/news/RADV-Force-Shader-Re-Comp)
- Mesa environment variables (RADV_DEBUG): [docs.mesa3d.org/envvars.html](https://docs.mesa3d.org/envvars.html)
- DeepWiki Fossilize overview: [deepwiki.com/ValveSoftware/Fossilize/1-overview](https://deepwiki.com/ValveSoftware/Fossilize/1-overview)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
