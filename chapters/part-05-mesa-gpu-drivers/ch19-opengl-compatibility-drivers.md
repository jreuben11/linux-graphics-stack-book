# Chapter 19: OpenGL and Compatibility Drivers

**Part V — Mesa GPU Drivers**

OpenGL is thirty years old, yet it remains the rendering API for a substantial fraction of GPU workloads on Linux: native OpenGL games (including many titles from the Steam back catalogue), Wine's D3D8/D3D9 path, scientific and engineering applications, embedded and automotive displays, and X11 applications accelerated through Glamor inside XWayland. This chapter covers the three Mesa components that keep OpenGL alive and performant on modern AMD and Intel hardware: **radeonsi** (AMD's OpenGL/ES Gallium driver), **iris** (Intel's modern Gallium driver for Gen8+), and **Zink** (Mesa's Vulkan-backed OpenGL implementation). Together they represent the concrete realisation of the Gallium3D abstractions introduced in Chapter 13, now running against real hardware through the amdgpu and i915/xe kernel drivers described in Chapter 5.

---

## 1. The Gallium Pipe Driver Interface in Practice

Chapter 13 introduced Gallium3D's three central abstractions — `pipe_screen`, `pipe_context`, and `pipe_resource` — as interfaces between the hardware-independent Mesa state tracker and hardware-specific driver code. This section explains how radeonsi and iris satisfy those interfaces in practice.

### Registration and Loader Entry Points

Every Gallium pipe driver exposes a single C entry point that the Mesa driver loader (Chapter 12) calls to obtain a `pipe_screen *`. For radeonsi this entry point is `radeonsi_screen_create()` in `src/gallium/drivers/radeonsi/si_pipe.c`. For iris it is `iris_screen_create()` in `src/gallium/drivers/iris/iris_screen.c`. The loader discovers these symbols by name after `dlopen()`-ing the driver `.so`; the `GALLIUM_DRIVER` environment variable can override which driver `.so` is opened, enabling Zink to be substituted at runtime with `GALLIUM_DRIVER=zink`.

Inside `radeonsi_screen_create()`, the driver allocates an `si_screen` — a struct that embeds a `pipe_screen` as its first member — and fills in every function pointer on `pipe_screen`: `get_name`, `get_vendor`, `get_param`, `resource_create`, `context_create`, and so on. The same pattern applies to iris's `iris_screen`. This C-struct-as-vtable layout is Gallium's design throughout.

### The `pipe_screen` Capability System

`pipe_screen::get_param()` queries hardware capabilities through the `PIPE_CAP_*` enumeration. A driver returns an integer from this function for each cap — for example `PIPE_CAP_MAX_TEXTURE_2D_SIZE`, `PIPE_CAP_COMPUTE`, `PIPE_CAP_TGSI_VS_LAYER_VIEWPORT`. The state tracker reads these caps during context creation and at extension-query time to decide which OpenGL extensions to advertise.

Shader capabilities use a parallel `PIPE_SHADER_CAP_*` enumeration queried through `get_shader_param()`: maximum number of uniforms, maximum input/output registers, whether the driver supports integers in shaders, and so on. A driver that returns 0 for `PIPE_SHADER_CAP_MAX_SHADER_BUFFERS` disables `GL_ARB_shader_storage_buffer_object` at the state tracker level.

Source paths for these cap tables: `src/gallium/include/pipe/p_defines.h` for the enumerations; `src/gallium/drivers/radeonsi/si_get.c` for radeonsi's answers; `src/gallium/drivers/iris/iris_screen.c` for iris's answers.

### `pipe_context` and `pipe_resource` Lifecycle

`pipe_context` is created per OpenGL context; it is not thread-safe. All draw calls, state changes, and resource uploads flow through `pipe_context` function pointers. Key operations: `draw_vbo` (the draw call dispatcher), `set_blend_state`, `set_rasterizer_state`, `create_sampler_state`, `texture_subdata`.

`pipe_resource` is the generic buffer/texture object. The lifecycle is: `screen->resource_create()` → use → `context->resource_copy_region()` or `context->transfer_map()` → `screen->resource_destroy()`. Under the hood, every `pipe_resource` in radeonsi is an `si_resource` backed by a GEM buffer object (BO) allocated through the `radeon_winsys` layer in `src/gallium/winsys/amdgpu/`. In iris it is an `iris_resource` backed by a GEM BO through the i915 or Xe kernel driver.

The winsys abstraction (`radeon_winsys` for AMD, the iris-internal equivalent for Intel) insulates the driver from kernel-level BO allocation details, providing a clean interface for BO creation, mapping, and command stream attachment — the same boundary at which Chapter 5's kernel driver discussion ends.

---

## 2. radeonsi: AMD OpenGL/ES

### 2.1 Architecture and Supported Hardware

radeonsi replaced the older r600g driver in Mesa and supports every AMD GPU from the GCN1 generation (Southern Islands, GFX6) through the current RDNA4 (GFX12). It is, alongside RADV (Chapter 18), the primary path by which AMD GPUs are used on Linux.

The object hierarchy mirrors the Gallium pattern with AMD-specific extensions:

- `si_screen` (embeds `pipe_screen`): per-GPU global state, GPU info (`radeon_info`), the shader disk cache, LLVM target machine reference
- `si_context` (embeds `pipe_context`): per-context command stream, dirty state bitmask, CSO caches, descriptor ring
- `si_resource` / `si_texture` (embeds `pipe_resource` / `pipe_texture`): GEM BOs with additional AMD-specific metadata (DCC, HTILE)

Hardware generation gating uses the `gfx_level` field (values `GFX6` through `GFX12`) from `radeon_info`, the same enumeration that RADV uses. Code paths guarded by `sscreen->info.gfx_level >= GFX10` versus `>= GFX11` are common throughout radeonsi.

radeonsi and RADV share significant infrastructure under `src/amd/common/` (hardware register definitions, format tables, NIR lowering passes) and `src/amd/llvm/` (the LLVM backend, detailed in Section 8). This sharing means a fix to, say, GFX11 image descriptor construction benefits both drivers simultaneously.

### 2.2 Command Submission and the CS Pipeline

