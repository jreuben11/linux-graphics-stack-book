# Chapter 173 — VK_EXT_shader_object: Pipeline-Free Shader Binding in Vulkan

This chapter targets three overlapping audiences:

- **Vulkan graphics application developers** — who want to eliminate pipeline compilation stutter in their applications
- **Mesa driver contributors** — implementing or maintaining `VK_EXT_shader_object` support in RADV, ANV, and NVK
- **Game engine developers** — evaluating a migration path away from the `VkPipeline` object model

Readers should be comfortable with core Vulkan concepts — command buffers, descriptor sets, render passes — before proceeding.

---

## Table of Contents

1. [Introduction: The Pipeline Compilation Problem](#1-introduction-the-pipeline-compilation-problem)
2. [VK_EXT_shader_object Overview](#2-vk_ext_shader_object-overview)
3. [Dynamic State: The Companion Feature](#3-dynamic-state-the-companion-feature)
4. [Interaction with VK_KHR_dynamic_rendering](#4-interaction-with-vk_khr_dynamic_rendering)
5. [Shader Compilation Workflow](#5-shader-compilation-workflow)
6. [Mesa Driver Implementations](#6-mesa-driver-implementations)
7. [Performance Characteristics](#7-performance-characteristics)
8. [Migration Guide: From VkPipeline to VkShaderEXT](#8-migration-guide-from-vkpipeline-to-vkshaderext)
9. [Integrations](#9-integrations)

---

## 1. Introduction: The Pipeline Compilation Problem

### 1.1 What VkPipeline Bundles

The `VkPipeline` object is one of Vulkan's most recognisable design choices. A `VkGraphicsPipeline` bakes together — at creation time — the compiled machine code for every shader stage, the rasterization state (fill mode, cull mode, polygon offset), the depth/stencil state, the colour-blend state, the vertex-input layout, the primitive topology, and the render-pass format. The rationale was sound: by handing the driver a complete description of what the draw call will do, the driver can perform full inter-stage optimisations and produce ISA code tuned to the exact fixed-function state the hardware expects. This was a direct response to the OpenGL driver model, where the driver had to defer much of this work to draw time because the application could change state at will.

However, the model's strength is also its weakness. State that will not change for the lifetime of a draw call *must still be re-declared* every time a new pipeline variant is needed. In a rendering engine with N material types, M lighting configurations, and K render-target formats, the combinatorial explosion of pipeline variants can reach the tens or hundreds of thousands. The pipeline creation cost — dominated by the final shader compilation step that converts SPIR-V or an intermediate IR into GPU machine code — is paid for each variant. On PC GPU drivers, this can take milliseconds to hundreds of milliseconds per pipeline. The consequence in shipped titles is *pipeline compilation stutter*: a visible frame-time spike the first time an unseen state combination appears, often at precisely the worst moment — a new area loads, a new particle effect fires, an enemy type appears for the first time.

### 1.2 Prior Mitigations

The Vulkan ecosystem developed several mitigations, none fully satisfactory.

**Pipeline caches** (`VkPipelineCache`) allow compiled pipeline objects to be serialised to disk and reloaded in a subsequent run. On a warm cache the compilation step is skipped. But the first run still stutters, the cache is driver-specific and not portable across driver versions, and the cache must be invalidated whenever the driver or GPU changes.

**Background compilation** (deferred pipeline creation, available via `VK_EXT_pipeline_creation_cache_control`) allows the application to submit pipeline creation work to a background thread while continuing to render with a placeholder. This trades the stutter for a potentially incorrect rendering until the pipeline is ready. Managing the lifecycle — detecting completion, swapping in the real pipeline — adds application complexity.

**Graphics pipeline libraries** (`VK_KHR_pipeline_library`, `VK_EXT_graphics_pipeline_library`) decompose the pipeline into four independently compiled parts: vertex input, pre-rasterization shaders, fragment shader, and fragment output. The first three can be compiled without knowing the render-target format; only the fast-link step combining them requires the complete state. This reduces the per-variant compilation burden and enables earlier compilation of partial pipelines. However, it does not eliminate the link step, and the link step still requires knowing the complete set of state at link time. The combination-space problem remains.

[Source — VK_EXT_graphics_pipeline_library proposal](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_graphics_pipeline_library.html)

### 1.3 The Root Cause

The pipeline mitigations share a fundamental limitation: they are refinements of the same pipeline paradigm. As the `VK_EXT_shader_object` proposal document states:

> "Pipelines have provided some of their desired benefits for some implementations, but for others they have largely just shifted draw time overhead to pipeline bind time."

Applications adopted hash-and-cache patterns that effectively moved driver complexity into application code. The rigid static state model proved incompatible with the dynamic rendering patterns common in modern game engines, which frequently change material state within a frame in ways that cannot be enumerated at compile time. A more radical solution was needed.

[Source — VK_EXT_shader_object proposal, KhronosGroup/Vulkan-Docs](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/proposals/VK_EXT_shader_object.adoc)

---

## 2. VK_EXT_shader_object Overview

### 2.1 History and Specification Status

`VK_EXT_shader_object` was introduced in Vulkan specification revision 1.3.246, published on 2023-03-30. It was developed collaboratively by engineers from NVIDIA, AMD, Valve, Google, ARM, Samsung, Imagination Technologies, Nintendo, Collabora, LunarG, Roblox, Activision, and Igalia. The extension carries 28 named contributors. [Source — Vulkan 1.3.246 release, Phoronix](https://www.phoronix.com/news/Vulkan-1.3.246-VK-shader-object)

**The extension was not promoted into Vulkan 1.4 core.** The Vulkan 1.4 specification, released in early 2024, promoted a different set of extensions — including `VK_EXT_host_image_copy`, `VK_KHR_push_descriptor`, `VK_KHR_maintenance5/6`, and several shader utility extensions — but `VK_EXT_shader_object` was not among them. [Source — Vulkan 1.4 features document](https://docs.vulkan.org/features/latest/features/proposals/VK_VERSION_1_4.html) It remains a ratified `EXT` extension at revision 1, requiring explicit enumeration and enablement. The normative reference for the extension is at [`registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_shader_object.html`](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_shader_object.html).

### 2.2 The Conceptual Shift: Shaders as First-Class Objects

The extension takes the most radical of three candidate approaches considered by the Khronos working group: rather than extending the pipeline model further, it abandons it. As the proposal states: "Abandon pipelines entirely and introduce new functionality to compile and bind shaders directly."

The core addition is the `VkShaderEXT` opaque handle. Each `VkShaderEXT` represents a single compiled shader stage — vertex, tessellation control, tessellation evaluation, geometry, fragment, or compute. These objects are created independently of any pipeline state. At draw time, the application binds one or more `VkShaderEXT` objects to the command buffer along with the dynamic state required for the draw, then issues the draw call. No `VkPipeline` is created or bound.

The implications are significant:
- Shader compilation is triggered per-stage, not per-pipeline-variant. A vertex shader compiled once can be used with any fragment shader.
- GPU driver state that was previously baked into the pipeline object (blend state, rasterization mode, depth compare op) is now set via `vkCmdSet*` commands immediately before the draw.
- The combinatorial explosion of pipeline variants collapses. Instead of `N * M * K` pipelines, the application maintains `N + M + K` shader stage variants.

### 2.3 VkShaderCreateInfoEXT and vkCreateShadersEXT

Shader objects are created with `vkCreateShadersEXT`, which accepts an array of `VkShaderCreateInfoEXT` structures and returns a corresponding array of `VkShaderEXT` handles. The batch API exists because linked shaders (see §5.4) must be created together.

```c
typedef struct VkShaderCreateInfoEXT {
    VkStructureType             sType;          // VK_STRUCTURE_TYPE_SHADER_CREATE_INFO_EXT
    const void*                 pNext;
    VkShaderCreateFlagsEXT      flags;          // e.g. VK_SHADER_CREATE_LINK_STAGE_BIT_EXT
    VkShaderStageFlagBits       stage;          // e.g. VK_SHADER_STAGE_VERTEX_BIT
    VkShaderStageFlags          nextStage;      // hint: which stages may follow
    VkShaderCodeTypeEXT         codeType;       // SPIRV or BINARY
    size_t                      codeSize;
    const void*                 pCode;          // SPIR-V words or binary blob
    const char*                 pName;          // entry point, e.g. "main"
    uint32_t                    setLayoutCount;
    const VkDescriptorSetLayout* pSetLayouts;
    uint32_t                    pushConstantRangeCount;
    const VkPushConstantRange*  pPushConstantRanges;
    const VkSpecializationInfo* pSpecializationInfo;
} VkShaderCreateInfoEXT;

// Creating a vertex and fragment shader from SPIR-V:
VkShaderCreateInfoEXT createInfos[2];

// Vertex shader
createInfos[0].sType     = VK_STRUCTURE_TYPE_SHADER_CREATE_INFO_EXT;
createInfos[0].pNext     = nullptr;
createInfos[0].flags     = VK_SHADER_CREATE_LINK_STAGE_BIT_EXT;
createInfos[0].stage     = VK_SHADER_STAGE_VERTEX_BIT;
createInfos[0].nextStage = VK_SHADER_STAGE_FRAGMENT_BIT;
createInfos[0].codeType  = VK_SHADER_CODE_TYPE_SPIRV_EXT;
createInfos[0].codeSize  = vertSpirv.size() * sizeof(uint32_t);
createInfos[0].pCode     = vertSpirv.data();
createInfos[0].pName     = "main";
createInfos[0].setLayoutCount         = 1;
createInfos[0].pSetLayouts            = &descriptorSetLayout;
createInfos[0].pushConstantRangeCount = 0;
createInfos[0].pPushConstantRanges    = nullptr;
createInfos[0].pSpecializationInfo    = nullptr;

// Fragment shader (mirror of above with stage/nextStage changed)
createInfos[1]           = createInfos[0];
createInfos[1].stage     = VK_SHADER_STAGE_FRAGMENT_BIT;
createInfos[1].nextStage = 0; // nothing follows fragment

VkShaderEXT shaders[2];
VkResult result = vkCreateShadersEXT(device, 2, createInfos, nullptr, shaders);
// shaders[0] = vertex shader, shaders[1] = fragment shader
```

[Source — Vulkan Samples shader_object documentation](https://docs.vulkan.org/samples/latest/samples/extensions/shader_object/README.html) The normative spec for `vkCreateShadersEXT` is at [`registry.khronos.org/vulkan/specs/latest/man/html/vkCreateShadersEXT.html`](https://registry.khronos.org/vulkan/specs/latest/man/html/vkCreateShadersEXT.html).

### 2.4 vkCmdBindShadersEXT

Shader objects are bound during command buffer recording with `vkCmdBindShadersEXT`:

```c
void vkCmdBindShadersEXT(
    VkCommandBuffer              commandBuffer,
    uint32_t                     stageCount,
    const VkShaderStageFlagBits* pStages,
    const VkShaderEXT*           pShaders  // NULL element = unbind that stage
);

// Bind vertex and fragment:
VkShaderStageFlagBits stages[2] = {
    VK_SHADER_STAGE_VERTEX_BIT,
    VK_SHADER_STAGE_FRAGMENT_BIT
};
vkCmdBindShadersEXT(cmd, 2, stages, shaders);

// Explicitly unbind the geometry stage (important if previously bound):
VkShaderStageFlagBits geoStage = VK_SHADER_STAGE_GEOMETRY_BIT;
VkShaderEXT          nullShader = VK_NULL_HANDLE;
vkCmdBindShadersEXT(cmd, 1, &geoStage, &nullShader);
```

A `NULL` element in `pShaders` unbinds the corresponding stage. Applications must ensure all stages active in the draw call are bound and compatible; Vulkan provides no draw-time validation and violations produce undefined behaviour. The normative spec for `vkCmdBindShadersEXT` is at [`registry.khronos.org/vulkan/specs/latest/man/html/vkCmdBindShadersEXT.html`](https://registry.khronos.org/vulkan/specs/latest/man/html/vkCmdBindShadersEXT.html).

### 2.5 Binary Shader Format

`vkGetShaderBinaryDataEXT` retrieves the driver's compiled binary representation of a `VkShaderEXT`:

```c
// Query size
size_t binarySize = 0;
vkGetShaderBinaryDataEXT(device, shaderObj, &binarySize, nullptr);

// Retrieve binary
std::vector<uint8_t> binary(binarySize);
vkGetShaderBinaryDataEXT(device, shaderObj, &binarySize, binary.data());

// Save to disk, then reload:
VkShaderCreateInfoEXT binaryInfo = /* same as original, but: */;
binaryInfo.codeType = VK_SHADER_CODE_TYPE_BINARY_EXT;
binaryInfo.codeSize = binary.size();
binaryInfo.pCode    = binary.data();

VkShaderEXT reloadedShader;
VkResult result = vkCreateShadersEXT(device, 1, &binaryInfo, nullptr, &reloadedShader);
if (result == VK_INCOMPATIBLE_SHADER_BINARY_EXT) {
    // Binary is incompatible with this device/driver version; fall back to SPIR-V
    compileSpirvFallback();
}
```

**Important:** the binary blob is device-specific and driver-version-specific. It is not portable across GPU vendors or even across major driver updates. `VK_INCOMPATIBLE_SHADER_BINARY_EXT` is a *success code* (not an error), following the pattern of `VK_INCOMPLETE`, allowing graceful fallback. Applications shipping on consoles or other fixed platforms gain the most value from binary serialisation, since device homogeneity guarantees compatibility.

[Source — VK_EXT_shader_object proposal, binary portability section](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_shader_object.html)

---

## 3. Dynamic State: The Companion Feature

### 3.1 The Implicit Dynamic State Contract

When `VK_EXT_shader_object` is enabled and at least one `VkShaderEXT` is bound for a draw call, *all pipeline state is dynamic*. There is no pipeline object to bake state into. Every piece of GPU state that was previously specified at pipeline creation time must now be commanded via `vkCmdSet*` functions before the draw call.

Crucially, the spec provides a key convenience: when the `shaderObject` feature is enabled, the dynamic state commands from `VK_EXT_extended_dynamic_state`, `VK_EXT_extended_dynamic_state2`, `VK_EXT_extended_dynamic_state3`, and `VK_EXT_vertex_input_dynamic_state` may be used **without those extensions being separately enumerated or enabled**. The driver that exposes `VK_EXT_shader_object` implicitly provides all of their commands. [Source — Khronos blog, "You Can Use Vulkan Without Pipelines Today"](https://www.khronos.org/blog/you-can-use-vulkan-without-pipelines-today)

### 3.2 Required Dynamic State Commands

Below is the state that must be set per-draw when using shader objects. The table groups commands by the extension that originally introduced them:

**Core Vulkan 1.0 dynamic state (already dynamic in many pipelines):**
- `vkCmdSetViewport` / `vkCmdSetScissor`
- `vkCmdSetLineWidth`
- `vkCmdSetDepthBias`
- `vkCmdSetBlendConstants`
- `vkCmdSetDepthBounds`
- `vkCmdSetStencilCompareMask` / `vkCmdSetStencilWriteMask` / `vkCmdSetStencilReference`

**From VK_EXT_extended_dynamic_state (promoted to Vulkan 1.3):**
- `vkCmdSetCullMode` — replaces `VkPipelineRasterizationStateCreateInfo::cullMode`
- `vkCmdSetFrontFace`
- `vkCmdSetPrimitiveTopology` — when using shader objects, the topology class restriction is lifted
- `vkCmdSetViewportWithCount` / `vkCmdSetScissorWithCount`
- `vkCmdSetDepthTestEnable` / `vkCmdSetDepthWriteEnable` / `vkCmdSetDepthCompareOp`
- `vkCmdSetDepthBoundsTestEnable`
- `vkCmdSetStencilTestEnable` / `vkCmdSetStencilOp`

**From VK_EXT_extended_dynamic_state2 (promoted to Vulkan 1.3):**
- `vkCmdSetRasterizerDiscardEnable`
- `vkCmdSetDepthBiasEnable`
- `vkCmdSetPrimitiveRestartEnable`
- `vkCmdSetPatchControlPointsEXT`

**From VK_EXT_extended_dynamic_state3 (not core as of Vulkan 1.4):**
- `vkCmdSetPolygonModeEXT`
- `vkCmdSetRasterizationSamplesEXT`
- `vkCmdSetSampleMaskEXT`
- `vkCmdSetAlphaToCoverageEnableEXT`
- `vkCmdSetAlphaToOneEnableEXT`
- `vkCmdSetColorBlendEnableEXT`
- `vkCmdSetColorBlendEquationEXT`
- `vkCmdSetColorWriteMaskEXT`
- `vkCmdSetDepthClampEnableEXT`
- `vkCmdSetLogicOpEnableEXT` / `vkCmdSetLogicOpEXT`
- `vkCmdSetConservativeRasterizationModeEXT` (if conservative rasterization is used)
- `vkCmdSetLineRasterizationModeEXT`, `vkCmdSetLineStippleEnableEXT`

**From VK_EXT_vertex_input_dynamic_state:**
- `vkCmdSetVertexInputEXT` — replaces `VkPipelineVertexInputStateCreateInfo`

### 3.3 Practical Approach: Setting Only Required State

Applications need not set all dynamic state for every draw call. The spec requires only state that is *active* for the current draw to be set. An application can maintain a dirty-state shadow of the current GPU state and issue only `vkCmdSet*` calls for state that has changed since the previous draw. This is analogous to the state tracking that OpenGL drivers do internally — but it is now explicit in application code, where it can be batched, cached, and threaded.

```cpp
// Helper: set all required rasterization state for a draw
void setRasterizationState(VkCommandBuffer cmd, const MaterialState& mat) {
    vkCmdSetCullModeEXT(cmd, mat.cullMode);
    vkCmdSetFrontFaceEXT(cmd, VK_FRONT_FACE_COUNTER_CLOCKWISE);
    vkCmdSetPolygonModeEXT(cmd, mat.wireframe ? VK_POLYGON_MODE_LINE
                                               : VK_POLYGON_MODE_FILL);
    vkCmdSetRasterizationSamplesEXT(cmd, mat.msaaSamples);
    vkCmdSetSampleMaskEXT(cmd, mat.msaaSamples, &mat.sampleMask);
    vkCmdSetAlphaToCoverageEnableEXT(cmd, mat.alphaToCoverage);
    vkCmdSetAlphaToOneEnableEXT(cmd, VK_FALSE);
}

void setDepthState(VkCommandBuffer cmd, const DepthState& ds) {
    vkCmdSetDepthTestEnableEXT(cmd, ds.testEnable);
    vkCmdSetDepthWriteEnableEXT(cmd, ds.writeEnable);
    vkCmdSetDepthCompareOpEXT(cmd, ds.compareOp);
    vkCmdSetDepthBoundsTestEnableEXT(cmd, VK_FALSE);
}

void setBlendState(VkCommandBuffer cmd, uint32_t attachmentCount,
                  const BlendState* blends) {
    for (uint32_t i = 0; i < attachmentCount; ++i) {
        vkCmdSetColorBlendEnableEXT(cmd, i, 1, &blends[i].enable);
        vkCmdSetColorBlendEquationEXT(cmd, i, 1, &blends[i].equation);
        vkCmdSetColorWriteMaskEXT(cmd, i, 1, &blends[i].writeMask);
    }
}
```

---

## 4. Interaction with VK_KHR_dynamic_rendering

### 4.1 Two Halves of the Same Problem

`VK_KHR_dynamic_rendering` (promoted to Vulkan 1.3 core) removes the requirement to create `VkRenderPass` and `VkFramebuffer` objects before rendering. Instead, render pass instances are begun with `vkCmdBeginRendering` and a `VkRenderingInfo` struct that specifies attachments inline. `VK_EXT_shader_object` removes the requirement to create `VkPipeline` objects. Together, the two extensions eliminate the two dominant sources of GPU-driver busy-wait at draw-preparation time in traditional Vulkan.

Importantly, `VK_EXT_shader_object` **requires** `VK_KHR_dynamic_rendering` (or Vulkan 1.3) as an extension dependency — a driver cannot expose `VK_EXT_shader_object` unless it also supports `VK_KHR_dynamic_rendering`. Furthermore, the Vulkan spec restricts graphics shader objects to use exclusively within dynamic render pass instances: shader objects must be used inside render passes begun with `vkCmdBeginRendering`, not with the legacy `vkCmdBeginRenderPass`. [Source — Vulkan shader_object sample: "Shader objects may only be used within VK_KHR_dynamic_rendering render passes"](https://docs.vulkan.org/samples/latest/samples/extensions/shader_object/README.html) [Source — Vulkan-ExtensionLayer, "The VK_EXT_shader_object extension requires VK_KHR_maintenance2 and VK_KHR_dynamic_rendering"](https://github.com/KhronosGroup/Vulkan-ExtensionLayer/blob/main/docs/shader_object_layer.md)

### 4.2 VkRenderingInfo Replacing VkRenderPassBeginInfo

```cpp
// Traditional pipeline draw loop (before):
VkRenderPassBeginInfo rpBegin{};
rpBegin.sType           = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO;
rpBegin.renderPass      = renderPass;   // pre-created VkRenderPass
rpBegin.framebuffer     = framebuffer; // pre-created VkFramebuffer
rpBegin.renderArea      = {{0,0}, {width, height}};
rpBegin.clearValueCount = 1;
rpBegin.pClearValues    = &clearColor;
vkCmdBeginRenderPass(cmd, &rpBegin, VK_SUBPASS_CONTENTS_INLINE);
vkCmdBindPipeline(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS, pipeline);
vkCmdDraw(cmd, 3, 1, 0, 0);
vkCmdEndRenderPass(cmd);
```

```cpp
// Shader object draw loop (after):
VkRenderingAttachmentInfo colorAttachment{};
colorAttachment.sType       = VK_STRUCTURE_TYPE_RENDERING_ATTACHMENT_INFO;
colorAttachment.imageView   = swapchainImageView;
colorAttachment.imageLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
colorAttachment.loadOp      = VK_ATTACHMENT_LOAD_OP_CLEAR;
colorAttachment.storeOp     = VK_ATTACHMENT_STORE_OP_STORE;
colorAttachment.clearValue  = {{0.0f, 0.0f, 0.0f, 1.0f}};

VkRenderingInfo renderingInfo{};
renderingInfo.sType                = VK_STRUCTURE_TYPE_RENDERING_INFO;
renderingInfo.renderArea           = {{0,0}, {width, height}};
renderingInfo.layerCount           = 1;
renderingInfo.colorAttachmentCount = 1;
renderingInfo.pColorAttachments    = &colorAttachment;

vkCmdBeginRendering(cmd, &renderingInfo);

// Bind shader objects
VkShaderStageFlagBits stages[] = {
    VK_SHADER_STAGE_VERTEX_BIT,
    VK_SHADER_STAGE_FRAGMENT_BIT
};
vkCmdBindShadersEXT(cmd, 2, stages, shaders);

// Set all required dynamic state
setRasterizationState(cmd, currentMaterial);
setDepthState(cmd, currentDepth);
setBlendState(cmd, 1, &currentBlend);
vkCmdSetPrimitiveTopologyEXT(cmd, VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST);
vkCmdSetViewportWithCountEXT(cmd, 1, &viewport);
vkCmdSetScissorWithCountEXT(cmd, 1, &scissor);

// Bind resources and draw
vkCmdBindDescriptorSets(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS,
                        pipelineLayout, 0, 1, &descriptorSet, 0, nullptr);
vkCmdDraw(cmd, vertexCount, 1, 0, 0);

vkCmdEndRendering(cmd);
```

### 4.3 Why Dynamic Rendering Is Required

The render-pass requirement is not an arbitrary coupling. The `VkRenderPass` object traditionally carried attachment format information used by some GPU architectures during shader compilation. By mandating `VK_KHR_dynamic_rendering`, the spec ensures that the extension can be built on a consistent rendering model where attachment state is passed at draw time rather than baked into render pass objects. The Khronos Extension Layer documentation for the software emulation layer confirms: "The `VK_EXT_shader_object` extension requires the `VK_KHR_maintenance2` and `VK_KHR_dynamic_rendering` extensions; this layer will not work on devices that do not implement them." [Source — Vulkan-ExtensionLayer shader_object_layer.md](https://github.com/KhronosGroup/Vulkan-ExtensionLayer/blob/main/docs/shader_object_layer.md)

A practical implication is that applications mixing the extension with UI rendering that uses legacy render passes (via `vkCmdBeginRenderPass`) must separate those paths: shader objects for the scene, traditional pipelines inside the legacy render pass. The Vulkan shader object sample explicitly demonstrates this: "pipelines within conventional render passes (which are used to render the UI) and shader objects with dynamic rendering (which are used to render the scene)." [Source — Vulkan shader object sample](https://docs.vulkan.org/samples/latest/samples/extensions/shader_object/README.html)

---

## 5. Shader Compilation Workflow

### 5.1 From SPIR-V to VkShaderEXT

When `vkCreateShadersEXT` is called with `VK_SHADER_CODE_TYPE_SPIRV_EXT`, the driver performs the standard SPIR-V → driver IR → GPU machine code compilation path. On Mesa drivers this means SPIR-V → NIR → ACO (RADV) or NIR → Intel ISA (ANV) or NIR → NAK (NVK). The key difference from pipeline compilation is that this compilation is scoped to a single stage: there is no inter-stage linking, no baking of vertex-input layout, no dependency on the target attachment format, and no dependency on colour-blend state.

This is where the stutter elimination occurs. The first compilation from SPIR-V may still take time (milliseconds), but it is amortised over far fewer variants. Applications typically need one vertex shader per geometry complexity level and one fragment shader per material type — not one pipeline per (geometry, material, lighting, attachment-format) combination.

### 5.2 Binary Export and Reload

Once compiled, the binary can be exported and cached:

```cpp
// After successful vkCreateShadersEXT from SPIR-V:
size_t binarySize = 0;
vkGetShaderBinaryDataEXT(device, vertShader, &binarySize, nullptr);

std::vector<uint8_t> vertBinary(binarySize);
vkGetShaderBinaryDataEXT(device, vertShader, &binarySize, vertBinary.data());

// Persist to disk, keyed by (shader hash, device UUID, driver version)
writeCacheFile(cacheKey, vertBinary);

// --- Next launch ---

// Attempt to load from cache:
std::vector<uint8_t> cachedBinary = readCacheFile(cacheKey);
if (!cachedBinary.empty()) {
    VkShaderCreateInfoEXT binaryInfo{};
    binaryInfo.sType     = VK_STRUCTURE_TYPE_SHADER_CREATE_INFO_EXT;
    binaryInfo.stage     = VK_SHADER_STAGE_VERTEX_BIT;
    binaryInfo.nextStage = VK_SHADER_STAGE_FRAGMENT_BIT;
    binaryInfo.codeType  = VK_SHADER_CODE_TYPE_BINARY_EXT;
    binaryInfo.codeSize  = cachedBinary.size();
    binaryInfo.pCode     = cachedBinary.data();
    binaryInfo.pName     = "main";
    // ... descriptor set layouts, push constants ...

    VkShaderEXT loaded;
    if (vkCreateShadersEXT(device, 1, &binaryInfo, nullptr, &loaded)
            == VK_INCOMPATIBLE_SHADER_BINARY_EXT) {
        // Binary stale — recompile from SPIR-V and update cache
        compileSpirvAndUpdateCache();
    }
}
```

The spec requires that binary creation be fast — no slower than 150% of the cost of copying an equivalent amount of data into device-local memory. [Source — Khronos blog performance guarantees](https://www.khronos.org/blog/you-can-use-vulkan-without-pipelines-today) This means binary reload is effectively free from a stutter perspective.

**Critical point:** Binary shaders are not portable. The cache key must include at minimum the device's `VkPhysicalDeviceProperties.pipelineCacheUUID` and the driver version. Distributing binary shaders across different GPUs or driver versions will trigger `VK_INCOMPATIBLE_SHADER_BINARY_EXT`.

### 5.3 Linked vs. Unlinked Shaders

The `VK_SHADER_CREATE_LINK_STAGE_BIT_EXT` flag enables **linked shaders**: a set of shader stages that the application promises will always be used together. When this bit is set, all stages in the linked set must be created in the same `vkCreateShadersEXT` call (i.e., as an array of infos). The driver can then perform cross-stage optimisations such as interface variable elimination.

**Linked shaders** incur a one-time cross-stage compilation cost at creation time, amortised over subsequent draws. They are appropriate when the vertex/fragment combination is fixed for a material class.

**Unlinked shaders** (the default, no `LINK_STAGE_BIT`) are compiled independently with no knowledge of other stages. They provide maximum flexibility for mix-and-match binding — any vertex shader can be combined with any compatible fragment shader. The spec notes that linked shaders "may perform anywhere between the same to substantially better than equivalent unlinked shaders."

Compute shaders are always unlinked (they are single-stage by definition).

### 5.4 Shader Variant Management

The shader-object model changes the structure of variant management. In the pipeline model, a variant is a (vertex_state, fragment_state, blend_state, …) tuple that maps to exactly one `VkPipeline`. In the shader-object model, a variant is per-stage. A material system that previously tracked 10,000 pipeline variants might instead track:

- 200 vertex shader variants (geometry complexity, skinning, morphing)
- 50 fragment shader variants (shading model, feature flags)
- A small set of per-draw dynamic state values set via `vkCmdSet*`

The application's variant cache shrinks from the cross-product to the sum of per-stage variants.

---

## 6. Mesa Driver Implementations

### 6.1 RADV (AMD)

RADV was among the first Mesa drivers to implement `VK_EXT_shader_object`. Initial support was merged behind the `RADV_PERFTEST=shader_object` environment flag in Mesa 23.x; the flag was used to enable it in CI testing against the Polaris10 platform in early 2024. [Source — Mesa GitLab MR !27139](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/27139)

In Mesa 24.1 (released May 2024), Samuel Pitoiset (Valve) landed the change to enable `VK_EXT_shader_object` by default in RADV without requiring any opt-in flag. `RADV_DEBUG=noeso` is now available as an opt-out for testing or regression isolation. [Source — Phoronix, "RADV Vulkan Driver Enables EXT_shader_object By Default With Mesa 24.1"](https://www.phoronix.com/news/RADV-Default-ESO-Support)

The implementation lives at `src/amd/vulkan/radv_shader_object.c` within the Mesa repository at [gitlab.freedesktop.org/mesa/mesa](https://gitlab.freedesktop.org/mesa/mesa). This file implements the `vkCreateShadersEXT`, `vkDestroyShaderEXT`, `vkCmdBindShadersEXT`, and `vkGetShaderBinaryDataEXT` entry points for RADV.

**ACO and shader objects.** RADV's ACO compiler backend (located at `src/amd/compiler/`) operates on NIR IR produced from SPIR-V. For shader objects, ACO compiles each stage independently at `vkCreateShadersEXT` time. Because no inter-stage linking occurs for unlinked shader objects, ACO cannot perform interface-based optimisations (such as eliminating outputs not consumed by the next stage) unless `VK_SHADER_CREATE_LINK_STAGE_BIT_EXT` is set. When linked, RADV performs a NIR-level I/O optimisation pass before invoking ACO.

The binary data returned by `vkGetShaderBinaryDataEXT` in RADV contains the AMD GFX ISA (RDNA/GCN machine code) for the compiled shader together with any driver-internal metadata needed for shader binding. The exact binary format is an internal implementation detail of RADV; the spec requires only that it round-trips correctly through `vkCreateShadersEXT` with `VK_SHADER_CODE_TYPE_BINARY_EXT` on a compatible device. The binary is tied to the specific GFX generation (GFX10 for RDNA 2, GFX11 for RDNA 3, etc.) and is not portable across AMD GPU families or across major RADV version upgrades.

### 6.2 ANV (Intel)

ANV's support for `VK_EXT_shader_object` arrived significantly later. Patches enabling the extension in the Intel ANV open-source Vulkan driver landed in Mesa 25.3-devel (development branch active in late 2025). [Source — Phoronix, "Intel's Vulkan Linux Driver Finally Exposes VK_EXT_shader_object"](https://www.phoronix.com/news/Intel-ANV-VK_EXT_shader_object)

The ANV shader compilation path follows the standard Mesa pattern: SPIR-V → NIR → Intel ISA via the `src/intel/compiler/` backend (the Intel EU or Xe ISA compiler, depending on GPU generation). For Arc (Xe) hardware, the compiler targets the Xe ISA. The shader object implementation in ANV is located in the `src/intel/vulkan/` directory — **Note: needs verification** of the exact filename, as the ANV shader object file path was not publicly documented at time of writing.

The reason for ANV's later adoption relative to RADV and NVK is not publicly documented. **Note: needs verification** of the specific Intel architectural constraints that caused the implementation to take longer.

### 6.3 NVK (NVIDIA)

The NVK open-source NVIDIA Vulkan driver, maintained primarily by Faith Ekstrand and Collabora, added `VK_EXT_shader_object` support in February 2024. This was merged alongside `VK_EXT_graphics_pipeline_library` support. [Source — Phoronix, "NVK Vulkan Driver Lands Shader Object & Graphics Pipeline Library"](https://www.phoronix.com/news/NVK-ESO-GPL-Support) [Source — Mesa 24.1 Collabora blog](https://www.collabora.com/news-and-blog/news-and-events/mesa-241-brings-new-hardware-support-for-arm-and-nvidia-gpus.html)

A notable contribution of the NVK implementation was the introduction of a **common Mesa Vulkan runtime framework** for shader objects. Rather than implementing the VK_EXT_shader_object API entirely per-driver, the NVK team added infrastructure to the shared Mesa Vulkan runtime (`src/vulkan/runtime/`) that makes everything internally look like shader objects. Drivers opt into baking state into shaders by providing a `vk_graphics_pipeline_state` at compile time, in which case the runtime hashes the baked state. Drivers that do not provide this struct get fully dynamic behaviour. This framework benefits all Mesa Vulkan drivers including ANV, which can rely on the shared infrastructure rather than re-implementing the shader object state machine.

NVK's shader compiler backend is **NAK** (Nouveau Accelerated Kompiler, `src/nouveau/compiler/`), written in Rust and merged into Mesa 24.0. NAK compiles NIR to NVIDIA ISA and is used by default for Turing (RTX 20 series, GTX 16xx, SM75) and later GPUs; older architectures use the legacy backend. [Source — Phoronix, "Rust-Written NAK Compiler Merged For Nouveau/NVK In Mesa 24.0"](https://www.phoronix.com/news/NAK-Merged-Mesa-24.0) NAK handles per-stage compilation for shader objects. The NVK implementation is located in `src/nouveau/vulkan/`.

### 6.4 The Khronos Extension Layer (Software Fallback)

For drivers that do not yet support `VK_EXT_shader_object` natively, Khronos provides `VK_LAYER_KHRONOS_shader_object` in the [Vulkan-ExtensionLayer repository](https://github.com/KhronosGroup/Vulkan-ExtensionLayer). This layer implements the extension entirely in software, translating shader object API calls into pipeline create/bind calls internally. It requires `VK_KHR_dynamic_rendering` and `VK_KHR_maintenance2` on the underlying driver. The layer uses pipeline pre-caching internally (controllable via `VK_SHADER_OBJECT_DISABLE_PIPELINE_PRE_CACHING`). It automatically disables itself when the driver natively exposes the extension.

---

## 7. Performance Characteristics

### 7.1 Elimination of Compile Stutter

The primary motivation for `VK_EXT_shader_object` is the elimination of first-use pipeline compilation stutter. When the initial compilation from SPIR-V is performed at `vkCreateShadersEXT` time — typically during asset loading, scene streaming, or a background precompile pass — the per-draw code path requires no compilation work. The binary blob path (`VK_SHADER_CODE_TYPE_BINARY_EXT`) further reduces this: binary reload is bounded to 150% of a data copy cost, making it practically instant. [Source — Khronos blog performance guarantees](https://www.khronos.org/blog/you-can-use-vulkan-without-pipelines-today)

No publicly available rigorous quantitative benchmark data measures the exact stutter reduction from `VK_EXT_shader_object` versus `VkPipeline` + caching in released titles. The extension's stutter-elimination benefit comes primarily from the collapsed variant space (fewer distinct compilation units) and the portable binary cache (no cold-cache first runs), both of which are architectural improvements rather than single-number benchmark wins.

### 7.2 Per-Draw CPU Overhead: The Spec's Conformance Guarantee

The spec defines explicit CPU overhead conformance requirements:

- Draw calls using shader objects must not take more than **150% of the CPU time** of draw calls using fully static graphics pipelines.
- Draw calls using shader objects must not take more than **120% of the CPU time** of draw calls using maximally dynamic graphics pipelines (pipelines with all supported dynamic state enabled).
- Compute dispatch with shader objects must not be measurably slower than compute pipeline dispatch.
- Binary shader creation must not exceed **150% of the cost of copying equivalent data** into device-local memory.

These guarantees are minimum conformance floors, not typical-case numbers. On implementations where pipeline state was already internally dynamic (such as NVIDIA's unified shader binding table model), the actual per-draw overhead of shader objects may be comparable to or lower than pipeline overhead. On implementations where state baking in the pipeline provides genuine optimisation (such as AMD hardware where shader variants are tuned to blend state), the shader object path may carry a small per-draw premium relative to the most optimised static pipeline. The 150%/120% bounds ensure this premium is architecturally bounded.

### 7.3 GPU-Side Overhead

GPU-side overhead from dynamic state is hardware-dependent. On AMD RDNA hardware, some rasterization state is uploaded to a set of registers that the PM4 command stream must program before each draw. In a pipeline model, this register programming is partly amortised by delta tracking (the driver only reprograms changed registers between pipeline binds). With shader objects, the application is responsible for tracking what has changed and calling only the relevant `vkCmdSet*` functions. A well-implemented application state tracker can achieve the same or better delta tracking than a driver's pipeline-bind logic.

The critical insight is that the delta-tracking work still happens — it moves from the driver into the application. An application with a poor state-tracker that calls every `vkCmdSet*` command before every draw — regardless of what changed — will incur a measurable CPU overhead penalty. Correct use of shader objects requires the same disciplined state management that high-performance OpenGL drivers applied internally.

### 7.4 Application State Tracker Design

A robust shader-object application state tracker maintains a shadow copy of the current dynamic GPU state and issues `vkCmdSet*` calls only for dirty entries. A minimal structure looks like:

```cpp
struct ShaderObjectState {
    // Rasterization
    VkCullModeFlags    cullMode       = VK_CULL_MODE_BACK_BIT;
    VkFrontFace        frontFace      = VK_FRONT_FACE_COUNTER_CLOCKWISE;
    VkPolygonMode      polygonMode    = VK_POLYGON_MODE_FILL;
    VkBool32           depthBiasEnable = VK_FALSE;
    // Depth
    VkBool32           depthTestEnable  = VK_TRUE;
    VkBool32           depthWriteEnable = VK_TRUE;
    VkCompareOp        depthCompareOp   = VK_COMPARE_OP_LESS;
    // Blend (per attachment, simplified)
    VkBool32           blendEnable    = VK_FALSE;
    // Dirty flags
    uint64_t           dirtyFlags     = ~0ULL; // all dirty on init
};

void flushDirtyState(VkCommandBuffer cmd, ShaderObjectState& state,
                     const ShaderObjectState& desired) {
    if (state.cullMode != desired.cullMode) {
        vkCmdSetCullModeEXT(cmd, desired.cullMode);
        state.cullMode = desired.cullMode;
    }
    if (state.depthTestEnable != desired.depthTestEnable) {
        vkCmdSetDepthTestEnableEXT(cmd, desired.depthTestEnable);
        state.depthTestEnable = desired.depthTestEnable;
    }
    // ... repeat for all state ...
}
```

This pattern is conceptually identical to what a Vulkan driver does internally when switching between pipelines with different dynamic state: compare, set dirty, emit commands for changed entries only.

### 7.5 When to Prefer Shader Objects vs. Pipeline Libraries

| Scenario | Recommendation |
|---|---|
| New rendering engine with dynamic material system | Shader objects: matches natural engine structure |
| Porting D3D12 StateObjects or Metal Function Pipeline | Shader objects: closest semantic match |
| Existing engine with well-profiled pipeline cache | Pipeline libraries: lower porting risk, incremental improvement |
| Targeting NVIDIA proprietary driver (not NVK) | Verify driver support; proprietary driver added shader object support in 2023 beta |
| Fixed-function compute workloads | Either; compute pipelines and compute shader objects are equivalent |
| Ray tracing | Shader objects not yet supported (spec explicitly excludes ray tracing stages) |
| Console / fixed-platform titles | Shader objects with binary caching: eliminates the shader compilation layer entirely |
| Cross-platform engine with MoltenVK (macOS/iOS) | Verify MoltenVK support; dynamic_rendering dependency means Android may be limited |

The hybrid approach — using shader objects for graphics stages while keeping compute pipelines — is valid and supported. Mixing pipelines and shader objects within the same frame is permitted by the spec.

---

## 8. Migration Guide: From VkPipeline to VkShaderEXT

### 8.1 Feature Detection

```cpp
// Check for shader object support
VkPhysicalDeviceShaderObjectFeaturesEXT shaderObjFeatures{};
shaderObjFeatures.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_SHADER_OBJECT_FEATURES_EXT;

VkPhysicalDeviceFeatures2 features2{};
features2.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_FEATURES_2;
features2.pNext = &shaderObjFeatures;
vkGetPhysicalDeviceFeatures2(physicalDevice, &features2);

if (!shaderObjFeatures.shaderObject) {
    // Fall back to VkPipeline path
    useShaderObjects = false;
} else {
    useShaderObjects = true;
}

// Enable at device creation:
VkPhysicalDeviceShaderObjectFeaturesEXT enableShaderObj{};
enableShaderObj.sType        = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_SHADER_OBJECT_FEATURES_EXT;
enableShaderObj.shaderObject = VK_TRUE;

VkDeviceCreateInfo deviceInfo{};
deviceInfo.pNext = &enableShaderObj;
// ...
```

### 8.2 Before and After: A Complete Graphics Draw

The following comparison shows a minimal triangle draw using the traditional pipeline approach versus shader objects.

**Before (VkPipeline):**

```cpp
// --- Setup phase (done once, typically at application init) ---

// 1. Create VkRenderPass
VkRenderPassCreateInfo rpCI{};
// ... fill attachment, subpass, dependency descriptions ...
VkRenderPass renderPass;
vkCreateRenderPass(device, &rpCI, nullptr, &renderPass);

// 2. Create VkFramebuffer
VkFramebufferCreateInfo fbCI{};
fbCI.renderPass      = renderPass;
fbCI.attachmentCount = 1;
fbCI.pAttachments    = &swapImageView;
fbCI.width  = width; fbCI.height = height; fbCI.layers = 1;
VkFramebuffer framebuffer;
vkCreateFramebuffer(device, &fbCI, nullptr, &framebuffer);

// 3. Create VkShaderModule(s)
VkShaderModuleCreateInfo vertSMCI{};
vertSMCI.codeSize = vertSpirv.size() * 4;
vertSMCI.pCode    = vertSpirv.data();
VkShaderModule vertModule;
vkCreateShaderModule(device, &vertSMCI, nullptr, &vertModule);
// ... same for fragment ...

// 4. Create VkPipeline
VkGraphicsPipelineCreateInfo pipelineCI{};
// ... dozens of fields: shader stages, vertex input, rasterization,
//     depth/stencil, blend, viewport, dynamic state list, render pass ... 
VkPipeline pipeline;
vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &pipelineCI,
                          nullptr, &pipeline);
// (This step may take 10–200ms on first compile)

// --- Draw phase (per frame) ---
vkCmdBeginRenderPass(cmd, &rpBeginInfo, VK_SUBPASS_CONTENTS_INLINE);
vkCmdBindPipeline(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS, pipeline);
vkCmdBindDescriptorSets(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS,
                        pipelineLayout, 0, 1, &descriptorSet, 0, nullptr);
vkCmdDraw(cmd, 3, 1, 0, 0);
vkCmdEndRenderPass(cmd);
```

**After (VkShaderEXT):**

```cpp
// --- Setup phase ---

// 1. Create shader objects from SPIR-V (or from binary cache)
VkShaderCreateInfoEXT shaderInfos[2] = {};

shaderInfos[0].sType     = VK_STRUCTURE_TYPE_SHADER_CREATE_INFO_EXT;
shaderInfos[0].flags     = VK_SHADER_CREATE_LINK_STAGE_BIT_EXT;
shaderInfos[0].stage     = VK_SHADER_STAGE_VERTEX_BIT;
shaderInfos[0].nextStage = VK_SHADER_STAGE_FRAGMENT_BIT;
shaderInfos[0].codeType  = VK_SHADER_CODE_TYPE_SPIRV_EXT;
shaderInfos[0].codeSize  = vertSpirv.size() * sizeof(uint32_t);
shaderInfos[0].pCode     = vertSpirv.data();
shaderInfos[0].pName     = "main";
shaderInfos[0].setLayoutCount         = 1;
shaderInfos[0].pSetLayouts            = &descriptorSetLayout;
shaderInfos[0].pushConstantRangeCount = 0;

shaderInfos[1]           = shaderInfos[0];
shaderInfos[1].stage     = VK_SHADER_STAGE_FRAGMENT_BIT;
shaderInfos[1].nextStage = 0;
shaderInfos[1].codeSize  = fragSpirv.size() * sizeof(uint32_t);
shaderInfos[1].pCode     = fragSpirv.data();

VkShaderEXT shaders[2];
vkCreateShadersEXT(device, 2, shaderInfos, nullptr, shaders);
// No VkRenderPass, no VkFramebuffer, no VkPipeline

// --- Draw phase ---
VkRenderingAttachmentInfo colorAttach{};
colorAttach.sType       = VK_STRUCTURE_TYPE_RENDERING_ATTACHMENT_INFO;
colorAttach.imageView   = swapImageView;
colorAttach.imageLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
colorAttach.loadOp      = VK_ATTACHMENT_LOAD_OP_CLEAR;
colorAttach.storeOp     = VK_ATTACHMENT_STORE_OP_STORE;
colorAttach.clearValue  = {.color = {0.0f, 0.0f, 0.0f, 1.0f}};

VkRenderingInfo renderInfo{};
renderInfo.sType                = VK_STRUCTURE_TYPE_RENDERING_INFO;
renderInfo.renderArea           = {{0,0}, {width, height}};
renderInfo.layerCount           = 1;
renderInfo.colorAttachmentCount = 1;
renderInfo.pColorAttachments    = &colorAttach;

vkCmdBeginRendering(cmd, &renderInfo);

// Bind shaders
VkShaderStageFlagBits stages[] = {VK_SHADER_STAGE_VERTEX_BIT,
                                   VK_SHADER_STAGE_FRAGMENT_BIT};
vkCmdBindShadersEXT(cmd, 2, stages, shaders);

// Set required dynamic state (must be set before the draw)
vkCmdSetViewportWithCountEXT(cmd, 1, &viewport);
vkCmdSetScissorWithCountEXT(cmd, 1, &scissor);
vkCmdSetCullModeEXT(cmd, VK_CULL_MODE_BACK_BIT);
vkCmdSetFrontFaceEXT(cmd, VK_FRONT_FACE_COUNTER_CLOCKWISE);
vkCmdSetPrimitiveTopologyEXT(cmd, VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST);
vkCmdSetRasterizerDiscardEnableEXT(cmd, VK_FALSE);
vkCmdSetDepthBiasEnableEXT(cmd, VK_FALSE);
vkCmdSetDepthTestEnableEXT(cmd, VK_FALSE);
vkCmdSetDepthWriteEnableEXT(cmd, VK_FALSE);
vkCmdSetStencilTestEnableEXT(cmd, VK_FALSE);
vkCmdSetPolygonModeEXT(cmd, VK_POLYGON_MODE_FILL);
vkCmdSetRasterizationSamplesEXT(cmd, VK_SAMPLE_COUNT_1_BIT);
VkSampleMask sampleMask = 0xFFFFFFFF;
vkCmdSetSampleMaskEXT(cmd, VK_SAMPLE_COUNT_1_BIT, &sampleMask);
vkCmdSetAlphaToCoverageEnableEXT(cmd, VK_FALSE);
vkCmdSetAlphaToOneEnableEXT(cmd, VK_FALSE);
VkBool32 colorBlendEnable = VK_FALSE;
vkCmdSetColorBlendEnableEXT(cmd, 0, 1, &colorBlendEnable);
VkColorComponentFlags writeMask = VK_COLOR_COMPONENT_R_BIT |
                                   VK_COLOR_COMPONENT_G_BIT |
                                   VK_COLOR_COMPONENT_B_BIT |
                                   VK_COLOR_COMPONENT_A_BIT;
vkCmdSetColorWriteMaskEXT(cmd, 0, 1, &writeMask);

// Vertex input (if using VK_EXT_vertex_input_dynamic_state)
vkCmdSetVertexInputEXT(cmd, 0, nullptr, 0, nullptr); // no vertex buffers

// Bind resources and draw
vkCmdBindDescriptorSets(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS,
                        pipelineLayout, 0, 1, &descriptorSet, 0, nullptr);
vkCmdDraw(cmd, 3, 1, 0, 0);

vkCmdEndRendering(cmd);
```

### 8.3 APIs That Still Take VkPipeline

Not all Vulkan APIs accept a shader object in place of a pipeline. The following APIs still require a `VkPipeline` handle or have no shader-object equivalent:

- `vkCmdBindPipeline` — replaced by `vkCmdBindShadersEXT`; do not call both for the same draw
- `vkCreateComputePipelines` — compute pipelines remain valid; compute shader objects are the alternative but not a requirement
- Ray tracing pipeline creation (`vkCreateRayTracingPipelinesKHR`) — ray tracing is explicitly excluded from `VK_EXT_shader_object`

Descriptor sets (`vkCmdBindDescriptorSets`) work identically with shader objects. The `VkPipelineLayout` is still required to specify the descriptor binding layout; it is passed via `pSetLayouts` in `VkShaderCreateInfoEXT` rather than at pipeline creation time.

**Compute shader objects.** For compute dispatches, `VkShaderEXT` objects with `stage = VK_SHADER_STAGE_COMPUTE_BIT` can be bound and dispatched as follows:

```cpp
// Create a compute shader object
VkShaderCreateInfoEXT compInfo{};
compInfo.sType     = VK_STRUCTURE_TYPE_SHADER_CREATE_INFO_EXT;
compInfo.stage     = VK_SHADER_STAGE_COMPUTE_BIT;
compInfo.nextStage = 0; // compute is always terminal
compInfo.codeType  = VK_SHADER_CODE_TYPE_SPIRV_EXT;
compInfo.codeSize  = compSpirv.size() * sizeof(uint32_t);
compInfo.pCode     = compSpirv.data();
compInfo.pName     = "main";
compInfo.setLayoutCount         = 1;
compInfo.pSetLayouts            = &compDescSetLayout;
compInfo.pushConstantRangeCount = 1;
compInfo.pPushConstantRanges    = &pushRange;

VkShaderEXT compShader;
vkCreateShadersEXT(device, 1, &compInfo, nullptr, &compShader);

// Dispatch using shader object:
VkShaderStageFlagBits compStage = VK_SHADER_STAGE_COMPUTE_BIT;
vkCmdBindShadersEXT(cmd, 1, &compStage, &compShader);
vkCmdBindDescriptorSets(cmd, VK_PIPELINE_BIND_POINT_COMPUTE,
                        compPipelineLayout, 0, 1, &compDescSet, 0, nullptr);
vkCmdDispatch(cmd, groupCountX, groupCountY, groupCountZ);
```

The spec guarantees that compute shader object dispatch is not measurably slower than compute pipeline dispatch. For most compute workloads — where the per-variant cost is already low (one shader stage, no fixed-function state) — the practical benefit of shader objects over compute pipelines is mainly the consistent binary-caching API rather than compile-stutter elimination.

### 8.4 Mixing Shader Objects and Pipelines in the Same Frame

The spec permits mixing. Within a single command buffer, some draw calls can use `VkPipeline` (via `vkCmdBindPipeline`) and others can use shader objects (via `vkCmdBindShadersEXT`). However, the two binding models are exclusive per draw call — issuing `vkCmdBindPipeline` after `vkCmdBindShadersEXT` switches back to the pipeline model for subsequent draws until `vkCmdBindShadersEXT` is called again. This allows incremental migration: a large engine can migrate one render pass at a time without changing other code paths.

### 8.5 Managing the VkPipelineLayout Requirement

One sometimes-overlooked porting consideration is that `VkPipelineLayout` is still required. In the pipeline model, the pipeline layout is specified at `vkCreateGraphicsPipelines` time. With shader objects, the layout is specified per-stage in `VkShaderCreateInfoEXT::pSetLayouts`. All shader stages used in a draw must have compatible layouts. Applications should centralise layout management — typically one layout per material tier (opaque, transparent, UI) — rather than duplicating layouts per shader.

---

## Roadmap

### Near-term (6–12 months)

- **ANV (Intel) production hardening**: Intel's ANV driver only exposed `VK_EXT_shader_object` in Mesa 25.3-devel (late 2025), making it the last of the three major Mesa Vulkan drivers to reach parity. Near-term work focuses on closing performance gaps — particularly for linked-stage binaries on Xe-HPG and Xe2 hardware — and enabling the extension unconditionally rather than behind a feature flag. [Source — Intel's Vulkan Linux Driver Finally Exposes VK_EXT_shader_object, Phoronix](https://www.phoronix.com/news/Intel-ANV-VK_EXT_shader_object)
- **Zink (OpenGL-on-Vulkan) shader object path**: Zink already landed initial `VK_EXT_shader_object` support as an alternative to its pipeline-based code path. Near-term effort is directed at qualifying the shader-object path for default use on drivers that expose the extension, eliminating the Zink-internal pipeline cache layer entirely for those drivers. [Source — Zink OpenGL-On-Vulkan Driver Enables Shader Object Support, Phoronix Forums](https://www.phoronix.com/forums/forum/linux-graphics-x-org-drivers/opengl-vulkan-mesa-gallium3d/1384817-zink-opengl-on-vulkan-driver-enables-shader-object-support)
- **`VK_KHR_pipeline_binary` interop**: The ratified `VK_KHR_pipeline_binary` extension (Vulkan 1.4) provides a driver-agnostic, versioned binary caching mechanism for compiled shader code. Work is ongoing to align the `vkGetShaderBinaryDataEXT` / `vkCreateShadersEXT` binary round-trip with the `VK_KHR_pipeline_binary` model so applications can use a single binary store for both code paths. [Source — VK_KHR_pipeline_binary proposal, Vulkan Documentation Project](https://docs.vulkan.org/features/latest/features/proposals/VK_KHR_pipeline_binary.html)
- **`VK_EXT_descriptor_heap` co-design**: The Khronos working group published `VK_EXT_descriptor_heap` in early 2026 as a ground-up redesign of Vulkan's descriptor model. Because shader objects decouple shader compilation from descriptor binding, they are a natural fit for descriptor-heap-style binding; ensuring that the two extensions compose correctly without requiring a `VkPipelineLayout` is an active area. [Source — Vulkan Introduces Roadmap 2026 and New Descriptor Heap Extension, Khronos Blog](https://www.khronos.org/blog/vulkan-introduces-roadmap-2026-and-new-descriptor-heap-extension)

### Medium-term (1–3 years)

- **Possible promotion to `KHR` or Vulkan core**: `VK_EXT_shader_object` was not included in Vulkan 1.4 core promotions, but the Khronos working group has indicated that broad cross-vendor adoption (RADV, NVK, ANV, proprietary NVIDIA and AMD drivers, ARM Mali, Imagination) is the prerequisite for KHR promotion. If all major implementations stabilise, promotion could appear in a future Vulkan Roadmap milestone beyond 2026. Note: no concrete timeline has been announced; this is speculative based on standard Khronos promotion criteria. [Source — Vulkan Roadmap 2026 Milestone, Khronos Blog](https://www.khronos.org/blog/vulkan-introduces-roadmap-2026-and-new-descriptor-heap-extension)
- **Ray-tracing shader stage support**: The current spec explicitly excludes ray-tracing pipeline stages (`VK_SHADER_STAGE_RAYGEN_BIT_KHR`, `VK_SHADER_STAGE_CLOSEST_HIT_BIT_KHR`, etc.) from `VK_EXT_shader_object`. Extending the extension to cover ray-tracing stages — allowing individual ray-tracing shaders to be compiled and bound without a `VkRayTracingPipelineKHR` — is a logical follow-on but requires resolving shader-table binding semantics. Note: no formal proposal has been published; this direction has been discussed in Khronos forums. [Source — VK_EXT_shader_object Khronos Forums thread](https://community.khronos.org/t/vk-ext-shader-object/109628)
- **Mesh and task shader object parity**: `VK_EXT_mesh_shader` stage objects (`VK_SHADER_STAGE_MESH_BIT_EXT`, `VK_SHADER_STAGE_TASK_BIT_EXT`) are nominally in scope for `VK_EXT_shader_object` but hardware support is inconsistent. On GPUs without native mesh shader support, `vkCmdBindShadersEXT` must unbind those stages rather than emulate them. Broader hardware coverage and validation layer improvements for mesh-shader-object combinations are expected as mesh shaders become more prevalent in the Vulkan Roadmap 2026 hardware tier.
- **Game engine adoption maturity**: Unreal Engine's RHI and Unity's DOTS renderer are exploring shader-object-based backends to replace their pipeline-variant caches. Practical feedback from these large-scale integrations is expected to drive spec clarifications and new helper extensions (e.g., a standardised stutter-free warm-up API). Note: engine roadmap details are not public; this reflects publicly available developer discussions.

### Long-term

- **Vulkan successor or core-without-pipelines profile**: Some members of the Vulkan working group have publicly speculated that a future "Vulkan 2.0" or a minimal-profile specification might make the shader-object model the primary path, with `VkPipeline` becoming a compatibility layer. This would invert the current relationship and allow IHV driver teams to optimise primarily for the unbundled-state model. Note: this is highly speculative and no formal proposal exists.
- **Integration with GPU shader execution environments beyond rasterisation**: As heterogeneous compute (ML inference, ray tracing, video encode/decode shaders) converges on a unified Vulkan execution model, per-stage shader objects may be extended to cover additional programmable stages that today lack `VkPipeline` analogues. The `VK_EXT_descriptor_heap` trajectory suggests Khronos is willing to revisit foundational abstractions; shader objects may similarly expand in scope.
- **Formal standardisation of binary portability**: Today, `VkShaderEXT` binaries from `vkGetShaderBinaryDataEXT` are driver-specific blobs. Long-term, there is interest in a standardised container format (analogous to SPIR-V but for post-compiled ISA) that would allow shader binaries to be shared across driver versions or even vendors implementing the same ISA (e.g., different Mesa RADV versions). Note: needs verification — no concrete proposal has been published as of mid-2026.

---

## 9. Integrations

This chapter connects to several other chapters in the book:

**Ch24 — Vulkan and EGL for Application Developers** (`ch24-vulkan-egl-application-developers.md`)
Chapter 24 establishes the Vulkan instance, device, and surface creation workflow that is a prerequisite for everything in this chapter. The feature-detection pattern for `VkPhysicalDeviceShaderObjectFeaturesEXT` follows the same `pNext` chain idiom explained there. EGL surface creation is unchanged by shader objects; the swapchain image views fed to `VkRenderingAttachmentInfo` are obtained by the same path.

**Ch148 — Vulkan Synchronisation** (`ch148-vulkan-synchronisation.md`)
Shader objects do not change the synchronisation model. Image layout transitions for swapchain images (`UNDEFINED → COLOR_ATTACHMENT_OPTIMAL → PRESENT_SRC_KHR`) must still be performed with `VkImageMemoryBarrier2`, and pipeline stage flags (`VK_PIPELINE_STAGE_2_COLOR_ATTACHMENT_OUTPUT_BIT`, `VK_PIPELINE_STAGE_2_ALL_GRAPHICS_BIT`) apply identically. The move from `VkRenderPass` to `vkCmdBeginRendering` does not affect barrier requirements.

**Ch157 — Vulkan Descriptor Binding** (`ch157-vulkan-descriptor-binding.md`)
Descriptor sets, descriptor set layouts, and push constants work without change. The only difference is that the `VkPipelineLayout` used with `vkCmdBindDescriptorSets` must be consistent with the `pSetLayouts` array passed in `VkShaderCreateInfoEXT`. Push descriptors (`VK_KHR_push_descriptor`, promoted to Vulkan 1.4) work identically with shader objects.

**Ch154 — GPU-Driven Rendering** (`ch154-gpu-driven-rendering.md`)
Indirect draws (`vkCmdDrawIndirect`, `vkCmdDrawIndirectCount`, `vkCmdDrawMeshTasksIndirectEXT`) function identically with shader objects. The command buffer records the same draw commands; shader objects affect only the shader binding and dynamic state, not the draw command encoding. GPU-driven culling pipelines that produce indirect draw buffers require no changes to interoperate with a shader-object-based draw submission path.

**Ch143 — RADV Internals** (**Note: chapter number as referenced in plan.md**)
The RADV chapter covers the ACO compiler architecture that underlies the RADV `VK_EXT_shader_object` implementation described in §6.1. ACO's NIR optimisation passes, register allocation, and ISA emission are the same for shader objects as for pipeline shaders; the primary difference is the absence of inter-stage linking (unless `VK_SHADER_CREATE_LINK_STAGE_BIT_EXT` is used). Readers interested in the compiler-level detail of how RADV translates `vkGetShaderBinaryDataEXT` binaries back to GPU ISA should refer to that chapter.

---

*Sources referenced in this chapter:*

- [VK_EXT_shader_object proposal — KhronosGroup/Vulkan-Docs](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/proposals/VK_EXT_shader_object.adoc)
- [VK_EXT_shader_object features page — Vulkan Documentation Project](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_shader_object.html)
- [Vulkan 1.4 features/promotions — Vulkan Documentation Project](https://docs.vulkan.org/features/latest/features/proposals/VK_VERSION_1_4.html)
- [You Can Use Vulkan Without Pipelines Today — Khronos Blog](https://www.khronos.org/blog/you-can-use-vulkan-without-pipelines-today)
- [Shader Object sample README — Vulkan Documentation Project](https://docs.vulkan.org/samples/latest/samples/extensions/shader_object/README.html)
- [VK_LAYER_KHRONOS_shader_object documentation — Vulkan-ExtensionLayer](https://github.com/KhronosGroup/Vulkan-ExtensionLayer/blob/main/docs/shader_object_layer.md)
- [RADV Vulkan Driver Enables EXT_shader_object By Default With Mesa 24.1 — Phoronix](https://www.phoronix.com/news/RADV-Default-ESO-Support)
- [NVK Vulkan Driver Lands Shader Object & Graphics Pipeline Library — Phoronix](https://www.phoronix.com/news/NVK-ESO-GPL-Support)
- [Intel's Vulkan Linux Driver Finally Exposes VK_EXT_shader_object — Phoronix](https://www.phoronix.com/news/Intel-ANV-VK_EXT_shader_object)
- [Mesa 24.1 brings new hardware support for Arm and NVIDIA GPUs — Collabora](https://www.collabora.com/news-and-blog/news-and-events/mesa-241-brings-new-hardware-support-for-arm-and-nvidia-gpus.html)
- [Vulkan 1.3.246 Released With VK_EXT_shader_object — Phoronix](https://www.phoronix.com/news/Vulkan-1.3.246-VK-shader-object)
- [RADV CI: enable RADV_PERFTEST=shader_object for vkcts-polaris10-valve — Mesa GitLab MR !27139](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/27139)
- [Mesa src/amd/vulkan/radv_shader_object.c — gitlab.freedesktop.org/mesa/mesa](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/vulkan/radv_shader_object.c)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
