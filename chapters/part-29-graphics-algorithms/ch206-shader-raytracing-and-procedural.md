# Chapter 206: Shader Algorithm Catalog — Ray Tracing and Procedural Content (Part XXIX)

*Part XXIX — Graphics Algorithms*

**Target audiences**: Graphics application developers implementing hardware-accelerated ray tracing, procedural world generation, or GPU-based environment simulation; systems developers understanding divergence and memory access costs of BVH traversal and path tracing.

This chapter is the third volume of the Shader Algorithm Catalog. It covers three complementary domains: Category VII (Ray Tracing) treats hardware ray tracing from shader patterns through complete path tracer pipelines, ReSTIR resampling, and denoising; Category VIII (Procedural Content and Geometry) covers noise functions, SDF sphere marching, isosurface extraction, subdivision, and mesh shader patterns; Category IX (Environment and Simulation) covers terrain, water, fluid simulation, fog, and participating media. Each entry follows the same structure as Chapter 204: what it computes, when to use it, named variants, costs, and a primary reference.

## Table of Contents

- **[VII. Ray Tracing](#vii-ray-tracing)**
  - [Ray Tracing Shader Patterns](#ray-tracing-shader-patterns)
  - [Complete Path Tracer Pipeline](#complete-path-tracer-pipeline)
  - [ReSTIR PT — Path Tracing Resampling](#restir-pt-path-tracing-resampling)
  - [RT Denoising — SVGF and NRD](#rt-denoising-svgf-and-nrd)
  - [Ray Differentials](#ray-differentials)
  - [Software BVH Traversal](#software-bvh-traversal)
- **[VIII. Procedural Content and Geometry](#viii-procedural-content-and-geometry)**
  - [Procedural Generation and Noise](#procedural-generation-and-noise)
  - [Signed Distance Functions and Sphere Marching](#signed-distance-functions-and-sphere-marching)
  - [3D SDF Voxelization from Mesh](#3d-sdf-voxelization-from-mesh)
  - [Isosurface Extraction — Marching Cubes and Surface Nets](#isosurface-extraction-marching-cubes-and-surface-nets)
  - [Catmull-Clark Subdivision on GPU](#catmull-clark-subdivision-on-gpu)
  - [Displacement Mapping via Compute](#displacement-mapping-via-compute)
  - [Phong Tessellation and Adaptive Tessellation](#phong-tessellation-and-adaptive-tessellation)
  - [Geometry and Mesh Shader Patterns](#geometry-and-mesh-shader-patterns)
- **[IX. Environment and Simulation](#ix-environment-and-simulation)**
  - [Terrain, Vegetation, and Foliage](#terrain-vegetation-and-foliage)
  - [Terrain Material Splatting](#terrain-material-splatting)
  - [Water, Ocean, and Fluid Surface](#water-ocean-and-fluid-surface)
  - [Fluid Simulation — SPH and Eulerian Grids](#fluid-simulation-sph-and-eulerian-grids)
  - [GPU Particle Systems](#gpu-particle-systems)
  - [Exponential and Height Fog](#exponential-and-height-fog)
  - [Volumetric Rendering and Participating Media](#volumetric-rendering-and-participating-media)
  - [Direct Volume Rendering](#direct-volume-rendering)


---

## VII. Ray Tracing

Ray tracing unifies several previously separate techniques — shadows, reflections, ambient occlusion, global illumination, and caustics — under a single BVH traversal abstraction. This category covers the Vulkan ray tracing pipeline structure (ray generation, closest-hit, any-hit, miss shaders), the complete path-tracer pattern, the ReSTIR resampling algorithm that makes many-light path tracing tractable in real time, and the denoising passes (SVGF, NRD) that make undersampled ray budgets visually acceptable. Ray differentials and software BVH traversal are included as the supporting mathematical and algorithmic infrastructure.

### Ray Tracing Shader Patterns

Vulkan ray tracing (`VK_KHR_ray_tracing_pipeline`) introduces five shader stages: **ray generation** (rgen), **intersection** (rint), **any-hit** (rahit), **closest-hit** (rchit), and **miss** (rmiss). These are not standalone programs — they communicate through the **Shader Binding Table (SBT)** and through payload structs passed by reference. The patterns below describe how to wire these stages together for the most common rendering tasks.

**Shader stage**: `vkCmdTraceRaysKHR` dispatches a 2D grid of **ray generation** invocations. Each call to `traceRayEXT()` invokes intersection, any-hit, closest-hit, or miss shaders. The entire pipeline is asynchronous with respect to the rasterization pipeline; results are written to storage images or SSBOs.

#### RT Shadow Ray
Traces a shadow ray from a G-Buffer world position toward a light, returning a binary occlusion value. The any-hit shader terminates immediately on any opaque intersection; alpha-tested geometry calls `ignoreIntersectionEXT()` probabilistically.

```glsl
// Ray generation shader — one shadow ray per pixel per light
layout(set=0, binding=0) uniform accelerationStructureEXT tlas;
layout(set=0, binding=1, rgba16f) uniform image2D shadow_map;
layout(set=0, binding=2) uniform sampler2D g_depth;

layout(location=0) rayPayloadEXT bool shadow_hit;

void main() {
    ivec2 px   = ivec2(gl_LaunchIDEXT.xy);
    vec3  P    = reconstruct_position(px);   // from depth
    vec3  L    = normalize(light_pos - P);
    float dist = length(light_pos - P) - 0.01;

    shadow_hit = true;  // assume hit; any-hit clears to false if transparent
    traceRayEXT(tlas,
        gl_RayFlagsTerminateOnFirstHitEXT | gl_RayFlagsSkipClosestHitShaderEXT,
        0xFF, 0, 0, 0,   // mask, SBT offsets
        P, 0.001,        // origin, tmin
        L, dist,         // direction, tmax
        0);              // payload location
    imageStore(shadow_map, px, vec4(shadow_hit ? 0.0 : 1.0));
}
```

**Use cases** — soft shadow rays (trace multiple jittered rays, average); contact shadows; area light penumbra.  
**Reference** — [Vulkan Ray Tracing Best Practices (NVIDIA)](https://developer.nvidia.com/blog/best-practices-using-nvidia-rtx-ray-tracing/)

#### RT Ambient Occlusion (RTAO)
Traces N hemisphere rays per pixel using a cosine-weighted sample distribution; counts the fraction that miss all geometry within a maximum distance. Replaces SSAO entirely for scene-correct occlusion without screen-space limitations.

**Use cases** — interior occlusion, contact darkening; often combined with denoising (SVGF or OIDN) to reduce ray count to 1–4 per pixel.  
**Limitations** — requires denoising; 1 ray/pixel at 1080p = 2M rays/frame, ~0.3–1ms on modern hardware.  
**Reference** — [NVIDIA RTXAO sample](https://github.com/NVIDIAGameWorks/RTXAO)

#### RT Reflection Ray
Traces a specular reflection ray from the G-Buffer surface position along the reflection vector, evaluating full material shading at the hit point via the closest-hit shader. Replaces SSR for off-screen and high-roughness reflections.

**Use cases** — glass, polished floors, water; hybrid: use SSR for on-screen reflections (free), RTRT for off-screen misses.  
**Key variants** — stochastic roughness: importance-sample the GGX NDF to generate a perturbed reflection ray; use a small sample count + temporal accumulation.  
**Reference** — [Stachowiak: Stochastic Screen-Space Reflections (SIGGRAPH 2015)](https://advances.realtimerendering.com/s2015/)

#### Any-Hit Alpha Masking
The any-hit shader samples the material's opacity texture at the ray hit UV and calls `ignoreIntersectionEXT()` if the texel is below the alpha threshold, making the ray pass through alpha-tested foliage transparently.

**Use cases** — ray traversal through foliage, fences, alpha-tested decals; prevents hard occlusion on alpha-tested geometry.  
**Reference** — [Vulkan Ray Tracing Specification — Any-Hit Shader](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkRayTracingShaderGroupCreateInfoKHR.html)

#### Hybrid Rasterization + RT
Rasterizes the primary visibility (G-Buffer), then traces rays from G-Buffer surface positions for secondary effects: RT shadows, RTAO, RT reflections, RT global illumination one bounce. Avoids the ray generation cost for primary visibility where rasterization is faster.

**Use cases** — the standard production hybrid pipeline in all AAA engines using ray tracing (Cyberpunk 2077, Control, Metro Exodus).  
**Reference** — [Moreau et al.: Combining Rasterization and Ray Tracing (SIGGRAPH 2020)](https://advances.realtimerendering.com/s2020/)

---

### Complete Path Tracer Pipeline

The ray tracing shader stages (§22) described individual shader types in isolation. This section shows how they wire together into a complete unidirectional Monte Carlo path tracer — the reference algorithm that all production hybrid and real-time RT pipelines approximate. Understanding the full pipeline contextualises why each denoising, importance sampling, and ReSTIR technique exists.

**Shader stage**: the pipeline is driven by a **ray generation shader** (`rgen`) dispatching one thread per pixel. It calls `traceRayEXT()` which invokes **closest-hit** (`rchit`), **any-hit** (`rahit`), and **miss** (`rmiss`) shaders. The path state is passed via a `rayPayloadEXT` struct.

#### Path Payload
The payload struct is the only communication channel between shader stages across a `traceRayEXT()` call. It must be small (register pressure), self-contained (the path state for one bounce), and include an RNG state for reproducible sampling.

```glsl
struct PathPayload {
    vec3  throughput;     // accumulated path weight (starts at vec3(1))
    vec3  radiance;       // accumulated emitted/direct light
    vec3  origin;         // next ray origin
    vec3  direction;      // next ray direction
    uint  rng_state;      // random number generator state (PCG hash)
    bool  terminated;     // set by any-hit or miss shaders to stop the path
};
layout(location=0) rayPayloadEXT PathPayload payload;
```

#### Ray Generation Shader (rgen)
Drives the path tracing loop: initialise the payload, trace the primary ray, then iterate for up to `MAX_BOUNCES` secondary bounces. Each bounce accumulates the `radiance` contribution from direct light evaluated at the hit and updates `throughput` by the BRDF weight.

```glsl
layout(set=0, binding=0) uniform accelerationStructureEXT tlas;
layout(set=0, binding=1, rgba32f) uniform image2D accumulation;   // running average
uniform uint  frame_index;
uniform int   max_bounces;    // typically 4–8

void main() {
    ivec2 px      = ivec2(gl_LaunchIDEXT.xy);
    vec2  jitter  = vec2(pcg_float(px, frame_index * 2u),
                         pcg_float(px, frame_index * 2u + 1u));
    vec2  uv      = (vec2(px) + jitter) / vec2(gl_LaunchSizeEXT.xy);
    vec4  clip    = vec4(uv * 2.0 - 1.0, 0.0, 1.0);
    vec4  world_h = inv_proj_view * clip;
    vec3  ray_dir = normalize(world_h.xyz / world_h.w - cam_pos);

    payload.throughput  = vec3(1.0);
    payload.radiance    = vec3(0.0);
    payload.origin      = cam_pos;
    payload.direction   = ray_dir;
    payload.rng_state   = pcg_init(px, frame_index);
    payload.terminated  = false;

    for (int bounce = 0; bounce < max_bounces && !payload.terminated; ++bounce) {
        traceRayEXT(tlas,
            gl_RayFlagsOpaqueEXT,
            0xFF,           // cull mask
            0, 1, 0,        // SBT offsets: hitgroup 0, stride 1, miss 0
            payload.origin,
            0.001,          // tmin (avoid self-intersection)
            payload.direction,
            1e9,            // tmax
            0);             // payload location
    }

    // Temporal accumulation: running mean
    vec3 prev = imageLoad(accumulation, px).rgb;
    float t   = 1.0 / float(frame_index + 1u);
    imageStore(accumulation, px, vec4(mix(prev, payload.radiance, t), 1.0));
}
```

#### Closest-Hit Shader (rchit)
Evaluates the material at the ray hit: loads geometry (position, normal, UV) via barycentric interpolation, samples material textures, computes direct lighting using MIS (§29), then samples the BRDF for the next bounce direction.

```glsl
layout(location=0) rayPayloadInEXT PathPayload payload;
hitAttributeEXT vec2 bary;   // barycentric coordinates from rasterizer

void main() {
    // 1. Reconstruct hit geometry (barycentric interpolation)
    vec3 P = interpolate_position(gl_PrimitiveID, bary);
    vec3 N = interpolate_normal(gl_PrimitiveID, bary);
    vec2 uv= interpolate_uv(gl_PrimitiveID, bary);

    // 2. Load material
    MaterialData mat = load_material(gl_InstanceCustomIndexEXT, uv);

    // 3. Emission: if the hit surface is emissive, add contribution
    payload.radiance += payload.throughput * mat.emission;

    // 4. Direct light via MIS (sample one light + one BRDF direction, combine)
    payload.radiance += payload.throughput *
        evaluate_direct_mis(P, N, mat, -payload.direction, payload.rng_state);

    // 5. Sample BRDF for next bounce
    float pdf;
    vec3  L = sample_BRDF(N, -payload.direction, mat.roughness,
                           mat.F0, payload.rng_state, pdf);
    if (pdf < 1e-6) { payload.terminated = true; return; }

    vec3 brdf = eval_BRDF(-payload.direction, L, N, mat);
    payload.throughput *= brdf * max(0.0, dot(N, L)) / pdf;

    // Russian roulette path termination
    float p_survive = min(1.0, max(payload.throughput.r,
                          max(payload.throughput.g, payload.throughput.b)));
    if (pcg_float_rng(payload.rng_state) > p_survive) {
        payload.terminated = true; return;
    }
    payload.throughput /= p_survive;

    // 6. Setup next ray
    payload.origin    = P + N * 0.001;
    payload.direction = L;
}
```

#### Miss Shader (rmiss)
Called when the ray escapes the scene. Samples the environment map for sky radiance. Sets `terminated = true` to stop the path.

```glsl
layout(location=0) rayPayloadInEXT PathPayload payload;
layout(set=0, binding=2) uniform samplerCube env_map;

void main() {
    payload.radiance  += payload.throughput * texture(env_map, payload.direction).rgb;
    payload.terminated = true;
}
```

#### Russian Roulette and Path Termination
Paths are terminated probabilistically when throughput falls below a threshold: survive with probability `p = luminance(throughput)`, boost throughput by `1/p` if it survives. This keeps the estimator unbiased while capping path length in low-contribution regions.

**Complete pipeline summary**:
1. `rgen` — one thread per pixel, iterates bounces
2. `traceRayEXT()` → `rchit` — material eval + MIS direct light + BRDF sample
3. `traceRayEXT()` (shadow) → `rmiss` or `rahit` — visibility test for direct light
4. `rmiss` — environment map sample, terminate path
5. Accumulation image — running mean over frames
6. TAA / SVGF denoiser — clean up the noisy per-frame accumulation

**Reference** — [Pharr, Jakob, Humphreys: PBRT-v4 (free online)](https://pbrt.org/); [NVIDIA Vulkan Ray Tracing Tutorial](https://nvpro-samples.github.io/vk_raytracing_tutorial_KHR/)

---

### ReSTIR PT — Path Tracing Resampling

ReSTIR DI (§29) resamples direct illumination from millions of candidate light samples. **ReSTIR PT** extends this to the **full path**: rather than resampling only the first-vertex light sample, it resamples entire path suffixes — the sequence of vertices from the first bounce to the light. Each pixel stores a **reservoir** holding a complete subpath (a sequence of scattering events), and neighbouring pixels and previous-frame pixels are used as additional sample donors through **spatiotemporal resampling**, dramatically reducing variance at low sample-per-pixel counts.

**Shader stage**: all passes run as **ray generation (rgen) shaders** or **compute shaders** (for non-RT resampling steps). Each pixel's reservoir is stored in a G-buffer–like storage image.

#### Path Reservoir Structure
A path reservoir holds a reconnection vertex (the point at which the path is "split" for reuse), the outgoing path suffix as an opaque token (compressed as MC seed or explicit vertices), and the reservoir weight.

```glsl
// ReSTIR PT reservoir — stored per pixel in a storage buffer
struct PathReservoir {
    vec3  reconnect_pos;      // world-space vertex where suffix begins
    vec3  reconnect_normal;
    vec3  path_contribution;  // L_i at the reconnection vertex (from cached path)
    float M;                  // number of candidates contributing to this reservoir
    float W;                  // unbiased contribution weight (= p_hat / p_proposal)
    uint  rng_seed;           // RNG seed to reproduce the full path if needed
};
```

#### Initial Candidate Generation (rgen)
For each pixel, trace a primary path (k bounces), storing the path's radiance contribution and selecting a reconnection vertex using weighted reservoir sampling (WRS) over the path vertices.

```glsl
// ReSTIR PT initial sampling — rgen shader (simplified)
layout(location=0) rayPayloadEXT PathPayload payload;

void main() {
    ivec2 px    = ivec2(gl_LaunchIDEXT.xy);
    uint  rng   = pcg_init(uvec2(px), frame_index);

    // Trace primary path (1 rgen + k rchit bounces via payload continuation)
    payload = init_payload(cam_ray_origin(px), cam_ray_dir(px), rng);
    trace_path(payload, MAX_BOUNCES);

    // Build reservoir from path vertices using WRS (§29 Reservoir struct)
    Reservoir res = reservoir_init();
    float accum_w = 0.0;
    for (int b = 0; b < payload.num_vertices; ++b) {
        float p_hat = luminance(payload.vertex_contrib[b]);  // target PDF: contribution
        float w     = p_hat / max(payload.vertex_pdf[b], 1e-6);
        if (reservoir_update(res, b, w, pcg_float(rng))) {
            // selected vertex b as reconnection point
        }
    }

    // Compute unbiased weight W = (1/p_hat(selected)) * (res.w_sum / res.M)
    float p_hat_sel = luminance(payload.vertex_contrib[res.selected]);
    res.W = (p_hat_sel > 0.0) ? res.w_sum / (res.M * p_hat_sel) : 0.0;

    path_reservoirs[px.y * resolution.x + px.x] = pack_reservoir(res, payload);
}
```

#### Spatiotemporal Resampling
For each pixel, gather neighbouring pixels' reservoirs and the previous-frame reservoir (reprojected via motion vectors). Merge them using WRS with the **MIS weights** to maintain unbiasedness. The merged reservoir has M candidates from all donors; its W is renormalised accordingly.

```glsl
// ReSTIR PT temporal reuse — merge current reservoir with previous frame's
PathReservoir curr = path_reservoirs[px.y * w + px.x];
ivec2 prev_px = reproject(px, motion_vectors);
PathReservoir prev = prev_path_reservoirs[prev_px.y * w + prev_px.x];

// Reconnection shift: check if previous path's reconnection vertex is visible from current pixel
bool shift_valid = test_reconnection_visibility(curr.primary_hit, prev.reconnect_pos, prev.reconnect_normal);

if (shift_valid) {
    // Compute Jacobian of the path shift (accounts for geometry change between pixels)
    float jacobian = reconnection_jacobian(curr.primary_hit, prev.reconnect_pos);
    float p_hat_prev = luminance(prev.path_contribution) * jacobian;
    float w_prev     = p_hat_prev * prev.W * float(prev.M);

    // Merge into current reservoir
    if (reservoir_update(curr, PREV_SOURCE, w_prev, pcg_float(rng))) {
        curr.reconnect_pos    = prev.reconnect_pos;
        curr.path_contribution = prev.path_contribution;
    }
    curr.M += prev.M;
    curr.W  = reservoir_w_sum(curr) / (float(curr.M) * luminance(curr.path_contribution) + 1e-6);
}
path_reservoirs[px.y * w + px.x] = curr;
```

**Unbiasedness**: the reconnection shift's Jacobian ensures the resampling is unbiased — the estimator converges to the correct path integral. Without the Jacobian, spatial reuse introduces bias (the so-called "shift mapping" requirement).

**Reference** — [Ouyang et al.: ReSTIR GI: Path Resampling for Real-Time Path Tracing (HPG 2021)](https://dl.acm.org/doi/10.1145/3478512.3488613); [Lin et al.: ReSTIR PT: Generalized Resampling of Paths in Path Tracing (SIGGRAPH 2022)](https://research.nvidia.com/publication/2022-07_restir-pt-generalized-resampling-paths-path-tracing)

---

### RT Denoising — SVGF and NRD

Ray tracing at 1–4 samples per pixel produces extremely noisy images. Denoising reconstructs a clean image by exploiting temporal coherence (the current frame is similar to the last) and spatial coherence (neighbouring pixels with similar depth/normal have similar radiance). Without denoising, every RT technique in §22 produces unusable output at real-time sample budgets.

**Shader stage**: all denoising runs in **compute shaders** operating on the noisy input and G-Buffer (depth, normal, motion vectors) render targets. The output overwrites or accumulates into the final HDR framebuffer.

#### SVGF — Spatiotemporal Variance-Guided Filtering
SVGF (Schied et al., 2017) is the standard academic and industry starting point for RT denoising. It has three stages:

1. **Temporal accumulation** — reproject the previous frame's filtered output using motion vectors; blend with current frame using an alpha of ~0.1 (10% current, 90% history). Track per-pixel sample count for variance estimation.

2. **Variance estimation** — compute the spatial variance of luminance in a 7×7 neighbourhood of the noisy input.

3. **À-Trous wavelet filter** — run 5 iterations of an edge-stopping spatial filter with exponentially increasing kernel step sizes (1, 2, 4, 8, 16 pixels). At each iteration, the bilateral weight combines luminance variance, normal similarity, and depth similarity — stopping the blur across geometric edges.

```glsl
// SVGF À-Trous filter pass — one of 5 iterations
layout(local_size_x=8, local_size_y=8) in;
layout(set=0, binding=0) uniform sampler2D input_color;   // noisy or previous iteration
layout(set=0, binding=1) uniform sampler2D g_normal;
layout(set=0, binding=2) uniform sampler2D g_depth;
layout(set=0, binding=3) uniform sampler2D g_variance;
layout(set=0, binding=4, rgba16f) uniform image2D output_color;

uniform int   step_size;   // 1, 2, 4, 8, 16 per iteration
uniform float sigma_l;     // luminance weight (typically 4.0)
uniform float sigma_n;     // normal weight (typically 128.0)
uniform float sigma_d;     // depth weight (typically 1.0)

// 3×3 À-Trous kernel weights
const float kernel[3] = float[](3.0/8.0, 1.0/4.0, 1.0/16.0);

void main() {
    ivec2 px    = ivec2(gl_GlobalInvocationID.xy);
    vec4  c_p   = texelFetch(input_color, px, 0);
    vec3  n_p   = texelFetch(g_normal, px, 0).xyz;
    float d_p   = texelFetch(g_depth, px, 0).r;
    float var_p = texelFetch(g_variance, px, 0).r;
    float lum_p = dot(c_p.rgb, vec3(0.2126, 0.7152, 0.0722));

    vec4  sum_c = vec4(0.0);
    float sum_w = 0.0;

    for (int dy = -1; dy <= 1; ++dy)
    for (int dx = -1; dx <= 1; ++dx) {
        ivec2 nb   = px + ivec2(dx, dy) * step_size;
        vec4  c_nb = texelFetch(input_color, nb, 0);
        vec3  n_nb = texelFetch(g_normal, nb, 0).xyz;
        float d_nb = texelFetch(g_depth, nb, 0).r;
        float lum_nb = dot(c_nb.rgb, vec3(0.2126, 0.7152, 0.0722));

        // Edge-stopping weights
        float w_l = exp(-abs(lum_p - lum_nb) / (sigma_l * sqrt(var_p) + 1e-4));
        float w_n = pow(max(0.0, dot(n_p, n_nb)), sigma_n);
        float w_d = exp(-abs(d_p - d_nb) / (sigma_d * abs(d_p) + 1e-4));

        float h   = kernel[abs(dx)] * kernel[abs(dy)];
        float w   = h * w_l * w_n * w_d;
        sum_c    += w * c_nb;
        sum_w    += w;
    }
    imageStore(output_color, px, sum_c / max(sum_w, 1e-4));
}
```

**Reference** — [Schied et al.: Spatiotemporal Variance-Guided Filtering (HPG 2017)](https://research.nvidia.com/publication/2017-07_spatiotemporal-variance-guided-filtering-real-time-reconstruction-path-traced)

#### A-SVGF — Adaptive SVGF
Replaces fixed temporal accumulation with per-pixel adaptive history length: pixels with high temporal variance (disocclusions, fast-moving objects) use shorter history; stable pixels accumulate longer. Reduces ghosting at the cost of more bookkeeping.

**Reference** — [Schied et al.: Gradient Estimation for Real-Time Adaptive Temporal Filtering (HPG 2018)](https://cg.ivd.kit.edu/atf.php)

#### NRD — NVIDIA Real-Time Denoisers
NVIDIA's open-source NRD library provides production-quality denoisers for specific signal types: `REBLUR` (recurrent blur, world-space signal) and `RELAX` (relaxed Lyman-alpha, designed for ReSTIR DI/GI noisy input). Both are multi-pass compute pipelines operating in a different signal domain than SVGF, with separate denoisers for diffuse, specular, and shadow signals.

**Reference** — [NVIDIA NRD GitHub](https://github.com/NVIDIAGameWorks/RayTracingDenoiser); [NRD Integration Guide](https://github.com/NVIDIAGameWorks/NRDSample)

---

### Ray Differentials

In a path tracer, each camera ray corresponds to a small frustum on the image sensor. When the ray hits a surface and samples a texture, the correct mip level is determined by the texture's screen-space footprint — analogous to `dFdx`/`dFdy` in rasterization but unavailable in ray tracing shaders. **Ray differentials** propagate a pair of auxiliary rays (offset by one pixel in X and Y) through the scene alongside the primary ray, tracking how the footprint evolves through reflection, refraction, and travel. The result is a texture LOD that is correct for reflections, refractions, and indirect paths — eliminating aliasing from undersampled textures.

**Shader stage**: ray differentials are maintained in the **path payload** (alongside throughput and direction) and updated in the **closest-hit shader** at each bounce.

#### Ray Differential Payload
Extend the path payload (§55) with the spatial and directional differentials of the ray footprint.

```glsl
struct RayDiff {
    vec3 dP_dx;   // derivative of ray origin w.r.t. pixel X
    vec3 dP_dy;   // derivative of ray origin w.r.t. pixel Y
    vec3 dD_dx;   // derivative of ray direction w.r.t. pixel X
    vec3 dD_dy;   // derivative of ray direction w.r.t. pixel Y
};

struct PathPayload {
    vec3     throughput;
    vec3     radiance;
    vec3     origin;
    vec3     direction;
    RayDiff  diff;       // ray differential for texture LOD
    uint     rng_state;
    bool     terminated;
};
```

#### Primary Ray Differential Initialisation
The primary ray's differential is set from the camera's pixel footprint: `dD/dx` is the change in ray direction when moving one pixel in X, derived from the inverse projection matrix.

```glsl
// rgen shader — compute primary ray differentials
vec3 ray_dir       = normalize(world_dir_from_pixel(px));
vec3 ray_dir_dx    = normalize(world_dir_from_pixel(px + ivec2(1, 0)));
vec3 ray_dir_dy    = normalize(world_dir_from_pixel(px + ivec2(0, 1)));

payload.diff.dP_dx = vec3(0.0);   // camera is a point; origin has no footprint
payload.diff.dP_dy = vec3(0.0);
payload.diff.dD_dx = ray_dir_dx - ray_dir;   // directional differential in X
payload.diff.dD_dy = ray_dir_dy - ray_dir;   // directional differential in Y
```

#### Differential Propagation at a Surface Hit
After the ray travels a distance `t` to the hit point, propagate the differential: the footprint grows as the ray travels (spreading). At the surface, project the differential onto the tangent plane to get the UV footprint.

```glsl
// In rchit shader — propagate ray differential to hit surface
float t = gl_HitTEXT;

// Propagate origin differential: new dP = old dP + t * old dD + dt * D
// dt = -(dP · N + t * dD · N) / (D · N)  [from the ray-plane intersection]
float denom  = dot(payload.direction, hit_normal);
float dt_dx  = -dot(payload.diff.dP_dx + t * payload.diff.dD_dx, hit_normal) / denom;
float dt_dy  = -dot(payload.diff.dP_dy + t * payload.diff.dD_dy, hit_normal) / denom;

vec3 new_dP_dx = payload.diff.dP_dx + t * payload.diff.dD_dx + dt_dx * payload.direction;
vec3 new_dP_dy = payload.diff.dP_dy + t * payload.diff.dD_dy + dt_dy * payload.direction;

// Project dP onto UV space using the surface tangent frame (T, B)
// duv/dx = (dP_dx · T / |T|², dP_dx · B / |B|²)
vec2 duv_dx = vec2(dot(new_dP_dx, hit_tangent)   / dot(hit_tangent,   hit_tangent),
                   dot(new_dP_dx, hit_bitangent)  / dot(hit_bitangent, hit_bitangent));
vec2 duv_dy = vec2(dot(new_dP_dy, hit_tangent)   / dot(hit_tangent,   hit_tangent),
                   dot(new_dP_dy, hit_bitangent)  / dot(hit_bitangent, hit_bitangent));

// Compute LOD from UV footprint (same formula as rasterizer's implicit LOD)
float lod = 0.5 * log2(max(dot(duv_dx, duv_dx), dot(duv_dy, duv_dy)) * tex_size * tex_size);
vec4 albedo = textureLod(material_tex, uv, lod);
```

#### Differential Update for Specular Reflection
For a mirror reflection, the differential is updated by reflecting `dD` about the surface normal and adding a curvature term from the normal differential `dN` (which depends on the surface second derivative). For diffuse bounces, differentials are typically reset — the bounce direction is random, so the footprint is determined by the new solid angle, not the previous path.

```glsl
// Specular reflection — reflect directional differential
vec3 reflect_diff(vec3 dD, vec3 N, vec3 dN) {
    // dR/dx = dD/dx - 2*(dD/dx · N + D · dN/dx) * N - 2*(D · N) * dN/dx
    float dDdotN = dot(dD, N);
    float DdotN  = dot(payload.direction, N);
    return dD - 2.0*(dDdotN * N + DdotN * dN);
}
payload.diff.dD_dx = reflect_diff(payload.diff.dD_dx, hit_normal, dN_dx);
payload.diff.dD_dy = reflect_diff(payload.diff.dD_dy, hit_normal, dN_dy);
payload.diff.dP_dx = new_dP_dx;
payload.diff.dP_dy = new_dP_dy;
```

**Use cases** — production path tracers (Cycles, Arnold, PBRT) all implement ray differentials or a cone approximation (ray cones); eliminates aliasing on glossy reflections of high-frequency textures; critical when the path tracer renders fine detail at low sample counts with denoising.  
**Reference** — [Igehy: Tracing Ray Differentials (SIGGRAPH 1999)](https://dl.acm.org/doi/10.1145/311535.311555); [Akenine-Möller et al.: Texture Level of Detail Strategies for Real-Time Ray Tracing (Ray Tracing Gems, 2019)](https://www.realtimerendering.com/raytracinggems/)

---

### Software BVH Traversal

Hardware ray tracing (§22, §55) requires RTX-class GPUs. **Software BVH traversal** implements the same ray-AABB and ray-triangle intersection in a compute shader, enabling ray casting on any GPU. This is used for: occlusion queries in path guidng, proxy shadow casting on mobile, BVH debug visualisation, and custom intersection types (hair curves, displacement) not supported by hardware RT.

**Shader stage**: all traversal runs in **compute**. The BVH (precomputed on CPU or GPU) is stored in a storage buffer.

#### BVH Node Layout
```glsl
// Compressed BVH2 node (32 bytes — fits one cache line)
struct BVHNode {
    vec3  aabb_min;
    int   left_child;   // > 0: inner node index; < 0: -(leaf_tri_offset + 1)
    vec3  aabb_max;
    int   prim_count;   // 0 = inner node; > 0 = leaf with this many triangles
};
layout(set=0, binding=0) readonly buffer BVHNodes { BVHNode nodes[]; };
layout(set=0, binding=1) readonly buffer Triangles { vec4  tri_data[]; };  // packed positions
```

#### Stackless BVH Traversal (Restart Trail)
GPU stacks are expensive (per-thread register pressure). A simple alternative is the **restart trail** method: traverse nodes sequentially from the root, re-descending from the lowest ancestor that still has unvisited children. For small trees (< 64 nodes) this is cheaper than allocating a per-thread stack.

```glsl
// Iterative BVH traversal with explicit stack (32-entry stack in shared memory)
bool bvh_any_hit(vec3 origin, vec3 dir, float t_max) {
    float inv_dir[3] = {1.0/dir.x, 1.0/dir.y, 1.0/dir.z};
    int   stack[32];
    int   stack_top = 0;
    stack[stack_top++] = 0;   // push root

    while (stack_top > 0) {
        int  node_idx = stack[--stack_top];
        BVHNode node  = nodes[node_idx];

        // Ray-AABB intersection (slab method)
        float t_min_x = (node.aabb_min.x - origin.x) * inv_dir[0];
        float t_max_x = (node.aabb_max.x - origin.x) * inv_dir[0];
        float t_min_y = (node.aabb_min.y - origin.y) * inv_dir[1];
        float t_max_y = (node.aabb_max.y - origin.y) * inv_dir[1];
        float t_min_z = (node.aabb_min.z - origin.z) * inv_dir[2];
        float t_max_z = (node.aabb_max.z - origin.z) * inv_dir[2];

        float t_enter = max(max(min(t_min_x, t_max_x), min(t_min_y, t_max_y)), min(t_min_z, t_max_z));
        float t_exit  = min(min(max(t_min_x, t_max_x), max(t_min_y, t_max_y)), max(t_min_z, t_max_z));

        if (t_enter > t_exit || t_exit < 0.0 || t_enter > t_max) continue;

        if (node.prim_count > 0) {
            // Leaf: test triangles
            int base = -node.left_child - 1;
            for (int t = 0; t < node.prim_count; ++t) {
                int ti = (base + t) * 3;
                vec3 v0 = tri_data[ti    ].xyz;
                vec3 v1 = tri_data[ti + 1].xyz;
                vec3 v2 = tri_data[ti + 2].xyz;
                // Möller–Trumbore intersection
                vec3  e1 = v1 - v0, e2 = v2 - v0;
                vec3  h  = cross(dir, e2);
                float a  = dot(e1, h);
                if (abs(a) < 1e-8) continue;
                float f  = 1.0 / a;
                vec3  s  = origin - v0;
                float u  = f * dot(s, h);
                if (u < 0.0 || u > 1.0) continue;
                vec3  q  = cross(s, e1);
                float v  = f * dot(dir, q);
                if (v < 0.0 || u + v > 1.0) continue;
                float t_hit = f * dot(e2, q);
                if (t_hit > 0.001 && t_hit < t_max) return true;  // any-hit
            }
        } else {
            // Inner: push children (push farther first for front-to-back order)
            stack[stack_top++] = node.left_child;
            stack[stack_top++] = node.left_child + 1;
        }
    }
    return false;
}
```

**Reference** — [Aila & Laine: Understanding the Efficiency of Ray Traversal on GPUs (HPG 2009)](https://research.nvidia.com/publication/understanding-efficiency-ray-traversal-gpus); [Möller & Trumbore: Fast, Minimum Storage Ray/Triangle Intersection (JGT 1997)](https://dl.acm.org/doi/10.1145/1198555.1198748)

---


---

## VIII. Procedural Content and Geometry

Procedural techniques generate geometry and surface detail algorithmically rather than from artist-authored data. Noise functions (value, gradient, simplex, FBM, domain warping) are the primitive from which terrains, clouds, and procedural textures are constructed; signed distance functions provide a unified representation for implicit surfaces, collision queries, and font rendering; isosurface extraction bridges between volumetric scalar fields and renderable triangle meshes; and tessellation and subdivision add geometric detail at render time without storing it on disk. Mesh shader patterns expose fine-grained GPU control over the vertex/primitive pipeline for procedural geometry generation.

### Procedural Generation and Noise

Noise functions are the foundation of procedural content generation: they produce spatially coherent, pseudo-random fields that can stand in for any natural phenomenon without storing a texture. The taxonomy runs from simple lattice-interpolation schemes (Value, Gradient/Perlin) through simplex and cellular variants to compositional techniques (FBM, domain warping) that layer primitives into complex organic results. The practical starting point for most work is Gradient noise + FBM; swap in Voronoi or blue noise when the application specifically needs cellular structure or low-discrepancy sampling. Hash functions belong here too — they underpin every noise implementation and should be chosen carefully to avoid aliasing.

**Shader stage**: All noise functions in this section are pure mathematical functions callable from any shader stage. In practice they appear most often in the **fragment shader** (procedural texture evaluation per-pixel) and **compute shader** (texture generation offline or per-frame). Vertex shader use is less common due to the lack of automatic partial derivatives for mip selection.

#### Value Noise
Interpolates random values at integer lattice points using smooth cubic or quintic blending. Produces smooth, low-frequency random fields with visible grid artifacts at high lacunarity.

**Use cases** — cheap cloud density fields, simple terrain height variation, prototype procedural textures.  
**Key variants** — quintic interpolation (Perlin's improved formula, 2002) eliminates second-derivative discontinuities visible as grid seams in normals.  
**Limitations** — directional grid bias; blocked or banded appearance when multiple octaves compound. Replaced by gradient noise in most production use.  
**Reference** — [Inigo Quilez: Value Noise](https://iquilezles.org/articles/valuenoise/)

#### Gradient (Perlin) Noise
Computes dot products of lattice-point random gradient vectors with offset vectors, then interpolates. Produces isotropic, band-limited noise with a characteristic organic appearance.

**Use cases** — terrain heightmaps, cloud/fog density, animated water ripples, procedural wood/marble texture.  
**Key variants** — Classic Perlin (1985); Improved Perlin (2002, quintic interpolation); Simplex Noise (2001, simplex lattice, fewer gradient evaluations).  
**Limitations** — axis-aligned directional bias in classic form; gradient artifacts visible at near-zero-frequency inputs.  
**Reference** — [Ken Perlin: Improved Noise](https://mrl.nyu.edu/~perlin/noise/) | [GPU Gems Ch5](https://developer.nvidia.com/gpugems/gpugems/part-i-natural-effects/chapter-5-implementing-improved-perlin-noise)

#### Simplex Noise
Evaluates noise on a simplex lattice (triangles in 2D, tetrahedra in 3D) with fewer lattice point evaluations than Perlin noise and no axis-aligned artifacts.

**Use cases** — same as gradient noise; preferred when performance or isotropy is critical; GPU-friendly implementation in GLSL.  
**Key variants** — OpenSimplex2 (2021) removes the patent issues of the original Simplex formulation and produces higher quality output.  
**Limitations** — slightly more complex to implement; patent on the original formulation expired 2022 but caused ecosystem fragmentation.  
**Reference** — [Stefan Gustavson: Simplex Noise Demystified](https://weber.itn.liu.se/~stegu/simplexnoise/simplexnoise.pdf) | [OpenSimplex2](https://github.com/KdotJPG/OpenSimplex2)

#### Fractal Brownian Motion (FBM)
Sums multiple octaves of a noise basis function with geometrically decreasing amplitude (persistence) and increasing frequency (lacunarity) to produce fractal self-similar fields.

**Use cases** — terrain, clouds, fire, turbulence, procedural planet surfaces — any natural phenomenon with fractal self-similarity.  
**Key variants** — Ridged FBM (absolute-value noise for mountain ridges); Billowy (1−abs, for cumulus clouds); Warped FBM (domain warping).  
**Limitations** — cost scales linearly with octave count; naive FBM has uniform energy across all directions (not always perceptually correct).  
**Reference** — [Inigo Quilez: FBM](https://iquilezles.org/articles/fbm/)

#### Domain Warping
Distorts the input coordinates of a noise function using another noise evaluation before sampling, producing twisted, turbulent patterns that cannot be achieved by layering alone.

**Use cases** — cloud interiors, marble veining, alien landscapes, fluid-like turbulence without simulation.  
**Key variants** — single warp (fbm(p + fbm(p))); double warp (fbm(p + fbm(p + fbm(p)))); warp by gradient field.  
**Limitations** — multiplicative cost (2–3× more noise evaluations); result is hard to art-direct precisely; does not correspond to any physical process.  
**Reference** — [Inigo Quilez: Domain Warping](https://iquilezles.org/articles/warp/)

#### Voronoi / Worley Cellular Noise
Computes the distance from each point to its nearest (F1) or second-nearest (F2) seed point in a random cellular partition, producing cracked, crystalline, or cellular patterns.

**Use cases** — cracked mud, stone, cellular tissue, liquid metal, leather, scale textures.  
**Key variants** — F1 (nearest distance); F2 (second-nearest); F2−F1 (sharp cell borders); Manhattan and Chebyshev distance metrics for different cell shapes.  
**Limitations** — naive implementation is O(n) in seed count; grid-based acceleration limits to O(1) but requires careful seeding to avoid empty cells.  
**Reference** — [Inigo Quilez: Voronoi](https://iquilezles.org/articles/voronoilines/)

#### Voronoise
Unifies gradient noise and Voronoi noise via a single blending parameter, smoothly interpolating between Perlin-like and cellular appearance.

**Use cases** — when a noise type must be artist-tuned between smooth and cellular without code changes; useful in material graph systems.  
**Limitations** — less efficient than either pure form; the blend parameter has no physical interpretation.  
**Reference** — [Inigo Quilez: Voronoise](https://iquilezles.org/articles/voronoise/)

#### Blue Noise
A random distribution with suppressed low-frequency energy, producing visually uniform scatter without the clumping of white noise.

**Use cases** — dithering for HDR-to-SDR quantisation; sample point distributions for SSAO, AO ray casting, TAA jitter; shadow map sample patterns.  
**Key variants** — pre-baked void-and-cluster textures (Ulichney); runtime generation via Mitchell's best-candidate; free blue-noise texture atlases (Christoph Peters).  
**Limitations** — generating high-quality blue noise at runtime is expensive; pre-baked textures must tile without breaking the spectrum.  
**Reference** — [Christoph Peters: Free Blue Noise Textures](https://momentsingraphics.de/BlueNoise.html)

#### Hash Functions for Shaders
Integer-to-float hash functions that replace `fract(sin(dot()))` (which aliases at high frequency) with quality PRNGs suitable for shader use.

**Use cases** — procedural texture seeds, particle instance variation, any per-pixel random number need.  
**Key variants** — PCG (Permuted Congruential Generator); xxhash; Murmur-inspired; Mark Jarzynski & Marc Olano hash.  
**Limitations** — `fract(sin(dot()))` is widely seen in tutorials but has visible periodicity; replace with a proper hash for production.  
**Reference** — [Jarzynski & Olano: Hash Functions for GPU Rendering](https://jcgt.org/published/0009/03/02/)

---

### Signed Distance Functions and Sphere Marching

A signed distance function (SDF) returns the distance from any point in space to the nearest surface, with negative values inside and positive outside. This representation makes it trivial to march a ray through a scene without missing surfaces (sphere marching), to combine geometry with Boolean operations, and to compute accurate normals and soft shadows analytically — all without a triangle mesh. The trade-off is that complex scenes require many SDF evaluations per ray, making performance highly dependent on step count and SDF complexity. SDFs are the native language of Shadertoy-style GPU raytracing and are increasingly used in game engines for collision, font rendering, and procedural modelling.

**Shader stage**: Entirely **fragment shader** (or **compute shader** for offline baking). The sphere marching loop, SDF evaluation, normal estimation, and lighting all run per-pixel in the fragment stage — there is no geometry; the "scene" is described purely as a mathematical function.

#### Sphere Marching Loop
Iteratively advances a ray by the value of a signed distance function at the current position, guaranteeing no surface intersection is missed while taking the largest safe step.

**Use cases** — rendering implicit surfaces, CSG models, procedural geometry defined analytically rather than by triangle meshes; Shadertoy-style GPU raytracing.  
**Key variants** — relaxed sphere marching (overstepping with backtrack); enhanced sphere tracing (Keinert 2014).  
**Limitations** — step count (typically 64–128) dominates cost; non-Lipschitz SDFs (e.g. incorrect smooth-min) cause infinite loops or missed surfaces.  
**Reference** — [Inigo Quilez: Raymarching SDFs](https://iquilezles.org/articles/raymarchingdf/)

#### SDF Primitive Library
Closed-form signed distance functions for 40+ geometric primitives including sphere, box, rounded box, cylinder, capsule, torus, cone, hexagonal prism, triangular prism, link, ellipsoid, and octahedron.

**Use cases** — CSG modelling in shaders; collision detection; font rendering; any application needing exact analytical geometry without a mesh.  
**Key variants** — exact SDFs (correct sign and magnitude everywhere) vs bound SDFs (conservative overestimate, cheaper but miss-stepable).  
**Limitations** — complex shapes require compositing many primitives; no efficient SDF for arbitrary meshes (requires a baked distance field texture).  
**Reference** — [Inigo Quilez: 3D SDF Primitives](https://iquilezles.org/articles/distfunctions/)

#### SDF Boolean Operations
Combines multiple SDFs via union (min), subtraction (max of negated), and intersection (max) to perform constructive solid geometry in shader code.

**Use cases** — procedural modelling: drilling holes, combining objects, clipping regions; real-time CSG for tools or game mechanics.  
**Key variants** — smooth union/subtraction/intersection using `smin` (polynomial or exponential smooth-min); produces rounded blending between surfaces.  
**Limitations** — smooth operations are not true CSG (gradient is not conservative at the blend boundary); deep nesting of operations degrades step efficiency.  
**Reference** — [Inigo Quilez: SDF Operations](https://iquilezles.org/articles/distfunctions/)

#### Normal Estimation from SDF Gradient
Approximates the surface normal at a raymarched hit point by finite-differencing the SDF in six (or four) nearby positions.

**Use cases** — required for all lighting calculations in SDF raymarching; no vertex normals exist.  
**Key variants** — central differences (6 samples, more accurate); tetrahedron method (4 samples, fewer evaluations, slight asymmetry).  
**Limitations** — each normal estimate costs 4–6 additional SDF evaluations; noisy SDFs produce noisy normals.  
**Reference** — [Inigo Quilez: SDF Normals](https://iquilezles.org/articles/normalsSDF/)

#### Soft Shadows from SDF
Estimates shadow penumbra width by tracking the minimum clearance of the shadow ray to any surface during sphere marching, producing contact-hardened soft shadows without area light sampling.

**Use cases** — ambient scenes rendered entirely in SDF; stylised soft shadows without shadow maps.  
**Key variants** — original Quilez formula; improved version (2023) with correct geometric derivation and less over-darkening.  
**Limitations** — self-shadowing requires a surface-offset epsilon; shadow quality depends on step count; not physically correct for large area lights.  
**Reference** — [Inigo Quilez: Soft Shadows](https://iquilezles.org/articles/rmshadows/)

#### Ambient Occlusion from SDF
Estimates local occlusion by sampling the SDF along the surface normal at several distances and weighting closer misses more heavily.

**Use cases** — essentially free AO for SDF scenes; approximates sky visibility without ray casting.  
**Limitations** — samples only the SDF along one direction; misses occlusion from geometry on the opposite side of the surface.  
**Reference** — [Inigo Quilez: AO](https://iquilezles.org/articles/ao/)

#### SDF Extrusion and Revolution
Constructs 3D SDFs from 2D SDFs by extruding along an axis or revolving around an axis, enabling complex solids from simple 2D profiles.

**Use cases** — architectural mouldings, lathe-turned objects, thread profiles, any rotationally symmetric or prismatic shape.  
**Limitations** — extrusion along non-axis-aligned paths requires expensive arc-length parameterisation; only exact for Lipschitz 2D SDFs.  
**Reference** — [Inigo Quilez: 2D SDFs](https://iquilezles.org/articles/distfunctions2d/)

---

### 3D SDF Voxelization from Mesh

The SDF sphere-marching section (§2) renders an *existing* signed distance field. Generating a 3D SDF from a triangle mesh requires a different pipeline: for each voxel, compute the signed distance to the nearest triangle. This is used to initialise VXGI volumes (§6), build collision distance fields (for soft-body and cloth, §37), and provide the initial SDF for dynamic mesh deformation.

**Shader stage**: voxelization runs in a **compute shader** dispatched over the 3D voxel grid. A BVH (§42 LBVH) accelerates the nearest-triangle query; a brute-force scan over all triangles is feasible for small meshes (< 10k triangles) or coarse volumes.

#### Triangle-Distance Voxelization (Brute Force)
For each voxel centre, find the closest triangle and compute the signed distance. Sign is determined by the dot product of the triangle normal with the voxel-to-closest-point vector (positive = outside, negative = inside).

```glsl
layout(local_size_x=4, local_size_y=4, local_size_z=4) in;
layout(set=0, binding=0, r32f) uniform image3D sdf_volume;
layout(set=0, binding=1) readonly buffer Vertices  { vec4 verts[]; };
layout(set=0, binding=2) readonly buffer Indices   { uint idxs[]; };
layout(set=0, binding=3) readonly buffer Normals   { vec4 norms[]; };  // per-triangle

uniform vec3  vol_min;    // world-space AABB of volume
uniform vec3  vol_max;
uniform ivec3 vol_res;    // voxel grid resolution
uniform uint  tri_count;

// Closest point on triangle (p0, p1, p2) to point p
vec3 closest_point_on_triangle(vec3 p, vec3 p0, vec3 p1, vec3 p2) {
    vec3 ab = p1 - p0, ac = p2 - p0, ap = p - p0;
    float d1 = dot(ab,ap), d2 = dot(ac,ap);
    if (d1 <= 0.0 && d2 <= 0.0) return p0;
    vec3 bp = p - p1;
    float d3 = dot(ab,bp), d4 = dot(ac,bp);
    if (d3 >= 0.0 && d4 <= d3) return p1;
    vec3 cp = p - p2;
    float d5 = dot(ab,cp), d6 = dot(ac,cp);
    if (d6 >= 0.0 && d5 <= d6) return p2;
    float vc = d1*d4 - d3*d2;
    if (vc <= 0.0 && d1 >= 0.0 && d3 <= 0.0) {
        return p0 + (d1 / (d1 - d3)) * ab;
    }
    float vb = d5*d2 - d1*d6;
    if (vb <= 0.0 && d2 >= 0.0 && d6 <= 0.0) {
        return p0 + (d2 / (d2 - d6)) * ac;
    }
    float va = d3*d6 - d5*d4;
    float w  = d4 - d3, x = d5 - d6;
    if (va <= 0.0 && w >= 0.0 && x >= 0.0) {
        return p1 + (w / (w + x)) * (p2 - p1);
    }
    float denom = 1.0 / (va + vb + vc);
    return p0 + (vb*denom)*ab + (vc*denom)*ac;
}

void main() {
    ivec3 vox  = ivec3(gl_GlobalInvocationID);
    if (any(greaterThanEqual(vox, vol_res))) return;
    vec3  p    = vol_min + (vec3(vox) + 0.5) / vec3(vol_res) * (vol_max - vol_min);

    float min_dist  = 1e9;
    float sign_val  = 1.0;

    for (uint t = 0; t < tri_count; ++t) {
        uint i0 = idxs[t*3+0], i1 = idxs[t*3+1], i2 = idxs[t*3+2];
        vec3 p0 = verts[i0].xyz, p1 = verts[i1].xyz, p2 = verts[i2].xyz;
        vec3 cp = closest_point_on_triangle(p, p0, p1, p2);
        float d = length(p - cp);
        if (d < min_dist) {
            min_dist = d;
            // Sign from triangle normal: negative = inside mesh
            vec3 tri_normal = norms[t].xyz;
            sign_val = (dot(p - cp, tri_normal) >= 0.0) ? 1.0 : -1.0;
        }
    }
    imageStore(sdf_volume, vox, vec4(sign_val * min_dist));
}
```

**Optimisation**: the brute-force O(voxels × triangles) complexity is only feasible for coarse volumes. For production use, build an LBVH (§42) over triangles and traverse it per voxel, reducing the complexity to O(voxels × log(triangles)).

**Sign robustness**: the per-triangle sign test is not robust at surface boundaries. More robust alternatives: (1) winding number (Jacobson 2013, exact but expensive); (2) ray casting — count intersections along multiple axis-aligned rays and take the majority vote.

**Use cases** — VXGI voxel grid initialisation, rigid-body SDF collision fields, cloth-obstacle distance fields for PBD constraints (§37), starting volume for GPU sculpting.  
**Reference** — [Bærentzen & Aanæs: Generating Signed Distance Fields From Triangle Meshes (DTU 2002)](https://orbit.dtu.dk/files/146201972/Signed_Distance_Fields.pdf); [Jacobson et al.: Robust Inside-Outside Segmentation Using Generalized Winding Numbers (SIGGRAPH 2013)](https://dl.acm.org/doi/10.1145/2461912.2461916)

---

### Isosurface Extraction — Marching Cubes and Surface Nets

Isosurface extraction converts a scalar field stored in a 3D grid (a signed distance field, a density volume, a voxel occupancy grid) into a triangle mesh. The scalar field exists on GPU-side SSBOs or 3D textures; the extraction runs entirely in **compute shaders**, producing a vertex and index buffer ready for standard rendering without CPU readback.

**Use cases** — procedural voxel terrain (Minecraft-style and smooth), GPU fluid surface meshes, SDF-to-mesh export, medical volume rendering, Dual Contouring for sharp features.

#### Marching Cubes
Each voxel cell is a cube whose 8 corners have scalar values. If a corner is inside the isosurface (value < iso_level) it is "active". The 8 corner states produce a `uint8` case index (0–255) indexing into a precomputed lookup table of up to 5 triangles per cube.

```glsl
// Marching Cubes compute shader — one workgroup per voxel cell
layout(local_size_x=4, local_size_y=4, local_size_z=4) in;

layout(set=0, binding=0) uniform sampler3D density_vol;       // scalar field
layout(set=0, binding=1) writeonly buffer VertexBuf { vec4 verts[]; };
layout(set=0, binding=2) writeonly buffer IndexBuf  { uint  idxs[]; };
layout(set=0, binding=3) buffer Counters { uint vert_count; uint idx_count; };

// Precomputed tables (uploaded as SSBOs at init):
layout(set=0, binding=4) readonly buffer EdgeTable { int edge_table[256]; };
layout(set=0, binding=5) readonly buffer TriTable  { int tri_table[256][16]; };

uniform float iso_level;
uniform vec3  cell_size;

vec3 interp_vertex(vec3 p0, float v0, vec3 p1, float v1) {
    float t = (iso_level - v0) / (v1 - v0);
    return mix(p0, p1, clamp(t, 0.0, 1.0));
}

void main() {
    ivec3 cell = ivec3(gl_GlobalInvocationID);
    // Sample 8 corners
    float val[8];
    vec3  pos[8];
    for (int i = 0; i < 8; ++i) {
        ivec3 c = cell + ivec3((i)&1, (i>>1)&1, (i>>2)&1);
        pos[i]  = vec3(c) * cell_size;
        val[i]  = texelFetch(density_vol, c, 0).r;
    }
    // Build case index
    uint cube_idx = 0u;
    for (int i = 0; i < 8; ++i)
        if (val[i] < iso_level) cube_idx |= (1u << i);
    if (cube_idx == 0u || cube_idx == 255u) return;  // entirely inside or outside

    // Compute edge intersections
    vec3 edge_verts[12];
    int emask = edge_table[cube_idx];
    // [12 edge cases abbreviated — each edge connects two corners; interp if bit set]
    if ((emask & 1)   != 0) edge_verts[0]  = interp_vertex(pos[0],val[0],pos[1],val[1]);
    if ((emask & 2)   != 0) edge_verts[1]  = interp_vertex(pos[1],val[1],pos[2],val[2]);
    if ((emask & 4)   != 0) edge_verts[2]  = interp_vertex(pos[2],val[2],pos[3],val[3]);
    if ((emask & 8)   != 0) edge_verts[3]  = interp_vertex(pos[3],val[3],pos[0],val[0]);
    if ((emask & 16)  != 0) edge_verts[4]  = interp_vertex(pos[4],val[4],pos[5],val[5]);
    if ((emask & 32)  != 0) edge_verts[5]  = interp_vertex(pos[5],val[5],pos[6],val[6]);
    if ((emask & 64)  != 0) edge_verts[6]  = interp_vertex(pos[6],val[6],pos[7],val[7]);
    if ((emask & 128) != 0) edge_verts[7]  = interp_vertex(pos[7],val[7],pos[4],val[4]);
    if ((emask & 256) != 0) edge_verts[8]  = interp_vertex(pos[0],val[0],pos[4],val[4]);
    if ((emask & 512) != 0) edge_verts[9]  = interp_vertex(pos[1],val[1],pos[5],val[5]);
    if ((emask & 1024)!= 0) edge_verts[10] = interp_vertex(pos[2],val[2],pos[6],val[6]);
    if ((emask & 2048)!= 0) edge_verts[11] = interp_vertex(pos[3],val[3],pos[7],val[7]);

    // Emit triangles via atomic counter
    for (int t = 0; tri_table[cube_idx][t] != -1; t += 3) {
        uint base = atomicAdd(vert_count, 3u);
        verts[base + 0] = vec4(edge_verts[tri_table[cube_idx][t+0]], 1.0);
        verts[base + 1] = vec4(edge_verts[tri_table[cube_idx][t+1]], 1.0);
        verts[base + 2] = vec4(edge_verts[tri_table[cube_idx][t+2]], 1.0);
    }
}
```

**Limitations** — produces vertices independently per cell (no shared vertices); a welding pass or a surface-net approach is needed for a proper indexed mesh. Normals can be computed from the SDF gradient (`normalize(grad(density))`).  
**Reference** — [Lorensen & Cline: Marching Cubes (SIGGRAPH 1987)](https://dl.acm.org/doi/10.1145/37402.37422); [Paul Bourke's Marching Cubes lookup tables](http://paulbourke.net/geometry/polygonise/)

#### Surface Nets (Naive Surface Nets)
An alternative to Marching Cubes that places a single vertex per active cell (one whose corners straddle the isosurface) at the average of all edge-crossing positions, then connects adjacent active cells with quads. Produces a coarser but topologically simpler mesh with shared vertices and no lookup tables.

**Use cases** — smooth voxel terrain where the lookup table complexity of MC is undesirable; the basis for Dual Contouring (which refines vertex placement using SDF gradients to preserve sharp features).  
**Reference** — [Gibson: Constrained Elastic Surface Nets (MICCAI 1998)](https://dl.acm.org/doi/10.5555/646290.687765); [0fps: Meshing in a Minecraft Game (2012)](https://0fps.net/2012/07/12/smooth-voxel-terrain-part-2/)

---

### Catmull-Clark Subdivision on GPU

Catmull-Clark subdivision (CCS) converts a coarse control-mesh into a smooth limit surface by iterative refinement: each subdivision step inserts a face point at each polygon centroid, an edge point at each edge midpoint, and updates each vertex using its valence-dependent stencil. The key GPU challenge is **extraordinary vertices** (valence ≠ 4) which require special evaluation stencils and cannot be processed by the uniform patch pipeline. Three GPU approaches exist: **full refinement** (compute all levels), **feature-adaptive subdivision** (ACC, Patney 2009 — refine only near extraordinary vertices), and **tessellation-shader-based** evaluation (GLSL/HLSL hull+domain shaders with precomputed ACC patches).

**Shader stage**: the compact on-GPU approach uses **tessellation control and evaluation shaders** (TCS/TES) to evaluate B-spline patches (regular regions) and precomputed ACC limit patches (extraordinary regions).

#### Regular B-Spline Patch Evaluation (Valence-4 Regions)
Regular regions of a CCS surface converge to a bicubic B-spline at the limit. The TES evaluates this directly from the 16-point control cage patch.

```glsl
// TES — bicubic B-spline patch evaluation (regular Catmull-Clark region)
layout(quads, fractional_odd_spacing, ccw) in;

layout(location=0) in  vec3 tc_pos[];   // 16 control points from TCS
layout(location=0) out vec3 te_pos;
layout(location=1) out vec3 te_normal;

// Bicubic B-spline basis functions
vec4 cubic_bspline_basis(float t) {
    float t2 = t * t, t3 = t2 * t;
    return vec4(
        (-t3 + 3.0*t2 - 3.0*t + 1.0) / 6.0,
        ( 3.0*t3 - 6.0*t2 + 4.0) / 6.0,
        (-3.0*t3 + 3.0*t2 + 3.0*t + 1.0) / 6.0,
        t3 / 6.0
    );
}

void main() {
    float u = gl_TessCoord.x, v = gl_TessCoord.y;
    vec4  Bu = cubic_bspline_basis(u);
    vec4  Bv = cubic_bspline_basis(v);

    // Evaluate position from 4×4 control points
    vec3 pos = vec3(0.0);
    for (int j = 0; j < 4; ++j)
        for (int i = 0; i < 4; ++i)
            pos += Bu[i] * Bv[j] * tc_pos[j*4 + i];
    te_pos = pos;

    // Evaluate tangents (derivative of B-spline basis)
    vec4 dBu = vec4(-Bu.x + Bu.y,   // dB/dt
                    -3.0*Bu.x/2.0 + Bu.z/2.0,
                    3.0*Bu.x/2.0 - Bu.z/2.0,
                    Bu.x - Bu.y);
    vec3 dPdu = vec3(0.0), dPdv = vec3(0.0);
    for (int j = 0; j < 4; ++j)
        for (int i = 0; i < 4; ++i) {
            dPdu += dBu[i] * Bv[j]  * tc_pos[j*4 + i];
            dPdv += Bu[i]  * dBu[j] * tc_pos[j*4 + i];  // same basis, swap axes
        }
    te_normal = normalize(cross(dPdu, dPdv));
    gl_Position = mvp * vec4(pos, 1.0);
}
```

#### Extraordinary Vertex Handling (ACC)
For extraordinary vertices (valence ≠ 4), precompute a set of 32-point **ACC (Approximation of Catmull-Clark) patches** offline, store them in a per-patch buffer, and evaluate them in the TES using the ACC evaluation formulae (Schäfer & Warren 2007). This avoids the combinatorial complexity of explicit iterative refinement.

**Reference** — [DeRose, Kass & Truong: Subdivision Surfaces in Character Animation (SIGGRAPH 1998)](https://dl.acm.org/doi/10.1145/280814.280836); [Schäfer & Warren: Exact Evaluation of Catmull-Clark Subdivision Surfaces at Arbitrary Parameter Values (CGF 2007)](https://dl.acm.org/doi/10.1145/1073204.1073333); [OpenSubdiv (Pixar)](https://graphics.pixar.com/opensubdiv/docs/intro.html)

---

### Displacement Mapping via Compute

Tessellation-based displacement (§51) displaces mesh vertices in the TES and is limited by the tessellation rate and the fixed function pipeline. **Compute-based displacement** instead runs a pre-pass that reads the displacement map and writes a fully displaced vertex buffer, giving greater flexibility: the displaced mesh can be used for both rasterisation and as a BVH input for ray tracing, and any compute operation (procedural noise, physics-driven, cache coherent) can drive the displacement.

**Shader stage**: displacement pre-pass runs in **compute**; the displaced vertex buffer is then consumed by the standard **vertex shader** draw.

#### Compute Displacement Pre-Pass
Dispatch over the tessellated mesh's vertex count. Each thread reads the vertex position, samples the displacement map, offsets along the normal, and writes to the output vertex buffer.

```glsl
// Compute displacement — one thread per vertex
layout(local_size_x=64) in;

struct Vertex {
    vec3 position;
    vec3 normal;
    vec2 uv;
    vec4 tangent;
};

layout(set=0, binding=0) readonly  buffer SrcVertices { Vertex src[]; };
layout(set=0, binding=1) writeonly buffer DstVertices { Vertex dst[]; };
layout(set=0, binding=2) uniform sampler2D displacement_map;
layout(set=0, binding=3) uniform sampler2D normal_map;   // for re-normals after displace

uniform float displace_scale;   // world-space displacement strength (e.g. 0.1 m)
uniform uint  num_vertices;

void main() {
    uint idx = gl_GlobalInvocationID.x;
    if (idx >= num_vertices) return;

    Vertex v = src[idx];

    // Sample displacement and displace along normal
    float height   = texture(displacement_map, v.uv).r;
    v.position    += v.normal * (height * displace_scale);

    // Optionally update normal from displacement map gradient
    float eps      = 1.0 / 2048.0;
    float h_dx     = texture(displacement_map, v.uv + vec2(eps, 0.0)).r;
    float h_dy     = texture(displacement_map, v.uv + vec2(0.0, eps)).r;
    // Finite difference normal: gradient of height → tangent-space normal
    vec3  disp_normal = normalize(vec3(-(h_dx - height) / eps * displace_scale,
                                       -(h_dy - height) / eps * displace_scale,
                                       1.0));
    // Transform tangent-space normal to object space using TBN
    vec3  T = v.tangent.xyz;
    vec3  B = cross(v.normal, T) * v.tangent.w;
    v.normal = normalize(T * disp_normal.x + B * disp_normal.y + v.normal * disp_normal.z);

    dst[idx] = v;
}
```

#### BVH Rebuild After Displacement
For ray-traced shadows or reflections to see the displaced geometry, the TLAS must be rebuilt against the displaced vertex buffer each frame (for dynamic/animated displacement) or once at load time (for static terrain). The BVH rebuild cost is the primary limit on displacement frequency.

```c
// After compute displacement, trigger BLAS rebuild against new vertex buffer
VkAccelerationStructureBuildGeometryInfoKHR build_info = {
    .type  = VK_ACCELERATION_STRUCTURE_TYPE_BOTTOM_LEVEL_KHR,
    .flags = VK_BUILD_ACCELERATION_STRUCTURE_PREFER_FAST_BUILD_BIT_KHR,
    .mode  = VK_BUILD_ACCELERATION_STRUCTURE_MODE_UPDATE_KHR,  // update from prior BLAS
    // geometry points to displaced vertex buffer
};
vkCmdBuildAccelerationStructuresKHR(cmd, 1, &build_info, &pRangeInfo);
```

**Use cases** — terrain heightmap with RT shadow contact; ocean surface swell with correct silhouette; character skin wrinkle system driven by blend shape weights.  
**Reference** — [Donnelly: GPU Gems 2: Per-Pixel Displacement Mapping (2005)](https://developer.nvidia.com/gpugems/gpugems2/part-i-geometric-complexity/chapter-8-pixel-displacement-mapping-distance-functions)

---

### Phong Tessellation and Adaptive Tessellation

The geometry/mesh shader section covers PN Triangles for smooth curved surfaces and mesh shader cluster culling. Two missing tessellation patterns are in universal use: **Phong tessellation** (simpler curved-surface approximation than PN Triangles) and **adaptive tessellation** (dynamically scaling the tessellation factor to maintain roughly one triangle per pixel).

**Shader stage**: tessellation runs in the **tessellation control shader** (TCS, sets tessellation levels) and **tessellation evaluation shader** (TES, computes displaced vertex positions). Vulkan exposes this via the `tessellationShader` pipeline stage.

#### Phong Tessellation
Phong tessellation projects each tessellated interior point onto the average of the three vertex tangent planes, weighted by barycentric coordinates. It is simpler than PN Triangles (no cubic Bézier patch construction) and produces smoother silhouettes than flat tessellation with minimal extra cost.

```glsl
// Tessellation Evaluation Shader — Phong tessellation
layout(triangles, equal_spacing, ccw) in;

layout(location=0) in vec3 in_pos[];    // 3 control point positions
layout(location=1) in vec3 in_normal[]; // 3 control point normals

layout(location=0) out vec3 out_pos;
layout(location=1) out vec3 out_normal;

uniform mat4 proj_view;
uniform float phong_strength;   // 0.0 = flat, 1.0 = full Phong; typically 0.75

// Project point p onto tangent plane at vertex v with normal n
vec3 project_to_plane(vec3 p, vec3 v, vec3 n) {
    return p - dot(p - v, n) * n;
}

void main() {
    vec3 bc = gl_TessCoord;   // barycentric coordinates

    // Linear interpolation (flat tessellation baseline)
    vec3 P = bc.x * in_pos[0] + bc.y * in_pos[1] + bc.z * in_pos[2];
    vec3 N = normalize(bc.x * in_normal[0] + bc.y * in_normal[1] + bc.z * in_normal[2]);

    // Phong projection: project P onto each vertex tangent plane, blend
    vec3 P0 = project_to_plane(P, in_pos[0], in_normal[0]);
    vec3 P1 = project_to_plane(P, in_pos[1], in_normal[1]);
    vec3 P2 = project_to_plane(P, in_pos[2], in_normal[2]);
    vec3 P_phong = bc.x * P0 + bc.y * P1 + bc.z * P2;

    // Blend between flat and Phong
    out_pos    = mix(P, P_phong, phong_strength);
    out_normal = N;
    gl_Position = proj_view * vec4(out_pos, 1.0);
}
```

**Use cases** — character skin subdivision, smooth terrain, any mesh intended for smooth silhouettes without authoring Bézier patches.  
**Reference** — [Boubekeur & Alexa: Phong Tessellation (SIGGRAPH Asia 2008)](https://dl.acm.org/doi/10.1145/1457515.1409020)

#### Adaptive Tessellation (Distance and Screen-Space Edge Length)
The tessellation control shader computes a tessellation factor for each edge proportional to the projected screen-space length of that edge, targeting approximately N pixels per tessellated edge. This concentrates triangles where the mesh is close and reduces them where it is distant — maintaining nearly constant screen-space triangle density.

```glsl
// Tessellation Control Shader — adaptive level-of-detail
layout(vertices = 3) out;

layout(location=0) in vec3 in_pos[];
layout(location=0) out vec3 out_pos[];

uniform mat4  proj_view;
uniform vec2  screen_size;
uniform float target_pixels_per_edge;  // e.g. 16.0

float screen_edge_length(vec3 a, vec3 b) {
    vec4 ca = proj_view * vec4(a, 1.0);
    vec4 cb = proj_view * vec4(b, 1.0);
    vec2 sa = ca.xy / ca.w * screen_size * 0.5;
    vec2 sb = cb.xy / cb.w * screen_size * 0.5;
    return length(sa - sb);
}

float tess_level(vec3 a, vec3 b) {
    return clamp(screen_edge_length(a, b) / target_pixels_per_edge, 1.0, 64.0);
}

void main() {
    out_pos[gl_InvocationID] = in_pos[gl_InvocationID];
    if (gl_InvocationID == 0) {
        gl_TessLevelOuter[0] = tess_level(in_pos[1], in_pos[2]);
        gl_TessLevelOuter[1] = tess_level(in_pos[2], in_pos[0]);
        gl_TessLevelOuter[2] = tess_level(in_pos[0], in_pos[1]);
        gl_TessLevelInner[0] = (gl_TessLevelOuter[0] +
                                 gl_TessLevelOuter[1] +
                                 gl_TessLevelOuter[2]) / 3.0;
    }
}
```

**Frustum and backface culling in TCS**: before computing tessellation levels, cull the entire patch if its bounding sphere is outside the view frustum or all three normals face away from the camera — avoiding tessellation of invisible geometry.

**Use cases** — terrain heightmap tessellation (pairs with §14 CDLOD), character skin refinement, displacement-mapped surfaces; universal in all production tessellation pipelines.  
**Reference** — [Cantlay: DirectX 11 Terrain Tessellation (NVIDIA GPU Pro)](https://developer.nvidia.com/sites/default/files/akamai/gameworks/samples/PN-AEN-Triangles.pdf)

---

### Geometry and Mesh Shader Patterns

These are shader-stage patterns rather than shading algorithms: techniques that operate on geometry topology, vertex attributes, or the amplification/culling of primitives before they reach the fragment stage. Mesh shaders (task + mesh) have replaced geometry shaders for most new work, offering better GPU occupancy and explicit meshlet-level culling. The entries here span debug visualisation (wireframe, face normals), tessellation and subdivision, instanced geometry (fur shells), and the mesh shader cluster culling pattern that underpins GPU-driven rendering. Many of these patterns are described in more detail in Ch154 (GPU-Driven Rendering); this section gives a catalog-level summary for quick reference.

**Shader stage**: This section is explicitly about shader stages — each entry names its own stage. Summary: wireframe barycentric overlay → **fragment shader**; mesh shader culling → **task shader** (amplification) + **mesh shader**; PN triangles → **tessellation control** (TCS) + **tessellation evaluation** (TES); fur shells → **vertex shader** (shell offset) or legacy **geometry shader** (shell emit); face normals → **fragment shader** (`dFdx`/`dFdy`).

#### Wireframe Overlay via Barycentric Coordinates
Passes barycentric coordinates as a vertex attribute and uses their minimum value in the fragment shader to draw lines along triangle edges without geometry duplication or a second render pass.

**Use cases** — debug visualisation, technical illustration, stylised wireframe aesthetics.  
**Reference** — [Quilez: Inverse Bilinear Interpolation / Barycentric Wireframe](https://iquilezles.org/articles/ibilinear/)

#### Mesh Shader Cluster Culling
Task (amplification) shader tests each meshlet against frustum and occlusion criteria and emits only visible meshlets for the mesh shader stage, eliminating invisible geometry before vertex processing.

**Use cases** — the core of GPU-driven rendering LOD and culling; see Ch154 for full coverage.  
**Reference** — [Wihlidal: GPU-Driven Rendering Pipelines (SIGGRAPH 2015)](https://advances.realtimerendering.com/s2015/)

#### PN Triangles (Curved Point-Normal Triangles)
Replaces each linear triangle with a degree-3 Bézier patch computed from the vertices and their normals, producing smooth curved surfaces without explicit subdivision topology.

**Use cases** — real-time tessellation of low-poly models where subdivision is not available; DX11-era tessellation demo technique.  
**Reference** — [Vlachos et al.: Curved PN Triangles (I3D 2001)](https://doi.org/10.1145/364338.364349)

#### Geometry Shader Fur Shells
Emits multiple offset copies of the base mesh (shells) with opacity decreasing toward the outermost shell, producing a volume fur effect without individual strand geometry.

**Use cases** — stylised fur, grass, feathers; cheap alternative to strand rendering for non-close-up distances.  
**Limitations** — overdraw cost scales with shell count; silhouette is always flat (no strand silhouettes); superseded by strand-based approaches for hero characters.

#### Face Normal via Screen-Space Derivatives
Computes a flat face normal in the fragment shader as the cross product of `dFdx(worldPos)` and `dFdy(worldPos)`, eliminating the need for a normal vertex attribute on simple faceted geometry.

**Use cases** — procedural geometry, voxel terrain, debug normals on unlit meshes.  
**Reference** — [Inigo Quilez: Flat Shading](https://iquilezles.org/articles/areas/)

---


---

## IX. Environment and Simulation

Environmental effects — terrain, water, foliage, fog, and volumetrics — span the geometry and shading domains. Terrain systems manage LOD and streaming for kilometre-scale meshes; water surfaces require FFT-based wave spectra and fluid-surface shaders; particle systems and GPU simulation drive foliage motion and dynamic environmental detail; and volumetrics and height fog add atmospheric depth and aerial perspective. These techniques are grouped because they share a common concern: representing continuous, large-scale natural phenomena on a GPU that prefers discrete, bounded geometry.

### Terrain, Vegetation, and Foliage

Large-scale natural environments present a distinct set of rendering problems: terrain must remain detailed near the camera and degrade gracefully at distance (LOD), foliage must be instanced at counts impossible to process on the CPU, and wind animation must be cheap enough to run per-vertex for millions of blades or leaves simultaneously. These techniques are tightly coupled to the GPU-driven rendering pipeline described in Ch154: terrain LOD decisions (CDLOD), grass instance culling, and draw-call generation all happen in compute shaders, with the results consumed by mesh or vertex shaders for final rendering.

**Shader stage**: Terrain heightmap tessellation uses the **tessellation control shader** (TCS) to set tessellation factors and the **tessellation evaluation shader** (TES) to displace vertices from the heightmap. CDLOD quadtree LOD selection runs in a **compute shader**; final rendering uses a standard **vertex + fragment** pair. GPU grass generation uses a **task shader** (culling) + **mesh shader** (blade geometry emit) + **fragment shader** (blade shading). Wind animation runs entirely in the **vertex shader**.

#### Heightmap Tessellation with Displacement
Tessellates a coarse terrain mesh and displaces vertices along the normal by a heightmap, producing smooth terrain with LOD-driven tessellation factor based on view distance and screen error.

**Use cases** — open-world terrain LOD; the standard approach in Unreal Landscape, Unity Terrain, and custom engines.  
**Reference** — [GPU Gems 2 Ch2: Terrain Rendering Using GPU-Based Geometry Clipmaps](https://developer.nvidia.com/gpugems/gpugems2/part-i-geometric-complexity/chapter-2-terrain-rendering-using-gpu-based-geometry)

#### CDLOD Terrain (Continuous Distance-Dependent LOD)
A quadtree-based terrain LOD system that morphs smoothly between LOD levels using vertex interpolation, preventing popping without explicit crack fixing.

**Use cases** — the reference algorithm for real-time streaming terrain LOD; Strugar 2009 paper is the standard implementation guide.  
**Reference** — [Strugar: Continuous Distance-Dependent LOD for Rendering Heightmaps (2009)](https://www.vertexasylum.com/downloads/cdlod/cdlod_latest.pdf)

#### GPU Grass Rendering
Generates blade geometry (triangle strips or mesh shader amplification) per grass instance from a density map, with per-blade wind animation driven by a Fourier wind model sampled from a texture.

**Use cases** — large-scale grass fields without per-blade CPU simulation; Horizon Forbidden West, Ghost of Tsushima grass systems.  
**Reference** — [Acerola: Grass Rendering and Simulation](https://www.acerola.technology/articles/grass/) | [Advances 2021: Ghost of Tsushima Grass](https://advances.realtimerendering.com/s2021/)

#### Wind Animation (Vertex Shader)
Displaces foliage vertices by a time-varying wind force sampled from a scrolling flow texture, with displacement weighted by a stiffness mask (typically vertex colour) to keep base stems anchored.

**Use cases** — trees, bushes, grass, cloth; the standard approach in SpeedTree and game engine foliage shaders.  
**Reference** — [GPU Gems 3 Ch16: Vegetation Procedural Animation and Shading](https://developer.nvidia.com/gpugems/gpugems3/part-iii-rendering/chapter-16-vegetation-procedural-animation-and-shading)

#### Octahedral Impostor
Bakes N×M orthographic views of a 3D object distributed over an octahedral sphere into an atlas texture. At runtime, selects the two nearest baked views based on the view direction and blends between them, producing a convincing representation of the 3D object at a fraction of its polygon cost.

**Use cases** — distant trees, rocks, and complex props where full LOD geometry is too expensive; the technique behind SpeedTree and UE5 Nanite foliage impostors.  
**Key variants** — alpha-clipped 8-frame impostors (cheap, coarse); full octahedral 64-frame impostors (smooth, higher memory); animated impostors (multiple frames baked per octant for e.g. water).  
**Reference** — [Shaderbits: Octahedral Impostors (2018)](https://shaderbits.com/blog/octahedral-impostors)

---

### Terrain Material Splatting

Terrain surfaces blend between multiple materials (rock, grass, dirt, snow) based on altitude, slope angle, and curvature — not a single material UV-mapped over the mesh. A **splat map** encodes material weights per terrain texel (one channel per material layer, up to 4 per RGBA8 splat texture), sampled in the fragment shader and used to blend tiled material textures.

**Shader stage**: terrain material blending runs in the **fragment shader** of the terrain mesh (heightmap tessellated or mesh-based).

#### RGBA Splat Map Blending
The simplest implementation: sample the splat map at the terrain UV, use the four channels as blend weights, trilinearly sample four tiled material textures, and lerp between them.

```glsl
layout(set=0, binding=0) uniform sampler2D splat_map;      // RGBA8: per-channel weight
layout(set=0, binding=1) uniform sampler2DArray mat_albedo; // layer 0..3 albedo
layout(set=0, binding=2) uniform sampler2DArray mat_normal; // layer 0..3 normal
layout(set=0, binding=3) uniform sampler2DArray mat_height; // layer 0..3 height for blend

layout(location=0) in vec2 terrain_uv;
layout(location=1) in vec2 tiled_uv;    // terrain_uv * tile_scale

void main() {
    vec4 weights = texture(splat_map, terrain_uv);
    weights /= (weights.r + weights.g + weights.b + weights.a + 1e-6); // renormalise

    vec3 albedo = vec3(0.0);
    vec3 normal = vec3(0.0);
    for (int i = 0; i < 4; ++i) {
        albedo += weights[i] * texture(mat_albedo, vec3(tiled_uv, float(i))).rgb;
        normal += weights[i] * texture(mat_normal, vec3(tiled_uv, float(i))).rgb;
    }
    // ... lighting with blended albedo and normal
}
```

#### Height-Based Blend Sharpening
Naïve linear blending produces blurry material transitions. Sharpening by the material height map at the transition gives natural overlap (rock peaks through grass in crevices, snow settles on flat areas only):

```glsl
// Height-aware blend — each material's height map biases its weight at transitions
vec4 h = vec4(
    texture(mat_height, vec3(tiled_uv, 0.0)).r,
    texture(mat_height, vec3(tiled_uv, 1.0)).r,
    texture(mat_height, vec3(tiled_uv, 2.0)).r,
    texture(mat_height, vec3(tiled_uv, 3.0)).r
);
// Add height to base weight, then sharpen with a contrast parameter
vec4 blend = weights + h * 0.4;
float blend_max = max(max(blend.r,blend.g), max(blend.b,blend.a));
blend = max(blend - (blend_max - 0.2), vec4(0.0));  // contrast=0.2
blend /= (blend.r + blend.g + blend.b + blend.a + 1e-6);
```

**Reference** — [Golus: Avoiding Texture Repetition (2018)](https://bgolus.medium.com/normal-mapping-for-a-triplanar-shader-10bf39dca05a)

#### Slope-Based and Altitude-Based Layer Assignment
Computes per-fragment material weight from the terrain surface normal slope and world-space height, without a pre-baked splat map:

```glsl
// Slope-based: rock on steep slopes, grass on flat
float slope   = 1.0 - abs(dot(world_normal, vec3(0,1,0)));  // 0=flat, 1=vertical
float w_grass = smoothstep(0.5, 0.3, slope);
float w_rock  = smoothstep(0.3, 0.5, slope);

// Altitude-based: snow above a height threshold
float altitude = world_pos.y;
float w_snow   = smoothstep(snow_altitude - 5.0, snow_altitude + 5.0, altitude);

// Final 3-way blend (normalise to sum to 1):
float total  = w_grass + w_rock + w_snow + 1e-6;
w_grass /= total;  w_rock /= total;  w_snow /= total;
```

**Use cases** — procedural terrain without artist-painted splat maps; runtime material selection for infinite procedural worlds.  
**Reference** — [UE5 Landscape Technical Guide](https://docs.unrealengine.com/5.0/en-US/landscape-technical-guide-in-unreal-engine/)

#### Puddle Accumulation (Wetness / Rain)
Modulates material roughness and blends in a water normal map in concave areas (where rain would collect), estimated by the world-space up-facing fraction of the surface normal:

```glsl
float flatness   = pow(max(0.0, dot(world_normal, vec3(0,1,0))), 4.0);
float wetness    = rain_intensity * flatness;
float roughness  = mix(dry_roughness, 0.0, wetness);  // wet = very smooth
vec3  wet_normal = mix(surface_normal, water_ripple_normal, wetness * 0.5);
```

**Reference** — [Lagarde & de Rousiers: Moving Frostbite to PBR (SIGGRAPH 2014)](https://seblagarde.wordpress.com/2015/07/14/siggraph-2014-moving-frostbite-to-physically-based-rendering/)

---

### Water, Ocean, and Fluid Surface

Water surface rendering spans a wide quality range depending on scale and fidelity requirements. Gerstner waves are the analytic minimum — cheap, art-directable, no FFT — and are sufficient for stylised or background water. FFT ocean (Tessendorf) produces statistically correct open-ocean surfaces and is the standard in AAA nautical and simulation titles. Both approaches feed into the same shading model: Fresnel-blended transmission (refraction of the underwater scene) and reflection (SSR or IBL), with a chromatic offset for realism. Caustics and foam are typically separate passes composited on top.

**Shader stage**: Gerstner wave displacement runs in the **vertex shader** (displaces mesh vertices per-frame). FFT ocean spectrum evolution and IFFT run in **compute shaders** producing displacement/normal map textures; a standard **vertex shader** then samples these textures to displace the water mesh, and the **fragment shader** handles the Fresnel/refraction/reflection shading. The water Fresnel and refraction logic is entirely **fragment shader**.

#### Gerstner Waves
An analytic trochoidal wave model that displaces surface vertices along circular orbits rather than vertically, producing the characteristic peaked crests and flat troughs of ocean surface waves.

**Use cases** — stylised and semi-realistic ocean/lake surfaces in real-time rendering; cheap enough for mobile.  
**Key variants** — sum of N Gerstner waves with different amplitude/frequency/direction; Jonswap spectrum fitting for realistic energy distribution.  
**Limitations** — loop instability at high steepness (vertices cross); no breaking waves or foam without supplementary simulation.  
**Reference** — [GPU Gems Ch1: Effective Water Simulation from Physical Models](https://developer.nvidia.com/gpugems/gpugems/part-i-natural-effects/chapter-1-effective-water-simulation-physical-models)

#### FFT Ocean (Tessendorf)
Simulates ocean surface using the inverse FFT of a statistical wave spectrum (Phillips or JONSWAP), producing statistically correct ocean displacement and normal maps updated per frame.

**Use cases** — realistic large-scale ocean in games, film, ship simulation; used in Sea of Thieves, Assassin's Creed Black Flag, and most AAA ocean systems.  
**Limitations** — repeating tiling pattern visible at large scales without tiling suppression; FFT cost scales with grid resolution.  
**Reference** — [Tessendorf: Simulating Ocean Water (SIGGRAPH 1999 Course Notes)](https://people.computing.clemson.edu/~jtessen/reports/papers_files/coursenotes2004.pdf)

#### Water Fresnel and Refraction
Blends transmitted (refracted, chromatic-offset underwater scene) and reflected (SSR or IBL) contributions by the Fresnel equation as a function of view angle.

**Use cases** — any water surface rendering; shallow water shows more refraction, deep water more reflection.  
**Reference** — [GPU Gems Ch2: Rendering Water Caustics](https://developer.nvidia.com/gpugems/gpugems/part-i-natural-effects/chapter-2-rendering-water-caustics)

---

### Fluid Simulation — SPH and Eulerian Grids

Real-time fluid simulation on the GPU uses either **particle-based (Lagrangian) SPH** — where each particle carries mass, velocity, and pressure — or **grid-based (Eulerian) Navier-Stokes** — where velocity and pressure are stored on a voxel grid and advected each step. Both run entirely in compute shaders, writing results into SSBOs or 3D textures that feed the rendering pipeline.

**Shader stage**: all simulation dispatches are **compute shaders**. SPH rendering uses the particle billboard pipeline from §24; grid-based fluids render via isosurface extraction (§39) or direct volume rendering.

#### SPH — Smoothed Particle Hydrodynamics
SPH approximates fluid quantities (density, pressure, velocity) as a weighted sum over nearby particles using a smooth kernel function `W(r, h)` (e.g., the cubic spline or the Poly6 kernel), where `h` is the smoothing radius. Each frame: build neighbour list, compute density/pressure, accumulate forces, integrate.

```glsl
// SPH density computation — one thread per particle
layout(local_size_x=256) in;
layout(set=0, binding=0) buffer Positions { vec4 pos[]; };
layout(set=0, binding=1) buffer Densities { float density[]; };
// Neighbour list built in prior pass (spatial hash, §42)
layout(set=0, binding=2) readonly buffer Neighbors { uint neighbors[MAX_PART][MAX_NEIGH]; };
layout(set=0, binding=3) readonly buffer NCount    { uint ncount[]; };

uniform float h;       // smoothing radius
uniform float mass;    // particle mass (uniform for all particles)
uniform float poly6_k; // 315 / (64π h^9)

float poly6(float r2, float h2) {
    float x = h2 - r2;
    return (x > 0.0) ? poly6_k * x * x * x : 0.0;
}

void main() {
    uint i = gl_GlobalInvocationID.x;
    if (i >= particle_count) return;
    vec3  pi  = pos[i].xyz;
    float rho = 0.0;
    float h2  = h * h;
    for (uint k = 0; k < ncount[i]; ++k) {
        uint j   = neighbors[i][k];
        vec3 r   = pi - pos[j].xyz;
        float r2 = dot(r, r);
        rho     += mass * poly6(r2, h2);
    }
    density[i] = rho;
}
```

**Pressure force** (Spiky kernel gradient) and **viscosity force** (Laplacian) follow the same neighbour loop with different kernel functions. Pressure is computed from the equation of state `p = k(ρ - ρ₀)`.

**Use cases** — GPU water splashes, fluid-coupled particles, lava, granular materials.  
**Limitations** — SPH is compressible and requires small timesteps; WCSPH or PCISPH correct for density errors.  
**Reference** — [Müller et al.: Particle-Based Fluid Simulation for Interactive Applications (SCA 2003)](https://dl.acm.org/doi/10.5555/846276.846298); [NVIDIA FleX](https://developer.nvidia.com/flex)

#### Eulerian Grid Fluid (Stable Fluids)
Stores velocity `u` and pressure `p` on a MAC (Marker-and-Cell) staggered 3D grid in 3D textures or SSBOs. Each simulation step:

1. **Advect** velocity semi-Lagrangian: trace particle backward `x' = x - u(x)·dt`, sample velocity at `x'` via trilinear interpolation.
2. **Apply forces** (gravity, buoyancy, vorticity confinement).
3. **Pressure solve** (Jacobi or conjugate gradient iterations): ensure divergence-free velocity field via Poisson equation `∇²p = ∇·u/dt`.
4. **Project**: subtract pressure gradient from velocity `u := u - ∇p·dt`.

```glsl
// Jacobi pressure iteration — one compute pass per iteration (20–50 iterations)
layout(set=0, binding=0) uniform sampler3D pressure_in;
layout(set=0, binding=1) uniform sampler3D divergence;
layout(set=0, binding=2, r32f) uniform image3D pressure_out;

void main() {
    ivec3 p    = ivec3(gl_GlobalInvocationID);
    float div  = texelFetch(divergence, p, 0).r;
    float p_xm = texelFetch(pressure_in, p + ivec3(-1,0,0), 0).r;
    float p_xp = texelFetch(pressure_in, p + ivec3( 1,0,0), 0).r;
    float p_ym = texelFetch(pressure_in, p + ivec3(0,-1,0), 0).r;
    float p_yp = texelFetch(pressure_in, p + ivec3(0, 1,0), 0).r;
    float p_zm = texelFetch(pressure_in, p + ivec3(0,0,-1), 0).r;
    float p_zp = texelFetch(pressure_in, p + ivec3(0,0, 1), 0).r;
    float new_p = (p_xm + p_xp + p_ym + p_yp + p_zm + p_zp - div) / 6.0;
    imageStore(pressure_out, p, vec4(new_p));
}
```

**Use cases** — smoke, fire, explosions, weather effects; grid resolution of 64³–256³ is real-time feasible on desktop GPUs.  
**Reference** — [Stam: Stable Fluids (SIGGRAPH 1999)](https://dl.acm.org/doi/10.1145/311535.311548); [Harris: Fast Fluid Dynamics Simulation on the GPU (GPU Gems Ch38)](https://developer.nvidia.com/gpugems/gpugems/part-vi-beyond-triangles/chapter-38-fast-fluid-dynamics-simulation-gpu)

---

### GPU Particle Systems

CPU particle systems bottleneck at ~100k particles per emitter. GPU particle systems maintain the particle pool in SSBOs with atomic counters, run spawn and update in compute shaders, and submit draw calls via `vkCmdDrawIndirect` without CPU readback.

**Shader stage**: Spawn and update run in **compute shaders**. A **vertex shader** expands particle IDs to screen-space quads. A **fragment shader** applies texture and alpha. Depth sorting (for correct alpha compositing) uses the radix sort from §19 (GPU Compute Algorithm Primitives).

#### Particle Pool with Dead/Alive Lists
Maintains two SSBOs — a dead list and an alive list — each with an atomic counter. Spawn: pop from dead list, push to alive list. Death: pop from alive list, push to dead list. The alive list is passed as the indirect draw argument source.

```glsl
// Particle spawn compute shader — one thread per new particle
layout(set=0, binding=0) buffer DeadList  { uint dead_ids[]; };
layout(set=0, binding=1) buffer AliveList { uint alive_ids[]; };
layout(set=0, binding=2) buffer DeadCount { uint dead_count; };
layout(set=0, binding=3) buffer AliveCount{ uint alive_count; };
layout(set=0, binding=4) buffer Particles { Particle particles[]; };

layout(local_size_x=64) in;
void main() {
    if (gl_GlobalInvocationID.x >= spawn_count) return;
    uint dead_idx = atomicAdd(dead_count, uint(-1)) - 1;
    uint id       = dead_ids[dead_idx];
    uint alive_idx= atomicAdd(alive_count, 1u);
    alive_ids[alive_idx] = id;
    // initialise particle at id...
    particles[id].pos      = emitter_pos;
    particles[id].vel      = random_hemisphere_dir(id) * spawn_speed;
    particles[id].lifetime = max_lifetime;
}
```

**Reference** — [GPU Pro 6: GPU-Based Particle Systems](https://gpupro.blogspot.com/)

#### Particle Update (Integration + Death)
Each compute thread integrates one particle: applies velocity, gravity, drag; decrements lifetime; on death, moves the particle from alive to dead list via atomic operations. Alive count after update becomes the `instanceCount` for the indirect draw command.

**Use cases** — physics-simulated particles (fire, sparks, rain, snow, explosion debris); any simulation that would stall the CPU.

#### Billboard Expansion (Vertex Shader)
A vertex shader receives a particle instance ID, reads position/size/colour/rotation from the particle SSBO, and expands a unit quad (4 vertices) to the correct screen-aligned or velocity-aligned orientation.

**Use cases** — all sprite-based particle rendering; soft particles (fade by comparing particle depth to scene depth).  
**Reference** — [GPU Gems 3 Ch23: High-Speed, Off-Screen Particles](https://developer.nvidia.com/gpugems/gpugems3/part-iv-image-effects/chapter-23-high-speed-screen-space-occlusion-ambient)

---

### Exponential and Height Fog

Ray-marched volumetric fog (§9) provides physically correct in-scattering but costs one ray per pixel per step. Analytical fog functions compute the fog transmittance and in-scattered luminance exactly along the view ray through a homogeneous or height-stratified atmosphere in a single fragment shader instruction — at near-zero cost. Every game uses analytical fog for baseline atmospheric depth; volumetric fog is layered on top for local effects.

**Shader stage**: analytical fog is applied in the **fragment shader** at the end of the lighting evaluation, or as a full-screen post-process combining the depth buffer with the scene colour.

#### Exponential Fog
Density is constant throughout the volume; transmittance along a ray of length `d` through density `ρ` is `T = exp(-ρ·d)`. In-scattered colour is the fog colour modulated by `1 - T`.

```glsl
// Exponential fog — applied after lighting in fragment shader
uniform float fog_density;    // e.g. 0.002
uniform vec3  fog_color;      // sky or ambient colour
uniform vec3  cam_pos;

vec3 apply_exponential_fog(vec3 lit_color, vec3 world_pos) {
    float dist = length(world_pos - cam_pos);
    float T    = exp(-fog_density * dist);           // transmittance
    return mix(fog_color, lit_color, T);             // blend scene with fog
}
```

**Sun scattering variant**: bias the fog colour toward a brighter value when the view direction aligns with the sun direction, simulating the Mie forward-scattering peak:

```glsl
float sun_factor  = pow(max(0.0, dot(view_dir, sun_dir)), 8.0);
vec3  fog_col_sun = mix(fog_color, sun_color * 2.0, sun_factor * 0.5);
```

#### Exponential Height Fog
Density falls off exponentially with altitude: `ρ(y) = ρ₀ · exp(-β · y)`. The integral of this along a view ray has a closed-form solution, giving analytically correct height-stratified fog that is thick in valleys and thin on hilltops.

```glsl
// Height fog — analytic integral along view ray
// Based on: Gu et al. (2006) and UE4 ExponentialHeightFog
uniform float fog_density;     // surface density ρ₀
uniform float fog_falloff;     // height falloff β (larger = thinner layer)
uniform float fog_height;      // altitude at which fog begins (world-space Y)
uniform vec3  fog_color;
uniform vec3  cam_pos;

float height_fog_integral(vec3 ray_start, vec3 ray_end) {
    float dy   = ray_end.y - ray_start.y;
    float y0   = ray_start.y - fog_height;
    float y1   = ray_end.y   - fog_height;
    float len  = length(ray_end - ray_start);
    float beta = fog_falloff;

    // Analytic integral: ∫ ρ₀·exp(-β·y) dy = (ρ₀/β)·(exp(-β·y0) - exp(-β·y1))
    float col_den;
    if (abs(dy) < 1e-4) {
        col_den = fog_density * exp(-beta * y0) * len;
    } else {
        col_den = fog_density / beta *
                  abs((exp(-beta * y0) - exp(-beta * y1)) / (dy / len));
    }
    return col_den;
}

vec3 apply_height_fog(vec3 lit_color, vec3 world_pos) {
    float optical_depth = height_fog_integral(cam_pos, world_pos);
    float T             = exp(-optical_depth);
    return mix(fog_color, lit_color, T);
}
```

**Use cases** — atmospheric depth in every 3D game; height fog for ground-level haze, morning mist, rain-soaked urban streets; used as the base fog layer in UE4/UE5, Unity HDRP, and Godot 4.  
**Reference** — [Unreal Engine 4: Exponential Height Fog](https://docs.unrealengine.com/4.27/en-US/BuildingWorlds/FogEffects/HeightFog/); [Wenzel: Real-Time Atmospheric Effects in Games (GDC 2006)](https://developer.nvidia.com/gpugems/gpugems2/part-ii-shading-antialiasing-and-shadows/chapter-16-accurate-atmospheric-scattering)

---

### Volumetric Rendering and Participating Media

Participating media — fog, smoke, fire, clouds, underwater haze — scatter and absorb light along a ray rather than only at surfaces. The physics are governed by three coefficients: absorption (σ_a), scattering (σ_s), and extinction (σ_t = σ_a + σ_s). In real-time rendering, the full volume rendering equation is approximated either analytically (Beer-Lambert for homogeneous media) or by ray-marching a 3D density field with in-scattering accumulation. The froxel (frustum-aligned voxel grid) technique amortises this cost by pre-computing scattering into a 3D texture, reducing per-pixel cost to a single texture fetch. Clouds and atmosphere occupy the high end of cost and quality in this section.

**Shader stage**: Froxel scattering accumulation runs in a **compute shader** (fills a 3D texture). Per-pixel volumetric fog lookup runs in the **fragment shader** (single 3D texture sample). Raymarched volumes (clouds, atmosphere) run in the **fragment shader** or as a dedicated full-screen **compute shader** writing to a half-resolution render target that is then upscaled and composited. Atmospheric precomputation (Bruneton) runs entirely in **compute shaders** offline.

#### Beer-Lambert Extinction
Computes transmittance along a ray through a homogeneous participating medium as exp(−σ_t × d), where σ_t is the extinction coefficient and d is path length.

**Use cases** — the baseline formula for fog, haze, underwater rendering, smoke transmittance.  
**Reference** — [PBR Book: Volume Scattering](https://pbr-book.org/4ed/Volume_Scattering)

#### Henyey-Greenstein Phase Function
A single-parameter (g, asymmetry factor) approximation to volumetric scattering directionality: g = 0 is isotropic, g > 0 forward-scattering (fog, smoke), g < 0 backward-scattering (retroreflective particles).

**Use cases** — all real-time volumetric fog and cloud scattering; fast GPU evaluation.  
**Reference** — [PBR Book: Phase Functions](https://pbr-book.org/4ed/Volume_Scattering/Phase_Functions)

#### Ray-Marched Volumetric Fog
Marches a ray through a 3D density field (texture or procedural), accumulating in-scattered light and transmittance at each step.

**Use cases** — atmospheric haze, local fog banks, explosion smoke; replaces depth fog with volumetric accuracy.  
**Key variants** — view-space froxel grid (Wronski 2014): pre-computes scattering into a frustum-aligned 3D texture for O(1) per-pixel lookup.  
**Reference** — [Wronski: Volumetric Fog (SIGGRAPH 2014)](https://advances.realtimerendering.com/s2014/)

#### Volumetric Clouds
Renders real-time physically plausible clouds by ray marching a 3D noise density field (weather map → cloud base/top coverage → detail erosion), with scattering computed via Henyey-Greenstein and temporal reprojection for cost amortisation.

**Use cases** — open-world games, flight simulators; the Nubis/Horizon Zero Dawn cloud system is the production reference.  
**Reference** — [Schneider & Vos: The Real-Time Volumetric Cloudscapes of Horizon Zero Dawn (SIGGRAPH 2015)](https://advances.realtimerendering.com/s2015/)

#### Atmospheric Scattering (Bruneton-Neyret)
Pre-computes Rayleigh (molecular, blue sky) and Mie (aerosol, haze) scattering into a set of lookup tables; at runtime, fetches the tables to compute sky colour, aerial perspective, and sun disc for any view/sun direction.

**Use cases** — physically correct sky rendering in games and simulations; runs in real time after precomputation.  
**Reference** — [Bruneton & Neyret: Precomputed Atmospheric Scattering (EGSR 2008)](https://inria.hal.science/inria-00288758/)

#### Procedural Star Field
Generates a night sky by hashing screen-space or view-space directions to determine star presence, brightness, and colour temperature, with a superimposed procedural Milky Way band (elongated noise field in galactic coordinates).

**Use cases** — night skies in open-world games, space simulations; runs at negligible cost as a full-screen fragment or sky-dome shader.  
**Key variants** — animated twinkling via time-varying hash perturbation; Milky Way density from a precomputed spherical texture; moon disc via signed distance from the moon direction vector.  
**Reference** — [Shadertoy: Procedural Stars (many implementations)](https://www.shadertoy.com/results?query=stars)

#### Procedural Aurora
Simulates aurora borealis / australis as undulating luminous curtains using layered domain-warped noise in clip space, with emission colour ramp driven by curtain height.

**Use cases** — arctic/antarctic environments, space games; pure fragment shader, no geometry.  
**Reference** — [Shadertoy: Aurora (Iq)](https://www.shadertoy.com/view/XtGGRt)

---

### Direct Volume Rendering

Direct Volume Rendering (DVR) ray-casts through a 3D scalar or RGBA volume texture, accumulating colour and opacity along the ray using a **transfer function** that maps each voxel value to a colour and opacity. Unlike isosurface extraction (§39), DVR does not produce a mesh — it renders the interior of the volume directly, revealing all density layers simultaneously. This is the primary rendering technique for CT/MRI medical imaging, scientific field visualization (fluid vorticity, electromagnetic fields), and smoke/explosion rendering from simulation volumes.

**Shader stage**: DVR runs as a full-screen **fragment shader** or **compute shader** that ray-marches through the bounding box of the volume. The volume is stored in a `sampler3D` (or `image3D` for write access). Each step samples the volume and composites the result into the running accumulator using front-to-back Porter-Duff OVER.

#### Transfer Function
Maps a scalar voxel density to a `(RGB, α)` emission-absorption tuple. A 1D texture lookup encodes the user-controlled mapping; multiple opacity peaks isolate different density layers (e.g., skin vs bone in a CT scan).

```glsl
layout(set=0, binding=0) uniform sampler3D volume;        // scalar density [0,1]
layout(set=0, binding=1) uniform sampler1D transfer_func; // density → RGBA
```

#### Front-to-Back Ray Casting
Composites N sample slabs along the view ray using the front-to-back OVER operator. Terminates early when accumulated opacity reaches 1.

```glsl
// DVR fragment shader — ray cast through axis-aligned volume bounding box
layout(location=0) out vec4 frag_color;

uniform mat4  inv_model;      // world → volume local space [0,1]³
uniform vec3  cam_pos_world;
uniform float step_size;      // in volume space (e.g. 1/256)

void main() {
    // Reconstruct world ray from fragment position
    vec3 ray_origin = (inv_model * vec4(cam_pos_world, 1.0)).xyz;
    vec3 ray_dir    = normalize((inv_model * vec4(reconstruct_world_dir(), 0.0)).xyz);

    // Intersect with unit cube [0,1]³
    vec3 tmin3 = (vec3(0.0) - ray_origin) / ray_dir;
    vec3 tmax3 = (vec3(1.0) - ray_origin) / ray_dir;
    vec3 t1    = min(tmin3, tmax3);
    vec3 t2    = max(tmin3, tmax3);
    float tenter = max(max(t1.x, t1.y), t1.z);
    float texit  = min(min(t2.x, t2.y), t2.z);
    if (tenter >= texit || texit < 0.0) discard;
    tenter = max(tenter, 0.0);

    // Front-to-back compositing
    vec4  result    = vec4(0.0);   // accumulated (RGB premultiplied, A)
    float t         = tenter;

    while (t < texit && result.a < 0.99) {
        vec3  pos     = ray_origin + t * ray_dir;
        float density = texture(volume, pos).r;
        vec4  sample  = texture(transfer_func, density); // RGBA from TF
        sample.a      = 1.0 - pow(1.0 - sample.a, step_size * 200.0); // opacity correction

        // Front-to-back OVER: C_out = C_in + (1 - A_in) * C_sample * A_sample
        result.rgb += (1.0 - result.a) * sample.rgb * sample.a;
        result.a   += (1.0 - result.a) * sample.a;

        t += step_size;
    }
    frag_color = vec4(result.rgb / max(result.a, 1e-6), result.a); // un-premultiply
}
```

#### Empty Space Skipping
A 3D min-max mip pyramid over the volume lets the ray skip regions that map to fully transparent transfer function values. For each ray step, look up the min-max at the current mip level; if the entire block is transparent, jump to the next non-empty block.

**Use cases** — medical CT/MRI visualisation, smoke and explosion rendering from simulation grids, scientific field visualization (fluid vorticity, temperature), cloud volume rendering.  
**Reference** — [Engel et al.: Real-Time Volume Graphics (AK Peters 2006)](https://www.real-time-volume-graphics.org/); [Krüger & Westermann: Acceleration Techniques for GPU-based Volume Rendering (IEEE Vis 2003)](https://doi.org/10.1109/VIS.2003.10056)

---


---


## Integrations

**Ch204** (this series) establishes the rasterisation pipeline within which procedural and environment techniques run. **Ch205** (this series) covers the material models (PBR, SSS, participating media absorption) that ray tracing evaluates at hit points. **Ch207** (this series) covers post-processing and denoising passes that consume ray-traced GI and AO outputs. **Ch135** (Vulkan Ray Tracing) provides the complete Vulkan API for Cat VII — acceleration structure lifecycle, shader binding tables, and the ray tracing pipeline. **Ch208**–**ch212** (GPU Geometry Algorithms) cover the geometry algorithms (BVH construction, SDF baking, Marching Cubes, terrain LOD) that feed the ray tracing and procedural pipelines catalogued here.