The radeonsi command stream is represented by `radeon_cmdbuf` (historically called `radeon_cs`). Each `si_context` has a primary command buffer for rendering. The winsys layer manages allocation of indirect buffers (IBs) from a ring allocator; the driver writes PM4 packets directly into these IBs.

Flush is triggered by: the IB reaching capacity, an explicit `glFlush` or `glFinish`, or an implicit flush at `SwapBuffers` time. The flush function calls through to `amdgpu_cs_submit_raw2` in `src/gallium/winsys/amdgpu/drm/amdgpu_cs.c`, which packages the IB list into a multi-IB submission and calls `libdrm_amdgpu`'s CS submit interface. Fences are attached per submission; `glFinish` waits on the most recently submitted fence.

Conditional rendering (`glBeginConditionalRender`) is implemented in hardware: the occlusion query result BO is used as a predicate for `DRAW_INDIRECT` with a conditional bit, avoiding a CPU readback.

Transform feedback (streamout) is handled by `si_emit_streamout_begin` for pre-GFX11 hardware. GFX11 (RDNA3) rearchitected NGG streamout, requiring a separate code path in `si_state_draw.cpp` to handle the new packet-level differences.

Source: `src/gallium/drivers/radeonsi/si_state_draw.cpp`, `src/gallium/winsys/amdgpu/drm/amdgpu_cs.c`.

### 2.3 State Management and CSO Caching

Gallium's Constant State Object (CSO) pattern works as follows: the state tracker calls a `create_*_state` function once when a GL state object is created (e.g., on first call to `glBlendFuncSeparate` with a given combination), receiving an opaque CSO pointer. radeonsi pre-compiles the blend equations, rasteriser settings, and depth/stencil state into packed register words at CSO creation time — this defers the register-encoding work away from the draw call hot path.

Key CSO creation functions: `si_create_blend_state`, `si_create_rasterizer_state`, `si_create_depth_stencil_alpha_state`. Each returns a driver-internal struct containing the pre-computed register values.

Dirty state tracking uses `si_context::dirty_states` — a bitmask of flags such as `SI_DIRTY_BLEND`, `SI_DIRTY_RASTERIZER`, `SI_DIRTY_VIEWPORT`. `si_draw_vbo` consults this bitmask and calls the corresponding emit function for each dirty bit before issuing the draw packet. This per-bit emission pattern keeps the critical draw path fast: only changed state is re-emitted.

**Shader variant keys** are central to radeonsi's performance model. A shader's compiled binary is not unique to the GLSL source alone; it is keyed on the combination of source and pipeline state that is baked into the shader (primitive topology, vertex buffer formats, blend state, etc.). The `si_shader_key` struct encodes this information; when any key field changes, radeonsi compiles a new shader variant. This allows the compiler to emit optimised code for specific state combinations — for example, inlining blend constants when they are known at compile time — at the cost of potentially many variants per shader.

```c
// src/gallium/drivers/radeonsi/si_shader.h (simplified)
union si_shader_key {
    struct {
        // Vertex shader key fields
        uint8_t  vs_export_prim_id : 1;
        uint8_t  vs_as_ls          : 1;   // VS used as LS for tessellation
        uint8_t  vs_as_es          : 1;   // VS used as ES for geometry
        // ... vertex buffer format fields per-attrib ...
    } vs;
    struct {
        // Fragment shader key fields
        uint8_t  ps_clamp_color      : 1;
        uint8_t  ps_alpha_to_one     : 1;
        uint32_t ps_blend_func_rgb;       // encoded blend equation
        // ...
    } ps;
};
```

### 2.4 Shader Compilation: NIR Backend

GLSL shaders arrive at radeonsi already translated to NIR by the Mesa state tracker (`st_glsl_to_nir`). radeonsi then runs its own NIR lowering passes before handing the IR to the LLVM backend (Section 8 covers this path in depth).

radeonsi-specific NIR passes include: `si_nir_lower_vs_inputs` (maps vertex buffer descriptors and instance divisors into NIR), `si_nir_lower_ps_color_input` (handles fragment shader colour input interpolation variants), and various GLSL-to-hardware varying packing passes.

Asynchronous shader compilation is implemented via `util_queue`: when a new shader variant is needed, radeonsi enqueues the compilation job on a background thread pool and substitutes a NOP (no-operation) shader for the first few frames until compilation completes. The `RADEONSI_DEBUG=noshaderqueue` flag disables async compilation for debugging — replacing it with synchronous compilation so hangs are easier to attribute.

Source: `src/gallium/drivers/radeonsi/si_shader_nir.c`, `src/amd/llvm/`, `src/amd/common/ac_nir*.c`.

### 2.5 Texture and Sampler Management

`si_texture` extends `pipe_resource` with AMD-specific fields: the GEM BO containing the texel data, a separate BO for DCC metadata, and flag fields recording whether DCC or HTILE is active.

**DCC (Delta Colour Compression)** is AMD's lossless colour compression, available from GFX9 onward on most formats. The DCC metadata buffer stores per-block compression state; a hardware DCC decompression pass is required before formats become accessible to non-compressed paths (e.g., CPU readback, format conversion blits). radeonsi tracks whether DCC is active on a texture and inserts decompress blits where needed — the `si_decompress_dcc` function handles this.

**HTILE** (Hierarchical Depth) provides lossless depth buffer compression and early depth rejection. The HTILE compatibility rules constrain which operations can be performed while HTILE is active; radeonsi disables HTILE when operations violate those rules (e.g., certain depth-stencil copies).

Texture descriptors (`T#` SRDs in GCN terminology) are constructed by `si_make_texture_descriptor()`. Sampler descriptors are constructed by `si_make_sampler_state()`, which includes workarounds for border colour hardware limitations on some GFX generations (border colours are indexed from a hardware table on older GCN, not encoded directly in the sampler descriptor).

### 2.6 Compute in radeonsi: rusticl Integration

