# Chapter 135: Vulkan Ray Tracing on Linux

**Target audiences**: Vulkan application developers building ray-traced rendering, game engine and rendering research engineers, Mesa driver developers (RADV/ANV), and advanced users wanting to understand how hardware ray traversal maps to the Vulkan API on Linux.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Ray Tracing Pipeline and Shader Stages](#ray-tracing-pipeline-and-shader-stages)
3. [Acceleration Structures: BLAS and TLAS](#acceleration-structures-blas-and-tlas)
4. [BVH Construction Algorithms](#bvh-construction-algorithms)
   - [Self-Intersection Avoidance](#self-intersection-avoidance)
5. [Shader Binding Table](#shader-binding-table)
6. [RADV: RDNA2+ Ray Tracing](#radv-rdna2-ray-tracing)
7. [ANV: Intel Xe-HPG Ray Tracing](#anv-intel-xe-hpg-ray-tracing)
8. [VK_KHR_ray_query and Inline Ray Tracing](#vk_khr_ray_query-and-inline-ray-tracing)
9. [Practical Usage on Linux](#practical-usage-on-linux)
10. [SVGF: Spatiotemporal Variance-Guided Filtering](#svgf-spatiotemporal-variance-guided-filtering)
11. [Integrations](#integrations)

---

## Introduction

Vulkan ray tracing, finalised as a set of KHR extensions in 2020, provides portable API-level access to hardware ray traversal units that are now standard on discrete GPUs released since 2020–2021. On Linux, two open-source Mesa drivers fully implement the Vulkan KHR ray tracing stack: **RADV** (for AMD RDNA2 and later, i.e. RX 6000 series and above) and **ANV** (for Intel DG2/Arc Alchemist and later Xe-HPG hardware). NVIDIA's proprietary driver also supports these extensions on Linux via `vulkan-nvidia`.

The key extension set consists of:

- `VK_KHR_ray_tracing_pipeline` — the full ray tracing pipeline with dedicated shader stages
- `VK_KHR_acceleration_structure` — two-level BVH acceleration structures
- `VK_KHR_ray_query` — inline ray intersection queries from any shader stage
- `VK_KHR_deferred_host_operations` — parallelised CPU-side BVH construction

Unlike CUDA/OptiX, which are NVIDIA-only, Vulkan ray tracing runs on all three GPU vendors on Linux. This chapter covers:

- **hardware architecture of ray traversal units**
- **complete Vulkan API** — building acceleration structures and ray tracing pipelines
- **GLSL/SPIR-V shader model**
- **RADV and ANV driver implementations**

[Vulkan Ray Tracing specification](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#ray-tracing)

---

## Ray Tracing Pipeline and Shader Stages

### Motivation: Hybrid Rendering

Modern games and renderers use hybrid rendering: rasterisation for primary visibility (G-buffer) and ray tracing for high-quality shadows, ambient occlusion, reflections, and global illumination. Ray tracing excels at effects that are global in nature (effects that require knowledge of scene geometry beyond the current fragment), which are expensive or impossible with rasterisation alone.

### Five New Shader Stages

`VK_KHR_ray_tracing_pipeline` introduces five specialised shader stages:

| Stage | Abbreviation | Purpose |
|---|---|---|
| Ray Generation | `rgen` | Entry point; calls `traceRayEXT()` |
| Intersection | `rint` | Custom primitive intersection test |
| Any-Hit | `rahit` | Called for each potential hit (alpha-testing, semi-transparency) |
| Closest Hit | `rchit` | Called for the confirmed closest hit |
| Miss | `rmiss` | Called when no geometry is hit |

These stages are compiled into a ray tracing pipeline with `vkCreateRayTracingPipelinesKHR`.

### A Minimal Ray Generation Shader (GLSL)

```glsl
#version 460
#extension GL_EXT_ray_tracing : require

layout(set = 0, binding = 0) uniform accelerationStructureEXT topLevelAS;
layout(set = 0, binding = 1, rgba8) uniform image2D outputImage;

layout(location = 0) rayPayloadEXT vec3 hitColor;

void main() {
    ivec2 pixel = ivec2(gl_LaunchIDEXT.xy);
    vec2 uv = (vec2(pixel) + 0.5) / vec2(gl_LaunchSizeEXT.xy);

    vec3 origin = vec3(uv * 2.0 - 1.0, -1.0);
    vec3 direction = vec3(0.0, 0.0, 1.0);

    traceRayEXT(
        topLevelAS,           // acceleration structure
        gl_RayFlagsOpaqueEXT, // ray flags
        0xFF,                 // cull mask
        0,                    // sbt record offset
        0,                    // sbt record stride
        0,                    // miss index
        origin,               // ray origin
        0.001,                // t_min
        direction,            // ray direction
        10000.0,              // t_max
        0                     // payload location
    );

    imageStore(outputImage, pixel, vec4(hitColor, 1.0));
}
```

### Closest Hit Shader (GLSL)

```glsl
#version 460
#extension GL_EXT_ray_tracing : require

layout(location = 0) rayPayloadInEXT vec3 hitColor;

hitAttributeEXT vec2 baryCoord;

void main() {
    // gl_HitTEXT is the distance, gl_PrimitiveID the triangle index
    vec3 bary = vec3(1.0 - baryCoord.x - baryCoord.y, baryCoord.x, baryCoord.y);
    hitColor = bary; // visualise barycentric coordinates
}
```

### Miss Shader

```glsl
#version 460
#extension GL_EXT_ray_tracing : require

layout(location = 0) rayPayloadInEXT vec3 hitColor;

void main() {
    hitColor = vec3(0.1, 0.2, 0.4); // sky colour
}
```

### Dispatching Rays

```c
vkCmdBindPipeline(cmd, VK_PIPELINE_BIND_POINT_RAY_TRACING_KHR, rtPipeline);
vkCmdBindDescriptorSets(cmd, VK_PIPELINE_BIND_POINT_RAY_TRACING_KHR, ...);

vkCmdTraceRaysKHR(
    cmd,
    &rgenSBTRegion,   // shader binding table: ray gen
    &missSBTRegion,   // miss
    &hitSBTRegion,    // hit group
    &callableSBTRegion,
    width, height, 1  // dispatch dimensions
);
```

[GL_EXT_ray_tracing GLSL extension specification](https://github.com/KhronosGroup/GLSL/blob/main/extensions/ext/GLSL_EXT_ray_tracing.txt)

---

## Acceleration Structures: BLAS and TLAS

### Two-Level BVH Hierarchy

Vulkan ray tracing uses a two-level acceleration structure:

- **Bottom-Level Acceleration Structure (BLAS)**: contains the actual geometry (triangles or AABBs for custom primitives). One BLAS per mesh or object.
- **Top-Level Acceleration Structure (TLAS)**: contains instances of BLASes with 3×4 transformation matrices, instance IDs, and SBT offsets.

This two-level design enables efficient instancing (one BLAS, many TLAS instances) and dynamic updates to transformations without rebuilding BLASes.

### Building a BLAS

```c
VkAccelerationStructureGeometryKHR geometry = {
    .sType = VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_GEOMETRY_KHR,
    .geometryType = VK_GEOMETRY_TYPE_TRIANGLES_KHR,
    .geometry.triangles = {
        .sType = VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_GEOMETRY_TRIANGLES_DATA_KHR,
        .vertexFormat = VK_FORMAT_R32G32B32_SFLOAT,
        .vertexData.deviceAddress = vertexBufferAddress,
        .vertexStride = sizeof(Vertex),
        .maxVertex = vertexCount - 1,
        .indexType = VK_INDEX_TYPE_UINT32,
        .indexData.deviceAddress = indexBufferAddress,
    },
    .flags = VK_GEOMETRY_OPAQUE_BIT_KHR,
};

VkAccelerationStructureBuildGeometryInfoKHR buildInfo = {
    .sType = VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_BUILD_GEOMETRY_INFO_KHR,
    .type = VK_ACCELERATION_STRUCTURE_TYPE_BOTTOM_LEVEL_KHR,
    .flags = VK_BUILD_ACCELERATION_STRUCTURE_PREFER_FAST_TRACE_BIT_KHR,
    .mode = VK_BUILD_ACCELERATION_STRUCTURE_MODE_BUILD_KHR,
    .geometryCount = 1,
    .pGeometries = &geometry,
};

uint32_t primitiveCount = indexCount / 3;
VkAccelerationStructureBuildSizesInfoKHR sizeInfo = {
    .sType = VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_BUILD_SIZES_INFO_KHR,
};
vkGetAccelerationStructureBuildSizesKHR(
    device, VK_ACCELERATION_STRUCTURE_BUILD_TYPE_DEVICE_KHR,
    &buildInfo, &primitiveCount, &sizeInfo);

// Allocate scratch + result buffers, create VkAccelerationStructureKHR...

VkAccelerationStructureBuildRangeInfoKHR range = {
    .primitiveCount = primitiveCount,
    .primitiveOffset = 0,
    .firstVertex = 0,
    .transformOffset = 0,
};
const VkAccelerationStructureBuildRangeInfoKHR *pRange = &range;

vkCmdBuildAccelerationStructuresKHR(cmd, 1, &buildInfo, &pRange);
```

### Building a TLAS

Each TLAS instance is a 64-byte `VkAccelerationStructureInstanceKHR`:

```c
VkAccelerationStructureInstanceKHR instance = {
    // 3×4 row-major transform matrix (identity)
    .transform = {
        .matrix = {
            {1.0f, 0.0f, 0.0f, 0.0f},
            {0.0f, 1.0f, 0.0f, 0.0f},
            {0.0f, 0.0f, 1.0f, 0.0f},
        }
    },
    .instanceCustomIndex = 0,         // gl_InstanceCustomIndexEXT
    .mask = 0xFF,                     // cull mask
    .instanceShaderBindingTableRecordOffset = 0,
    .flags = VK_GEOMETRY_INSTANCE_TRIANGLE_FACING_CULL_DISABLE_BIT_KHR,
    .accelerationStructureReference = blasDeviceAddress,
};
```

### Build Flags

| Flag | Effect |
|---|---|
| `PREFER_FAST_TRACE` | Optimise BVH quality at build-time cost; best for static geometry |
| `PREFER_FAST_BUILD` | Faster build, lower traversal quality; good for deformable geometry |
| `ALLOW_UPDATE` | Enable incremental `UPDATE` mode re-fit |
| `ALLOW_COMPACTION` | Enable post-build compaction query |

### BVH Compaction

After building, compact to reduce VRAM usage (typically 30–60% size reduction):

```c
// Query compacted size
VkQueryPool compactPool;
vkCmdWriteAccelerationStructuresPropertiesKHR(
    cmd, 1, &blas,
    VK_QUERY_TYPE_ACCELERATION_STRUCTURE_COMPACTED_SIZE_KHR,
    compactPool, 0);

// After pipeline barrier + readback:
VkDeviceSize compactedSize;
vkGetQueryPoolResults(device, compactPool, 0, 1,
    sizeof(VkDeviceSize), &compactedSize, sizeof(VkDeviceSize),
    VK_QUERY_RESULT_64_BIT | VK_QUERY_RESULT_WAIT_BIT);

// Allocate smaller buffer, copy-compact
vkCmdCopyAccelerationStructureKHR(cmd, &(VkCopyAccelerationStructureInfoKHR){
    .sType = VK_STRUCTURE_TYPE_COPY_ACCELERATION_STRUCTURE_INFO_KHR,
    .src = blas,
    .dst = compactedBlas,
    .mode = VK_COPY_ACCELERATION_STRUCTURE_MODE_COMPACT_KHR,
});
```

[VK_KHR_acceleration_structure extension spec](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_acceleration_structure.html)

---

## BVH Construction Algorithms

### Surface Area Heuristic (SAH)

The gold standard BVH construction strategy. For each split candidate, compute:

```
cost(split) = C_trav + (SA_left/SA_parent) * N_left * C_isect
                     + (SA_right/SA_parent) * N_right * C_isect
```

where `SA` is surface area and `C_trav`, `C_isect` are traversal and intersection costs. GPU-side SAH requires O(N log N) sorting and binned approximations for real-time construction. Used by AMD's offline BVH builder and Embree.

### LBVH (Linear BVH)

Assigns Morton codes to primitive centroids, radix-sorts them, and constructs a hierarchical binary tree from the sorted order. GPU-parallel; O(N) after sort. Used as the fast-build path in RADV.

### PLOC (Parallel Locally-Ordered Clustering)

Iterative bottom-up approach that clusters nearby leaves. Better quality than LBVH, nearly as fast. Used in RADV's high-quality build path.

### RADV GPU BVH Builder

RADV implements BVH construction as GPU compute shaders in `src/amd/vulkan/bvh/` [Source](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/amd/vulkan/bvh). The pipeline consists of:

1. **Morton code generation** (`lbvh_generate_ir.comp.glsl`): compute Morton codes for centroids
2. **Radix sort** (`radix_sort.comp.glsl`): 4-pass 8-bit radix sort
3. **Hierarchy generation** (`lbvh_main.comp.glsl`): Karras 2012 binary radix tree
4. **Collapse** (`ploc.comp.glsl` or SAH refinement): improve quality
5. **Encode** (`encode.comp.glsl`): convert to AMD hardware BVH format

The hardware BVH node format for RDNA is not publicly documented; RADV reverse-engineered it via `src/amd/vulkan/bvh/bvh.h`.

### ANV GRL (Generic Ray Tracing Library)

Intel's ANV driver uses the **GRL** (Generic Ray Tracing Library), open-sourced as part of Mesa at `src/intel/vulkan/grl/` [Source](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/intel/vulkan/grl). GRL kernels are written in an OpenCL-like C dialect and compiled to Intel GPU compute shaders. Intel's Synchro BVH builder (a state-of-the-art PLOC variant) has been open-sourced since Xe2.

### Deferred Host Operations

`VK_KHR_deferred_host_operations` allows CPU-side BVH construction to be split across threads:

```c
VkDeferredOperationKHR deferredOp;
vkCreateDeferredOperationKHR(device, NULL, &deferredOp);

VkResult result = vkBuildAccelerationStructuresKHR(
    device, deferredOp, 1, &buildInfo, &pRange);
// result == VK_OPERATION_DEFERRED_KHR

// Join from worker threads:
while (vkDeferredOperationJoinKHR(device, deferredOp) == VK_THREAD_IDLE_KHR) {
    // more work available — other threads may call join too
}
```

---

### Self-Intersection Avoidance

A persistent practical problem in any ray tracer is **self-intersection**: a shadow or reflection ray originating from a surface hits the same triangle that generated it, producing false occlusion (surface acne) or incorrect self-shadowing. This happens because floating-point arithmetic places the ray origin slightly inside the surface.

NVIDIA Research published reference GLSL/HLSL code ([github.com/NVIDIA/self-intersection-avoidance](https://github.com/NVIDIA/self-intersection-avoidance)) for a geometrically correct offset method that pushes the origin along the surface normal by an amount proportional to the geometric error at that point — adaptive rather than a fixed epsilon, which either clips geometry at grazing angles or fails at large-distance geometry.

```glsl
/* Self-intersection avoidance — GLSL adaptation of NVIDIA's reference code.
   Source: github.com/NVIDIA/self-intersection-avoidance                    */

/* Offset a position along a normal to avoid self-intersection.
   p: hit position (object space), n: geometric normal (object space),
   Returns safe ray origin for reflection / shadow rays.                    */
vec3 offsetPositionAlongNormal(vec3 p, vec3 n) {
    /* The integer offset technique avoids FP cancellation error. */
    const float origin      = 1.0 / 32.0;
    const float float_scale = 1.0 / 65536.0;
    const float int_scale   = 256.0;

    ivec3 of_i = ivec3(int_scale * n.x, int_scale * n.y, int_scale * n.z);
    vec3 p_i = vec3(
        intBitsToFloat(floatBitsToInt(p.x) + ((p.x < 0) ? -of_i.x : of_i.x)),
        intBitsToFloat(floatBitsToInt(p.y) + ((p.y < 0) ? -of_i.y : of_i.y)),
        intBitsToFloat(floatBitsToInt(p.z) + ((p.z < 0) ? -of_i.z : of_i.z)));

    return vec3(
        abs(p.x) < origin ? p.x + float_scale * n.x : p_i.x,
        abs(p.y) < origin ? p.y + float_scale * n.y : p_i.y,
        abs(p.z) < origin ? p.z + float_scale * n.z : p_i.z);
}

/* Usage in a closest-hit shader: */
layout(location = 0) rayPayloadInEXT Payload payload;

void main() {
    vec3 P = gl_WorldRayOriginEXT + gl_HitTEXT * gl_WorldRayDirectionEXT;
    vec3 N = /* interpolated geometric normal at hit */;

    /* Shadow ray: offset origin to avoid self-shadowing */
    vec3 shadowOrigin = offsetPositionAlongNormal(P, N);
    traceRayEXT(topLevelAS, gl_RayFlagsOpaqueEXT,
                0xFF, 0, 0, 0,
                shadowOrigin, 0.001, lightDir, 1e6, 1 /* shadow payload */);
}
```

The key insight is using integer arithmetic on the float representation: adding an integer offset to the mantissa/exponent of a float is equivalent to shifting by a number of ULPs (Units in the Last Place) proportional to the float's magnitude — inherently adaptive to the scale of the geometry.

[Source: NVIDIA self-intersection-avoidance GitHub](https://github.com/NVIDIA/self-intersection-avoidance)
[Source: NVIDIA blog — A Fast and Robust Method for Avoiding Self-Intersection](https://developer.nvidia.com/blog/solving-self-intersection-artifacts-in-directx-raytracing/)

---

## Shader Binding Table

### Layout

The Shader Binding Table (SBT) is a `VkBuffer` divided into four regions, each containing fixed-size records aligned to `shaderGroupHandleAlignment` (typically 32 or 64 bytes):

```
[ rgen region   ]  — exactly 1 record
[ miss region   ]  — N_miss records (indexed by traceRayEXT miss index)
[ hit group region ] — N_hit records (indexed by instance SBT offset + geometry index)
[ callable region]  — N_callable records
```

Each record = 32-byte shader group handle + user-defined data (materials, textures, etc.).

### Building the SBT

```c
// Query handle size
VkPhysicalDeviceRayTracingPipelinePropertiesKHR rtProps = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_RAY_TRACING_PIPELINE_PROPERTIES_KHR,
};
VkPhysicalDeviceProperties2 props2 = { .pNext = &rtProps, ... };
vkGetPhysicalDeviceProperties2(physicalDevice, &props2);

uint32_t handleSize = rtProps.shaderGroupHandleSize; // 32 bytes

// Get all handles
std::vector<uint8_t> handles(shaderGroupCount * handleSize);
vkGetRayTracingShaderGroupHandlesKHR(
    device, rtPipeline, 0, shaderGroupCount,
    handles.size(), handles.data());

// Copy rgen handle to SBT buffer at rgen offset
// Copy miss handles at aligned miss offset
// Copy hit group handles at aligned hit offset
```

### Hit Group Indexing Formula

When a ray hits instance `I` at geometry index `G`:

```
hitGroupIndex = instance.instanceShaderBindingTableRecordOffset
              + geometryIndex * sbtStride  (passed to vkCmdTraceRaysKHR)
              + traceRayEXT sbtRecordOffset parameter
```

This allows per-material shaders: each geometry within a BLAS can have a different hit group, and each TLAS instance can offset into a different set of hit groups.

---

## RADV: RDNA2+ Ray Tracing

### Hardware Support

RADV supports ray tracing on RDNA2 (RX 6000 series, `gfx1030`) and later. RDNA2 adds dedicated BVH traversal instructions in the shader ISA:

- `image_bvh_intersect_ray` — tests a ray against a BVH node
- `image_bvh64_intersect_ray` — 64-bit address variant

These instructions are emitted by the LLVM AMD backend and used by RADV's traversal shaders.

### Key Source Files

| File | Role |
|---|---|
| `src/amd/vulkan/radv_acceleration_structure.c` | BLAS/TLAS build, compaction, copy |
| `src/amd/vulkan/radv_rt_pipeline.c` | RT pipeline creation, SBT layout |
| `src/amd/vulkan/radv_nir_lower_ray_queries.c` | Inline ray query lowering |
| `src/amd/vulkan/bvh/` | GPU BVH construction compute shaders |
| `src/amd/vulkan/radv_nir_lower_rt_io.c` | Lower SPIR-V RT built-ins to SSA |

[RADV RT source](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/amd/vulkan)

### RT Pipeline as Compute

RADV compiles the entire ray tracing pipeline into a **single compute shader** that emulates the RT state machine in software, using `image_bvh_intersect_ray` for hardware-accelerated BVH traversal. The traversal loop, any-hit, and intersection shaders are all inlined into one kernel.

`radv_build_rt_shader()` in `radv_rt_pipeline.c` generates this mega-kernel by:

1. Emitting a traversal loop that calls `image_bvh_intersect_ray`
2. Calling NIR functions corresponding to each RCHIT/RAHIT/RMISS shader group
3. Lowering RT payload access to SSA via `radv_nir_lower_rt_io`

### Debugging RADV RT

```bash
# Force software emulation of ray tracing (any RDNA GPU):
RADV_PERFTEST=emulate_rt vulkaninfo

# Dump RT shader IR:
RADV_DEBUG=spirv,shaders RADV_RT_DEBUG=1 ./rt_app

# Disable mesh shader path (unrelated but useful for isolation):
RADV_DEBUG=nort
```

[RADV debugging options](https://docs.mesa3d.org/drivers/radv.html#environment-variables)

---

## ANV: Intel Xe-HPG Ray Tracing

### Hardware Support

ANV supports ray tracing on Intel DG2 (Arc Alchemist, `gfx1255`) and later Xe-HPG and Xe2 hardware. Intel's ray tracing hardware is a dedicated BVH traversal unit on each render slice, exposed through EU (Execution Unit) messages.

### Key Source Files

| File | Role |
|---|---|
| `src/intel/vulkan/anv_acceleration_structure.c` | BLAS/TLAS build |
| `src/intel/vulkan/grl/` | GRL BVH construction kernels |
| `src/intel/vulkan/anv_nir_lower_ray_queries.c` | Inline ray query lowering |
| `src/intel/vulkan/genX_cmd_buffer.c` | `genX(cmd_buffer_trace_rays)` |

[ANV source](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/intel/vulkan)

### GRL Kernels

The Generic Ray Tracing Library at `src/intel/vulkan/grl/` contains BVH construction kernels written in Intel's OpenCL-C variant. These are compiled via Intel's offline compiler to Intel GPU binary at Mesa build time. The Synchro builder implements a parallel PLOC+SAH pipeline:

1. Centroid computation
2. Morton code assignment
3. Radix sort
4. PLOC clustering
5. SAH quality refinement
6. Encode to Intel hardware BVH format

Intel's 64-byte BVH node format is defined in the [Intel Xe-HPG BVH documentation](https://www.intel.com/content/www/us/en/developer/articles/technical/introduction-to-ray-tracing-in-vulkan.html).

### Debugging ANV RT

```bash
INTEL_DEBUG=rt,spirv ./rt_app
# Or per-stage debug:
ANV_ENABLE_PIPELINE_CACHE=0 INTEL_DEBUG=vs,cs ./rt_app
```

---

## VK_KHR_ray_query and Inline Ray Tracing

### Inline Ray Intersection from Any Stage

`VK_KHR_ray_query` allows firing rays from **any** shader stage (vertex, fragment, compute, mesh) without a full RT pipeline:

```glsl
#version 460
#extension GL_EXT_ray_query : require

layout(set=0, binding=0) uniform accelerationStructureEXT topLevelAS;

// In a fragment shader — ambient occlusion:
float traceAO(vec3 pos, vec3 normal) {
    rayQueryEXT query;
    rayQueryInitializeEXT(
        query, topLevelAS,
        gl_RayFlagsTerminateOnFirstHitEXT | gl_RayFlagsOpaqueEXT,
        0xFF,
        pos + normal * 0.001,  // offset to avoid self-intersection
        0.001,                 // t_min
        normal,                // fire along normal (hemisphere AO)
        1.0                    // t_max
    );

    while (rayQueryProceedEXT(query)) {
        if (rayQueryGetIntersectionTypeEXT(query, false)
                == gl_RayQueryCandidateIntersectionTriangleEXT) {
            rayQueryConfirmIntersectionEXT(query);
        }
    }

    return (rayQueryGetIntersectionTypeEXT(query, true)
            == gl_RayQueryCommittedIntersectionNoneEXT) ? 1.0 : 0.0;
}
```

### VK_KHR_ray_tracing_maintenance1

Added utility features:

- `VK_RAY_TRACING_INVOCATION_REORDER_MODE_REORDER_NV` (NVIDIA-specific): shader execution reordering (SER) for better RT throughput
- `vkCmdTraceRaysIndirectKHR` — dispatch count from GPU buffer
- `VkTraceRaysIndirectCommand2KHR` — full indirect with SBT addresses

### Software Fallback via SSBO BVH

For hardware that lacks RT support (older RDNA1, Intel Gen11), a software BVH traversal in compute using SSBOs is feasible but 5–20× slower than hardware. Useful for development:

```glsl
// Software BVH node traversal (conceptual)
layout(std430, binding=1) readonly buffer BVHNodes { BVHNode nodes[]; };

bool intersectBVH(Ray ray) {
    uint stack[64]; int top = 0;
    stack[top++] = 0; // root node
    while (top > 0) {
        BVHNode n = nodes[stack[--top]];
        if (!intersectAABB(ray, n.aabb)) continue;
        if (n.isLeaf) return intersectTriangles(ray, n);
        stack[top++] = n.leftChild;
        stack[top++] = n.rightChild;
    }
    return false;
}
```

### Intel Open Image Denoise

Post-ray tracing denoising for noisy 1 spp or 4 spp images. [Intel OIDN](https://www.openimagedenoise.org/) runs on CPU (AVX-512) or GPU (Xe compute). Vulkan interop via `oidnNewExternalBuffer` (DMA-BUF / Vulkan external memory).

---

## Practical Usage on Linux

### Checking RT Support

```bash
vulkaninfo 2>/dev/null | grep -i -A2 "ray"
# Look for rayTracingPipeline: VK_TRUE, accelerationStructure: VK_TRUE

# Or with jq:
vulkaninfo --json | jq '.[] | .features | .rayTracingPipeline // .VkPhysicalDeviceRayTracingPipelineFeaturesKHR'
```

### Feature and Property Queries

```c
VkPhysicalDeviceRayTracingPipelineFeaturesKHR rtFeatures = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_RAY_TRACING_PIPELINE_FEATURES_KHR,
};
VkPhysicalDeviceAccelerationStructureFeaturesKHR asFeatures = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_ACCELERATION_STRUCTURE_FEATURES_KHR,
    .pNext = &rtFeatures,
};

VkPhysicalDeviceRayTracingPipelinePropertiesKHR rtProps = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_RAY_TRACING_PIPELINE_PROPERTIES_KHR,
};
// After vkGetPhysicalDeviceProperties2:
// rtProps.maxRayRecursionDepth — minimum 1, typically 31
// rtProps.shaderGroupHandleSize — 32 bytes
// rtProps.shaderGroupHandleAlignment — 32 or 64 bytes
// rtProps.maxRayHitAttributeSize — 32 bytes
// rtProps.maxRayDispatchInvocationCount — typically 2^30
```

### Compiling RT Shaders

```bash
# glslc (from Shaderc) with Vulkan 1.2 target:
glslc --target-env=vulkan1.2 -fshader-stage=rgen raygen.rgen.glsl -o raygen.spv
glslc --target-env=vulkan1.2 -fshader-stage=rchit closesthit.rchit.glsl -o closesthit.spv
glslc --target-env=vulkan1.2 -fshader-stage=rmiss miss.rmiss.glsl -o miss.spv

# Validate with spirv-val:
spirv-val --target-env vulkan1.2 raygen.spv

# Disassemble:
spirv-dis raygen.spv | grep -i "traceray\|optype"
```

### Real-World Usage on Linux

- **Blender Cycles** (2025): Vulkan RT backend using `VK_KHR_ray_tracing_pipeline` for GPU path tracing on RADV and ANV
- **Godot 4.x**: Global Illumination via `VK_KHR_ray_query` in screen-space GI shaders
- **Q2RTX** (Quake II RT): NVIDIA-specific but ported to run on RADV RDNA2 via Mesa; `VK_KHR_ray_tracing_pipeline`
- **Vulkan-Samples** (Khronos): reference implementations at [https://github.com/KhronosGroup/Vulkan-Samples](https://github.com/KhronosGroup/Vulkan-Samples)
- **vkdt** (darktable Vulkan): uses ray query for lens simulation

### Performance Notes

On RDNA2 (RX 6700 XT):
- BVH build: ~500 ms for 1M triangle scene (GPU LBVH), ~1.5 s (GPU SAH)
- RT throughput: ~5–8 Grays/s for primary ray + shadow ray scene
- Compare NVIDIA RTX 3070: ~12–15 Grays/s

Compaction reduces BLAS memory 40–50%; critical for complex scenes. Use `PREFER_FAST_TRACE` for static geometry, `PREFER_FAST_BUILD` + `ALLOW_UPDATE` for skinned meshes.

---

## SVGF: Spatiotemporal Variance-Guided Filtering

**SVGF** (Schied et al. 2017, [arxiv.org/abs/1812.09904](https://arxiv.org/abs/1812.09904)) is the canonical open-source GPU denoiser for single-sample-per-pixel ray tracing. It combines temporal accumulation with spatial à-trous wavelet filtering guided by estimated per-pixel variance. NRD's REBLUR algorithm (Ch70 §5) is a direct descendant. The reference implementation is at [github.com/christoph-schied/spatiotemporal-variance-guided-filtering](https://github.com/christoph-schied/spatiotemporal-variance-guided-filtering).

NRD (Ch70 §5) is production-quality but NVIDIA-hosted. Intel OIDN is open-source but CPU/Xe-oriented. SVGF fills the gap: a fully open, GPU-native, Vulkan-implementable denoiser that runs on any hardware with ray tracing support at 1–2 ms per frame at 1080p.

### Stage 1: Temporal Accumulation

Each frame, the previous denoised frame is reprojected using motion vectors. Moments (luminance M1 = E[x] and M2 = E[x²]) are accumulated in a separate buffer to estimate per-pixel variance:

```glsl
// SVGF temporal accumulation — compute shader
layout(binding = 0) uniform sampler2D uInput;       // noisy 1spp input
layout(binding = 1) uniform sampler2D uPrevColor;   // denoised previous frame
layout(binding = 2) uniform sampler2D uPrevMoments; // xy = M1,M2; z = history length
layout(binding = 3) uniform sampler2D uVelocity;    // screen-space UV offset
layout(binding = 4) uniform sampler2D uDepth;
layout(binding = 5) uniform sampler2D uPrevDepth;
layout(binding = 6) uniform sampler2D uNormal;
layout(binding = 7) uniform sampler2D uPrevNormal;

const float ALPHA_C = 0.2;    // blend weight for current frame colour
const float ALPHA_M = 0.1;    // blend weight for moments

void main() {
    vec2 vel    = texture(uVelocity, texCoord).xy;
    vec2 prevUV = texCoord - vel;

    vec3  cur   = texture(uInput, texCoord).rgb;
    float curL  = dot(cur, vec3(0.2126, 0.7152, 0.0722));

    // Depth + normal consistency tests reject invalid history
    float z     = texture(uDepth,     texCoord).r;
    float pz    = texture(uPrevDepth, prevUV).r;
    vec3  n     = texture(uNormal,    texCoord).xyz;
    vec3  pn    = texture(uPrevNormal,prevUV).xyz;
    bool  valid = all(greaterThan(prevUV,vec2(0))) && all(lessThan(prevUV,vec2(1)))
               && (abs(z-pz) < 0.1*z) && (dot(n,pn) > 0.95);

    float hist  = valid ? texture(uPrevMoments,prevUV).z + 1.0 : 1.0;
    float ac    = valid ? max(ALPHA_C, 1.0/hist) : 1.0;
    float am    = valid ? max(ALPHA_M, 1.0/hist) : 1.0;

    outColor    = mix(texture(uPrevColor,prevUV).rgb, cur, ac);
    float m1    = mix(texture(uPrevMoments,prevUV).x, curL,    am);
    float m2    = mix(texture(uPrevMoments,prevUV).y, curL*curL, am);
    outMoments  = vec3(m1, m2, hist);  // variance = m2 - m1*m1
}
```

### Stage 2: Spatial À-Trous Wavelet Filtering

Five iterations of a B3-spline à-trous wavelet double the step size each pass (1, 2, 4, 8, 16 pixels), covering a 65×65 footprint from a 5-tap kernel. Three **edge-stopping weights** prevent blurring across depth discontinuities, geometry edges, and high-variance regions:

```glsl
// SVGF à-trous wavelet pass — one of 5 iterations
layout(push_constant) uniform PC { int stepWidth; };  // 1,2,4,8,16

const float h[3] = float[](3.0/8.0, 1.0/4.0, 1.0/16.0);  // B3 spline weights
const float SIGMA_Z = 1.0;    // depth sensitivity
const float SIGMA_N = 128.0;  // normal sensitivity (exponent)

void main() {
    vec3  cCol = texture(uInput,    texCoord).rgb;
    float cL   = dot(cCol, vec3(0.2126, 0.7152, 0.0722));
    float cZ   = texture(uDepth,   texCoord).r;
    vec3  cN   = texture(uNormal,  texCoord).xyz;
    float var  = texture(uVariance,texCoord).r;
    float sigL = 4.0 * sqrt(max(0.0, var));  // luminance sensitivity from variance

    vec3 sumC = vec3(0); float sumW = 0.0;

    for (int i = -2; i <= 2; ++i) {
        vec2  uv  = texCoord + vec2(i,0)*float(stepWidth)*texelSize;  // horizontal
        vec3  sC  = texture(uInput,   uv).rgb;
        float sL  = dot(sC, vec3(0.2126,0.7152,0.0722));
        float sZ  = texture(uDepth,  uv).r;
        vec3  sN  = texture(uNormal, uv).xyz;

        float wZ  = exp(-abs(cZ-sZ) / (SIGMA_Z * abs(float(i)*dFdx(cZ)) + 1e-4));
        float wN  = pow(max(0.0, dot(cN,sN)), SIGMA_N);
        float wL  = exp(-abs(cL-sL) / (sigL + 1e-4));

        float w   = h[abs(i)] * wZ * wN * wL;
        sumC += sC * w;  sumW += w;
    }
    outColor = sumC / max(sumW, 1e-4);
}
// Repeat with vertical offsets for the separable vertical pass each iteration
```

After 5 iterations (horizontal + vertical each), the filter has an effective support of 65×65 pixels. Variance is re-estimated from the accumulated moments after each pass to adaptively scale `sigL`.

### SVGF vs NRD vs Intel OIDN

| Feature | SVGF | NRD v4.17 (RELAX) | Intel OIDN 2.x |
|---------|------|-------------------|----------------|
| Algorithm | Variance-guided à-trous wavelet | Adaptive bilateral + temporal | Regression CNN |
| G-buffer | Normal, depth, velocity | Normal, depth, velocity, albedo, hit-dist | Normal, albedo (optional) |
| Open source | Yes (BSD) | Yes (MIT) | Yes (Apache 2.0) |
| Latency 1080p | 1–2 ms | 1.5–3 ms | <2 ms (Xe GPU), 5–20 ms CPU |
| Quality | Good | Very good | Excellent |
| GPU requirement | Any Vulkan compute | Any Vulkan compute | Intel Xe or any CUDA/CPU |
| ReSTIR inputs | Manual tuning | RELAX mode (dedicated) | N/A |

```bash
# Build and run the SVGF reference renderer (Vulkan + GLFW)
git clone https://github.com/christoph-schied/spatiotemporal-variance-guided-filtering
cd spatiotemporal-variance-guided-filtering
cmake -B build -DCMAKE_BUILD_TYPE=Release && cmake --build build -j$(nproc)
./build/svgf --scene data/cornell_box.obj   # 1spp path tracer + SVGF denoiser
```

---

## Roadmap

### Near-term (6–12 months)

- **`VK_EXT_ray_tracing_invocation_reorder` (SER) broad adoption**: Shader Execution Reordering, which graduated from the NVIDIA-specific `VK_NV_ray_tracing_invocation_reorder` to a multi-vendor EXT extension, is expected to land in RADV and other open-source Mesa drivers after already shipping for NVIDIA and Intel Arc B-series hardware; benchmarks on the Vulkan glTF path tracer show up to 47% throughput improvement. [Source](https://www.khronos.org/blog/boosting-ray-tracing-performance-with-shader-execution-reordering-introducing-vk-ext-ray-tracing-invocation-reorder)
- **Mesa 26.x RADV ray tracing optimisations**: Mesa 26.0 (released February 2026) introduced pipeline compilation changes and reduced shader inlining that delivered >2x faster ray tracing passes in titles like Ghostwire Tokyo; subsequent 26.x releases continue this tuning trajectory, driven in part by Valve's graphics team. [Source](https://www.phoronix.com/news/Mesa-26.0-Released)
- **Vulkan Roadmap 2026 conformance push**: The Vulkan Working Group published Roadmap 2026 milestone requirements (announced January 2026), raising mandatory feature and limit baselines beyond Vulkan 1.4; driver teams are aligning conformance test suites with these raised requirements, which indirectly validates RT extension correctness in CTS runs. [Source](https://www.khronos.org/blog/vulkan-introduces-roadmap-2026-and-new-descriptor-heap-extension)
- **Blender Cycles Vulkan RT stabilisation**: The Vulkan ray tracing backend for Blender Cycles, which uses `VK_KHR_ray_tracing_pipeline` on RADV and ANV, is expected to move from experimental to default for Linux GPU rendering as Mesa RT path stability improves. (Note: needs verification for exact release milestone)

### Medium-term (1–3 years)

- **Ray tracing in Vulkan Roadmap mandatory tier**: While ray tracing extensions are not yet mandatory in Roadmap 2026, industry discussion is exploring a future milestone that would require `VK_KHR_ray_query` on all high-end Vulkan implementations; this would effectively mandate RT support on all discrete-class GPUs shipping with Vulkan conformance. (Note: needs verification — no firm Khronos announcement as of mid-2026)
- **Hardware-accelerated BVH construction on RDNA4 / future Intel Xe**: Current open-source Mesa drivers implement BVH build on compute shaders; future AMD and Intel hardware generations may expose dedicated BVH build hardware, enabling `VK_ACC_STRUCT_BUILD_FLAGS` fast-build paths an order of magnitude faster than LBVH compute. [Source](https://www.khronos.org/news/archives/new-vulkan-extension-boosts-ray-tracing-performance)
- **`VK_NV_displacement_micromap` / AMD equivalent standardisation**: Micro-mesh displacement maps (NVIDIA Ada feature) allow compact high-detail geometry in BVHs; a cross-vendor KHR or EXT standardisation is expected as AMD RDNA4 and Intel Xe2 both target micro-triangle hardware. (Note: needs verification — no public KHR draft confirmed)
- **NVK (Nouveau Vulkan) ray tracing support**: The NVK open-source Vulkan driver for NVIDIA hardware is approaching feature-completeness for Vulkan 1.3 core; RT extension support (`VK_KHR_ray_tracing_pipeline`) is a natural next milestone once the core pipeline model stabilises. [Source](https://www.gamingonlinux.com/2026/02/mesa-26-0-is-out-bringing-ray-tracing-performance-improvements-for-amd-radv/)

### Long-term

- **Unified ray tracing + mesh shader pipeline integration**: Architectural proposals in the Khronos working group discussion explore combining mesh shaders (Task + Mesh stages) with ray tracing dispatch to allow procedural geometry generation at BVH-build time, eliminating the CPU round-trip for dynamic scenes. (Note: speculative — no public RFC as of mid-2026)
- **Neural radiance cache / AI denoising standardisation in Vulkan**: As GPU-integrated AI inference accelerators become standard (e.g., NPU-on-die in AMD RDNA4, Intel Xe3), a future Vulkan cooperative matrix or inference extension may standardise the denoiser workload that currently relies on vendor-specific libraries (DLSS-RR, XeSS, FSR4). (Note: speculative direction)
- **Ray tracing on integrated GPUs and mobile-class Vulkan**: The `VK_KHR_ray_query` inline path is simpler to implement than the full pipeline; long-term roadmap discussion targets bringing ray query to AMD RDNA integrated graphics (Phoenix/Hawk Point) and Intel Arc integrated, enabling mainstream Linux desktop ray query without discrete GPU. (Note: needs verification for specific hardware timelines)

## Integrations

- **Ch18 (RADV driver)** — RADV architecture; RT is implemented as a compute mega-kernel using RDNA2 `image_bvh_intersect_ray` instructions
- **Ch19 (ANV driver)** — ANV GRL BVH construction and Xe-HPG hardware ray traversal unit
- **Ch24 (Vulkan API)** — Vulkan pipeline model; RT pipeline is a parallel pipeline type alongside graphics and compute
- **Ch77 (SPIR-V)** — `OpTraceRayKHR`, `OpExecuteCallableKHR`, `OpHitObjectNV` SPIR-V opcodes produced by glslc
- **Ch110 (SPIR-V tooling)** — spirv-opt for RT shader optimisation, spirv-val for validation
- **Ch127 (Mesh Shaders)** — hybrid rendering: mesh shaders for primary G-buffer, RT for shadows/reflections
- **Ch133 (Async Compute)** — RT pipelines dispatch on compute queue alongside graphics; timeline semaphore synchronisation
- **Ch141 (Cooperative Matrices)** — denoising via cooperative matrix GEMM in the post-RT denoising compute shader

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
