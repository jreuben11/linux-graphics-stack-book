# Chapter 154: GPU-Driven Rendering: Indirect Draw, Meshlets, and GPU Culling

**Target audiences**: Graphics application developers implementing high-performance rendering with Vulkan; engine developers migrating from CPU-driven to GPU-driven architectures; and engineers studying the data structures behind mesh shaders, indirect dispatch, and GPU-side culling.

---

## Table of Contents

1. [Introduction](#introduction)
   - [1.1 What is GPU-Driven Rendering?](#11-what-is-gpu-driven-rendering)
   - [1.2 What is an Indirect Draw Command?](#12-what-is-an-indirect-draw-command)
   - [1.3 What is a Meshlet?](#13-what-is-a-meshlet)
   - [1.4 What is GPU Culling?](#14-what-is-gpu-culling)
2. [The CPU-GPU Bottleneck: Why GPU-Driven?](#the-cpu-gpu-bottleneck-why-gpu-driven)
3. [Indirect Draw Commands](#indirect-draw-commands)
4. [Multi-Draw Indirect and Draw Count](#multi-draw-indirect-and-draw-count)
5. [GPU Culling with Compute Shaders](#gpu-culling-with-compute-shaders)
6. [Meshlets and Mesh Shaders](#meshlets-and-mesh-shaders)
7. [Task and Mesh Shader Pipeline](#task-and-mesh-shader-pipeline)
8. [Persistent GPU Scene Representation](#persistent-gpu-scene-representation)
9. [Bindless Resources](#bindless-resources)
10. [Hierarchical Z-Buffer (Hi-Z) Pyramid](#hierarchical-z-buffer-hi-z-pyramid)
11. [Visibility Buffer / Deferred Texturing](#visibility-buffer--deferred-texturing)
12. [Cone Culling in Mesh Shaders](#cone-culling-in-mesh-shaders)
13. [Cluster LOD Hierarchy](#cluster-lod-hierarchy)
14. [VK_AMDX_shader_enqueue: Vulkan Workgraphs](#vk_amdx_shader_enqueue-vulkan-workgraphs)
15. [Pipeline Statistics Queries](#pipeline-statistics-queries)
16. [Roadmap](#roadmap)
17. [Integrations](#integrations)

---

## Introduction

Traditional rendering architectures are CPU-driven: the CPU iterates over scene objects, culls them, sets per-draw state (descriptor sets, push constants), and submits `vkCmdDraw` calls. At large scene scales this bottleneck becomes critical — a scene with 100,000 objects requires 100,000 CPU draw calls even if 90,000 are offscreen.

GPU-driven rendering inverts this: the GPU holds a persistent scene representation, runs its own culling in a compute shader, and generates the draw commands itself via `vkCmdDrawIndexedIndirectCount`. The CPU submits one compute dispatch and one indirect draw call per frame regardless of scene size.

This chapter covers:

- **Vulkan API for indirect drawing** — `vkCmdDrawIndexedIndirect`, `vkCmdDrawIndexedIndirectCount`, and the `VkDrawIndexedIndirectCommand` buffer layout
- **GPU culling pipeline** — compute-shader frustum and occlusion culling, Hi-Z pyramid construction, and atomic draw-count generation
- **Mesh shader (task + mesh) model** — meshlet-based rendering, a hardware feature on NVIDIA Turing+, AMD RDNA2+, and Intel Xe

Key implementations studied:

- **`vkguide.dev` GPU-driven tutorial** — step-by-step indirect draw and GPU culling walkthrough
- **Niagara** — high-performance meshlet renderer by Arseny Kapoulkine
- **Bevy's GPU culling pass** — open-source Rust game engine GPU-driven pipeline

### 1.1 What is GPU-Driven Rendering?

GPU-driven rendering is an architectural pattern for real-time 3D rendering where the GPU, rather than the CPU, is responsible for determining which objects to draw and generating the draw commands. In traditional CPU-driven pipelines, the application traverses the scene graph on the CPU, performs visibility culling, and issues one `vkCmdDraw` or `vkCmdDrawIndexed` call per visible object. At large scene scales — tens of thousands of draw calls per frame — this becomes a CPU bottleneck: command recording and driver overhead dominate the frame budget regardless of GPU utilization.

GPU-driven rendering eliminates this bottleneck by storing the entire scene's geometry metadata, bounding volumes, and material indices in GPU-resident buffers. A compute shader reads this data, determines which objects pass frustum and occlusion tests, and writes `VkDrawIndexedIndirectCommand` structures into a draw command buffer. The graphics pipeline then consumes that buffer via `vkCmdDrawIndexedIndirectCount`, issuing only the surviving draw calls. The CPU contributes a single compute dispatch and a single indirect draw submission per frame, decoupling CPU cost from scene complexity.

This pattern is central to modern game engines and renderers targeting scenes with 100,000 or more unique objects. Vulkan's explicit memory model and indirect drawing extensions make it the natural API for GPU-driven work on Linux, where driver overhead is lower than on older APIs but still non-zero. This chapter examines the full pipeline from GPU scene representation through indirect draw, culling, and mesh shader extensions.

### 1.2 What is an Indirect Draw Command?

An indirect draw command is a GPU draw call whose parameters — index count, instance count, vertex offset, and so on — are read from a GPU-resident buffer rather than supplied directly by the CPU at command-recording time. In Vulkan, this mechanism is exposed through `vkCmdDrawIndexedIndirect` and the more powerful `vkCmdDrawIndexedIndirectCount`, both of which accept a `VkBuffer` containing one or more `VkDrawIndexedIndirectCommand` structs.

The key property of indirect drawing is that the buffer contents can be written by another GPU operation — typically a compute shader performing culling — before the draw is executed. This makes draw parameters data-dependent on GPU computation without any CPU readback. The GPU decides how many draws to issue, which geometry to render, and from which buffer offsets to fetch indices, all within a single frame and without stalling the CPU-GPU pipeline.

The indirect mechanism originated in OpenGL with `glMultiDrawElementsIndirect`, was formalized in Direct3D 12's `ExecuteIndirect`, and is exposed in Vulkan 1.2 core through the `drawIndirectCount` feature flag (originally `VK_KHR_draw_indirect_count`). On Linux, all major Vulkan drivers — RADV, ANV, NVK, and the proprietary NVIDIA driver — support this feature on hardware from roughly 2016 onwards. `VK_BUFFER_USAGE_INDIRECT_BUFFER_BIT` must be set on the buffer containing the commands, and a pipeline barrier with `VK_ACCESS_INDIRECT_COMMAND_READ_BIT` ensures the compute-written data is visible to the draw stage.

### 1.3 What is a Meshlet?

A meshlet is a small, fixed-size cluster of triangles carved from a larger mesh, designed to be processed by a single GPU workgroup in the mesh shader pipeline. Where traditional vertex processing works on individual triangles or entire meshes, meshlet decomposition groups roughly 64–126 triangles and up to 64–128 vertices into a self-contained unit with its own local index buffer and a set of precomputed bounding data. The bounding cone and bounding sphere stored per meshlet enable the GPU to cull entire clusters before invoking per-triangle rasterization work.

Meshlet decomposition is performed offline using tools such as meshoptimizer, which optimizes clustering for vertex reuse and spatial coherence. At runtime, a task shader evaluates per-meshlet visibility and emits a variable number of mesh shader workgroups, each of which processes exactly one meshlet. The mesh shader generates the final rasterizable primitives — positions, attributes, and primitive indices — and outputs them in a compact primitive list rather than through a traditional vertex fetch.

In Vulkan, mesh shaders are exposed via `VK_EXT_mesh_shader` (promoted from the earlier `VK_NV_mesh_shader`). The extension introduces two new programmable stages, `VK_SHADER_STAGE_TASK_BIT_EXT` and `VK_SHADER_STAGE_MESH_BIT_EXT`, which replace the vertex, tessellation, and geometry shader stages when active. Hardware support is available on NVIDIA Turing and later, AMD RDNA2 and later, and Intel Xe graphics. This chapter covers the Vulkan API, meshlet data structures, and cone culling within the mesh shader stage.

### 1.4 What is GPU Culling?

GPU culling is the process of discarding geometry that does not contribute to the final rendered image before issuing rasterization work, where the culling decision is made entirely by GPU-side compute shaders rather than by CPU code. Two primary strategies are relevant at the GPU-driven level: frustum culling, which rejects objects whose bounding volumes lie entirely outside the camera's view frustum, and occlusion culling, which additionally rejects objects hidden behind other geometry using a hierarchical depth buffer.

In the GPU-driven pipeline, a culling compute shader receives an array of per-object bounding data (typically bounding spheres or axis-aligned bounding boxes), tests each against the camera frustum planes in parallel — one GPU thread per object — and for surviving objects atomically writes a `VkDrawIndexedIndirectCommand` into the draw buffer and increments a GPU-side draw count. Occlusion culling extends this by sampling the previous frame's depth buffer (the Hi-Z pyramid) to determine whether the projected bounding box is fully occluded by nearer geometry.

The Hi-Z (hierarchical Z) pyramid is a mipmap-like structure built from the depth buffer: each level stores the maximum depth over a 2×2 block of the previous level, allowing a single texture sample to conservatively test whether an object's depth bounds exceed the nearest occluder in its screen-space footprint. This one-frame latency is an acceptable approximation for fast-moving cameras because geometry revealed after a missed cull simply appears one frame late rather than causing corruption. This chapter provides GLSL compute shader implementations of both frustum and Hi-Z occlusion culling in Vulkan.

---

## The CPU-GPU Bottleneck: Why GPU-Driven?

### Per-Draw CPU Overhead

Each `vkCmdDrawIndexed` call on the CPU involves:
- Descriptor set binding (`vkCmdBindDescriptorSets`)
- Push constant update (`vkCmdPushConstants`)
- Command buffer recording overhead
- Driver validation (debug layers add ~10–50 µs each)

For a AAA scene with 50,000–500,000 draws, this serialized CPU work consumes 3–10 ms per frame, a large fraction of a 16 ms budget.

### The GPU-Driven Solution

```
CPU side (once per frame):
  vkCmdDispatch(cull_compute, N/64, 1, 1)  // GPU culls N objects
  vkCmdDrawIndexedIndirectCount(draw_buf, count_buf, max_count)

GPU side (parallel, 10,000+ threads):
  cull.comp:
    for each object[i] in parallel:
      if frustum_cull(object[i].bounds, camera):
        draw_commands[atomicAdd(draw_count)] = make_draw(object[i])
  
  vertex.vert + fragment.frag:
    // draws exactly the surviving objects
```

---

## Indirect Draw Commands

### VkDrawIndexedIndirectCommand

Vulkan's indirect draw reads its arguments from a GPU buffer:

```c
/* vulkan/vulkan_core.h */
typedef struct VkDrawIndexedIndirectCommand {
    uint32_t indexCount;
    uint32_t instanceCount;
    uint32_t firstIndex;
    int32_t  vertexOffset;
    uint32_t firstInstance;
} VkDrawIndexedIndirectCommand;
```

Each struct in the buffer is one draw call. The GPU — not the CPU — writes these structs.

### Basic Indirect Draw

```c
/* Buffer layout:
   [VkDrawIndexedIndirectCommand × N] in draw_buffer */

VkBuffer draw_buffer;  /* GPU-writable, INDIRECT_BUFFER usage */
/* ... create and populate from CPU for first frame or GPU always ... */

vkCmdBindIndexBuffer(cmd, index_buf, 0, VK_INDEX_TYPE_UINT32);
vkCmdBindVertexBuffers(cmd, 0, 1, &vertex_buf, &offset_zero);

vkCmdDrawIndexedIndirect(
    cmd,
    draw_buffer,        /* buffer containing VkDrawIndexedIndirectCommand array */
    0,                  /* offset into buffer */
    draw_count,         /* number of draw commands */
    sizeof(VkDrawIndexedIndirectCommand)  /* stride */
);
```

---

## Multi-Draw Indirect and Draw Count

### VkDrawIndexedIndirectCount

`vkCmdDrawIndexedIndirectCount` (core in Vulkan 1.2) reads the draw count from a GPU buffer, enabling the GPU to determine how many draws to issue:

```c
/* After GPU culling writes draw_count_buffer: */
vkCmdDrawIndexedIndirectCount(
    cmd,
    draw_commands_buffer,   /* VkDrawIndexedIndirectCommand[] */
    0,
    draw_count_buffer,      /* uint32_t: GPU-written surviving draw count */
    0,
    max_draws,              /* upper bound for driver */
    sizeof(VkDrawIndexedIndirectCommand)
);
```

The extension is `VK_KHR_draw_indirect_count`, promoted to Vulkan 1.2 core. Check support:

```c
VkPhysicalDeviceVulkan12Features features12 = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_VULKAN_1_2_FEATURES };
/* features12.drawIndirectCount must be VK_TRUE */
```

### Buffer Requirements

```c
VkBufferCreateInfo draw_buf_ci = {
    .size  = max_draws * sizeof(VkDrawIndexedIndirectCommand),
    .usage = VK_BUFFER_USAGE_INDIRECT_BUFFER_BIT
           | VK_BUFFER_USAGE_STORAGE_BUFFER_BIT  /* compute writes */
           | VK_BUFFER_USAGE_TRANSFER_DST_BIT,
};
/* Allocate in GPU-local memory (VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT) */
```

---

## GPU Culling with Compute Shaders

### Scene Object Representation

```glsl
/* types.glsl — shared between CPU (via UBO) and GPU */
struct ObjectData {
    mat4  transform;
    vec4  bounds_sphere;  /* xyz = center, w = radius */
    uint  mesh_index;
    uint  material_index;
    uint  index_count;
    uint  first_index;
    int   vertex_offset;
    uint  _pad[3];
};

struct DrawCommand {
    uint  indexCount;
    uint  instanceCount;  /* 1 for non-instanced */
    uint  firstIndex;
    int   vertexOffset;
    uint  firstInstance;
};
```

### Frustum Culling Compute Shader

```glsl
/* cull.comp */
#version 460
#extension GL_EXT_buffer_reference : require

layout(local_size_x = 64) in;

layout(set = 0, binding = 0) readonly buffer ObjectBuffer {
    ObjectData objects[];
};
layout(set = 0, binding = 1) writeonly buffer DrawCommandBuffer {
    DrawCommand draws[];
};
layout(set = 0, binding = 2) buffer DrawCountBuffer {
    uint draw_count;
};

layout(push_constant) uniform CullData {
    mat4  view_proj;
    vec4  frustum_planes[6];   /* left, right, bottom, top, near, far */
    uint  object_count;
};

bool frustum_cull(vec4 sphere) {
    for (int i = 0; i < 6; i++) {
        if (dot(frustum_planes[i], vec4(sphere.xyz, 1.0)) < -sphere.w)
            return false;  /* outside this plane */
    }
    return true;
}

void main() {
    uint idx = gl_GlobalInvocationID.x;
    if (idx >= object_count) return;

    ObjectData obj = objects[idx];
    vec4 world_sphere = vec4(
        (obj.transform * vec4(obj.bounds_sphere.xyz, 1.0)).xyz,
        obj.bounds_sphere.w * max(
            length(obj.transform[0].xyz),
            max(length(obj.transform[1].xyz), length(obj.transform[2].xyz)))
    );

    if (!frustum_cull(world_sphere)) return;

    uint draw_idx = atomicAdd(draw_count, 1);
    draws[draw_idx] = DrawCommand(
        obj.index_count,
        1,
        obj.first_index,
        obj.vertex_offset,
        draw_idx   /* firstInstance: used to fetch per-object data in vertex shader */
    );
}
```

**Slang equivalent** — `ParameterBlock<SceneBindings>` groups all scene SSBOs into a typed, reflection-visible block, and `[Differentiable]` on `frustum_cull()` enables `bwd_diff()` to compute gradients through the frustum-sphere signed distance for differentiable rendering losses.

```slang
// File: cull.slang
// GPU frustum culling compute shader — matches cull.comp
// Slang improvements:
//   - ParameterBlock<SceneBindings> groups scene SSBOs into a typed, reflection-visible block
//     eliminating manual set/binding bookkeeping across CPU and shader
//   - [Differentiable] on frustum_cull() enables bwd_diff() for gradient-based cull tuning
//     (e.g. differentiating a soft-culling loss w.r.t. learned frustum plane parameters)
//   - [[vk::push_constant]] replaces layout(push_constant) uniform; struct is first-class

import scene_types;  // shared ObjectData / DrawCommand definitions

struct SceneBindings {
    StructuredBuffer<ObjectData>    objects;
    RWStructuredBuffer<DrawCommand> draws;
    RWStructuredBuffer<uint>        draw_count;
};

struct CullPushConstants {
    float4x4 view_proj;
    float4   frustum_planes[6];
    uint     object_count;
};

ParameterBlock<SceneBindings>      scene;
[[vk::push_constant]] CullPushConstants g_cull;

// [Differentiable]: Slang auto-generates the backward pass for this predicate,
// allowing gradient flow through the per-plane signed-distance expression used in
// soft culling losses during neural scene representation training.
[Differentiable]
bool frustum_cull(float4 sphere, float4 frustum_planes[6]) {
    [ForceUnroll]
    for (int i = 0; i < 6; i++) {
        if (dot(frustum_planes[i], float4(sphere.xyz, 1.0)) < -sphere.w)
            return false;
    }
    return true;
}

[shader("compute")]
[numthreads(64, 1, 1)]
void main(uint3 dispatch_id : SV_DispatchThreadID) {
    uint idx = dispatch_id.x;
    if (idx >= g_cull.object_count) return;

    ObjectData obj = scene.objects[idx];

    // Transform bounding sphere into world space
    float3 world_center = mul(obj.transform, float4(obj.bounds_sphere.xyz, 1.0)).xyz;
    float  scale = max(length(obj.transform[0].xyz),
                   max(length(obj.transform[1].xyz), length(obj.transform[2].xyz)));
    float4 world_sphere = float4(world_center, obj.bounds_sphere.w * scale);

    if (!frustum_cull(world_sphere, g_cull.frustum_planes)) return;

    uint draw_idx;
    InterlockedAdd(scene.draw_count[0], 1u, draw_idx);

    DrawCommand cmd = { obj.index_count, 1u, obj.first_index,
                        obj.vertex_offset, draw_idx };
    scene.draws[draw_idx] = cmd;
}
```

### Synchronisation Between Cull and Draw

```c
/* Pipeline barrier: cull compute writes → indirect draw reads */
VkMemoryBarrier2 barrier = {
    .sType         = VK_STRUCTURE_TYPE_MEMORY_BARRIER_2,
    .srcStageMask  = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
    .srcAccessMask = VK_ACCESS_2_SHADER_WRITE_BIT,
    .dstStageMask  = VK_PIPELINE_STAGE_2_DRAW_INDIRECT_BIT,
    .dstAccessMask = VK_ACCESS_2_INDIRECT_COMMAND_READ_BIT,
};
VkDependencyInfo dep = {
    .sType = VK_STRUCTURE_TYPE_DEPENDENCY_INFO,
    .memoryBarrierCount = 1,
    .pMemoryBarriers = &barrier,
};
vkCmdPipelineBarrier2(cmd, &dep);
```

### Occlusion Culling (Two-Phase)

Advanced GPU culling uses a depth pyramid (Hi-Z) for occlusion:

```
Frame N:
  Phase 1: Frustum cull → draw surviving objects → render depth buffer
  Phase 2: Occlusion cull against depth pyramid → draw previously culled objects that are now visible

Frame N+1: repeat using Frame N's depth buffer as Hi-Z source
```

```glsl
/* Occlusion test in cull.comp (simplified): */
bool occlusion_cull(vec4 sphere) {
    /* Project sphere to screen space */
    vec4 clip = view_proj * vec4(sphere.xyz, 1.0);
    clip /= clip.w;
    vec2 uv = clip.xy * 0.5 + 0.5;
    float proj_radius = sphere.w / clip.w;

    /* Sample Hi-Z mip that covers the sphere's screen footprint */
    float mip = ceil(log2(proj_radius * max(screen_width, screen_height)));
    float depth_sample = textureLod(depth_pyramid, uv, mip).r;

    /* Sphere is occluded if its nearest depth is behind the sampled depth */
    float nearest_depth = (clip.z - sphere.w / clip.w);
    return nearest_depth < depth_sample;
}
```

---

## Meshlets and Mesh Shaders

### What Is a Meshlet?

A **meshlet** is a small cluster of triangles (typically 64–124) pre-packaged for GPU processing. Instead of one giant index buffer for a mesh, a meshlet-based mesh stores:

```c
struct Meshlet {
    uint32_t vertex_offset;    /* offset into meshlet vertex buffer */
    uint32_t triangle_offset;  /* offset into meshlet triangle buffer */
    uint8_t  vertex_count;     /* number of unique vertices (≤ 64) */
    uint8_t  triangle_count;   /* number of triangles (≤ 124) */
    /* Bounding sphere for per-meshlet culling: */
    float    bounds_center[3];
    float    bounds_radius;
    /* Cone for backface culling: */
    float    cone_apex[3];
    float    cone_axis[3];
    float    cone_cutoff;
};
```

Meshlet generation: [meshoptimizer](https://github.com/zeux/meshoptimizer) by Arseny Kapoulkine.

```c
/* meshoptimizer API: */
size_t meshlet_count = meshopt_buildMeshletsBound(index_count, 64, 124);
meshopt_Meshlet *meshlets = malloc(meshlet_count * sizeof(meshopt_Meshlet));
unsigned int *meshlet_vertices = malloc(meshlet_count * 64 * sizeof(unsigned int));
unsigned char *meshlet_triangles = malloc(meshlet_count * 124 * 3);

meshopt_buildMeshlets(meshlets, meshlet_vertices, meshlet_triangles,
    indices, index_count,
    positions, vertex_count, sizeof(float) * 3,
    64, 124, 0.0f);
```

---

## Task and Mesh Shader Pipeline

### Pipeline Stages

Mesh shaders replace the vertex+geometry pipeline:

```
Traditional: Input Assembly → Vertex Shader → (Geometry Shader) → Rasterizer
Mesh:        Task Shader (optional) → Mesh Shader → Rasterizer
```

- **Task shader** (amplification shader): runs per-meshlet workgroup, performs per-meshlet culling, emits mesh shader workgroups for surviving meshlets
- **Mesh shader**: runs per-meshlet, outputs primitives directly (no index buffer)

### Extension Requirements

```c
/* VK_EXT_mesh_shader (promoted from NV version, cross-vendor): */
VkPhysicalDeviceMeshShaderFeaturesEXT mesh_features = {
    .sType      = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_MESH_SHADER_FEATURES_EXT,
    .taskShader = VK_TRUE,
    .meshShader = VK_TRUE,
};
```

Support: NVIDIA Turing+ (GeForce RTX 20xx+), AMD RDNA2+ (RX 6000+), Intel Xe-HPG (Arc A-series).

### Task Shader (Per-Meshlet Culling)

```glsl
/* task.task.glsl */
#version 460
#extension GL_EXT_mesh_shader : require

layout(local_size_x = 32) in;  /* 32 meshlets per task workgroup */

taskPayloadSharedEXT struct {
    uint meshlet_indices[32];
    uint count;
} payload;

layout(set = 0, binding = 0) readonly buffer MeshletBuffer {
    Meshlet meshlets[];
};
layout(push_constant) uniform CullData {
    mat4 view_proj;
    vec4 frustum_planes[6];
    uint meshlet_count;
};

shared uint visible_count;

void main() {
    uint tid = gl_LocalInvocationID.x;
    uint meshlet_id = gl_WorkGroupID.x * 32 + tid;

    if (tid == 0) visible_count = 0;
    barrier();

    if (meshlet_id < meshlet_count) {
        Meshlet m = meshlets[meshlet_id];
        bool visible = frustum_cull(vec4(m.bounds_center, m.bounds_radius));
        /* Backface cone cull: */
        visible = visible && !cone_cull(m.cone_apex, m.cone_axis, m.cone_cutoff);

        if (visible) {
            uint slot = atomicAdd(visible_count, 1);
            payload.meshlet_indices[slot] = meshlet_id;
        }
    }

    barrier();
    if (tid == 0) payload.count = visible_count;

    /* Emit one mesh shader workgroup per visible meshlet: */
    EmitMeshTasksEXT(payload.count, 1, 1);
}
```

### Mesh Shader (Primitive Output)

```glsl
/* mesh.mesh.glsl */
#version 460
#extension GL_EXT_mesh_shader : require

layout(local_size_x = 64) in;
layout(triangles, max_vertices = 64, max_primitives = 124) out;

taskPayloadSharedEXT struct {
    uint meshlet_indices[32];
    uint count;
} payload;

layout(location = 0) out vec3 out_normal[];
layout(location = 1) out vec2 out_uv[];

void main() {
    uint meshlet_id = payload.meshlet_indices[gl_WorkGroupID.x];
    Meshlet m = meshlets[meshlet_id];

    SetMeshOutputsEXT(m.vertex_count, m.triangle_count);

    uint tid = gl_LocalInvocationID.x;

    /* Output vertices: */
    if (tid < m.vertex_count) {
        uint vi = meshlet_vertices[m.vertex_offset + tid];
        gl_MeshVerticesEXT[tid].gl_Position = view_proj *
            model * vec4(positions[vi], 1.0);
        out_normal[tid] = normals[vi];
        out_uv[tid]     = uvs[vi];
    }

    /* Output triangles: */
    if (tid < m.triangle_count) {
        uint base = (m.triangle_offset + tid) * 3;
        gl_PrimitiveTriangleIndicesEXT[tid] = uvec3(
            meshlet_triangles[base],
            meshlet_triangles[base + 1],
            meshlet_triangles[base + 2]);
    }
}
```

### Drawing Mesh Shader Pipeline

```c
/* No vertex buffer binding — mesh shader generates vertices: */
vkCmdDrawMeshTasksIndirectCountEXT(
    cmd,
    task_dispatch_buffer,  /* VkDrawMeshTasksIndirectCommandEXT[] */
    0,
    count_buffer, 0,
    max_dispatches,
    sizeof(VkDrawMeshTasksIndirectCommandEXT)
);
```

```c
typedef struct VkDrawMeshTasksIndirectCommandEXT {
    uint32_t groupCountX;
    uint32_t groupCountY;
    uint32_t groupCountZ;
} VkDrawMeshTasksIndirectCommandEXT;
```

---

## Persistent GPU Scene Representation

### Scene Buffer Layout

```c
/* GPU-side scene: one large buffer, updated incrementally */
struct GPUScene {
    ObjectData  objects[MAX_OBJECTS];   /* transform, bounds, mesh refs */
    MeshData    meshes[MAX_MESHES];     /* vertex/index buffer offsets */
    MaterialData materials[MAX_MATS];   /* texture handles, PBR params */
    /* Updated via streaming: */
    uint32_t    object_count;
};
```

All meshes share one vertex buffer and one index buffer (or one buffer per attribute). This avoids binding changes between draws.

### Streaming Updates

Scene updates go through a staging buffer:

```c
/* Each frame: copy modified objects to GPU */
void *mapped;
vkMapMemory(device, staging_mem, 0, update_size, 0, &mapped);
memcpy(mapped, dirty_objects, update_size);
vkUnmapMemory(device, staging_mem);

VkBufferCopy copy = { .srcOffset = 0, .dstOffset = dirty_start * sizeof(ObjectData),
                      .size = dirty_count * sizeof(ObjectData) };
vkCmdCopyBuffer(cmd, staging_buf, scene_buf, 1, &copy);
```

---

## Bindless Resources

GPU-driven rendering requires bindless access to textures and buffers — the draw shader doesn't know which texture each object uses until it reads the material index at runtime.

### Pros and Cons

Bindless is not a strict improvement over traditional descriptor binding — it makes a deliberate tradeoff: eliminate CPU-side descriptor state changes at the cost of GPU-side non-uniform access complexity and programmer-managed resource hazards.

#### Advantages

**Enables GPU-driven rendering.** This is the primary motivation. When the GPU generates draw calls via `vkCmdDrawIndexedIndirectCount`, the CPU does not know which objects will survive culling or in what order. The shader must look up the material at runtime using the draw's material index. A traditional per-material descriptor set cannot be bound ahead of time because the draw list is unknown until after the GPU culling pass completes.

**Eliminates per-draw descriptor bind overhead.** Traditional pipelines call `vkCmdBindDescriptorSets` for every draw that changes texture set, flushing the GPU's descriptor cache and serialising submission. With a single global bindless descriptor set bound once per frame, this cost disappears. At 100k+ draws per frame (typical for open-world scenes), this can be the difference between a CPU-bound and GPU-bound renderer.

**Reduces pipeline state explosion.** Each unique combination of bound resources can require a separate pipeline if resource counts vary. Bindless collapses all material variants to a single runtime-indexed array, enabling the uber-shader / megashader pattern: one pipeline handles every material type by branching on the material data.

**Supports texture streaming and virtual texturing.** `VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT` allows slots to be unoccupied — unstreamed texture slots need not be valid descriptors. `VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT` lets the streaming system insert new texture descriptors after the descriptor set has been submitted to a command buffer (provided the GPU has not yet executed the commands that reference those slots).

**Mandatory for ray tracing.** In a ray tracing pass, any-hit and closest-hit shaders may execute on any surface in the scene. There is no equivalent to per-draw descriptor binding; the shader must dynamically select the material from a global table. Bindless is architecturally required.

**Buffer device address removes descriptors for buffers entirely.** `VK_KHR_buffer_device_address` (Vulkan 1.2 core) gives shaders raw 64-bit GPU pointers, enabling pointer-based GPU data structures (BVH nodes, linked lists, sparse voxel octrees) with zero descriptor overhead.

#### Disadvantages and Costs

**Non-uniform indexing serialises wave execution.** When different invocations within a subgroup access different textures, the hardware must handle each unique index separately — effectively serialising what would otherwise be parallel texture fetches within the wave. The `nonuniformEXT` qualifier (`GL_EXT_nonuniform_qualifier`) is required to tell the driver this is happening; omitting it is undefined behaviour. On modern Turing+ and RDNA 2+ hardware the cost is low due to improved texture unit scheduling, but it remains a real concern on older hardware and at very high material diversity per wave.

**Implicit LOD / mip selection breaks at material boundaries.** `texture()` computes implicit LOD from texture coordinate derivatives across a 2×2 pixel quad. If adjacent pixels in the quad sample *different* textures (nonuniform access), the derivative is still computed but applies to coordinates whose texture indices may differ — mip selection can be incorrect at hard material transitions. The fix is to use `textureGrad()` with explicit derivatives or `textureLod()` with pre-computed LOD, at additional ALU cost.

**Resource hazard tracking becomes the programmer's responsibility.** Vulkan validation can verify that a bound descriptor refers to an image in the correct layout — but only when the image is bound explicitly. With a partially-bound bindless array, the driver cannot know at command recording time which slots the shader will access. All textures that might be referenced must be transitioned to `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL` before any draw that uses the bindless set. Missing a transition is silently undefined behaviour; validation layers cannot catch it.

**Debugging is harder.** RenderDoc can display which descriptor was accessed at a given draw, but correlating a runtime texture index (e.g. `material_id = 4712`) back to the source asset requires either debug tooling that maps indices to asset names, or careful instrumentation in the material streaming layer. GPU-based validation (`VK_EXT_device_address_binding_report`) provides some coverage but not the same per-draw clarity as traditional descriptor binding.

**All resident textures must be memory-backed (absent sparse/virtual texturing).** In a traditional renderer, only the textures for currently-visible objects need VRAM. With a fully populated bindless array, all textures in the scene must be resident or you must implement a streaming/eviction system with slot management. Without virtual textures (`VK_EXT_image_drm_format_modifier` + sparse binding), VRAM pressure scales with scene size rather than visible-object count.

**Feature requirements limit portability.** The full bindless feature set (`runtimeDescriptorArray`, `shaderSampledImageArrayNonUniformIndexing`, `descriptorBindingPartiallyBound`, `descriptorBindingUpdateAfterBind`) requires `VK_EXT_descriptor_indexing` (promoted to Vulkan 1.2 core). WebGPU does not currently expose bindless descriptor arrays; OpenGL exposes `ARB_bindless_texture` but with a different model. Portability to older Android or embedded Vulkan devices requires fallback paths.

**Summary table:**

| Dimension | Traditional binding | Bindless |
|-----------|--------------------|-|
| Per-draw CPU cost | `vkCmdBindDescriptorSets` per state change | Zero — bind once per frame |
| GPU-driven rendering | Not possible (CPU must know draw order) | Required capability |
| Non-uniform texture access | Impossible by design | Supported; wave serialisation cost |
| Implicit mip / LOD | Correct (uniform access) | May need explicit `textureGrad`/`textureLod` |
| Resource hazard tracking | Driver/validation assisted | Programmer responsibility |
| Partial occupancy | Impossible | `PARTIALLY_BOUND` flag |
| Texture streaming | Per-draw descriptor update | `UPDATE_AFTER_BIND` slot management |
| Debugging | Per-draw visibility in RenderDoc | Requires index→asset mapping tooling |
| Vulkan minimum | 1.0 | 1.2 (or `VK_EXT_descriptor_indexing`) |

### VK_EXT_descriptor_indexing

```c
VkPhysicalDeviceDescriptorIndexingFeatures di_features = {
    .runtimeDescriptorArray             = VK_TRUE,
    .shaderSampledImageArrayNonUniformIndexing = VK_TRUE,
    .descriptorBindingPartiallyBound    = VK_TRUE,
    .descriptorBindingUpdateAfterBind   = VK_TRUE,
};
```

### Bindless Descriptor Set

```c
/* One descriptor set with all textures: */
VkDescriptorSetLayoutBinding texture_binding = {
    .binding         = 0,
    .descriptorType  = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
    .descriptorCount = 65536,  /* max textures */
    .stageFlags      = VK_SHADER_STAGE_FRAGMENT_BIT,
};
VkDescriptorBindingFlags flags =
    VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT |
    VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT;
```

### GLSL Bindless Access

```glsl
#extension GL_EXT_nonuniform_qualifier : require

layout(set = 0, binding = 0) uniform sampler2D textures[];

layout(location = 0) in flat uint material_id;

void main() {
    MaterialData mat = materials[material_id];
    /* Non-uniform index — different invocations may access different textures: */
    vec4 albedo = texture(textures[nonuniformEXT(mat.albedo_texture)], uv);
}
```

### Buffer Device Address

For scene buffers, `VK_KHR_buffer_device_address` (Vulkan 1.2 core) gives shaders raw 64-bit GPU pointers:

```glsl
#extension GL_EXT_buffer_reference : require

layout(buffer_reference, scalar) readonly buffer ObjectBuffer {
    ObjectData data[];
};

layout(push_constant) uniform PC {
    ObjectBuffer objects;  /* 64-bit GPU address */
    uint object_count;
};

void main() {
    ObjectData obj = objects.data[gl_DrawID];
}
```

`gl_DrawID` (available with `GL_ARB_shader_draw_parameters`) gives the index of the current draw within a multi-draw-indirect call.

---

## Hierarchical Z-Buffer (Hi-Z) Pyramid

Two-phase occlusion culling (introduced in the GPU Culling section above) depends on a **hierarchical Z-buffer** (Hi-Z pyramid): a mip chain of the depth buffer where each mip stores the minimum depth over a 2×2 region of the level above. The culling compute shader then samples the mip level whose texel footprint covers the projected bounding sphere, and rejects the object if the sphere's nearest depth is behind that conservative depth value. [Source](https://www.rastergrid.com/blog/2010/10/hierarchical-z-map-based-occlusion-culling/)

### Min-Reduction Algorithm

Each mip level samples the four texels from the previous level and writes their minimum. This is a conservative reduction: a texel in the pyramid is guaranteed to be no closer than any depth in the covered region, so a failed occlusion test is a true negative (no false culls). [Source](https://github.com/google/filament/blob/main/filament/src/materials/ssao/mipmapDepth.mat)

For **non-power-of-two** dimensions (common on modern displays), the source mip at the odd boundary has only 1 or 2 texels along that axis. The standard fix is to widen the sample footprint at the last column/row: if `(src_width & 1) != 0`, sample a third column; if `(src_height & 1) != 0`, sample a third row. Filament's depth mip shader handles this explicitly. [Source](https://github.com/google/filament/blob/main/filament/src/materials/ssao/mipmapDepth.mat)

### GLSL Compute Shader — Single Mip Reduction Pass

```glsl
/* hiz_reduce.comp — reduces mip level (N-1) → mip level N */
#version 460

layout(local_size_x = 8, local_size_y = 8) in;

/* Previous mip (read) and current mip (write) bound as separate images */
layout(set = 0, binding = 0) uniform sampler2D src_depth;   /* mip N-1 */
layout(set = 0, binding = 1, r32f) writeonly uniform image2D dst_depth; /* mip N */

layout(push_constant) uniform PushConstants {
    ivec2 src_size;   /* dimensions of mip N-1 */
    ivec2 dst_size;   /* dimensions of mip N   */
};

void main() {
    ivec2 dst_coord = ivec2(gl_GlobalInvocationID.xy);
    if (any(greaterThanEqual(dst_coord, dst_size))) return;

    /* Map dst texel to 2×2 source region */
    ivec2 src_base = dst_coord * 2;

    /* Gather 4 samples from previous mip level; clamp to border */
    float d00 = texelFetch(src_depth, clamp(src_base + ivec2(0,0), ivec2(0), src_size - 1), 0).r;
    float d10 = texelFetch(src_depth, clamp(src_base + ivec2(1,0), ivec2(0), src_size - 1), 0).r;
    float d01 = texelFetch(src_depth, clamp(src_base + ivec2(0,1), ivec2(0), src_size - 1), 0).r;
    float d11 = texelFetch(src_depth, clamp(src_base + ivec2(1,1), ivec2(0), src_size - 1), 0).r;

    /* Conservative minimum — for reversed-Z, replace min with max */
    float min_depth = min(min(d00, d10), min(d01, d11));

    /* Extra column for odd-width source */
    if ((src_size.x & 1) != 0) {
        float dx0 = texelFetch(src_depth, clamp(src_base + ivec2(2,0), ivec2(0), src_size - 1), 0).r;
        float dx1 = texelFetch(src_depth, clamp(src_base + ivec2(2,1), ivec2(0), src_size - 1), 0).r;
        min_depth = min(min_depth, min(dx0, dx1));
    }
    /* Extra row for odd-height source */
    if ((src_size.y & 1) != 0) {
        float dy0 = texelFetch(src_depth, clamp(src_base + ivec2(0,2), ivec2(0), src_size - 1), 0).r;
        float dy1 = texelFetch(src_depth, clamp(src_base + ivec2(1,2), ivec2(0), src_size - 1), 0).r;
        min_depth = min(min_depth, min(dy0, dy1));
    }

    imageStore(dst_depth, dst_coord, vec4(min_depth));
}
```

**Slang equivalent** — a `generic<let USE_MAX : bool>` compile-time parameter unifies the standard (min) and reversed-Z (max) depth reduction paths into one shader body, selected via a Vulkan specialisation constant rather than a preprocessor permutation or two separate shader objects.

```slang
// File: hiz_reduce.slang
// Hi-Z pyramid single-mip depth reduction — matches hiz_reduce.comp
// Slang improvements:
//   - generic<let USE_MAX : bool> eliminates a preprocessor #define permutation for
//     reversed-Z (max) vs. standard (min) depth without any runtime branch on the hot path
//   - [vk::constant_id(0)] selects the depth convention at pipeline-creation time;
//     no shader recompile needed when switching cameras between depth conventions
//   - Texture2D<float>/RWTexture2D<float> express the r32f format intent explicitly;
//     Load() replaces texelFetch for consistent HLSL/Slang syntax

[[vk::binding(0, 0)]] Texture2D<float>   g_src;
[[vk::binding(1, 0)]] RWTexture2D<float> g_dst;

struct HiZPC { int2 src_size; int2 dst_size; };
[[vk::push_constant]] HiZPC g_pc;

// Compile-time reduction operator — no runtime branch on the critical inner loop
generic<let USE_MAX : bool>
float depth_reduce(float a, float b) { return USE_MAX ? max(a, b) : min(a, b); }

generic<let USE_MAX : bool>
float sample_clamped(Texture2D<float> tex, int2 coord, int2 bound) {
    return tex.Load(int3(clamp(coord, int2(0), bound - int2(1)), 0));
}

generic<let USE_MAX : bool>
void hiz_reduce_impl(uint3 id) {
    int2 dst   = int2(id.xy);
    if (any(dst >= g_pc.dst_size)) return;

    int2 base  = dst * 2;
    int2 bound = g_pc.src_size;

    float d00 = sample_clamped<USE_MAX>(g_src, base + int2(0,0), bound);
    float d10 = sample_clamped<USE_MAX>(g_src, base + int2(1,0), bound);
    float d01 = sample_clamped<USE_MAX>(g_src, base + int2(0,1), bound);
    float d11 = sample_clamped<USE_MAX>(g_src, base + int2(1,1), bound);

    float result = depth_reduce<USE_MAX>(depth_reduce<USE_MAX>(d00, d10),
                                         depth_reduce<USE_MAX>(d01, d11));

    // Non-power-of-two source: widen footprint at odd column/row boundary
    if ((g_pc.src_size.x & 1) != 0) {
        float dx0 = sample_clamped<USE_MAX>(g_src, base + int2(2,0), bound);
        float dx1 = sample_clamped<USE_MAX>(g_src, base + int2(2,1), bound);
        result = depth_reduce<USE_MAX>(result, depth_reduce<USE_MAX>(dx0, dx1));
    }
    if ((g_pc.src_size.y & 1) != 0) {
        float dy0 = sample_clamped<USE_MAX>(g_src, base + int2(0,2), bound);
        float dy1 = sample_clamped<USE_MAX>(g_src, base + int2(1,2), bound);
        result = depth_reduce<USE_MAX>(result, depth_reduce<USE_MAX>(dy0, dy1));
    }

    g_dst[dst] = result;
}

// Specialisation constant: 0 = standard depth (min), 1 = reversed-Z (max).
// The host picks the variant at VkPipeline creation; no separate shader object required.
[vk::constant_id(0)] const bool kReversedZ = false;

[shader("compute")]
[numthreads(8, 8, 1)]
void main(uint3 id : SV_DispatchThreadID) {
    if (kReversedZ) hiz_reduce_impl<true>(id);
    else            hiz_reduce_impl<false>(id);
}
```

### C Dispatch Loop

Building the full pyramid requires one dispatch per mip level, with an image memory barrier between each pass to ensure the previous write is visible to the next read:

```c
/* hiz_build() — call once after the depth pre-pass each frame */
void hiz_build(VkCommandBuffer cmd,
               VkImageView     *hiz_mip_views,  /* one view per mip, r32f */
               VkDescriptorSet *hiz_desc_sets,  /* one set per mip transition */
               uint32_t         mip_levels,
               uint32_t         base_width,
               uint32_t         base_height)
{
    for (uint32_t mip = 1; mip < mip_levels; mip++) {
        uint32_t src_w = max(base_width  >> (mip - 1), 1u);
        uint32_t src_h = max(base_height >> (mip - 1), 1u);
        uint32_t dst_w = max(base_width  >> mip,       1u);
        uint32_t dst_h = max(base_height >> mip,       1u);

        /* Barrier: previous mip write → this mip's sampler read */
        VkImageMemoryBarrier2 img_barrier = {
            .sType            = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER_2,
            .srcStageMask     = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
            .srcAccessMask    = VK_ACCESS_2_SHADER_WRITE_BIT,
            .dstStageMask     = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
            .dstAccessMask    = VK_ACCESS_2_SHADER_READ_BIT,
            .oldLayout        = VK_IMAGE_LAYOUT_GENERAL,
            .newLayout        = VK_IMAGE_LAYOUT_GENERAL,
            .image            = hiz_image,
            .subresourceRange = { VK_IMAGE_ASPECT_COLOR_BIT, mip - 1, 1, 0, 1 },
        };
        VkDependencyInfo dep = {
            .sType                   = VK_STRUCTURE_TYPE_DEPENDENCY_INFO,
            .imageMemoryBarrierCount = 1,
            .pImageMemoryBarriers    = &img_barrier,
        };
        vkCmdPipelineBarrier2(cmd, &dep);

        /* Bind descriptor set for this mip pair and dispatch */
        struct { int32_t src_w, src_h, dst_w, dst_h; } pc = {
            (int32_t)src_w, (int32_t)src_h, (int32_t)dst_w, (int32_t)dst_h };
        vkCmdBindDescriptorSets(cmd, VK_PIPELINE_BIND_POINT_COMPUTE,
                                hiz_layout, 0, 1, &hiz_desc_sets[mip - 1], 0, NULL);
        vkCmdPushConstants(cmd, hiz_layout,
                           VK_SHADER_STAGE_COMPUTE_BIT, 0, sizeof(pc), &pc);
        vkCmdDispatch(cmd,
                      (dst_w + 7) / 8,
                      (dst_h + 7) / 8,
                      1);
    }
}
```

Note: reversed-Z depth buffers (near=1, far=0) are common in Vulkan because they improve floating-point precision. In that case replace `min` with `max` throughout the reduction, and invert the occlusion comparison in the cull shader. [Source](https://developer.nvidia.com/content/depth-precision-visualized)

---

## Visibility Buffer / Deferred Texturing

The G-buffer in a classic deferred renderer stores world-space position, normal, and material parameters for every visible fragment. On depth-complex scenes this wastes bandwidth: fragments that are later overwritten still write a full G-buffer record. The **visibility buffer** (also called the material buffer) decouples geometry rasterisation from shading by deferring all material evaluation to a full-screen compute or fragment pass. [Source](http://filmicworlds.com/blog/visibility-buffer-rendering-with-material-graphs/)

### Pass 1: Visibility Rasterisation

The first pass rasterises the scene with a minimal fragment shader that writes only the draw ID and primitive ID, packed into a single `uvec2`:

```glsl
/* visibility.frag — Pass 1: write visibility buffer */
#version 460
#extension GL_ARB_shader_draw_parameters : require

layout(location = 0) out uvec2 vis_buf;  /* R32G32_UINT render target */

void main() {
    /* gl_DrawID: index within vkCmdDrawIndexedIndirectCount batch (0-based)
       gl_PrimitiveID: triangle index within the current draw */
    vis_buf = uvec2(uint(gl_DrawID), uint(gl_PrimitiveID));
}
```

The render target format is `VK_FORMAT_R32G32_UINT`. Because the fragment shader does no texture sampling, the rasterisation pass is bandwidth-minimal and runs at high occupancy even on depth-complex scenes. Production adoption: *The Surge* (Deck13) used a visibility buffer variant to handle its dense industrial environments; *Ghost of Tsushima* PC port uses a similar deferred-texturing approach for its foliage. [Source](https://advances.realtimerendering.com/s2015/)

### Pass 2: Full-Screen Shading with Barycentric Interpolation

The shading pass reads the visibility buffer and reconstructs every shading attribute from BDA (buffer device address) scene buffers. UV coordinates are interpolated using `gl_BaryCoordEXT` from `VK_KHR_fragment_shader_barycentric`, which exposes the hardware barycentric weights of the current screen-space position within the triangle. [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_fragment_shader_barycentric.html)

```glsl
/* shade.frag — Pass 2: full-screen deferred texturing */
#version 460
#extension GL_EXT_buffer_reference        : require
#extension GL_KHR_fragment_shader_barycentric : require   /* VK_KHR_fragment_shader_barycentric */
#extension GL_EXT_nonuniform_qualifier    : require

/* --- BDA scene layout ------------------------------------------------ */
layout(buffer_reference, scalar) readonly buffer IndexBuffer   { uint  idx[]; };
layout(buffer_reference, scalar) readonly buffer PositionBuffer{ vec3  pos[]; };
layout(buffer_reference, scalar) readonly buffer UVBuffer      { vec2  uv[];  };
layout(buffer_reference, scalar) readonly buffer DrawDataBuffer{
    /* per-draw: BDA pointers to the mesh's own index/vertex buffers */
    IndexBuffer    indices;
    PositionBuffer positions;
    UVBuffer       uvs;
    uint           material_id;
};

layout(push_constant) uniform PC {
    DrawDataBuffer draws;         /* array base; index with draw_id */
    uint64_t       material_buf;  /* MaterialData[] BDA */
};

/* Bindless texture array */
layout(set = 0, binding = 0) uniform sampler2D textures[];

/* Visibility buffer from Pass 1 */
layout(set = 0, binding = 1) uniform usampler2D vis_buf;

layout(location = 0) out vec4 out_color;

void main() {
    ivec2 coord = ivec2(gl_FragCoord.xy);
    uvec2 vis   = texelFetch(vis_buf, coord, 0).rg;

    uint draw_id = vis.x;
    uint prim_id = vis.y;

    /* Fetch per-draw BDA pointers */
    DrawDataBuffer draw = draws[draw_id];  /* BDA array index */

    /* Reconstruct the three vertex indices for this primitive */
    uint i0 = draw.indices.idx[prim_id * 3 + 0];
    uint i1 = draw.indices.idx[prim_id * 3 + 1];
    uint i2 = draw.indices.idx[prim_id * 3 + 2];

    /* Hardware barycentric weights — available as gl_BaryCoordEXT (vec3)
       interpolated across the triangle at the current screen pixel */
    vec3 bary = gl_BaryCoordEXT;   /* w0, w1, w2 */

    /* Interpolate UV */
    vec2 uv0 = draw.uvs.uv[i0];
    vec2 uv1 = draw.uvs.uv[i1];
    vec2 uv2 = draw.uvs.uv[i2];
    vec2 uv  = bary.x * uv0 + bary.y * uv1 + bary.z * uv2;

    /* Material lookup */
    uint mat_id = draw.material_id;
    /* (MaterialData fetched via BDA — omitted for brevity) */
    uint albedo_tex = /* mat.albedo_texture */ mat_id;  /* placeholder */

    out_color = texture(textures[nonuniformEXT(albedo_tex)], uv);
}
```

**Slang equivalent** — `[[vk::buffer_reference]]` structs are first-class Slang types with generic and reflection support, `ParameterBlock<BindlessScene>` unifies the bindless texture array and visibility buffer in one typed block, and `SV_Barycentrics` exposes hardware barycentric weights portably across Vulkan (`VK_KHR_fragment_shader_barycentric`) and D3D12 SM 6.1 without a manual extension pragma.

```slang
// File: shade.slang
// Visibility-buffer full-screen deferred texturing — matches shade.frag
// Slang improvements:
//   - [[vk::buffer_reference]] structs are typed, generic-compatible, and IDE-navigable,
//     unlike GLSL's opaque buffer_reference layout qualifier
//   - ParameterBlock<BindlessScene> groups the bindless texture array, shared sampler,
//     and visibility buffer into a single type-checked descriptor set binding
//   - SV_Barycentrics: Slang emits BuiltIn BaryCoordKHR in SPIR-V automatically;
//     NonUniformResourceIndex() replaces nonuniformEXT() with portable semantics

// --- Buffer device address types (64-bit GPU pointers) ---
[[vk::buffer_reference, vk::buffer_reference_align(4)]]
struct IndexBufRef    { uint   idx[]; };
[[vk::buffer_reference, vk::buffer_reference_align(4)]]
struct PositionBufRef { float3 pos[]; };
[[vk::buffer_reference, vk::buffer_reference_align(4)]]
struct UVBufRef       { float2 uv[];  };

[[vk::buffer_reference, vk::buffer_reference_align(8)]]
struct DrawData {
    IndexBufRef    indices;
    PositionBufRef positions;
    UVBufRef       uvs;
    uint           material_id;
};

// Array of per-draw BDA structs, accessed via 64-bit base address
[[vk::buffer_reference, vk::buffer_reference_align(8)]]
struct DrawDataArray { DrawData draws[]; };

struct PushConstants {
    DrawDataArray draws;        // 64-bit GPU address of per-draw data array
    uint64_t      material_buf; // MaterialData[] BDA (omitted for brevity)
};

struct BindlessScene {
    Texture2D<float4>  textures[];  // runtime-size bindless texture array
    SamplerState       sampler;
    Texture2D<uint2>   vis_buf;     // R32G32_UINT visibility buffer from Pass 1
};

[[vk::push_constant]]         PushConstants    g_pc;
ParameterBlock<BindlessScene> scene;

[shader("fragment")]
float4 main(float4 frag_coord : SV_Position,
            float3 bary       : SV_Barycentrics) : SV_Target {
    int2  coord = int2(frag_coord.xy);
    uint2 vis   = scene.vis_buf.Load(int3(coord, 0));

    uint draw_id = vis.x;
    uint prim_id = vis.y;

    // BDA pointer arithmetic — fully type-checked by the Slang compiler
    DrawData draw = g_pc.draws.draws[draw_id];

    uint i0 = draw.indices.idx[prim_id * 3 + 0];
    uint i1 = draw.indices.idx[prim_id * 3 + 1];
    uint i2 = draw.indices.idx[prim_id * 3 + 2];

    // SV_Barycentrics: Slang requests VK_KHR_fragment_shader_barycentric
    // automatically and emits BuiltIn BaryCoordKHR in the SPIR-V output
    float2 uv = bary.x * draw.uvs.uv[i0]
              + bary.y * draw.uvs.uv[i1]
              + bary.z * draw.uvs.uv[i2];

    // NonUniformResourceIndex(): Slang emits OpNonUniform decoration automatically
    // (equivalent to GLSL nonuniformEXT) — no explicit extension pragma required
    uint albedo_id = draw.material_id;  // real code chases BDA to MaterialData
    return scene.textures[NonUniformResourceIndex(albedo_id)].Sample(scene.sampler, uv);
}
```

The key advantage over a G-buffer: only pixels that survive the final depth test are shaded. On scenes where many triangles overlap (e.g., dense foliage, crowds), this dramatically reduces fragment shader invocations and memory bandwidth. [Source](http://filmicworlds.com/blog/visibility-buffer-rendering-with-material-graphs/)

---

## Cone Culling in Mesh Shaders

Frustum culling eliminates meshlets outside the camera frustum, but many meshlets inside the frustum face away from the camera and will produce zero visible fragments. **Backface cone culling** rejects these meshlets before the mesh shader even launches, saving rasterisation and fragment shader work. [Source](https://github.com/zeux/meshoptimizer#mesh-shading)

### Cone Representation

`meshopt_computeMeshletBounds()` computes, for each meshlet, a bounding cone that encompasses all triangle face normals:

```c
/* meshoptimizer/src/clusterizer.cpp */
meshopt_Bounds bounds = meshopt_computeMeshletBounds(
    meshlet_vertices + m.vertex_offset,
    meshlet_triangles + m.triangle_offset,
    m.triangle_count,
    positions, vertex_count, sizeof(float) * 3);

/* bounds fields relevant to cone culling:
     bounds.cone_apex[3]   — world-space apex of the normal cone
     bounds.cone_axis[3]   — unit normal pointing away from the cluster face
     bounds.cone_cutoff    — cos(half-angle): reject if all normals face away
     bounds.cone_cutoff == -1  (stored as INT8_MIN in the packed form) →
       degenerate meshlet (e.g. coplanar faces all facing the same way at 180°),
       always consider visible */
```

[Source](https://github.com/zeux/meshoptimizer/blob/master/src/clusterizer.cpp)

### Culling Test

A meshlet is backface-culled if the vector from any camera position to the cone apex, projected onto the cone axis, is greater than `cone_cutoff`. In practice: if `dot(normalize(camera_pos - cone_apex), cone_axis) >= cone_cutoff`, every triangle in the meshlet faces away from the camera. [Source](https://github.com/zeux/meshoptimizer#mesh-shading)

### GLSL Task Shader — Frustum + Cone Culling Combined

The task shader below combines frustum culling (sphere test) with cone culling in a single invocation, avoiding two separate ballot passes:

```glsl
/* task_cull.task.glsl */
#version 460
#extension GL_EXT_mesh_shader : require

layout(local_size_x = 32) in;

struct MeshletBounds {
    vec3  bounds_center;
    float bounds_radius;
    vec3  cone_apex;
    vec3  cone_axis;
    float cone_cutoff;   /* -1.0 = always visible (degenerate) */
};

layout(set = 0, binding = 0) readonly buffer BoundsBuffer {
    MeshletBounds bounds[];
};

layout(push_constant) uniform CullPC {
    mat4  view_proj;
    vec4  frustum_planes[6];
    vec3  camera_pos;
    uint  meshlet_count;
};

taskPayloadSharedEXT struct {
    uint meshlet_indices[32];
    uint count;
} payload;

shared uint visible_count;

bool frustum_cull_sphere(vec3 center, float radius) {
    for (int i = 0; i < 6; i++) {
        if (dot(frustum_planes[i], vec4(center, 1.0)) < -radius)
            return false;
    }
    return true;
}

bool cone_cull(MeshletBounds b) {
    /* Degenerate: always visible */
    if (b.cone_cutoff < -0.999) return false;

    /* Vector from cone apex to camera */
    vec3 to_camera = normalize(camera_pos - b.cone_apex);

    /* If the camera lies within the backface cone, cull */
    return dot(to_camera, b.cone_axis) >= b.cone_cutoff;
}

void main() {
    uint tid        = gl_LocalInvocationID.x;
    uint meshlet_id = gl_WorkGroupID.x * 32 + tid;

    if (tid == 0) visible_count = 0;
    barrier();

    if (meshlet_id < meshlet_count) {
        MeshletBounds b = bounds[meshlet_id];

        bool visible = frustum_cull_sphere(b.bounds_center, b.bounds_radius);
        if (visible) visible = !cone_cull(b);  /* skip cone test if already culled */

        if (visible) {
            uint slot = atomicAdd(visible_count, 1);
            payload.meshlet_indices[slot] = meshlet_id;
        }
    }

    barrier();
    if (tid == 0) payload.count = visible_count;

    EmitMeshTasksEXT(payload.count, 1, 1);
}
```

**Slang equivalent** — `[shader("amplification")]` replaces the GLSL `GL_EXT_mesh_shader` extension pragma; `ParameterBlock<MeshletScene>` gives type-safe, reflection-visible access to the meshlet bounds buffer; and `[Differentiable]` on `cone_cull()` makes the cone cutoff angle a differentiable parameter learnable via `bwd_diff()`.

```slang
// File: task_cull.slang
// Task (amplification) shader: frustum + cone culling — matches task_cull.task.glsl
// Slang improvements:
//   - [shader("amplification")] is the standard Slang entry-point attribute; no
//     extension pragma or #extension directive needed
//   - ParameterBlock<MeshletScene> provides type-safe, reflection-friendly scene binding
//     usable with Slang's automatic descriptor set layout generation
//   - [Differentiable] on cone_cull() enables bwd_diff() to compute gradients w.r.t.
//     the cone_cutoff parameter for learning per-cluster visibility classifiers

struct MeshletBounds {
    float3 bounds_center;
    float  bounds_radius;
    float3 cone_apex;
    float3 cone_axis;
    float  cone_cutoff;  // -1.0 = degenerate (always visible)
};

struct MeshletScene {
    StructuredBuffer<MeshletBounds> bounds;
};

struct CullPC {
    float4x4 view_proj;
    float4   frustum_planes[6];
    float3   camera_pos;
    uint     meshlet_count;
};

struct TaskPayload {
    uint meshlet_indices[32];
    uint count;
};

ParameterBlock<MeshletScene>    scene;
[[vk::push_constant]] CullPC   g_pc;

bool frustum_cull_sphere(float3 center, float radius) {
    [ForceUnroll]
    for (int i = 0; i < 6; i++) {
        if (dot(g_pc.frustum_planes[i], float4(center, 1.0)) < -radius)
            return false;
    }
    return true;
}

// [Differentiable]: enables bwd_diff(cone_cull) to propagate gradients through
// the dot-product comparison w.r.t. cone_cutoff — useful when tuning meshlet
// bounds offline for a given camera distribution.
[Differentiable]
bool cone_cull(MeshletBounds b, float3 camera_pos) {
    if (b.cone_cutoff < -0.999f) return false;          // degenerate: always visible
    float3 to_camera = normalize(camera_pos - b.cone_apex);
    return dot(to_camera, b.cone_axis) >= b.cone_cutoff; // true → all faces back, cull
}

groupshared uint gs_visible_count;

[shader("amplification")]
[numthreads(32, 1, 1)]
void main(uint3 group_id : SV_GroupID,
          uint  local_id : SV_GroupIndex,
          out TaskPayload payload) {
    uint meshlet_id = group_id.x * 32 + local_id;

    if (local_id == 0) gs_visible_count = 0;
    GroupMemoryBarrierWithGroupSync();

    if (meshlet_id < g_pc.meshlet_count) {
        MeshletBounds b = scene.bounds[meshlet_id];

        bool visible = frustum_cull_sphere(b.bounds_center, b.bounds_radius);
        if (visible) visible = !cone_cull(b, g_pc.camera_pos);

        if (visible) {
            uint slot;
            InterlockedAdd(gs_visible_count, 1u, slot);
            payload.meshlet_indices[slot] = meshlet_id;
        }
    }

    GroupMemoryBarrierWithGroupSync();
    if (local_id == 0) payload.count = gs_visible_count;

    // DispatchMesh: Slang's equivalent of EmitMeshTasksEXT; emits the payload
    // and launches exactly gs_visible_count mesh shader workgroups
    DispatchMesh(gs_visible_count, 1, 1, payload);
}
```

Note: cone culling is only exact for orthographic projections. Under perspective, the correct test requires checking whether the camera is outside the *apex* half-space, which the `cone_apex` + `cone_axis` formulation above handles correctly. [Source](https://github.com/zeux/meshoptimizer#mesh-shading)

---

## Cluster LOD Hierarchy

Nanite (Unreal Engine 5) demonstrated that per-cluster LOD selection — where each meshlet cluster can be drawn at a different level of detail depending on its screen-space footprint — eliminates the discrete LOD popping and over-tessellation that plague traditional object-level LOD. The key insight is that LOD transitions happen at cluster boundaries, not mesh boundaries, so nearby clusters of the same object can be at different detail levels simultaneously. [Source](https://advances.realtimerendering.com/s2021/Karis_Nanite_SIGGRAPH_Advances_2021_final.pdf)

### DAG Structure

Clusters are organised into a directed acyclic graph (DAG). Each node (cluster group) stores two error bounds as projected screen-space error in pixels:

- `self_error`: the geometric error introduced by this level of simplification relative to the original mesh.
- `parent_error`: the error of the coarser parent group that replaces this group at greater distance.

The LOD selection invariant is: **draw a cluster at this level if `parent_error > threshold AND self_error <= threshold`**. This guarantees exactly one level of the DAG is selected for each region of the mesh, with no gaps and no overlaps. [Source](https://advances.realtimerendering.com/s2021/Karis_Nanite_SIGGRAPH_Advances_2021_final.pdf)

### CPU-Side Hierarchy with meshoptimizer

Building the DAG requires iterative simplification. `meshopt_simplify()` reduces triangle count while minimising geometric error, returning an error metric that maps directly to the `self_error` field:

```c
/* Build one LOD level from the previous */
float lod_error = 0.0f;
size_t target_indices = index_count / 2;   /* 50% reduction per level */

size_t new_index_count = meshopt_simplify(
    simplified_indices,       /* output index buffer */
    indices, index_count,     /* input */
    positions, vertex_count, sizeof(float) * 3,
    target_indices,
    1e-2f,                    /* error threshold (relative to mesh AABB) */
    0,                        /* options: 0 = default */
    &lod_error);              /* out: actual geometric error achieved */

/* lod_error is in mesh-local units; convert to pixels at a reference distance
   to get a screen-space error threshold for the GPU LOD test */
float screen_error = lod_error * projection_scale / reference_distance;
```

[Source](https://github.com/zeux/meshoptimizer#simplification)

### GPU LOD Selection in the Task Shader

Each cluster carries its `self_error` and `parent_error` in the persistent GPU scene buffer. The task shader performs the LOD test before deciding whether to emit a mesh shader workgroup:

```glsl
/* lod_task.task.glsl (extends the cone-cull task shader) */
struct ClusterData {
    vec3  bounds_center;
    float bounds_radius;
    vec3  cone_apex;
    vec3  cone_axis;
    float cone_cutoff;
    float self_error;    /* screen-space error at this LOD level  */
    float parent_error;  /* screen-space error of coarser parent  */
};

layout(push_constant) uniform CullPC {
    /* ... frustum/cone fields from earlier ... */
    float lod_threshold;   /* pixels: typ. 1.0 for sub-pixel error */
    vec3  camera_pos;
};

float projected_error(float world_error, vec3 center) {
    /* Project a world-space error distance to screen pixels.
       Assumes a vertical FoV of ~60° and a 1080p display. */
    float dist = max(length(camera_pos - center), 0.001);
    return world_error * 1080.0 / (2.0 * dist * tan(radians(30.0)));
}

bool lod_select(ClusterData c) {
    float se = projected_error(c.self_error,   c.bounds_center);
    float pe = projected_error(c.parent_error, c.bounds_center);
    /* Select this cluster if it's fine enough AND its parent is too coarse */
    return (pe > lod_threshold) && (se <= lod_threshold);
}
```

Note: the full Nanite implementation adds streaming — clusters outside a visibility range are evicted from GPU memory and fetched on demand. This section covers the static (fully-resident) case. Streaming integration is an active research area in open-source engines. [Source](https://advances.realtimerendering.com/s2021/Karis_Nanite_SIGGRAPH_Advances_2021_final.pdf)

---

## VK_AMDX_shader_enqueue: Vulkan Workgraphs

Every GPU-driven technique described so far still requires at least one `vkCmdDispatch` or `vkCmdDrawIndexedIndirectCount` command recorded by the CPU. `VK_AMDX_shader_enqueue` eliminates this residual CPU involvement: shaders can enqueue work for other shaders entirely on the GPU, without any host-side command recording round-trip. The concept is identical to D3D12 Work Graphs (both were co-designed by AMD). [Source](https://gpuopen.com/learn/work-graphs-in-vulkan/)

### Core Concepts

A **work graph** is a set of shader nodes connected by typed payload edges. Each node is a compute or mesh shader. When a node shader writes a payload to an output slot, the runtime schedules the downstream node automatically — no CPU involvement, no readback. `vkCmdDispatchGraphAMDX` launches the root node(s) of the graph from the CPU; all subsequent dispatch decisions happen on the GPU. [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_AMDX_shader_enqueue.html)

Current hardware support: RDNA3 (RX 7000 series) with RADV driver on Linux. [Source](https://gpuopen.com/learn/work-graphs-in-vulkan/)

### Extension Setup

```c
/* Feature query */
VkPhysicalDeviceShaderEnqueueFeaturesAMDX enqueue_features = {
    .sType         = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_SHADER_ENQUEUE_FEATURES_AMDX,
    .shaderEnqueue = VK_TRUE,
};

/* Pipeline type: VK_PIPELINE_BIND_POINT_EXECUTION_GRAPH_AMDX */
VkExecutionGraphPipelineCreateInfoAMDX eg_ci = {
    .sType      = VK_STRUCTURE_TYPE_EXECUTION_GRAPH_PIPELINE_CREATE_INFO_AMDX,
    .stageCount = num_nodes,
    .pStages    = node_stage_infos,   /* VkPipelineShaderStageCreateInfo[] */
    .pLibraries = NULL,
    .flags      = 0,
};
VkPipeline exec_graph;
vkCreateExecutionGraphPipelinesAMDX(device, VK_NULL_HANDLE, 1, &eg_ci,
                                    NULL, &exec_graph);
```

### Two-Node Cull → Draw Work Graph

The simplest GPU-driven work graph replaces the `vkCmdDispatch` + barrier + `vkCmdDrawIndexedIndirectCount` triple with two connected nodes:

```glsl
/* Node 0: cull.comp — enqueues mesh draw payloads */
#version 460
#extension GL_AMDX_shader_enqueue : require

layout(local_size_x = 64) in;

/* Payload type sent to the draw node */
struct DrawPayload {
    uint meshlet_id;
};

/* Output queue to node "draw_node" */
layout(max_payloads = 1024)
    coalescing nodePayloadAMDX(DrawPayload) draw_queue;  /* → draw_node */

layout(push_constant) uniform PC { uint meshlet_count; };

void main() {
    uint id = gl_GlobalInvocationID.x;
    if (id >= meshlet_count) return;

    if (passes_cull_tests(id)) {
        DrawPayload p;
        p.meshlet_id = id;
        /* Enqueue one mesh shader workgroup for this meshlet */
        EnqueueNodePayloadsAMDX(draw_queue, p);
    }
}
```

```glsl
/* Node 1: draw_node.mesh — receives payload, outputs geometry */
#version 460
#extension GL_AMDX_shader_enqueue : require
#extension GL_EXT_mesh_shader      : require

layout(local_size_x = 64) in;
layout(triangles, max_vertices = 64, max_primitives = 124) out;

/* Input payload from the cull node */
layout() nodePayloadAMDX(DrawPayload) in_payload;

void main() {
    uint meshlet_id = in_payload.meshlet_id;
    /* ... emit vertices and triangles as in a normal mesh shader ... */
}
```

The CPU dispatch becomes a single call:

```c
VkDispatchGraphInfoAMDX graph_info = {
    .nodeIndex = cull_node_index,
    .payloadCount = { .staticCount = 1 },
    /* seed payload: global meshlet count */
};
vkCmdDispatchGraphAMDX(cmd, scratch_buf, scratch_offset, &graph_info);
```

[Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_AMDX_shader_enqueue.html)

Work graphs are currently AMD-vendor-only. Khronos is tracking the concept for a future cross-vendor extension, analogous to how `VK_NV_mesh_shader` eventually became `VK_EXT_mesh_shader`. [Source](https://gpuopen.com/learn/work-graphs-in-vulkan/)

---

## Pipeline Statistics Queries

Culling is only worthwhile if it actually removes significant work. `VK_QUERY_TYPE_PIPELINE_STATISTICS` provides hardware counters that measure how many primitives survived the clipping stage and how many fragment shader invocations were executed, allowing cull efficiency to be computed in-frame without GPU readback stalls. [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VkQueryPipelineStatisticFlagBits.html)

### Relevant Flag Bits

| Flag | Meaning |
|------|---------|
| `VK_QUERY_PIPELINE_STATISTIC_CLIPPING_INVOCATIONS_BIT` | Primitives submitted to the clipper (after vertex/mesh shading) |
| `VK_QUERY_PIPELINE_STATISTIC_CLIPPING_PRIMITIVES_BIT` | Primitives that survived clipping (sent to rasteriser) |
| `VK_QUERY_PIPELINE_STATISTIC_FRAGMENT_SHADER_INVOCATIONS_BIT` | Total fragment shader invocations |
| `VK_QUERY_PIPELINE_STATISTIC_VERTEX_SHADER_INVOCATIONS_BIT` | Vertex shader invocations (0 for mesh shader pipelines) |
| `VK_QUERY_PIPELINE_STATISTIC_TASK_SHADER_INVOCATIONS_BIT_EXT` | Task shader invocations (requires `VK_EXT_mesh_shader`) |
| `VK_QUERY_PIPELINE_STATISTIC_MESH_SHADER_INVOCATIONS_BIT_EXT` | Mesh shader invocations |

[Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VkQueryPipelineStatisticFlagBits.html)

**Cull efficiency** = `clipping_primitives / clipping_invocations`. Values below 0.5 indicate that more than half of all primitives are being clipped away (expected behaviour for good frustum + backface culling). Values near 1.0 suggest culling is not working.

### C Code — Query Pool, Recording, and Result Readback

```c
/* --- 1. Create query pool ------------------------------------------- */
VkQueryPoolCreateInfo qp_ci = {
    .sType      = VK_STRUCTURE_TYPE_QUERY_POOL_CREATE_INFO,
    .queryType  = VK_QUERY_TYPE_PIPELINE_STATISTICS,
    .queryCount = 1,   /* one stats query wrapping the whole indirect draw */
    .pipelineStatistics =
        VK_QUERY_PIPELINE_STATISTIC_CLIPPING_INVOCATIONS_BIT  |
        VK_QUERY_PIPELINE_STATISTIC_CLIPPING_PRIMITIVES_BIT   |
        VK_QUERY_PIPELINE_STATISTIC_FRAGMENT_SHADER_INVOCATIONS_BIT,
};
VkQueryPool stats_pool;
vkCreateQueryPool(device, &qp_ci, NULL, &stats_pool);

/* --- Result struct matching the enabled bits (order = bit order) ----- */
typedef struct {
    uint64_t clipping_invocations;
    uint64_t clipping_primitives;
    uint64_t fragment_invocations;
} PipelineStats;

/* --- 2. Per-frame recording ------------------------------------------ */
vkCmdResetQueryPool(cmd, stats_pool, 0, 1);
vkCmdBeginQuery(cmd, stats_pool, 0, 0 /* flags */);

    /* The indirect draw under measurement: */
    vkCmdDrawIndexedIndirectCount(cmd, draw_buf, 0,
                                  count_buf, 0,
                                  max_draws,
                                  sizeof(VkDrawIndexedIndirectCommand));

vkCmdEndQuery(cmd, stats_pool, 0);

/* --- 3. Readback (next frame, to avoid pipeline stall) --------------- */
PipelineStats stats = {0};
vkGetQueryPoolResults(
    device, stats_pool,
    0, 1,                              /* first query, count */
    sizeof(PipelineStats), &stats,     /* data + stride */
    sizeof(PipelineStats),
    VK_QUERY_RESULT_64_BIT | VK_QUERY_RESULT_WAIT_BIT);

float cull_efficiency = (stats.clipping_invocations > 0)
    ? (float)stats.clipping_primitives / (float)stats.clipping_invocations
    : 1.0f;

printf("Cull efficiency: %.1f%%  (%llu → %llu primitives, %llu frag invocations)\n",
       cull_efficiency * 100.0f,
       (unsigned long long)stats.clipping_invocations,
       (unsigned long long)stats.clipping_primitives,
       (unsigned long long)stats.fragment_invocations);
```

The `VK_QUERY_RESULT_WAIT_BIT` flag blocks until results are available. For production use, store results to a host-visible buffer with `vkCmdCopyQueryPoolResults` and read them one or two frames later to avoid stalling the GPU pipeline. [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/vkGetQueryPoolResults.html)

Pipeline statistics queries are supported on all Vulkan 1.0 hardware that exposes the `pipelineStatisticsQuery` physical device feature. They are disabled on some mobile GPUs. Check `VkPhysicalDeviceFeatures.pipelineStatisticsQuery` before creating the pool. [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VkPhysicalDeviceFeatures.html)

---

## Roadmap

### Near-term (6–12 months)

- **Vulkan Roadmap 2026 milestone mandates multi-draw indirect and shader draw parameters** as required features for conformant drivers, closing hardware gaps that previously made GPU-driven pipelines opt-in only. [Source](https://www.phoronix.com/news/Vulkan-Roadmap-2026)
- **`VK_EXT_device_generated_commands` (DGC) is shipping in production drivers**: the extension allows GPU-side generation of full command sequences including pipeline/state changes per draw, going beyond `vkCmdDrawIndexedIndirectCount` which only selects draw arguments. Practical adoption details were presented at Vulkanised 2025. [Source](https://rg3.name/202503111630.html)
- **AMD DGF (Discrete Geometry Format) super-compression** for meshlet geometry is now available via AMD GPUOpen, reducing the memory footprint of large meshlet scene representations. [Source](https://gpuopen.com/learn/introducing-amd-dgf-supercompression/)
- **Vulkan Roadmap 2026 requires higher descriptor and shader interface limits**, enabling larger bindless descriptor arrays (textures, buffers) without per-vendor workarounds. [Source](https://docs.vulkan.org/spec/latest/appendices/roadmap.html)

### Medium-term (1–3 years)

- **GPU Work Graphs with mesh nodes (`VK_AMDX_shader_enqueue`)**: AMD's experimental extension adds mesh nodes to work graphs, allowing a single payload dispatch to spawn both compute and mesh shader work entirely GPU-side — a deeper form of GPU-driven rendering than indirect draw. Khronos is tracking this for eventual cross-vendor promotion. [Source](https://www.khronos.org/news/archives/amd-blog-gpu-work-graphs-mesh-node-are-now-in-vulkan)
- **Standardised Variable Rate Shading (VRS) mandated by Roadmap 2026**: combined with GPU-driven culling, per-tile VRS rates computed by a compute shader can skip shading in low-detail regions, reducing fragment load on the surviving draw set. [Source](https://videocardz.com/newz/vulkan-api-sets-2026-feature-baseline-roadmap-milestone-with-variable-rate-shading)
- **Wider engine adoption of two-phase occlusion culling with HZB (Hierarchical Z-Buffer)**: Bevy, Godot 4, and other open-source engines are expanding GPU-driven passes with hierarchical depth reprojection from the previous frame to reject occluded meshlets before mesh shader launch. Note: needs verification for specific merge/release timelines.
- **Indirect Execution Sets (DGC `vkUpdateIndirectExecutionSetPipelineEXT`)**: enabling GPU-driven shader switching per draw without CPU rebinding, closing the last CPU-driven bottleneck for material diversity at large scene scale. [Source](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_device_generated_commands.html)
- **Reconvergence guarantees (Roadmap 2026 requirement)** will make subgroup ops inside mesh and task shaders more predictable, enabling tighter meshlet culling algorithms that rely on ballot/vote intrinsics. [Source](https://www.phoronix.com/news/Vulkan-Roadmap-2026)

### Long-term

- **GPU Work Graphs as first-class Vulkan extension**: if `VK_AMDX_shader_enqueue` reaches multi-vendor agreement, a standardised work graph extension could replace the indirect dispatch + indirect draw pattern entirely — the GPU would traverse a scene DAG, cull, and shade without any host-side dispatch. Note: needs verification — currently AMD-vendor-only.
- **Ray-traced occlusion replacing rasterised HZB culling**: as ray tracing hardware becomes cheaper per-ray, some engines may use sparse ray queries (inline `rayQueryEXT`) inside compute culling shaders to get accurate per-object visibility without a separate depth pre-pass or HZB construction step.
- **Unified GPU scene graphs in OS compositor**: long-term, Wayland compositors (e.g., weston, cosmic-comp) may adopt persistent GPU scene representations similar to game-engine GPU-driven passes for managing surface trees, reducing CPU work per-frame in desktop compositing. Note: needs verification — speculative direction.

---

## Integrations

- **Ch19 (Vulkan Architecture)** — indirect draw commands (`vkCmdDrawIndexedIndirectCount`) and compute dispatch are core Vulkan; barrier types (INDIRECT_COMMAND_READ, SHADER_WRITE) are covered in the synchronisation chapter
- **Ch20 (SPIR-V)** — mesh and task shaders compile to SPIR-V with capability `MeshShadingEXT`; the task payload is a SPIR-V `TaskPayloadWorkgroupEXT` variable
- **Ch22 (RADV)** — RADV implements mesh shaders on RDNA2+ via the ACO backend's NGG (Next Generation Geometry) path
- **Ch23 (ANV)** — ANV implements mesh shaders on Xe-HPG using the EU thread dispatch path
- **Ch145 (GPU Profiling)** — GPU-driven culling effectiveness is measured with `vkCmdWriteTimestamp2` around the cull dispatch and the indirect draw; AMD RGP shows per-meshlet occupancy
- **Ch152 (Rust GPU)** — wgpu does not yet expose mesh shaders; `ash` is required for `VK_EXT_mesh_shader`
- **Ch127 (Mesh Shaders / VRS)** — task and mesh shader pipeline details, variable-rate shading integration with GPU-driven tile classification; VRS rate image generated by a preceding compute pass maps directly onto the surviving draw set from GPU culling
- **Ch157 (Descriptor Binding — Bindless / BDA)** — `VK_EXT_descriptor_indexing` partially-bound arrays and `VK_KHR_buffer_device_address` 64-bit pointers used in the visibility buffer shading pass and the cluster LOD GPU buffers
- **Ch135 (Ray Tracing — TLAS Build as GPU-Driven Op)** — acceleration structure builds (`vkCmdBuildAccelerationStructuresKHR`) consume the same persistent instance buffer updated by the GPU culling pass; TLAS compaction and update share the indirect dispatch pattern
- **Ch97 (UE5 Nanite)** — production cluster LOD DAG implementation; the `self_error`/`parent_error` selection criterion described in this chapter derives directly from the Nanite SIGGRAPH 2021 presentation; streaming and virtual geometry are covered in Ch97
- **Ch133 (Vulkan Compute Queues — Async Cull Passes)** — the Hi-Z pyramid build and frustum/occlusion cull dispatches are natural candidates for an async compute queue, overlapping with graphics rasterisation; queue family ownership transfer and semaphore signalling for the depth pyramid are detailed in Ch133

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