rusticl (Mesa's OpenCL implementation, covered in Chapter 25) attaches to radeonsi through the same `pipe_context` interface that OpenGL uses. Compute dispatches use `PIPE_SHADER_COMPUTE` and the `pipe_context::launch_grid()` function pointer, which in radeonsi calls `si_emit_compute_state` and issues a `DISPATCH_DIRECT` PM4 packet.

The integration is seamless from radeonsi's perspective: storage buffers, images, and constant buffers are bound through the same resource descriptor mechanism used for graphics shaders. A compute dispatch and a draw call from the same `si_context` see the same resource binding infrastructure.

### 2.7 Shader DB and Regression Tracking

**Shader DB** is a regression-tracking system maintained alongside Mesa. It consists of a collection of canonical GLSL shader programs and a script that compiles them through the driver's shader compiler, measuring ISA-level metrics: instruction count, VGPR (vector register) count, SGPR (scalar register) count, and scratch memory usage.

The critical metric for GPU performance is VGPR count: GCN GPUs can run more wavefronts concurrently (higher "occupancy") when each wavefront uses fewer VGPRs. A patch that increases VGPR count for a common fragment shader reduces occupancy and causes a measurable performance regression even if no functional test fails.

Before merging a radeonsi patch, developers run shader-db against the full corpus and compare output. A typical regression report looks like:

```text
# shader-db comparison (radeonsi, GFX10 / Navi10)
shaders/game_title/uber_fragment.glsl  VGPR: 48 -> 56  (+8, -16% occupancy)
shaders/game_title/terrain_vs.glsl     VGPR: 24 -> 24  (no change)
shaders/game_title/shadow_ps.glsl      Code: 1024 -> 896  (-12%, regression fixed)
```

The `RADEONSI_DEBUG=vs,ps,cs` flags dump the final compiled ISA for each shader stage to stderr, enabling manual comparison when shader-db output is ambiguous.

Source: `src/tools/glsl_trace/`; shader-db repository: https://gitlab.freedesktop.org/mesa/shader-db.

---

## 3. iris: Intel OpenGL/ES on Gen8+

### 3.1 Architecture and Supported Hardware

iris replaced the older i965 driver as Intel's primary OpenGL/ES implementation. While i965 still handles Gen4 through Gen7.5 (Ivy Bridge, Haswell), iris covers Gen8 (Broadwell) through the current Xe2 (Battlemage) generation. iris uses the same kernel driver (i915 for Gen8–Gen12, the new `xe` DRM driver for Meteor Lake and later) as the ANV Vulkan driver (Chapter 18).

The object hierarchy:

- `iris_screen` (embeds `pipe_screen`): per-GPU state, `intel_device_info`, shader cache
- `iris_context` (embeds `pipe_context`): render and blitter batch rings, dirty state, shader programs, resource binding
- `iris_resource` (embeds `pipe_resource`): GEM BO, tiling/compression metadata

`intel_device_info` is shared between iris and ANV. It encodes the GPU's capability flags, per-subslice thread counts, workaround tables, and constants derived from Intel's PRMs. This shared structure means that hardware detection code and workaround flags are maintained once and benefit both drivers.

Source: `src/gallium/drivers/iris/`.

### 3.2 Batch Execution and Ring Management

`iris_batch` is the analogue of ANV's batch buffer management. Unlike radeonsi, which uses a single command stream, iris maintains **two batch rings per context**:

- `IRIS_BATCH_RENDER`: commands for 3D rendering (draw calls, state packets, resolves)
- `IRIS_BATCH_BLITTER`: commands for the GPU's fixed-function copy engine (fast memory copies, format conversions, clear operations)

Separating the render and blitter rings allows the blitter to run concurrently with rendering on hardware that supports it, and simplifies synchronisation reasoning. The two rings synchronise via `MI_SEMAPHORE_WAIT` packets: the render ring waits on a semaphore value written by the blitter ring after a copy completes, and vice versa.

`iris_batch_flush()` is called to submit outstanding commands. It constructs the final batch buffer, attaches relocations (for Gen8–Gen11 which require relocation) or directly encodes GPU virtual addresses (for Gen12+, which has 48-bit PPGTT), and calls either `DRM_IOCTL_I915_GEM_EXECBUFFER2` (i915 driver) or `xe_exec` (Xe driver). The `xe_exec` path was added in Mesa 24.1 to support Meteor Lake and later hardware running the Xe DRM kernel driver.

Source: `src/gallium/drivers/iris/iris_batch.c`, `src/gallium/drivers/iris/iris_fence.c`.

### 3.3 Surface State Heap and the Binding Table

Intel's 3D hardware uses a fixed-function binding table: an array of indices into a surface state heap buffer object, where each index points to a `RENDER_SURFACE_STATE` structure describing a texture or render target.

`iris_binder` is the iris allocator for the surface state heap BO. Before each draw call, iris constructs a binding table mapping resource bindings to surface state entries. Each entry encodes format, tiling mode, width, height, mip levels, and other surface properties, all packed according to the hardware format described in Intel's Graphics PRMs.

This approach contrasts with ANV's global bindless heap: ANV (using `VK_EXT_descriptor_indexing`) can maintain a persistent descriptor heap and avoid per-draw binding table construction. iris uses per-draw binding tables for OpenGL compatibility — OpenGL's implicit binding model requires re-examining bindings at each draw call, while Vulkan's explicit model allows pre-built persistent descriptor sets.

The performance implication is that binding table construction is in the critical draw-call path for iris. At very high draw-call rates this becomes a measurable CPU overhead, which is one reason iris benefits significantly from `mesa_glthread` (Section 5).

### 3.4 Shader Compilation

iris and ANV share the same Intel compiler infrastructure: `src/intel/compiler/brw_compile_*.c`. This is a purpose-built compiler that lowers NIR directly to EU (Execution Unit) ISA — Intel's internal GPU instruction set — without LLVM as an intermediate. Section 8.7 discusses why this is significant for compile-time latency.

The iris entry points are `iris_compile_vs`, `iris_compile_fs`, `iris_compile_cs`, and their tessellation/geometry equivalents. Each constructs a `brw_vs_prog_key` (or `brw_fs_prog_key`, etc.) from current pipeline state — the same role as radeonsi's `si_shader_key` — and passes it to the shared `brw_compile_*` functions.

Push constants versus pull constants: Intel EU has a fixed-size push constant register file. Small uniform buffer objects (UBOs) whose contents fit in the push register file are inlined at draw time; larger UBOs are accessed via "pull" data port messages during shader execution. The compiler makes this decision based on UBO size at compile time.

The `IRIS_DEBUG=bat,pc,state` flags enable tracing of batch contents, push constants, and state packets, respectively.

```c
// src/gallium/drivers/iris/iris_program.c (simplified)
static void
iris_compile_vs(struct iris_screen *screen,
                struct iris_uncompiled_shader *ish,
                const struct brw_vs_prog_key *key)
{
    struct brw_compile_vs_params params = {
        .base = {
            .mem_ctx    = mem_ctx,
            .nir        = nir,               // NIR from state tracker
            .log_data   = &screen->dev_info,
        },
        .key       = key,
        .prog_data = &prog_data,
    };
    // Calls into src/intel/compiler/brw_compile_vs.c
    const unsigned *assembly = brw_compile_vs(screen->compiler, &params);
    // Store compiled binary in iris_shader_state cache
}
```

### 3.5 Hardware Workarounds in iris

Intel GPUs accumulate errata (bugs and limitations) that software must work around. iris encodes these in the `genX(3DSTATE_*)` emission functions, annotated with workaround identifiers from Intel's internal bug database.

Examples relevant to OpenGL:

- **WA_1607087103** (Gfx11): depth buffer format restrictions; iris emits an additional depth format state packet on Gfx11 to avoid a hardware bug with certain depth format configurations.
- **WA_14010840176** (Gfx12): stencil resolve operation must be done with a specific stencil-only attachment configuration; iris inserts a synchronisation stall before the resolve.

`iris_resolve_color` and `iris_resolve_depth` handle the two resolution operations that Intel hardware requires: render target (MCS) resolves to make render output accessible to sampling, and HiZ (hierarchical Z) resolves to synchronise the depth pyramid with the full-resolution depth buffer.

Gen12 (Tiger Lake) introduced **CCS_E** (Compressed Clear State E), a new multi-sample compression format that changed the semantics of colour clears and resolves. The CCS_E support required significant iris changes landing in Mesa 21.0; older Mesa on Gen12 hardware had known correctness issues with certain render target operations.

Source: `src/gallium/drivers/iris/iris_resolve.c`.

### 3.6 Glamor: 2D X11 Acceleration via OpenGL ES

Glamor is an X server extension that implements X11 2D drawing operations using OpenGL or OpenGL ES rather than a dedicated 2D hardware engine. XWayland uses Glamor to accelerate 2D operations for X11 clients running under a Wayland compositor.

Glamor reaches iris (and radeonsi) through an EGL context created on the DRM render node (`/dev/dri/renderD128`). The context is created with `EGL_OPENGL_ES_API` — specifically GLES2 or GLES3 — which routes through the same `iris_context` code path as any other GLES application. X11 Render `CompositeTriangles` operations become GLES triangle draws in Glamor; `XCopyArea` becomes a textured-quad draw.

The architectural advantage: any GPU with a working GLES implementation gets hardware-accelerated X11 2D for free, without a separate 2D driver. The trade-off: GPU dispatch overhead makes Glamor slower than software for very small, frequent `XCopyArea` operations. Section 9 covers Glamor architecture in full breadth; this section focuses on how iris's GLES path is exercised.

---

## 4. Zink: OpenGL on Vulkan

### 4.1 Design Goals and Architecture

Zink is a Gallium pipe driver that implements the pipe interface not by generating GPU-specific command streams, but by translating Gallium calls into Vulkan API calls. The underlying GPU is addressed entirely through Vulkan.

The motivation is strategic: as new GPU families arrive, they often have Vulkan drivers (via community efforts like Turnip for Adreno, Panfrost for Mali) before native OpenGL drivers. Zink gives those GPUs full OpenGL support immediately. More broadly, Mesa developers view Zink as the long-term path for OpenGL: rather than maintaining separate OpenGL and Vulkan code paths for every GPU family, the OpenGL layer is written once (Zink), and hardware work concentrates on Vulkan drivers.

Zink's position in the stack: it sits between the Mesa state tracker (which produces Gallium calls) and a Vulkan driver (which accepts Vulkan calls). It does not call into any hardware-specific code directly.

Source: `src/gallium/drivers/zink/`.

### 4.2 Translating Gallium to Vulkan

**Resources**: `pipe_resource` maps to either `VkBuffer` (for buffer objects) or `VkImage` (for textures and render targets). Allocation goes through `vkAllocateMemory` (or, on drivers supporting `VK_EXT_memory_budget`, through a Mesa-internal allocator on top of that extension).

**Draw calls**: `pipe_draw_info` translates to `vkCmdDraw` or `vkCmdDrawIndexed`. The base vertex and base instance fields, which Vulkan encodes as push constants rather than as draw call parameters, are uploaded immediately before the draw command.

**Render passes**: Zink builds `VkRenderPass` objects (or uses `VK_KHR_dynamic_rendering` when available) from Gallium framebuffer state. Dynamic rendering avoids the explicit render pass object creation and is preferred when the underlying Vulkan driver supports Vulkan 1.3 or the extension.

**Pipeline objects**: This is Zink's most complex mapping challenge. Gallium CSOs correspond to sub-parts of a Vulkan graphics pipeline. Zink uses `VK_EXT_graphics_pipeline_library` when available to build pipelines incrementally from independently-cached sub-pipelines (vertex input, pre-rasterisation, fragment output, fragment shader), amortising the cost of full pipeline compilation across CSO changes.

```c
// src/gallium/drivers/zink/zink_draw.cpp (simplified call flow)
void zink_draw_vbo(struct pipe_context *pctx,
                   const struct pipe_draw_info *dinfo,
                   ...)
{
    struct zink_context *ctx = zink_context(pctx);

    // 1. Ensure all resource image layouts are correct (insert barriers if needed)
    zink_update_barriers(ctx, false, dinfo->index.resource, ...);

    // 2. Update descriptor sets (lazy or descriptor-buffer mode)
    zink_descriptors_update(ctx, false);

    // 3. Bind the current graphics pipeline
    vkCmdBindPipeline(cmdbuf, VK_PIPELINE_BIND_POINT_GRAPHICS, ctx->gfx_pipeline);

    // 4. Upload push constants (base vertex, base instance, etc.)
    vkCmdPushConstants(cmdbuf, layout, stages, 0, sizeof(push), &push);

    // 5. Issue the draw
    if (dinfo->index_size)
        vkCmdDrawIndexed(cmdbuf, dinfo->count, dinfo->instance_count, ...);
    else
        vkCmdDraw(cmdbuf, dinfo->count, dinfo->instance_count, ...);
}
```

Source: `src/gallium/drivers/zink/zink_draw.cpp`, `src/gallium/drivers/zink/zink_pipeline.c`.

### 4.3 Shader Translation: GLSL → NIR → SPIR-V

GLSL shaders arrive at Zink as NIR (already translated by the state tracker). Zink compiles NIR to SPIR-V using `nir_to_spirv()` in `src/compiler/spirv/nir_to_spirv.c`, then passes the SPIR-V to `vkCreateShaderModule` in the underlying Vulkan driver, which re-compiles it to machine code.

The full round-trip is: GLSL → (Mesa state tracker) → NIR → (Zink) → SPIR-V → (Vulkan driver) → NIR → machine code. This is longer than radeonsi's or iris's path, but the NIR optimisation passes running before SPIR-V emission mean the SPIR-V Zink produces is typically cleaner than raw GLSL-to-SPIR-V translations.

`ZINK_DEBUG=spirv` dumps the emitted SPIR-V for each shader to stderr, useful for diagnosing incorrect shader output or inspecting what Vulkan extensions Zink requires the underlying driver to support.

### 4.4 Synchronisation Model Mismatch

OpenGL's synchronisation model is **implicit**: the driver is responsible for ensuring that a texture written by a draw call is visible when that texture is subsequently sampled. Vulkan's model is **explicit**: the application must insert pipeline barriers between operations that write and read the same resource, specifying source/destination pipeline stages and access masks.

Zink must bridge this gap entirely in software. It tracks the image layout and pending access type for every `VkImage` in a per-resource state object. Before each draw call and before each image read, `zink_resource_image_barrier()` compares the current and required layouts and inserts `vkCmdPipelineBarrier` calls as needed.

```c
// src/gallium/drivers/zink/zink_resource.c (conceptual)
void zink_resource_image_barrier(struct zink_context *ctx,
                                 struct zink_resource *res,
                                 VkImageLayout new_layout,
                                 VkAccessFlags new_access,
                                 VkPipelineStageFlags new_stage)
{
    if (res->layout == new_layout &&
        res->access == new_access)
        return;  // no transition needed

    VkImageMemoryBarrier barrier = {
        .sType            = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER,
        .oldLayout        = res->layout,
        .newLayout        = new_layout,
        .srcAccessMask    = res->access,
        .dstAccessMask    = new_access,
        .image            = res->obj->image,
        .subresourceRange = full_range,
    };
    vkCmdPipelineBarrier(cmdbuf,
                         res->access_stage, new_stage,
                         0, 0, NULL, 0, NULL, 1, &barrier);
    res->layout       = new_layout;
    res->access       = new_access;
    res->access_stage = new_stage;
}
```

Over-synchronisation (inserting barriers that are more conservative than strictly necessary) is the primary correctness-for-performance trade-off in Zink. The synchronisation code has been progressively refined to be less conservative, but it remains one of Zink's headline overheads in high-throughput rendering.

### 4.5 Descriptor Management

Zink implements two descriptor management modes, selected via the `ZINK_DESCRIPTORS` environment variable:

- **LAZY** (default): Descriptor sets are allocated from pools and updated with `vkUpdateDescriptorSets` before each draw call. Zink maintains a cache keyed on the current resource binding state to avoid redundant updates, but the overhead is still proportional to the number of unique binding combinations seen per frame.
- **DB** (descriptor buffer): Uses `VK_EXT_descriptor_buffer` to write descriptors directly into a GPU-visible buffer, bypassing descriptor pool management. This mode reduces CPU overhead for descriptor writes at the cost of requiring driver support for the extension.

Push descriptors (`VK_KHR_push_descriptor`) provide a fast path for frequently-changing bindings — Zink uses them for uniforms and other per-draw data when the extension is available.

`ZINK_DEBUG=descriptor` traces descriptor allocation and update operations.

### 4.6 Performance Ceiling and Known Limitations

The fundamental overhead of Zink's translation layer is CPU-time: state translation, barrier insertion, and descriptor management all happen on the CPU at draw time. In workloads with few draw calls and expensive GPU shaders, this overhead is negligible. In draw-call-limited workloads (many small objects, immediate-mode rendering, display lists), Zink's CPU overhead can cause meaningful performance regression versus native radeonsi or iris.

Areas where Zink is competitive: compute-heavy workloads, large-batch rendering, workloads on hardware that lacks a native GL driver (Mali/Panfrost, Adreno/Turnip), and CI/conformance testing via Lavapipe+Zink.

Zink achieved OpenGL 4.6 conformance with RADV as the backing driver in Mesa 23.1, and with ANV in Mesa 24.0. These represent full official conformance, not just extension coverage. `GALLIUM_DRIVER=zink` and `MESA_LOADER_DRIVER_OVERRIDE=zink` force Zink for testing purposes.

---

## 5. The Long Tail: Maintaining OpenGL 4.6 on Modern Hardware

OpenGL 4.6 is not a legacy API that Linux has abandoned — it is actively required. Steam's Linux game catalogue includes many titles that use OpenGL directly; Wine uses OpenGL for D3D8 and D3D9 games; scientific software (MATLAB, ParaView, Blender's EEVEE legacy path) requires it; automotive and industrial Linux deployments depend on GLES 3.x. The Mesa developers who maintain radeonsi, iris, and Zink are funding full-time OpenGL maintenance.

