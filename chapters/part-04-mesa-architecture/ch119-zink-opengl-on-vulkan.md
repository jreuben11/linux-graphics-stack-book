# Chapter 119: Zink — OpenGL on Vulkan

This chapter targets three audiences: **Mesa contributors** studying how a full OpenGL state tracker is built atop Vulkan; **application developers** who need OpenGL on Vulkan-only drivers (NVK, v3dv, PowerVR, Turnip); and **graphics engineers** interested in the architectural impedance mismatch between OpenGL's implicit-state model and Vulkan's explicit design. Readers should be comfortable with both the Gallium3D driver interface (Chapter 13) and core Vulkan concepts (Chapter 24).

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Architecture Overview](#2-architecture-overview)
3. [OpenGL State to Vulkan Translation](#3-opengl-state-to-vulkan-translation)
4. [Shader Translation: GLSL → NIR → SPIR-V](#4-shader-translation-glsl--nir--spirv)
5. [Descriptor Management](#5-descriptor-management)
6. [Pipeline State Object Caching](#6-pipeline-state-object-caching)
7. [Memory Management and Resources](#7-memory-management-and-resources)
8. [OpenGL Compatibility Profile](#8-opengl-compatibility-profile)
9. [Zink as a Backend for Other Drivers](#9-zink-as-a-backend-for-other-drivers)
10. [Performance and Debugging](#10-performance-and-debugging)
11. [Integrations](#11-integrations)

---

## 1. Introduction

Zink is Mesa's Gallium3D driver that implements OpenGL and OpenGL ES entirely by translating API calls into Vulkan. It emits no GPU-vendor-specific commands: instead of speaking to hardware directly, Zink speaks Vulkan, and any conformant Vulkan ICD becomes Zink's hardware backend. This single design decision gives Zink three distinct practical uses.

**Providing OpenGL to Vulkan-only drivers.** Mesa's newer hardware drivers — NVK for NVIDIA Turing+ GPUs, v3dv for the Raspberry Pi 5's V3D engine, Turnip for Qualcomm Adreno, and more recently the upstream PowerVR Vulkan driver — implement only Vulkan. Rather than writing a separate Gallium GL driver for each of these, Mesa relies on Zink to supply the entire OpenGL API on top of the Vulkan driver. With Mesa 25.1, this strategy reached a milestone: the legacy Nouveau OpenGL driver was retired for Turing-and-later GPUs, replaced by Zink running over NVK as the default open-source OpenGL stack for NVIDIA hardware. [Source: GamingOnLinux — Mesa 25.1 Zink+NVK](https://www.gamingonlinux.com/2025/03/mesa-25-1-will-default-to-zink-nvk-instead-of-the-old-nouveau-opengl-driver-for-nvidia-on-linux/)

**Hardware without a Gallium GL driver.** Embedded and mobile SoCs often have vendor Vulkan ICDs but no open-source OpenGL driver. Zink fills this gap: once a Vulkan driver exists, OpenGL follows with no additional GPU-specific code.

**Conformance target and correctness reference.** Because Zink must faithfully translate every GL call to Vulkan, it serves as a concrete specification of what a GL implementation must produce. Zink ships as a conformant OpenGL 4.6 implementation and is used in Mesa's CI pipeline to verify the Gallium GL state tracker.

Zink lives in `src/gallium/drivers/zink/` within the Mesa repository. [Source: Mesa Zink documentation](https://docs.mesa3d.org/drivers/zink.html)

### A Brief History

Zink was first published by Erik Faye-Lund (Collabora) in October 2018 as a proof-of-concept: at the time, it could render basic geometry using Vulkan but was far from conformant. [Source: Collabora: Introducing Zink (2018)](https://www.collabora.com/news-and-blog/blog/2018/10/31/introducing-zink-opengl-implementation-vulkan/) The original design identified the core challenges that persist today — polygon-mode differences, border-color emulation, pipeline-state hashing overhead — but the initial implementation was intentionally minimal. Over the following years, Mike Blumenkrantz (Valve) became the primary maintainer and rewrote or substantially refactored most of the driver. Major milestones:

- **Mesa 20.2 (2020)**: first upstream merge; OpenGL 2.0 capable.
- **Mesa 21.2 (2021)**: `GL_ARB_sparse_buffer` support; OpenGL ES 3.0.
- **Mesa 22.0 (2022)**: OpenGL 4.6 conformance on Lavapipe and ANV; `GL_ARB_sparse_texture` support.
- **Mesa 22.2 (2022)**: Kopper WSI merged; OpenGL 4.6 CTS passed on RADV.
- **Mesa 23.1 (2023)**: descriptor-buffer (`db`) mode introduced, cutting peak VRAM by >10%.
- **Mesa 25.1 (2025)**: Zink+NVK becomes the default for NVIDIA Turing+ GPUs; legacy Nouveau GL retired.
- **Mesa 26.1 (2026)**: PowerVR Vulkan driver upstream, using Zink as its sole OpenGL implementation.

### Vulkan Version and Extension Requirements

Zink requires a minimum of **Vulkan 1.0** for the most basic OpenGL 2.1 support, escalating to higher GL versions as more Vulkan capabilities become available:

| OpenGL version | Minimum Vulkan features required |
|---------------|----------------------------------|
| 2.1 | Vulkan 1.0 + `VK_KHR_maintenance1`, `VK_EXT_custom_border_color`, `VK_EXT_line_rasterization` |
| 3.0 | `independentBlend` feature + `VK_EXT_transform_feedback`, `VK_EXT_conditional_rendering` |
| 4.0 | `tessellationShader` + `imageCubeArray` |
| 4.6 | `samplerAnisotropy`, `depthBiasClamp`, `VK_KHR_draw_indirect_count` |

[Source: Mesa Zink documentation](https://docs.mesa3d.org/drivers/zink.html)

Achieving the high-performance fast path additionally requires `VK_EXT_descriptor_buffer`, `VK_EXT_extended_dynamic_state{1,2,3}`, `VK_EXT_graphics_pipeline_library`, and `VK_KHR_dynamic_rendering`. All of these are available in RADV on RDNA2+, ANV on Xe+, and NVK on Turing+.

---

## 2. Architecture Overview

Zink is a Gallium driver. In Mesa's layered architecture, the Gallium3D framework separates the OpenGL frontend (the "state tracker") from the hardware-specific backend (the "driver"). Zink occupies the driver slot — it implements the `pipe_driver`, `pipe_screen`, and `pipe_context` interfaces — but instead of talking to a kernel DRM driver, it talks to a Vulkan ICD.

```
Application (OpenGL call)
    │
    ▼
Mesa GL / GLES State Tracker   (src/mesa/state_tracker/st_*.c)
    │   pipe_context ops
    ▼
Gallium3D Framework            (src/gallium/include/pipe/*.h)
    │
    ▼
Zink Gallium Driver            (src/gallium/drivers/zink/)
    │   Vulkan API calls
    ▼
Vulkan ICD (RADV, ANV, NVK, v3dv, Turnip, …)
    │
    ▼
GPU Hardware
```

This mirrors the position of other Gallium drivers such as radeonsi, iris, and etnaviv — Zink simply substitutes a Vulkan ICD for a DRM kernel driver.

### Key Data Structures

**`zink_screen`** (defined in `src/gallium/drivers/zink/zink_types.h`) wraps `VkPhysicalDevice` and `VkDevice`. It holds the Vulkan instance, device handles, queue family index, the device feature and extension flags (`screen->info`), the pipeline cache (`VkPipelineCache`), disk-cache state, and capability information that is translated upward to Gallium's `pipe_screen` interface. Multiple `zink_context` objects share a single `zink_screen`.

**`zink_context`** extends `pipe_context`. It owns the current graphics and compute program state, the draw-state structures, descriptor update state, a pool of `zink_batch_state` objects, and the ring of in-flight command buffers. Every thread gets its own `zink_context`.

**`zink_batch_state`** wraps a `VkCommandBuffer`. Zink maintains multiple batch states per context and uses them in a round-robin pattern protected by timeline semaphores. Each batch state also carries its own descriptor pool and resource-reference sets used for lifetime tracking:

```c
struct zink_batch_state {
    struct zink_fence        fence;
    VkCommandPool            cmdpool;
    VkCommandBuffer          cmdbuf;            /* primary draw commands */
    VkCommandBuffer          reordered_cmdbuf;  /* blits, copies */
    VkCommandPool            unsynchronized_cmdpool;
    VkCommandBuffer          unsynchronized_cmdbuf; /* truly independent work */
    VkSemaphore              signal_semaphore;
    struct util_dynarray     wait_semaphores;
    struct zink_batch_descriptor_data dd;
    bool                     has_work;
    /* ... */
};
```

[Source: FireBurn/mesa zink_types.h](https://github.com/FireBurn/mesa/blob/main/src/gallium/drivers/zink/zink_screen.c)

**`zink_resource`** is the Gallium `pipe_resource` implementation. It wraps either a `VkBuffer` or a `VkImage` (never both; Zink does not alias buffer/image in the same allocation). It contains the underlying `zink_resource_object` with the Vulkan handle, the `zink_bo` memory allocation, and per-layout / per-access tracking fields used for lazy barrier insertion.

### Screen Initialisation and Extension Detection

`zink_screen` initialisation (`zink_screen_create()`) follows a fixed sequence that mirrors what any Vulkan application must do:

1. Create `VkInstance` with the required platform WSI extensions (surface, display, etc.).
2. Enumerate physical devices via `vkEnumeratePhysicalDevices`.
3. Check extension support via `vkEnumerateDeviceExtensionProperties` and populate the `screen->info.have_*` boolean flags for every extension Zink knows about.
4. Query device features via `vkGetPhysicalDeviceFeatures2` (with chained structures for every relevant feature extension).
5. Create `VkDevice` enabling only the features and extensions actually present.
6. Create the graphics `VkQueue` and its associated mutex (`screen->queue_lock`).
7. Initialise the pipeline cache from disk (`disk_cache_init()`), keyed on the blake3 hash of driver build ID and `pipelineCacheUUID`.
8. Initialise format support tables by calling `vkGetPhysicalDeviceFormatProperties` in bulk.
9. Compute Gallium capability values from the Vulkan device limits and report them through `pipe_screen::get_param()`.

The extension-detection step is central to Zink's design: virtually every feature is guarded by an `if (screen->info.have_EXT_*)` check. This means a single Zink binary adapts its behaviour to the backend Vulkan driver at runtime, spanning the full range from a minimal Vulkan 1.0 device to a modern GPU with 30+ extensions.

### Command Buffer Architecture

Zink maintains three command buffers per `zink_batch_state`:

- **`cmdbuf`**: primary ordered command buffer into which all draw calls and render passes are recorded. Submitted once per batch to `vkQueueSubmit`.
- **`reordered_cmdbuf`**: receives operations that can be freely reordered with respect to draws — chiefly `vkCmdCopyImage`, `vkCmdCopyBuffer`, and `vkCmdBlitImage` calls that do not touch active render pass attachments. These are submitted *before* `cmdbuf` in the final `vkQueueSubmit` call.
- **`unsynchronized_cmdbuf`**: fully independent operations (e.g., buffer-to-buffer copies of staging data that is not yet in use by any pending GPU work). Submitted to an independent timeline semaphore chain.

The three-buffer split was introduced to solve a performance problem on tile-based mobile GPUs: when a copy operation interrupts a render pass (e.g., a `glCopyTexImage2D` mid-scene), naive implementations break the render pass, incurring a costly `VK_ATTACHMENT_LOAD_OP_LOAD` on resume. By routing reorderable operations into `reordered_cmdbuf` and scheduling them before the main render pass, Zink avoids unnecessary render-pass splits. Resources that are known to contain valid data (tracked per-resource) use `VK_ATTACHMENT_LOAD_OP_LOAD`; resources flagged as containing uninitialised data prefer `VK_ATTACHMENT_LOAD_OP_DONT_CARE` or `VK_ATTACHMENT_LOAD_OP_CLEAR`. [Source: supergoodcode.com Zink development blog](https://www.supergoodcode.com/new-news/)

---

## 3. OpenGL State to Vulkan Translation

The core challenge Zink faces is the impedance mismatch between OpenGL's implicit, state-machine model and Vulkan's explicit, pipeline-object model. The gap manifests differently for each subsystem.

### Immediate Mode (glBegin/glEnd)

GL's immediate mode has no Vulkan equivalent. Zink handles this through Mesa's existing `vbo` (vertex buffer object) module in the Gallium state tracker, which buffers all `glVertex*`, `glColor*`, `glTexCoord*`, and related calls into a temporary heap-allocated VBO. By the time a `glEnd()` reaches Zink, the data has been packaged into a `pipe_draw_info` with a vertex buffer, and Zink issues a normal `vkCmdDraw`.

### Draw Calls

The main draw entry point is `zink_draw_vbo()` in `src/gallium/drivers/zink/zink_draw.cpp`. It calls `update_gfx_pipeline()` to resolve the current pipeline state object, `barrier_draw_buffers()` to insert any needed Vulkan memory barriers for index and indirect buffers, and `zink_bind_vertex_buffers()` to call `vkCmdBindVertexBuffers`. The final translation is direct:

```c
/* zink_draw.cpp (simplified) */
if (dinfo->index_size)
    VKCTX(CmdDrawIndexed)(batch->state->cmdbuf,
                          draws[0].count, dinfo->instance_count,
                          draws[0].start, dinfo->index_bias,
                          dinfo->start_instance);
else
    VKCTX(CmdDraw)(batch->state->cmdbuf,
                   draws[0].count, dinfo->instance_count,
                   draws[0].start, dinfo->start_instance);
```

The `VKCTX` macro dispatches through the screen's function pointer table, which is populated at startup from `vkGetDeviceProcAddr`.

### Framebuffer Objects

GL Framebuffer Objects (FBOs) map onto Vulkan render passes and framebuffers. Zink maintains a `zink_render_pass` cache keyed on an attachment descriptor hash — the set of attachment VkFormats, load-ops (clear, load, don't-care), store-ops, and whether depth/stencil is read-only. When the FBO configuration changes, `zink_get_render_pass()` either returns a cached `VkRenderPass` or creates a new one via `vkCreateRenderPass2`.

For drivers that support `VK_KHR_dynamic_rendering` (checked at startup as `screen->info.have_KHR_dynamic_rendering`), Zink avoids creating `VkRenderPass` objects altogether and instead calls `vkCmdBeginRenderingKHR` with `VkRenderingAttachmentInfoKHR` structures populated from the active FBO attachments. This path reduces object overhead and simplifies the render-pass transition logic. [Source: PowerVR Zink blog](https://blog.imaginationtech.com/powervr-the-path-to-open-source-zink-and-opengl-es-support/)

### Textures and Samplers

Each Gallium `pipe_sampler_view` maps to a `zink_sampler_view` containing a `VkImageView`. The `create_surface()` function in `zink_surface.c` populates a `VkImageViewCreateInfo`, calling `vkviewtype_from_pipe()` to map Gallium texture targets (`PIPE_TEXTURE_2D`, `PIPE_TEXTURE_CUBE`, etc.) to `VkImageViewType` values.

GL sampler objects become `VkSampler` objects. Zink caches sampler objects because `vkCreateSampler` is not cheap. The cache key covers min/mag/mip filter, address modes, compare function, anisotropy, border color, and LOD clamp.

Border colors require special handling: Vulkan only supports a fixed set of border colors (transparent black, opaque black, opaque white), while GL allows arbitrary RGBA. When the Vulkan device exposes `VK_EXT_custom_border_color`, Zink uses it; otherwise it emulates non-standard border colors through shader code.

### Uniform Buffers

GL Uniform Buffer Objects (UBOs) map to `VkBuffer` allocations with `VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT`. The buffer is associated with a descriptor of type `VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER` and bound to the pipeline's descriptor set. Zink tracks which UBO slots are dirty between draws and updates descriptors only when they change.

### Transform Feedback

GL's `GL_ARB_transform_feedback3` (and core transform feedback in GL 4.0) maps to `VK_EXT_transform_feedback`, which Zink checks at startup (`screen->info.have_EXT_transform_feedback`). The extension provides `vkCmdBindTransformFeedbackBuffersEXT` and `vkCmdBeginTransformFeedbackEXT`, which Zink calls from `zink_set_stream_output_targets()` and the corresponding draw path.

### Fixed-Function Pipeline Emulation

OpenGL 1.x and 2.x include fixed-function lighting, texenv combiners, alpha testing, and fog — none of which exist in Vulkan. Zink emulates these through NIR shaders generated at draw time:

- **Fog**: a NIR pass injects fog distance calculation and blending into the fragment shader. The fog factor formula (linear, exponential, exponential-squared) and the fog colour are passed as push constants so they can change without recompiling SPIR-V.
- **Alpha test**: a NIR pass inserts a conditional `discard` instruction based on the active compare function (`GL_LESS`, `GL_GREATER`, etc.) and the reference value. Vulkan has no native alpha test; `VK_EXT_legacy_vertex_attributes` addresses vertex attribute legacy formats, not the fragment discard.
- **TexEnv combiners**: the `st_create_fixed_func_fragment_program()` path in the Mesa GL state tracker compiles GL 1.x `GL_MODULATE`, `GL_DECAL`, `GL_ADD`, and combiner equations into GLSL programs, which then feed into Mesa's GLSL→NIR frontend. Zink receives fully translated NIR and is unaware it originated from a fixed-function texenv state.

### Synchronisation and Barrier Insertion

Zink uses a **lazy barrier** model: rather than inserting `VkImageMemoryBarrier` and `VkMemoryBarrier` records speculatively, it defers them until the resource is actually needed. Each `zink_resource` carries:

- `access`: current `VkAccessFlags` on the resource.
- `access_stage`: current `VkPipelineStageFlags`.
- `layout`: current `VkImageLayout` for images.
- `unordered_access`: access flags accumulated from the `reordered_cmdbuf` path.

When a draw call or copy needs a resource in a layout or access mode that differs from its current state, `zink_update_barriers()` generates the needed barrier record and queues it into the pre-draw barrier batch. Barriers are then emitted with a single `vkCmdPipelineBarrier2` call (using `VK_KHR_synchronization2`, available in Vulkan 1.3) before the draw command.

For `dma-buf` imported images (used in X11 implicit sync via `EGL_EXT_image_dma_buf_import`), Zink must additionally perform a queue family ownership transfer from `VK_QUEUE_FAMILY_EXTERNAL` to the 3D queue family. This requires a pair of barriers: one in the `reordered_cmdbuf` with the `RELEASE` operation and another in `cmdbuf` with the `ACQUIRE` operation. A 2025 fix corrected a bug where the detect-and-barrier logic for dma-buf only fired on first use, causing unsynchronised access and flickering on subsequent frames. [Source: Collabora: Fixing Zink synchronisation (2025)](https://www.collabora.com/news-and-blog/blog/2025/10/27/from-browsers-to-better-drivers-fixing-synchronization-in-zink/)

---

## 4. Shader Translation: GLSL → NIR → SPIR-V

Zink does not parse GLSL or SPIR-V itself. Shader compilation proceeds in stages, with Mesa's common infrastructure handling each transition:

```
GLSL source
    │  glsl_to_nir() — Mesa GLSL frontend
    ▼
NIR (architecture-independent IR)
    │  Zink-specific NIR lowering passes
    ▼
NIR (Zink-lowered form)
    │  nir_to_spirv() — Mesa's NIR→SPIR-V emitter
    ▼
SPIR-V binary (in-memory)
    │  vkCreateShaderModule()
    ▼
VkShaderModule
    │  vkCreateGraphicsPipelines()
    ▼
VkPipeline
```

The key function is `nir_to_spirv()` in `src/compiler/nir/nir_to_spirv/nir_to_spirv.c`. This is Mesa's own SPIR-V emitter (distinct from `spirv-tools`): it walks the NIR instruction stream and emits SPIR-V words using a `spirv_builder` structure. The emitter is shared with other Mesa components (e.g., it is also used by `iris` for Vulkan-backed compute and by `dozen` for D3D12). [Source: Mesa SPIR-V debugging docs](https://docs.mesa3d.org/spirv/index.html)

### Zink-Specific NIR Passes

Before calling `nir_to_spirv()`, Zink runs several NIR lowering passes in `zink_compiler.c`:

- **`zink_nir_lower_b2b`**: Converts NIR bool-to-bool casts that the SPIR-V emitter cannot represent directly.
- **`zink_nir_lower_texcoord_replace`**: Replaces `gl_TexCoord[i]` inputs with interpolated varyings when the GL state tracker requests point sprite texture coordinate generation.
- **`lower_drawid`**: Converts `nir_intrinsic_load_draw_id` to a push constant access, since Vulkan exposes draw ID via `VK_KHR_draw_indirect_count` or push constants rather than as a built-in in the same form.
- **`create_gfx_pushconst`**: Defines the push constant layout used for items like draw ID, base vertex, and base instance.
- **Quads emulation**: a geometry shader generated by `zink_create_quads_emulation_gs()` tessellates `GL_QUADS` primitives into triangles (see Section 8).

### Shader Variants (`zink_shader_key`)

Zink may need to compile multiple SPIR-V variants of the same NIR shader. Variant keys are captured in per-stage structs (`zink_fs_key`, `zink_vs_key`, `zink_gs_key`, `zink_tcs_key`). A fragment shader variant key includes, for example, whether alpha-to-coverage is active, whether point sprites are enabled, and the number of colour outputs. The shader key is hashed into `zink_gfx_pipeline_state::key[]` and looked up in a per-program hash table.

### The `zink_shader` Object

```c
struct zink_shader {
    struct util_live_shader  base;    /* reference counting */
    uint32_t                 hash;
    struct blob              blob;    /* serialised NIR */
    struct shader_info       info;    /* nir_shader_info copy */
    nir_shader              *nir;
    uint16_t                 xfb_stride[PIPE_MAX_SO_BUFFERS];
    uint32_t                 ubos_used;
    uint32_t                 ssbos_used;
    bool                     bindless;
    struct spirv_shader     *spirv;   /* compiled SPIR-V, or NULL */
    struct set              *programs; /* back-pointer to zink_gfx_program */
};
```

The `spirv` field is populated lazily: SPIR-V compilation is deferred until the shader is first used in a pipeline.

### Shader Objects: `VK_EXT_shader_object`

When the Vulkan device supports `VK_EXT_shader_object` (checked via `zink_can_use_shader_objects()` in `zink_program.h`), Zink can switch from `VkPipeline`-based binding to shader-object binding. With shader objects, each shader stage is independently compiled into a `VkShaderEXT` object via `vkCreateShadersEXT` and bound at draw time via `vkCmdBindShadersEXT`. This eliminates the final pipeline-link step entirely — state changes no longer require creating a new `VkPipeline`. The shader-object path supersedes the GPL path on drivers that expose it. RADV enabled `VK_EXT_shader_object` support in Mesa 24.x, making the shader-object path available on AMD RDNA2+ hardware.

### Push Constants for Built-ins

Several GL built-in values that GLSL exposes as shader inputs have no direct Vulkan equivalent and are passed via push constants:

| GL built-in | Push constant slot |
|-------------|-------------------|
| `gl_DrawID` | `GFX_PUSHCONST_DRAW_ID` |
| `gl_BaseVertex` | `GFX_PUSHCONST_BASE_VERTEX` |
| `gl_BaseInstance` | `GFX_PUSHCONST_BASE_INSTANCE` |
| `gl_PointSize` (default) | inline constant |

The push constant layout is established by `create_gfx_pushconst()` in `zink_compiler.c` and recorded in the `VkPipelineLayout`. The `lower_drawid()` NIR pass replaces `nir_intrinsic_load_draw_id` with a push-constant load at the appropriate offset.

---

## 5. Descriptor Management

OpenGL's resource binding model is implicit: an application calls `glBindTexture(GL_TEXTURE_2D, tex)`, and the texture becomes available to all subsequent draw calls on unit 0. Vulkan, by contrast, requires every resource used by a pipeline to be described in a `VkDescriptorSet` that is explicitly bound before each draw. Bridging this gap is one of Zink's most performance-sensitive subsystems.

### Descriptor Types

Zink defines four fundamental descriptor categories:

| Enum | Vulkan type | GL source |
|------|-------------|-----------|
| `ZINK_DESCRIPTOR_TYPE_UBO` | `VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER` | `glBindBufferBase(GL_UNIFORM_BUFFER, …)` |
| `ZINK_DESCRIPTOR_TYPE_SAMPLER_VIEW` | `VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER` | `glBindTexture`, `glBindSampler` |
| `ZINK_DESCRIPTOR_TYPE_SSBO` | `VK_DESCRIPTOR_TYPE_STORAGE_BUFFER` | `glBindBufferBase(GL_SHADER_STORAGE_BUFFER, …)` |
| `ZINK_DESCRIPTOR_TYPE_IMAGE` | `VK_DESCRIPTOR_TYPE_STORAGE_IMAGE` | `glBindImageTexture` |

### Descriptor Modes

The `ZINK_DESCRIPTORS` environment variable selects one of three descriptor management strategies:

**`lazy` mode** — the simplest path. On every draw call that has changed any resource binding, Zink allocates a new `VkDescriptorSet` from the per-batch `VkDescriptorPool`, writes all descriptors into it with `vkUpdateDescriptorSets`, and calls `vkCmdBindDescriptorSets`. Sets are never reused across frames; they are freed when the batch completes. This is correct and easy to implement but wastes CPU time on allocation and descriptor writes.

**`db` mode (descriptor buffers)** — the high-performance path, enabled when the device supports `VK_EXT_descriptor_buffer`. Instead of `VkDescriptorPool` and `vkUpdateDescriptorSets`, Zink maintains a large GPU-visible buffer and writes descriptor data directly into it using `vkGetDescriptorEXT`. Binding is done via `vkCmdBindDescriptorBuffersEXT` and `vkCmdSetDescriptorBufferOffsetsEXT`. This eliminates one level of indirection from the descriptor update path, reduces synchronisation between descriptor set allocation and binding, and in practice cuts peak VRAM usage by over 10% compared to lazy mode. [Source: Phoronix — Mesa Zink DB Descriptor Mode](https://www.phoronix.com/news/Mesa-Zink-DB-Descriptor-Mode)

**`auto` mode** — the default. Zink queries for `VK_EXT_descriptor_buffer` support at startup; if present, it uses `db` mode; otherwise it falls back to `lazy`.

### The Update Hot Path

`zink_descriptors_update()` in `src/gallium/drivers/zink/zink_descriptors.c` is the per-draw hot path. It compares the current binding state against the descriptors already committed to the active set, identifies which descriptor types have changed, and applies only the minimum necessary writes. In `db` mode, the function calls the `update_descriptor_state_ubo_db()` family of helpers that write directly into the descriptor buffer at the precomputed offset for the current pipeline layout.

### Bindless Descriptors

When `screen->info.have_EXT_descriptor_indexing` is true, Zink can expose `GL_ARB_bindless_texture` — GL's own extension for GPU-resident texture handles that bypass the per-draw descriptor update path entirely. Bindless textures are allocated as `VkImageView` handles stored in a large descriptor array. The `zink_shader::bindless` flag records whether a shader uses bindless descriptors, which affects pipeline layout creation (a `VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT` descriptor array is added to the layout).

### Descriptor Set Layout Strategy

Each Zink pipeline layout uses up to four descriptor sets, one per descriptor category (UBO, sampler, SSBO, image). The `ZINK_DEBUG=compact` flag restricts to four sets maximum, matching the Vulkan 1.0 guaranteed minimum of four sets. On hardware that supports more sets (the spec mandates at least four; most desktop hardware supports 8–32), Zink uses additional sets for compute-specific bindings.

Pipeline layout creation is handled by `zink_pipeline_layout_create()` in `zink_program.c`. The layout is cached in a hash table keyed on the set of active descriptor types and binding counts; the same layout is shared across programs with compatible bindings.

---

## 6. Pipeline State Object Caching

Vulkan requires all non-dynamic rendering state (blend mode, rasterization, depth/stencil test configuration, vertex attribute formats, primitive topology, etc.) to be baked into a `VkPipeline` at creation time. GL allows these to change at any time. This mismatch historically caused Zink to stall mid-frame to compile new pipelines when the GL application changed rasterizer state.

### Dynamic State Extensions

Zink's primary mitigation is aggressive use of dynamic state extensions:

- **`VK_EXT_extended_dynamic_state`** (Vulkan 1.3 core): viewport, scissor, cull mode, front face, primitive topology, line width, depth test/write, stencil ops.
- **`VK_EXT_extended_dynamic_state2`**: patch control points, rasterizer discard, primitive restart.
- **`VK_EXT_extended_dynamic_state3`**: polygon mode, blend factors, colour write masks, depth clamp, depth clip, alpha-to-coverage — nearly the entire rasterizer and blend state.

When all three extensions are available (as is the case on RADV and ANV on modern hardware), almost every GL-mutable state can be set dynamically with `vkCmdSet*` commands, and the `VkPipeline` object reduces to a small skeleton containing only the shader stages and vertex input layout.

[Source: Mesa GitLab issue — handle more pipeline states dynamically](https://gitlab.freedesktop.org/mesa/mesa/-/issues/3359)

### `zink_gfx_pipeline_state`

The non-dynamic remainder is captured in `zink_gfx_pipeline_state`:

```c
struct zink_gfx_pipeline_state {
    uint32_t                           rast_samples:6;
    uint32_t                           rp_state:16;     /* render pass hash */
    VkSampleMask                       sample_mask;
    uint32_t                           blend_id;
    uint32_t                           hash;            /* pipeline cache key */
    struct zink_pipeline_dynamic_state1 dyn_state1;
    struct zink_pipeline_dynamic_state2 dyn_state2;
    VkShaderModule   modules[MESA_SHADER_MESH_STAGES - 1];
    struct zink_vertex_elements_hw_state *element_state;
    struct zink_shader_key              key[5];
    struct zink_blend_state            *blend_state;
    VkFormat         rendering_formats[PIPE_MAX_COLOR_BUFS];
    VkPipeline                         pipeline;
};
```

The `hash` field is a CRC of the non-dynamic fields. `zink_get_gfx_pipeline()` looks up the hash in a per-program hash table and calls `vkCreateGraphicsPipelines` on a miss.

### Graphics Pipeline Library (GPL)

When the device supports `VK_EXT_graphics_pipeline_library`, Zink splits each pipeline into four independently compiled parts:

1. **Vertex input** — vertex buffer bindings and attribute formats.
2. **Pre-rasterization shaders** — vertex, tessellation control/evaluation, geometry shaders plus their pipeline layout.
3. **Fragment shader** — the fragment stage.
4. **Fragment output** — blend state, render-target formats, sample count.

Parts 1 and 4 are cheap to compile and change rarely. Parts 2 and 3 can be compiled asynchronously in a background thread (`zink_gfx_program_compile_queue()`). A "fast link" final step calls `vkCreateGraphicsPipelines` with `VK_PIPELINE_CREATE_LINK_TIME_OPTIMIZATION_BIT_EXT` omitted, producing a linked pipeline in milliseconds. This dramatically reduces first-draw stalls. `zink_can_use_pipeline_libs()` enables this path when GPL is supported.

### On-Disk Pipeline Cache

`zink_screen` creates a `VkPipelineCache` at startup via `vkCreatePipelineCache`. Zink serialises this cache to disk using Mesa's `disk_cache` infrastructure. The cache key incorporates the driver's build ID, the `VkPhysicalDeviceProperties::pipelineCacheUUID`, and active debug flags (computed with blake3 hashing). On subsequent runs, pipeline compilation hits are served from disk, eliminating stalls entirely for warm workloads.

---

## 7. Memory Management and Resources

### `zink_bo` and Memory Heaps

Zink's memory allocator operates through `zink_bo`, a thin wrapper around a `VkDeviceMemory` allocation that also records the read/write usage batch IDs for lifetime tracking:

```c
struct zink_bo {
    struct pb_buffer   base;       /* Gallium buffer base */
    VkDeviceMemory     mem;
    uint64_t           offset;     /* within the VkDeviceMemory */
    uint32_t           unique_id;
    const char        *name;
    struct zink_bo_usage reads;
    struct zink_bo_usage writes;
};
```

Zink recognises three heap tiers, exposed upward to Gallium:

| `zink_heap` enum | Vulkan property flags | GL usage |
|------------------|----------------------|----------|
| `ZINK_HEAP_DEVICE_LOCAL` | `DEVICE_LOCAL` | GPU-resident textures, render targets |
| `ZINK_HEAP_DEVICE_LOCAL_VISIBLE` | `DEVICE_LOCAL | HOST_VISIBLE` | BAR / resizable-BAR uploads |
| `ZINK_HEAP_HOST_VISIBLE_COHERENT` | `HOST_VISIBLE | HOST_COHERENT` | Staging buffers, CPU readbacks |

Small buffer allocations (below ~256 KB) use a slab allocator to amortise `vkAllocateMemory` overhead, since the Vulkan spec permits a minimum of only 4096 device allocations.

Video memory capacity is reported to GL via `pipe_screen::get_param(PIPE_CAP_VIDEO_MEMORY)` by summing all `VkMemoryHeap` entries with the `VK_MEMORY_HEAP_DEVICE_LOCAL_BIT` flag from `vkGetPhysicalDeviceMemoryProperties`.

### Buffer Mapping

`glMapBuffer(GL_ARRAY_BUFFER, GL_WRITE_ONLY)` initiates one of two paths:

- If the buffer's backing `zink_bo` resides in `ZINK_HEAP_HOST_VISIBLE_COHERENT` or `ZINK_HEAP_DEVICE_LOCAL_VISIBLE`, Zink calls `vkMapMemory` directly and returns a CPU pointer to the application. This is the zero-copy path.
- If the buffer is `DEVICE_LOCAL` only (no host visibility), Zink allocates a temporary staging buffer in host-visible memory, returns a pointer to it, and on `glUnmapBuffer` copies the staged data to the device-local buffer using `vkCmdCopyBuffer` in a batch submission.

### Resource Copies and Blits

`zink_resource_copy_region()` maps to `vkCmdCopyBuffer`, `vkCmdCopyImage`, or `vkCmdCopyBufferToImage`/`vkCmdCopyImageToBuffer` depending on resource types. Before each copy, `zink_update_barriers()` processes the deferred barrier list, inserting `VkMemoryBarrier2` or `VkImageMemoryBarrier2` records with appropriate `srcStageMask`/`dstStageMask` flags.

`zink_blit()` handles `glBlitFramebuffer` and format-converting copies. It maps to `vkCmdBlitImage` when the source and destination formats are compatible, or falls back to a meta blit operation using a fullscreen triangle pipeline.

### Format Support

At startup, `zink_screen` calls `vkGetPhysicalDeviceFormatProperties` for every GL-relevant format and caches the results. The capability bits (`linearTilingFeatures`, `optimalTilingFeatures`, `bufferFeatures`) are translated into Gallium format capability masks and reported through `pipe_screen::is_format_supported()`. Formats with missing `VK_FORMAT_FEATURE_SAMPLED_IMAGE_FILTER_LINEAR_BIT` are reported as not supporting linear filtering; formats without `VK_FORMAT_FEATURE_COLOR_ATTACHMENT_BIT` cannot be render targets.

The helper `have_fp32_filter_linear()` in `zink_screen.c` exemplifies this pattern:

```c
/* zink_screen.c — check linear filtering for fp32 formats */
static bool
have_fp32_filter_linear(struct zink_screen *screen)
{
    const VkFormat fp32_formats[] = {
        VK_FORMAT_R32_SFLOAT, VK_FORMAT_R32G32_SFLOAT,
        VK_FORMAT_R32G32B32A32_SFLOAT, /* ... */
    };
    for (unsigned i = 0; i < ARRAY_SIZE(fp32_formats); i++) {
        VkFormatProperties props;
        VKSCR(GetPhysicalDeviceFormatProperties)(screen->pdev,
                                                  fp32_formats[i], &props);
        if (!(props.optimalTilingFeatures &
              VK_FORMAT_FEATURE_SAMPLED_IMAGE_FILTER_LINEAR_BIT))
            return false;
    }
    return true;
}
```

[Source: FireBurn/mesa zink_screen.c](https://github.com/FireBurn/mesa/blob/main/src/gallium/drivers/zink/zink_screen.c)

### Sparse Textures and Buffers

`GL_ARB_sparse_texture` (OpenGL 4.6 core) allows textures whose pages are only partially resident in GPU memory — essential for virtual texturing in large-world rendering engines. Zink maps this to Vulkan's sparse binding feature:

- Sparse image creation: `VkImageCreateInfo::flags |= VK_IMAGE_CREATE_SPARSE_RESIDENCY_BIT`.
- Page commitment: `vkQueueBindSparse` with `VkSparseImageMemoryBind` records maps individual mip-level tiles to physical memory pages.
- `GL_ARB_sparse_buffer` (added in Mesa 21.2): uses `VK_BUFFER_CREATE_SPARSE_RESIDENCY_BIT` for sparse buffers, also bound via `vkQueueBindSparse`.

Mesa 24.1 extended sparse support by using `VK_KHR_buffer_device_address`-backed sparse buffers, enabling sparse residency for use with SSBO data. The `PIPE_CAP_SPARSE_TEXTURE_FULL_ARRAY_CUBE_LEVELS` capability is reported based on whether the device exposes `VkPhysicalDeviceSparseProperties::residencyAlignedMipSize`.

---

## 8. OpenGL Compatibility Profile

The most ambitious aspect of Zink is its support for the OpenGL 4.6 **compatibility profile** — the superset of GL that includes all deprecated and removed features from GL 1.x through 3.x. The compatibility profile is required by a large class of professional and scientific applications:

- **CAD and engineering**: Blender's legacy viewport (pre-4.x cycles), FreeCAD, AutoCAD running under Wine.
- **Scientific visualisation**: ParaView, VTK, VisIt — applications that have OpenGL 2.x-era rendering code maintained across decades.
- **Game emulators and legacy games**: DXVK fallback paths, older OpenGL game engines that use `GL_QUADS`, display lists, or the GL 1.x matrix stack.

### GL_QUADS and Primitive Emulation

Vulkan has no `GL_QUADS` primitive. Zink emulates it with a geometry shader generated by `zink_create_quads_emulation_gs()` in `zink_compiler.c`. The GS receives adjacency-stripped quad input and emits two triangles per quad. When `GL_QUADS` is active, this GS is silently inserted between the application's vertex shader and fragment shader.

Similarly, `GL_QUAD_STRIP` and `GL_POLYGON` are tessellated to triangle strips and triangle fans by the vertex buffer object (VBO) module before reaching Zink.

### Display Lists

GL display lists (`glNewList`/`glCallList`) are a server-side command list mechanism. Mesa's Gallium state tracker implements display list compilation entirely on the CPU, executing them as a sequence of normal `pipe_context` calls. Zink sees only the resulting draw calls and state changes — it has no display-list-aware code itself. The performance implication is that display list "compilation" in Zink/Gallium merely records CPU-side state, not GPU command buffers; `glCallList` replays that CPU state. This is correct but not as efficient as the original hardware-accelerated display list concept.

### Accumulation Buffer Emulation

The accumulation buffer (`GL_ACCUM_BUFFER_BIT`, `glAccum()`) was removed from the GL core profile but is required by the compat profile. Mesa emulates it with a software path: `glAccum(GL_ACCUM, factor)` performs a read-back of the colour buffer, a scalar multiply, and a write to a per-context accumulation texture. Zink propagates this emulation without modification.

### Fixed-Function Lighting

The GL 1.x/2.x lighting model (up to eight `GL_LIGHT` sources with ambient, diffuse, specular, attenuation, and spotlight parameters) is compiled by the Mesa state tracker into a fragment shader using NIR. The same `st_create_fixed_func_fragment_program()` path that generates shaders for radeonsi and iris generates them for Zink. Zink receives them as NIR and passes them through its normal NIR→SPIR-V pipeline.

### ARB_separate_shader_objects

`GL_ARB_separate_shader_objects` (SSO) enables GL programs to mix-and-match vertex, geometry, and fragment shader stages from separate program objects via program pipelines (`glUseProgramStages`). Zink handles SSO through `zink_program_compile_separate_shader()`, which compiles each separable program stage independently and links them only at draw time when the full pipeline configuration is known.

### Conformance Status

Zink passes the OpenGL 4.6 core profile Khronos Conformance Test Suite (CTS) on RADV and ANV. Compatibility profile conformance is a work in progress; the major features (texenv, fog, alpha test, quads, display lists via CPU emulation, fixed-function lighting) are functional, and `MESA_GL_VERSION_OVERRIDE=4.6COMPAT` can be used to advertise the compat profile to applications for testing.

---

## 9. Zink as a Backend for Other Drivers

### NVK (NVIDIA Turing+)

NVK is Mesa's open-source Vulkan driver for NVIDIA Turing (RTX 20xx, GTX 16xx) and later GPUs (see Chapter 20). It implements only Vulkan — there is no separate NVK OpenGL driver. Starting with Mesa 25.1, the Mesa loader automatically routes OpenGL requests on NVK-capable hardware to Zink:

```bash
# Mesa 25.1+ automatically selects Zink+NVK for Turing+ GPUs.
# To test manually with an explicit ICD:
MESA_LOADER_DRIVER_OVERRIDE=zink \
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nouveau_icd.x86_64.json \
glxinfo | grep "OpenGL version"
```

This transition was motivated by the poor state of the legacy Nouveau GL driver: "At this point, we're fairly confident that Zink+NVK will be an improvement over Nouveau GL across the board." (Faith Ekstrand, Collabora)

### v3dv (Raspberry Pi 5)

The Raspberry Pi 5 uses Broadcom's V3D engine, supported by the `v3dv` Vulkan driver. `v3dv` does not include an OpenGL implementation. Zink over `v3dv` provides OpenGL ES 3.1 and, where `v3dv` extensions permit, higher GL versions, enabling the large body of existing Pi software that requires OpenGL.

### Turnip (Qualcomm Adreno)

Turnip is Mesa's open-source Vulkan driver for Qualcomm Adreno GPUs, used widely in Android devices. Zink over Turnip provides OpenGL ES on Adreno, complementing the existing `freedreno` Gallium driver (which handles GL directly). Both paths coexist; the choice depends on which driver is more mature for a given feature set.

### PowerVR

Imagination Technologies' open-source PowerVR Vulkan driver, merged upstream in Mesa 26.1, uses Zink as its only OpenGL implementation. The PowerVR path required implementing `VK_KHR_dynamic_rendering` and `VK_EXT_image_drm_format_modifier` in the PowerVR Vulkan driver before Zink support was complete. [Source: PowerVR Zink blog](https://blog.imaginationtech.com/powervr-the-path-to-open-source-zink-and-opengl-es-support/)

### Kopper: WSI Integration

The original Zink implementation used GBM and EGL for window system integration, which limited it to scenarios where GBM surfaces were available. **Kopper** (`src/gallium/winsys/kopper/`) is a WSI abstraction layer that lets Zink use any platform's Vulkan WSI (Window System Integration) for display. As described by Christian F.K. Schaller: "Kopper is the layer that allows you to translate OpenGL and GLX window handling to Vulkan WSI handling." [Source: GNOME blog on Kopper](https://blogs.gnome.org/uraeus/2022/04/07/why-is-kopper-and-zink-important-aka-the-future-of-opengl/)

Kopper's acquire/present flow:

1. On `eglSwapBuffers()` or `glXSwapBuffers()`, Kopper calls `vkAcquireNextImageKHR()` with a per-frame semaphore.
2. Zink renders the frame into the acquired swapchain image.
3. Kopper calls `vkQueuePresentKHR()` with the wait semaphore on completion.

Present modes are selected to match the GL swap interval:
- `VK_PRESENT_MODE_FIFO_KHR`: V-sync enabled (swap interval 1).
- `VK_PRESENT_MODE_MAILBOX_KHR`: triple-buffered, no tearing.
- `VK_PRESENT_MODE_IMMEDIATE_KHR`: no V-sync (swap interval 0).

The `KOPPER_DISPLAY_SYNC` environment variable overrides present mode selection.

Kopper also manages Wayland-specific threading requirements: Wayland requires all protocol messages to be sent synchronously within `eglSwapBuffers`. A 2025 synchronisation fix ensured Zink does not queue presents asynchronously on Wayland, avoiding protocol violations. [Source: Collabora sync blog post](https://www.collabora.com/news-and-blog/blog/2025/10/27/from-browsers-to-better-drivers-fixing-synchronization-in-zink/)

Additionally, Kopper caches `VkSurface` objects keyed on the native surface handle (`wl_surface`, `xcb_window_t`, etc.). A 2025 bug fix corrected a case where Mesa's proxy-wrapper `wl_surface` was being cached instead of the original, causing duplicate surfaces on re-initialisation.

---

## 10. Performance and Debugging

### Performance Characteristics

Zink's overhead relative to a native Gallium GL driver (radeonsi, iris) depends heavily on the descriptor mode, the available dynamic state extensions, and the application's draw call and state-change patterns. Rough empirical benchmarks on RADV and ANV show:

- **Best case** (descriptor-buffer mode + extended dynamic state 3 + GPL fast link): 0–5% overhead for games with consistent pipeline state and high vertex throughput.
- **Typical case** (descriptor-buffer mode, no GPL): 5–15% overhead for mixed workloads with frequent state changes.
- **Worst case** (lazy descriptor mode, no dynamic state extensions, frequent pipeline compiles): 20–40% overhead with observable mid-frame stutters.

The recommended fast path — `ZINK_DESCRIPTORS=db` with `VK_EXT_descriptor_buffer`, `VK_EXT_graphics_pipeline_library`, and `VK_EXT_extended_dynamic_state3` — is available on RADV (RDNA2+), ANV (Xe+ Intel), and NVK (Turing+) as of Mesa 25.x.

### Bottlenecks

The primary CPU-side bottlenecks, in order of typical impact:

1. **Descriptor updates** — mitigated by `db` mode.
2. **Pipeline compilation latency** — mitigated by GPL pre-compilation and disk cache warm-up.
3. **Render pass transitions** — image layout changes require `VkImageMemoryBarrier` records; `VK_KHR_dynamic_rendering` removes full `VkRenderPass` compilation overhead but not individual layout transitions.
4. **Buffer suballocation** — slab allocators address this for small buffers.

### `ZINK_DEBUG` Flags

The `ZINK_DEBUG` environment variable accepts a comma-separated list:

| Flag | Effect |
|------|--------|
| `nir` | Print NIR of all shaders to stderr before SPIR-V emission |
| `spirv` | Write binary SPIR-V to files in the current directory (`zink_*.spv`) |
| `tgsi` | Print TGSI form of TGSI shaders (legacy path) |
| `validation` | Enable Vulkan validation layers (equivalent to setting `VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation`) |
| `sync` | Insert full pipeline barriers between every operation (aids race-condition debugging) |
| `rp` | Verbose render pass logging |
| `norp` | Disable render pass optimisations (useful for isolating render pass bugs) |
| `cache` | Log pipeline cache hits and misses |
| `compact` | Restrict to a maximum of 4 descriptor sets |
| `gpl` | Force Graphics Pipeline Library usage even on drivers that would disable it |
| `mem` | Enable memory allocation accounting; print stats on context destruction |

```bash
# Dump NIR and SPIR-V for every shader compiled in a session:
ZINK_DEBUG=nir,spirv glxgears

# Force the lazy descriptor mode and log cache misses:
ZINK_DESCRIPTORS=lazy ZINK_DEBUG=cache glxgears

# Run with Vulkan validation layers enabled:
ZINK_DEBUG=validation glxgears
```

### CI and Testing

Zink's primary CI targets are RADV (on AMD RX 5700/6800 series) and ANV (on Intel Arc) with `ZINK_DESCRIPTORS=db`. The Mesa CI runs `deqp-runner` with the full OpenGL ES CTS and portions of the OpenGL 4.6 CTS via Zink. Shader-db runs measure compiled SPIR-V instruction counts as a proxy for compiler quality regressions.

The `MESA_GL_VERSION_OVERRIDE` environment variable can be used to advertise a specific GL version to an application, useful for testing compatibility:

```bash
# Advertise OpenGL 4.6 compatibility profile via Zink:
MESA_LOADER_DRIVER_OVERRIDE=zink \
MESA_GL_VERSION_OVERRIDE=4.6COMPAT \
glxinfo | grep "OpenGL version"
```

### Render Pass Optimisation

One of the more nuanced performance concerns is render pass fragmentation — the situation where a single "frame" from GL's perspective is split into many small Vulkan render passes because of interleaved resource reads and writes. Each render pass break on a tile-based GPU (ARM Mali, Qualcomm Adreno, PowerVR) is especially costly: the tile buffers must be flushed to DRAM at `VK_ATTACHMENT_STORE_OP_STORE` and reloaded at `VK_ATTACHMENT_LOAD_OP_LOAD` on the next pass, consuming memory bandwidth.

Zink addresses this through the three-command-buffer architecture described in Section 2: operations that can be safely reordered (blits, buffer copies) are promoted to `reordered_cmdbuf` and submitted before the main render pass, reducing interruptions. Additionally, `ZINK_DEBUG=rp` enables verbose render pass logging that prints each pass's attachments, load-ops, store-ops, and the GL call that triggered the break — invaluable for diagnosing fragmentation in a specific application.

The `ZINK_DEBUG=norp` flag disables all render-pass optimisations, forcing each draw call into its own pass. This is a debugging aid only; performance degrades severely with `norp` enabled.

### X11 Binary Semaphore Interop

When applications use `GL_EXT_semaphore` (for Vulkan–GL interop, as browser engines do when sharing textures between Vulkan compositing and GL rendering), Zink must correctly handle imported binary semaphores. A 2025 fix addressed a violation where Zink was discarding an imported `VkSemaphore` after its first signal, breaking the persistent-signal semantics required by the GL extension spec. The fix treats imported semaphores as persistent objects that can be signalled and waited on repeatedly without deletion. [Source: Collabora: Fixing Zink synchronisation (2025)](https://www.collabora.com/news-and-blog/blog/2025/10/27/from-browsers-to-better-drivers-fixing-synchronization-in-zink/)

### GL Threading

Starting with Mesa 25.1, Zink unconditionally enables GL threading (`mesa_glthread`). GL threading moves the Mesa GL API call processing to a separate CPU thread, issuing commands to the driver thread via a ring buffer. This decouples application logic from GPU submission, reducing the impact of CPU-side translation overhead on frame timing. On CPU-bound workloads, GL threading alone can recover several percentage points of the performance gap between Zink and a native GL driver.

---

## 11. Integrations

**Chapter 13 (Gallium3D)** — Zink is a Gallium driver: it implements `pipe_screen`, `pipe_context`, and the resource/surface interfaces. All Gallium3D concepts (pipe caps, bind flags, resource creation, draw info) flow through Zink.

**Chapter 14 (NIR)** — Every shader that Zink compiles passes through NIR. Zink runs its own NIR lowering passes (texcoord replacement, bool-to-bool lowering, draw-ID push constants, quads emulation GS insertion) before handing NIR to `nir_to_spirv()`.

**Chapter 16 (Mesa Vulkan Common)** — Zink does not use the Mesa Vulkan common infrastructure (which targets Mesa Vulkan *drivers*, not Vulkan *consumers*), but it does use the common Vulkan WSI support through Kopper and the common format-property query utilities.

**Chapter 18 (RADV)** — RADV is the primary Zink test and development target. RADV's broad extension coverage (descriptor buffers, GPL, extended dynamic state 3, dynamic rendering) enables Zink's full fast path, and RADV+Zink is the combination run in Mesa's CI.

**Chapter 20 (NVK)** — Since Mesa 25.1, NVK is Zink's production backend for NVIDIA hardware. The Zink+NVK stack replaces the legacy Nouveau OpenGL driver for Turing+ GPUs.

**Chapter 24 (Vulkan)** — Zink is arguably the largest consumer of optional Vulkan extensions in the Mesa ecosystem: `VK_EXT_descriptor_buffer`, `VK_EXT_extended_dynamic_state{1,2,3}`, `VK_EXT_graphics_pipeline_library`, `VK_KHR_dynamic_rendering`, `VK_EXT_transform_feedback`, `VK_EXT_custom_border_color`, and many more are all exploited by Zink to bridge the GL/Vulkan gap.

**Chapter 61 (SPIR-V Tooling)** — Zink emits SPIR-V through Mesa's `nir_to_spirv()` emitter, not through `spirv-tools`. The resulting SPIR-V is consumed directly by the backend Vulkan ICD's shader compiler (e.g., RADV's ACO). `ZINK_DEBUG=spirv` dumps the emitted binaries for inspection with `spirv-dis` or `spirv-val`.

**Chapter 116 (GPU Memory / TTM)** — Zink's `zink_bo` memory management parallels the BO allocation patterns used by Gallium drivers talking to TTM-based DRM kernel drivers, but routes through `vkAllocateMemory` rather than kernel GEM ioctls. The slab allocator and heap-tier concepts are directly analogous.

---

*Sources consulted for this chapter:*
- [Mesa Zink driver documentation](https://docs.mesa3d.org/drivers/zink.html)
- [DeepWiki: Zink OpenGL to Vulkan Translation Layer (bminor/mesa-mesa)](https://deepwiki.com/bminor/mesa-mesa/3.2-zink-opengl-to-vulkan-translation-layer)
- [Collabora: Introducing Zink (2018)](https://www.collabora.com/news-and-blog/blog/2018/10/31/introducing-zink-opengl-implementation-vulkan/)
- [Collabora: From browsers to better drivers — fixing Zink synchronisation (2025)](https://www.collabora.com/news-and-blog/blog/2025/10/27/from-browsers-to-better-drivers-fixing-synchronization-in-zink/)
- [GamingOnLinux: Mesa 25.1 Zink+NVK replacing Nouveau GL](https://www.gamingonlinux.com/2025/03/mesa-25-1-will-default-to-zink-nvk-instead-of-the-old-nouveau-opengl-driver-for-nvidia-on-linux/)
- [Phoronix: Mesa Zink DB descriptor mode](https://www.phoronix.com/news/Mesa-Zink-DB-Descriptor-Mode)
- [GNOME blog: Why Kopper and Zink are important](https://blogs.gnome.org/uraeus/2022/04/07/why-is-kopper-and-zink-important-aka-the-future-of-opengl/)
- [Imagination Technologies: PowerVR path to Zink](https://blog.imaginationtech.com/powervr-the-path-to-open-source-zink-and-opengl-es-support/)
- [Mesa GitLab: zink_types.h, zink_screen.c, zink_program.h (FireBurn/mesa mirror)](https://github.com/FireBurn/mesa/blob/main/src/gallium/drivers/zink/zink_screen.c)
- [Mesa GitLab issue: handle more pipeline states dynamically](https://gitlab.freedesktop.org/mesa/mesa/-/issues/3359)
