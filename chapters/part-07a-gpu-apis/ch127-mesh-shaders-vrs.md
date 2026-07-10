# Chapter 127: Mesh Shaders and Variable Rate Shading

This chapter targets **Vulkan application developers**, **game engine developers**, **GPU hardware architects**, and **shader programmers** who want to understand and apply the two most significant programmable pipeline extensions of the RDNA2/Ampere era: mesh shaders (`VK_EXT_mesh_shader`) and variable rate shading (`VK_KHR_fragment_shading_rate`). It examines each extension's API surface, the hardware execution model that backs it, Linux driver support in RADV, ANV, and NVK, and how the two extensions interoperate to achieve Nanite-style rendering densities without sacrificing GPU throughput.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [The Traditional Vertex Pipeline and Its Limits](#2-the-traditional-vertex-pipeline-and-its-limits)
3. [Mesh Shaders: Task and Mesh Stages](#3-mesh-shaders-task-and-mesh-stages)
4. [Mesh Shader Programming](#4-mesh-shader-programming)
5. [Hardware Support on Linux](#5-hardware-support-on-linux)
6. [Variable Rate Shading: Motivation](#6-variable-rate-shading-motivation)
7. [VRS API Usage](#7-vrs-api-usage)
8. [VRS Hardware Support and Driver Implementation](#8-vrs-hardware-support-and-driver-implementation)
9. [Combined Usage: VRS + Mesh Shaders](#9-combined-usage-vrs--mesh-shaders)
10. [Real-World Usage on Linux](#10-real-world-usage-on-linux)
11. [Integrations](#11-integrations)

---

## 1. Introduction

Modern real-time rendering places contradictory demands on the GPU pipeline. Scenes contain billions of triangles — generated procedurally, streamed from disk, or assembled from micro-mesh clusters — yet frames must finish in under 8 ms for 120 Hz VR. At the same time, the pixel budget is enormous: a 4K monitor at 120 Hz needs the fragment shader to evaluate roughly 50 billion shaded samples per second, and many of those samples contribute marginally — a featureless sky dome, a distant shadow receiver, a peripheral-vision region where the human eye resolves less detail.

Two Vulkan extensions attack these problems from opposite directions:

- **`VK_EXT_mesh_shader`** ([spec](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_mesh_shader.html)) replaces the fixed Input Assembler → Vertex → Geometry/Tessellation → Rasterizer chain with a two-stage compute-like pipeline: an optional *task shader* that amplifies or culls work, and a *mesh shader* that writes vertices and primitives directly to the rasterizer. This eliminates the IA bottleneck, enables GPU-driven geometry at Nanite scale, and supports per-cluster culling without CPU round-trips.

- **`VK_KHR_fragment_shading_rate`** ([spec](https://registry.khronos.org/vulkan/specs/latest/man/html/VkPhysicalDeviceFragmentShadingRateFeaturesKHR.html)) lets the application reduce the frequency of fragment shader invocations in regions of the framebuffer where full resolution is unnecessary. A single control image can make the GPU shade 1 sample per 4×4-pixel tile in featureless regions while keeping 1:1 resolution on foreground geometry — providing a performance multiplier with minimal perceptual impact.

Both extensions are fully supported on RDNA2+ via RADV, Gen12.5+/Xe via ANV, and on NVIDIA Ampere+ via both the proprietary driver and (as of 2025–2026) the open-source NVK driver. This chapter covers their design, their API contracts, their hardware execution models, and how to combine them in production.

---

## 2. The Traditional Vertex Pipeline and Its Limits

### 2.1 Fixed Pipeline Stages

The canonical Vulkan pipeline for a draw call flows through fixed hardware blocks:

```
Input Assembler (IA) → Vertex Shader (VS) → [Hull Shader (HS) / Domain Shader (DS)]
  → [Geometry Shader (GS)] → Rasterizer → Fragment Shader (FS)
```

Each stage is optional except VS and FS, but the fixed IA and the in-order output semantics of GS impose structural constraints that have plagued GPU-driven rendering for a decade.

### 2.2 Input Assembler Bottleneck

The Input Assembler reads index and vertex buffers, expands index lists, and generates VS invocations. Even with `VkDrawIndexedIndirectCommand` and `vkCmdDrawIndexedIndirect`, the IA is fundamentally sequential in the sense that index fetch bandwidth is a shared resource. For scenes with millions of small meshlets, thousands of individual `vkCmdDraw*` calls generate substantial CPU-side command overhead, and the IA cannot skip invisible primitives — all culling must have happened before the draw is issued.

GPU-driven rendering via indirect draws alleviates the CPU side but does not help the IA: every triangle still passes through the fixed fetch pipeline whether it is visible or not.

### 2.3 Geometry Shaders: Use Cases and Limitations

The Geometry Shader (GS) stage sits between the vertex/tessellation stages and the rasterizer, running once per input primitive and emitting zero or more output primitives. Despite its reputation as a performance trap, it enables several effects that are awkward to express any other way.

**Legitimate use cases.** The historically compelling reason to reach for a GS was **layered rendering in a single pass**: a GS can route each emitted primitive to a different framebuffer layer by writing `gl_Layer`, so one draw can populate all six faces of a cubemap shadow map, or all cascades of a cascaded shadow map, without re-submitting geometry per layer. This was long the *killer* use case, though `VK_KHR_multiview` now covers most layered-rendering needs more efficiently and portably. A GS is also natural for **billboard and point expansion** — taking a single input point and emitting a camera-facing quad — which drives particle systems, GPU sprites, and impostor rendering. It supports **debug visualization** by emitting line primitives from surface data: per-vertex normal and tangent lines, or wireframe overlays generated from triangle edges. Finally, because a GS has access to a primitive's full set of vertices with adjacency, it can perform **silhouette detection** — comparing face normals across shared edges to find silhouette edges — which underlies toon outline rendering and shadow-volume/fin extrusion techniques.

**Performance traps.** These capabilities come at a steep cost:

- **Output buffering forces a ring-buffer round-trip**: because a GS emits a variable number of primitives, hardware must stage GS output into a temporary ring buffer in memory before it reaches the rasterizer, creating substantial memory bandwidth pressure that scales with amplification.
- **In-order output serialisation**: the API guarantees that GS output primitives appear in input-primitive order, which forces the hardware to serialise the reassembly of logically independent invocations.
- **High register pressure, low occupancy**: large per-invocation state (the buffered output vertices) means a GS typically launches far fewer simultaneous wavefronts than a VS or FS, throttling occupancy.
- **Limited amplification**: a GS may emit at most 1024 total output components per invocation, capping true geometry amplification well below what modern GPU-driven pipelines demand.

**Mobile and MoltenVK caveat.** Geometry shaders are not universally available. Many mobile Vulkan implementations expose no GS support at all, and MoltenVK on Apple hardware (which has no native geometry-shader stage) likewise omits it — in both cases `VkPhysicalDeviceLimits.maxGeometryOutputVertices == 0` and `VkPhysicalDeviceFeatures.geometryShader` is `VK_FALSE`. Any renderer that must run on those targets cannot depend on the GS stage, making it a poor foundation for portable engines.

For general geometry amplification, culling, and per-cluster LOD, mesh shaders (§3) replace the geometry shader entirely, discarding the ring-buffer staging and in-order constraints that make the GS slow.

### 2.4 Tessellation Shaders: Use Cases and Limitations

Where the geometry shader amplifies primitives in a general but slow way, the tessellation pipeline amplifies geometry through **dedicated fixed-function silicon**, subdividing a coarse control mesh into a fine one on the fly.

**The three sub-stages.** Tessellation inserts three sub-stages between the vertex shader and the rasterizer:

1. The **Tessellation Control Shader** (TCS; the *hull shader* in D3D) runs once per control-point patch. Beyond optionally transforming control points, its job is to write the per-edge and interior subdivision factors into the built-ins `gl_TessLevelOuter[4]` and `gl_TessLevelInner[2]`.
2. A **fixed-function tessellator** subdivides an abstract patch domain — triangles, quads, or isolines — according to those factors. It produces a mesh of domain-space vertices but knows nothing of world-space geometry.
3. The **Tessellation Evaluation Shader** (TES; the *domain shader* in D3D) runs once per generated vertex. It receives `gl_TessCoord` (barycentric coordinates for the triangle domain, or UV coordinates for quads/isolines) and interpolates the control points to compute the final vertex position.

**Vulkan API.** A tessellation pipeline sets `VkPipelineTessellationStateCreateInfo` with `patchControlPoints` giving the number of control points per patch, and the input topology must be `VK_PRIMITIVE_TOPOLOGY_PATCH_LIST`. The TCS and TES appear as ordinary `VkPipelineShaderStageCreateInfo` entries (stages `VK_SHADER_STAGE_TESSELLATION_CONTROL_BIT` and `VK_SHADER_STAGE_TESSELLATION_EVALUATION_BIT`), with tessellation factors written from the TCS via the GLSL built-ins above. `VkTessellationDomainOriginStateCreateInfoKHR` selects the domain-origin convention (upper-left vs. lower-left) to match a given source API.

**Use cases.**

1. **Terrain with adaptive LOD** — the classic application. The TCS computes each patch edge's tessellation factor from that edge's screen-space projected length or its distance to the camera: nearby terrain is subdivided into many triangles while distant terrain stays coarse. The critical constraint is **crack-free joins** — two patches sharing an edge must be handed *identical* factors for that edge, which is achieved by deriving the factor from a symmetric function of the shared edge's endpoints so both patches compute the same value.
2. **Displacement mapping** — tessellate a flat or low-poly base mesh, then in the TES sample a heightmap and push each generated vertex along the interpolated surface normal. This is the standard technique for rock surfaces, ocean waves, fine terrain detail, and snow deformation.
3. **Smooth curved surfaces** — evaluate Bézier or B-spline patches, or PN-triangle approximations, in the TES so that characters and CAD models stay smooth at any zoom level. Because the hardware tessellator only understands abstract quad/triangle/isoline domains, each NURBS patch must first be converted to Bézier form (offline or in a compute pass) before it can be evaluated.
4. **Hair, fur, and grass** — tessellate guide curves in the isolines domain into many strand instances, with per-strand variation seeded in the TCS.
5. **Projected-grid water** — build a radial or screen-space grid whose vertex density follows the camera, then displace it by a wave function in the TES.

The following TCS computes distance-based tessellation factors for a quad terrain patch:

```glsl
// TCS: adaptive terrain tessellation
layout(vertices = 4) out;
uniform vec3 uCameraPos;

void main() {
    gl_out[gl_InvocationID].gl_Position = gl_in[gl_InvocationID].gl_Position;
    if (gl_InvocationID == 0) {
        float d = distance(uCameraPos, (gl_in[0].gl_Position.xyz + gl_in[2].gl_Position.xyz) * 0.5);
        float lod = clamp(64.0 / (d * 0.1), 1.0, 64.0);
        gl_TessLevelOuter[0] = gl_TessLevelOuter[1] =
        gl_TessLevelOuter[2] = gl_TessLevelOuter[3] = lod;
        gl_TessLevelInner[0] = gl_TessLevelInner[1] = lod;
    }
}
```

**Performance characteristics.** The fixed-function tessellator is more efficient than a geometry shader — dedicated silicon subdivides the domain with no ring-buffer staging — but it has its own pitfalls. Efficiency is poor at low tessellation factors: a factor of 1 still incurs tessellator setup overhead relative to a plain vertex-shader draw, so tessellating patches that stay coarse wastes throughput. The control shader also runs per control point (up to 32), adding invocation overhead proportional to patch complexity.

**Portability.** Most desktop Vulkan drivers support tessellation, but it is absent on many mobile implementations (Mali prior to Valhall, Adreno prior to the 730 generation). An application must query `VkPhysicalDeviceFeatures.tessellationShader` before creating a tessellation pipeline.

**Modern alternatives.** Mesh shaders with per-meshlet LOD selection (§3) replace adaptive tessellation for general geometry, and compute-driven displacement writing directly into vertex buffers replaces TES displacement. Tessellation nonetheless remains a viable, well-supported choice for terrain and displacement on hardware that lacks mesh-shader support.

### 2.5 GPU-Driven Rendering: Indirect Draws and Their Remaining Limits

`vkCmdDrawIndexedIndirect` ([spec](https://registry.khronos.org/vulkan/specs/latest/man/html/vkCmdDrawIndexedIndirect.html)) allows a GPU-resident buffer to supply draw parameters:

```c
typedef struct VkDrawIndexedIndirectCommand {
    uint32_t indexCount;
    uint32_t instanceCount;
    uint32_t firstIndex;
    int32_t  vertexOffset;
    uint32_t firstInstance;
} VkDrawIndexedIndirectCommand;
```

A compute pass can generate an array of these commands, write them to a `VkBuffer`, and then `vkCmdDrawIndexedIndirectCount` reads the count from a second buffer — no CPU reads the GPU-generated count. This eliminates draw-call CPU overhead and enables GPU-driven LOD selection.

However, fundamental IA constraints remain:
- Every draw still passes through the fixed-function Index Assembler, which fetches index data regardless of visibility.
- Per-draw vertex buffer binding changes or push constant updates require additional CPU commands, scaling poorly with draw count.
- The `firstVertex` model requires the application to allocate vertex ranges in advance; dynamic per-cluster deduplication is impossible.
- Sub-pixel micro-triangles still reach the rasterizer and generate coverage tests, wasting fixed-function hardware on invisible geometry.

### 2.6 Why Nanite Required Mesh Shaders

Unreal Engine 5's Nanite system renders scenes containing billions of source triangles by maintaining a hierarchical cluster DAG and selecting, per-frame, which clusters at which LOD to render. In a traditional pipeline:

- LOD selection and cluster streaming requires a CPU round-trip or a multi-pass indirect scheme that still suffers IA bandwidth.
- Each cluster would require its own draw call or a massive indirect buffer that still traverses all cluster data through the IA.
- Degenerate micro-triangles (sub-pixel clusters) cannot be efficiently rejected before reaching the rasterizer.
- Clusters at different LOD levels have different vertex counts, making pre-allocation of IA index ranges impractical for tens of thousands of clusters.

Mesh shaders solve all four: the task shader selects clusters and their LOD on-GPU, the mesh shader outputs only the triangles needed (with no IA allocation), micro-triangle rejection can be implemented as shader code (`gl_CullPrimitiveEXT`) before any triangle reaches the rasterizer, and variable cluster sizes are handled naturally since the mesh shader declares its output count dynamically via `SetMeshOutputsEXT`.

---

## 3. Mesh Shaders: Task and Mesh Stages

### 3.1 Extension Overview

`VK_EXT_mesh_shader` was ratified in Vulkan 1.3.226 (September 2022) after cross-vendor collaboration among NVIDIA, Valve, Intel, AMD, and ARM ([Khronos announcement](https://vulkan.org/news/auto-21020-fff15aa5e7dafc437c185bd6b4f35b6c)). It supersedes the earlier vendor-specific `VK_NV_mesh_shader`, adding three-dimensional workgroup dispatch, mesh primitive pipeline statistics queries, and mandatory symmetry with DirectX 12 Mesh Shaders.

The new pipeline is:

```
Task Shader (optional) → Mesh Shader → Rasterizer → Fragment Shader
```

No IA. No fixed vertex fetch. The task and mesh shaders read any data they like from storage buffers, and write geometry directly.

### 3.2 Task Shader Stage

The task shader is an optional, compute-like stage. One workgroup runs per "task" — typically one per input meshlet cluster or per coarse spatial cell. Its responsibility is **amplification**: deciding how many mesh shader workgroups to launch, and what read-only data (the *payload*) to pass to each.

The GLSL built-in that terminates the task shader and dispatches mesh workgroups is:

```glsl
void EmitMeshTasksEXT(uint groupCountX, uint groupCountY, uint groupCountZ);
```

This call is mandatory, must execute exactly once under uniform control flow, and acts as a workgroup barrier. The task shader implicitly communicates via the `taskPayloadSharedEXT` global, which is written by the task shader and broadcast read-only to all mesh workgroups it launches ([GLSL_EXT_mesh_shader spec](https://github.khronos.org/Vulkan-Site/glslext/latest/glslext/ext/GLSL_EXT_mesh_shader.html)).

Key limits governing the task stage (queried from `VkPhysicalDeviceMeshShaderPropertiesEXT`):
- `maxTaskWorkGroupCount[3]`: maximum dispatch grid per dimension
- `maxTaskWorkGroupInvocations`: typically 128
- `maxTaskPayloadSize`: maximum payload size in bytes (at least 16 KB)
- `maxTaskPayloadAndSharedMemorySize`: combined payload + shared memory limit

### 3.3 Mesh Shader Stage

The mesh shader runs one workgroup per cluster. Each workgroup:

1. Calls `SetMeshOutputsEXT(vertexCount, primitiveCount)` to declare the number of outputs it will produce (must be called before writing any output).
2. Writes vertex positions and attributes into `gl_MeshVerticesEXT[]`.
3. Writes triangle indices into `gl_PrimitiveTriangleIndicesEXT[]` (uvec3 per triangle).
4. Optionally writes per-primitive attributes into `gl_MeshPrimitivesEXT[]` using the `perprimitiveEXT` qualifier.

Hard limits:
- Up to **256 vertices** per mesh workgroup output
- Up to **256 primitives** per mesh workgroup output
- Maximum output components: queried from `maxMeshOutputComponents`

The SPIR-V opcodes underlying these GLSL functions are `OpSetMeshOutputsEXT` and `OpEmitMeshTasksEXT` ([SPIR-V EXT_mesh_shader specification](https://github.com/KhronosGroup/SPIRV-Registry/blob/main/extensions/EXT/SPV_EXT_mesh_shader.asciidoc)).

### 3.4 Per-Primitive Outputs

The `perprimitiveEXT` qualifier annotates output variables in the mesh shader that the fragment shader reads as flat-interpolated per-primitive data. This is critical for material IDs, object IDs, and — as discussed in §9 — per-primitive shading rates:

```glsl
perprimitiveEXT out gl_MeshPerPrimitiveEXT {
    int  gl_PrimitiveID;
    int  gl_Layer;
    int  gl_ViewportIndex;
    bool gl_CullPrimitiveEXT;
    int  gl_PrimitiveShadingRateEXT;
} gl_MeshPrimitivesEXT[];
```

Setting `gl_CullPrimitiveEXT = true` for a primitive causes the rasterizer to skip it entirely without incurring triangle setup costs.

### 3.5 Draw Commands

Three new Vulkan commands replace `vkCmdDraw*`:

```c
// Direct dispatch: taskCount workgroups in X dimension
vkCmdDrawMeshTasksEXT(VkCommandBuffer cmd,
                      uint32_t groupCountX,
                      uint32_t groupCountY,
                      uint32_t groupCountZ);

// GPU-side draw parameters from buffer
vkCmdDrawMeshTasksIndirectEXT(VkCommandBuffer cmd,
                               VkBuffer buffer, VkDeviceSize offset,
                               uint32_t drawCount, uint32_t stride);

// GPU-side draw count from a second buffer (no CPU visibility into count)
vkCmdDrawMeshTasksIndirectCountEXT(VkCommandBuffer cmd,
                                    VkBuffer buffer, VkDeviceSize offset,
                                    VkBuffer countBuffer, VkDeviceSize countBufferOffset,
                                    uint32_t maxDrawCount, uint32_t stride);
```

`vkCmdDrawMeshTasksIndirectCountEXT` is the key to fully GPU-driven rendering: the number of active meshlet clusters is determined by the GPU culling pass, written to `countBuffer`, and the draw executes without CPU involvement.

---

## 4. Mesh Shader Programming

### 4.1 GLSL Extension Header

Every GLSL mesh or task shader must enable the extension and declare its execution model:

```glsl
#version 450
#extension GL_EXT_mesh_shader : require

// Task shader
layout(local_size_x = 32) in;

// Mesh shader
layout(local_size_x = 128) in;
layout(triangles, max_vertices = 64, max_primitives = 124) out;
```

The `local_size_x` for a mesh shader is typically 128, covering the maximum 124 primitives with one thread each plus slack for vertex processing.

### 4.2 Offline Meshlet Generation with meshoptimizer

The primary tool for converting triangle meshes into meshlet clusters is [meshoptimizer](https://meshoptimizer.org/) by Arseny Kapoulkine. Its C API:

```c
// Step 1: allocate worst-case storage
size_t max_meshlets = meshopt_buildMeshletsBound(
    index_count, max_vertices, max_triangles);  // e.g. 64, 124

meshopt_Meshlet* meshlets         = malloc(sizeof(meshopt_Meshlet) * max_meshlets);
unsigned int*    meshlet_vertices = malloc(sizeof(unsigned int)     * max_meshlets * max_vertices);
unsigned char*   meshlet_triangles= malloc(max_meshlets * max_triangles * 3);

// Step 2: generate meshlets
// cone_weight: 0.0 = pure topology, 1.0 = maximise backface culling cone
size_t meshlet_count = meshopt_buildMeshlets(
    meshlets, meshlet_vertices, meshlet_triangles,
    indices, index_count,
    vertex_positions, vertex_count, vertex_stride,
    max_vertices, max_triangles, /* cone_weight = */ 0.5f);

// meshopt_Meshlet fields:
//   .vertex_offset   — start in meshlet_vertices array
//   .triangle_offset — start in meshlet_triangles array
//   .vertex_count    — vertices used (≤ max_vertices)
//   .triangle_count  — triangles (≤ max_triangles)

// Step 3: per-meshlet bounding sphere + cone (for task-shader culling)
for (size_t i = 0; i < meshlet_count; i++) {
    meshopt_Bounds b = meshopt_computeMeshletBounds(
        meshlet_vertices  + meshlets[i].vertex_offset,
        meshlet_triangles + meshlets[i].triangle_offset,
        meshlets[i].triangle_count,
        vertex_positions, vertex_count, vertex_stride);
    // b.center[3], b.radius, b.cone_axis[3], b.cone_cutoff
}
```

The resulting four buffers — `meshlets`, `meshlet_vertices`, `meshlet_triangles`, and per-meshlet bounds — are uploaded to device-local `VkBuffer` objects with `VK_BUFFER_USAGE_STORAGE_BUFFER_BIT` and bound as descriptor sets to both the task and mesh shaders.

### 4.3 Task Shader: Frustum and Cone Culling

A representative task shader that culls meshlets against the view frustum and the backface cone:

```glsl
#version 450
#extension GL_EXT_mesh_shader : require

layout(local_size_x = 32) in;   // one thread per meshlet candidate

// Meshlet bounding data from meshoptimizer
struct MeshletBounds {
    vec3  center;
    float radius;
    vec3  cone_axis;
    float cone_cutoff;
    vec3  cone_apex;
    float _pad;
};

layout(set = 0, binding = 0) readonly buffer MeshletBoundsBuffer {
    MeshletBounds bounds[];
};
layout(set = 0, binding = 1) readonly buffer DrawData {
    uint  meshlet_count;
    mat4  view_proj;
    vec3  camera_pos;
};

struct TaskPayload {
    uint base_meshlet;
    uint visible_meshlets[32];  // packed visible indices within this workgroup
};
taskPayloadSharedEXT TaskPayload payload;

shared uint s_visible_count;

// Simple sphere-plane frustum cull (6 planes pre-computed on CPU)
layout(set = 0, binding = 2) readonly buffer Frustum {
    vec4 planes[6];
};

bool sphereInFrustum(vec3 center, float r) {
    for (int i = 0; i < 6; i++)
        if (dot(planes[i].xyz, center) + planes[i].w < -r) return false;
    return true;
}

bool coneBackface(MeshletBounds b) {
    vec3 apex_to_cam = normalize(camera_pos - b.cone_apex);
    return dot(apex_to_cam, b.cone_axis) >= b.cone_cutoff;
}

void main() {
    uint tid = gl_LocalInvocationID.x;

    if (tid == 0) {
        s_visible_count = 0;
        payload.base_meshlet = gl_WorkGroupID.x * 32;
    }
    barrier();

    uint meshlet_idx = gl_WorkGroupID.x * 32 + tid;
    if (meshlet_idx < meshlet_count) {
        MeshletBounds b = bounds[meshlet_idx];
        bool visible = sphereInFrustum(b.center, b.radius) &&
                       !coneBackface(b);
        if (visible) {
            uint slot = atomicAdd(s_visible_count, 1);
            payload.visible_meshlets[slot] = tid;
        }
    }
    barrier();

    EmitMeshTasksEXT(s_visible_count, 1, 1);
}
```

> **Note**: This example omits Hi-Z occlusion culling (two-phase approach) and subgroup-wave compaction that a production implementation would use. The Themaister Granite renderer demonstrates a production-quality approach ([Modernizing Granite's mesh rendering, January 2024](https://themaister.net/blog/2024/01/17/modernizing-granites-mesh-rendering/)).

### 4.4 Mesh Shader: Emitting Geometry

```glsl
#version 450
#extension GL_EXT_mesh_shader : require

layout(local_size_x = 128) in;
layout(triangles, max_vertices = 64, max_primitives = 124) out;

struct TaskPayload {
    uint base_meshlet;
    uint visible_meshlets[32];
};
taskPayloadSharedEXT TaskPayload payload;

struct Meshlet {
    uint vertex_offset;
    uint triangle_offset;
    uint vertex_count;
    uint triangle_count;
};

layout(set = 0, binding = 3) readonly buffer MeshletBuffer  { Meshlet  meshlets[];  };
layout(set = 0, binding = 4) readonly buffer VertexIndices  { uint     v_indices[]; };
layout(set = 0, binding = 5) readonly buffer TriangleBuffer { uint8_t  tri_buf[];   };
layout(set = 0, binding = 6) readonly buffer PositionBuffer { vec4     positions[]; };

layout(push_constant) uniform PC { mat4 mvp; };

// Per-primitive material/object ID (read by FS as flat input)
perprimitiveEXT out uint primMaterialID[];

void main() {
    // Which meshlet does this workgroup handle?
    uint local_idx  = payload.visible_meshlets[gl_WorkGroupID.x];
    uint meshlet_id = payload.base_meshlet + local_idx;
    Meshlet m       = meshlets[meshlet_id];

    SetMeshOutputsEXT(m.vertex_count, m.triangle_count);

    uint tid = gl_LocalInvocationID.x;

    // Emit vertices
    if (tid < m.vertex_count) {
        uint vi = v_indices[m.vertex_offset + tid];
        gl_MeshVerticesEXT[tid].gl_Position = mvp * positions[vi];
    }

    // Emit triangles (3 bytes per tri, packed as uint8)
    if (tid < m.triangle_count) {
        uint base = (m.triangle_offset + tid) * 3;
        gl_PrimitiveTriangleIndicesEXT[tid] = uvec3(
            tri_buf[base + 0],
            tri_buf[base + 1],
            tri_buf[base + 2]);
        primMaterialID[tid] = meshlet_id & 0xFFFF;  // example per-primitive data
    }
}
```

### 4.5 Capability Query

Before using the extension, check hardware capabilities:

```c
VkPhysicalDeviceMeshShaderFeaturesEXT mesh_features = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_MESH_SHADER_FEATURES_EXT,
};
VkPhysicalDeviceFeatures2 features2 = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_FEATURES_2,
    .pNext = &mesh_features,
};
vkGetPhysicalDeviceFeatures2(physdev, &features2);
// mesh_features.taskShader, mesh_features.meshShader

VkPhysicalDeviceMeshShaderPropertiesEXT mesh_props = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_MESH_SHADER_PROPERTIES_EXT,
};
VkPhysicalDeviceProperties2 props2 = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_PROPERTIES_2,
    .pNext = &mesh_props,
};
vkGetPhysicalDeviceProperties2(physdev, &props2);
// mesh_props.maxMeshOutputVertices (≥256), .maxMeshOutputPrimitives (≥256)
// mesh_props.maxPreferredMeshWorkGroupInvocations (vendor hint)
// mesh_props.prefersLocalInvocationVertexOutput (NVIDIA prefers true)
```

Shell command to verify support:

```bash
vulkaninfo 2>/dev/null | grep -i -E "meshShader|taskShader"
```

---

## 5. Hardware Support on Linux

### 5.1 NVIDIA (Proprietary Driver and NVK)

NVIDIA's proprietary Vulkan driver has supported mesh shaders since Ampere (RTX 30xx, GA10x), initially via `VK_NV_mesh_shader` and subsequently via `VK_EXT_mesh_shader`. The proprietary driver on Linux ships the full EXT extension.

The **NVK** open-source NVIDIA Vulkan driver, built on the NAK compiler back-end and maintained in the Mesa tree, merged `VK_EXT_mesh_shader` support in mid-2026, targeting the Mesa 26.x release series ([Phoronix: NVK Mesh Shaders Merged](https://www.phoronix.com/news/NVK-Mesh-Shaders-Merged)). Support covers Ampere (GA10x) and later; as of mid-2026, verify the current status via `vulkaninfo` on a Mesa 26.x+ install.

**`VK_NV_mesh_shader` vs `VK_EXT_mesh_shader`**: The NV-specific extension uses `gl_PrimitiveTriangleIndicesNV`, `EmitMeshTasksNV()`, and `taskNV`/`meshNV` stage qualifiers. Code targeting the EXT extension uses `GL_EXT_mesh_shader` and is cross-vendor.

### 5.2 AMD (RADV)

RADV's mesh shader journey reflects the significant architectural difference between RDNA2 and RDNA3:

**RDNA2 (GFX10.3, RX 6000 series)**: Mesh shader support landed in Mesa 22.3 (late 2022). A critical hardware limitation on RDNA2 is that each mesh shader invocation can only directly write one vertex and one primitive output — diverging from the compute-shader-style "any lane writes any output" model. RADV works around this via compiler-generated temporary storage and coordinate remapping. Per the RDNA3 mesh shading analysis by Timur Kristóf ([AMD RDNA3 mesh shading with RADV, 2024](https://timur.hu/blog/2024/rdna3-mesh-shading)), this limits RDNA2 efficiency.

**RDNA3 (GFX11, RX 7000 series)**: Full mesh shader support, including per-primitive outputs, landed in Mesa 23.1. RDNA3 introduces two hardware innovations:

- **Attribute Ring**: replaces the parameter cache with VRAM-based ring storage backed by the Infinity Cache. Any invocation can now write attributes for any other invocation, lifting the RDNA2 one-vertex-per-invocation constraint.
- **Row Exports**: position and clip/cull distance data uses new row-export modes that allow lanes to write data for any lane in the same row.

RADV implements two "fast launch" modes for mesh shaders: a *legacy* mode (compatible with RDNA2's topology) and a *new* mode where invocation counts follow the compute shader model, eliminating vertex/primitive count matching requirements. RDNA4 is expected to use the new mode exclusively.

Mesh/task shader pipeline statistics queries (for `VK_QUERY_TYPE_PIPELINE_STATISTICS`) landed in Mesa 24.0 for GFX10.3 ([Phoronix](https://www.phoronix.com/forums/forum/linux-graphics-x-org-drivers/open-source-amd-linux/1424397-mesh-task-shader-queries-land-for-radv-with-rdna2-rdna3-support-on-the-way)).

### 5.3 Intel (ANV)

Intel's ANV driver added experimental `VK_EXT_mesh_shader` support targeting DG2/Alchemist (Xe-HPG, Gen12.5+) GPU hardware ([Phoronix: Intel ANV Mesh Shader](https://www.phoronix.com/news/Intel-Lands-Mesh-Shaders)). The implementation initially shipped behind an environment variable guard; later Mesa releases promoted it to stable. As of mid-2026, Xe2 and Arc B-series hardware should have production-quality mesh shader support — verify with `vulkaninfo`.

### 5.4 Quick Support Check

```bash
# Check for VK_EXT_mesh_shader
vulkaninfo 2>/dev/null | grep -E "VK_EXT_mesh_shader|taskShader|meshShader"

# Disable mesh shader support on RDNA2+ for debugging/comparison
RADV_DEBUG=nomeshshader ./my_vulkan_app

# Dump RADV shader IR for the mesh pipeline (ISA + NIR)
RADV_DEBUG=shaders ./my_vulkan_app 2>&1 | grep -A 20 "mesh"
```

---

## 6. Variable Rate Shading: Motivation

### 6.1 The Observation

Not every pixel in a rendered frame justifies the same shader effort. Consider:

| Region | Typical characteristic | Optimal shading density |
|--------|----------------------|------------------------|
| Featureless sky | Smooth gradient | 1 shade per 4×4 pixels |
| Distant terrain | Low spatial frequency | 1 shade per 2×2 pixels |
| Close foreground | High detail, specular highlights | 1 shade per pixel (1:1) |
| UI/HUD text | Sub-pixel features | 1:1 or supersampled |

Fragment shader invocations are expensive: each invocation requires register file allocation, texture cache occupancy, and fixed-function per-sample output merging. Reducing invocation count proportionally reduces shader cost while the rasterizer, depth test, and stencil hardware still operate at full pixel resolution.

This is the core premise of **Variable Rate Shading**: decouple the *shading rate* (how often the fragment shader runs) from the *render resolution* (how many pixels are covered by the rasterizer).

### 6.2 The Three Modes

`VK_KHR_fragment_shading_rate` defines three independent sources of shading rate that are combined via configurable *combiner operations*:

1. **Per-pipeline rate**: a single rate applied to all fragments drawn by a pipeline. Set via `VkPipelineFragmentShadingRateStateCreateInfoKHR` or dynamically via `vkCmdSetFragmentShadingRateKHR`.

2. **Per-primitive rate**: the VS, GS, or mesh shader writes `gl_PrimitiveShadingRateEXT` for each primitive, communicating the desired rate to the rasterizer. Requires the `primitiveFragmentShadingRate` feature.

3. **Per-tile (attachment) rate**: a `VkImage` attachment (the *shading rate image*) where each texel governs a rectangular tile of the framebuffer. Requires the `attachmentFragmentShadingRate` feature and the `VkFragmentShadingRateAttachmentInfoKHR` subpass structure.

### 6.3 Shading Rate Representation

Shading rates are expressed as `(width, height)` in pixels per fragment shader invocation:

| Rate | Invocations per tile | Use case |
|------|---------------------|----------|
| 1×1 | 1 per pixel (full) | Foreground, specular |
| 2×1 | 1 per 2-pixel row | Horizontal smooth gradients |
| 1×2 | 1 per 2-pixel column | Vertical gradients |
| 2×2 | 1 per 4 pixels | Sky, distant terrain |
| 4×2 | 1 per 8 pixels | Background |
| 4×4 | 1 per 16 pixels | VR periphery, deep background |

Not all rates are guaranteed: `(1,1)`, `(2,1)`, and `(2,2)` are mandatory; others depend on `VkPhysicalDeviceFragmentShadingRateKHR.sampleCounts`.

### 6.4 Combiner Operations

The three rate sources are combined in two sequential steps (each using a `VkFragmentShadingRateCombinerOpKHR` value):

```
combiner[0]: (pipeline_rate) op (primitive_rate) → intermediate
combiner[1]: (intermediate)  op (attachment_rate) → final_rate
```

The five combiner ops are:

| Combiner | Behaviour |
|----------|-----------|
| `KEEP`    | Use first operand |
| `REPLACE` | Use second operand |
| `MIN`     | Min of both (higher quality) |
| `MAX`     | Max of both (lower quality / higher perf) |
| `MUL`     | Multiply (if `fragmentShadingRateStrictMultiplyCombiner`; otherwise log2-space saturating add) |

---

## 7. VRS API Usage

### 7.1 Per-Pipeline Rate

The simplest usage sets a static rate for all draw calls using a given pipeline:

```c
VkPipelineFragmentShadingRateStateCreateInfoKHR vrs_state = {
    .sType         = VK_STRUCTURE_TYPE_PIPELINE_FRAGMENT_SHADING_RATE_STATE_CREATE_INFO_KHR,
    .fragmentSize  = { .width = 2, .height = 2 },   // 2×2: quarter rate
    .combinerOps   = {
        VK_FRAGMENT_SHADING_RATE_COMBINER_OP_KEEP_KHR,     // ignore primitive rate
        VK_FRAGMENT_SHADING_RATE_COMBINER_OP_KEEP_KHR,     // ignore attachment rate
    },
};
// Attach to VkGraphicsPipelineCreateInfo::pNext
```

For dynamic pipelines, override per-command buffer:

```c
VkExtent2D rate = { .width = 2, .height = 2 };
VkFragmentShadingRateCombinerOpKHR ops[2] = {
    VK_FRAGMENT_SHADING_RATE_COMBINER_OP_REPLACE_KHR,  // primitive overrides pipeline
    VK_FRAGMENT_SHADING_RATE_COMBINER_OP_MIN_KHR,      // attachment takes min with primitive
};
vkCmdSetFragmentShadingRateKHR(cmd, &rate, ops);
```

### 7.2 Per-Primitive Rate via Vertex/Mesh Shader

The `gl_PrimitiveShadingRateEXT` built-in in `gl_MeshPrimitivesEXT` (or in a VS output) sets the rate per primitive. The value uses an OR of flag bits:

```glsl
// In mesh shader:
// Fragment Shading Rate flag bits (from GL_EXT_fragment_shading_rate):
//   gl_ShadingRateFlag2VerticalPixelsEXT   = 0x1  → height = 2 pixels per invocation
//   gl_ShadingRateFlag4VerticalPixelsEXT   = 0x2  → height = 4 pixels per invocation
//   gl_ShadingRateFlag2HorizontalPixelsEXT = 0x4  → width  = 2 pixels per invocation
//   gl_ShadingRateFlag4HorizontalPixelsEXT = 0x8  → width  = 4 pixels per invocation
// Convention: rate W×H, so 2×1 means 2 pixels wide, 1 pixel tall per invocation.
const uint SR_1x1 = 0;
const uint SR_2x1 = gl_ShadingRateFlag2HorizontalPixelsEXT;   // 0x4 (width=2, height=1)
const uint SR_1x2 = gl_ShadingRateFlag2VerticalPixelsEXT;     // 0x1 (width=1, height=2)
const uint SR_2x2 = gl_ShadingRateFlag2HorizontalPixelsEXT |
                    gl_ShadingRateFlag2VerticalPixelsEXT;      // 0x5 (width=2, height=2)

// Example: distant primitives get 2×2, near ones get 1×1
if (tid < m.triangle_count) {
    float cluster_dist = length(cluster_center_view);
    gl_MeshPrimitivesEXT[tid].gl_PrimitiveShadingRateEXT =
        (cluster_dist > 50.0) ? SR_2x2 : SR_1x1;
}
```

Note that the `primitiveFragmentShadingRateMeshShader` feature must be true to use this built-in from a mesh shader.

### 7.3 Per-Tile Attachment Rate

The attachment-based approach provides the finest spatial control. Implementation steps:

**Step 1: Create the shading rate image**

```c
VkPhysicalDeviceFragmentShadingRatePropertiesKHR vrs_props = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_FRAGMENT_SHADING_RATE_PROPERTIES_KHR,
};
// ...query via vkGetPhysicalDeviceProperties2...

// Tile size (e.g. 8×8 or 16×16 pixels per texel)
VkExtent2D tile = vrs_props.minFragmentShadingRateAttachmentTexelSize; // commonly 8×8

uint32_t vrs_width  = (render_width  + tile.width  - 1) / tile.width;
uint32_t vrs_height = (render_height + tile.height - 1) / tile.height;

VkImageCreateInfo vrs_img_info = {
    .sType       = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO,
    .imageType   = VK_IMAGE_TYPE_2D,
    .format      = VK_FORMAT_R8_UINT,     // only valid format for VRS attachment
    .extent      = { vrs_width, vrs_height, 1 },
    .mipLevels   = 1,
    .arrayLayers = 1,
    .samples     = VK_SAMPLE_COUNT_1_BIT,
    .tiling      = VK_IMAGE_TILING_OPTIMAL,
    .usage       = VK_IMAGE_USAGE_FRAGMENT_SHADING_RATE_ATTACHMENT_BIT_KHR |
                   VK_IMAGE_USAGE_STORAGE_BIT,  // so compute can write it
};
```

**Step 2: Encode rates**

Each `R8_UINT` texel encodes the shading rate using:

```
texel_value = (log2(width) << 2) | log2(height)
```

Where `width` and `height` are the pixel dimensions per invocation ([Vulkan spec: Fragment Shading Rate Attachment](https://registry.khronos.org/vulkan/specs/latest/chapters/fragops.html#fragment-shading-rate-attachment)):

| Rate | width | height | log2(w) | log2(h) | Encoded value |
|------|-------|--------|---------|---------|---------------|
| 1×1  | 1     | 1      | 0       | 0       | 0x00          |
| 2×1  | 2     | 1      | 1       | 0       | 0x04          |
| 1×2  | 1     | 2      | 0       | 1       | 0x01          |
| 2×2  | 2     | 2      | 1       | 1       | 0x05          |
| 4×2  | 4     | 2      | 2       | 1       | 0x09          |
| 4×4  | 4     | 4      | 2       | 2       | 0x0A          |

**Step 3: Generate the VRS image via compute shader**

```glsl
#version 450
#extension GL_KHR_shader_subgroup_arithmetic : enable

layout(local_size_x = 8, local_size_y = 8) in;
layout(binding = 0, r8ui) writeonly uniform uimage2D vrs_output;
layout(binding = 1) uniform sampler2D luma_prev;   // previous frame luminance

void main() {
    ivec2 coord = ivec2(gl_GlobalInvocationID.xy);
    vec2 uv     = (vec2(coord) + 0.5) / vec2(imageSize(vrs_output));

    // Sample luminance variance across this VRS tile
    float luma_c  = texture(luma_prev, uv).r;
    float luma_dx = dFdx(luma_c);
    float luma_dy = dFdy(luma_c);
    float variance = luma_dx * luma_dx + luma_dy * luma_dy;

    // Select rate based on spatial variance
    uint rate;
    if      (variance > 0.02) rate = 0x00;  // 1×1 full rate
    else if (variance > 0.005) rate = 0x05; // 2×2
    else                       rate = 0x0A; // 4×4

    imageStore(vrs_output, coord, uvec4(rate, 0, 0, 0));
}
```

**Step 4: Attach to dynamic rendering**

```c
VkRenderingFragmentShadingRateAttachmentInfoKHR vrs_attach = {
    .sType                          = VK_STRUCTURE_TYPE_RENDERING_FRAGMENT_SHADING_RATE_ATTACHMENT_INFO_KHR,
    .imageView                      = vrs_image_view,
    .imageLayout                    = VK_IMAGE_LAYOUT_FRAGMENT_SHADING_RATE_ATTACHMENT_OPTIMAL_KHR,
    .shadingRateAttachmentTexelSize = tile,
};
VkRenderingInfo render_info = {
    .sType = VK_STRUCTURE_TYPE_RENDERING_INFO,
    .pNext = &vrs_attach,
    // ... color attachments, depth, renderArea ...
};
vkCmdBeginRendering(cmd, &render_info);
```

### 7.4 Foveated Rendering Pattern

For XR applications with eye tracking, the VRS image encodes a radially-varying rate centred on the gaze point:

```glsl
// In the VRS generation compute shader:
layout(push_constant) uniform Fovea {
    vec2 gaze_uv;   // gaze point [0,1]²
    float inner_r;  // radius of full-rate circle
    float outer_r;  // radius of half-rate circle
};

void main() {
    ivec2 coord = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv    = (vec2(coord) + 0.5) / vec2(imageSize(vrs_output));

    float dist = length(uv - gaze_uv);
    uint  rate;
    if      (dist < inner_r)  rate = 0x00;  // 1×1 foveal
    else if (dist < outer_r)  rate = 0x05;  // 2×2 mid-periphery
    else                      rate = 0x0A;  // 4×4 far periphery

    imageStore(vrs_output, coord, uvec4(rate, 0, 0, 0));
}
```

---

## 8. VRS Hardware Support and Driver Implementation

### 8.1 Feature Query

```c
VkPhysicalDeviceFragmentShadingRateFeaturesKHR vrs_features = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_FRAGMENT_SHADING_RATE_FEATURES_KHR,
};
// ... chain into VkPhysicalDeviceFeatures2 and call vkGetPhysicalDeviceFeatures2 ...

// Three independent booleans:
// vrs_features.pipelineFragmentShadingRate  — always required if ext is present
// vrs_features.primitiveFragmentShadingRate — per-primitive (VS/GS/MS output)
// vrs_features.attachmentFragmentShadingRate — per-tile image
```

Shell check:

```bash
vulkaninfo 2>/dev/null | grep -i -E "shadingRate|ShadingRate"
```

### 8.2 RADV (AMD)

RADV added `VK_KHR_fragment_shading_rate` support for GFX10.3 (RDNA2) in Mesa 21.1 ([Phoronix: RADV VRS](https://www.phoronix.com/forums/forum/linux-graphics-x-org-drivers/open-source-amd-linux/1226101-radv-vulkan-driver-enables-fragment-shading-rate-support-limited-to-gfx10-3-rdna-2)). On RDNA2, per-pipeline and per-primitive VRS are supported. Per-tile attachment VRS via `VRS_IMAGE` requires GFX11 (RDNA3, RX 7000 series).

The RADV implementation sets the shading rate via `radv_emit_fragment_shading_rate()` in `src/amd/vulkan/radv_cmd_buffer.c`, which programs the `PA_SC_VRS_RATE_CNTL` hardware register on GFX10.3+. Mesa also exposes an environment variable for forcing a specific per-pipeline VRS rate for debugging:

```bash
# Force 2×2 VRS globally on RDNA2+ (debug only)
RADV_FORCE_VRS=2x2 vkcube
# Supported values: 2x2, 1x2, 2x1, 1x1 (GFX10.3+)
```

Tile sizes available on RDNA2/3 are typically 8×8 or 16×16 pixels; query `fragmentShadingRateAttachmentTexelSize` from properties.

### 8.3 ANV (Intel)

Intel's ANV driver added `VK_KHR_fragment_shading_rate` support starting with Gen12 (Tiger Lake), covering per-pipeline rate and — on Gen12.5 / DG2 and later — per-primitive and attachment rates ([Phoronix: Intel ANV Fragment Shading Rate](https://www.phoronix.com/news/Intel-ANV-Fragment-Shading-Rate)). The `fragmentShadingRateWithSampleMask` property indicates whether VRS can be combined with MSAA sample masks.

### 8.4 NVK / NVIDIA Proprietary

NVIDIA's proprietary driver has supported VRS on Turing (RTX 20xx) and later via `VK_NV_shading_rate_image` and subsequently via `VK_KHR_fragment_shading_rate` on Ampere+. The open-source NVK driver is expected to gain VRS support alongside mesh shaders in the Mesa 26.x era; verify current status via `vulkaninfo`.

### 8.5 Precision and Quality Considerations

VRS introduces visual artefacts in certain conditions:

- **Specular highlights**: high-frequency specular lobes alias heavily under 4×4 VRS. Mitigation: limit VRS rate near specular surfaces (detected via G-buffer roughness < threshold).
- **Thin geometry / wireframes**: single-pixel lines may disappear under 2×2 or coarser rates. Always use 1×1 for UI or wireframe overlays.
- **Dithering at rate boundaries**: the transition between shading rate regions can produce visible dithering bands. Some drivers apply a temporal dither or blend across the boundary. Temporal anti-aliasing (TAA) mitigates this in practice.
- **Sub-pixel features**: text rendering should always use 1×1 rate; preferably VRS should be disabled for UI render passes entirely.

---

## 9. Combined Usage: VRS + Mesh Shaders

### 9.1 Per-Primitive VRS from the Mesh Shader

The most powerful integration is having the mesh shader set per-primitive VRS based on cluster screen-space coverage. This requires enabling both `primitiveFragmentShadingRateMeshShader` and `meshShader` features simultaneously.

```glsl
// In the mesh shader (continuing from §4.4):
if (tid < m.triangle_count) {
    // ... triangle indices as before ...

    // Compute approximate screen-space area of this cluster
    // (cluster_screen_area was packed into the meshlet bounds data)
    float screen_area = cluster_screen_area[meshlet_id];

    uint rate;
    if      (screen_area > 1000.0) rate = SR_1x1;   // large on screen
    else if (screen_area > 250.0)  rate = SR_2x1;   // medium
    else                           rate = SR_2x2;   // small / distant

    gl_MeshPrimitivesEXT[tid].gl_PrimitiveShadingRateEXT = rate;
}
```

The combiner chain is set so the attachment rate takes `MIN` with the primitive rate, ensuring the attachment map can only increase quality (reduce shading rate), never override a finer per-primitive request.

### 9.2 Full GPU-Driven Pipeline with VRS

The draw command structure for the mesh shader indirect path:

```c
// Populated by the GPU compute culling pass
typedef struct VkDrawMeshTasksIndirectCommandEXT {
    uint32_t groupCountX;  // number of task/mesh workgroups in X
    uint32_t groupCountY;
    uint32_t groupCountZ;
} VkDrawMeshTasksIndirectCommandEXT;
```

A production frame using both extensions looks like:

```
1. [Compute] Hi-Z reduction from previous frame depth buffer
   Barrier: VK_PIPELINE_STAGE_LATE_FRAGMENT_TESTS_BIT →
            VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT

2. [Compute] Visibility + meshlet culling:
   - Reads meshlet bounding spheres
   - Tests against Hi-Z and frustum
   - Writes VkDrawMeshTasksIndirectCommandEXT[] + count to GPU buffer
   Barrier: VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT →
            VK_PIPELINE_STAGE_DRAW_INDIRECT_BIT
            (src access: SHADER_WRITE, dst access: INDIRECT_COMMAND_READ)

3. [Compute] VRS image generation (luminance variance from prev frame colour)
   Barrier: VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT →
            VK_PIPELINE_STAGE_FRAGMENT_SHADING_RATE_ATTACHMENT_BIT_KHR
            (src access: SHADER_WRITE, dst access: FRAGMENT_SHADING_RATE_ATTACHMENT_READ)

4. [Mesh/Task → Rasterizer → FS] Begin dynamic rendering with VRS attachment:
      vkCmdDrawMeshTasksIndirectCountEXT(
          cmd, meshlet_dispatch_buffer, 0,
          visible_count_buffer, 0,
          MAX_MESHLETS, sizeof(VkDrawMeshTasksIndirectCommandEXT));
   - Task shader: fine per-cluster culling (cone, small screen area)
   - Mesh shader: geometry output + per-primitive VRS rate
   - Fragment shader: reads primMaterialID (perprimitiveEXT), GBuffer outputs
   End dynamic rendering.

5. [Post-process] TAA / temporal upscale (FSR3, DLSS via NvAPI, or native)
```

The GPU never returns to the CPU between steps 1–5. The pipeline barriers are expressed as `vkCmdPipelineBarrier2` calls with `VkDependencyInfo` (Vulkan 1.3 synchronization2).

### 9.3 Performance Analysis

Use Vulkan query pools to measure mesh shader efficiency:

```c
// Pipeline statistics query
VkQueryPoolCreateInfo qp_info = {
    .sType              = VK_STRUCTURE_TYPE_QUERY_POOL_CREATE_INFO,
    .queryType          = VK_QUERY_TYPE_PIPELINE_STATISTICS,
    .queryCount         = 1,
    .pipelineStatistics =
        VK_QUERY_PIPELINE_STATISTIC_TASK_SHADER_INVOCATIONS_BIT_EXT |
        VK_QUERY_PIPELINE_STATISTIC_MESH_SHADER_INVOCATIONS_BIT_EXT,
};
```

For VRS efficiency, vendor tools are most informative:
- **AMD Radeon GPU Profiler (RGP)**: shows shading rate breakdown per draw.
- **NVIDIA Nsight Graphics**: VRS occupancy lane.
- **Intel GPA**: fragment efficiency metric.

The Granite renderer's measurements on an RTX 3070 ([Themaister, January 2024](https://themaister.net/blog/2024/01/17/modernizing-granites-mesh-rendering/)) show the optimisation stack from basic indexed draws to mesh shaders with micro-polygon rejection: frame time dropping from 5.5 ms (basic) to 1.0 ms (optimised mesh shaders + HiZ culling), a 5.5× speedup. Note that task shader hierarchical culling alone showed a regression on NVIDIA hardware due to AMD-style tiny indirect dispatch overhead; `vkCmdDrawMeshTasksIndirectCountEXT` with GPU-generated counts from the preceding compute pass was more universally efficient across AMD and NVIDIA.

### 9.4 Common Pitfalls

**Meshlet size portability**: NVIDIA hardware performs best with 32 vertices / 32 primitives per workgroup (matching warp width), while AMD RDNA3 can sustain throughput with smaller meshlets but degrades with very large ones. meshoptimizer's default of 64 vertices / 124 triangles is a reasonable cross-vendor starting point but should be profiled per vendor.

**Task shader over-dispatch on NVIDIA**: Task shaders internally emit small indirect dispatches on AMD hardware, adding latency that can outweigh culling savings when few meshlets are actually culled. For scenes with mostly-visible geometry (e.g. no frustum culling), skip the task stage and use `vkCmdDrawMeshTasksIndirectCountEXT` with pre-culled counts instead.

**VRS and MSAA interaction**: `VK_KHR_fragment_shading_rate` and MSAA can be combined only if `fragmentShadingRateWithSampleMask` and compatible sample counts are advertised. VRS with MSAA 4× means a 2×2 shading rate evaluates the fragment shader once per 4-pixel tile but still produces 4 coverage samples per pixel — the shader cost is reduced but not the stencil/depth memory bandwidth.

**Ordering of `SetMeshOutputsEXT`**: The spec requires `SetMeshOutputsEXT` to execute before any writes to mesh outputs. A common bug is writing to `gl_MeshVerticesEXT[i]` in a branch that executes before the unconditional `SetMeshOutputsEXT`, which triggers undefined behaviour. Place `SetMeshOutputsEXT` as the first statement in `main()`.

**`perprimitiveEXT` in fragment shader**: Fragment shader inputs that read per-primitive data from the mesh shader must be declared with the `perprimitiveEXT` qualifier and the `flat` interpolation qualifier. Omitting either causes validation layer errors or incorrect reads.

---

## 10. Real-World Usage on Linux

### 10.1 Granite Render Engine

The [Granite](https://github.com/Themaister/Granite) renderer by Hans-Kristian Arntzen ("Themaister") is the most complete open-source reference for mesh shaders + VRS on Linux Vulkan. The January 2024 blog series ([Modernizing Granite's mesh rendering](https://themaister.net/blog/2024/01/)) documents:

- Meshlet format using 256-vertex/primitive workgroups divided into 8×32 "sublets"
- Position compression: 3×16-bit SINT with shared exponent
- UV: 2×16-bit SINT with fixup for \[0,1\] range
- Normal/tangent: octahedral encoding, 4×8-bit SNORM
- Index buffers: 5-bit packed indices per sublet
- Vertex ID passthrough optimisation for fragment-shader attribute fetch (the single biggest gain)
- Cross-vendor tuning: NVIDIA prefers 32/32 meshlet size; AMD RDNA3 favours smaller meshlets

Granite is MIT-licensed and targets Linux/Vulkan as a primary platform.

### 10.2 Vulkan-Samples (Khronos)

The [KhronosGroup/Vulkan-Samples](https://github.com/KhronosGroup/Vulkan-Samples) repository contains Linux-compatible mesh shader and VRS samples:

```bash
# Build and run the mesh shader culling sample
git clone https://github.com/KhronosGroup/Vulkan-Samples
cd Vulkan-Samples && cmake -S . -B build -DVKB_VALIDATION_LAYERS=ON
cmake --build build --target vulkan_samples -j$(nproc)
./build/app/bin/vulkan_samples --sample mesh_shader_culling
./build/app/bin/vulkan_samples --sample fragment_shading_rate_dynamic
```

The `mesh_shader_culling` sample demonstrates task shader distance culling with `GL_EXT_mesh_shader`. The `fragment_shading_rate_dynamic` sample shows the two-pass luminance-derivative approach described in §7.3.

### 10.3 NVIDIA Mesh Shader CAD Scene Sample

NVIDIA's [gl_vk_meshlet_cadscene](https://github.com/nvpro-samples/gl_vk_meshlet_cadscene) demonstrates mesh shaders in a CAD rendering context, updated for `VK_EXT_mesh_shader`. Despite the NV provenance, the meshlet data generation pipeline is vendor-neutral and the Vulkan EXT path runs on RADV and ANV.

### 10.4 Godot 4 and Blender EEVEE Next

Godot 4 uses GPU-driven rendering via compute shaders and indirect draw but does not yet use mesh shaders at the GLSL/Vulkan level (as of 2025); instead it leverages `RenderingDevice` with `VK_BUFFER_USAGE_INDIRECT_BUFFER_BIT` for batched indirect draws. Mesh LOD selection is done on the CPU and via `VkDrawIndexedIndirectCommand` batches. Mesh shaders remain a candidate for future versions.

Blender's EEVEE Next (Cycles renderer uses compute) targets GPU-driven passes for AO, shadow maps, and depth of field — areas where per-tile VRS could help. As of mid-2026 neither Blender nor Godot expose VRS through their Vulkan backends; both are candidates for community contribution.

### 10.5 WebGPU and Dawn

WebGPU does not yet standardise mesh shaders. The [Tint compiler](https://dawn.googlesource.com/dawn/+/refs/heads/main/src/tint/) and [Dawn](https://dawn.googlesource.com/dawn) runtime include experimental `WGPUFeature_MeshShaders` feature flags in their native (non-browser) API paths, but these are not exposed through the W3C WebGPU spec as of 2026. VRS similarly has no WebGPU equivalent.

### 10.6 Debugging and Profiling on Linux

```bash
# RADV: dump shader assembly for all compiled shaders (includes mesh/task)
RADV_DEBUG=shaders ./my_vulkan_app 2>&1 | less

# RADV: disable mesh shaders on RDNA2+ (comparison baseline)
RADV_DEBUG=nomeshshader ./my_vulkan_app

# RADV: force VRS rate for debugging (GFX10.3+)
RADV_FORCE_VRS=2x2 ./my_vulkan_app
# Supported values: 2x2, 1x2, 2x1, 1x1

# RADV: PERFTEST flags (mesh_shader was early opt-in; Mesa 23.1+ always on for RDNA2+)
# RADV_PERFTEST=mesh_shader was only needed before Mesa 23.1

# Validation layers: check mesh shader feature chains
VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation ./my_vulkan_app 2>&1 | grep -i mesh
```

For in-depth GPU timeline analysis on AMD, use [Radeon GPU Profiler](https://gpuopen.com/rgp/) (runs on Linux via Radeon Developer Panel + SSH) or `radv-perf-trace` via Mesa's RGP export. NVIDIA's Nsight Systems (`nsys`) and Nsight Graphics (`ngfx`) both run on Linux.

---

## Roadmap

### Near-term (6–12 months)

- **Vulkan Roadmap 2026 mandates VRS**: The Khronos Vulkan working group's January 2026 milestone formally requires `VK_KHR_fragment_shading_rate` support from conforming drivers targeting desktops, laptops, and mid-to-high-end devices, with most adopters expected to have conformant implementations by end of 2026. [Source](https://www.phoronix.com/news/Vulkan-Roadmap-2026)
- **RADV and ANV pipeline optimisations for mesh shaders**: Following Mesa 23.1's RDNA3 mesh shader enablement and RDNA2 general availability, ongoing optimisation of the "new fast launch" attribute ring mode in RADV and ANV's Xe2 mesh pipeline codegen are expected to close remaining performance gaps with the NV proprietary driver. [Source](https://www.phoronix.com/news/RADV-Vulkan-Mesh-Shaders)
- **NVK mesh shader stabilisation**: NVK's open-source NVIDIA Vulkan driver merged initial `VK_EXT_mesh_shader` support (2025); the near-term focus is reaching feature parity with proprietary driver performance on Ada Lovelace and completing `VK_KHR_fragment_shading_rate` support. [Source](https://www.phoronix.com/news/NVK-Mesh-Shaders-Merged)
- **Godot 4 and Blender EEVEE Next mesh shader adoption**: Both engines are community candidates for integrating `VK_EXT_mesh_shader` dispatch paths to replace CPU-driven indirect draw batching; contributors are tracking Mesa driver stability before upstreaming (Note: needs verification for concrete merge timeline).
- **Vulkan SDK Q1 2026 tooling update**: Khronos planned an early-2026 Vulkan SDK release targeting Roadmap 2026 validation layers, conformance tests, and SPIRV-Tools updates covering mesh shader and VRS validation rules. [Source](https://www.khronos.org/blog/vulkan-introduces-roadmap-2026-and-new-descriptor-heap-extension)

### Medium-term (1–3 years)

- **WebGPU mesh shader proposal**: Dawn's native-path `WGPUFeature_MeshShaders` flag and wgpu's `SHADER_STAGE_MESH` feature bit are upstream, but no W3C WebGPU specification proposal has been filed as of mid-2026. A formal extension proposal is expected once mesh shaders achieve consistent cross-vendor coverage on desktop; WGSL shading language changes would be required. [Source](https://docs.rs/wgpu/latest/wgpu/struct.Features.html)
- **Variable rate shading in open-source display compositors**: Neither KWin (KDE) nor GNOME Mutter exposes VRS to client surfaces via the Wayland protocol; a future `wp_fragment_shading_rate` Wayland protocol extension is a likely prerequisite, analogous to the existing `wp_presentation_time` and `wp_content_type_v1` protocols (Note: needs verification for active proposal status).
- **Per-primitive fragment shading rate in mesh shaders**: `VK_KHR_fragment_shading_rate`'s `primitiveFragmentShadingRateWithMultipleViewports` feature, combined with `PerPrimitiveEXT`-decorated VRS outputs from mesh shaders, enables cluster-level shading rate selection without an image attachment; broader driver support and tooling for this combination is expected as Vulkan Roadmap 2026 adoption matures.
- **AMDGPU gang-submit kernel interface stabilisation**: Reliable multi-process use of task shaders on AMD hardware depends on gang-submit support in the `amdgpu` DRM driver to prevent cross-process deadlocks; the kernel-side interface is under active AMDGPU developer work and is a prerequisite for fully unconditional task shader support in RADV. [Source](https://www.phoronix.com/news/RADV-Vulkan-Mesh-Shaders)
- **Mesh shader support in Zink (OpenGL-on-Vulkan)**: Zink's translation of `GL_NV_mesh_shader` (and eventually an OpenGL mesh path) onto `VK_EXT_mesh_shader` is a medium-term Mesa project item, enabling mesh shader use from OpenGL applications without a Vulkan port.

### Long-term

- **Unified geometry pipeline successor**: GPU hardware architects (AMD, Intel, NVIDIA, ARM) are researching post-mesh-shader geometry pipeline models that eliminate even the task/mesh boundary, replacing it with a single programmable amplification unit feeding directly into a hardware rasterizer with dynamic LOD — an area of active GPU architecture research (Note: needs verification; no public RFC or ISA specification exists as of 2026).
- **Ray tracing and mesh shader convergence**: Long-term Vulkan roadmap discussion includes tighter integration between ray tracing (`VK_KHR_ray_tracing_pipeline`) and mesh shaders, specifically allowing mesh shader outputs to feed directly into acceleration structure update passes — eliminating a BLAS rebuild step for deformable mesh clusters (Note: needs verification for formal Khronos proposal).
- **VRS integration with AI-driven foveated rendering**: Hardware-accelerated eye tracking on consumer headsets (Meta Quest, Steam Deck successor, etc.) is expected to drive a `VK_EXT_fragment_shading_rate_enums` successor that accepts per-eye gaze vectors directly, replacing the image-attachment VRS approach with a sparser, hardware-managed shading map generated by an on-chip neural inference unit (Note: speculative; no public Khronos proposal as of 2026).
- **OpenXR VRS standardisation**: The OpenXR working group is expected to standardise a foveated rendering extension that wraps `VK_KHR_fragment_shading_rate` with per-layer, per-view rate control, enabling portable VRS across Vulkan and Metal-backed OpenXR runtimes (Note: needs verification for timeline).

---

## 11. Integrations

- **Ch18 — RADV Driver Internals**: The RADV implementation of mesh shaders (legacy vs. new fast launch mode, attribute ring integration, RDNA2 invocation-per-vertex constraint) and the `radv_emit_fragment_shading_rate` VRS implementation in `radv_cmd_buffer.c`.

- **Ch19 — ANV Driver Internals**: Intel ANV's mesh shader pipeline for Xe-HPG (DG2) and later, VRS attachment support on Gen12/Tiger Lake+, and `fragmentShadingRateWithSampleMask` interactions with MSAA.

- **Ch24 — Vulkan API for Application Developers**: The base Vulkan extension framework, feature/properties query patterns, pipeline creation, dynamic state, and descriptor set binding used throughout this chapter.

- **Ch77 — Shader Compilation Toolchain**: The GLSL → SPIR-V → NIR → ISA path for mesh and task shaders: `glslang` handling of `GL_EXT_mesh_shader`, SPIRV-Cross for reflection, and how RADV's NIR back-end lowers `OpSetMeshOutputsEXT` to AMD GCN register writes.

- **Ch97 — Unreal Engine 5 on Linux**: Nanite's cluster DAG, the mesh shader dispatch path used on RDNA2+/Ampere+ hardware, software rasterization fallback for micro-triangles, and integration with Lumen.

- **Ch110 — SPIR-V Internals**: The `SPV_EXT_mesh_shader` extension opcodes: `OpEmitMeshTasksEXT`, `OpSetMeshOutputsEXT`, `TaskPayloadWorkgroupEXT` storage class, `PerPrimitiveEXT` decoration, and the SPIR-V validation rules for mesh pipeline stages.

- **Ch112 — Variable Refresh Rate and Display Synchronisation**: The relationship between VRR (FreeSync/G-Sync) and VRS in VR rendering pipelines. VRR eliminates tearing at variable frame rates; VRS reduces per-frame GPU cost, enabling VRR to operate at higher sustained frame rates. Together they form the performance floor for OpenXR reprojection: VRS keeps the frame rate above the minimum refresh, VRR stretches the display to match.

---

*Sources and further reading:*

- [VK_EXT_mesh_shader specification](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_mesh_shader.html) — Khronos Registry
- [VK_EXT_mesh_shader design proposal](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_mesh_shader.html) — Vulkan Documentation Project
- [GLSL_EXT_mesh_shader specification](https://github.khronos.org/Vulkan-Site/glslext/latest/glslext/ext/GLSL_EXT_mesh_shader.html) — Khronos GLSL extensions
- [VK_KHR_fragment_shading_rate proposal](https://docs.vulkan.org/features/latest/features/proposals/VK_KHR_fragment_shading_rate.html) — Vulkan Documentation Project
- [Fragment Shading Rate Dynamic sample](https://docs.vulkan.org/samples/latest/samples/extensions/fragment_shading_rate_dynamic/README.html) — Vulkan-Samples
- [Mesh Shader Culling sample](https://docs.vulkan.org/samples/latest/samples/extensions/mesh_shader_culling/README.html) — Vulkan-Samples
- [AMD RDNA3 mesh shading with RADV](https://timur.hu/blog/2024/rdna3-mesh-shading) — Timur Kristóf, 2024
- [Modernizing Granite's mesh rendering](https://themaister.net/blog/2024/01/17/modernizing-granites-mesh-rendering/) — Hans-Kristian Arntzen, January 2024
- [Mesh Shading for Vulkan](https://www.khronos.org/blog/mesh-shading-for-vulkan) — Khronos Blog
- [meshoptimizer](https://meshoptimizer.org/) — Arseny Kapoulkine
- [NVK Mesh Shaders Merged](https://www.phoronix.com/news/NVK-Mesh-Shaders-Merged) — Phoronix
- [RADV VRS GFX10.3](https://www.phoronix.com/forums/forum/linux-graphics-x-org-drivers/open-source-amd-linux/1226101-radv-vulkan-driver-enables-fragment-shading-rate-support-limited-to-gfx10-3-rdna-2) — Phoronix
- [Intel ANV Fragment Shading Rate](https://www.phoronix.com/news/Intel-ANV-Fragment-Shading-Rate) — Phoronix
- [Intel ANV Mesh Shaders](https://www.phoronix.com/news/Intel-Lands-Mesh-Shaders) — Phoronix
- [Mesa 23.1 RADV Mesh Shaders for RDNA3](https://www.phoronix.com/news/RADV-Mesh-Shaders-RDNA3) — Phoronix

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
