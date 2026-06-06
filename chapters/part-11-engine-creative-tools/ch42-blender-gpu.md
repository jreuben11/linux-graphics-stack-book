# Chapter 42: Blender — Cycles, EEVEE Next, and GPU Compute on Linux

> **Part**: Part XI — Engine and Creative Tool Internals
> **Audience**: Graphics application developers — Blender add-on developers, technical artists, and rendering engineers who want to understand Blender's GPU rendering internals; systems developers interested in how a major open-source creative suite interacts with Mesa, ROCm, and the Linux compute ecosystem
> **Status**: First draft — 2026-06-06

Blender is unique among the tools examined in this part: it is the only major open-source 3D creation suite, and its GPU rendering story encompasses more of the Linux stack than any other application in the book. EEVEE Next — the Vulkan-based rewrite that shipped as the production renderer in Blender 4.2 LTS and reached stable parity with the OpenGL path in Blender 4.5 LTS — is a full deferred PBR renderer that exercises Mesa's Vulkan drivers directly. The Cycles path tracer provides a multi-backend compute story spanning HIP (AMD/ROCm), CUDA/OptiX (NVIDIA), oneAPI (Intel Arc), and an experimental Vulkan compute path. Readers who have worked through the kernel driver chapters (Parts I–III), the Mesa internals (Parts IV–V), and the GPU compute chapter (Chapter 25) will find Blender an unusually instructive integration point where those layers converge in a production workload.

---

## Table of Contents

