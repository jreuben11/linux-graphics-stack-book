# Chapter 154: GPU-Driven Rendering: Indirect Draw, Meshlets, and GPU Culling

**Target audiences**: Graphics application developers implementing high-performance rendering with Vulkan; engine developers migrating from CPU-driven to GPU-driven architectures; and engineers studying the data structures behind mesh shaders, indirect dispatch, and GPU-side culling.

---

## Table of Contents

1. [Introduction](#introduction)
2. [The CPU-GPU Bottleneck: Why GPU-Driven?](#the-cpu-gpu-bottleneck-why-gpu-driven)
3. [Indirect Draw Commands](#indirect-draw-commands)
4. [Multi-Draw Indirect and Draw Count](#multi-draw-indirect-and-draw-count)
5. [GPU Culling with Compute Shaders](#gpu-culling-with-compute-shaders)
6. [Meshlets and Mesh Shaders](#meshlets-and-mesh-shaders)
7. [Task and Mesh Shader Pipeline](#task-and-mesh-shader-pipeline)
8. [Persistent GPU Scene Representation](#persistent-gpu-scene-representation)
9. [Bindless Resources](#bindless-resources)
10. [Integrations](#integrations)

---

## Introduction

Traditional rendering architectures are CPU-driven: the CPU iterates over scene objects, culls them, sets per-draw state (descriptor sets, push constants), and submits `vkCmdDraw` calls. At large scene scales this bottleneck becomes critical — a scene with 100,000 objects requires 100,000 CPU draw calls even if 90,000 are offscreen.

GPU-driven rendering inverts this: the GPU holds a persistent scene representation, runs its own culling in a compute shader, and generates the draw commands itself via `vkCmdDrawIndexedIndirectCount`. The CPU submits one compute dispatch and one indirect draw call per frame regardless of scene size.

This chapter covers the Vulkan API for indirect drawing, the GPU culling pipeline, and the mesh shader (task + mesh) model for meshlet-based rendering — a hardware feature on NVIDIA Turing+, AMD RDNA2+, and Intel Xe. Key implementations studied: `vkguide.dev` GPU-driven tutorial, Niagara (Arseny Kapoulkine), Bevy's GPU culling pass.

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