**The extension burden**: OpenGL 4.6 mandates `GL_ARB_gl_spirv` (SPIR-V shader ingestion), `GL_ARB_bindless_texture` (bindless texture handles), and `GL_ARB_direct_state_access` (DSA). These are not lightweight additions — SPIR-V ingestion requires a full SPIR-V → NIR round-trip path, and bindless textures require hardware support or software emulation of non-indexed texture access.

**Deprecated paths that persist**: The OpenGL compatibility profile requires that `glBegin`/`glEnd` immediate mode, display lists, feedback mode, and selection mode all continue to work. Mesa handles these through the state tracker's compatibility profile emulation layer (`src/mesa/main/`). Coverage is partial: Mesa's compatibility profile is sufficient for most legacy applications but does not pass the full `GL_ARB_compatibility` CTS.

**`mesa_glthread`**: The asynchronous GL thread feature, implemented in `src/mesa/main/glthread.c`, decouples the application's GL API calls from the driver's `pipe_context` operations. When enabled (controlled by driconf or `mesa_glthread=true`), OpenGL calls from the application thread are serialised into a work queue; a background thread dequeues them and calls the actual pipe driver. This reduces stalls on applications whose GL submission is CPU-bound (the application waits for each GL call to return before making the next one).

`mesa_glthread` helps latency-bound single-threaded applications. It can hurt multi-threaded applications or those that call `glFinish` frequently, because `glFinish` on the main thread must flush the queue and wait, adding a round-trip through the queue.

