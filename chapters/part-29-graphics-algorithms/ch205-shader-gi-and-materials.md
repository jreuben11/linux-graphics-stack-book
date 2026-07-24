# Chapter 205: Shader Algorithm Catalog — Global Illumination and Materials (Part XXIX)

*Part XXIX — Graphics Algorithms*

**Target audiences**: Graphics application developers implementing PBR materials, GI, and character shading; browser engineers mapping WebGPU lighting workloads; systems developers understanding radiance computation cost.

This chapter is the second volume of the Shader Algorithm Catalog. It covers the algorithms responsible for surface radiance: how indirect lighting propagates through a scene (Category IV), how surface micro-geometry responds to light (Category V), and how organic character materials — skin, hair, fur, cloth — are rendered at production quality (Category VI). Each entry follows the same structure as Chapter 204: what it computes, when to use it, named variants, costs, and a primary reference.

## Table of Contents

- **[IV. Global Illumination and Radiance](#iv-global-illumination-and-radiance)**
  - [Global Illumination](#global-illumination)
  - [Image-Based Lighting](#image-based-lighting)
  - [Dynamic Environment Map Capture](#dynamic-environment-map-capture)
  - [Lightmapping and Baked GI](#lightmapping-and-baked-gi)
  - [Stochastic Lightmap Baking via GPU Path Tracing](#stochastic-lightmap-baking-via-gpu-path-tracing)
  - [Light Propagation Volumes](#light-propagation-volumes)
  - [Screen-Space Global Illumination (SSGI)](#screen-space-global-illumination-ssgi)
  - [World-Space Radiance Cache](#world-space-radiance-cache)
  - [Spherical Harmonics Lighting](#spherical-harmonics-lighting)
  - [Spherical Gaussians](#spherical-gaussians)
- **[V. Material Models and BRDF](#v-material-models-and-brdf)**
  - [Physically Based Shading — BRDF Models](#physically-based-shading-brdf-models)
  - [Anisotropic BRDF](#anisotropic-brdf)
  - [Clear Coat and Sheen BRDFs](#clear-coat-and-sheen-brdfs)
  - [Glint and Sparkle Microstructure BRDF](#glint-and-sparkle-microstructure-brdf)
  - [Iridescence and Thin-Film Interference](#iridescence-and-thin-film-interference)
  - [Translucency and Backlit Thin-Surface Materials](#translucency-and-backlit-thin-surface-materials)
  - [Subsurface Scattering](#subsurface-scattering)
  - [Eye Rendering](#eye-rendering)
  - [Non-Photorealistic Rendering (NPR)](#non-photorealistic-rendering-npr)
- **[VI. Character and Organic Rendering](#vi-character-and-organic-rendering)**
  - [Skin, Hair, and Character Rendering](#skin-hair-and-character-rendering)
  - [Fur and Shell Rendering](#fur-and-shell-rendering)
  - [Cloth and Soft-Body Simulation](#cloth-and-soft-body-simulation)
  - [Skeletal Animation and Vertex Skinning](#skeletal-animation-and-vertex-skinning)


---

## IV. Global Illumination and Radiance

Global illumination addresses light that arrives at a surface after one or more bounces — indirect diffuse, indirect specular, and the accumulated irradiance from the full hemisphere. The techniques here range from cheap pre-baked solutions (lightmaps, spherical harmonic coefficients) through GPU-resident radiance structures (Light Propagation Volumes, world-space radiance caches) to screen-space and ML-upscaled approaches (SSGI). Spherical Gaussians and SH are placed here because they are radiance encoding schemes rather than material models. Entries are ordered roughly from static/offline to fully dynamic.

### Global Illumination

Global illumination (GI) accounts for light that has bounced at least once before reaching the camera — the soft fill light in a sunlit room, colour bleeding from a red wall onto adjacent objects, and the brightening of surfaces that face other bright surfaces. The fundamental trade-off is between temporal stability and dynamic scene support: pre-baked probes are stable but require re-baking when geometry changes; dynamic probe methods (DDGI, Radiance Cascades) update every frame but cost GPU time and require filtering. The entries progress from the most-deployed technique (irradiance probes + SH) through increasingly dynamic and ray-traced approaches, ending with neural radiance caching, which trades probe cost for neural network inference cost.

**Shader stage**: GI is multi-stage. Probe update passes run in **compute shaders** (casting rays, accumulating radiance). The final GI contribution is evaluated in the **fragment shader** (probe interpolation, SH evaluation, or screenspace probe lookup). DDGI and ReSTIR GI use **ray generation / closest-hit / miss shaders** for the probe ray cast. Radiance Cascades merging runs in **compute shaders**.

#### Irradiance Probes / Light Probes
Pre-baked or real-time probes placed in the scene that capture the incoming radiance from all directions (as a low-order spherical harmonic or small cubemap) and are blended at runtime based on interpolation weights.

**Use cases** — indoor and outdoor diffuse GI in all major game engines; the baseline GI solution.  
**Key variants** — baked static probes (Lightmass, Bakery); dynamic probes (DDGI, LPGI); SH L2 (9 coefficients) vs SH L3 (16).  
**Limitations** — probe density limits spatial variation; sharp lighting features (narrow sun shafts) cannot be captured in SH.  
**Reference** — [Ramamoorthi & Hanrahan: Irradiance Environment Maps (SIGGRAPH 2001)](https://doi.org/10.1145/383259.383317)

#### DDGI (Dynamic Diffuse Global Illumination)
Places a grid of probes that each cast rays, accumulate radiance into a probe texture, and blend at runtime to produce dynamic, bounce-illuminated GI without baking.

**Use cases** — dynamic scenes where lights or geometry move; used in Unreal Engine 5's Lumen (probe component).  
**Key variants** — Majercik et al. (2019) original; Lumen's Radiance Cache (far-field probes).  
**Reference** — [Majercik et al.: Dynamic Diffuse Global Illumination with Ray-Traced Irradiance Fields (JCGT 2019)](https://jcgt.org/published/0008/02/01/)

#### Radiance Cascades
A screen-space GI algorithm that maintains a hierarchy of probe grids at exponentially increasing spacing and angular resolution, merging cascades to propagate light from far intervals into near intervals.

**Use cases** — 2D and 3D real-time GI without baking; growing adoption in indie and real-time rendering (Godot 4 GI, community implementations).  
**Key variants** — 2D (Lenn 2024 original); 3D extensions in development.  
**Reference** — [Alexander Sannikov: Radiance Cascades (2024)](https://radiance-cascades.com/)

#### Voxel Cone Tracing (VXGI)
Voxelises scene radiance into a 3D mip-hierarchy, then traces wide cones through the voxel volume to integrate incoming radiance for diffuse and specular GI.

**Use cases** — real-time GI in engines with voxelisation budget (NVIDIA VXGI, CryEngine); medium-quality specular GI.  
**Key variants** — sparse voxel octree (higher resolution, higher cost); clipmap voxel grid (fixed cost, lower resolution far from camera).  
**Limitations** — voxel resolution limits GI quality; dynamic objects require re-voxelisation; memory cost for large scenes.  
**Reference** — [Crassin et al.: Interactive Indirect Illumination Using Voxel Cone Tracing (2011)](https://research.nvidia.com/publication/2011-09_interactive-indirect-illumination-using-voxel-cone-tracing)

#### ReSTIR GI
Spatiotemporal Reservoir Resampling for Global Illumination: maintains a reservoir of importance-weighted light path samples per pixel, sharing samples spatially and temporally to achieve high-quality indirect illumination with a small per-pixel ray budget.

**Use cases** — real-time path-traced GI on hardware RT GPUs; used in NVIDIA's Real-Time Denoisers pipeline.  
**Limitations** — bias from spatiotemporal reuse when scene changes; requires a denoiser for final output.  
**Reference** — [Bitterli et al.: Spatiotemporal Reservoir Resampling for Real-Time Ray Tracing (SIGGRAPH 2020)](https://research.nvidia.com/publication/2020-07_spatiotemporal-reservoir-resampling-real-time-ray-tracing-dynamic-direct)

#### Lumen (Unreal Engine 5)
Combines screen-space ray tracing (first rays), mesh distance fields (medium-range rays), and surface cache probes for far-field GI; falls back to hardware ray tracing for high-quality mode.

**Use cases** — fully dynamic GI in Unreal Engine 5 for consoles and PC.  
**Reference** — [Advances in Real-Time Rendering 2022: Lumen](https://advances.realtimerendering.com/s2022/)

#### Neural Radiance Caching
Trains a compact MLP online per frame to predict incoming radiance at query points, seeded by a small number of path-traced rays; queries the network for all secondary bounce paths.

**Use cases** — dramatically reduces sample count needed for convergent path tracing on GPUs; used in NVIDIA's NRC for game rendering.  
**Reference** — [Müller et al.: Real-Time Neural Radiance Caching for Path Tracing (SIGGRAPH 2021)](https://research.nvidia.com/publication/2021-06_real-time-neural-radiance-caching-path-tracing)

---

### Image-Based Lighting

Image-based lighting (IBL) uses a captured or procedurally generated HDR environment map as an omnidirectional light source, replacing dozens of analytical lights with a single high-frequency representation of the full lighting environment. The PBR split-sum approximation (Epic, 2013) separates the irradiance integral into two pre-integrated lookups — a prefiltered specular mip chain (one per roughness level) and a 2D BRDF LUT — plus a diffuse irradiance convolution (or SH projection), making real-time IBL feasible with two texture samples per fragment. These three artefacts together constitute the "IBL bake" present in every modern game, archviz tool, and glTF viewer.

**Shader stage**: The IBL precomputation (irradiance convolution, prefiltered mip generation, BRDF LUT bake) runs in **compute shaders** offline or at load time. At runtime, the IBL contribution is evaluated in the **fragment shader**: a `textureLod()` call into the prefiltered cubemap plus a `texture()` call into the BRDF LUT, combined per the split-sum formula.

#### Diffuse Irradiance Convolution
Integrates incoming radiance from an HDR environment map over the hemisphere to produce a blurred irradiance map (or SH coefficients) used for diffuse IBL.

**Use cases** — diffuse environmental lighting for all PBR workflows; SH projection enables real-time evaluation without a texture lookup.  
**Reference** — [LearnOpenGL: Diffuse Irradiance](https://learnopengl.com/PBR/IBL/Diffuse-irradiance)

#### Specular Prefiltered Environment Map
Pre-convolves the HDR environment at multiple roughness levels (stored in mip levels) using importance-sampled GGX, enabling plausible glossy reflections at runtime with a single texture sample.

**Use cases** — specular IBL in all PBR engines; the "environment map" visible in reflective surfaces.  
**Limitations** — pre-integration assumes the dominant reflection direction is the view-reflection vector (incorrect for very rough surfaces at grazing angles).  
**Reference** — [Karis: Real Shading in Unreal Engine 4 (SIGGRAPH 2013)](https://blog.selfshadow.com/publications/s2013-shading-course/)

#### Split-Sum BRDF LUT
A 2D lookup table (NdotV × roughness) that stores the scale and bias factors of the Schlick-Fresnel-integrated GGX specular BRDF, completing the split-sum approximation for specular IBL.

**Use cases** — required by all engines using the Epic split-sum approximation for PBR IBL; baked once, reused for all materials.  
**Reference** — [Karis 2013 Unreal SIGGRAPH notes](https://blog.selfshadow.com/publications/s2013-shading-course/) | [Khronos glTF IBL Sampler](https://github.com/KhronosGroup/glTF-IBL-Sampler)

#### Parallax-Corrected Reflection Captures
Adjusts the IBL fetch direction to account for the finite distance of the local environment (box or sphere proxy geometry), reducing the appearance of "floating" reflections indoors.

**Use cases** — indoor scenes, corridors, rooms — any confined space where the environment map should appear at a specific distance.  
**Reference** — [GPU Gems: Environment Mapping Techniques](https://developer.nvidia.com/gpugems/gpugems/part-iii-materials/chapter-17-ambient-occlusionmicrogeometry)

---

### Dynamic Environment Map Capture

The IBL section (§7) covers consuming pre-captured environment maps. This section covers the capture pipeline itself: rendering the scene from a probe position into all 6 faces of a cubemap, generating specular pre-filtered mipmaps, and managing probe update scheduling for runtime use.

**Shader stage**: the capture render uses the standard scene pipeline with a **90° FOV perspective view matrix** rotated for each face. Mipmap generation runs in **compute**. The probe blend and lookup use the IBL fragment shader patterns from §7.

#### Six-Face Capture
Render the scene 6 times from the probe origin, each time with a view matrix targeting one cube face (±X, ±Y, ±Z) and a 90° square FOV projection. Write into a `VkImageViewType::CUBE` render target. A geometry shader multi-view variant can render all 6 faces in one pass by layering output (`gl_Layer`).

```glsl
// Geometry shader — single-pass cube face selection
layout(triangles) in;
layout(triangle_strip, max_vertices=18) out;  // 6 faces × 3 vertices

uniform mat4 face_proj_views[6];   // 6 view-projection matrices
layout(location=0) in  vec3 v_world_pos[];
layout(location=0) out vec3 g_world_pos;
layout(location=1) out flat int g_face;

void main() {
    for (int face = 0; face < 6; ++face) {
        gl_Layer = face;
        g_face   = face;
        for (int v = 0; v < 3; ++v) {
            g_world_pos = v_world_pos[v];
            gl_Position = face_proj_views[face] * vec4(v_world_pos[v], 1.0);
            EmitVertex();
        }
        EndPrimitive();
    }
}
```

#### Specular Pre-Filtering (Runtime)
After capture, generate the specular pre-filtered mip pyramid for use with the split-sum IBL (§7). Each mip level corresponds to a roughness value; for each mip, importance-sample the captured radiance using the GGX VNDF (§29) with the appropriate roughness.

```glsl
// Compute shader — pre-filter captured cubemap mip level for roughness r
layout(set=0, binding=0) uniform samplerCube env_capture;
layout(set=0, binding=1, rgba16f) uniform imageCube env_prefilter;

uniform float roughness;   // 0.0 = mirror, 1.0 = fully diffuse
uniform int   num_samples; // typically 1024

void main() {
    ivec3 px       = ivec3(gl_GlobalInvocationID);
    vec2  uv       = (vec2(px.xy) + 0.5) / vec2(imageSize(env_prefilter).xy);
    vec3  R        = cube_uv_to_dir(uv, px.z);  // reflection direction for this texel
    vec3  N        = R;                           // assume N = V = R (split-sum approx)
    vec3  prefiltered = vec3(0.0);
    float total_weight = 0.0;
    for (int i = 0; i < num_samples; ++i) {
        vec2  xi  = hammersley(i, num_samples);
        vec3  H   = sample_GGX_VNDF(R, roughness*roughness, roughness*roughness, xi.x, xi.y);
        vec3  L   = reflect(-R, H);
        float NdL = max(0.0, dot(N, L));
        if (NdL > 0.0) {
            prefiltered  += texture(env_capture, L).rgb * NdL;
            total_weight += NdL;
        }
    }
    imageStore(env_prefilter, px, vec4(prefiltered / max(total_weight, 1e-4), 1.0));
}
```

#### Probe Update Scheduling
Rendering 6 faces × full scene per probe per frame is prohibitive. Production strategies:

- **Dirty-flag update**: only re-capture a probe when dynamic geometry within its influence radius has moved (animated characters, opening doors).
- **Interleaved face update**: update one face per frame (6-frame full refresh), suitable for slowly changing environments.
- **Temporal blending**: blend the new capture with the previous result using a slow alpha (e.g., α = 0.05 per frame) to avoid pop.

**Use cases** — car paint reflections (interior/exterior environment), glossy office floors, wet streets, any scene requiring high-quality specular reflections beyond SSR and DDGI.  
**Reference** — [Lagarde & de Rousiers: Moving Frostbite to PBR — Reflection Captures (SIGGRAPH 2014)](https://seblagarde.wordpress.com/2015/07/14/siggraph-2014-moving-frostbite-to-physically-based-rendering/)

---

### Lightmapping and Baked GI

Baked lighting pre-computes the irradiance contribution of static lights onto static geometry, storing results in UV-mapped textures (lightmaps). The runtime shader reads the lightmap and combines it with dynamic direct lighting, enabling high-quality indirect illumination at near-zero runtime cost. Lightmapping is the dominant static GI technique in games, archviz, and VR where ray-budget limits dynamic GI quality.

**Shader stage**: Baking runs as a specialised GPU render: texel-space path tracing dispatched from a **compute shader** or via a specialised rasterisation pipeline. Runtime lightmap sampling runs in the **fragment shader** — typically just a `texture()` call.

#### Texel-Space Path Tracing
For each lightmap texel, reconstructs the world-space surface position and normal from the UV layout, then traces N hemisphere rays using the TLAS and accumulates radiance. The output is a noisy irradiance map that is subsequently denoised.

**Use cases** — the standard GPU lightmap baking algorithm; used in Blender's Cycles bake, Unity's GPU Lightmapper, UE5's GPU Lightmass.  
**Reference** — [UE5 GPU Lightmass](https://docs.unrealengine.com/5.0/en-US/gpu-lightmass-global-illumination-in-unreal-engine/)

#### Lightmap Denoising
Applies a spatial/temporal denoiser (Intel OIDN, NVIDIA OptiX Denoiser) to the noisy path-traced output to produce a clean irradiance map at low sample counts (16–64 spp rather than 4096).

**Use cases** — required for any production lightmap pipeline; reduces bake time by 10–100× at equivalent visual quality.  
**Reference** — [Intel Open Image Denoise](https://www.openimagedenoise.org/)

#### Runtime Lightmap Sampling
The fragment shader samples the baked irradiance at lightmap UV2 coordinates and multiplies by the material albedo to produce the indirect diffuse contribution.

```glsl
layout(set=1, binding=0) uniform sampler2D lightmap;
layout(set=1, binding=1) uniform sampler2D directional_lm;  // HL2 dominant direction

layout(location=2) in vec2 v_lm_uv;

// In the fragment shader:
vec3 irradiance = texture(lightmap, v_lm_uv).rgb;
// Directional lightmap: modulate by NdotL with baked dominant direction
vec3 dominant_dir = texture(directional_lm, v_lm_uv).rgb * 2.0 - 1.0;
irradiance *= max(0.5, dot(world_normal, dominant_dir));
vec3 indirect_diffuse = albedo * irradiance;
```

**Use cases** — all static geometry in lightmapped scenes; the baseline runtime cost of baked GI is one texture fetch.

#### Directional Lightmaps (HL2 Basis)
Stores the baked irradiance projected onto three or four basis directions (the HL2 half-life 2 basis: three hemispheric lobes), enabling normal maps to respond to baked indirect lighting with correct self-shadowing.

**Use cases** — whenever surface normal detail must be preserved in lightmapped scenes; the standard in Source engine, Unity, and UE5 lightmass.  
**Reference** — [Valve: Half-Life 2 Shading (SIGGRAPH 2004)](https://advances.realtimerendering.com/s2004/)

---

### Stochastic Lightmap Baking via GPU Path Tracing

Static scene lighting is baked offline into lightmap textures that are read at no runtime shading cost. Traditional lightmap bakers (Autodesk Beast, Enlighten, Lumen static) use hemisphere sampling or form factors; GPU path-tracing bakers (Blender Cycles on GPU, UE5's GPU Lightmass) use Monte Carlo path tracing from each lightmap texel, accumulating samples across multiple frames until convergence. This section covers the per-texel accumulation loop and the denoising strategy that makes fast GPU baking practical.

**Shader stage**: path tracing runs in **ray generation + closest-hit shaders**. Denoising runs in **compute**. The final result is written to a persistent `RGBA32F` lightmap texture accumulated over N frames.

#### Per-Texel Accumulation Loop
Each texel in the lightmap corresponds to a world-space position and normal (recovered from the mesh's UV parameterisation). A path is traced per sample, accumulating irradiance. Samples are accumulated over many frames using running average (Welford online algorithm for variance tracking).

```glsl
// Lightmap baking rgen shader — one ray per lightmap texel per frame
layout(set=0, binding=0) uniform accelerationStructureEXT tlas;
layout(set=0, binding=1) uniform sampler2D  position_atlas;  // world-space XYZ per texel
layout(set=0, binding=2) uniform sampler2D  normal_atlas;    // world-space normal per texel
layout(set=0, binding=3, rgba32f) uniform image2D accum_irr; // running sum
layout(set=0, binding=4, r32ui)   uniform uimage2D sample_count; // samples so far

uniform uint frame_seed;   // changes each frame for temporal stratification

layout(location=0) rayPayloadEXT PathPayload payload;

void main() {
    ivec2 px  = ivec2(gl_LaunchIDEXT.xy);
    vec3  P   = texelFetch(position_atlas, px, 0).xyz;
    vec3  N   = normalize(texelFetch(normal_atlas, px, 0).xyz);
    if (length(P) < 1e-6) return;   // unoccupied texel

    uint  rng = pcg_init(uvec2(px), frame_seed);
    vec3  L   = cosine_hemisphere_sample(N, pcg_float(rng), pcg_float(rng));

    payload = init_path_payload(P + N * 0.001, L, rng);
    // Trace k-bounce path (rchit shader handles bounces via payload recursion)
    traceRayEXT(tlas, gl_RayFlagsNoneEXT, 0xFF, 0, 0, 0,
                P + N * 0.001, 0.001, L, 1e27, 0);

    vec3  sample_irr = payload.radiance;

    // Online Welford accumulation
    uint  n_prev = imageLoad(sample_count, px).r;
    vec4  prev   = imageLoad(accum_irr, px);
    vec3  accum  = prev.rgb + (sample_irr - prev.rgb) / float(n_prev + 1u);
    imageStore(accum_irr,    px, vec4(accum, 1.0));
    imageStore(sample_count, px, uvec4(n_prev + 1u));
}
```

#### Lightmap Denoising (OIDN via Compute)
After accumulating enough samples (or as a real-time preview after just 8–16 samples), apply Intel Open Image Denoise (OIDN) or a SVGF-style À-Trous filter (§40) to the raw lightmap. OIDN is a neural denoiser that accepts the noisy colour, optional surface normal, and albedo as auxiliary buffers.

```c
// CPU side: launch OIDN on the GPU-rendered lightmap
OIDNDevice device = oidnNewDevice(OIDN_DEVICE_TYPE_GPU);
oidnCommitDevice(device);
OIDNFilter filter = oidnNewFilter(device, "RTLightmap");
oidnSetFilterImage(filter, "color",  gpu_lightmap_ptr, OIDN_FORMAT_FLOAT3, w, h, 0, 0, 0);
oidnSetFilterImage(filter, "output", denoised_ptr,     OIDN_FORMAT_FLOAT3, w, h, 0, 0, 0);
oidnSetFilter1b(filter, "hdr", true);
oidnCommitFilter(filter);
oidnExecuteFilter(filter);
```

#### UV Chart Seam Handling
Lightmap charts (UV islands) have seams at their boundaries. After baking and before denoising, dilate the border texels of each chart outward by 2–4 pixels to prevent black seams visible at the boundary between charts when bilinear filtering reads across the padding.

```glsl
// Seam dilation compute — expand valid texels outward into empty padding
// One thread per texel; if this texel is empty and any neighbour is valid, copy nearest valid
layout(local_size_x=8, local_size_y=8) in;
layout(set=0, binding=0, rgba32f) uniform image2D lightmap;
layout(set=0, binding=1, r8ui)    uniform uimage2D valid_mask;  // 1 = baked, 0 = padding

void main() {
    ivec2 px = ivec2(gl_GlobalInvocationID.xy);
    if (imageLoad(valid_mask, px).r == 1u) return;  // already valid
    // Search 4-connected neighbours
    const ivec2 DIRS[4] = ivec2[](ivec2(1,0),ivec2(-1,0),ivec2(0,1),ivec2(0,-1));
    for (int d = 0; d < 4; ++d) {
        ivec2 nb = px + DIRS[d];
        if (imageLoad(valid_mask, nb).r == 1u) {
            imageStore(lightmap, px, imageLoad(lightmap, nb));
            return;
        }
    }
}
```

**Use cases** — arch-viz static GI; game levels with static geometry; any scenario where pre-baked lighting quality needs to exceed what real-time GI can deliver.  
**Reference** — [UE5 GPU Lightmass documentation](https://docs.unrealengine.com/5.0/en-US/gpu-lightmass-global-illumination-in-unreal-engine/); [Intel OIDN](https://www.openimagedenoise.org/); [Christensen & Jarosz: The Path to Path-Traced Movies (2016)](https://dl.acm.org/doi/10.1145/2988458.2988459)

---

### Light Propagation Volumes

Light Propagation Volumes (LPV) is CryEngine's 2009 real-time GI technique: inject light samples from a Reflective Shadow Map (RSM) into a 3D grid of SH probes, propagate the SH radiance through the grid over several iterations, then evaluate the accumulated radiance at each surface point for indirect diffuse lighting. LPV runs on any GPU without ray tracing hardware and at lower cost than DDGI (no per-probe ray tracing), at the cost of lower quality (light leaks through thin walls, no specular GI).

**Shader stage**: RSM injection runs in **compute**. Propagation runs in **compute** (one pass per grid cell per iteration). Evaluation runs in the **fragment shader** by sampling the 3D SH grid texture.

#### Reflective Shadow Map (RSM) Generation
An RSM is a shadow map augmented with additional MRTs: world-space position, world-space normal, and reflected flux (albedo × incoming irradiance) at each visible lit texel. It captures the first-bounce indirect light sources.

```glsl
// RSM generation — additional fragment outputs beyond depth
layout(location=0) out vec3 rsm_world_pos;
layout(location=1) out vec3 rsm_world_normal;
layout(location=2) out vec3 rsm_flux;     // albedo × direct irradiance

void main() {
    rsm_world_pos    = v_world_pos;
    rsm_world_normal = normalize(v_world_normal);
    // Flux: albedo × NdotL × light_power / (num_rsm_samples)
    float NdotL  = max(0.0, dot(rsm_world_normal, light_dir));
    rsm_flux     = albedo * NdotL * light_color * rsm_texel_area;
}
```

#### LPV Injection (RSM → Grid)
For each RSM texel, find the grid cell containing its world position and accumulate the texel's flux as an SH lobe (oriented in the RSM normal direction) into the cell's SH coefficients.

```glsl
// LPV injection compute — one thread per RSM texel sample
layout(set=0, binding=0) uniform sampler2D rsm_pos;
layout(set=0, binding=1) uniform sampler2D rsm_normal;
layout(set=0, binding=2) uniform sampler2D rsm_flux;
layout(set=0, binding=3, rgba16f) uniform image3D lpv_r;  // SH coefficients for R
layout(set=0, binding=4, rgba16f) uniform image3D lpv_g;  // SH coefficients for G
layout(set=0, binding=5, rgba16f) uniform image3D lpv_b;  // SH coefficients for B

uniform vec3  grid_min;
uniform vec3  grid_cell_size;
uniform ivec3 grid_res;

void main() {
    ivec2 rsm_px = ivec2(gl_GlobalInvocationID.xy);
    vec3  P      = texelFetch(rsm_pos,    rsm_px, 0).xyz;
    vec3  N      = texelFetch(rsm_normal, rsm_px, 0).xyz;
    vec3  flux   = texelFetch(rsm_flux,   rsm_px, 0).rgb;

    ivec3 cell   = ivec3((P - grid_min) / grid_cell_size);
    if (any(lessThan(cell, ivec3(0))) || any(greaterThanEqual(cell, grid_res))) return;

    // Project flux into SH2 lobe oriented along N
    float Y[9]; sh_basis_L2(N, Y);
    vec4  sh_r = vec4(flux.r * Y[0], flux.r * Y[1], flux.r * Y[2], flux.r * Y[3]);
    vec4  sh_g = vec4(flux.g * Y[0], flux.g * Y[1], flux.g * Y[2], flux.g * Y[3]);
    vec4  sh_b = vec4(flux.b * Y[0], flux.b * Y[1], flux.b * Y[2], flux.b * Y[3]);

    // Atomic add into 3D SH grid (imageAtomicAdd for float requires extension)
    // In practice: accumulate per-cell with subgroup operations or two-pass reduce
    imageStore(lpv_r, cell, imageLoad(lpv_r, cell) + sh_r);
    imageStore(lpv_g, cell, imageLoad(lpv_g, cell) + sh_g);
    imageStore(lpv_b, cell, imageLoad(lpv_b, cell) + sh_b);
}
```

#### LPV Propagation
Run 4–8 propagation iterations. Each iteration, every cell distributes its SH radiance to its 6 face-neighbours through a solid-angle-weighted transfer, approximating how radiance would propagate through the diffuse volume.

```glsl
// LPV propagation — one pass, one compute thread per cell
layout(set=0, binding=0) uniform sampler3D lpv_in_r;
layout(set=0, binding=3, rgba16f) uniform image3D lpv_out_r;
// ... similarly for G and B channels

const ivec3 FACES[6] = ivec3[](
    ivec3(1,0,0), ivec3(-1,0,0), ivec3(0,1,0),
    ivec3(0,-1,0), ivec3(0,0,1), ivec3(0,0,-1)
);

void main() {
    ivec3 cell   = ivec3(gl_GlobalInvocationID);
    vec4  accum  = vec4(0.0);
    for (int f = 0; f < 6; ++f) {
        ivec3 nb     = cell - FACES[f];  // neighbour sending to us
        vec4  nb_sh  = texelFetch(lpv_in_r, nb, 0);
        // Evaluate SH of nb in direction FACES[f] (outgoing from nb toward us)
        vec3  face_dir = vec3(FACES[f]);
        float Y[9]; sh_basis_L2(face_dir, Y);
        float contrib = max(0.0, nb_sh.x*Y[0] + nb_sh.y*Y[1] + nb_sh.z*Y[2] + nb_sh.w*Y[3]);
        // Re-project onto SH lobe facing the incoming direction
        vec3 in_dir = -face_dir;
        float Yin[9]; sh_basis_L2(in_dir, Yin);
        accum += contrib * vec4(Yin[0], Yin[1], Yin[2], Yin[3]);
    }
    imageStore(lpv_out_r, cell, imageLoad(lpv_out_r, cell) + accum * 0.25);
}
```

#### LPV Evaluation
Sample the 3D SH grid at the world-space fragment position, evaluate the SH irradiance in the surface normal direction.

```glsl
// LPV evaluation — fragment shader
layout(set=1, binding=0) uniform sampler3D lpv_r;
layout(set=1, binding=1) uniform sampler3D lpv_g;
layout(set=1, binding=2) uniform sampler3D lpv_b;

vec3 lpv_indirect(vec3 P, vec3 N) {
    vec3 uvw    = (P - grid_min) / (grid_res * grid_cell_size);
    vec4 sh_r   = texture(lpv_r, uvw);
    vec4 sh_g   = texture(lpv_g, uvw);
    vec4 sh_b   = texture(lpv_b, uvw);
    float Y[9]; sh_basis_L2(N, Y);
    float basis = Y[0]*1.0 + Y[1]*1.0 + Y[2]*1.0 + Y[3]*1.0; // L1 only (4 coefficients)
    return max(vec3(0.0), vec3(dot(sh_r, vec4(Y[0],Y[1],Y[2],Y[3])),
                                dot(sh_g, vec4(Y[0],Y[1],Y[2],Y[3])),
                                dot(sh_b, vec4(Y[0],Y[1],Y[2],Y[3]))));
}
```

**Use cases** — mid-range hardware GI where DDGI is too expensive; dynamic GI for games without RT hardware; Crysis 2/3, Far Cry 3/4, Ryse: Son of Rome.  
**Reference** — [Kaplanyan & Dachsbacher: Cascaded Light Propagation Volumes for Real-Time Indirect Illumination (I3D 2010)](https://dl.acm.org/doi/10.1145/1730804.1730821)

---

### Screen-Space Global Illumination (SSGI)

SSGI extends SSAO's per-pixel hemisphere tracing to also read the *colour* at the traced hit point, accumulating indirect diffuse irradiance instead of just occlusion. The result is a per-pixel indirect light colour that captures colour bleeding and single-bounce GI without probes or ray tracing hardware. It is used in UE5 as a fast GI fallback ("Lumen Screen Space") and in many mid-budget games where DDGI or ReSTIR GI is too expensive.

**Shader stage**: SSGI runs as a **compute or full-screen fragment shader** reading the depth buffer, normal buffer, and HDR colour buffer from the previous frame (or the current frame's early-Z pass for normal/depth).

**Difference from SSAO**: SSAO writes a scalar occlusion factor; SSGI writes a `vec3` indirect irradiance colour. The sampling loop is the same structure but accumulates `colour × NdotL` instead of `NdotL` alone.

```glsl
// SSGI compute shader — one thread per pixel
layout(local_size_x=8, local_size_y=8) in;

layout(set=0, binding=0) uniform sampler2D g_depth;
layout(set=0, binding=1) uniform sampler2D g_normal;     // world-space normals
layout(set=0, binding=2) uniform sampler2D hdr_color;    // previous or current frame colour
layout(set=0, binding=3, rgba16f) uniform image2D ssgi_out;

layout(set=0, binding=4) uniform sampler2D blue_noise;   // 64×64 blue noise

uniform mat4  proj_view;
uniform mat4  inv_proj_view;
uniform vec2  resolution;
uniform uint  frame_index;
uniform float max_distance;   // world-space max trace distance (e.g. 2.0 m)
uniform int   num_samples;    // typically 4–8

vec3 reconstruct_world(ivec2 px) {
    vec2  uv  = (vec2(px) + 0.5) / resolution;
    float d   = texelFetch(g_depth, px, 0).r;
    vec4  h   = inv_proj_view * vec4(uv * 2.0 - 1.0, d, 1.0);
    return h.xyz / h.w;
}

vec2 world_to_uv(vec3 p) {
    vec4 clip = proj_view * vec4(p, 1.0);
    return clip.xy / clip.w * 0.5 + 0.5;
}

void main() {
    ivec2 px     = ivec2(gl_GlobalInvocationID.xy);
    vec3  P      = reconstruct_world(px);
    vec3  N      = texelFetch(g_normal, px, 0).xyz;

    // Blue noise base sample — temporally decorrelated
    float bn     = texelFetch(blue_noise, px % 64, 0).r;
    bn           = fract(bn + float(frame_index) * 0.61803398875);

    vec3 indirect = vec3(0.0);
    float weight  = 0.0;

    for (int i = 0; i < num_samples; ++i) {
        // Cosine-weighted hemisphere sample (Malley's method)
        float u1  = fract(bn + float(i) / float(num_samples));
        float u2  = fract(bn * 1.618 + float(i) * 0.382);
        float phi = 2.0 * 3.14159 * u1;
        float r   = sqrt(u2);
        vec3  dir = vec3(cos(phi)*r, sin(phi)*r, sqrt(1.0 - u2)); // hemisphere local
        // Transform to world space using N as up
        vec3  T   = normalize(abs(N.x) > 0.9 ? cross(N, vec3(0,1,0)) : cross(N, vec3(1,0,0)));
        vec3  B   = cross(N, T);
        vec3  L   = T*dir.x + B*dir.y + N*dir.z;

        // Step along ray in screen space — ray march 8 steps
        vec3  hit  = P;
        bool  found = false;
        for (int s = 1; s <= 8; ++s) {
            hit = P + L * (max_distance * float(s) / 8.0);
            vec2 uv  = world_to_uv(hit);
            if (any(lessThan(uv, vec2(0.0))) || any(greaterThan(uv, vec2(1.0)))) break;
            float scene_d = texture(g_depth, uv).r;
            vec4  scene_h = inv_proj_view * vec4(uv*2.0-1.0, scene_d, 1.0);
            vec3  scene_p = scene_h.xyz / scene_h.w;
            float hit_d   = length(hit - P);
            float scene_hit_d = length(scene_p - P);
            if (hit_d > scene_hit_d && hit_d - scene_hit_d < 0.3) {
                // Hit! Sample colour at this screen location
                vec3 col  = texture(hdr_color, uv).rgb;
                float ndotl = max(0.0, dot(N, L));
                indirect += col * ndotl;
                weight   += ndotl;
                found = true;
                break;
            }
        }
        if (!found) weight += max(0.0, dot(N, L));
    }

    indirect = (weight > 0.0) ? indirect / weight : vec3(0.0);
    imageStore(ssgi_out, px, vec4(indirect, 1.0));
}
```

**Temporal accumulation**: the SSGI output is temporally accumulated with TAA-style reprojection (using motion vectors, §48) to reduce noise at 4–8 samples/pixel.

**Use cases** — UE5 Lumen Screen Space mode, HDRP Screen Space GI (Unity), mid-budget deferred engines; provides colour bleeding in corners, under overhangs, between coloured walls.  
**Limitations** — screen-space: off-screen surfaces contribute no indirect light; breaks in disoccluded regions; limited trace distance.  
**Reference** — [UE5: Lumen Technical Details](https://docs.unrealengine.com/5.0/en-US/lumen-technical-details-in-unreal-engine/); [Unity HDRP: Screen Space Global Illumination](https://docs.unity3d.com/Packages/com.unity.render-pipelines.high-definition@latest)

---

### World-Space Radiance Cache

The **World-Space Radiance Cache (WSRC)** is the GI architecture at the core of Unreal Engine 5's **Lumen** GI system: a sparse set of probes distributed through the world, each storing SH2 (or SH3) radiance computed by shooting short screen-space and hardware-traced rays, updated asynchronously and filtered spatially and temporally. WSRC extends screen-space GI (§50) to surfaces not visible to the camera and provides stable indirect lighting for large open worlds where DDGI probe count would be prohibitive.

The broader concept also describes other world-space probe caches: **DDGI** (Dynamic Diffuse GI, Majercik et al. 2019) stores per-probe irradiance and visibility in 2D octahedral maps and traces rays from probes; **Radiance Caching** (Müller et al. 2021) stores per-vertex or per-cell 5D radiance for path guiding. This section covers the algorithmic pattern common to all of them.

**Shader stage**: probe ray tracing runs as a **ray generation shader**. Probe update (irradiance SH projection) and spatial filtering run in **compute**. Probe evaluation (interpolation at surface points) runs in the **fragment or compute** lighting pass.

#### Probe Placement and Ray Tracing
Probes are placed on a world-space grid (or sparse voxel structure); each frame a subset are updated. Each probe shoots N rays (typically 64–256) uniformly distributed over the hemisphere (or full sphere), traces them via TLAS, and accumulates radiance. For Lumen, the rays are short (~3 m) and terminate against screen-space geometry first, falling back to hardware RT or SSDF for longer distances.

```glsl
// Probe ray generation shader — DDGI-style
layout(set=0, binding=0) uniform accelerationStructureEXT tlas;
layout(set=0, binding=1, rgba16f) uniform image2D probe_radiance; // N_probes × N_rays
layout(set=0, binding=2, rg16f)   uniform image2D probe_visibility;// depth+depth²

uniform int    rays_per_probe;   // e.g. 64
uniform vec3   grid_origin;
uniform vec3   grid_spacing;
uniform ivec3  grid_dims;
uniform mat3   random_rotation;  // changes each frame (temporal stratification)

layout(location=0) rayPayloadEXT vec4 ray_payload;  // .rgb = radiance, .a = hit_dist

void main() {
    ivec2 dispatch_id = ivec2(gl_LaunchIDEXT.xy);
    int   probe_idx   = dispatch_id.x;
    int   ray_idx     = dispatch_id.y;

    ivec3 probe_coords = ivec3(probe_idx % grid_dims.x,
                               (probe_idx / grid_dims.x) % grid_dims.y,
                               probe_idx / (grid_dims.x * grid_dims.y));
    vec3  probe_pos    = grid_origin + vec3(probe_coords) * grid_spacing;

    // Stratified spherical ray direction (Fibonacci + random rotation)
    vec3  ray_dir  = random_rotation * spherical_fibonacci(ray_idx, rays_per_probe);

    ray_payload = vec4(0.0, 0.0, 0.0, 1e27);
    traceRayEXT(tlas, gl_RayFlagsNoneEXT, 0xFF, 0, 0, 0,
        probe_pos, 0.001, ray_dir, max_ray_dist, 0);

    imageStore(probe_radiance,   ivec2(probe_idx, ray_idx), ray_payload);
    imageStore(probe_visibility, ivec2(probe_idx, ray_idx),
               vec4(ray_payload.a, ray_payload.a * ray_payload.a, 0.0, 1.0));
}
```

#### Irradiance Update (Rays → SH)
Project the N ray samples into SH2 coefficients for each probe. Run as a compute shader with one workgroup per probe, each lane processing one ray.

```glsl
// Probe irradiance update — project ray samples into SH2
layout(local_size_x=64) in;  // one thread per ray
layout(set=0, binding=0) readonly  uniform image2D probe_radiance;
layout(set=0, binding=1) buffer    ProbesSH { vec4 probes_sh[][9]; };  // L0+L1+L2 = 9 coeffs

shared vec4 s_sh[9][64];  // per-thread partial SH accumulation

void main() {
    int probe_idx = int(gl_WorkGroupID.x);
    int ray_idx   = int(gl_LocalInvocationIndex);

    vec3 L  = imageLoad(probe_radiance, ivec2(probe_idx, ray_idx)).rgb;
    vec3 dir = spherical_fibonacci(ray_idx, 64);

    float Y[9]; sh_basis_L2(dir, Y);
    for (int b = 0; b < 9; ++b)
        s_sh[b][ray_idx] = vec4(L * Y[b], 0.0);
    barrier();

    // Parallel reduction
    for (uint stride = 32u; stride > 0u; stride >>= 1u) {
        if (ray_idx < int(stride))
            for (int b = 0; b < 9; ++b)
                s_sh[b][ray_idx] += s_sh[b][ray_idx + stride];
        barrier();
    }

    if (ray_idx == 0) {
        float normalization = 4.0 * 3.14159 / 64.0;
        // Temporal blend: mix new SH with existing (hysteresis ~0.97)
        for (int b = 0; b < 9; ++b)
            probes_sh[probe_idx][b] = mix(s_sh[b][0] * normalization,
                                          probes_sh[probe_idx][b], 0.97);
    }
}
```

#### Probe Interpolation at Surface (Fragment)
At the lit surface, find the 8 surrounding probes, weight by distance and visibility (to avoid light leaking through thin walls using Chebyshev visibility), accumulate SH irradiance.

```glsl
// Fragment: evaluate world-space radiance cache
vec3 wsrc_irradiance(vec3 P, vec3 N, vec3 V) {
    ivec3 probe_coord = ivec3((P - grid_origin) / grid_spacing);
    vec3  blend       = fract((P - grid_origin) / grid_spacing);
    vec3  irr         = vec3(0.0);
    float total_w     = 0.0;

    // Trilinear interpolation over 8 surrounding probes
    for (int ox = 0; ox <= 1; ++ox) for (int oy = 0; oy <= 1; ++oy) for (int oz = 0; oz <= 1; ++oz) {
        ivec3 c    = clamp(probe_coord + ivec3(ox, oy, oz), ivec3(0), grid_dims - 1);
        int   pidx = c.x + c.y * grid_dims.x + c.z * grid_dims.x * grid_dims.y;

        // Trilinear weight
        float w = mix(1.0-blend.x, blend.x, float(ox))
                * mix(1.0-blend.y, blend.y, float(oy))
                * mix(1.0-blend.z, blend.z, float(oz));

        // Visibility weight: Chebyshev test using stored depth moments
        vec3  probe_pos = grid_origin + vec3(c) * grid_spacing;
        float dist      = length(P - probe_pos);
        vec2  vis       = texelFetch(probe_vis_tex, ivec2(pidx, 0), 0).rg;
        float var       = abs(vis.y - vis.x * vis.x);
        float vis_w     = (dist <= vis.x) ? 1.0
                        : var / (var + (dist - vis.x) * (dist - vis.x));
        w *= max(vis_w * vis_w * vis_w, 0.0001);

        // Evaluate SH irradiance in normal direction
        float Y[9]; sh_basis_L2(N, Y);
        vec3  probe_irr = vec3(0.0);
        for (int b = 0; b < 9; ++b)
            probe_irr += probes_sh[pidx][b].rgb * Y[b];
        irr     += max(probe_irr, vec3(0.0)) * w;
        total_w += w;
    }
    return irr / max(total_w, 1e-5);
}
```

**Use cases** — Unreal Engine 5 Lumen (WSRC + SSGI + hardware RT fallback); DDGI (id Software, Wolfenstein II); any open-world dynamic GI where screen-space techniques alone are insufficient.  
**Reference** — [Majercik et al.: Dynamic Diffuse Global Illumination with Ray-Traced Irradiance Fields (JCGT 2019)](https://jcgt.org/published/0008/02/01/); [Lumen Technical Details (UE5 Docs)](https://docs.unrealengine.com/5.0/en-US/lumen-technical-details-in-unreal-engine/)

---

### Spherical Harmonics Lighting

Spherical Harmonics (SH) are a set of orthonormal basis functions defined on the unit sphere, analogous to the Fourier basis on the circle. Any spherical function — an environment map, the irradiance at a surface point, the visibility from a probe — can be projected into SH coefficients and reconstructed at runtime with a dot product. Order-2 SH (9 coefficients per colour channel = 27 floats) captures all low-frequency lighting including directional light, sky gradient, and diffuse colour bleeding, at the cost of 27 multiply-adds per pixel.

**Why SH matters**: irradiance probes (§6), lightmaps, and character ambient lighting all ultimately store SH coefficients. DDGI probes store irradiance as 2nd-order SH. Directional lightmaps (§26) encode the dominant direction as an SH-motivated basis. Understanding SH is prerequisite to understanding any probe-based GI system.

**Shader stage**: SH projection runs in **compute** (one thread per probe direction sample). SH evaluation runs in the **fragment shader** during lighting, as a simple dot product with the surface normal.

#### SH Basis Functions and Coefficients
The real SH basis functions `Y_l^m(θ, φ)` are grouped by band `l` (0, 1, 2, …). Each band `l` has `2l+1` functions. For rendering, bands 0–2 (9 functions) are sufficient for diffuse irradiance.

```glsl
// Real spherical harmonic basis functions up to l=2 (9 terms)
// Input: normalised direction vector (x, y, z)
// Conventions follow Sloan 2008 (ZH ordering, Condon-Shortley excluded)
void sh_basis_L2(vec3 d, out float Y[9]) {
    // Band 0 (l=0): constant
    Y[0] = 0.282095;                               // 1/2 * sqrt(1/π)

    // Band 1 (l=1): linear
    Y[1] = 0.488603 * d.y;                         // sqrt(3/4π) * y
    Y[2] = 0.488603 * d.z;                         // sqrt(3/4π) * z
    Y[3] = 0.488603 * d.x;                         // sqrt(3/4π) * x

    // Band 2 (l=2): quadratic
    Y[4] = 1.092548 * d.x * d.y;                  // sqrt(15/4π)   * xy
    Y[5] = 1.092548 * d.y * d.z;                  // sqrt(15/4π)   * yz
    Y[6] = 0.315392 * (3.0*d.z*d.z - 1.0);        // 1/4 sqrt(5/π) * (3z²-1)
    Y[7] = 1.092548 * d.x * d.z;                  // sqrt(15/4π)   * xz
    Y[8] = 0.546274 * (d.x*d.x - d.y*d.y);        // sqrt(15/16π)  * (x²-y²)
}
```

#### Projecting a Radiance Environment into SH
Numerically integrates the radiance function `L(ω)` over the sphere, weighted by each SH basis function. For an environment map this samples N directions distributed over the hemisphere (cosine-weighted or uniform).

```glsl
// Compute SH projection in a compute shader — one thread contributes one sample
// Each sample direction d has radiance rgb; accumulate into 9×3 coefficient array
layout(set=0, binding=0) buffer SHCoeffs { vec4 coeffs[9]; }; // vec3 packed as vec4

shared vec4 s_coeffs[9][64];  // per-thread partial sums (64 threads/group)

layout(local_size_x=64) in;
void main() {
    uint  tid  = gl_LocalInvocationID.x;
    uint  gid  = gl_GlobalInvocationID.x;
    vec3  d    = fibonacci_sphere(gid, total_samples);   // equidistributed direction
    vec3  rgb  = sample_env_map(d);
    float pdf  = 1.0 / (4.0 * 3.14159);                 // uniform sphere

    float Y[9];
    sh_basis_L2(d, Y);

    for (int i = 0; i < 9; ++i)
        s_coeffs[i][tid] = vec4(rgb * Y[i] / pdf / float(total_samples), 0.0);

    barrier();
    // Parallel reduction within workgroup
    for (uint stride = 32; stride > 0; stride >>= 1) {
        if (tid < stride)
            for (int i = 0; i < 9; ++i)
                s_coeffs[i][tid] += s_coeffs[i][tid + stride];
        barrier();
    }
    if (tid == 0)
        for (int i = 0; i < 9; ++i)
            atomicAdd_vec4(coeffs[i], s_coeffs[i][0]);  // accumulate globally
}
```

#### SH Irradiance Evaluation
Given 9 SH coefficients (from a probe or lightmap), evaluate the irradiance incident on a surface with normal `N` by a dot product of the coefficients with the SH-projected clamped-cosine kernel (the "ZH × SH convolution" — precomputed scaling constants `A_l`).

```glsl
// Evaluate irradiance from 9 SH coefficients at surface normal N
// Coefficients are pre-convolved with the clamped-cosine kernel (Ramamoorthi 2001)
// so no A_l scaling needed at runtime — just evaluate the SH basis.
uniform vec3 sh[9];   // 9 coefficient vectors (one per colour channel)

vec3 sh_irradiance(vec3 N) {
    float Y[9];
    sh_basis_L2(N, Y);
    vec3 irr = vec3(0.0);
    for (int i = 0; i < 9; ++i) irr += sh[i] * Y[i];
    return max(vec3(0.0), irr);
}
// Alternatively: Sloan's polynomial form (avoids basis function calls)
vec3 sh_irradiance_fast(vec3 N) {
    // Coefficients pre-multiplied by Ramamoorthi-Hanrahan A_l and π
    return max(vec3(0.0),
        sh[0]
      + sh[1]*N.y + sh[2]*N.z + sh[3]*N.x
      + sh[4]*N.x*N.y + sh[5]*N.y*N.z + sh[6]*(3.0*N.z*N.z-1.0)
      + sh[7]*N.x*N.z + sh[8]*(N.x*N.x-N.y*N.y));
}
```

**Use cases** — DDGI probe irradiance storage (27 floats/probe), ambient character lighting from nearest probe, directional lightmap irradiance, sky-colour ambient in outdoor scenes.  
**Reference** — [Ramamoorthi & Hanrahan: An Efficient Representation for Irradiance Environment Maps (SIGGRAPH 2001)](https://dl.acm.org/doi/10.1145/383259.383317); [Sloan: Stupid Spherical Harmonics (GDC 2008)](https://www.ppsloan.org/publications/StupidSH36.pdf)

---

### Spherical Gaussians

Spherical Gaussians (SGs) are an alternative basis to SH for representing directional functions on the sphere. An SG lobe is defined by a centre direction `µ`, a sharpness `λ`, and an amplitude `a`:

`G(d; µ, λ, a) = a · exp(λ · (dot(µ, d) − 1))`

Unlike SH (which is a global polynomial basis, smooth but incapable of sharp features), a single SG can represent an arbitrarily sharp directional lobe. Multiple SGs are summed to approximate an environment. SGs support analytic products, integrals, and BRDF convolution — making them more flexible than SH for glossy and specular GI.

**Shader stage**: SG lighting evaluation runs in the **fragment shader** — one dot product + exp per SG per fragment. SG projection runs in **compute** (gradient descent or EM fitting against a set of sample directions).

#### SG Evaluation and Irradiance Integral
The integral of an SG over the hemisphere (for diffuse irradiance) has a closed form. The product of two SGs (for BRDF × lighting) is another SG.

```glsl
struct SG {
    vec3  amplitude;   // RGB colour weight
    vec3  axis;        // normalised centre direction µ
    float sharpness;   // λ — larger = narrower lobe
};

// Evaluate SG lobe at direction d
vec3 sg_eval(SG sg, vec3 d) {
    return sg.amplitude * exp(sg.sharpness * (dot(sg.axis, d) - 1.0));
}

// Product of two SGs — result is a new SG (exact, up to normalisation)
SG sg_product(SG a, SG b) {
    SG result;
    result.sharpness = a.sharpness + b.sharpness;
    result.axis      = normalize(a.sharpness * a.axis + b.sharpness * b.axis);
    float norm_factor = exp(dot(a.sharpness * a.axis + b.sharpness * b.axis,
                                result.axis) - a.sharpness - b.sharpness);
    result.amplitude = a.amplitude * b.amplitude * norm_factor;
    return result;
}

// Hemispherical integral of SG × clamped cosine (for diffuse irradiance)
// Wang et al. 2009 approximation
vec3 sg_diffuse_irradiance(SG sg, vec3 N) {
    float mu_dot_n = dot(sg.axis, N);
    float c0 = 0.36, c1 = 0.25 / 0.36;
    float eml  = exp(-sg.sharpness);
    float em2l = eml * eml;
    float rl   = 1.0 / sg.sharpness;
    float scale = 1.0 + 2.0 * em2l / 3.0 - eml;
    float b  = clamp(mu_dot_n, -1.0, 1.0);
    float t  = sqrt(1.0 - b*b);
    float d  = (1.0 - eml * (1.0 - 2.0*rl*(1.0 - eml) + rl * rl * (2.0 - em2l)));
    return sg.amplitude * scale * (c0 * t * sin(acos(b)) + (1.0 - c0*t) * b) * d;
}
```

#### SG Environment Lighting
An environment map projected into N=12 SG lobes (a common approximation) can be evaluated per-fragment as the sum of N SG irradiance integrals — far fewer shader instructions than environment map importance sampling.

```glsl
#define NUM_SG 12
uniform SG sg_lights[NUM_SG];

vec3 sg_env_irradiance(vec3 N) {
    vec3 irr = vec3(0.0);
    for (int i = 0; i < NUM_SG; ++i)
        irr += sg_diffuse_irradiance(sg_lights[i], N);
    return irr;
}
```

**Use cases** — Lumen uses SGs for storing radiance in surface cache probes; mobile GI systems use SGs as a specular-capable alternative to SH probes; baked GI with glossy response.  
**Reference** — [Wang et al.: All-Frequency Rendering of Dynamic, Spatially-Varying Reflectance (SIGGRAPH 2009)](https://dl.acm.org/doi/10.1145/1618452.1618479); [Pettineo: Baking and Rendering with Spherical Gaussians (The Danger Zone)](https://therealmjp.github.io/posts/sg-series-part-1-a-brief-and-incomplete-history-of-baked-lighting-representations/)

---


---

## V. Material Models and BRDF

A BRDF (Bidirectional Reflectance Distribution Function) describes how a surface scatters incident light. This category covers the complete material model vocabulary used in production rendering: the GGX-based physically based core, anisotropic extensions for brushed metals and fabrics, microstructure models for glitter and sparkle, thin-film interference for soap bubbles and iridescent paint, and the volumetric sub-surface models required for skin, wax, and leaves. Non-photorealistic rendering is included here because it is also a per-pixel shading decision — one that deliberately relaxes physical constraints for stylistic effect.

### Physically Based Shading — BRDF Models

A BRDF (Bidirectional Reflectance Distribution Function) describes how a surface scatters light as a function of incoming and outgoing direction. Modern real-time PBR pipelines build the shading equation from three independently swappable components: the normal distribution function (NDF, which governs highlight shape), the masking-shadowing term (G, which prevents energy from dark grazing angles), and the Fresnel term (F, which controls the view-angle reflectance balance). GGX + Smith + Schlick is the de facto standard combination, wrapped in the Disney "Principled" parameterisation for artist friendliness. The entries below move from the simplest building blocks (Lambertian, Cook-Torrance) through the individual NDF/G/F components to specialised models for cloth, iridescence, and anisotropy.

**Shader stage**: All BRDF evaluation happens in the **fragment shader** (rasterisation) or in the **ray generation / closest-hit shader** (ray tracing). The BRDF function itself is a pure mathematical evaluation with no geometry dependency; the same GLSL/HLSL/WGSL function body is portable across stages.

#### Lambertian Diffuse
Models perfectly matte diffuse reflection as albedo × max(0, N·L) / π, with no view-dependence.

**Use cases** — base diffuse term in all PBR pipelines; ground truth for matte surfaces.  
**Limitations** — incorrect at grazing angles (no retroreflection, no darkening); use Oren-Nayar for rough surfaces.  
**Reference** — [PBR Book: Reflection Models](https://pbr-book.org/4ed/Reflection_Models)

#### Oren-Nayar Diffuse
Extends Lambertian diffuse with a microfacet roughness parameter σ (standard deviation of facet slopes) to model the retroreflective darkening of rough matte surfaces.

**Use cases** — fabric, unfinished stone, dry soil, rough plaster — any surface that appears brighter when viewed and lit from the same direction.  
**Key variants** — full Oren-Nayar (expensive); Fujii's qualitative approximation (fast, widely used in games).  
**Limitations** — more expensive than Lambert; the σ parameter has no direct physical mapping to roughness in other BRDF models.  
**Reference** — [Oren & Nayar (1994) SIGGRAPH](https://doi.org/10.1145/192161.192213)

#### Cook-Torrance Specular BRDF
The standard microfacet specular BRDF: (D × G × F) / (4 × NdotV × NdotL), where D is the normal distribution, G is the geometry/masking term, and F is the Fresnel factor.

**Use cases** — the dominant specular model in all modern real-time PBR pipelines; the correct physically based alternative to Phong.  
**Key variants** — each of D, G, F is independently swappable; GGX + Smith + Schlick is the industry-standard combination.  
**Reference** — [Cook & Torrance (1982)](https://doi.org/10.1145/357290.357293) | [PBR Book: Microfacet Models](https://pbr-book.org/4ed/Reflection_Models/Roughness_Using_Microfacet_Theory)

#### GGX / Trowbridge-Reitz Normal Distribution (NDF)
Describes the statistical distribution of microfacet surface normals; GGX has a wider, longer-tailed specular highlight than Beckmann, matching measured metals and dielectrics better.

**Use cases** — specular highlight shape in all modern PBR; α = roughness² is the standard remap.  
**Key variants** — isotropic GGX; anisotropic GGX (separate αx, αy); multi-scattering GGX (Kulla-Conty).  
**Limitations** — single-lobe; cannot model retroreflective NDFs without extension.  
**Reference** — [Walter et al. (2007): Microfacet Models](https://www.graphics.cornell.edu/~bjw/microfacetbsdf.pdf)

#### Smith Masking-Shadowing Function (G)
Estimates the fraction of microfacets visible from both the light and view directions, accounting for shadowing of microfacets by neighbours.

**Key variants** — separable Smith (G1(V) × G1(L)); height-correlated Smith (Heitz 2014) — more accurate at grazing angles at slight extra cost.  
**Limitations** — separable form underestimates masking at grazing angles; correlated form should be preferred for conductors.  
**Reference** — [Heitz: Understanding the Masking-Shadowing Function (JCGT 2014)](https://jcgt.org/published/0003/02/03/)

#### Schlick Fresnel Approximation
Approximates the wavelength-dependent Fresnel reflectance as F0 + (1 − F0)(1 − NdotV)⁵, where F0 is the normal-incidence reflectance (0.04 for dielectrics, metal colour for conductors).

**Use cases** — all real-time PBR; metallic/roughness workflow F0 parameterisation.  
**Limitations** — inaccurate for conductors with complex IOR; the full Fresnel equations are needed for thin-film and iridescence effects.  
**Reference** — [Schlick (1994)](https://doi.org/10.1111/1467-8659.1330233)

#### Multiscattering BRDF Correction (Kulla-Conty)
Adds a compensating energy term to single-scatter GGX BRDFs to restore energy lost by bright rough surfaces, which appear artificially dark without it.

**Use cases** — physically correct bright rough metals and dielectrics; critical for roughness > 0.5 on metals.  
**Key variants** — pre-integrated LUT lookup (fast); analytic approximation (Turquin, 2019).  
**Reference** — [Kulla & Conty: Revisiting Physically Based Shading (SIGGRAPH 2017)](https://blog.selfshadow.com/publications/s2017-shading-course/)

#### Disney "Principled" BRDF
An art-directed, artist-friendly 10-parameter BRDF (base colour, subsurface, metallic, specular, specular tint, roughness, anisotropic, sheen, sheen tint, clearcoat, clearcoat gloss) designed for physical plausibility without requiring physical parameterisation.

**Use cases** — the parameterisation basis for Unreal, Unity, Godot, Blender's Principled BSDF, and glTF 2.0 materials.  
**Key variants** — glTF metallic-roughness (subset); specular-glossiness (legacy); Autodesk Standard Surface (superset).  
**Reference** — [Burley: Physically Based Shading at Disney (SIGGRAPH 2012)](https://blog.selfshadow.com/publications/s2012-shading-course/)

#### Anisotropic GGX
Extends isotropic GGX with separate roughness along tangent and bitangent axes, producing elongated specular highlights aligned to surface flow direction.

**Use cases** — brushed metal, satin, hair fibre cross-sections, CDRom surfaces, carbon fibre.  
**Key variants** — Burley bent-normal variant (rotates the highlight); Ward anisotropic (older, energy non-conserving).  
**Limitations** — requires consistent tangent vectors across the mesh; tangent discontinuities cause highlight seams.  
**Reference** — [Burley 2012 Disney BRDF](https://blog.selfshadow.com/publications/s2012-shading-course/)

#### Estevez-Kulla Sheen / Cloth BRDF
A specular lobe with retroreflective character (brightens as NdotV → 0 and NdotL → 0) designed to model fibrous cloth surfaces such as velvet and microfibre.

**Use cases** — fabric rendering in games and film; the "Charlie" sheen model used in Blender's Principled BSDF and glTF KHR_materials_sheen extension.  
**Reference** — [Estevez & Kulla: Production Friendly Microfacet Sheen BRDF (SIGGRAPH 2017)](https://blog.selfshadow.com/publications/s2017-shading-course/)

#### Iridescence / Thin-Film Interference
Models wavelength-dependent phase-shift Fresnel effects from sub-wavelength-thin dielectric coatings (soap bubbles, beetle shells, oil slicks), producing view-angle-dependent colour shifts.

**Use cases** — soap bubbles, beetle/butterfly wings, oil on water, iridescent paint finishes, CD/DVD surfaces.  
**Key variants** — Belcour-Barla spectral model (2017) converted to RGB; glTF KHR_materials_iridescence extension.  
**Reference** — [Belcour & Barla: Practical Rendering of Layered Materials (SIGGRAPH 2017)](https://belcour.github.io/blog/research/2017/05/01/brdf-thin-film.html)

---

### Anisotropic BRDF

Isotropic BRDFs (§3) depend only on the half-angle `θ_h` between view and light and are rotationally symmetric around the normal. **Anisotropic** BRDFs additionally depend on the azimuthal angle `φ` relative to the surface tangent, producing elongated specular highlights along one axis — characteristic of brushed metal, satin fabric, hair, CD tracks, and skin pores aligned in a direction.

**Shader stage**: evaluated per light or per environment sample in the **fragment shader**. The tangent direction is either from the mesh's tangent attribute or a flow map.

#### GGX Anisotropic (Heitz 2014)
Extends the isotropic GGX NDF with separate roughness values `α_x` and `α_y` along the tangent `T` and bitangent `B`. The NDF becomes a bivariate Gaussian in `(h·T, h·B)` space.

```glsl
// Anisotropic GGX NDF — Heitz 2014
float D_GGX_aniso(vec3 H, vec3 N, vec3 T, vec3 B, float alpha_x, float alpha_y) {
    float HdotT = dot(H, T);
    float HdotB = dot(H, B);
    float HdotN = dot(H, N);
    float denom = (HdotT * HdotT) / (alpha_x * alpha_x)
                + (HdotB * HdotB) / (alpha_y * alpha_y)
                + HdotN * HdotN;
    return 1.0 / (3.14159 * alpha_x * alpha_y * denom * denom);
}

// Anisotropic Smith masking-shadowing (approximate)
float G1_GGX_aniso(vec3 V, vec3 N, vec3 T, vec3 B, float alpha_x, float alpha_y) {
    float VdotN = abs(dot(V, N));
    float VdotT = dot(V, T);
    float VdotB = dot(V, B);
    float alpha2 = sqrt((VdotT * alpha_x) * (VdotT * alpha_x)
                      + (VdotB * alpha_y) * (VdotB * alpha_y)) / VdotN;
    return 2.0 / (1.0 + sqrt(1.0 + alpha2 * alpha2));
}

// Full anisotropic BRDF evaluation
vec3 brdf_aniso(vec3 L, vec3 V, vec3 N, vec3 T, vec3 B,
                float alpha_x, float alpha_y, vec3 F0) {
    vec3  H    = normalize(L + V);
    float NdotL = max(dot(N, L), 1e-4);
    float NdotV = max(dot(N, V), 1e-4);
    float D    = D_GGX_aniso(H, N, T, B, alpha_x, alpha_y);
    float G    = G1_GGX_aniso(L, N, T, B, alpha_x, alpha_y)
               * G1_GGX_aniso(V, N, T, B, alpha_x, alpha_y);
    vec3  F    = F0 + (1.0 - F0) * pow(1.0 - max(dot(H, V), 0.0), 5.0);
    return (D * G * F) / (4.0 * NdotL * NdotV);
}
```

#### Tangent Bending and Kajiya-Kay (Hair)
Hair strands use the **Kajiya-Kay** model where the NDF is evaluated along the hair tangent rather than the surface normal — producing the characteristic shifted specular band seen in real hair.

```glsl
// Kajiya-Kay hair specular — one lobe
float kajiya_kay_specular(vec3 T, vec3 L, vec3 V, float roughness, float shift) {
    // Shift tangent by 'shift' along normal direction (two lobes: primary + secondary)
    vec3  Ts     = normalize(T + shift * N);
    float TdotL  = dot(Ts, L);
    float TdotV  = dot(Ts, V);
    float sinTL  = sqrt(max(0.0, 1.0 - TdotL * TdotL));
    float sinTV  = sqrt(max(0.0, 1.0 - TdotV * TdotV));
    float spec   = max(0.0, sinTL * sinTV + TdotL * TdotV);
    return pow(spec, 1.0 / max(roughness * roughness, 0.001));
}
```

#### Anisotropy from Flow Map
For surfaces without explicit tangent attributes (cloth weave, brushed surface), read the anisotropy direction from a 2-channel flow map and rotate the tangent frame accordingly.

```glsl
vec2  flow      = texture(flow_map, uv).rg * 2.0 - 1.0;  // direction in tangent space
vec3  aniso_T   = normalize(T * flow.x + B * flow.y);
vec3  aniso_B   = cross(N, aniso_T);
float alpha_x   = roughness * (1.0 + anisotropy);    // stretch in T direction
float alpha_y   = roughness * (1.0 - anisotropy);    // compress in B direction
```

**Reference** — [Heitz: Understanding the Masking-Shadowing Function in Microfacet-Based BRDFs (JCGT 2014)](https://jcgt.org/published/0003/02/03/); [Marschner et al.: Light Scattering from Human Hair Fibers (SIGGRAPH 2003)](https://dl.acm.org/doi/10.1145/1201775.882345)

---

### Clear Coat and Sheen BRDFs

Two additional material layers that extend the base PBR model (§3) for common real-world materials:

- **Clear coat**: a thin, smooth dielectric layer on top of the base BRDF (car paint, lacquered wood, nail polish). It adds a second specular lobe with its own roughness and attenuates the base layer by the coat's Fresnel term.
- **Sheen**: a retroreflective rim highlight for fabric microfibre (velvet, silk, satin). The Charlie distribution peaks at grazing angles, opposite to GGX which peaks at normal incidence.

**Shader stage**: both are additions to the **fragment shader** BRDF evaluation, combined additively (clear coat) or as a separate lobe (sheen).

#### Clear Coat BRDF (KHR_materials_clearcoat)
```glsl
// Clear coat layer — dielectric coat over base BRDF
uniform float coat_roughness;   // typically 0.0–0.1 (smooth lacquer)
uniform float coat_ior;         // e.g. 1.5 (polyurethane)
uniform float coat_weight;      // 0–1 blend factor

vec3 brdf_clearcoat(vec3 N, vec3 V, vec3 L, vec3 base_brdf_result) {
    vec3  H       = normalize(V + L);
    float NdotH   = max(dot(N, H), 1e-4);
    float NdotV   = max(dot(N, V), 1e-4);
    float NdotL   = max(dot(N, L), 1e-4);
    float HdotV   = max(dot(H, V), 0.0);

    // Coat specular: GGX with coat roughness
    float alpha_c = coat_roughness * coat_roughness;
    float D_c     = D_GGX(NdotH, alpha_c);
    float G_c     = G_smith(NdotV, NdotL, alpha_c);
    // Coat Fresnel: dielectric (F0 from IOR)
    float f0_coat = pow((coat_ior - 1.0) / (coat_ior + 1.0), 2.0);
    float F_c     = f0_coat + (1.0 - f0_coat) * pow(1.0 - HdotV, 5.0);

    float coat_spec = D_c * G_c * F_c / (4.0 * NdotV * NdotL + 1e-5);

    // Coat attenuates base: energy absorbed by coat Fresnel
    vec3 result = base_brdf_result * (1.0 - coat_weight * F_c)
                + vec3(coat_spec) * coat_weight;
    return result;
}
```

#### Sheen BRDF — Charlie Distribution
The **Charlie distribution** (Estevez & Kulla 2017) models fabric microfibre: the NDF has a sinusoidal shape that peaks at grazing incidence, producing the soft rim glow of velvet.

```glsl
// Charlie sheen NDF (Estevez & Kulla, "Production Friendly Microfacet Sheen BRDF" 2017)
float D_charlie(float NdotH, float sheen_roughness) {
    float invAlpha = 1.0 / max(sheen_roughness, 0.001);
    float cos2h    = NdotH * NdotH;
    float sin2h    = max(1.0 - cos2h, 0.0078125);  // avoid zero
    return (2.0 + invAlpha) * pow(sin2h, invAlpha * 0.5) / (2.0 * 3.14159);
}

// Sheen visibility (Neubelt approximate for cloth)
float V_sheen(float NdotL, float NdotV) {
    return clamp(1.0 / (4.0 * (NdotL + NdotV - NdotL * NdotV)), 0.0, 1.0);
}

vec3 brdf_sheen(vec3 N, vec3 V, vec3 L, vec3 sheen_color, float sheen_roughness) {
    vec3  H     = normalize(V + L);
    float NdotH = max(dot(N, H), 1e-4);
    float NdotV = max(dot(N, V), 1e-4);
    float NdotL = max(dot(N, L), 1e-4);
    float D = D_charlie(NdotH, sheen_roughness);
    float V_vis = V_sheen(NdotL, NdotV);
    return sheen_color * D * V_vis * NdotL;
}

// Combined in fragment shader:
// total = base_brdf + sheen_weight * brdf_sheen(...) + clearcoat layer
```

**Energy conservation**: the sheen lobe adds energy on top of the base diffuse; scale the base diffuse by `(1 - sheen_weight * sheen_albedo)` to maintain energy conservation.

**Reference** — [Estevez & Kulla: Production Friendly Microfacet Sheen BRDF (2017)](https://blog.selfshadow.com/publications/s2017-shading-course/); [KhronosGroup: KHR_materials_clearcoat (glTF 2.0)](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_materials_clearcoat)

---

### Glint and Sparkle Microstructure BRDF

Standard microfacet BRDFs integrate over millions of sub-pixel facets, producing a smooth averaged specular lobe. **Glint rendering** resolves individual facets — producing sharp, flickering specular spots on metallic flake, glitter, sequins, and sparkle-finish car paint. Each visible micro-facet produces a point-like specular reflection; at normal-map resolution each texel covers thousands of facets, but the GPU can stochastically evaluate a per-pixel representative sample.

**Shader stage**: evaluated in the **fragment shader** per lit pixel. Requires a precomputed normal distribution texture or procedural normal sampling.

#### Discrete Stochastic Glint (Chermain 2021)
Sample a small number of facet normals per pixel using a hierarchical normal distribution precomputed into a multi-level texture. Each sampled normal produces a sharp Dirac-delta specular contribution if it is close enough to the half-vector to be visible.

```glsl
// Stochastic glint BRDF — discrete facet sampling
layout(set=0, binding=5) uniform sampler2DArray glint_ndf;  // hierarchical NDF atlas
uniform int   glint_samples;    // typically 4–8 per pixel
uniform float glint_density;    // facets per mm²
uniform float glint_roughness;  // individual facet roughness

// Per-pixel RNG seeded by UV and frame (temporal variation)
uint rng = pcg_init(uvec2(gl_FragCoord.xy), frame_index);

vec3 glint_radiance = vec3(0.0);
float pixel_footprint = length(fwidth(v_uv)) * tex_size_mm;  // pixel size in mm

for (int i = 0; i < glint_samples; ++i) {
    // Sample a facet normal from the NDF at this footprint scale
    // (In full implementation: hierarchical NDF texture lookup per Chermain 2021)
    float u1 = pcg_float(rng), u2 = pcg_float(rng);

    // GGX-distributed facet normal in tangent space
    float phi   = u1 * 6.2832;
    float cos_t = sqrt((1.0 - u2) / (1.0 + (glint_roughness*glint_roughness - 1.0) * u2));
    float sin_t = sqrt(1.0 - cos_t * cos_t);
    vec3  m     = normalize(T * sin_t * cos(phi) + B * sin_t * sin(phi) + N * cos_t);

    // Specular if half-vector matches facet normal within pixel footprint
    vec3  H_pixel = normalize(L + V);
    float angular_dist = acos(clamp(dot(m, H_pixel), -1.0, 1.0));
    float footprint_angle = pixel_footprint / (2.0 * view_dist);  // half-angle of pixel

    if (angular_dist < footprint_angle) {
        // This facet is visible and facing toward H — sharp specular contribution
        float NdotL = max(dot(m, L), 0.0);
        float NdotV = max(dot(m, V), 0.0);
        // Weight by probability of sampling this facet in this pixel
        float w = 1.0 / float(glint_samples) / max(footprint_angle * footprint_angle, 1e-6);
        glint_radiance += F_schlick(F0, dot(m, V)) * NdotL * w * light_color;
    }
}
// Blend with smooth GGX for base roughness (covers all non-glint contribution)
vec3 base_brdf = brdf_cook_torrance(N, V, L, roughness, F0);
frag_color = vec4(base_brdf + glint_radiance * glint_density, 1.0);
```

**Reference** — [Chermain et al.: Glint Rendering Based on a Multiple-Scattering Patch BRDF (CGF 2021)](https://doi.org/10.1111/cgf.14382); [Chiang et al.: A Practical and Controllable Hair and Fur Model Based on a Separable Analytic BRDF (CGF 2015)](https://dl.acm.org/doi/10.1111/cgf.12511)

---

### Iridescence and Thin-Film Interference

Iridescence is structural colour arising from thin-film interference: a transparent coating of thickness `d` (tens to hundreds of nanometres — soap bubble membrane, butterfly wing scale, oil slick, car paint clear-coat) causes light to reflect from both its top and bottom surfaces; the two reflected beams interfere constructively or destructively depending on wavelength and angle, producing a hue shift that changes with viewing direction.

**Shader stage**: evaluated in the **fragment shader** as a modification to the Fresnel term of the BRDF.

#### Thin-Film Phase Shift
The optical path difference between the two reflections is `2 * n * d * cos(θ_t)` where `θ_t` is the refraction angle inside the film (Snell's law). The interference produces wavelength-dependent reflectance `R(λ)`.

```glsl
// Thin-film iridescence Fresnel (single-layer, monochromatic evaluation)
// Call once per RGB wavelength (450nm, 550nm, 650nm) for colour
float thin_film_reflectance(float cos_theta_i, float n_film, float d_nm, float lambda_nm) {
    // Refraction angle in film (Snell's law)
    float sin_t2 = (1.0 - cos_theta_i * cos_theta_i) / (n_film * n_film);
    if (sin_t2 >= 1.0) return 1.0;  // total internal reflection
    float cos_theta_t = sqrt(1.0 - sin_t2);

    // Fresnel at air→film and film→substrate interfaces (assuming n_substrate ~ 1.5)
    float n0 = 1.0, n1 = n_film, n2 = 1.5;
    float rs01 = (n0 * cos_theta_i - n1 * cos_theta_t) / (n0 * cos_theta_i + n1 * cos_theta_t);
    float rs12 = (n1 * cos_theta_t - n2 * sqrt(max(0.0, 1.0 - (n1*n1*(1.0-cos_theta_t*cos_theta_t))/(n2*n2))))
               / (n1 * cos_theta_t + n2 * sqrt(max(0.0, 1.0 - (n1*n1*(1.0-cos_theta_t*cos_theta_t))/(n2*n2))));

    // Optical path difference and phase
    float opd   = 2.0 * n1 * d_nm * cos_theta_t;
    float phi   = 2.0 * 3.14159 * opd / lambda_nm;

    // Interference formula
    float t01   = 1.0 - rs01 * rs01;
    float r01_2 = rs01 * rs01;
    float r12_2 = rs12 * rs12;
    float denom = 1.0 + r01_2 * r12_2 - 2.0 * rs01 * rs12 * cos(phi);
    float r_num = r01_2 + r12_2 - 2.0 * rs01 * rs12 * cos(phi);
    return clamp(r_num / max(denom, 1e-6), 0.0, 1.0);
}

// Sample at RGB representative wavelengths
vec3 iridescence_fresnel(float cos_theta, float n_film, float d_nm) {
    return vec3(
        thin_film_reflectance(cos_theta, n_film, d_nm, 650.0),  // red
        thin_film_reflectance(cos_theta, n_film, d_nm, 550.0),  // green
        thin_film_reflectance(cos_theta, n_film, d_nm, 450.0)   // blue
    );
}
```

#### Integration with PBR BRDF
Replace the standard scalar Fresnel with the thin-film Fresnel vector in the specular lobe, and blend with the base BRDF Fresnel by an `iridescence_factor` parameter.

```glsl
// Blend thin-film Fresnel into PBR specular
vec3  F0        = mix(vec3(0.04), albedo, metallic);
vec3  F_base    = F_schlick(F0, HdotV);
vec3  F_film    = iridescence_fresnel(HdotV, iridescence_ior, iridescence_thickness);
vec3  F         = mix(F_base, F_film, iridescence_factor);

// Specular contribution with film Fresnel
float D = D_GGX(NdotH, roughness);
float G = G_smith(NdotV, NdotL, roughness);
vec3  specular = (D * G * F) / (4.0 * NdotV * NdotL + 1e-5);
```

**Use cases** — car paint second-layer coat; insect wings in nature scenes; oil-on-water puddles; holographic material; dragon/beetle character scales.  
**Reference** — [Belcour & Barla: A Practical Extension to Microfacet Theory for the Modeling of Varying Iridescence (SIGGRAPH 2017)](https://belcour.fr/blog/research/2017/05/01/brdf-thin-film.html); [KhronosGroup: KHR_materials_iridescence (glTF 2.0 extension)](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_materials_iridescence)

---

### Translucency and Backlit Thin-Surface Materials

Subsurface scattering (SSS) models thick media like skin or marble where light enters one point and exits at a different point. Translucency handles the simpler case: **thin surfaces** (leaves, ears, fabric, paper, petals) where light passes through the material and exits on the far side with a colour shift determined by the material thickness. This is a single-scatter approximation, not full SSS.

**Shader stage**: evaluated in the **fragment shader** alongside the standard BRDF, requiring only the light direction, view direction, surface normal, and a thickness value (from a pre-baked thickness map or a constant).

#### Wrap Lighting
Extends the Lambertian `max(0, NdotL)` term to wrap around the silhouette by a `wrap` factor `w ∈ [0, 1]`, so the dark side of a thin object receives some illumination even when `NdotL < 0`. A simple and cheap first approximation for backlit leaves or fabric.

```glsl
// Wrap lighting — simulates light wrapping around thin surfaces
float diffuse_wrap(vec3 N, vec3 L, float w) {
    return max(0.0, (dot(N, L) + w) / ((1.0 + w) * (1.0 + w)));
}
```

#### Thickness-Map Translucency (Epic Games / UE4)
Adds a transmitted light lobe on the back face of the surface, driven by a pre-baked thickness texture. The transmitted colour is the light colour tinted by a sub-surface colour parameter and attenuated exponentially by the thickness value. The lobe is view-dependent (maximised when looking through the surface toward the light — the "backlit leaf" effect).

```glsl
// UE4-style translucency — evaluated in addition to standard BRDF
vec3 translucency(vec3 L, vec3 V, vec3 N,
                  vec3 subsurface_color, float thickness,
                  vec3 light_color) {
    float scatter_power   = 12.0;   // controls lobe width
    float scatter_scale   = 0.5;    // overall intensity
    // Transmitted direction: light through the surface toward the viewer
    vec3  H_trans         = normalize(L + N * 0.1);  // slightly bent into surface
    float trans_NdotL     = pow(clamp(dot(V, -H_trans), 0.0, 1.0), scatter_power)
                          * scatter_scale;
    float attenuation     = exp(-thickness * 8.0);   // thickness map: 0=thin, 1=thick
    return light_color * subsurface_color * trans_NdotL * attenuation;
}
```

**Use cases** — foliage (leaves, grass), ears, thin cloth, paper lanterns; adds negligible cost (one texture fetch + a few multiplies) when a thickness map is available.  
**Reference** — [Penner: Pre-Integrated Skin Shading (GPU Pro 2, 2011)]; [UE4 Shading Models: Two-Sided Foliage](https://docs.unrealengine.com/en-US/shading-models/)

#### Dual-Lobe Transmission (Volumetric Thin-Surface)
A more accurate model uses two Henyey-Greenstein lobes — a forward-scattering lobe and a back-scattering lobe — weighted by the material's mean free path and the Fresnel transmission coefficient. Used for high-quality foliage in film rendering.

**Reference** — [Jakob et al.: A Radiative Transfer Framework for Rendering Materials with Anisotropic Structure (SIGGRAPH 2010)](https://dl.acm.org/doi/10.1145/1778765.1778834)

---

### Subsurface Scattering

Subsurface scattering (SSS) models materials where light enters the surface, scatters through the volume, and exits at a different point — skin, marble, wax, milk. In rasterized real-time rendering the primary approaches are **screen-space SSS** (Jimenez), **separable SSS** (Jimenez et al. 2009, used in Unreal Engine's skin shading), and **pre-integrated skin shading** (Penner 2011, used in UE4). The common thread: approximate the diffusion profile as a weighted sum of Gaussian blurs in screen space.

**Shader stage**: the spatial blur passes run as **compute or fragment** post-processes on the subsurface-tagged pixels. The irradiance accumulation runs in the main **fragment shader**.

#### Pre-Integrated Skin Shading (Penner)
Bakes the diffuse SSS integration over a sphere of varying curvature into a 2D LUT indexed by `(NdotL * 0.5 + 0.5, curvature)`, precomputed offline. At runtime a single `texture()` fetch gives the colour-shifted BRDF integral — cheap and smooth.

```glsl
// Pre-integrated SSS — fragment shader (skin material)
layout(set=2, binding=4) uniform sampler2D sss_lut;  // 2D pre-integrated skin LUT
layout(set=2, binding=5) uniform sampler2D sss_thickness; // thickness map for back-scatter

uniform vec3 light_dir;
uniform vec3 light_color;

void main() {
    float NdotL       = dot(N, light_dir);
    // Estimate surface curvature from ddx/ddy of world normal
    float curvature   = length(fwidth(N)) / length(fwidth(v_world_pos)) * curvature_scale;

    // LUT encodes colour-shifted skin response per curvature
    vec3  sss_diffuse = texture(sss_lut, vec2(NdotL * 0.5 + 0.5, curvature)).rgb;
    sss_diffuse      *= light_color * shadow;

    // Thin-surface transmitted light (back-scatter, e.g. ear lobes)
    float thickness   = texture(sss_thickness, uv).r;
    vec3  transmit    = vec3(0.25, 0.05, 0.01)   // skin scatter tint (R > G >> B)
                      * light_color
                      * max(0.0, dot(-light_dir, V))
                      * exp(-thickness * 8.0);

    frag_color = vec4((sss_diffuse + transmit) * albedo, 1.0);
}
```

#### Separable SSS — Screen-Space Blur
After the lighting pass, objects tagged as SSS are blurred in screen space using a 1D kernel whose weights are derived from the sum-of-Gaussians diffusion profile. Two passes: horizontal then vertical. The kernel is colour-dependent (wider for red channel, narrower for blue).

```glsl
// Separable SSS blur — one axis (run twice: horizontal, vertical)
layout(set=0, binding=0) uniform sampler2D lighting_buf;  // colour buffer from main pass
layout(set=0, binding=1) uniform sampler2D depth_buf;
layout(set=0, binding=2) uniform sampler2D stencil_sss;   // 1 = SSS pixel
layout(set=0, binding=3, rgba16f) uniform image2D sss_out;

uniform vec2  blur_dir;      // (1,0) or (0,1) for horiz/vert pass
uniform float blur_width_mm; // e.g. 25.0 mm (skin scatter radius at 1m)
uniform float proj_scale;    // pixels per meter at depth 1m

// Sum-of-3-Gaussians kernel weights for skin (Jimenez 2010 profile)
// 7-tap kernel (simplified): weights for R/G/B per tap index
const float kernel_weights[7] = float[](0.006, 0.061, 0.242, 0.383, 0.242, 0.061, 0.006);
const vec3  kernel_rgb[7] = vec3[](
    vec3(0.006, 0.002, 0.001), vec3(0.061, 0.020, 0.005), vec3(0.242, 0.120, 0.030),
    vec3(0.383, 0.298, 0.080), vec3(0.242, 0.120, 0.030), vec3(0.061, 0.020, 0.005),
    vec3(0.006, 0.002, 0.001)
);

void main() {
    ivec2 px   = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv   = (vec2(px) + 0.5) / vec2(imageSize(sss_out));
    if (texture(stencil_sss, uv).r < 0.5) {
        imageStore(sss_out, px, texture(lighting_buf, uv));
        return;
    }

    float depth  = texture(depth_buf, uv).r;
    float scale  = proj_scale * blur_width_mm / (1000.0 * depth);  // kernel in pixels

    vec4  accum  = vec4(0.0);
    for (int i = 0; i < 7; ++i) {
        float t    = float(i - 3) * scale;
        vec2  s_uv = uv + blur_dir * t / resolution;
        // Depth similarity weight: clamp blur at depth discontinuities
        float s_depth = texture(depth_buf, s_uv).r;
        float dw      = exp(-abs(s_depth - depth) * depth_edge_scale);
        accum.rgb    += texture(lighting_buf, s_uv).rgb * kernel_rgb[i] * dw;
        accum.a      += kernel_weights[i] * dw;
    }
    imageStore(sss_out, px, vec4(accum.rgb / max(accum.a, 0.001), 1.0));
}
```

**Reference** — [Jimenez et al.: Separable Subsurface Scattering (CGF 2015)](https://iryoku.com/separable-sss/); [Penner: Pre-Integrated Skin Shading (GPU Pro 2, 2011)](https://developer.nvidia.com/gpugems/gpugems3)

---

### Eye Rendering

The human eye is optically complex: a roughly spherical sclera (white) with a hemispherical transparent cornea that refracts and magnifies the iris beneath it, a coloured iris with radial detail and a dark pupil, a thin limbal ring at the iris–sclera boundary, and subsurface scattering in both the sclera and iris. Accurate real-time eye rendering requires modelling corneal refraction, iris parallax depth, and directional limbal scatter.

**Shader stage**: all evaluated in the **fragment shader** per eye mesh. The eye is typically a two-mesh setup: an inner sphere (iris+pupil on a flat disc or curved surface) and an outer sphere (sclera+cornea).

#### Corneal Refraction and Iris Parallax
The iris appears behind the curved cornea. The refracted view ray is approximated by offsetting the iris UV lookup by the view angle, giving a parallax effect that makes the pupil appear to move as the camera orbits.

```glsl
// Cornea refraction — iris UV offset from view direction
layout(set=0, binding=0) uniform sampler2D iris_tex;
layout(set=0, binding=1) uniform sampler2D normal_map;
layout(set=0, binding=2) uniform sampler2D sclera_tex;
layout(set=0, binding=3) uniform sampler2D blood_vessel_tex;

uniform float iris_depth;        // distance behind cornea surface (e.g. 0.006 m)
uniform float cornea_ior;        // e.g. 1.336 (aqueous humour)
uniform float pupil_scale;       // 0 = fully open, 1 = closed

layout(location=0) in vec3 v_normal;
layout(location=1) in vec3 v_tangent;
layout(location=2) in vec2 v_uv;
layout(location=3) in vec3 v_view_dir;   // world-space camera direction

void main() {
    vec3 N  = normalize(v_normal);
    vec3 V  = normalize(-v_view_dir);

    // Refract view ray through cornea
    float eta   = 1.0 / cornea_ior;
    vec3  R_ref = refract(-V, N, eta);

    // Iris UV with parallax: offset by projected refracted ray at iris plane depth
    vec2  iris_offset = R_ref.xy / max(-R_ref.z, 0.001) * iris_depth;
    vec2  iris_uv     = v_uv + iris_offset * 0.5;  // scale to UV units

    // Pupil mask: scale UV toward centre
    vec2  centred_uv = iris_uv - 0.5;
    float r          = length(centred_uv);
    float pupil_r    = mix(0.35, 0.1, pupil_scale);    // adapt pupil radius
    float is_pupil   = step(r, pupil_r);
    vec4  iris_col   = mix(texture(iris_tex, iris_uv), vec4(0.01), is_pupil);

    // Cornea specular — GGX, very smooth (roughness ~ 0.0)
    float NdotL     = max(dot(N, light_dir), 0.0);
    vec3  H         = normalize(light_dir + V);
    float cornea_spec = D_GGX(dot(N, H), 0.02) * 0.04;  // F0 = 0.04 (water)

    // Sclera with veins and limbal ring darkening
    float limbal = 1.0 - smoothstep(0.47, 0.50, r);  // dark ring at iris edge
    vec4  sclera_col  = texture(sclera_tex, v_uv)
                      * texture(blood_vessel_tex, v_uv)
                      * limbal;

    // Blend iris and sclera
    float iris_mask = smoothstep(0.50, 0.48, r);
    vec3  surface   = mix(sclera_col.rgb, iris_col.rgb, iris_mask);
    frag_color = vec4(surface * NdotL + vec3(cornea_spec), 1.0);
}
```

#### Sclera Subsurface Scattering
The sclera scatters light below the surface, giving a soft translucent appearance near the limbal ring. Apply a separable SSS blur (§77) with a tight radius (sclera diffusion profile is narrow, ~1–2 mm) on the sclera stencil region.

**Reference** — [Jimenez et al.: Practical and Realistic Facial Wrinkles Animation (i3D 2011)](https://iryoku.com/wrinkles/); [Lim: Rendering Eyes in UE4 (GDC 2016)](https://www.gdcvault.com/play/1023510)

---

### Non-Photorealistic Rendering (NPR)

NPR deliberately departs from photorealism to achieve stylised, illustrative, or expressive looks — hand-drawn animation, ink lines, painterly abstraction, or technical illustration. Unlike PBR, there is no single correct model: the "right" NPR shader is defined by the target aesthetic, not by physics. Most NPR effects operate on the final image (post-process) rather than during shading, because they need global image context (edge detection, local colour statistics) that per-fragment shading cannot access. The practical challenge is making NPR techniques view-consistent and temporally stable — cel edges that flicker or Kuwahara patterns that shimmer under camera motion break the stylised illusion.

**Shader stage**: Cel/toon shading ramp evaluation runs in the **fragment shader**. Silhouette edge detection via back-face shell extrusion runs in the **vertex shader** (expand normals outward); edge detection on G-buffer normals/depth runs as a **compute shader** or **fragment shader** post-process. Kuwahara and hatching filters are full-screen **compute shaders** or **fragment shaders** operating on the final colour buffer.

#### Cel / Toon Shading
Quantises the diffuse NdotL ramp to discrete steps (typically 2–4) using a 1D ramp texture lookup, producing the flat-shaded appearance of hand-drawn animation.

**Use cases** — stylised games (Breath of the Wild, Genshin Impact, Guilty Gear Strive); anime aesthetic.  
**Key variants** — step ramp; smooth ramp with colour tinting; hatching integration; global rim light for silhouette definition.  
**Reference** — [GPU Gems Ch11: Non-Photorealistic Effects](https://developer.nvidia.com/gpugems/gpugems/part-iv-image-processing/chapter-24-fast-fourth-order-partial-differential-equations)

#### Silhouette Edge Rendering
Identifies and renders the silhouette (where front-face and back-face meet) as thick outlines, either via back-face shell extrusion in the vertex shader or via post-process edge detection on the GBuffer normals and depth.

**Use cases** — ink-line stylisation; cartoon outlines; technical illustration.  
**Key variants** — back-face extrusion (geometry approach, no depth test issues); Sobel/Laplacian on normals/depth (image approach, works on any geometry); geometry shader edge emission.  
**Reference** — [Raskar & Cohen: Image Precision Silhouette Edges (I3D 1999)](https://doi.org/10.1145/300523.300538)

#### Kuwahara Filter
Segments a neighbourhood around each pixel into overlapping quadrant regions, computes the mean and variance of each, and outputs the mean of the region with lowest variance — producing a painterly, flat-colour-like smoothing that preserves edges.

**Use cases** — painterly / oil-painting post-effect; combining with colour grading for an illustrative look.  
**Key variants** — isotropic Kuwahara (original, cheap); anisotropic Kuwahara (Kyprianidis 2009) — orients regions along the local flow field for brush-like directionality.  
**Reference** — [Kyprianidis et al.: Image and Video Abstraction by Anisotropic Kuwahara Filtering (CGF 2009)](https://doi.org/10.1111/j.1467-8659.2009.01574.x)

#### Hatching and Tonal Art Maps (TAM)
Renders tonal variation via procedurally or pre-baked stroke textures (lines, cross-hatch) blended according to local diffuse luminance, producing a pencil-sketch or pen-and-ink appearance.

**Use cases** — technical illustrations; scientific visualisations; stylised NPR rendering.  
**Reference** — [Praun et al.: Real-Time Hatching (SIGGRAPH 2001)](https://doi.org/10.1145/383259.383328)

---


---

## VI. Character and Organic Rendering

Characters and creatures have specialized rendering requirements that cut across the material, geometry, and animation domains. Skin requires subsurface scattering and careful BRDF parameterization; hair and fur require strand geometry and multiple-scattering BSDFs; cloth requires both a simulation back-end and anisotropic shading; and skeletal animation provides the vertex positions that all of the above are evaluated at. These four topics are collected here because they are tightly coupled in production character pipelines — optimizing one typically affects the others.

### Skin, Hair, and Character Rendering

Human characters demand the highest fidelity in most applications, because viewers are exquisitely sensitive to subtle errors in skin tone, hair light transport, and eye gaze. Skin requires subsurface scattering (light enters the surface, scatters through the dermis, and exits at a different point) — the techniques here span cheap pre-integrated LUT approaches to full separable screen-space filters. Hair requires a BSDF that models the cylindrical fibre geometry (Kajiya-Kay for games, Marschner for film). Eyes combine multiple distinct shading regions — sclera, iris, cornea, pupil — each with different optical behaviour. These algorithms are high-cost and high-reward: they are where the "uncanny valley" crossing happens.

**Shader stage**: Pre-integrated skin SSS is a pure **fragment shader** computation (LUT lookup keyed by NdotL and curvature). Separable SSS runs as one or more full-screen **compute shader** or **fragment shader** blur passes on the skin cluster's diffuse buffer. Hair BSDF (Kajiya-Kay, Marschner) evaluates in the **fragment shader** (or **closest-hit shader** for ray-traced hair). Eye rendering is entirely **fragment shader**. All character shading consumes geometry produced by the **vertex shader**.

#### Pre-Integrated Skin SSS (Penner)
Stores the diffuse subsurface scattering profile as a 2D LUT indexed by NdotL and surface curvature (estimated from derivative of world-space normal), producing plausible skin SSS at very low cost.

**Use cases** — real-time skin in games where screen-space SSS is too expensive; mobile character rendering.  
**Reference** — [Penner: Pre-Integrated Skin Shading (GPU Pro 2, 2011)](https://gpupro.blogspot.com/)

#### Separable Screen-Space SSS (Jimenez)
Blurs per-skin-cluster screen-space diffuse contributions using a 1D separable Gaussian kernel fitted to measured skin profiles (a 6-Gaussian sum for different subsurface depths).

**Use cases** — character close-ups in Uncharted, The Last of Us, and most AAA character pipelines; also available in Unreal Engine.  
**Reference** — [Jimenez et al.: Separable Subsurface Scattering (2015)](https://iryoku.com/separable-sss/)

#### Kajiya-Kay Hair Shading
Computes specular highlights from hair fibres using the fibre tangent vector rather than a surface normal, producing the characteristic elongated highlights of real hair without per-fibre normals.

**Use cases** — all real-time hair rendering systems; the baseline before physically correct Marschner.  
**Limitations** — does not model light transmission through the hair fibre; highlights are empirical, not physically derived.  
**Reference** — [Kajiya & Kay (1989): Rendering Fur with Three Dimensional Textures](https://doi.org/10.1145/74333.74361)

#### Marschner Hair BSDF
Models light transport through a cylindrical hair fibre as three scattering paths: R (surface reflection), TT (transmitted through, visible as forward-scattered glow), and TRT (transmitted, internally reflected, transmitted — the secondary highlight).

**Use cases** — physically correct hair rendering in film and high-fidelity game characters; the basis for all production hair BSDFs.  
**Key variants** — d'Eon et al. (2011) improved multiple-scattering approximation; Weta "dual scattering" for efficient global scattering.  
**Reference** — [Marschner et al.: Light Scattering from Human Hair Fibres (SIGGRAPH 2003)](https://doi.org/10.1145/882262.882345)

#### Eye Rendering
Combines sclera diffuse (+ limbus ring darkening), cornea specular (clear coat over iris), iris refraction (depth-parallax offset), and pupil dilation to produce convincing real-time eyes.

**Use cases** — character close-ups; photorealistic digital humans.  
**Reference** — [Advances in Real-Time Rendering: Rendering Eyes (various years)](https://advances.realtimerendering.com/)

---

### Fur and Shell Rendering

Fur, grass, and short hair cannot be represented faithfully by a single surface mesh — individual strands must be visible at close range and silhouette edges must show the fibrous structure. **Shell rendering** approximates this by extruding N concentric offset shells of the base mesh, each rendered with an alpha-masked strand texture that thins toward the tips. **Fin rendering** adds billboard quads along silhouette edges for a correct fur silhouette. Together they form the standard real-time fur pipeline.

**Shader stage**: shells use the **vertex shader** (offset along normal per shell index) and **fragment shader** (alpha discard). Fins use a **geometry shader** or are generated in **compute** as a billboard vertex buffer.

#### Shell Generation (Vertex Shader)
Each shell is a separate draw of the base mesh. A push constant or uniform carries the shell index `k` and total shell count `K`. The vertex is displaced along the normal by `k/K` scaled by the fur length.

```glsl
// Shell vertex shader — one draw call per shell
layout(location=0) in vec3 position;
layout(location=1) in vec3 normal;
layout(location=2) in vec2 uv;

uniform mat4  mvp;
uniform int   shell_index;    // 0 = base, K-1 = tip
uniform int   shell_count;    // typically 16–32
uniform float fur_length;     // max extrusion (e.g. 0.05 m)
uniform float fur_gravity;    // downward sag (e.g. 0.3)

layout(location=0) out vec2 v_uv;
layout(location=1) out float v_shell_t;  // 0 = root, 1 = tip

void main() {
    float t    = float(shell_index) / float(shell_count - 1);
    // Gravity sag: blend normal toward -Y as t increases
    vec3  dir  = normalize(mix(normal, vec3(0.0, -1.0, 0.0), t * fur_gravity));
    vec3  pos  = position + dir * (t * fur_length);
    gl_Position = mvp * vec4(pos, 1.0);
    v_uv        = uv;
    v_shell_t   = t;
}
```

#### Shell Alpha Masking (Fragment Shader)
At each shell, sample a fur strand texture (thin lines on black background, generated from a noise pattern) and discard fragments where the strand is absent. Strands thin toward the tips — scale the alpha threshold by `t`.

```glsl
// Shell fragment shader — alpha discard for strand presence
layout(set=0, binding=0) uniform sampler2D fur_strand_tex;   // tileable strand mask
layout(set=0, binding=1) uniform sampler2D fur_color_tex;

layout(location=0) in vec2  v_uv;
layout(location=1) in float v_shell_t;

uniform float fur_density;   // UV tiling scale (e.g. 30.0)
uniform float fur_thickness; // strand width at root (e.g. 0.8)
uniform vec3  light_dir;
uniform vec3  tip_color;
uniform vec3  root_color;

layout(location=0) out vec4 frag;

void main() {
    vec2  fur_uv = v_uv * fur_density;
    float strand = texture(fur_strand_tex, fur_uv).r;

    // Thin strands toward tips: threshold rises from 0 to (1 - fur_thickness)
    float threshold = v_shell_t * (1.0 - fur_thickness);
    if (strand < threshold) discard;

    // Colour: blend root to tip colour
    vec3  col  = mix(root_color, tip_color, v_shell_t);
    // Simple diffuse + ambient
    float ndotl = max(dot(normalize(normal), light_dir), 0.0) * 0.8 + 0.2;
    frag = vec4(col * ndotl, 1.0);
}
```

#### Fin Rendering (Silhouette Quads)
For each silhouette edge (edges shared by a front-facing and back-facing triangle), emit a billboard quad extruded along the average of the two edge vertex normals. The fin uses the same strand texture with UVs spanning the full fur length.

```glsl
// Geometry shader — emit fin quads at silhouette edges
layout(triangles) in;
layout(triangle_strip, max_vertices=4) out;

layout(location=0) in  vec3 g_world_pos[];
layout(location=1) in  vec3 g_world_normal[];
layout(location=0) out vec2 f_uv;
layout(location=1) out float f_t;

uniform mat4 mvp;
uniform float fur_length;

void emit_fin(vec3 p0, vec3 p1, vec3 n0, vec3 n1) {
    gl_Position = mvp * vec4(p0,                           1.0); f_uv = vec2(0.0, 0.0); f_t = 0.0; EmitVertex();
    gl_Position = mvp * vec4(p1,                           1.0); f_uv = vec2(1.0, 0.0); f_t = 0.0; EmitVertex();
    gl_Position = mvp * vec4(p0 + n0 * fur_length,         1.0); f_uv = vec2(0.0, 1.0); f_t = 1.0; EmitVertex();
    gl_Position = mvp * vec4(p1 + n1 * fur_length,         1.0); f_uv = vec2(1.0, 1.0); f_t = 1.0; EmitVertex();
    EndPrimitive();
}
```

**Use cases** — animal characters at close range (Ratchet & Clank, Spyro); grass fields (shells used for density with low K); carpet and velvet materials at close zoom; fur coats on character periphery.  
**Reference** — [Lengyel: Rendering with Fur (GDC 2000)](https://dl.acm.org/doi/10.1145/344779.344844); [Yuksel & Keyser: Deep Opacity Maps (CGF 2008)](https://dl.acm.org/doi/10.1111/j.1467-8659.2008.01149.x)

---

### Cloth and Soft-Body Simulation

Cloth and soft-body dynamics in games run entirely on the GPU in compute shaders, updating particle positions each frame without CPU readback. The two main paradigms are **spring-mass systems** (each edge is a Hooke spring; velocity integration) and **Position-Based Dynamics** (PBD, constraints enforced by direct position projection — the method used by Unreal Engine's Chaos Cloth and Unity DOTS Physics).

**Shader stage**: all simulation runs in **compute shaders**. The final vertex positions are written into a vertex buffer read by the standard rendering pipeline.

#### Spring-Mass Cloth
Stores cloth as a 2D grid of particle positions and velocities in SSBOs. Each frame: accumulate forces (gravity, spring forces on structural/shear/bend springs, wind), integrate velocity (Verlet or semi-implicit Euler), apply constraints (collision with collider objects, pin constraints for fixed attachment points).

```glsl
layout(local_size_x=16, local_size_y=16) in;
layout(set=0, binding=0) buffer Positions { vec4 pos[]; };
layout(set=0, binding=1) buffer Velocities{ vec4 vel[]; };

uniform float dt, stiffness, damping;
uniform vec3  gravity;

void main() {
    uvec2 gid = gl_GlobalInvocationID.xy;
    if (any(greaterThanEqual(gid, grid_size))) return;
    uint idx = gid.y * grid_size.x + gid.x;

    vec3 p = pos[idx].xyz;
    vec3 v = vel[idx].xyz;

    // Accumulate spring forces from 4 structural neighbours
    vec3 force = gravity;
    for (int d = 0; d < 4; ++d) {
        ivec2 nb_gid = ivec2(gid) + neighbour_offsets[d];
        if (any(lessThan(nb_gid, ivec2(0))) ||
            any(greaterThanEqual(nb_gid, ivec2(grid_size)))) continue;
        uint nb_idx = uint(nb_gid.y) * grid_size.x + uint(nb_gid.x);
        vec3 delta  = pos[nb_idx].xyz - p;
        float len   = length(delta);
        float rest  = rest_lengths[d];
        force      += stiffness * (len - rest) / max(len, 1e-6) * delta;
    }

    // Semi-implicit Euler integration
    v += force * dt;
    v *= (1.0 - damping * dt);  // velocity damping
    p += v * dt;

    pos[idx].xyz = p;
    vel[idx].xyz = v;
}
```

**Use cases** — flags, banners, character cloaks; multiple dispatch passes per frame (substeps) for stability at high stiffness.  
**Reference** — [Jakobsen: Advanced Character Physics (GDC 2001)](https://www.cs.cmu.edu/afs/cs/academic/class/15462-s13/www/lec_slides/Jakobsen.pdf)

#### Position-Based Dynamics (PBD)
PBD replaces force-based integration with direct constraint projection. Each substep: (1) integrate gravity into predicted positions `p*`; (2) iteratively project distance constraints (edge length preservation), collision constraints (no penetration), and pin constraints (fixed vertices); (3) commit `p* → p`, derive velocity from `(p_new - p_old) / dt`. PBD is unconditionally stable at any dt — it cannot explode.

```glsl
// PBD distance constraint projection — one invocation per edge
layout(set=0, binding=0) buffer Positions { vec4 pos[]; };

uniform float compliance;   // 0 = rigid; larger = softer spring (XPBD)

void main() {
    uint e    = gl_GlobalInvocationID.x;
    uint i    = edges[e].a;
    uint j    = edges[e].b;
    float rest = edges[e].rest_length;

    vec3 delta  = pos[j].xyz - pos[i].xyz;
    float len   = length(delta);
    float C     = len - rest;                       // constraint violation
    float w_i   = inv_mass[i], w_j = inv_mass[j];

    float lambda = -C / (w_i + w_j + compliance);  // XPBD: compliance / dt²
    vec3 grad    = delta / max(len, 1e-6);

    // Atomic position update — race condition: requires parallel Gauss-Seidel or graph colouring
    pos[i].xyz -= w_i * lambda * grad;
    pos[j].xyz += w_j * lambda * grad;
}
```

**Parallelism caveat**: naïve PBD has read-write races. Production implementations use **graph colouring** (partition edges into independent sets, dispatch one colour at a time) or **XPBD with Jacobi updates** (accumulate corrections, apply once after all constraints).

**Use cases** — Unreal Engine Chaos Cloth, Unity DOTS Cloth, any soft-body or rope simulation. Supports inextensibility constraints (distance), volume preservation (sum-of-tetrahedra volume), and collision constraints in the same framework.  
**Reference** — [Müller et al.: Position-Based Dynamics (VRIPHYS 2006)](https://doi.org/10.1016/j.jvcir.2007.01.005); [Macklin et al.: XPBD: Position-Based Simulation of Compliant Constrained Dynamics (MIG 2016)](https://doi.org/10.1145/2994258.2994272)

---

### Skeletal Animation and Vertex Skinning

Character animation deforms meshes by blending a set of bone transforms, each weighted per vertex. The geometry is stored in a bind-pose reference frame; each vertex stores up to 4 (bone index, weight) pairs. At render time, the skinning shader transforms each vertex into world space by computing a weighted sum of the influencing bone transforms.

**Shader stage**: LBS and DQS run in the **vertex shader** when the mesh is rendered once (e.g., shadow-only or single view). A **compute skinning pre-pass** is preferred when the mesh is visible to multiple cameras simultaneously (shadow cascades, reflection probes) — skin once, draw N times from the resulting vertex buffer.

#### Linear Blend Skinning (LBS)
Transforms each vertex position and normal by a weighted sum of up to 4 bone matrices, stored as a per-frame SSBO of `mat4` bone transforms (joint matrices = inverse bind matrix × world bone matrix).

```glsl
// Vertex shader skinning — LBS
layout(set=1, binding=0) readonly buffer BoneSSBO { mat4 bones[]; };

layout(location=0) in vec3  in_pos;
layout(location=1) in vec3  in_normal;
layout(location=2) in uvec4 in_joints;
layout(location=3) in vec4  in_weights;

void main() {
    mat4 skin =
        in_weights.x * bones[in_joints.x] +
        in_weights.y * bones[in_joints.y] +
        in_weights.z * bones[in_joints.z] +
        in_weights.w * bones[in_joints.w];

    vec4 world_pos = skin * vec4(in_pos, 1.0);
    vec3 world_nrm = normalize(mat3(skin) * in_normal);
    gl_Position    = view_proj * world_pos;
}
```

**Use cases** — the universal baseline; glTF 2.0 mandates LBS support. Fast, broadly correct, collapses under high-twist rotations (candy-wrapper artifact at shoulders, wrists).  
**Reference** — [glTF 2.0 Specification: Skinning](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html#skins)

#### Dual Quaternion Skinning (DQS)
Converts bone matrices to dual quaternions, blends in dual quaternion space (normalised linear blend), then converts back to a matrix. Eliminates the candy-wrapper artifact at the cost of a more complex vertex shader.

**Use cases** — characters with expressive limb twists (shoulders, wrists, spine); glTF extension `KHR_mesh_quantization` + DQS is the current best practice for high-fidelity characters.  
**Limitations** — slightly more expensive than LBS; blending non-antipodal dual quaternions produces bulging artifacts (requires shortest-path selection).  
**Reference** — [Kavan et al.: Skinning with Dual Quaternions (I3D 2007)](https://doi.org/10.1145/1230100.1230107)

#### Morph Targets (Blend Shapes)
Each morph target is a per-vertex delta (position, normal, tangent) stored as a texture or SSBO. The vertex shader accumulates weighted deltas onto the base mesh. Used for facial animation (phoneme shapes, FACS targets), secondary body shapes, and corrective shapes.

**Use cases** — facial animation in real-time characters; shape keys in Blender/glTF; corrective blend shapes for LBS artifacts.  
**Reference** — [glTF 2.0: Morph Targets](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html#morph-targets)

#### Compute Skinning Pre-Pass
A compute shader skins the mesh once per frame into a `VkBuffer` acting as a vertex buffer, which is then bound for all subsequent draw calls (direct, shadow, reflection). Avoids redundant per-vertex skinning work when a character appears in multiple views.

**Use cases** — characters casting shadows into multiple cascade levels; characters visible in both main view and planar reflections; any mesh drawn N > 1 times per frame.  
**Reference** — [UE4 GPU Skin Cache](https://docs.unrealengine.com/en-US/gpu-skinning-cache/)

---


---


## Integrations

**Ch204** (this series) establishes the rendering pipeline configurations (deferred, forward+, GPU-driven) that the GI and material algorithms here run within. **Ch206** (this series) covers ray tracing algorithms that extend GI beyond screen-space approximations. **Ch135** (Vulkan Ray Tracing) provides the API for path-traced lightmap baking in Cat IV. **Ch127** (Mesh Shaders and VRS) documents how VRS interacts with material shading cost budgets. **Ch208** (GPU Geometry Algorithms — Surface Representation) covers the subdivision and NURBS geometry that feeds the PBR shading pipeline.
