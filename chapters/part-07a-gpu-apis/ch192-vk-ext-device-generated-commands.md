# Chapter 192: GPU-Generated Commands — `VK_EXT_device_generated_commands` and Work Graphs

**Part VII — Application APIs & Middleware**

**Audiences**: Graphics application developers and systems developers. Application developers learn a new dispatch model that eliminates the last CPU bottleneck in large-scale GPU-driven scenes. Driver developers get an annotated walk through RADV's DGC compute shader and scratch buffer layout. Gaming layer developers see how VKD3D-Proton uses DGC to implement D3D12 `ExecuteIndirect` and emulate Work Graphs.

---

## Table of Contents

1. [Why the Command Buffer Model Stalls at Scale](#1-why-the-command-buffer-model-stalls-at-scale)
2. [Extension Fundamentals — Objects and Lifecycle](#2-extension-fundamentals--objects-and-lifecycle)
   - [VkIndirectCommandsLayoutEXT — The Token Sequence](#21-vkindirectcommandslayoutext--the-token-sequence)
   - [VkIndirectExecutionSetEXT — GPU-Visible Pipeline Array](#22-vkindirectexecutionsetext--gpu-visible-pipeline-array)
   - [Scratch Memory Requirements](#23-scratch-memory-requirements)
3. [Token Type Reference](#3-token-type-reference)
4. [Preprocess and Execute — The Two-Phase Model](#4-preprocess-and-execute--the-two-phase-model)
   - [Inline Execution](#41-inline-execution)
   - [Explicit Preprocess with Async Overlap](#42-explicit-preprocess-with-async-overlap)
5. [Complete Worked Example — GPU-Driven Multi-Material Scene](#5-complete-worked-example--gpu-driven-multi-material-scene)
6. [RADV Implementation Internals](#6-radv-implementation-internals)
   - [The DGC Prepare Compute Shader](#61-the-dgc-prepare-compute-shader)
   - [Scratch Buffer Layout](#62-scratch-buffer-layout)
   - [IB Chaining for Large Sequence Counts](#63-ib-chaining-for-large-sequence-counts)
7. [VKD3D-Proton: ExecuteIndirect and Work Graphs](#7-vkd3d-proton-executeindirect-and-work-graphs)
   - [ExecuteIndirect Batching](#71-executeindirect-batching)
   - [D3D12 Work Graphs Emulation](#72-d3d12-work-graphs-emulation)
8. [VK_AMDX_shader_enqueue and the Road to VK_KHR_work_graphs](#8-vk_amdx_shader_enqueue-and-the-road-to-vk_khr_work_graphs)
9. [Performance Guide](#9-performance-guide)
10. [Integrations](#integrations)
11. [References](#references)

---

## 1. Why the Command Buffer Model Stalls at Scale

Vulkan's command buffer model is deliberately CPU-authoritative. Each frame, the application records draw calls, binds descriptor sets, changes pipelines, and submits. For small scenes this is fine. For large GPU-driven scenes — 100,000 mesh instances, 2,000 materials, GPU-side occlusion culling — it breaks down in two places.

**The indirect draw ceiling.** `vkCmdDrawIndexedIndirectCount` lets the GPU write draw arguments and a count into a buffer, then the CPU submits one call that dispatches however many draws the GPU decided to emit. But it dispatches all draws with the same pipeline, the same descriptor binding, and the same push constants. If material 0 needs pipeline A and material 1 needs pipeline B, you need either two `vkCmdDrawIndexedIndirectCount` calls (one per material batch) or you bake all variation into a single uber-shader. Neither scales: the first requires CPU-driven sorting by material; the second creates a shader so large that register pressure and branch divergence eat back the gains.

**The GPU can't change state.** Nothing in the indirect draw API allows the GPU to switch pipelines, rebind vertex buffers, update push constants, or change index buffers between individual draws. The CPU must record all that before submission. This means any decision that requires knowing which objects survived culling — "these survived, they need this pipeline; those survived, they need that one" — must be deferred to the CPU after readback, adding a pipeline stall.

`VK_EXT_device_generated_commands` (DGC) resolves both problems. It lets a GPU compute shader write a complete command sequence — pipeline switch, index buffer bind, vertex buffer bind, push constant update, then draw — and then submit that sequence for execution without CPU intervention. The GPU fully drives the draw loop.

The extension replaces the earlier NVIDIA-only `VK_NV_device_generated_commands`, broadening support to AMD (RADV), Intel (ANV), Qualcomm (Turnip), Nouveau (NVK), and software (Lavapipe). All five shipped conformant support by Mesa 24.3. [Source — Phoronix: RADV DGC merge](https://www.phoronix.com/news/RADV-VK-EXT-DGC)

---

## 2. Extension Fundamentals — Objects and Lifecycle

Enable the extension and feature at device creation:

```c
VkPhysicalDeviceDeviceGeneratedCommandsFeaturesEXT dgc_features = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_DEVICE_GENERATED_COMMANDS_FEATURES_EXT,
    .deviceGeneratedCommands        = VK_TRUE,
    .dynamicGeneratedPipelineLayout = VK_FALSE, /* optional: VK_NULL_HANDLE layouts */
};
VkDeviceCreateInfo ci = {
    .sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO,
    .pNext = &dgc_features,
    /* ... */
};
```

Query limits before designing your token sequences:

```c
VkPhysicalDeviceDeviceGeneratedCommandsPropertiesEXT props = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_DEVICE_GENERATED_COMMANDS_PROPERTIES_EXT,
};
VkPhysicalDeviceProperties2 props2 = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_PROPERTIES_2,
    .pNext = &props,
};
vkGetPhysicalDeviceProperties2(phys_dev, &props2);
/* Key limits:
   props.maxIndirectPipelineCount         — slots in an IES (pipelines mode)
   props.maxIndirectShaderObjectCount     — slots in an IES (shader objects mode)
   props.maxIndirectSequenceCount         — sequences per vkCmdExecuteGeneratedCommandsEXT
   props.maxIndirectCommandsTokenCount    — tokens per VkIndirectCommandsLayoutEXT
   props.maxIndirectCommandsTokenOffset   — byte offset for any token in a sequence record
   props.maxIndirectCommandsIndirectStride — max byte stride between sequence records
*/
```

[Source — Vulkan spec: VkPhysicalDeviceDeviceGeneratedCommandsPropertiesEXT](https://registry.khronos.org/vulkan/specs/latest/man/html/VkPhysicalDeviceDeviceGeneratedCommandsPropertiesEXT.html)

### 2.1 VkIndirectCommandsLayoutEXT — The Token Sequence

A `VkIndirectCommandsLayoutEXT` describes the structure of a single *sequence record* — the fixed-stride blob that the GPU reads once per draw. Each record packs a sequence of typed tokens at declared byte offsets.

```c
/* Token 0: select which pipeline to use for this draw */
VkIndirectCommandsExecutionSetTokenEXT exec_set_token = {
    .type         = VK_INDIRECT_EXECUTION_SET_INFO_TYPE_PIPELINES_EXT,
    .shaderStages = VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT,
};
/* Token 1: update push constants (8 bytes: mesh index + material index) */
VkIndirectCommandsPushConstantTokenEXT pc_token = {
    .updateRange = { .stageFlags = VK_SHADER_STAGE_VERTEX_BIT |
                                   VK_SHADER_STAGE_FRAGMENT_BIT,
                     .offset = 0, .size = 8 },
};
/* Token 2: bind index buffer */
VkIndirectCommandsIndexBufferTokenEXT ib_token = {
    .mode = VK_INDIRECT_COMMANDS_INPUT_MODE_VULKAN_INDEX_BUFFER_EXT,
};

VkIndirectCommandsLayoutTokenEXT tokens[3] = {
    {
        .sType  = VK_STRUCTURE_TYPE_INDIRECT_COMMANDS_LAYOUT_TOKEN_EXT,
        .type   = VK_INDIRECT_COMMANDS_TOKEN_TYPE_EXECUTION_SET_EXT,
        .data   = { .pExecutionSet = &exec_set_token },
        .offset = 0,   /* bytes into the sequence record */
    },
    {
        .sType  = VK_STRUCTURE_TYPE_INDIRECT_COMMANDS_LAYOUT_TOKEN_EXT,
        .type   = VK_INDIRECT_COMMANDS_TOKEN_TYPE_PUSH_CONSTANT_EXT,
        .data   = { .pPushConstant = &pc_token },
        .offset = 4,   /* after the 4-byte pipeline index */
    },
    {
        .sType  = VK_STRUCTURE_TYPE_INDIRECT_COMMANDS_LAYOUT_TOKEN_EXT,
        .type   = VK_INDIRECT_COMMANDS_TOKEN_TYPE_INDEX_BUFFER_EXT,
        .data   = { .pIndexBuffer = &ib_token },
        .offset = 12,  /* after pipeline index (4) + push constants (8) */
    },
    /* The draw token follows at offset 28 (12 + VkBindIndexBufferIndirectCommandEXT = 16 bytes) */
};

VkIndirectCommandsLayoutTokenEXT draw_token = {
    .sType  = VK_STRUCTURE_TYPE_INDIRECT_COMMANDS_LAYOUT_TOKEN_EXT,
    .type   = VK_INDIRECT_COMMANDS_TOKEN_TYPE_DRAW_INDEXED_EXT,
    .offset = 28,
};
/* Extend tokens[] to 4 entries */

VkIndirectCommandsLayoutCreateInfoEXT layout_ci = {
    .sType          = VK_STRUCTURE_TYPE_INDIRECT_COMMANDS_LAYOUT_CREATE_INFO_EXT,
    .flags          = VK_INDIRECT_COMMANDS_LAYOUT_USAGE_EXPLICIT_PREPROCESS_BIT_EXT,
    .shaderStages   = VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT,
    .indirectStride = 48,    /* total record size, must be a multiple of 4 */
    .pipelineLayout = pipeline_layout,
    .tokenCount     = 4,
    .pTokens        = tokens,
};
VkIndirectCommandsLayoutEXT layout;
vkCreateIndirectCommandsLayoutEXT(device, &layout_ci, NULL, &layout);
```

**Rules:**
- The last token in `pTokens` must be a *dispatch work* token: `DRAW_INDEXED_EXT`, `DRAW_EXT`, `DRAW_INDEXED_COUNT_EXT`, `DRAW_COUNT_EXT`, `DRAW_MESH_TASKS_EXT`, `DISPATCH_EXT`, or `TRACE_RAYS2_EXT`. Exactly one dispatch token is permitted.
- Token offsets must be 4-byte aligned and non-overlapping.
- `indirectStride` must accommodate all tokens; RADV pads to IB packet alignment internally.
- `pipelineLayout` may be `VK_NULL_HANDLE` when `dynamicGeneratedPipelineLayout` is enabled, but then each sequence record must carry enough binding state for the driver to infer it.

[Source — Vulkan spec: VkIndirectCommandsLayoutCreateInfoEXT](https://registry.khronos.org/vulkan/specs/latest/man/html/VkIndirectCommandsLayoutCreateInfoEXT.html)

### 2.2 VkIndirectExecutionSetEXT — GPU-Visible Pipeline Array

A `VkIndirectExecutionSetEXT` (IES) is a device-side array of pipelines or shader objects. The GPU selects from it at runtime using a 32-bit index written into the sequence record. All entries in the array must share the same `VkPipelineLayout` (for the pipeline variant) or the same push-constant ranges and per-stage descriptor set layouts (for the shader-object variant).

**Pipeline-mode IES** (most common for graphics):

```c
/* Create the initial pipeline — must populate slot 0 */
VkPipeline initial_pipeline = create_pbr_pipeline(device, render_pass, 0);

VkIndirectExecutionSetPipelineInfoEXT ies_pipeline_info = {
    .sType            = VK_STRUCTURE_TYPE_INDIRECT_EXECUTION_SET_PIPELINE_INFO_EXT,
    .initialPipeline  = initial_pipeline,
    .maxPipelineCount = 128,  /* reserve 128 material pipeline slots */
};
VkIndirectExecutionSetCreateInfoEXT ies_ci = {
    .sType = VK_STRUCTURE_TYPE_INDIRECT_EXECUTION_SET_CREATE_INFO_EXT,
    .type  = VK_INDIRECT_EXECUTION_SET_INFO_TYPE_PIPELINES_EXT,
    .info  = { .pPipelineInfo = &ies_pipeline_info },
};
VkIndirectExecutionSetEXT ies;
vkCreateIndirectExecutionSetEXT(device, &ies_ci, NULL, &ies);

/* Populate remaining slots */
VkWriteIndirectExecutionSetPipelineEXT writes[127];
for (uint32_t i = 0; i < 127; i++) {
    writes[i] = (VkWriteIndirectExecutionSetPipelineEXT){
        .sType    = VK_STRUCTURE_TYPE_WRITE_INDIRECT_EXECUTION_SET_PIPELINE_EXT,
        .index    = i + 1,
        .pipeline = create_material_pipeline(device, render_pass, i + 1),
    };
}
vkUpdateIndirectExecutionSetPipelineEXT(device, ies, 127, writes);
```

**Shader-object mode IES** (for use with `VK_EXT_shader_object`):

```c
VkShaderEXT initial_shaders[2] = { vert_shader, frag_shader };
VkIndirectExecutionSetShaderLayoutInfoEXT layout_infos[2] = {
    { .sType = VK_STRUCTURE_TYPE_INDIRECT_EXECUTION_SET_SHADER_LAYOUT_INFO_EXT,
      .setLayoutCount = 1, .pSetLayouts = &desc_set_layout },
    { .sType = VK_STRUCTURE_TYPE_INDIRECT_EXECUTION_SET_SHADER_LAYOUT_INFO_EXT,
      .setLayoutCount = 1, .pSetLayouts = &desc_set_layout },
};
VkIndirectExecutionSetShaderInfoEXT ies_shader_info = {
    .sType                 = VK_STRUCTURE_TYPE_INDIRECT_EXECUTION_SET_SHADER_INFO_EXT,
    .shaderCount           = 2,
    .pInitialShaders       = initial_shaders,
    .pSetLayoutInfos       = layout_infos,
    .maxShaderCount        = 256,
    .pushConstantRangeCount = 1,
    .pPushConstantRanges   = &pc_range,
};
```

IES contents can be updated at any time with `vkUpdateIndirectExecutionSetPipelineEXT` / `vkUpdateIndirectExecutionSetShaderEXT`. Updates are not recorded into command buffers — they take effect immediately on the host side and must be followed by appropriate pipeline barriers before execution.

[Source — Vulkan spec: VkIndirectExecutionSetEXT](https://registry.khronos.org/vulkan/specs/latest/man/html/vkCreateIndirectExecutionSetEXT.html)

### 2.3 Scratch Memory Requirements

The driver may need a *preprocess buffer* in device memory to translate the application's sequence records into hardware-specific command packets. Query its size before allocating:

```c
VkGeneratedCommandsMemoryRequirementsInfoEXT req_info = {
    .sType                    = VK_STRUCTURE_TYPE_GENERATED_COMMANDS_MEMORY_REQUIREMENTS_INFO_EXT,
    .pNext                    = &(VkGeneratedCommandsPipelineInfoEXT){
        .sType    = VK_STRUCTURE_TYPE_GENERATED_COMMANDS_PIPELINE_INFO_EXT,
        .pipeline = initial_pipeline,
    },
    .indirectExecutionSet     = ies,
    .indirectCommandsLayout   = layout,
    .maxSequenceCount         = MAX_DRAWS,
    .maxDrawCount             = 0,  /* used only for DRAW_MESH_TASKS_COUNT_EXT */
};
VkMemoryRequirements2 mem_req = {
    .sType = VK_STRUCTURE_TYPE_MEMORY_REQUIREMENTS_2,
};
vkGetGeneratedCommandsMemoryRequirementsEXT(device, &req_info, &mem_req);

/* Allocate with VK_BUFFER_USAGE_2_PREPROCESS_BIT_EXT */
VkBufferCreateInfo buf_ci = {
    .sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO,
    .size  = mem_req.memoryRequirements.size,
    .usage = VK_BUFFER_USAGE_2_PREPROCESS_BIT_EXT |
             VK_BUFFER_USAGE_SHADER_DEVICE_ADDRESS_BIT,
};
/* ... allocate & bind ... */
```

On RADV, this buffer must reside in memory types within `pdev->memory_types_32bit` — AMD IB packets require 32-bit-addressable command memory. The size returned scales with `maxSequenceCount` multiplied by the per-sequence GFX and ACE command strides, plus upload area for push-constant data.

---

## 3. Token Type Reference

Every token corresponds to a specific GPU operation within a single sequence. The driver translates the token stream into hardware-specific command packets during preprocessing.

| Token type | Struct written by GPU compute shader | Effect at execute time |
|---|---|---|
| `EXECUTION_SET_EXT` | `uint32_t index` | Binds pipeline or shader object at that IES index |
| `PUSH_CONSTANT_EXT` | Raw bytes matching `updateRange.size` | Updates push constants for specified stages |
| `SEQUENCE_INDEX_EXT` | *(no data)* | Driver writes the zero-based sequence index here; shader reads it via `gl_DrawID` equivalent |
| `INDEX_BUFFER_EXT` | `VkBindIndexBufferIndirectCommandEXT {VkDeviceAddress, uint32_t size, VkIndexType}` | Binds index buffer |
| `VERTEX_BUFFER_EXT` | `VkBindVertexBufferIndirectCommandEXT {VkDeviceAddress, uint32_t size, uint32_t stride}` | Binds vertex buffer for specified binding slot |
| `DRAW_INDEXED_EXT` | `VkDrawIndexedIndirectCommand {indexCount, instanceCount, firstIndex, vertexOffset, firstInstance}` | Issues indexed draw |
| `DRAW_EXT` | `VkDrawIndirectCommand {vertexCount, instanceCount, firstVertex, firstInstance}` | Issues non-indexed draw |
| `DRAW_INDEXED_COUNT_EXT` | `VkDeviceAddress countAddr, VkDrawIndexedIndirectCommand[]` | GPU-side count + indexed draws |
| `DRAW_COUNT_EXT` | `VkDeviceAddress countAddr, VkDrawIndirectCommand[]` | GPU-side count + draws |
| `DISPATCH_EXT` | `VkDispatchIndirectCommand {x, y, z}` | Compute dispatch |
| `DRAW_MESH_TASKS_EXT` | `VkDrawMeshTasksIndirectCommandEXT {groupCountX, groupCountY, groupCountZ}` | Mesh + task shader dispatch |
| `DRAW_MESH_TASKS_COUNT_EXT` | `VkDeviceAddress countAddr, VkDrawMeshTasksIndirectCommandEXT[]` | GPU-side count + mesh dispatch |
| `TRACE_RAYS2_EXT` | `VkTraceRaysIndirectCommand2KHR` | Ray tracing dispatch |

The DXGI index buffer mode (`VK_INDIRECT_COMMANDS_INPUT_MODE_DXGI_INDEX_BUFFER_EXT`) encodes the index buffer as a `D3D12_INDEX_BUFFER_VIEW`-compatible structure, allowing VKD3D-Proton to write D3D12 index buffer views directly without reformatting.

---

## 4. Preprocess and Execute — The Two-Phase Model

### 4.1 Inline Execution

The simplest path: pass `isPreprocessed = VK_FALSE`. The driver preprocesses and executes in a single call, synchronously on the GPU. Use this when preprocessing cost is small (few tokens, no IES switch) or when latency matters more than throughput.

```c
VkGeneratedCommandsInfoEXT gen_info = {
    .sType                   = VK_STRUCTURE_TYPE_GENERATED_COMMANDS_INFO_EXT,
    .pNext                   = &(VkGeneratedCommandsPipelineInfoEXT){
        .sType    = VK_STRUCTURE_TYPE_GENERATED_COMMANDS_PIPELINE_INFO_EXT,
        .pipeline = initial_pipeline,
    },
    .shaderStages             = VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT,
    .indirectExecutionSet     = ies,
    .indirectCommandsLayout   = layout,
    .indirectAddress          = sequence_buffer_bda,
    .indirectAddressSize      = MAX_DRAWS * 48,  /* stride × count */
    .preprocessAddress        = preprocess_buffer_bda,
    .preprocessSize           = preprocess_size,
    .maxSequenceCount         = MAX_DRAWS,
    .sequenceCountAddress     = count_buffer_bda, /* GPU-written actual count */
    .maxDrawCount             = 0,
};

vkCmdExecuteGeneratedCommandsEXT(cmd, VK_FALSE, &gen_info);
```

The pipeline that will be active at execute time must be bound before this call, even when an IES is used — the driver uses the bound pipeline to validate token compatibility.

### 4.2 Explicit Preprocess with Async Overlap

Use `VK_INDIRECT_COMMANDS_LAYOUT_USAGE_EXPLICIT_PREPROCESS_BIT_EXT` on the layout to separate preprocessing and execution into distinct command buffers. This unlocks the primary performance benefit: run preprocessing on an async compute queue while the graphics queue finishes the previous frame's rendering.

```
Frame N timeline:
  Graphics queue: [render frame N-1] → [execute DGC (isPreprocessed=true)]
  Compute queue:  [GPU culling → write sequence records] → [preprocess DGC for frame N+1]
```

```c
/* === Compute command buffer (async compute queue) === */
/* Step 1: Run GPU culling compute shader → fills sequence_buffer */
vkCmdDispatch(compute_cmd, cull_groups_x, 1, 1);

/* Barrier: SHADER_WRITE → INDIRECT_COMMAND_READ */
VkMemoryBarrier2 cull_barrier = {
    .sType         = VK_STRUCTURE_TYPE_MEMORY_BARRIER_2,
    .srcStageMask  = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
    .srcAccessMask = VK_ACCESS_2_SHADER_WRITE_BIT,
    .dstStageMask  = VK_PIPELINE_STAGE_2_COMMAND_PREPROCESS_BIT_EXT,
    .dstAccessMask = VK_ACCESS_2_COMMAND_PREPROCESS_READ_BIT_EXT,
};
/* ... vkCmdPipelineBarrier2 ... */

/* Step 2: Preprocess — translates sequence_buffer into preprocess_buffer */
vkCmdPreprocessGeneratedCommandsEXT(compute_cmd, &gen_info,
                                     state_cmd  /* records the state active at execute */);
/* Signal timeline semaphore when preprocess is done */

/* === Graphics command buffer === */
/* Wait on the semaphore from above */
/* Barrier: COMMAND_PREPROCESS_WRITE → INDIRECT_COMMAND_READ */
VkMemoryBarrier2 exec_barrier = {
    .srcStageMask  = VK_PIPELINE_STAGE_2_COMMAND_PREPROCESS_BIT_EXT,
    .srcAccessMask = VK_ACCESS_2_COMMAND_PREPROCESS_WRITE_BIT_EXT,
    .dstStageMask  = VK_PIPELINE_STAGE_2_DRAW_INDIRECT_BIT,
    .dstAccessMask = VK_ACCESS_2_INDIRECT_COMMAND_READ_BIT,
};
vkCmdExecuteGeneratedCommandsEXT(gfx_cmd, VK_TRUE, &gen_info);
```

The `stateCommandBuffer` argument to `vkCmdPreprocessGeneratedCommandsEXT` is new in the EXT variant vs the NV predecessor. It must record the exact pipeline, descriptor set, and push constant state that will be active at execute time. This allows the preprocessing shader to patch absolute hardware addresses into the command stream without guessing what state will be current.

---

## 5. Complete Worked Example — GPU-Driven Multi-Material Scene

This example drives 64,000 mesh instances with 128 distinct material pipelines entirely from the GPU. No CPU readback, no per-material draw call sorting.

**Host-side setup (once per PSO change):**

```cpp
// 1. Allocate scene buffers
VkBuffer instance_buffer;   // 64000 × InstanceData (mesh BDA, material index, transform)
VkBuffer sequence_buffer;   // 64000 × 48 bytes (one DGC sequence record per instance)
VkBuffer count_buffer;      // uint32_t GPU-written visible count
VkBuffer preprocess_buffer; // sized by vkGetGeneratedCommandsMemoryRequirementsEXT

// 2. Build IES with 128 material pipelines
VkIndirectExecutionSetEXT ies = build_material_ies(device, 128, base_pipeline_layout);

// 3. Build indirect commands layout: [exec_set | push_consts | index_buf | draw_indexed]
VkIndirectCommandsLayoutEXT layout = build_dgc_layout(device, base_pipeline_layout);
```

**Per-frame GPU compute shader** (fills `sequence_buffer`):

```glsl
// cull_and_build_sequences.comp
layout(local_size_x = 64) in;

struct SequenceRecord {
    uint  pipeline_index;   // offset 0  → EXECUTION_SET token
    uint  mesh_index;       // offset 4  → push constant [0]
    uint  material_index;   // offset 8  → push constant [1]
    // pad to 12
    VkBindIndexBufferIndirectCommandEXT ib;  // offset 12 (VkDeviceAddress[2] + indexType)
    VkDrawIndexedIndirectCommand draw;        // offset 28
    // 5 × uint32 = 20 bytes → total = 48 bytes
};

layout(binding = 0) buffer Instances { InstanceData instances[]; };
layout(binding = 1) buffer Sequences { SequenceRecord records[]; };
layout(binding = 2) buffer Count     { uint visible_count; };

void main() {
    uint id = gl_GlobalInvocationID.x;
    if (id >= total_instance_count) return;

    InstanceData inst = instances[id];
    if (!frustum_cull(inst.bounds)) return;   // skip culled instances

    uint seq = atomicAdd(visible_count, 1);
    records[seq].pipeline_index = inst.material_pipeline_index;
    records[seq].mesh_index     = inst.mesh_index;
    records[seq].material_index = inst.material_index;
    records[seq].ib = VkBindIndexBufferIndirectCommandEXT{
        .bufferAddress = inst.index_buffer_bda,
        .size          = inst.index_buffer_bytes,
        .indexType     = VK_INDEX_TYPE_UINT32,
    };
    records[seq].draw = VkDrawIndexedIndirectCommand{
        .indexCount    = inst.index_count,
        .instanceCount = 1,
        .firstIndex    = 0,
        .vertexOffset  = 0,
        .firstInstance = seq,  // used to index per-instance data in vertex shader
    };
}
```

**Slang equivalent** — A shared `dgc_types` module gives both the cull shader and any future DGC consumers a single `SequenceRecord` definition with named fields instead of raw SSBO offset arithmetic, while `[vk::constant_id]` specialisation constants produce a zero-cost frustum-skip variant (e.g. for a depth prepass) without a preprocessor macro.

```slang
// File: dgc_types.slang
// Shared DGC payload types — import from the cull shader and any future DGC-consuming shaders
// Slang improvements:
//   - Named struct fields replace raw per-binding offset arithmetic; a single definition
//     serves both producer (cull) and consumer (vertex) shaders without duplication
//   - uint-pair address avoids 8-byte alignment padding at offset 12, keeping the
//     48-byte stride exact (matches indirectStride in VkIndirectCommandsLayoutCreateInfoEXT)
//   - [vk::constant_id] specialisation constants bake variant flags at pipeline-compile time

module dgc_types;

// Mirror of VkBindIndexBufferIndirectCommandEXT packed as four uint32s.
// Using two uints for the 64-bit device address avoids alignment padding at offset 12
// (a uint64_t would require 8-byte alignment, inserting 4 bytes of implicit padding).
struct BindIndexBufferCmd {
    uint bufferAddrLo;  // low 32 bits of VkDeviceAddress
    uint bufferAddrHi;  // high 32 bits of VkDeviceAddress
    uint size;          // byte size of index buffer region
    uint indexType;     // 0 = VK_INDEX_TYPE_UINT16, 1 = VK_INDEX_TYPE_UINT32
};                      // 16 bytes; sits at token offset 12 with no padding

// Mirror of VkDrawIndexedIndirectCommand (20 bytes)
struct DrawIndexedCmd {
    uint indexCount;
    uint instanceCount;
    uint firstIndex;
    int  vertexOffset;
    uint firstInstance;
};

// 48-byte sequence record matching VkIndirectCommandsLayoutCreateInfoEXT token offsets:
//   offset  0: pipeline_index  (EXECUTION_SET_EXT token)
//   offset  4: mesh_index      (PUSH_CONSTANT_EXT token [0])
//   offset  8: material_index  (PUSH_CONSTANT_EXT token [1])
//   offset 12: ib              (INDEX_BUFFER_EXT  token, 16 bytes)
//   offset 28: draw            (DRAW_INDEXED_EXT  token, 20 bytes)
//   4 + 4 + 4 + 16 + 20 = 48 bytes
struct SequenceRecord {
    uint               pipeline_index;
    uint               mesh_index;
    uint               material_index;
    BindIndexBufferCmd ib;
    DrawIndexedCmd     draw;
};

struct InstanceData {
    uint64_t index_buffer_bda;        // Vulkan device address of this mesh's index buffer
    uint     index_buffer_bytes;
    uint     index_count;
    uint     mesh_index;
    uint     material_index;
    uint     material_pipeline_index;
    float4   bounds;                  // xyz = sphere centre, w = sphere radius
};

bool frustum_cull(float4 bounds);     // declared here; defined in scene_geometry module

// ---- File: cull_and_build_sequences.slang ----

import dgc_types;  // SequenceRecord, InstanceData, frustum_cull

// Specialisation constants: baked at pipeline-compile time, not loaded as uniforms at runtime.
// TOTAL_INSTANCE_COUNT avoids an extra SGPR/UBO fetch inside the hot loop.
[vk::constant_id(0)] const uint TOTAL_INSTANCE_COUNT = 65536;
// ENABLE_FRUSTUM_CULL = false compiles a separate depth-prepass variant where the compiler
// eliminates the cull branch entirely — no runtime overhead, no preprocessor required.
[vk::constant_id(1)] const bool ENABLE_FRUSTUM_CULL  = true;

[[vk::binding(0, 0)]] StructuredBuffer<InstanceData>     instances;
[[vk::binding(1, 0)]] RWStructuredBuffer<SequenceRecord> records;
[[vk::binding(2, 0)]] RWStructuredBuffer<uint>           visible_count;

[shader("compute")]
[numthreads(64, 1, 1)]
void main(uint3 dispatchThreadID : SV_DispatchThreadID)
{
    uint id = dispatchThreadID.x;
    if (id >= TOTAL_INSTANCE_COUNT) return;

    InstanceData inst = instances[id];
    if (ENABLE_FRUSTUM_CULL && !frustum_cull(inst.bounds)) return;

    uint seq;
    InterlockedAdd(visible_count[0], 1u, seq);

    // Construct SequenceRecord with named fields — misaligned or missing assignments
    // are compile errors rather than silent silent stride mismatches at runtime.
    SequenceRecord rec;
    rec.pipeline_index     = inst.material_pipeline_index;
    rec.mesh_index         = inst.mesh_index;
    rec.material_index     = inst.material_index;
    // Split 64-bit BDA into two uint32s matching BindIndexBufferCmd layout at offset 12
    rec.ib.bufferAddrLo    = uint(inst.index_buffer_bda & 0xFFFFFFFFu);
    rec.ib.bufferAddrHi    = uint(inst.index_buffer_bda >> 32);
    rec.ib.size            = inst.index_buffer_bytes;
    rec.ib.indexType       = 1u;  // VK_INDEX_TYPE_UINT32
    rec.draw.indexCount    = inst.index_count;
    rec.draw.instanceCount = 1u;
    rec.draw.firstIndex    = 0u;
    rec.draw.vertexOffset  = 0;
    rec.draw.firstInstance = seq;
    records[seq] = rec;
}
```

**Graphics command buffer recording:**

```c
/* Bind the initial pipeline (required even with IES) */
vkCmdBindPipeline(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS, base_pipeline);
vkCmdBindDescriptorSets(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS, layout, 0, 1, &scene_set, 0, NULL);

/* Begin render pass ... */

/* GPU barrier: cull compute → DGC sequence read */

vkCmdExecuteGeneratedCommandsEXT(cmd, VK_FALSE, &gen_info);
/* This single call draws all visible instances, each with the correct pipeline,
   push constants, and index buffer — zero CPU involvement after submission. */
```

The vertex shader recovers per-instance data using `gl_InstanceIndex` (set to `firstInstance` from the draw record) to index into the instance buffer via a bindless SSBO.

---

## 6. RADV Implementation Internals

### 6.1 The DGC Prepare Compute Shader

RADV implements DGC preprocessing entirely as a JIT-compiled NIR compute shader rather than a fixed-function hardware mechanism. The relevant source is `src/amd/vulkan/radv_dgc.c` in the Mesa repository. [Source — Mesa GitLab](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/vulkan/radv_dgc.c)

The shader entry point is built at `radv_get_dgc_pipeline` (line 2921). Key characteristics:

```c
nir_builder b = radv_meta_nir_init_shader(MESA_SHADER_COMPUTE, "meta_dgc_prepare");
b.shader->info.workgroup_size[0] = 64;
/* One invocation per sequence. Invocation index = sequence index. */
```

The shader is compiled once per unique `VkIndirectCommandsLayoutEXT`, keyed via `RADV_META_OBJECT_KEY_DGC` and cached in the meta pipeline cache. Compilation happens at `vkCreateIndirectCommandsLayoutEXT` time, not at execute time.

The shader receives parameters via push constants in `struct radv_dgc_params`:
- Pointer to the application's sequence buffer (`indirectAddress`)
- Pointer to the preprocess output region
- Pointer to the sequence count address (if `sequenceCountAddress != 0`)
- Per-token offsets within the sequence record
- IES VA and stride (for `EXECUTION_SET_EXT` token processing)

For each enabled token, the shader reads the application's record, translates it into AMD GFX command packets (PM4 for graphics, CS packets for compute), and writes them into the preprocess buffer at the correct per-sequence offset.

**EXECUTION_SET token processing** involves reading the 32-bit IES index from the sequence record, computing `ies_va + index × stride` to reach the pipeline metadata entry in the IES BO, and then patching the SPI register programming (shader program address, user-data SGPRs, register counts) into the GFX command stream for that sequence.

### 6.2 Scratch Buffer Layout

The preprocess buffer is structured as contiguous sub-regions described by `struct dgc_cmdbuf_layout`:

```
┌──────────────────────────────────────────────────────────────┐
│ GFX trailer   (PKT3_INDIRECT_BUFFER target; 1 aligned block) │
├──────────────────────────────────────────────────────────────┤
│ GFX preamble  (optional double-jump IB; 1 aligned block)     │
├──────────────────────────────────────────────────────────────┤
│ GFX sequences: main_cmd_stride × maxSequenceCount            │
│  [seq 0: pipeline_bind + push_consts + ib_bind + draw]       │
│  [seq 1: ...]   ← IB-aligned stride                         │
│  ...                                                         │
├──────────────────────────────────────────────────────────────┤
│ ACE trailer   (only if mesh/task shaders involved)           │
├──────────────────────────────────────────────────────────────┤
│ ACE preamble  (optional)                                     │
├──────────────────────────────────────────────────────────────┤
│ ACE sequences: ace_cmd_stride × maxSequenceCount             │
├──────────────────────────────────────────────────────────────┤
│ Upload region: upload_stride × maxSequenceCount              │
│  (push constant staging data)                                │
└──────────────────────────────────────────────────────────────┘
```

`main_cmd_stride` is computed from the token set: each token type contributes a known number of PM4 dwords, rounded up to IB alignment (typically 4 bytes). The total buffer size returned by `vkGetGeneratedCommandsMemoryRequirementsEXT` equals `alloc_size` from this layout.

Memory placement constraint: the buffer must be in a memory type within `pdev->memory_types_32bit`. AMD's `PKT3_INDIRECT_BUFFER` packet carries a 28-bit base address shifted left by 4 — the command buffer must therefore lie within the low 4 GiB of the GPU address space.

### 6.3 IB Chaining for Large Sequence Counts

When `maxSequenceCount >= 65536`, RADV enables IB chaining: the prepare shader appends a `PKT3_INDIRECT_BUFFER` chain packet at the end of each sequence that jumps to the next sequence. This avoids the maximum IB length limit in AMD hardware (which caps a single IB submission to 65535 × 4-byte dwords).

The preamble optimization is engaged when `sequenceCountAddress != 0 && maxSequenceCount >= 64`. The GFX preamble is a two-hop `PKT3_INDIRECT_BUFFER` chain: the main command buffer jumps to the preamble, which reads the GPU-written actual sequence count and jumps to the appropriate point in the sequence stream, short-circuiting sequences beyond the actual count.

For mesh/task shaders, the preprocess buffer contains a parallel ACE (async compute engine) command stream. RADV submits both GFX and ACE IBs in tandem via the gang submit mechanism, synchronising at the task → mesh boundary using GDS (global data share) atomics as specified by the GFX11 architecture.

---

## 7. VKD3D-Proton: ExecuteIndirect and Work Graphs

### 7.1 ExecuteIndirect Batching

D3D12's `ExecuteIndirect` is the direct analogue of DGC — it executes a GPU-generated sequence of draws or dispatches from an `ID3D12CommandSignature`. VKD3D-Proton originally emulated it using `VK_NV_device_generated_commands` (from version 2.10), then migrated to `VK_EXT_device_generated_commands` as Mesa drivers gained EXT support.

Four execution modes are distinguished internally (`libs/vkd3d/command.c`):

```c
typedef enum {
    VKD3D_DGC_MODE_APPLICATION_CALL,       /* single call, preprocessing inline */
    VKD3D_DGC_MODE_PREPROCESS_ONLY,        /* preprocessing pass, no execute */
    VKD3D_DGC_MODE_EXECUTE_ONLY,           /* execute against already-preprocessed buffer */
    VKD3D_DGC_MODE_PREPROCESS_AND_EXECUTE, /* combined */
} vkd3d_dgc_mode;
```

Version 3.0.1 introduced a batching system that coalesces multiple `ExecuteIndirect` calls from within a D3D12 command list into a single DGC dispatch, amortising preprocessing overhead. Benchmarks reported by Hans-Kristian Arntzen showed:

- **Halo Infinite**: 15% frame-time improvement on RADV (DGC preprocessing amortised across the frame's ExecuteIndirect-heavy scene phases)
- **Starfield**: 1–2% improvement (less ExecuteIndirect dependent)
- **Crimson Desert**: benefited from v3.0.1 batched path

The DXGI index buffer mode (`VK_INDIRECT_COMMANDS_INPUT_MODE_DXGI_INDEX_BUFFER_EXT`) is critical here: D3D12's `D3D12_INDEX_BUFFER_VIEW` uses `{GpuVirtualAddress, SizeInBytes, DXGI_FORMAT}` — identical in layout to Vulkan's extended index buffer indirect command. VKD3D-Proton can copy D3D12 index buffer views into DGC sequence records verbatim rather than reformatting. [Source — VKD3D-Proton CHANGELOG](https://github.com/HansKristian-Work/vkd3d-proton/blob/master/CHANGELOG.md)

### 7.2 D3D12 Work Graphs Emulation

D3D12 Work Graphs (API introduced in Agility SDK 1.611.3) enable GPU shader nodes to enqueue other nodes, building producer-consumer computation graphs without CPU involvement. They are the D3D12 equivalent of `VK_AMDX_shader_enqueue`. As of mid-2026 no shipped game title uses Work Graphs in production, but several upcoming titles are expected to.

VKD3D-Proton implements Work Graphs without `VK_AMDX_shader_enqueue`, instead using a level-by-level barrier-plus-dispatch model described in `libs/vkd3d/workgraphs.c`. [Source — VKD3D-Proton workgraphs.c](https://github.com/HansKristian-Work/vkd3d-proton/blob/master/libs/vkd3d/workgraphs.c)

The design document states:

> "Even when we eventually have native work graph support in Vulkan (aside from the AMDX experimental extension), we will need a plain emulated path."

Three reasons for avoiding persistent-thread approaches:

1. **Debugability**: breadcrumbs and RenderDoc step-through require normal `vkCmdDispatchIndirect` commands; persistent spinning shaders break capture tools.
2. **Forward-progress hazard**: a workgroup spinning on a semaphore waiting for another workgroup's output can deadlock with no yield mechanism in Vulkan.
3. **Occupancy destruction**: persistent threads occupying all SMs prevent other work from launching.

The emulation model:

```
Level 0: dispatch entry-point nodes → barrier
         → expander pass (prefix-sum allocation, setup indirect dispatches) → barrier
Level 1: dispatch level-1 nodes → barrier → expander → barrier
...
Level N (up to MAX_WORKGRAPH_LEVELS = 32)
```

Payload communication uses a global memory ring buffer (per queue family). The expander pass uses an atomic-append + prefix-sum two-phase allocation: (1) each level-N node increments a per-output-node counter atomically, (2) the expander prefix-sums the per-node counts to produce final payload buffer offsets, avoiding worst-case pre-allocation.

Key constants from `workgraphs.c`:
- `MAX_WORKGRAPH_LEVELS = 32`
- `MAX_AMPLIFICATION_RATE = 1024` (max records enqueued per thread)
- Scratch budget: ~256 MiB per work graph state object

VKD3D-Proton's release notes for v3.0 (November 2025) reported: "the performance of this emulation can massively outperform native driver implementations of the feature in many scenarios we've tested." For shallow graphs with regular workload shape — the typical case in current work-graph prototypes — the barrier-plus-dispatch approach has lower scheduling overhead than `VK_AMDX_shader_enqueue` persistent threads on current AMD hardware.

---

## 8. VK_AMDX_shader_enqueue and the Road to VK_KHR_work_graphs

`VK_AMDX_shader_enqueue` is AMD's provisional vendor extension (the `AMDX` prefix designates experimental, non-ratified extensions) that enables true GPU-native work graphs: compute and mesh shaders call SPIR-V intrinsics to enqueue payloads for other shader nodes at runtime, with data-dependent graph traversal.

**New pipeline bind point:**

```c
VK_PIPELINE_BIND_POINT_EXECUTION_GRAPH_AMDX
```

**Pipeline creation:**

```c
VkExecutionGraphPipelineCreateInfoAMDX eg_ci = {
    .sType      = VK_STRUCTURE_TYPE_EXECUTION_GRAPH_PIPELINE_CREATE_INFO_AMDX,
    .flags      = VK_PIPELINE_CREATE_2_EXECUTION_GRAPH_BIT_AMDX,
    .stageCount = node_count,
    .pStages    = node_stages,
    .pLibraries = NULL,
    .layout     = pipeline_layout,
};
VkPipeline exec_graph;
vkCreateExecutionGraphPipelinesAMDX(device, cache, 1, &eg_ci, NULL, &exec_graph);
```

**SPIR-V shader intrinsics** (new storage class and opcodes):

```spirv
; Allocate output payloads
%payloads = OpAllocateNodePayloadsAMDX %NodePayloadArrayAMDX %count %node_index

; Write payload data
OpStore %payloads[0].field %value

; Dispatch the receiving node
OpEnqueueNodePayloadsAMDX %payloads
```

**Dispatch:**

```c
vkGetExecutionGraphPipelineScratchSizeAMDX(device, exec_graph, &scratch_size);
vkCmdInitializeGraphScratchMemoryAMDX(cmd, exec_graph, scratch_bda, scratch_size);
/* Required before first dispatch; can be cached if pipeline unchanged */

VkDispatchGraphCountInfoAMDX count_info = { .count = 1 };
VkDispatchGraphInfoAMDX dispatch_info = {
    .nodeIndex = 0,
    .payloadCount = 1,
    .payloads = { .hostAddress = &entry_payload, .stride = sizeof(entry_payload) },
};
vkCmdDispatchGraphAMDX(cmd, scratch_bda, &count_info);
```

**How it differs from DGC:**

| Dimension | `VK_EXT_device_generated_commands` | `VK_AMDX_shader_enqueue` |
|---|---|---|
| Model | Tokenized command stream (GPU reads a fixed-format buffer) | Dynamic graph (shaders call enqueue intrinsics at runtime) |
| Node selection | Static IES index in sequence record | Runtime SPIR-V `OpEnqueueNodePayloadsAMDX` |
| Recursion | Not supported | Supported (up to `maxExecutionGraphDepth = 32`) |
| Data flow | Fixed token offsets in sequence record | Typed payload structs via `NodePayloadAMDX` storage class |
| Use case | GPU-driven multi-draw, material switching | Producer-consumer compute graphs, scene traversal |
| Driver support | RADV, ANV, NVK, Turnip, Lavapipe | AMD RDNA2+ only (RADV and AMDVLK) |
| Standardisation | Ratified multi-vendor EXT | Provisional AMD vendor extension |

**Standardisation path:** AMD has publicly stated intent to work toward a multi-vendor `VK_KHR_work_graphs` extension. As of July 2026, no formal `VK_KHR_work_graphs` extension exists in the Vulkan registry. The Khronos Vulkan Working Group is tracking this work; AMD shipped mesh node enqueue support as an addendum to `VK_AMDX_shader_enqueue` (available on GPUOpen). Microsoft's work on D3D12 Work Graphs and AMD's AMDX extension are informing the Khronos design, but the multi-vendor API surface is not yet settled. *Note: this is subject to change; consult the Vulkan roadmap blog for current status.* [Source — Vulkan AMDX proposal](https://docs.vulkan.org/features/latest/features/proposals/VK_AMDX_shader_enqueue.html)

---

## 9. Performance Guide

**When DGC wins:**

- **Deep material diversity**: 50+ distinct pipelines across a scene. Without DGC, material sorting is mandatory; with DGC, the GPU culling shader deposits per-draw pipeline indices and the IES selects them without CPU involvement.
- **Variable-count draw lists**: the GPU culling pass writes a count directly into `sequenceCountAddress`. The CPU submits one DGC execute call with `maxSequenceCount` as the upper bound; the actual count is determined entirely on the GPU.
- **Heavy `ExecuteIndirect` titles via VKD3D-Proton**: Halo Infinite's scene architecture generates thousands of `ExecuteIndirect` calls per frame; DGC batch coalescing in VKD3D-Proton produces measurable frame-time improvements (15% documented on RADV).
- **Preprocessing overlap**: when preprocess is explicit and runs on the async compute queue, it overlaps with the previous frame's rendering. This hides preprocessing latency entirely on GPU-bound workloads.

**When DGC does not win:**

- **Few unique pipelines (< 8–16)**: CPU-driven `vkCmdBindPipeline` + `vkCmdDrawIndexedIndirectCount` per-batch incurs less overhead than DGC preprocessing for small pipeline counts.
- **Stable draw lists (no culling)**: if the CPU already knows every draw's parameters before submission, traditional indirect draw is simpler and avoids the DGC scratch allocation.
- **Preprocessing cannot be overlapped** and the preprocessing shader itself is a bottleneck: on some workloads RADV's `meta_dgc_prepare` shader is significant enough to measure on the GPU timeline. Profile with `RADV_PERFTEST=dgc_pipe_stats` to observe it.
- **Compute-only workloads with dynamic dispatch count**: `vkCmdDispatchIndirect` with a GPU-written count buffer may be sufficient; the `DISPATCH_EXT` token in DGC provides no additional benefit unless combined with push constant or pipeline switching per dispatch.

**Profiling DGC:**

```bash
# Capture with RenderDoc (DGC calls are recorded as "ExecuteGeneratedCommands" events)
renderdoccmd capture --working-dir . -- ./my_app

# GPU timestamp queries around preprocess + execute
vkCmdWriteTimestamp2(cmd, VK_PIPELINE_STAGE_2_TOP_OF_PIPE_BIT, query_pool, 0);
vkCmdPreprocessGeneratedCommandsEXT(cmd, &gen_info, state_cmd);
vkCmdWriteTimestamp2(cmd, VK_PIPELINE_STAGE_2_BOTTOM_OF_PIPE_BIT, query_pool, 1);
/* ... */
vkCmdWriteTimestamp2(cmd, VK_PIPELINE_STAGE_2_TOP_OF_PIPE_BIT, query_pool, 2);
vkCmdExecuteGeneratedCommandsEXT(cmd, VK_TRUE, &gen_info);
vkCmdWriteTimestamp2(cmd, VK_PIPELINE_STAGE_2_BOTTOM_OF_PIPE_BIT, query_pool, 3);
```

The `COMMAND_PREPROCESS_BIT_EXT` pipeline stage allows correct barrier placement between preprocess and execute without over-synchronising to `TOP_OF_PIPE`.

---

## Integrations

- **Ch154 — GPU-Driven Rendering**: The indirect draw techniques in that chapter (`vkCmdDrawIndexedIndirectCount`, GPU frustum culling, GPU occlusion queries) motivate DGC. This chapter provides the mechanism that removes the remaining CPU bottleneck: per-draw pipeline switching. The GPU culling compute shader in Ch154 §4 is the producer; the DGC sequence buffer is the consumer.

- **Ch173 — `VK_EXT_shader_object`**: The `VkIndirectExecutionSetEXT` in shader-object mode (`VK_INDIRECT_EXECUTION_SET_INFO_TYPE_SHADER_OBJECTS_EXT`) combines DGC with pipeline-free shader binding. `vkCmdBindShadersEXT` and `vkUpdateIndirectExecutionSetShaderEXT` interact; shader objects bound before `vkCmdExecuteGeneratedCommandsEXT` must satisfy the same layout constraints as pipeline-mode IES entries.

- **Ch127 — Mesh Shaders and Variable-Rate Shading**: The `DRAW_MESH_TASKS_EXT` token enables GPU-generated mesh shader dispatches. RADV generates a parallel ACE command stream for the task shader phase alongside the GFX IB; the GDS-synchronised gang submit model is shared with Ch127's coverage of task→mesh handoff.

- **Ch157 — Vulkan Descriptor Binding**: `VK_EXT_descriptor_heap` and DGC are co-designed in the Khronos working group — descriptor heap access is bindless and does not require `VkPipelineLayout`, which aligns with DGC's `dynamicGeneratedPipelineLayout` feature. When descriptor heap ships broadly, DGC sequence records can carry raw descriptor indices rather than push constants, further simplifying the per-draw data structure.

- **Ch104 — DXVK and VKD3D-Proton**: VKD3D-Proton's `ExecuteIndirect` batching (§7.1) and Work Graphs emulation (§7.2) are primary production consumers of `VK_EXT_device_generated_commands`. Ch104's coverage of the D3D12 root signature model explains why the DXGI index buffer mode and `VK_INDIRECT_COMMANDS_INPUT_MODE_DXGI_INDEX_BUFFER_EXT` exist.

- **Ch143 — RADV Internals**: `radv_dgc.c` is a substantial component (~3,600 lines, Copyright © 2024 Valve Corporation). Ch143's coverage of RADV's meta pipeline cache and the NIR meta shader infrastructure explains how `build_dgc_prepare_shader` compiles and caches the preprocessing compute shader.

- **Ch135 — Vulkan Ray Tracing**: The `TRACE_RAYS2_EXT` token extends DGC to ray-tracing pipelines. GPU-driven ray tracing dispatch (varying SBT offsets and ray count per region) benefits from the same pattern as GPU-driven rasterisation.

- **Ch28 — Windows Compatibility Layer**: D3D12 Work Graphs support in VKD3D-Proton depends on DGC for the `ExecuteIndirect`-backed implementation path. The performance analysis (`core Vulkan compute outperforms native AMD AMDX shader_enqueue for current game-scale graphs`) is discussed in both chapters.

- **Ch31 — Conformance and Regression Testing**: `dEQP-VK.device_generated_commands.*` is the test group covering this extension in the Khronos CTS. Mesa CI runs this group against hardware runners when `src/amd/vulkan/radv_dgc.c` is modified.

---

## References

1. Vulkan spec — `VK_EXT_device_generated_commands`: [https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_device_generated_commands.html](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_device_generated_commands.html)
2. Extension proposal / design rationale: [https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_device_generated_commands.html](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_device_generated_commands.html)
3. RADV DGC implementation — `src/amd/vulkan/radv_dgc.c`: [https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/vulkan/radv_dgc.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/vulkan/radv_dgc.c)
4. RADV DGC merge announcement — Phoronix: [https://www.phoronix.com/news/RADV-VK-EXT-DGC](https://www.phoronix.com/news/RADV-VK-EXT-DGC)
5. VKD3D-Proton CHANGELOG (DGC batching, NV→EXT migration): [https://github.com/HansKristian-Work/vkd3d-proton/blob/master/CHANGELOG.md](https://github.com/HansKristian-Work/vkd3d-proton/blob/master/CHANGELOG.md)
6. VKD3D-Proton Work Graphs design doc: [https://github.com/HansKristian-Work/vkd3d-proton/blob/master/docs/workgraphs.md](https://github.com/HansKristian-Work/vkd3d-proton/blob/master/docs/workgraphs.md)
7. `VK_AMDX_shader_enqueue` proposal: [https://docs.vulkan.org/features/latest/features/proposals/VK_AMDX_shader_enqueue.html](https://docs.vulkan.org/features/latest/features/proposals/VK_AMDX_shader_enqueue.html)
8. Mike Blumenkrantz (Valve) on DGC — supergoodcode.com: [https://www.supergoodcode.com/device-generated-commands/](https://www.supergoodcode.com/device-generated-commands/)
9. Ricardo Garcia (Igalia) — Vulkanised 2025 talk: [https://vulkan.org/user/pages/09.events/vulkanised-2025/T10-Ricardo-Garcia-Igalia.pdf](https://vulkan.org/user/pages/09.events/vulkanised-2025/T10-Ricardo-Garcia-Igalia.pdf)
10. NVIDIA nvpro-samples — vk_device_generated_cmds: [https://github.com/nvpro-samples/vk_device_generated_cmds](https://github.com/nvpro-samples/vk_device_generated_cmds)
11. Vulkan-Headers `vulkan_core.h` (spec version 1): [https://github.com/KhronosGroup/Vulkan-Headers/blob/main/include/vulkan/vulkan_core.h](https://github.com/KhronosGroup/Vulkan-Headers/blob/main/include/vulkan/vulkan_core.h)