**Version override environment variables**: `MESA_GL_VERSION_OVERRIDE=4.6` forces Mesa to report OpenGL 4.6 even if the driver has not fully qualified it. `MESA_GLSL_VERSION_OVERRIDE=460` similarly overrides the GLSL version. These are useful for testing and for applications that refuse to run if the reported version is below a threshold — but they do not enable extensions the hardware does not support. Using them to run applications that actually exercise unimplemented extension paths will cause crashes or incorrect rendering.

---

## 6. Comparing the Drivers: When to Use Which

**radeonsi vs. Zink-on-RADV for AMD OpenGL**: radeonsi is the production AMD OpenGL driver: mature, fully conformant, with a decades-long history of performance work and the shader-db regression system. Zink-on-RADV is the emerging path; it passed OpenGL 4.6 conformance in Mesa 23.1 and has been improving rapidly. For desktop gaming and mature applications, radeonsi is the right choice today. Zink-on-RADV is preferred for: testing Vulkan driver correctness, running OpenGL on hardware that only has a Vulkan driver, and as a migration path when radeonsi LLVM compilation time is a concern.

**iris vs. Zink-on-ANV for Intel OpenGL**: The same argument applies. iris is the production Intel OpenGL driver with Xe2 support and known-good hardware workarounds. Zink-on-ANV reached OpenGL 4.6 conformance in Mesa 24.0. The direction of travel is the same: Zink is the future, iris is production today.

