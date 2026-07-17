# Chapter 135: Vulkan Ray Tracing on Linux

**Target audiences**: Vulkan application developers building ray-traced rendering, game engine and rendering research engineers, Mesa driver developers (RADV/ANV), and advanced users wanting to understand how hardware ray traversal maps to the Vulkan API on Linux.

---

## Table of Contents

1. [Introduction](#introduction)
   - [What Ray Tracing Is](#what-ray-tracing-is)
   - [The Rendering Equation and Path Tracing](#the-rendering-equation-and-path-tracing)
   - [Ray Marching and SDF Tracing](#ray-marching-and-sdf-tracing)
   - [Other Light Transport Algorithms](#other-light-transport-algorithms)
   - [Why Hardware Acceleration](#why-hardware-acceleration)
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

### What Ray Tracing Is

Ray tracing is a rendering algorithm that simulates how light physically travels through a scene. The core insight — first articulated for computer graphics by Turner Whitted in his landmark 1980 paper *"An Improved Illumination Model for Shaded Display"* — is to reverse the direction of light propagation: instead of tracing photons from light sources (almost all of which would miss the camera), trace rays *backwards* from the camera through each pixel into the scene.

**The recursive ray model.** A primary ray is fired from the camera origin through a pixel. When it intersects scene geometry, the renderer evaluates the surface material and spawns secondary rays:

- **Shadow rays** — fired toward each light source to determine visibility (hard shadows are trivially correct; no shadow map biasing is needed)
- **Reflection rays** — the specularly reflected direction, for mirror-like surfaces
- **Refraction rays** — computed via Snell's law, for transparent materials
- **Diffuse (indirect) rays** — randomly sampled over the hemisphere for ambient lighting

Whitted-style ray tracing handles mirror reflections, refraction, and sharp shadows exactly. It does not handle diffuse inter-reflection (color bleeding, soft global illumination) — those require path tracing.

**Why rasterization cannot easily replace it.** Rasterization projects geometry onto the screen and evaluates shading per-fragment. It has no native notion of "what other geometry exists in direction D from this point." Achieving effects that require global scene knowledge — soft shadows, reflections that contain off-screen objects, ambient occlusion, caustics — requires indirect approximations: shadow maps, reflection probes, screen-space ray marching, precomputed lightmaps. Each approximation has visible failure cases. Ray tracing eliminates the approximations by querying the actual scene geometry.

---

### The Rendering Equation and Path Tracing

Kajiya (1986) formalized the problem in *"The Rendering Equation"*:

$$
L_o(x, \omega_o) = L_e(x, \omega_o) + \int_\Omega f_r(x, \omega_i, \omega_o)\, L_i(x, \omega_i)\, (\omega_i \cdot n)\, d\omega_i
$$

where $L_o$ is outgoing radiance, $L_e$ is emitted radiance, $f_r$ is the BRDF, and the integral is over the hemisphere. The integral has no closed form for complex scenes — Kajiya's solution was **path tracing**: Monte Carlo integration. At each hit point, randomly sample one or more directions according to the BRDF, trace a ray in each sampled direction, and recursively evaluate the integral. Average many samples per pixel until the estimator converges.

Path tracing is the basis of every physically-based renderer: PBRT, Cycles (Blender), LuxCoreRender, and the path tracer inside Unreal Engine 5's Lumen at maximum quality. It is unbiased — given enough samples, the result converges to ground truth.

**The convergence problem.** The estimator's variance (noise) decreases as $1/\sqrt{N}$ where $N$ is sample count. Achieving acceptably low noise at interactive frame rates requires either very fast ray traversal (hardware RT units) or sophisticated denoisers that use spatial and temporal information to infer a clean image from a noisy low-sample estimate. This is the motivation for SVGF (§10) and AI denoisers (OptiX AI Denoiser, ReLAX, DLSS Ray Reconstruction).

**Variants:**

| Algorithm | Description | Use case |
|-----------|-------------|----------|
| Whitted RT | Recursive, specular-only secondaries | Offline, educational |
| Path tracing | MC integration, one random bounce per hit | Production offline (Cycles, PBRT) |
| Bidirectional PT (BDPT) | Combines camera and light sub-paths | Scenes with complex indirect lighting |
| Metropolis Light Transport (MLT) | Importance-samples path space via mutations | Very dark scenes, caustics |
| Photon mapping | Forward emission pass + gather | Caustics in offline renderers |
| VCM / SPPM | Photon mapping + BDPT hybrid | State-of-the-art offline |
| ReSTIR | Resampled Importance Sampling across pixels and frames | Real-time GI (RTX 30/40 series) |

In real-time rendering, "ray tracing" usually means **hybrid rendering**: rasterize primary visibility (G-buffer) and trace one or two rays per pixel for specific effects (shadows, reflections, ambient occlusion), then denoise. Full path tracing at interactive rates became viable on high-end GPUs around 2022–2023.

---

### Ray Marching and SDF Tracing

Ray marching is a fundamentally different technique that does not require explicit triangle geometry. Instead of computing an analytical intersection, the algorithm steps along a ray in fixed or adaptive increments and evaluates a function at each step.

**Sphere marching with Signed Distance Fields (SDFs).** An SDF is a function $d(p)$ that returns the signed distance from point $p$ to the nearest surface: positive outside, negative inside, zero on the surface. Given an SDF, the optimal step size at any point is exactly $d(p)$ — the ray cannot intersect the surface within that distance. This gives the sphere marching algorithm:

```glsl
// GLSL compute shader — sphere marching a procedural SDF scene
// No acceleration structure, no triangle geometry, no VK_KHR_ray_tracing_pipeline
float sdf_sphere(vec3 p, vec3 center, float r) { return length(p - center) - r; }
float sdf_box(vec3 p, vec3 b) {
    vec3 q = abs(p) - b;
    return length(max(q, 0.0)) + min(max(q.x, max(q.y, q.z)), 0.0);
}
// Smooth minimum — blends two SDFs into an organic union
float smin(float a, float b, float k) {
    float h = max(k - abs(a - b), 0.0) / k;
    return min(a, b) - h * h * k * 0.25;
}
float scene_sdf(vec3 p) {
    float sphere = sdf_sphere(p, vec3(0.0, 0.5, 0.0), 0.5);
    float box    = sdf_box(p - vec3(0.0, -0.1, 0.0), vec3(0.8, 0.1, 0.8));
    return smin(sphere, box, 0.3);  // organic blend
}

vec3 sdf_normal(vec3 p) {
    // Tetrahedron technique — 4 samples, no central difference needed
    const vec2 e = vec2(0.001, -0.001);
    return normalize(
        e.xyy * scene_sdf(p + e.xyy) + e.yyx * scene_sdf(p + e.yyx) +
        e.yxy * scene_sdf(p + e.yxy) + e.xxx * scene_sdf(p + e.xxx));
}

// Returns hit distance or -1
float sphere_march(vec3 ro, vec3 rd) {
    float t = 0.0;
    for (int i = 0; i < 128; i++) {
        float d = scene_sdf(ro + rd * t);
        if (d < 0.001) return t;   // hit
        if (t > 200.0) return -1.0; // miss
        t += d;                    // advance by safe step size
    }
    return -1.0;
}
```

This runs entirely inside a compute or fragment shader — no BVH, no acceleration structure, no `VK_KHR_ray_tracing_pipeline`. It is the technique behind the [Shadertoy](https://www.shadertoy.com/) ecosystem, where complex animated scenes with soft shadows, ambient occlusion, and subsurface scattering are produced in a single fragment shader.

**Soft shadows and AO via ray marching.** Because the SDF value at intermediate march steps encodes "how close to a surface am I?", both ambient occlusion and penumbra soft shadows can be estimated cheaply without spawning new rays:

```glsl
// Soft shadow: tracks minimum normalized distance to occluder along the shadow ray
float soft_shadow(vec3 ro, vec3 rd, float mint, float maxt, float k) {
    float res = 1.0;
    for (float t = mint; t < maxt; ) {
        float h = scene_sdf(ro + rd * t);
        if (h < 0.001) return 0.0;         // fully occluded
        res = min(res, k * h / t);          // penumbra factor
        t += h;
    }
    return res;
}

// Ambient occlusion: how quickly does the scene SDF decrease along the normal?
float calc_ao(vec3 p, vec3 n) {
    float occ = 0.0, sca = 1.0;
    for (int i = 0; i < 5; i++) {
        float h = 0.01 + 0.12 * float(i) / 4.0;
        occ += (h - scene_sdf(p + n * h)) * sca;
        sca *= 0.95;
    }
    return clamp(1.0 - 3.0 * occ, 0.0, 1.0);
}
```

**Where SDF ray marching is used in production:**

- **Shadertoy / demo scene** — procedural scenes with no geometry budget (demoscene 64 KB intros)
- **Font rendering** — multi-channel SDF (MSDF) fonts render at any scale without aliasing; used in game engines and Qt text rendering
- **Lumen (Unreal Engine 5)** — software ray marching through a sparse voxel distance field for diffuse GI on hardware that lacks RT units; hardware RT is used in parallel for reflections on capable GPUs
- **Volume rendering** — ray marching through a 3D density field (clouds, smoke, medical CT); the SDF is replaced by a volumetric density/emission sample
- **GPU particle lighting** — SDF shadow probes for local particle transparency

**Limitations vs hardware ray tracing.** Ray marching over procedural SDFs cannot handle arbitrary triangle meshes efficiently (converting a triangle mesh to a watertight SDF is expensive and lossy). Hardware ray tracing (`VK_KHR_ray_tracing_pipeline`) is designed specifically for triangle-mesh scenes where the BVH encodes explicit geometry. The two techniques are complementary: SDF marching for procedural volumes and global distance fields, hardware RT for scene geometry.

---

### Other Light Transport Algorithms

**Voxel cone tracing (VXGI).** The scene is voxelized into a sparse 3D texture (mipmapped). Indirect lighting is approximated by "cone tracing" — casting a cone (not a ray) through the voxel grid and integrating the weighted voxel radiance along the path. NVIDIA's VXGI technique (used in some UE4 versions) implements this. It provides cheap, if low-frequency, diffuse GI without hardware RT. Voxel resolution limits fidelity; dynamic scenes require per-frame revoxelization.

**Screen-space techniques.** A family of approximations that "trace rays" through the depth buffer rather than against scene geometry. No hardware RT is required; they run as post-process passes.

| Technique | Rays traced | Quality limitation |
|-----------|-------------|-------------------|
| SSAO (Screen-Space Ambient Occlusion) | Short hemisphere samples | Missing off-screen occluders |
| HBAO / GTAO (Horizon-Based AO) | Horizon-angle samples in screen-space | Still screen-space only |
| SSR (Screen-Space Reflections) | Ray march in depth buffer | Missing off-screen reflections, breaks at grazing angles |
| SSGI (Screen-Space GI) | Short rays through depth buffer | Low frequency, fails at disocclusions |

All screen-space techniques share the same fundamental limit: the depth buffer only encodes the closest visible surface. Geometry behind the camera, below the horizon, or in shadow has no depth entry to sample.

**Radiosity.** A classic algorithm (Cohen & Greenberg, 1985) that solves diffuse inter-reflection by discretising the scene into surface patches and computing form factors (how much light patch A receives from patch B). The form-factor matrix is solved iteratively. Results are view-independent, which allowed radiosity to be precomputed for static scenes in early game engines (Quake's lightmaps use a simplified radiosity-like approach). It handles only perfectly diffuse (Lambertian) surfaces and does not scale to dynamic geometry.

**Photon mapping (Jensen, 1996).** A two-pass algorithm: (1) emit photons from light sources, trace them through the scene (reflections and refractions), and store hit points in a photon map (a k-d tree); (2) at render time, gather nearby photons at each surface hit to estimate indirect radiance. Photon mapping handles caustics (focused light through glass) well — a case where path tracing converges very slowly. Used in offline renderers (LuxCoreRender, PBRT) but not in real-time pipelines.

**ReSTIR (Resampled Importance Sampling).** A technique introduced by Bitterli et al. (2020) and shipped in production on NVIDIA RTX 30 series. Instead of drawing new samples from the BRDF, ReSTIR maintains a reservoir of past samples per pixel, resamples from neighbours (spatial reuse) and from previous frames (temporal reuse), and biases the estimator toward high-contribution samples. The result is a much lower-variance estimate for direct and indirect illumination with the same ray budget. ReSTIR GI and ReSTIR DI are implemented in NVIDIA Falcor and are the basis for RTXGI 2.0. Vulkan RT is the underlying hardware layer.

**Hardware RT vs software RT: the Lumen example.** Unreal Engine 5's Lumen global illumination system illustrates the hybrid approach clearly:

- **Software RT** (compute shader, no hardware extension): Marches rays through a precomputed distance field (per-mesh SDF + global SDF). Covers the full scene at low resolution, handles dynamic objects. Runs on any Vulkan 1.0 GPU.
- **Hardware RT** (VK_KHR_ray_tracing_pipeline / ray query): Traces rays against the actual BVH for high-quality reflections and direct lighting. Only on RT-capable GPUs (RDNA2+, Xe-HPG+, Turing+).
- **Screen-space GI**: Short-range fill for the gaps between software and hardware RT passes.

All three run simultaneously on capable hardware, blended by distance and effect type. This layered architecture is increasingly common in production engines.

---

### Why Hardware Acceleration

Classic software ray tracers (POV-Ray, PBRT on CPU) require minutes to hours per frame because ray-scene intersection is $O(\log N)$ per ray via BVH traversal, but the constant factor is large and the workload is incoherent (each ray tests a different path through the BVH). GPUs accelerate this through two mechanisms:

1. **Fixed-function traversal units.** AMD's Ray Accelerators (RDNA2+), NVIDIA's RT Cores (Turing+), and Intel's Ray Tracing Units (Xe-HPG+) implement BVH traversal and ray-AABB / ray-triangle intersection in dedicated silicon. The BVH traversal stack is maintained in hardware, freeing shader registers. RT Cores on NVIDIA A100 perform ~80 billion rays/second — roughly 10× faster than shader-only traversal.

2. **Massive parallelism.** A 1920×1080 frame has ~2M pixels. At 1 ray per pixel per frame at 60 fps, the GPU must trace 120M rays/second minimum. A GPU with 5000 shader processors can issue many ray tests per cycle in parallel, amortizing the latency of memory fetches through warp switching.

Vulkan's `VK_KHR_ray_tracing_pipeline` and `VK_KHR_ray_query` expose both mechanisms without requiring driver-specific extensions. On AMD and Intel, Mesa's RADV and ANV drivers lower these extensions to the hardware's native ray traversal ISA.

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

**Slang equivalent** — `RaytracingAccelerationStructure` and `TraceRay()` are first-class language constructs; `RayDesc` and a typed payload struct replace implicit `rayPayloadEXT` location slots and raw integer flag literals.

```slang
// File: raygen.slang
// Slang improvements:
// - [shader("raygen")] replaces stage-file extension (.rgen) convention
// - RaytracingAccelerationStructure is a built-in type; no #extension directives needed
// - Payload is a typed struct passed as inout — no rayPayloadEXT location slot required

struct RayPayload {
    float3 hitColor;
};

[[vk::binding(0, 0)]] RaytracingAccelerationStructure topLevelAS;
[[vk::binding(1, 0)]] RWTexture2D<float4>             outputImage;

[shader("raygen")]
[require(GL_EXT_ray_tracing)]
void rayGenMain() {
    uint2  pixel = DispatchRaysIndex().xy;
    uint2  dims  = DispatchRaysDimensions().xy;
    float2 uv    = (float2(pixel) + 0.5f) / float2(dims);

    RayDesc ray;
    ray.Origin    = float3(uv * 2.0f - 1.0f, -1.0f);
    ray.Direction = float3(0.0f, 0.0f, 1.0f);
    ray.TMin      = 0.001f;
    ray.TMax      = 10000.0f;

    RayPayload payload = {};
    TraceRay(
        topLevelAS,
        RAY_FLAG_FORCE_OPAQUE, // typed flag constant, not raw integer 0x1
        0xFF,                  // cull mask
        0, 0, 0,               // sbt record offset, sbt record stride, miss index
        ray,
        payload
    );

    outputImage[pixel] = float4(payload.hitColor, 1.0f);
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

**Slang equivalent** — `[shader("closesthit")]` takes payload and hit attributes as explicit typed parameters, eliminating `rayPayloadInEXT`/`hitAttributeEXT` storage qualifiers and implicit location numbers.

```slang
// File: closesthit.slang
// Slang improvements:
// - Payload and attributes are typed function parameters, not global storage qualifiers
// - BuiltInTriangleIntersectionAttributes provides .barycentrics as a named field
// - No separate rayPayloadInEXT / hitAttributeEXT declarations required

struct RayPayload {
    float3 hitColor;
};

[shader("closesthit")]
[require(GL_EXT_ray_tracing)]
void closestHitMain(inout RayPayload payload,
                    in BuiltInTriangleIntersectionAttributes attr) {
    float2 b    = attr.barycentrics;
    float3 bary = float3(1.0f - b.x - b.y, b.x, b.y);
    payload.hitColor = bary; // visualise barycentric coordinates
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

**Slang equivalent** — `asfloat()`/`asint()` and vector `select()` replace GLSL's `intBitsToFloat()`/`floatBitsToInt()` and per-component ternary expansion, making the ULP-offset logic cleaner and avoiding accidental scalar promotion.

```slang
// File: closesthit_shadow.slang
// Slang improvements:
// - asfloat() / asint() are standard intrinsics that accept int3/float3 directly
// - select(cond, a, b) applies component-wise on vectors, no manual x/y/z expansion
// - [[vk::push_constant]] on a named struct avoids anonymous-struct parse ambiguity

struct RayPayload    { float3 hitColor; };
struct ShadowPayload { bool   shadowed; };
struct PushConstants { float3 lightDir; };

[[vk::binding(0, 0)]] RaytracingAccelerationStructure topLevelAS;
[[vk::push_constant]] PushConstants pc;

float3 offsetPositionAlongNormal(float3 p, float3 n) {
    const float origin      = 1.0f / 32.0f;
    const float float_scale = 1.0f / 65536.0f;
    const float int_scale   = 256.0f;

    int3 of_i = int3(int_scale * n);
    // Add ULP offset to the float bit-pattern — adapts automatically to float magnitude
    int3   ix  = asint(p) + select(p < 0.0f, -of_i, of_i);
    float3 p_i = asfloat(ix);

    return select(abs(p) < origin, p + float_scale * n, p_i);
}

[shader("closesthit")]
[require(GL_EXT_ray_tracing)]
void closestHitShadow(inout RayPayload payload,
                      in BuiltInTriangleIntersectionAttributes attr) {
    float3 P = WorldRayOrigin() + RayTCurrent() * WorldRayDirection();
    float3 N = /* interpolated geometric normal at hit */;

    RayDesc shadowRay;
    shadowRay.Origin    = offsetPositionAlongNormal(P, N);
    shadowRay.Direction = pc.lightDir;
    shadowRay.TMin      = 0.001f;
    shadowRay.TMax      = 1e6f;

    ShadowPayload sp = { false };
    TraceRay(topLevelAS, RAY_FLAG_FORCE_OPAQUE, 0xFF, 0, 0, 1, shadowRay, sp);
    payload.hitColor = sp.shadowed ? float3(0.0f) : float3(1.0f);
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

**Slang equivalent** — `RayQuery<Flags>` encodes ray flags as a compile-time generic parameter for type safety; `[require(GL_EXT_ray_query)]` on the entry point replaces `#extension GL_EXT_ray_query : require` and participates in Slang's module capability system.

```slang
// File: ao_fragment.slang
// Slang improvements:
// - RayQuery<RAY_FLAG_TERMINATE_ON_FIRST_HIT | RAY_FLAG_FORCE_OPAQUE> — flags are
//   a compile-time generic, not a runtime integer; mismatched flags are a type error
// - COMMITTED_NOTHING / COMMITTED_TRIANGLE_HIT are typed enum values, not integers
// - Helper function traceAO() callable from any stage once the capability is declared

[[vk::binding(0, 0)]] RaytracingAccelerationStructure topLevelAS;

float traceAO(float3 pos, float3 normal) {
    RayDesc ray;
    ray.Origin    = pos + normal * 0.001f;
    ray.Direction = normal;  // fire along geometric normal for hemisphere AO
    ray.TMin      = 0.001f;
    ray.TMax      = 1.0f;

    RayQuery<RAY_FLAG_TERMINATE_ON_FIRST_HIT | RAY_FLAG_FORCE_OPAQUE> q;
    q.TraceRayInline(topLevelAS, RAY_FLAG_NONE, 0xFF, ray);

    // FORCE_OPAQUE: all candidates auto-committed; loop body never executes.
    // TERMINATE_ON_FIRST_HIT: traversal stops after the first committed hit.
    while (q.Proceed()) {}

    return (q.CommittedStatus() == COMMITTED_NOTHING) ? 1.0f : 0.0f;
}

[shader("fragment")]
[require(GL_EXT_ray_query)]
float4 aoFragmentMain(float3 worldPos    : TEXCOORD0,
                      float3 worldNormal : TEXCOORD1) : SV_Target {
    float ao = traceAO(worldPos, normalize(worldNormal));
    return float4(float3(ao), 1.0f);
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