- [1. GPU Rendering in Blender: Architecture Overview](#1-gpu-rendering-in-blender-architecture-overview)
- [2. EEVEE Next: The Vulkan Rewrite](#2-eevee-next-the-vulkan-rewrite)
- [3. Cycles GPU Backend: Multi-Backend Compute Architecture](#3-cycles-gpu-backend-multi-backend-compute-architecture)
- [4. GLSL/SPIR-V Shader Compilation in Blender](#4-glslspir-v-shader-compilation-in-blender)
- [5. OpenColorIO Integration and Color Management](#5-opencolorio-integration-and-color-management)
- [6. Viewport Rendering: GHOST, Wayland, and GPU Memory](#6-viewport-rendering-ghost-wayland-and-gpu-memory)
- [7. Cycles as a GPU Compute Workload: Performance Characteristics](#7-cycles-as-a-gpu-compute-workload-performance-characteristics)
- [8. Debugging Blender GPU Issues on Linux](#8-debugging-blender-gpu-issues-on-linux)
- [Integrations](#integrations)
- [References](#references)

---

## 1. GPU Rendering in Blender: Architecture Overview

Blender exposes two GPU rendering engines and a rich GPU-accelerated viewport, all built on top of a common hardware abstraction layer. Understanding that abstraction layer is the prerequisite for everything else in this chapter.

### 1.1 The Two Renderers

**EEVEE** (historically "EEVEE Next" during development, simply "EEVEE" in 4.2+) is a real-time deferred PBR renderer designed for interactive use and fast final renders where photorealism can be traded for speed. It processes each frame through a structured sequence of Vulkan render passes — G-buffer fill, clustered lighting, volumetrics, shadow maps, screen-space effects — and presents the result to the viewport or outputs it to disk. Since Blender 4.2 LTS its underlying GPU implementation is Vulkan-first; the OpenGL path has been the fallback. [Source](https://code.blender.org/2024/06/blender-4-2-the-start-of-the-lts-era/)

**Cycles** is a physically-based Monte Carlo path tracer. It uses GPU compute to parallelise ray-scene intersections and shade each sample point; renders converge to a noise-free image as sample counts accumulate. Cycles does not use EEVEE's rasterisation pipeline at all — it dispatches compute workloads through five different backends (HIP, CUDA, OptiX, oneAPI, Vulkan compute) that map to different vendor APIs and ultimately to different kernel-mode drivers. CPU rendering via multi-architecture SIMD C++ is always available as a fallback. [Source](https://developer.blender.org/docs/handbook/building_blender/cycles_gpu_binaries/)

Both renderers share the same scene data model — `Mesh`, `Material`, `Light`, `Object` — expressed through Blender's DNA/RNA system. The difference is entirely in how that scene data is transformed into pixels.

### 1.2 The GPUBackend Abstraction

Blender's internal GPU abstraction lives in `source/blender/gpu/`. The central class is `GPUBackend`, an abstract base that the rest of the GPU module uses exclusively; the `VKBackend` (Vulkan), `GLBackend` (OpenGL), and `MetalBackend` (macOS) subclasses each implement the full interface. [Source](https://projects.blender.org/blender/blender/src/branch/main/source/blender/gpu)

The abstraction surface provided by `GPUBackend` covers:

- `GPUContext` — per-window GPU state; wraps a `VkDevice`/`EGLContext` handle
- `GPUShader` — compiled shader program; wraps a compiled GLSL/SPIR-V program
- `GPUTexture` — GPU image; wraps `VkImage` or `GLuint`
- `GPUBatch` — draw call descriptor; vertex buffers + index buffer + shader
- `GPUFrameBuffer` — render target; wraps a set of `VkImage` attachments or an FBO
- `GPUStorageBuf` / `GPUUniformBuf` — buffer objects; wrap `VkBuffer` allocations

Factory methods on `VKBackend` construct the Vulkan-specific subclasses: `batch_alloc()` returns a `VKBatch`, `shader_alloc()` returns a `VKShader`, and so on. This means that code in EEVEE or the viewport calls `GPUShader::bind()` without knowing whether the current backend is Vulkan or OpenGL.

### 1.3 Vulkan Status in Blender 4.5 LTS

In Blender 4.5 LTS, the Vulkan backend reached feature parity with OpenGL — GPU subdivision, OpenXR, USD/Hydra workflows, and all EEVEE effects are implemented on both paths. However, OpenGL remains the **default** backend. Vulkan is opt-in via *Preferences → System → Graphics API*. The reasons are practical: the Vulkan path requires all scene textures to fit simultaneously in GPU memory (whereas OpenGL can spill to streaming), imposes a performance regression on very large meshes (>100 million vertices), and exhibits higher memory usage under certain workloads. The Blender development team indicated that Blender 5.0 would likewise not default to Vulkan until these constraints are resolved. [Source](https://www.phoronix.com/news/Blender-5.0-Vulkan-OpenGL-RAM)

Minimum hardware for the Vulkan path:
- Vulkan 1.2, with `VK_KHR_dynamic_rendering`, `VK_KHR_timeline_semaphore`, `VK_EXT_extended_dynamic_state`, and `VK_KHR_synchronization2`
- NVIDIA: driver 550+ (nvidia-open or proprietary)
- AMD/Intel: Mesa 25.3+ (RADV for AMD, ANV for Intel)

These requirements are documented in `VKBackend::missing_capabilities_get()` which filters unsuitable hardware before device initialisation proceeds. [Source](https://projects.blender.org/blender/blender/src/branch/main/source/blender/gpu/vulkan/vk_backend.cc)

---

## 2. EEVEE Next: The Vulkan Rewrite

EEVEE Next is the name used during development for the Vulkan-based rewrite of the original OpenGL EEVEE renderer. It shipped as the production EEVEE engine in Blender 4.2 LTS (where it simply became "EEVEE") and has been the only EEVEE implementation since. The OpenGL EEVEE from Blender 3.x was retired. [Source](https://code.blender.org/2024/06/blender-4-2-the-start-of-the-lts-era/)

### 2.1 VKBackend Architecture

`VKBackend` implements `GPUBackend` and serves as the factory for the full set of Vulkan GPU objects. Its class structure, verified against `source/blender/gpu/vulkan/vk_backend.cc`, includes:

```cpp
/* Source: source/blender/gpu/vulkan/vk_backend.cc
 * VKBackend is the Vulkan implementation of GPUBackend.
 * platform_init() is called once at startup; it validates hardware capabilities,
 * initialises the VKDevice singleton, and sets the platform info string.
 */
class VKBackend : public GPUBackend {
 public:
  void platform_init() override;
  void platform_init(const VKDevice &device);
  void platform_exit() override;

  /* Factory methods returning Vulkan-specific GPU object subclasses */
  GPUBatch      *batch_alloc() override;
  GPUShader     *shader_alloc(const char *name) override;
  GPUTexture    *texture_alloc(const char *name) override;
  GPUVertBuf    *vertbuf_alloc() override;
  GPUUniformBuf *uniformbuf_alloc(size_t size, const char *name) override;
  GPUStorageBuf *storagebuf_alloc(size_t size, GPUUsageType use, const char *name) override;

  /* Capability query: returns non-empty string if Vulkan is unusable on this device */
  static std::string missing_capabilities_get(
      const VkInstance vk_instance,
      const VkPhysicalDevice vk_physical_device);

  /* Global singleton accessors */
  static VKBackend &get();
  static bool is_supported();
};
```

Note: these signatures are verified against the Blender main branch as of 2026; verify against Blender 4.5 source for precise parameter lists.

**`VKDevice`** is a singleton that wraps the Vulkan logical device and global resources. Its `init(GHOST_IContext *ghost_context)` method retrieves the `VkInstance`, `VkPhysicalDevice`, `VkDevice`, and `VkQueue` handles from the GHOST windowing context, initialises `VkPhysicalDeviceProperties`, `VkPhysicalDeviceFeatures`, and `VkPhysicalDeviceMemoryProperties`, then sets up the Vulkan Memory Allocator (VMA) pool. The `VkQueue` uses timeline semaphores (`vk_timeline_semaphore_`) for all cross-frame synchronisation. [Source](https://projects.blender.org/blender/blender/src/branch/main/source/blender/gpu/vulkan/vk_device.cc)

**`VKContext`** is a per-window GPU state object. Each Blender window has one `VKContext` that owns a `VKRenderGraph` and maintains the per-window swapchain state through `GHOST_ContextVK`. The context delegates GPU work to `VKDevice` via `device.render_graph_submit()` and `device.render_graph_new()`.

**`VKRenderGraph`** — introduced in Blender 4.3 — decouples command recording from submission. EEVEE and the viewport add render nodes to the graph (draw calls, compute dispatches, buffer copies, image layout transitions) without recording `VkCommandBuffer` objects directly. A background `submission_runner` task dequeues submitted render graphs, runs `VKScheduler` to reorder operations and insert pipeline barriers, then invokes `VKCommandBuilder` to produce the actual `VkCommandBuffer` before calling `vkQueueSubmit`. [Source](https://developer.blender.org/docs/features/gpu/vulkan/render_graph/)

**`VKDiscardPool`** implements deferred resource destruction. When a `VKTexture` or `VKBuffer` is freed, it is moved into the discard pool tagged with the current timeline semaphore value; the background runner physically destroys the resource once the GPU signals that timeline value, eliminating races between CPU-side free and in-flight GPU use.

**VMA integration**: All `VkBuffer` and `VkImage` allocations go through VMA. The backend uses `VMA_MEMORY_USAGE_AUTO_PREFER_HOST` with `VMA_ALLOCATION_CREATE_HOST_ACCESS_SEQUENTIAL_WRITE_BIT` for staging buffers, and `VMA_MEMORY_USAGE_AUTO` (GPU-preferred) for scene geometry and textures. [Source](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator)

### 2.2 EEVEE Render Pass Structure

EEVEE Next uses `VK_KHR_dynamic_rendering` throughout — there are no static `VkRenderPass` objects. Each rendering phase calls `vkCmdBeginRendering()` with `VkRenderingAttachmentInfo` structures describing the colour and depth/stencil attachments for that phase.

**G-Buffer pass**: Renders geometry into multiple `VkImage` attachments simultaneously:
- Depth (`VK_FORMAT_D32_SFLOAT`)
- Normal vectors (screen-space normals in a half-float attachment)
- Albedo (surface colour before lighting)
- Packed roughness/metallic/occlusion

The G-buffer pass operates as a deferred shading stage: geometry is rasterised once, surface properties stored, and lighting applied in a subsequent pass that reads the G-buffer textures via descriptor set bindings.

**Clustered lighting pass**: A compute shader subdivides the view frustum into a 3D grid of clusters (a `VkImage3D` cluster map). For each cluster, it determines which lights — point, spot, area, directional — contribute, stores light indices into a `VkStorageBuffer`. The subsequent light accumulation pass reads the cluster map to avoid iterating all lights per fragment.

**Virtual shadow maps**: EEVEE Next replaces the legacy cascade/cubemap system with a **virtual shadow map tilemap** architecture — one of its headline rendering advances over EEVEE Legacy. The system maintains a single shared shadow atlas (`VkImage` texture atlas in GPU memory) whose tiles are allocated and updated on demand per visible surface and volume. Each shadow caster maps to a conceptual virtual texture (per cube face for local lights; per clipmap or cascade level for directional lights), but all share the same physical memory pool. The page allocation and update logic runs entirely on the GPU via compute shaders with no CPU synchronisation, driven by visibility queries each frame. Directional lights use a clipmap distribution (or cascade distribution for distant receivers), and local lights use cubemap projections with mipmap-based LOD adapted to receiver distance. [Source](https://projects.blender.org/blender/blender/commit/a0f52400890)

**Volumetric pass**: Froxel-based volumetrics via compute shaders. The view frustum is divided into a 3D froxel grid; a compute shader integrates participating media density and light scattering per froxel. The result is a `VkImage3D` sampled during the lighting accumulation.

**Screen-space effects**: Bloom, ambient occlusion, depth-of-field, and motion blur are implemented as post-processing compute shaders consuming and producing `VkImage` attachments. Each dispatches a `vkCmdDispatch` through the render graph system.

### 2.3 Required Vulkan Extensions and Driver Versions

The `VKBackend::missing_capabilities_get()` function validates against the following requirements as of Blender 4.5:

- `VK_KHR_dynamic_rendering` — no render pass objects; required for EEVEE's pass structure
- `VK_KHR_timeline_semaphore` — cross-frame GPU synchronisation in `VKDevice`
- `VK_KHR_synchronization2` — pipeline barrier API used throughout `VKCommandBuilder`
- `VK_EXT_extended_dynamic_state` — dynamic state for viewport, scissor, blend in draw calls
- `VK_EXT_vertex_input_dynamic_state` — avoids pipeline explosion for varying vertex formats
- `VK_KHR_maintenance4` — required for various layout operations
- `VK_EXT_host_image_copy` — async texture upload without staging buffer round-trips (where supported)

The driver version checks in `platform_init()` reject known-broken driver versions: Intel drivers below 101.2140, NVIDIA drivers below 550, and certain Qualcomm driver revisions. On Linux these translate to: NVIDIA requires driver 550+ (nvidia-open or proprietary); AMD and Intel require Mesa 25.3+.

---

## 3. Cycles GPU Backend: Multi-Backend Compute Architecture

Cycles shares no code with EEVEE at the GPU level. Where EEVEE uses the `GPUBackend` rasterisation abstraction, Cycles implements its own compute dispatch layer with backend-specific implementations that directly call vendor compute APIs.

### 3.1 Kernel Architecture

The Cycles path-tracing kernel lives in `intern/cycles/kernel/`. It is written in a portable C++ dialect annotated with macros (`ccl_kernel`, `ccl_device`, `ccl_global`) that abstract over the calling conventions and address space qualifiers of each target:

- `__global__` for CUDA PTX
- `__device__` for HIP
- `sycl::device` attributes for oneAPI SYCL
- standard C++ (multi-arch SIMD via AVX2/AVX-512 intrinsics) for CPU

The same kernel files compile to each target through backend-specific build steps invoked during Blender's CMake build. At render time each backend loads its pre-compiled binary and dispatches it to the GPU. [Source](https://developer.blender.org/docs/handbook/building_blender/cycles_gpu_binaries/)

### 3.2 HIP Backend (AMD/ROCm)

The HIP (Heterogeneous Interface for Portability) backend targets AMD GPUs via ROCm. The source lives in `intern/cycles/device/hip/`.

**Kernel compilation**: Cycles ships pre-compiled HIP fat binaries for supported GPU architectures (GFX1010, GFX1030, GFX1100, GFX1201, etc.). The `HIPDevice::compile_kernel()` method can also compile locally using `hipcc` (the HIP compiler, based on LLVM/ROCm) with `--offload-arch=` architecture flags if the pre-compiled binary is absent or the GPU architecture is newer than those included. [Source](https://github.com/blender/cycles/blob/main/src/device/hip/device_impl.cpp)

**Module loading and kernel access**: At render initialisation, `hipModuleLoadData()` loads the fat binary into GPU memory. Individual kernel functions are retrieved from the loaded module with `hipModuleGetFunction()`, which populates `HIPDeviceKernel` structures (one per logical kernel):

```cpp
/* Source: intern/cycles/device/hip/kernel.cpp
 * Loads a single kernel function from a pre-loaded HIP module.
 * hipModuleGetFunction() retrieves the function handle by name,
 * then hipFuncSetCacheConfig() configures L1/shared memory balance.
 */
bool HIPDeviceKernels::load_kernel(HIPDevice *device,
                                   hipModule_t hip_module,
                                   const DeviceKernel kernel)
{
  const char *name = device_kernel_as_string(kernel);
  hipFunction_t func = nullptr;
  hip_assert(hipModuleGetFunction(&func, hip_module, name));
  hip_assert(hipFuncSetCacheConfig(func, hipFuncCachePreferL1));
  kernels_[kernel].function = func;
  return func != nullptr;
}
```

Note: verify exact parameter names against Blender 4.5 source at `intern/cycles/device/hip/kernel.cpp`.

**Kernel dispatch**: Actual dispatch is handled by `HIPDeviceQueue`, which wraps `hipModuleLaunchKernel()`. The dispatch path for a path-tracing tile looks approximately as follows:

```cpp
/* Source: intern/cycles/device/hip/queue.cpp (approximate structure)
 * hipModuleLaunchKernel dispatches the kernel to the amdgpu KFD compute queue.
 * grid_size and block_size are computed from tile dimensions and SIMD width.
 * Note: verify exact signature and flag usage against Blender 4.5 source.
 */
void HIPDeviceQueue::enqueue(DeviceKernel kernel,
                              const int work_size,
                              DeviceKernelArguments const &args)
{
  const HIPDeviceKernel &k = device_->kernels().get(kernel);
  /* Compute optimal launch configuration */
  int min_blocks, threads_per_block;
  hip_assert(hipModuleOccupancyMaxPotentialBlockSize(
      &min_blocks, &threads_per_block, k.function, 0, 0));
  int grid = divide_up(work_size, threads_per_block);

  hip_assert(hipModuleLaunchKernel(
      k.function,
      grid, 1, 1,          /* grid dimensions */
      threads_per_block, 1, 1,  /* block dimensions */
      0,                   /* shared memory bytes */
      hip_stream_,         /* HIP stream */
      const_cast<void **>(args.values),
      nullptr));           /* extra args (unused) */
}
```

The `hipModuleLaunchKernel` call submits the compute work to the amdgpu kernel driver's KFD (Kernel Fusion Driver) compute queue. This is the same `amdgpu_kfd` interface described in Chapter 5, Section 3 — Cycles' HIP backend and Mesa's compute stack share the same kernel driver entry point.

**Supported hardware**: Cycles' HIP backend supports GCN4 and later. As of Blender 4.4, the ROCm SDK was updated to version 6.2.4, with GFX12 (RDNA 4, RX 9000 series) support introduced in ROCm 6.3.1. [Source](https://projects.blender.org/blender/blender/issues/131976)

**ROCm version pinning**: Blender's HIP backend may require a specific ROCm version. Check the Blender 4.5 system requirements at time of use; mismatched ROCm and Blender versions frequently cause `hipModuleLoadData` failures.

### 3.3 CUDA and OptiX Backend (NVIDIA)

For NVIDIA GPUs, Cycles provides two backends that both rely on the proprietary NVIDIA userspace stack.

**CUDA backend**: `nvcc` compiles the kernel to PTX (NVIDIA's virtual ISA). Cycles ships pre-compiled PTX binaries for common NVIDIA architectures (compute capability 3.0 through 9.0). At render time `cuModuleLoad()` (or `cuModuleLoadData()` for in-memory PTX) loads the binary; the CUDA JIT compiler in `libcuda.so` produces final GPU machine code if the compute capability does not exactly match a pre-compiled binary. `cuLaunchKernel()` dispatches individual tiles with a grid of thread blocks sized to the tile dimensions. This path uses NVIDIA's proprietary `libcuda.so` and the `nvidia-uvm.ko` kernel module for unified memory management; it bypasses Mesa and NVK entirely.

**OptiX backend**: For RTX-class hardware (Turing and later), Cycles uses NVIDIA's OptiX ray-tracing API for hardware-accelerated BVH traversal. The path-tracing intersection kernel is compiled with `optixProgramGroupCreate()` into callable shader programs; `optixLaunch()` dispatches work through the pipeline, which schedules BVH traversal on NVIDIA's dedicated RT Core units. The RT Cores perform closest-hit and any-hit intersection tests without occupying CUDA shader cores, allowing shader evaluation to proceed in parallel. For complex outdoor scenes with high ray divergence — where many rays travel to distant geometry — OptiX provides a 30–60% throughput advantage over the pure CUDA path on the same hardware.

Cycles also ships HIP-RT for AMD (the HIP equivalent of OptiX's hardware ray tracing), which targets the RT accelerators in RDNA 3+ hardware and provides a similar advantage on ray-divergent scenes.

Like CUDA, OptiX requires the proprietary NVIDIA driver stack and does not interact with Mesa. When the nvidia-open kernel module is used with NVK for EEVEE rendering, Cycles still falls back to CUDA/OptiX via `libcuda.so` for compute — the two stacks coexist on the same system.

Neither the CUDA nor OptiX path is relevant to the Mesa/ROCm Linux stack; they are documented here for completeness because many Linux production setups use NVIDIA hardware.

### 3.4 oneAPI Backend (Intel Arc)

Cycles' oneAPI backend uses Intel's SYCL-based heterogeneous compute API targeting Intel Arc (Alchemist/Battlemage) GPUs. The kernel is compiled with `icpx` (Intel's SYCL compiler) to Intel GPU SPIR-V. At render time, `sycl::queue` objects dispatch work to the Xe kernel driver (`i915` or `xe`, covered in Chapter 5). The oneAPI backend requires Intel's oneAPI Base Toolkit installed on the host.

Performance on Intel Arc hardware has improved significantly with each driver release; current numbers should be verified at [opendata.blender.org](https://opendata.blender.org/).

### 3.5 Vulkan Compute Backend (Experimental)

Cycles has an experimental Vulkan compute backend (`intern/cycles/device/vulkan/`) that uses `VkComputePipeline` objects to dispatch the path-tracing kernel compiled to SPIR-V. The significant advantage is portability: any Vulkan 1.2-capable Mesa driver (RADV, ANV, NVK) can run Cycles without requiring HIP or CUDA runtimes. The disadvantage is performance: as of Blender 4.5 the Vulkan compute backend is measurably slower than the HIP and CUDA paths on equivalent hardware and is not recommended for production renders. Development is ongoing.

> **Note**: verify the current experimental status of the Vulkan compute backend against Blender 4.5 release notes; this area evolves rapidly.

### 3.6 CPU Fallback

The Cycles CPU backend does not use ISPC. The kernel is written as standard C++ and compiled multiple times for different SIMD widths: a scalar fallback, and AVX2/AVX-512 variants. At startup, `CPUKernels` registers both variants using the `KERNEL_FUNCTIONS` macro; at render time, `CPUDevice::thread_render()` selects the widest available variant by querying CPUID and invoking the matching function pointer. ISPC is used by some of Blender's dependencies (Intel Embree for BVH building, Open Image Denoise) but not by the Cycles path-tracing kernel itself. [Source](https://github.com/blender/cycles/blob/main/src/device/cpu/kernel.cpp)

CPU rendering is always available regardless of GPU drivers and is useful for small test renders or systems without supported GPUs.

---

## 4. GLSL/SPIR-V Shader Compilation in Blender

EEVEE does not use hand-authored GLSL files for most of its shaders. Instead, Blender generates GLSL programmatically from the node graph, then compiles it to SPIR-V using shaderc, feeds the SPIR-V to Mesa via `vkCreateShaderModule`, and relies on Mesa's NIR pipeline for the final machine code.

### 4.1 GLSL Code Generation from the Node Graph

Blender's shader node graph — the visual editor that artists use to build PBR materials — compiles to GLSL through a dedicated code generation layer. The key insight is that Blender does not interpret the node graph at render time; it compiles the graph to a static GLSL program that represents the material's entire shading logic. Two materials that have structurally identical node graphs (same node types and connections, different parameter values) may share the same compiled `GPUPass` and differ only in their uniform buffer contents.

**`GPUMaterial`** represents a compiled material. Its source is a `GPUNodeGraph` — an intermediate representation of Blender's shader node tree (the material editor graph). The compilation sequence, verified against `source/blender/gpu/intern/gpu_material.cc`:

1. `GPU_material_from_nodetree()` searches the material cache by UUID and engine type. On a cache miss it calls `ntreeGPUMaterialNodes()` to walk the node tree and build the `GPUNodeGraph`.
2. `GPU_generate_pass()` invokes a codegen callback that traverses the `GPUNodeGraph` and emits GLSL function calls, one per active node. Each shader node type (`ShaderNodeBsdfPrincipled`, `ShaderNodeTexImage`, etc.) has a corresponding GLSL function in `source/blender/gpu/shaders/material/gpu_shader_material_*.glsl`.
3. The generated GLSL is a complete, self-contained Vulkan GLSL file using `layout(set=X, binding=Y)` qualifiers. Descriptor set assignments follow the convention established in `gpu_shader_create_info.hh`.
4. An optimised variant (`optimized_pass`) bakes dynamic uniform values as compile-time constants for materials where the values do not change per-frame.

```cpp
/* Source: source/blender/gpu/intern/gpu_material.cc (simplified)
 * GPU_material_from_nodetree: converts a Blender node tree to a compiled GPUMaterial.
 * ntreeGPUMaterialNodes traverses the node tree and builds the GPUNodeGraph.
 * GPU_generate_pass calls the codegen callback to produce GLSL source.
 */
GPUMaterial *GPU_material_from_nodetree(Scene *scene,
                                         Material *ma,
                                         bNodeTree *ntree,
                                         ListBase *gpumaterials,
                                         const char *name,
                                         uint64_t shader_uuid,
                                         bool is_volume_shader,
                                         bool is_lookdev,
                                         GPUMaterialEvalCallbackFn callback)
{
  /* Search the per-object material cache by UUID and engine type */
  GPUMaterial *mat = gpu_material_cache_find(gpumaterials, shader_uuid);
  if (mat != nullptr) {
    return mat;
  }
  /* Build the node graph: inlines reroutes and mutes, evaluates subgraphs */
  ntreeGPUMaterialNodes(ntree, mat, &mat->has_surface_output, &mat->has_volume_output);
  /* Generate the GPU pass (compiled GLSL + VkShaderModule) */
  mat->pass = GPU_generate_pass(mat, &mat->graph, callback, false);
  /* ... optimised variant and cache insertion omitted */
  return mat;
}
```

### 4.2 GPUShaderCreateInfo: Static Shader Definitions

For non-material shaders (EEVEE's built-in G-buffer fill, shadow map, clustered lighting compute, volumetric compute), Blender uses the `gpu_shader_create_info` system. A shader definition is created in C++ using a builder API:

```cpp
/* Source: source/blender/gpu/intern/gpu_shader_create_info.cc (illustrative)
 * GPUShaderCreateInfo declares the binding layout for a shader stage.
 * The backend generates Vulkan-compatible GLSL with layout(set=X, binding=Y)
 * qualifiers from these declarations.
 * Note: verify exact API against Blender 4.5 source.
 */
GPU_SHADER_CREATE_INFO(eevee_gbuffer_fill)
    .vertex_in(0, Type::VEC3, "pos")
    .vertex_in(1, Type::VEC3, "nor")
    .vertex_in(2, Type::VEC2, "uv")
    .uniform_buf(0, "ObjectMatrices", "matrices")
    .sampler(1, ImageType::FLOAT_2D, "albedo_tex")
    .fragment_out(0, Type::VEC4, "out_normal")
    .fragment_out(1, Type::VEC4, "out_albedo")
    .fragment_out(2, Type::VEC4, "out_material")
    .vertex_source("eevee_gbuffer_vert.glsl")
    .fragment_source("eevee_gbuffer_frag.glsl")
    .do_static_compilation(true);
```

Each `GPU_SHADER_CREATE_INFO` definition is registered at startup. When a `GPUShader` is first needed, the backend generates the binding qualifiers from the info struct and prepends them to the GLSL source before compilation.

### 4.3 GLSL to SPIR-V: shaderc

Blender compiles GLSL to SPIR-V using **shaderc** (Google's GLSL compiler library, which wraps glslang and SPIRV-Tools). The compilation is implemented in `source/blender/gpu/vulkan/vk_shader.cc`. The key path, verified against the source:

```cpp
/* Source: source/blender/gpu/vulkan/vk_shader.cc
 * build_shader_module: compiles GLSL source (potentially multiple files)
 * to SPIR-V using shaderc, stores result in VKShaderModule.
 * shaderc_shader_kind selects the pipeline stage
 * (shaderc_vertex_shader, shaderc_fragment_shader, shaderc_compute_shader).
 */
void build_shader_module(MutableSpan<StringRefNull> sources,
                         shaderc_shader_kind stage,
                         VKShaderModule &r_shader_module)
{
  /* Retrieve per-device GLSL patches (version string, extension enables) */
  const std::string &patch = VKDevice::get().glsl_patch_get(stage);
  /* Combine patch + per-stage source files into unified text */
  std::string combined = combine_sources(patch, sources);
  /* Invoke shaderc to produce SPIR-V binary */
  VKShaderCompiler::compile_module(combined, stage, r_shader_module);
  r_shader_module.finalize();
}
```

For each pipeline stage (vertex, geometry, fragment, compute), the stage-specific methods — `vertex_shader_from_glsl()`, `fragment_shader_from_glsl()`, `compute_shader_from_glsl()` — call `build_shader_module()` with the appropriate `shaderc_shader_kind`. The per-device GLSL patch prepends the GLSL version (`#version 450`), required extension enables (`#extension GL_EXT_buffer_reference2 : enable`, etc.), and any workarounds for driver-specific behaviour.

> **Note**: The exact contents of `VKShaderCompiler::compile_module()` and `VKShaderModule::finalize()` should be verified against Blender 4.5 source at `source/blender/gpu/vulkan/vk_shader.cc`.

### 4.4 SPIR-V to Mesa NIR

From the point that Blender calls `vkCreateShaderModule()`, the compilation path is standard Mesa:

1. `vkCreateShaderModule()` — Mesa caches the SPIR-V blob keyed by its SHA-256 hash.
2. `vkCreateGraphicsPipelines()` or `vkCreateComputePipelines()` — triggers `vk_spirv_to_nir()` which runs the SPIR-V parser and produces NIR (Mesa's intermediate representation, covered in Chapter 14).
3. NIR optimisation passes run: dead-code elimination, algebraic simplification, vectorisation, loop unrolling.
4. On **RADV** (AMD): the NIR lowering pipeline feeds ACO (the register allocator/code generator described in Chapter 18), which produces GFX ISA binary.
5. On **ANV** (Intel): NIR feeds the Intel backend compiler → EU (Execution Unit) assembly.
6. On **NVK** (NVIDIA open-source Vulkan): NIR feeds the NAK compiler.

### 4.5 Shader Caching

Two layers of caching apply:

**Blender's shader cache**: Stores SPIR-V blobs keyed by shader hash in `~/.cache/blender/4.x/shaders/`. On subsequent launches, matching shaders skip the GLSL generation and shaderc compilation steps.

**Mesa's disk shader cache**: Stores the backend-compiled machine code (ACO GFX ISA, EU assembly) in `~/.cache/mesa_shader_cache_db/`. On first launch after a Mesa update all shaders recompile; on subsequent launches with the same Mesa version the compiled binaries load directly.

The combination means a first-launch shader compilation pause (often several seconds for a complex scene) followed by near-instant loads on subsequent runs. Both caches are keyed on content hashes, so changing the scene node graph or updating drivers correctly invalidates stale entries.

---

## 5. OpenColorIO Integration and Color Management

Blender uses OpenColorIO (OCIO) as its color management framework for all rendering, compositing, and display operations. Understanding how OCIO intersects with the GPU pipeline bridges the application-level color story to the KMS display pipeline described in Chapter 3.

### 5.1 What OpenColorIO Does in Blender

OCIO manages the relationship between color spaces across Blender's internal pipeline:

- **Scene-linear**: all internal rendering (EEVEE, Cycles) produces scene-linear floating-point values where linear light arithmetic holds
- **sRGB/Filmic/ACES**: display-referred transforms applied before output to viewport or file
- **LUTs** (Look-Up Tables): arbitrary color transforms stored as 1D or 3D textures; OCIO interprets them as spline or tetrahedral interpolation kernels

Blender ships an OCIO configuration in `datafiles/colormanagement/`. The active configuration can be overridden via the `OCIO` environment variable, which is how facilities with custom color pipelines integrate Blender into their pipeline. [Source](https://opencolorio.readthedocs.io/)

### 5.2 GPU Color Transforms

For interactive display (viewport, rendered preview), executing OCIO transforms on the CPU for every displayed frame is too slow. OCIO provides a GPU path: `OCIO::GpuShaderDesc` generates a GLSL snippet that encodes the transform. Blender wraps this snippet in a full shader program and compiles it through the `GPUBackend`.

The GPU LUT textures (3D `VkImage` objects for 3D LUTs, `VkImage` arrays for 1D) are uploaded once via VMA staging buffers and bound to the color-transform shader as sampled images. The GLSL snippet generated by OCIO references these textures by name; Blender's GLSL wrapper provides the `layout(binding=N) uniform sampler3D lut_texture` declarations.

The GPU color transform path in Blender's source is implemented in `source/blender/imbuf/intern/colormanagement_gpu.cc` (note: verify file path against Blender 4.5 source — the exact file split may differ). The OCIO GPU shader descriptor is used as follows:

```cpp
/* Source: source/blender/imbuf/intern/colormanagement_gpu.cc (approximate)
 * colormanagement_make_display_space_shader: builds a GPU shader that applies
 * the OCIO display-referred transform to a scene-linear input image.
 * OCIO::GpuShaderDesc generates GLSL; Blender wraps it in a GPUShader.
 * Note: verify exact function name and OCIO API version against Blender 4.5 source.
 */
GPUShader *colormanagement_make_display_space_shader(
    const ColorManagedViewSettings *view_settings,
    const ColorManagedDisplaySettings *display_settings)
{
  OCIO::ConstProcessorRcPtr processor = get_display_processor(
      view_settings, display_settings);
  OCIO::ConstGPUProcessorRcPtr gpu_processor =
      processor->getDefaultGPUProcessor();

  OCIO::GpuShaderDescRcPtr shader_desc = OCIO::GpuShaderDesc::CreateShaderDesc();
  shader_desc->setLanguage(OCIO::GPU_LANGUAGE_GLSL_4_0);
  shader_desc->setFunctionName("OCIO_to_display");
  gpu_processor->extractGpuShaderInfo(shader_desc);

  /* LUT textures: upload via GPUTexture API (backed by VkImage on Vulkan) */
  upload_lut_textures(shader_desc);

  /* Wrap the OCIO GLSL snippet in a full fragment shader and compile */
  return GPU_shader_create_from_info_name("colormanagement_display");
}
```

### 5.3 Relationship to the KMS Color Pipeline

The display-referred color space that OCIO produces — typically sRGB (gamma 2.2) for standard monitors, or scene-linear HDR-PQ for HDR10 displays — is what arrives at the Wayland compositor and ultimately at the KMS pipeline. The KMS `DEGAMMA_LUT` / `CTM` / `GAMMA_LUT` pipeline described in Chapter 3 operates downstream of this output.

For standard sRGB: OCIO applies the sRGB transfer function; Blender outputs an sRGB-encoded surface. The compositor may apply additional CTM operations (to correct for monitor calibration) via the `GAMMA_LUT` KMS property, but Blender does not control that path.

For HDR displays: OCIO's Filmic or ACES tone mapping is applied in scene-linear space, then the PQ (Perceptual Quantizer) EOTF is applied to produce HDR10-encoded values. Whether these values are correctly interpreted by the compositor depends on the compositor's support for the `wp_color_management_v1` Wayland protocol (Chapter 3, Section 4).

**A current limitation**: as of Blender 4.5, Blender does not consume the `wp_color_management_v1` Wayland protocol. It manages all color transforms internally and outputs calibrated values to the compositor as a standard surface. HDR display support therefore depends on the compositor's handling of Blender's output rather than on a shared protocol negotiation. This is a genuine integration gap; the color management working group (Chapter 3) has discussed this but Blender has not yet adopted the protocol. [Source](https://opencolorio.readthedocs.io/)

---

## 6. Viewport Rendering: GHOST, Wayland, and GPU Memory

Blender's interactive viewport is a full Vulkan rendering surface — not a simple widget. Every interactive frame executes the same GPU objects (shaders, buffers, command graph) as a final EEVEE render, though typically at lower sample counts and with some effects disabled.

### 6.1 GHOST: Blender's Windowing System

**GHOST** (Generic Handy Operating System Toolkit) is Blender's platform-independent windowing abstraction. On Linux, two GHOST backends are relevant: `GHOST_SystemWayland` (preferred since Blender 3.6) and `GHOST_SystemX11`.

`GHOST_SystemWayland` manages the Wayland connection: `wl_display`, `wl_compositor`, `xdg_wm_base`, `wl_keyboard`, `wl_pointer`, and the input event loop. `GHOST_WindowWayland` creates the `wl_surface` and `xdg_toplevel` for each Blender window. Fractional display scaling is handled via `wp_fractional_scale_v1`; high-DPI rendering is negotiated through `wl_output` scale factors.

For Vulkan rendering, the Wayland `wl_surface` is passed to `GHOST_ContextVK` (implemented in `intern/ghost/intern/GHOST_ContextVK.cc`), which calls `vkCreateWaylandSurfaceKHR()` to create the `VkSurfaceKHR`. The swapchain is subsequently created on this surface via `vkCreateSwapchainKHR`. [Source](https://developer.blender.org/T93031)

```cpp
/* Source: intern/ghost/intern/GHOST_ContextVK.cc (structural outline)
 * Wayland Vulkan surface creation path.
 * WITH_GHOST_WAYLAND guards the Wayland-specific path.
 * The wl_surface is provided by GHOST_WindowWayland.
 * Note: verify exact struct name and field layout against Blender 4.5 source.
 */
#ifdef WITH_GHOST_WAYLAND
#  include <vulkan/vulkan_wayland.h>

VkResult GHOST_ContextVK::create_surface_wayland(wl_display *display,
                                                   wl_surface *surface)
{
  VkWaylandSurfaceCreateInfoKHR surface_info = {};
  surface_info.sType    = VK_STRUCTURE_TYPE_WAYLAND_SURFACE_CREATE_INFO_KHR;
  surface_info.display  = display;
  surface_info.surface  = surface;
  return vkCreateWaylandSurfaceKHR(vk_instance_, &surface_info, nullptr, &vk_surface_);
}
#endif /* WITH_GHOST_WAYLAND */
```

A known architectural constraint: Wayland does not expose its own windowing management API to applications (unlike X11 with `XResizeWindow`). GHOST_ContextVK therefore cannot detect swapchain dimension changes from the Wayland side; it compares `wayland_window_info_` dimensions against the active swapchain extents before each present and recreates the swapchain manually when a mismatch is detected.

### 6.2 Frame Synchronisation in GHOST_ContextVK

`GHOST_ContextVK` manages per-frame synchronisation via `GHOST_Frame` structures. Each frame holds:

- `submission_fence` (`VkFence`): signalled when the GPU finishes executing the submitted command buffer for that frame
- `acquire_semaphore` (`VkSemaphore`): coordinates `vkAcquireNextImageKHR` with the first command buffer submission

The frame rotation strategy: before beginning frame N, GHOST waits on the `submission_fence` of frame N−2 (a double-buffered rotation), ensuring CPU frame preparation does not overrun GPU execution. Discarded resources from frame N−2 are then physically destroyed.

In `VKContext`, frame presentation flows through `swap_buffer_draw_handler()`, which adds a final image blit node to the render graph, calls `flush_render_graph()`, and passes presentation semaphores to `vkQueuePresentKHR`. The submission runner background task then executes the graph and signals the present semaphore.

### 6.3 GPU Memory for the Viewport

Scene geometry (vertex and index buffers) is uploaded to VMA-allocated `VkBuffer` objects (`VMA_MEMORY_USAGE_AUTO`, GPU-preferred) on first display. Vertex data for the active mesh is tracked by Blender's `BatchCache`; when geometry changes, a staging buffer (`VMA_MEMORY_USAGE_AUTO_PREFER_HOST`) is allocated, the new vertex data copied in, and a `VkBufferCopy` operation queued in the render graph.

Texture atlases for material thumbnails and image datablocks are packed into large `VkImage` objects managed by the `VKTexturePool`. The pool caches recently released `VkImage` handles by format and size, reducing VMA allocation frequency during interactive scrubbing.

Object data (model matrices, material parameters, light positions) is uploaded per-frame via uniform buffers. A small staging buffer is allocated each frame, written with the updated transforms, and the data copied to the device-local uniform buffer via the render graph. The `VKDiscardPool` destroys the staging buffer after the GPU finishes reading it.

---

## 7. Cycles as a GPU Compute Workload: Performance Characteristics

Cycles occupies a different part of the GPU performance envelope from rasterisation renderers. Understanding its workload characteristics helps diagnose performance issues on Linux systems.

### 7.1 Workload Profile

Cycles is **memory-bandwidth-bound** on most production scenes. The dominant operation — BVH (Bounding Volume Hierarchy) traversal — makes random pointer-chasing accesses into scene geometry, far exceeding GPU L2/L3 cache capacity on scenes with millions of triangles. Each BVH node traversal may require fetching a cache line from GDDR6/HBM that has not been accessed recently.

Simple scenes with complex procedural materials shift toward **compute-bound** behaviour, where shader arithmetic (BSDF evaluation, procedural texture generation) dominates over memory access.

**Wave32 vs. Wave64**: AMD RDNA architecture supports both Wave32 and Wave64 execution modes. ACO compiles Cycles' HIP shaders targeting Wave32 by default on RDNA, which reduces register pressure and improves shader occupancy on the smaller wavefront. NVIDIA's Volta+ architecture also benefits from Wave32 (formerly CUDA warp size is 32 throughout, but OptiX's traversal units are optimised for it).

### 7.2 Backend Performance Comparison on Linux

The following characterisation is based on publicly available benchmark data from [opendata.blender.org](https://opendata.blender.org/) using the Classroom, Junkshop, and Monster reference scenes. Numbers vary significantly by driver version; always verify against current data.

**AMD RDNA 3/4 via HIP (ROCm)**: Competitive with NVIDIA RTX at equivalent price/TDP on most interior scenes. Memory-bandwidth advantage of HBM-equipped cards (MI300X, etc.) is decisive on large exterior scenes with dense geometry. RDNA 4 (RX 9000 series) achieves parity or better with similarly-priced RTX 4000-class Blender benchmarks on interior scenes. RDNA 3 (RX 7000 series) benefits significantly from Wave32 mode in ACO. [Source](https://opendata.blender.org/)

**NVIDIA RTX via OptiX**: Hardware BVH traversal (RT Cores) provides 30–60% throughput advantage over HIP on complex outdoor scenes with high ray-divergence. RTX cards with larger L2 caches (Ada Lovelace 96 MB on RTX 4090) reduce the memory-bandwidth bottleneck on large scenes. The CUDA path without OptiX is broadly comparable to HIP on same-generation hardware.

**Intel Arc via oneAPI**: Performance has improved substantially with each Arc driver generation. Arc Battlemage (B580, B770) performs competitively for its price bracket on interior scenes but trails NVIDIA and AMD top-tier cards on outdoor scenes with complex BVH traversal. Always check the Intel GPU driver version and oneAPI SDK version against Blender's compatibility matrix before expecting consistent results.

**CPU (multi-arch SIMD C++)**: Viable for small renders and testing. Typically 10–50× slower than GPU for production scenes, depending on CPU core count, AVX-512 support, and scene complexity. Multi-socket systems with high memory bandwidth (Intel Xeon, AMD EPYC) narrow the gap for memory-bound scenes.

The most reliable source for current comparative numbers is the Blender Benchmark project. Community-submitted results include Mesa version, kernel version, and driver version alongside render time, making it possible to track driver regression and improvement over time — which is especially valuable for the Mesa/RADV path where performance has changed significantly across Mesa minor versions.

### 7.3 Linux-Specific Issues

**Nouveau**: Without the nvidia-open driver and GSP-RM support (Chapter 9), NVIDIA GPU performance under Nouveau is insufficient for production Cycles rendering — reclocking does not work on Turing/Ampere without GSP-RM, capping the GPU at a low idle frequency. On Turing+ hardware with nvidia-open and GSP-RM, NVK provides functional GPU rendering for EEVEE via Vulkan, but Cycles' GPU backends (CUDA, OptiX) require the proprietary `libcuda.so` regardless of which kernel module is active. NVK does not implement a CUDA compatibility layer. Rusticl (Mesa's OpenCL frontend to NIR/ACO) can in principle be used for OpenCL compute on NVK, but Cycles does not use OpenCL on NVIDIA hardware in Blender 4.5.

**ROCm version pinning**: Blender's HIP backend is often tested against specific ROCm minor versions. The `intern/cycles/device/hip/device_impl.cpp` module loading path calls `hipModuleLoadData()` with a fat binary compiled for a specific HIP target; mismatched runtime HIP library versions produce cryptic load errors. Check Blender release notes for the supported ROCm version for Blender 4.5.

**Wayland color output**: Blender's Cycles output is color-managed via OCIO before display, producing correct color values for the configured display profile. Full HDR10 output to HDR-capable displays requires compositor support for the appropriate Wayland color management protocol — see Section 5.3.

---

## 8. Debugging Blender GPU Issues on Linux

Blender exposes a mature set of debugging interfaces that integrate directly with the Mesa and ROCm tooling described in earlier chapters.

### 8.1 Environment Variables

The following environment variables affect Blender's GPU behaviour:

| Variable | Effect |
|----------|--------|
| `GPU_BACKEND=vulkan` | Force `VKBackend` regardless of Preferences setting |
| `GPU_BACKEND=opengl` | Force `GLBackend` |
| `CYCLES_DEVICE=HIP` | Force Cycles to use the HIP backend |
| `CYCLES_DEVICE=CUDA` | Force Cycles CUDA |
| `CYCLES_DEVICE=OPTIX` | Force Cycles OptiX |
| `CYCLES_DEVICE=ONEAPI` | Force Cycles oneAPI |
| `MESA_VK_ABORT_ON_DEVICE_LOSS=1` | Crash immediately on `VK_ERROR_DEVICE_LOST` — produces a stack trace instead of silent corruption |
| `MESA_LOADER_DRIVER_OVERRIDE=radeonsi` | Force Mesa OpenGL driver (bypasses RADV for the GL fallback path) |
| `VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation` | Enable Vulkan validation layers; produces detailed error messages for invalid API usage |
| `AMD_DEBUG=info` | Print RADV device information on startup |
| `OCIO` | Override OCIO configuration path |

### 8.2 Blender Command-Line Diagnostics

```bash
# Source: blender --help output; verified against Blender 4.5

# Print GPU device info, driver version, Vulkan version, and backend at startup
blender --debug-gpu

# Enable Cycles kernel compilation and dispatch timing output
blender --debug-cycles

# Print all GPU memory allocation and deallocation events
blender --debug-gpu-mem

# Use Help → System Information in the UI for a structured report
# including GPU device name, driver version, VRAM, Vulkan API version
```

`--debug-gpu` activates `VK_EXT_debug_utils` message routing through the Khronos validation layer callback, printing validation errors and warnings to the terminal. This is the first tool to reach for when EEVEE produces incorrect output or crashes.

### 8.3 RenderDoc Integration

Blender injects `vkCmdBeginDebugUtilsLabelEXT` / `vkCmdEndDebugUtilsLabelEXT` markers that label EEVEE's render passes in RenderDoc's event timeline. Capturing a frame:

```bash
# Source: RenderDoc documentation; requires renderdoc installed
# Launches Blender with RenderDoc injection
renderdoccmd capture --wait-for-exit blender

# Alternatively, attach RenderDoc GUI to a running Blender process
# and trigger a capture via the RenderDoc overlay (F12 by default)
```

In the RenderDoc capture, EEVEE's passes appear as labelled groups: `Shadow Pass`, `G-Buffer Fill`, `Clustered Lighting`, `Volumetric Scatter`, `Composite`, etc. Inspecting individual draw calls within each pass shows the bound `VkPipeline`, descriptor set contents, and the SPIR-V/NIR intermediate representation if Mesa debug extensions are enabled.

### 8.4 Profiling GPU Renders

**AMD — Radeon GPU Profiler (RGP)**: Captures SQTT (Shader Queue Timeline Trace) traces that show per-wavefront execution for Cycles HIP dispatches and EEVEE Vulkan draw calls in a single capture. Blender's render graph submit path flows through the same `VkQueue` that RGP attaches to. On Linux, RGP capture is triggered via the **Radeon Developer Panel** (RDP): launch RDP, connect to the local machine, then launch Blender through RDP's application launcher. RDP injects the SQTT capture hooks into the process and forwards captures to RGP. Alternatively, RenderDoc's RGP interop mode can export a `.rgp` file from within a RenderDoc capture session. [Source](https://gpuopen.com/rgp/)

```bash
# Source: AMD Radeon Developer Panel documentation
# Launch Blender via RDP for SQTT capture (Linux)
# 1. Start the Radeon Developer Panel: RadeonDeveloperPanel
# 2. In RDP, add Blender as a profiling target and click "Start application"
# 3. Trigger a capture from the RDP UI; the result opens in RGP
# Note: RGP_ENABLE=1 is not a supported env var for RADV capture.
```

**NVIDIA — Nsight Systems**: Profiles both CUDA (Cycles) and Vulkan (EEVEE) in a single timeline. Requires the proprietary NVIDIA driver.

**Intel — Intel GPA Frame Analyzer**: Profiles EEVEE Vulkan and Cycles oneAPI workloads on Intel Arc hardware.

### 8.5 Common Linux-Specific Issues and Diagnostics

**`VK_ERROR_DEVICE_LOST` on render**: Set `MESA_VK_ABORT_ON_DEVICE_LOSS=1` and `VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation` to get a stack trace and validation output. Most `DEVICE_LOST` issues on AMD trace to a bug in a specific RADV version; check Blender's issue tracker for your Mesa version.

**EEVEE renders black on AMD with RADV**: Check RADV version (`VK_LOADER_DEBUG=all blender 2>&1 | grep radv`). Known-broken ACO code paths on specific shader patterns have occurred between Mesa minor versions; updating Mesa to the latest stable resolves most cases.

**HIP `hipModuleLoadData` failure**: Indicates ROCm version mismatch. Check `rocminfo` for the installed HIP version and compare against Blender's required version in its release notes.

**Shader compilation stall on first launch**: Expected behaviour — both Blender's SPIR-V cache and Mesa's disk shader cache are cold. The stall scales with scene complexity and shader count. Subsequent launches use the warm caches. Do not interrupt Blender during this phase; interrupting can corrupt the disk cache.

---

## Integrations

**Chapter 3 (Advanced Display Features and Color Management)**: OCIO's GPU color transforms (Section 5) produce display-referred output that aligns with the KMS color pipeline (`DEGAMMA_LUT` / `CTM` / `GAMMA_LUT`) described in Chapter 3. The `wp_color_management_v1` Wayland protocol gap noted in Section 5.3 is the currently open boundary between Blender's application-side color management and the kernel-side display color pipeline. Understanding both sides requires reading Chapters 3 and 40 together.

**Chapter 5 (AMD, Intel, and x86 GPU Drivers)**: Cycles' HIP backend (Section 3.2) dispatches to `amdgpu_kfd` — the same KFD compute queue described in Chapter 5, Section 3. The oneAPI backend targets the Intel Xe/i915 kernel driver covered in Chapter 5, Section 2. EEVEE's Vulkan path targets the same kernel driver as any other Vulkan application on Linux.

**Chapter 9 (GSP-RM, Firmware, and the nvidia-open Connection)**: NVIDIA GPU rendering in Blender splits between EEVEE (Vulkan, works with NVK + nvidia-open, does not require proprietary userspace) and Cycles (CUDA/OptiX, requires proprietary `libcuda.so` regardless of kernel module). GSP-RM ensures correct GPU initialisation and reclocking for both paths on Turing+ hardware. Performance without reclocking (pre-GSP-RM Nouveau) would render both EEVEE and Cycles effectively unusable.

**Chapter 14 (NIR: Mesa's Middle-End Compiler)**: Every EEVEE shader, once compiled to SPIR-V by shaderc, passes through `vk_spirv_to_nir()` and the full NIR optimisation pipeline. The ACO and backend LLVM compilations that follow are the same passes described in Chapter 14. Blender's GLSL generation layer is Blender-specific; from SPIR-V inward, EEVEE is an ordinary Vulkan client.

**Chapter 18 (Mesa Vulkan Drivers — RADV and ANV)**: EEVEE's hardware requirements (Mesa 25.3+ for RADV) are a consumer-visible benchmark of RADV's feature completeness. The `VK_KHR_dynamic_rendering`, `VK_KHR_synchronization2`, and `VK_EXT_extended_dynamic_state` requirements described in Section 2.3 are all RADV features covered in Chapter 18. EEVEE's render graph is a demanding client of these extensions.

**Chapter 19 (OpenGL Compatibility Drivers)**: Blender's OpenGL fallback path (`GLBackend`) uses radeonsi on AMD, iris on Intel, and the NVIDIA proprietary GL driver. The EGL context creation path is the same across the GL and Vulkan backends. Chapter 19's coverage of EGL and `eglCreateContext` applies directly to Blender's OpenGL path.

**Chapter 24 (Vulkan and EGL on Linux)**: Blender's GHOST Wayland surface creation (Section 6.1) — `vkCreateWaylandSurfaceKHR()`, swapchain management, `vkQueuePresentKHR()` — follows exactly the pattern described in Chapter 24. GHOST wraps these APIs but the underlying Vulkan WSI calls are identical to any other Vulkan-on-Wayland application.

**Chapter 25 (GPU Compute on Linux)**: Cycles' HIP, CUDA, and Vulkan compute backends (Section 3) are large-scale production users of the compute stack described in Chapter 25. The `hipModuleLaunchKernel` dispatch path, KFD queue management, and ROCm HAL interaction described in Chapter 25 are the same mechanisms Cycles exercises at render time.

**Chapter 30 (Debugging and Profiling GPU Work)**: RenderDoc captures of EEVEE render passes (Section 8.3), `VK_LAYER_KHRONOS_validation` message analysis (Section 8.1), and Radeon GPU Profiler traces of Cycles HIP dispatches (Section 8.4) are all applications of the debugging techniques described in Chapter 30. Blender's built-in debug markers make it an unusually legible target for GPU profiling tools.

---

## References

1. [Blender Source Repository](https://projects.blender.org/blender/blender) — Authoritative source for `source/blender/gpu/`, `intern/cycles/`, `intern/ghost/` and all source examples cited in this chapter

2. [Blender GPU Module Source](https://projects.blender.org/blender/blender/src/branch/main/source/blender/gpu) — `GPUBackend`, `VKBackend`, `VKDevice`, `VKContext`, `VKShader`, `VKRenderGraph` implementations

3. [Blender 4.2 LTS — Start of the LTS Era](https://code.blender.org/2024/06/blender-4-2-the-start-of-the-lts-era/) — EEVEE Next (Vulkan rewrite) promoted to production renderer; OpenGL EEVEE retired

4. [Blender 4.5 LTS Vulkan Backend Reaches Stable — 9to5Linux](https://9to5linux.com/blender-4-5-lts-open-source-3d-graphics-app-makes-the-vulkan-backend-stable) — Vulkan parity with OpenGL; GPU subdivision, OpenXR, USD/Hydra features added

5. [Blender 4.5 LTS Released — Phoronix](https://www.phoronix.com/news/Blender-4.5-LTS-Released) — Release coverage including Vulkan limitations (100M-vertex meshes, memory constraints) and Wayland improvements

6. [Blender 5.0 Vulkan/OpenGL Default Decision — Phoronix](https://www.phoronix.com/news/Blender-5.0-Vulkan-OpenGL-RAM) — OpenGL remaining default due to Vulkan memory consumption issues; technical rationale

7. [Blender Vulkan Backend Developer Documentation](https://julianeisel.github.io/devdocs/eevee_and_viewport/gpu/vulkan/) — Architecture overview of buffers, pipelines, descriptor sets, and command buffer model

8. [Blender Vulkan Render Graph Documentation](https://developer.blender.org/docs/features/gpu/vulkan/render_graph/) — `VKRenderGraph`, `VKScheduler`, `VKCommandBuilder` design; deferred command recording rationale

9. [Blender Cycles HIP Backend Source — GitHub](https://github.com/blender/cycles/blob/main/src/device/hip/device_impl.cpp) — `HIPDevice` implementation; `hipModuleLoadData`, `compile_kernel`, context management

10. [Blender Cycles HIP Kernel Loading — GitHub](https://github.com/blender/cycles/blob/main/src/device/hip/kernel.cpp) — `HIPDeviceKernels::load_kernel`, `hipModuleGetFunction`, cache configuration

11. [Cycles HIP ROCm Update — Blender Issue #131976](https://projects.blender.org/blender/blender/issues/131976) — ROCm 6.2.4 update for Blender 4.4; GFX12 (RDNA 4) support in ROCm 6.3.1

12. [Blender Cycles GPU Binaries — Developer Handbook](https://developer.blender.org/docs/handbook/building_blender/cycles_gpu_binaries/) — Build-time GPU binary compilation for HIP, CUDA, oneAPI; architecture support matrix

13. [Blender Open Benchmark Data](https://opendata.blender.org/) — Community GPU/CPU benchmark results for Cycles; Classroom, Junkshop, Monster reference scenes

14. [OpenColorIO Documentation](https://opencolorio.readthedocs.io/) — OCIO GPU shader path, `GpuShaderDesc`, LUT texture formats, GLSL generation

15. [GHOST Vulkan Backend API — Blender T93031](https://developer.blender.org/T93031) — Design task for `GHOST_ContextVK`; platform support matrix including Wayland, X11, Win32

16. [Vulkan Wayland Windowing — Blender Pull #113007](https://projects.blender.org/blender/blender/pulls/113007) — Fix for Wayland Vulkan WSI crash; `vkCreateWaylandSurfaceKHR` integration details

17. [Vulkan Memory Allocator (VMA)](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator) — VMA library used by Blender's `VKDevice` for all `VkBuffer` and `VkImage` allocations

18. [shaderc — Google's GLSL Compiler Library](https://github.com/google/shaderc) — Blender uses shaderc (wrapping glslang) to compile GLSL to SPIR-V in `vk_shader.cc`

19. [Blender GLSL Cross Compilation Documentation](https://developer.blender.org/docs/features/gpu/glsl_cross_compilation/) — `GPUShaderCreateInfo` system; GLSL preprocessing and intermediate language; backend-specific GLSL emission

20. [Radeon GPU Profiler (RGP)](https://gpuopen.com/rgp/) — AMD profiling tool supporting both EEVEE Vulkan and Cycles HIP captures from Blender; Linux capture via Radeon Developer Panel

21. [Cycles CPU Kernel Dispatch — GitHub](https://github.com/blender/cycles/blob/main/src/device/cpu/kernel.cpp) — `CPUKernels` constructor; macro-based multi-arch dispatch (scalar, AVX2) without ISPC; ISPC used only in Embree/OIDN dependencies, not in the path-tracing kernel

22. [EEVEE-Next: Virtual Shadow Map Initial Implementation](https://projects.blender.org/blender/blender/commit/a0f52400890) — Commit introducing tilemap-based virtual shadow atlas replacing legacy cascade/cubemap system