**Zink as a portability target**: Zink is already the primary OpenGL implementation for Panfrost (Mali GPU family) and was the first OpenGL path for Turnip (Adreno), which later added its own native OpenGL support. For any new GPU family with a working Vulkan driver, Zink provides OpenGL automatically.

**The future trajectory**: Mesa developers are increasingly treating Zink as the universal OpenGL implementation. Long-term, native GL drivers will concentrate on performance-critical paths while Zink handles conformance. This mirrors the industry-wide consolidation around Vulkan as the baseline GPU interface.

---

## 7. OpenGL ES and the Embedded Target

radeonsi and iris support OpenGL ES 3.2 using the same driver code as desktop OpenGL. The distinction is made at context creation time: `eglCreateContext` with `EGL_OPENGL_ES3_BIT` in the config attribute requests a GLES3 context, which Mesa routes to the same `si_context` or `iris_context` with GLES-specific capability reporting.

Important GLES extensions for Android compatibility layers and embedded Linux:

- `GL_OES_EGL_image`: allows EGL images (dmabuf-backed) to be imported as GLES textures; critical for camera, media decode, and compositor integration
- `GL_OES_EGL_sync`: EGL sync objects for cross-process GPU synchronisation; used by gralloc-based buffer sharing
- `GL_EXT_buffer_storage`: persistent, coherent buffer mapping, enabling zero-copy data paths between CPU and GPU

Non-Android embedded Linux contexts include automotive infotainment systems (many use GLES2/3 for cluster and IVI displays), digital signage, and kiosk applications. These often run on Intel Atom or AMD Embedded Radeon hardware where iris and radeonsi are the GPU drivers.

---

## 8. LLVM as a Mesa Compiler Backend

### 8.1 Why radeonsi Uses LLVM (Not ACO)

ACO (the AMD compiler in RADV, described in Chapter 15) was designed for the Vulkan driver's requirements: predictable, low-latency compilation with a self-contained code generator. It was never ported to radeonsi. The reasons are partly historical (radeonsi's LLVM integration was mature before ACO existed) and partly architectural (LLVM's general-purpose optimisations, though slower, can produce marginally better code for some shader patterns that appear more often in OpenGL workloads than in Vulkan workloads).

The practical consequence: radeonsi compiles all shader stages (vertex, fragment, geometry, tessellation, compute) through LLVM. RADV defaults to ACO and reaches LLVM only via `RADV_DEBUG=llvm`, kept as a correctness comparison path.

The OpenGL compilation model also differs from Vulkan's: OpenGL shaders are compiled on first use, with implicit caching. radeonsi's async shader queue (Section 2.4) and disk cache (Section 8.5) mitigate LLVM's startup cost in a way that is less applicable to Vulkan's explicit pipeline model, where the application controls pipeline compilation timing.

### 8.2 The radeonsi LLVM Path

The entry point is in `src/gallium/drivers/radeonsi/si_shader_llvm.c`, which calls into `src/amd/llvm/` for shared AMD LLVM infrastructure. The central function is `nir_to_llvm()` in `src/amd/llvm/ac_nir_to_llvm.c`, which walks NIR instructions and emits LLVM IR using the LLVM C API.

The `ac_llvm_context` struct (in `src/amd/llvm/ac_llvm_build.c`) wraps the LLVM `LLVMContextRef`, `LLVMModuleRef`, and the AMD-specific intrinsic function list. It is shared between radeonsi and RADV's LLVM path, ensuring that both drivers exercise the same backend code when RADV uses LLVM for debugging.

### 8.3 LLVM IR Generation: NIR Intrinsics to GCN Intrinsics

The NIR-to-LLVM translation covers each NIR opcode and intrinsic:

- **Image loads** (`nir_intrinsic_image_load`): emit `llvm.amdgcn.image.load.*` intrinsics with the resource descriptor (T#) as an argument. The intrinsic encodes the image dimension (1D/2D/3D/Cube) and the data format.
- **Atomic operations** (`nir_intrinsic_image_atomic_*`): map to `llvm.amdgcn.image.atomic.*`; buffer atomics use `llvm.amdgcn.buffer.atomic.*`.
- **Interpolation** (`nir_intrinsic_load_interpolated_input`): translates to the two-phase `llvm.amdgcn.interp.p1` / `llvm.amdgcn.interp.p2` intrinsics, matching GCN's hardware interpolation model.
- **Control flow**: NIR structured control flow (`nir_if`, `nir_loop`) is lowered to LLVM basic blocks and branches using `ac_build_if()` / `ac_build_bgnloop()` helper functions in `ac_llvm_build.c`.

Divergence: `nir_divergence_analysis` determines which values are wave-uniform (eligible for SGPRs) versus lane-varying (require VGPRs). The LLVM AMDGPU backend also performs its own divergence analysis, but Mesa's pre-pass guides which code path is taken before LLVM applies its register assignment.

### 8.4 The LLVMTargetMachine for GCN/RDNA

The AMDGPU backend (`lib/Target/AMDGPU/` in the LLVM tree) supports GFX6 through GFX12. Mesa creates the target machine via `ac_create_target_machine()` in `src/amd/llvm/ac_llvm_util.c`, selecting the appropriate `-mcpu=` string:

```text
GFX6  (Southern Islands)  → -mcpu=tahiti
GFX9  (Vega)              → -mcpu=gfx906
GFX10 (RDNA1)             → -mcpu=gfx1010
GFX11 (RDNA3)             → -mcpu=gfx1100
GFX12 (RDNA4)             → -mcpu=gfx1200
```

Target features such as `-mattr=+wavefrontsize64` (wave64 mode, required for pre-GFX10) and `-mattr=+cumode` (CU-mode scheduling) are set in `si_shader_llvm.c` based on the runtime `radeon_info`.

The optimisation level is `O2` for release builds and `O0` for debug. The output is an ELF object; Mesa extracts the `.text` section as the shader binary and reads shader metadata (SGPR/VGPR counts, LDS usage, scratch usage) from ELF `.note` sections.

### 8.5 Compile-Time Cost and Disk Cache Amortisation

LLVM compilation is significantly slower than ACO: a complex fragment shader may take 50–200 ms through LLVM versus 5–20 ms through ACO, with the difference dominated by LLVM's IR optimisation passes. This is the principal motivation for radeonsi's async shader queue — compilation runs on a background thread, and the application sees a NOP shader (no output rendered) for the first few frames of a new shader variant rather than a hard stall.

Mesa's disk shader cache stores compiled ELF binaries keyed on (shader source hash, driver version, GPU family). On a cache hit, LLVM is bypassed entirely; the ELF is loaded from disk and used immediately. This is the radeonsi equivalent of Vulkan's `VkPipelineCache`.

Relevant environment variables:

| Variable | Effect |
|---|---|
| `MESA_SHADER_CACHE_DISABLE=1` | Disables disk shader cache |
| `MESA_SHADER_CACHE_DIR` | Overrides cache directory path |
| `RADEONSI_DEBUG=noshaderqueue` | Forces synchronous (blocking) compilation |

First-run stutter (compiling all shaders on first encounter) is eliminated on subsequent runs from the warm cache. A Mesa upgrade invalidates the cache (the driver version component of the key changes), causing one more first-run stutter cycle.

### 8.6 When LLVM Is Used for RADV

By default, RADV uses ACO for all shader stages in graphics and compute pipelines. LLVM is available via `RADV_DEBUG=llvm` for correctness comparison and debugging. The precise set of cases where RADV delegates to LLVM changes between Mesa releases as ACO's coverage expands; to determine the current boundary, consult `src/amd/vulkan/radv_shader.c` in the Mesa tree at build time.

### 8.7 The Intel Shader Compiler: Not LLVM

iris and ANV use the purpose-built Intel compiler (`src/intel/compiler/brw_*.c`) that lowers NIR directly to EU ISA without LLVM. `brw_compile_vs`, `brw_compile_fs`, and related functions are the compilation entry points.

This makes Intel shader compilation substantially faster than radeonsi's LLVM path — iris historically shows lower first-frame stutter than radeonsi on shader-heavy games. LLVM appears in Intel's Mesa code only in performance measurement infrastructure (`src/intel/perf/`) and offline tooling, not in the hot compilation path.

### 8.8 llvmpipe and Lavapipe LLVM Usage

(Cross-reference Chapter 17 for full coverage.)

llvmpipe JIT-compiles fragment programs to CPU SIMD using LLVM. Tiles are rasterised in software; each tile's fragment program is compiled via the `lp_bld_*` infrastructure in `src/gallium/auxiliary/gallivm/` to use AVX2 or AVX-512 vector instructions. Key files: `lp_bld_arit.c` (arithmetic), `lp_bld_sample.c` (texture sampling), `lp_bld_flow.c` (control flow).

Lavapipe (the llvmpipe-backed Vulkan driver) follows the same path: NIR is lowered through `lp_bld_*` and compiled to CPU SIMD via LLVM. Lavapipe's LLVM dependency is why distributions must ship a compatible LLVM version with Mesa. The minimum LLVM version for llvmpipe is tracked in `src/gallium/auxiliary/gallivm/lp_bld_init.c`.

---

## 9. Glamor: OpenGL-Accelerated 2D for XWayland

Glamor is an X server extension (`glamor/` in the xserver tree) that implements X11 2D drawing operations — `XRender`, `XCopyArea`, `XPutImage`, pixmap compositing — using OpenGL or OpenGL ES rather than a dedicated hardware 2D engine. Eliminating the hardware 2D engine requirement means any GPU with a working GL/GLES implementation gets hardware-accelerated X11 2D for free via Glamor.

### XWayland Integration

`xwayland_glamor_init()` in `hw/xwayland/xwayland-glamor.c` (xserver tree) sets up a Glamor context when XWayland starts. All X11 pixmap operations for X11 clients running under XWayland are routed through Glamor. The EGL context Glamor creates uses `EGL_OPENGL_ES_API` on the DRM render node (`/dev/dri/renderD128`), avoiding root-privilege requirements and separating 2D compositing from display management:

```c
// hw/xwayland/xwayland-glamor.c (skeleton)
static Bool
xwayland_glamor_init(struct xwl_screen *xwl_screen)
{
    // Open the DRM render node
    int fd = open("/dev/dri/renderD128", O_RDWR);

    // Get the EGL display from the render node fd
    EGLDisplay dpy = eglGetDisplay((EGLNativeDisplayType)egl_display_from_fd(fd));
    eglInitialize(dpy, NULL, NULL);

    // Bind OpenGL ES API
    eglBindAPI(EGL_OPENGL_ES_API);

    // Create context — this resolves to iris or radeonsi pipe driver
    EGLint attribs[] = { EGL_CONTEXT_CLIENT_VERSION, 3, EGL_NONE };
    EGLContext ctx = eglCreateContext(dpy, config, EGL_NO_CONTEXT, attribs);

    // Hand context to Glamor
    glamor_init(screen, GLAMOR_USE_EGL_SCREEN);
    return TRUE;
}
```

### How Glamor Translates X11 Operations

X11 Render `CompositeTriangles` and similar operations become GLES triangle draws with Glamor-internal shaders. `XCopyArea` and `XCopyPlane` become textured-quad draws (`glamor_copy.c`). Glamor maintains a small set of GLSL programs compiled at init time; these are cached in the GL driver's shader cache. The shaders are simple enough that LLVM compilation time is not a concern.

### Performance Characteristics

GPU-accelerated 2D via Glamor outperforms software 2D for: large pixmap copies, compositing-heavy applications (Qt with OpenGL disabled, legacy GTK2 apps), and text rendering with many glyphs. Glamor can be slower than software for very small, very frequent `XCopyArea` operations — the GPU dispatch overhead dominates for sub-millisecond operations that a CPU can complete in nanoseconds.

Debugging:

- `GLAMOR_DEBUG=1` (or higher) enables Glamor-internal tracing
- `XWAYLAND_NO_GLAMOR=1` disables Glamor in XWayland to force the software 2D path

Connection to XWayland (Chapter 23): Glamor is the mechanism by which XWayland achieves hardware 2D acceleration. Without Glamor, all X11 pixmap operations would be performed in software by the CPU, making X11 applications running under XWayland visually correct but slow under compositing-heavy workloads.

---

## Summary

This chapter has traced the OpenGL pipeline from the Gallium pipe interface down to hardware in radeonsi and iris, examined Zink's Vulkan-translation approach, and explored the infrastructure (LLVM backend, Shader DB, Glamor) that surrounds these drivers. The key architectural contrasts to keep in mind:

- **radeonsi** uses LLVM for shader compilation, mitigated by async queues and disk cache; state is pre-compiled into CSOs; shader variants encode baked pipeline state.
- **iris** uses Intel's purpose-built NIR-to-EU compiler, giving lower compile-time latency; batch management separates render and blitter rings; per-draw binding tables trade memory bandwidth for OpenGL compatibility.
- **Zink** adds a translation layer but provides OpenGL on any Vulkan driver; its primary overheads are synchronisation barrier insertion and descriptor management.

Forward references: Chapter 20 (Wayland) will show how radeonsi and iris present rendered frames via `eglSwapBuffers` and `linux-dmabuf`; Chapter 23 (XWayland) expands on Glamor's role in XWayland acceleration; Chapter 25 (GPU Compute) shows rusticl's use of the same `pipe_context` that serves OpenGL draw calls; Chapter 28 (Wine) traces how Wine's OpenGL path reaches radeonsi and iris.

---

## References

1. Mesa source — radeonsi: `src/gallium/drivers/radeonsi/` — https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/drivers/radeonsi
2. Mesa source — iris: `src/gallium/drivers/iris/` — https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/drivers/iris
3. Mesa source — Zink: `src/gallium/drivers/zink/` — https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/drivers/zink
4. Mesa source — AMD LLVM shared backend: `src/amd/llvm/` — https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/amd/llvm
5. Mesa source — llvmpipe gallivm: `src/gallium/auxiliary/gallivm/` — https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/auxiliary/gallivm
6. Glamor source in xserver: `glamor/` — https://gitlab.freedesktop.org/xorg/xserver/-/tree/master/glamor
7. XWayland Glamor integration: `hw/xwayland/xwayland-glamor.c` — https://gitlab.freedesktop.org/xorg/xserver/-/blob/master/hw/xwayland/xwayland-glamor.c
8. Kenneth Graunke, "iris: A New Intel OpenGL Driver" — XDC 2019: https://xdc2019.x.org/event/5/contributions/296/
9. Mike Blumenkrantz, "Zink: OpenGL over Vulkan in Mesa" — XDC 2020: https://indico.freedesktop.org/event/2/contributions/46/
10. LWN.net "The iris driver" (2019): https://lwn.net/Articles/793947/
11. LWN.net "Zink: OpenGL on Vulkan" (2020): https://lwn.net/Articles/832340/
12. Mesa Shader DB repository: https://gitlab.freedesktop.org/mesa/shader-db
13. Gallium3D documentation in Mesa tree: `src/gallium/docs/` — https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/docs
14. LLVM AMDGPU backend documentation: https://llvm.org/docs/AMDGPUUsage.html
15. Marek Olšák, "AMD OpenGL performance tuning" — FOSDEM 2020: https://fosdem.org/2020/schedule/event/amd_opengl/
16. Tomeu Vizoso, "rusticl and the Gallium compute pipe" — XDC 2022: https://indico.freedesktop.org/event/2/contributions/67/
17. AMD RDNA3 ISA Reference Guide: https://developer.amd.com/resources/developer-guides-manuals/
18. Intel Graphics Developer Reference Manual (Gen9 through Xe2): https://01.org/linuxgraphics/documentation
19. Marek Olšák, "Optimising radeonsi" — XDC 2018: https://xdc2018.x.org/talks.html
20. Tom Stellard, "LLVM AMDGPU backend overview" — LLVM Developer Conference 2014: https://llvm.org/devmtg/2014-10/#talk18
