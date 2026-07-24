# Chapter 207: Shader Algorithm Catalog — Visual Effects, Post-Processing, and GPU Compute (Part XXIX)

*Part XXIX — Graphics Algorithms*

**Target audiences**: Graphics application developers implementing screen-space effects, TAA/upscaling, colour grading, and general GPU compute primitives; browser engineers mapping WebGPU post-processing pipelines; systems developers optimising wave intrinsics and subgroup operations.

This chapter is the fourth and final volume of the Shader Algorithm Catalog. It covers Category X (Texturing and Surface Detail — virtual texturing, bindless, parallax occlusion, MSDF text), Category XI (Transparency, Reflections, and Compositing — OIT, SSR, decals, 3D Gaussian Splatting, neural rendering), Category XII (Post-Processing and Image Effects — TAA, DLSS/FSR/XeSS, bloom, DoF, colour grading, auto-exposure, dithering), and Category XIII (GPU Compute Primitives — wave intrinsics, radix sort, bitonic sort, cooperative matrices, spatial hash structures, sampling). The chapter includes the full References section for all four Shader Algorithm Catalog chapters (ch204–ch207).

## Table of Contents

- **[X. Texturing and Surface Detail](#x-texturing-and-surface-detail)**
  - [Texture Mapping and Surface Detail](#texture-mapping-and-surface-detail)
  - [Normal and HDR Encoding](#normal-and-hdr-encoding)
  - [Normal Map Blending](#normal-map-blending)
  - [Parallax Occlusion Mapping](#parallax-occlusion-mapping)
  - [Virtual Texturing and Sparse Virtual Textures](#virtual-texturing-and-sparse-virtual-textures)
  - [Sparse Texture and Tiled Resources](#sparse-texture-and-tiled-resources)
  - [Bindless Texturing and Descriptor Indexing](#bindless-texturing-and-descriptor-indexing)
  - [Multi-Channel SDF Font Rendering](#multi-channel-sdf-font-rendering)
- **[XI. Transparency, Reflections, and Compositing](#xi-transparency-reflections-and-compositing)**
  - [Order-Independent Transparency](#order-independent-transparency)
  - [Stochastic Rendering](#stochastic-rendering)
  - [Planar Reflections](#planar-reflections)
  - [Screen-Space Reflections](#screen-space-reflections)
  - [Screen-Space Techniques](#screen-space-techniques)
  - [Decal Rendering](#decal-rendering)
  - [Outline and Selection Rendering](#outline-and-selection-rendering)
  - [Octahedral Impostor LOD](#octahedral-impostor-lod)
  - [3D Gaussian Splatting](#3d-gaussian-splatting)
  - [Neural Rendering Primitives](#neural-rendering-primitives)
- **[XII. Post-Processing and Image Effects](#xii-post-processing-and-image-effects)**
  - [Post-Processing and Image Effects](#post-processing-and-image-effects)
  - [Anti-Aliasing, TAA, and ML Upscaling](#anti-aliasing-taa-and-ml-upscaling)
  - [Motion Vector Generation](#motion-vector-generation)
  - [Bloom — Dual Kawase and FFT Convolution](#bloom-dual-kawase-and-fft-convolution)
  - [Depth of Field — CoC and Scatter-as-Gather](#depth-of-field-coc-and-scatter-as-gather)
  - [Motion Blur — Per-Object Reconstruction Filter](#motion-blur-per-object-reconstruction-filter)
  - [Chromatic Aberration](#chromatic-aberration)
  - [Lens Flare](#lens-flare)
  - [Film Grain](#film-grain)
  - [Vignette and Lens Distortion](#vignette-and-lens-distortion)
  - [Contrast Adaptive Sharpening](#contrast-adaptive-sharpening)
  - [Color Science in Shaders](#color-science-in-shaders)
  - [Color Grading and 3D LUT](#color-grading-and-3d-lut)
  - [Histogram-Based Auto-Exposure](#histogram-based-auto-exposure)
  - [Ordered Dithering and Output Quantization](#ordered-dithering-and-output-quantization)
- **[XIII. GPU Compute Primitives](#xiii-gpu-compute-primitives)**
  - [GPU Compute Algorithm Primitives](#gpu-compute-algorithm-primitives)
  - [Wave and Subgroup Intrinsics](#wave-and-subgroup-intrinsics)
  - [GPU Radix Sort](#gpu-radix-sort)
  - [Parallel Bitonic Sort](#parallel-bitonic-sort)
  - [Cooperative Matrix and Tensor Core Operations](#cooperative-matrix-and-tensor-core-operations)
  - [GPU Spatial Data Structures](#gpu-spatial-data-structures)
  - [Sampling Techniques](#sampling-techniques)
  - [Reversed-Z and Logarithmic Depth](#reversed-z-and-logarithmic-depth)




---

## X. Texturing and Surface Detail

Texturing adds high-frequency surface detail without increasing geometric complexity. This category covers the full texture pipeline from storage (virtual texturing, sparse/tiled resources, bindless descriptors for large texture arrays) through surface detail encoding (normal maps, parallax occlusion mapping, normal-map blending) to specialized formats (HDR encoding, multi-channel SDF font atlases). Bindless texturing belongs here because it is fundamentally a texturing scalability solution — it removes the per-draw descriptor set limit that constrains material variety.

### Texture Mapping and Surface Detail

These techniques add geometric and material detail to surfaces without increasing mesh polygon count. Normal mapping is the universal baseline; parallax occlusion mapping adds view-dependent depth at the cost of more texture samples; triplanar and stochastic tiling solve the UV parameterisation and repetition problems that arise at scale. At the other end of specificity, MSDF font rendering and decal projection solve narrower problems (scalable text, dynamic surface markings) that appear across many engine systems. The common thread is that all of these techniques manipulate UV coordinates, surface normals, or both — they do not change actual geometry.

**Shader stage**: Normal mapping, POM, triplanar, stochastic tiling, and MSDF rendering all execute in the **fragment shader** — they are per-pixel operations that sample textures and compute a modified normal or alpha. Deferred decals write modified albedo/normal/roughness values into the **G-buffer** in a dedicated **fragment shader** pass drawn after the main geometry pass.

#### Normal Mapping
Encodes surface microgeometry as a per-texel normal vector in tangent space (stored in an RGB texture), which is transformed to world space at runtime to perturb lighting calculations without adding geometry.

**Use cases** — ubiquitous; the baseline surface detail technique for all rasterised rendering.  
**Key variants** — object-space normals (simpler transform, no tangent requirement); world-space normals; mikktspace tangent convention (required for consistent results across tools).  
**Reference** — [Blinn (1978): Simulation of Wrinkled Surfaces](https://doi.org/10.1145/965139.507101)

#### Parallax Occlusion Mapping (POM)
Ray marches in tangent space along the view vector through a heightmap to find the intersection with the surface height profile, producing self-occlusion and correct perspective foreshortening of surface relief.

**Use cases** — deep surface relief: brick mortar, stone, worn metal; significantly more convincing than parallax mapping.  
**Limitations** — silhouette is not displaced (the mesh silhouette remains flat); expensive at grazing angles; 8–32 steps typical.  
**Reference** — [Tatarchuk: Practical Parallax Occlusion Mapping (GDC 2006)](https://advances.realtimerendering.com/s2006/)

#### Triplanar Mapping
Projects textures from three axis-aligned planes (XY, XZ, YZ) and blends them based on world-space normal, eliminating UV stretching on organic or curved surfaces.

**Use cases** — terrain, caves, organic meshes, procedurally generated geometry where UVs would stretch.  
**Limitations** — 3× texture samples; blend region at 45° produces a visible seam without smooth blending.

#### Stochastic Texture Tiling (Texture Bombing)
Randomly offsets and rotates texture tiles at a per-pixel hashed frequency, eliminating the visible periodic repetition of tiled textures without requiring large texture atlases.

**Use cases** — ground surfaces, terrain materials, any large repeated texture; near-zero performance overhead.  
**Reference** — [Bitterli: Nonrepeating Textures (2017)](https://benedikt-bitterli.me/histogram-tiling/)

#### UV-Space SDF Text Rendering (MSDF)
Encodes a glyph's contour as a multi-channel signed distance field, enabling crisp anti-aliased text rendering at arbitrary scale from a small texture, including correct sharp corners.

**Use cases** — GPU text rendering in UIs, HUDs, game engines, any application needing scalable vector text on GPU.  
**Reference** — [Chlumský: Multi-channel Signed Distance Field (2015)](https://github.com/Chlumsky/msdfgen)

#### Decal Projection
Clips a decal quad to the underlying geometry by testing against a depth buffer (screen-space decals) or by rendering into the GBuffer (deferred decals), enabling dynamic surface markings without modifying meshes.

**Use cases** — bullet holes, dirt/scorch marks, blood splatter, graffiti, tyre tracks.  
**Key variants** — screen-space decals (simpler, artifacts at steep angles); deferred decals (correct GBuffer write, depth tested).  
**Reference** — [GPU Pro 2: Deferred Decals](https://gpupro.blogspot.com/)

---

### Normal and HDR Encoding

G-Buffers and light probe atlases must store normals and HDR values in compact, GPU-friendly formats. Naïve storage (3×`RGBA16F` per normal) wastes bandwidth; careful encoding halves or quarters the footprint with negligible quality loss.

**Shader stage**: encode in the **geometry pass fragment shader** (G-Buffer write); decode in the **deferred lighting fragment or compute shader**.

#### Octahedral Normal Encoding
Maps the unit normal sphere to a 2D unit square via octahedral projection, encoding it in two 16-bit (or 8-bit) channels. Provably the most uniform encoding with respect to angular error per bit; no hemisphere ambiguity; decode requires only a `sign()` and normalise.

```glsl
// Encode: unit normal → octahedral [−1, 1]²
vec2 oct_encode(vec3 n) {
    vec2 p = n.xy / (abs(n.x) + abs(n.y) + abs(n.z));
    return (n.z < 0.0) ? (1.0 - abs(p.yx)) * sign(p) : p;
}

// Decode: octahedral [−1, 1]² → unit normal
vec3 oct_decode(vec2 e) {
    vec3 v = vec3(e.x, e.y, 1.0 - abs(e.x) - abs(e.y));
    if (v.z < 0.0) v.xy = (1.0 - abs(v.yx)) * sign(v.xy);
    return normalize(v);
}
```

**Use cases** — G-Buffer normal encoding in `RG16F` (saves one channel vs RGB16F); light probe normal maps; bent normal storage.  
**Reference** — [Meyer et al.: Survey of Efficient Representations for Independent Unit Vectors (JCGT 2010)](http://jcgt.org/published/0003/02/01/)

#### Spheremap (Lambert Azimuthal) Encoding
A cheaper alternative to octahedral: encodes the normal as a 2D point via the Lambert azimuthal equal-area projection. Slightly lower quality than octahedral but simpler arithmetic.

```glsl
vec2 spheremap_encode(vec3 n) {
    return n.xy / sqrt(8.0 * n.z + 8.0) + 0.5;
}
vec3 spheremap_decode(vec2 enc) {
    vec2 f = enc * 4.0 - 2.0;
    float r = dot(f, f);
    return vec3(f * sqrt(1.0 - r / 4.0), 1.0 - r / 2.0);
}
```

**Reference** — [Crytek: A Non-Integer Power Efficient Floating-Point Compressor (2009)](https://www.crytek.com/)

#### RGBM HDR Encoding
Stores HDR colour as an RGB mantissa plus a shared exponent in the alpha channel: `RGB = colour / (maxVal * M)`, `A = M`. Encodes values up to `maxVal` (typically 6 or 8) in a standard RGBA8 texture.

```glsl
vec4 rgbm_encode(vec3 color, float max_range) {
    color /= max_range;
    float M = max(max(color.r, color.g), max(color.b, 1e-6));
    M = ceil(M * 255.0) / 255.0;
    return vec4(color / M, M);
}
vec3 rgbm_decode(vec4 rgbm, float max_range) {
    return rgbm.rgb * rgbm.a * max_range;
}
```

**Use cases** — HDR environment map storage in RGBA8 textures (legacy mobile, WebGL 1); baked lightmap HDR storage; probe atlas.  
**Limitations** — `M` clamped to range; introduces banding at low values. Prefer `R11G11B10_UFLOAT` on hardware that supports it.

#### R11G11B10 Unsigned Float
The Vulkan format `VK_FORMAT_B10G11R11_UFLOAT_PACK32` packs 11 bits of mantissa for R/G and 10 bits for B into a 32-bit uint, covering ~[0, 65504] with no sign bit (HDR only, no negatives). Uses the same exponent for all three channels, giving excellent precision for non-negative HDR colour.

**Use cases** — HDR framebuffer, light accumulation buffer, bloom targets; preferred over RGBM on all desktop hardware.  
**Reference** — [Quilez: Packing Floats](https://iquilezles.org/articles/floatpacking/)

#### LogLuv HDR Encoding
Encodes luminance logarithmically and chrominance (u′v′) linearly in an RGBA8 texture, exploiting the logarithmic response of human vision. Better perceptual uniformity than RGBM for wide dynamic range; higher decode cost.

**Reference** — [Ward: LogLuv Encoding for Full Gamut, High Dynamic Range Images (JGTE 1998)](https://dl.acm.org/doi/10.1145/271895.271938)

---

### Normal Map Blending

A surface often needs multiple normal maps combined: a tiling detail normal map on top of a base surface normal, a decal normal overlaid on both, a terrain blend between rock and soil normals. Simple addition of normal vectors breaks normalization and produces incorrect lighting. Two well-established blending methods handle this correctly.

**Shader stage**: normal blending runs in the **fragment shader** immediately before the lighting evaluation, producing a single world-space normal from two or more tangent-space inputs.

#### UDN Blending (Unreal Detail Normals)
Designed for overlaying a detail normal map `n2` onto a base normal map `n1`. Whiteout-extends `n1` (treats it as the dominant base) and re-normalizes. Cheap: one normalize, no cross products.

```glsl
// UDN blend — n1 is base (low-frequency), n2 is detail (high-frequency)
// Both normals in tangent space, range [−1, 1]
vec3 blend_normals_udn(vec3 n1, vec3 n2) {
    // Treat n1 as a perturbation of (0,0,1); add n2 as a second perturbation
    return normalize(vec3(n1.xy + n2.xy, n1.z));
}
```

**Limitation**: UDN assumes `n1.z > 0` (normals pointing generally upward in tangent space). It breaks for normals with large angles from the tangent plane (e.g., under strong displacement). For those cases use whiteout blending.

#### Whiteout Blending
Treats both normals symmetrically, giving them equal influence. More correct for arbitrary normal angles; slightly more expensive (one extra multiply).

```glsl
// Whiteout blend — symmetric, works for large angular differences
vec3 blend_normals_whiteout(vec3 n1, vec3 n2) {
    return normalize(vec3(n1.xy + n2.xy, n1.z * n2.z));
}
```

**When to use**: UDN for detail normal maps at small scales (pores, scratches, fabric weave); whiteout for blending between two equal-weight materials (terrain rock/grass normals, decal normals at steep angles).  
**Reference** — [Mikkelsen: Blending in Detail (2010)](https://blog.selfshadow.com/publications/blending-in-detail/)

#### Reoriented Normal Mapping (RNM)
Blends a detail normal map whose local Z axis is rotated to align with the base normal, using a full rotation. The most correct method: preserves the correct normal frame for large base-normal variations (e.g., curved walls), at the cost of a `cross` and two normalizations.

```glsl
// RNM blend — full rotation of n2 into n1's frame
vec3 blend_normals_rnm(vec3 n1, vec3 n2) {
    vec3 t  = n1 + vec3(0.0, 0.0, 1.0);
    vec3 u  = n2 * vec3(-1.0, -1.0, 1.0);
    return normalize(t * dot(t, u) - u * t.z);
}
```

**Reference** — [Colin Barré-Brisebois: Practical Layered Materials in Call of Duty: Infinite Warfare (SIGGRAPH 2016)](https://advances.realtimerendering.com/s2016/)

#### Decal Normal Compositing
When applying a decal normal (in world space or object space) on top of a surface normal, transform both to the same space, blend with whiteout, then transform the result to world space for lighting.

```glsl
// Decal normal (world space) blended onto surface normal (world space)
// weight: 0 = full surface, 1 = full decal influence
vec3 composite_decal_normal(vec3 surface_n, vec3 decal_n, float weight) {
    vec3 blended = blend_normals_whiteout(surface_n, mix(vec3(0,0,1), decal_n, weight));
    return normalize(blended);
}
```

---

### Parallax Occlusion Mapping

Parallax Occlusion Mapping (POM) simulates geometric surface depth using only a heightmap — producing correct self-occlusion, silhouette parallax, and self-shadowing at the cost of multiple texture samples in the fragment shader. It is superior to basic parallax mapping (which does a single offset) and to tessellation-based displacement (which requires extra geometry), and is the standard technique for brick walls, stone surfaces, and worn metal in AAA games.

**Shader stage**: runs entirely in the **fragment shader** using the tangent-space view direction.

#### Iterative Ray March (Linear + Binary Refinement)
March the view ray through the height field until it dips below the surface, then binary-search for the precise intersection.

```glsl
// Parallax Occlusion Mapping — fragment shader
layout(set=0, binding=0) uniform sampler2D albedo_map;
layout(set=0, binding=1) uniform sampler2D height_map;
layout(set=0, binding=2) uniform sampler2D normal_map;

uniform float pom_depth_scale;    // world-space depth range (e.g. 0.05)
uniform int   pom_min_steps;      // e.g. 8
uniform int   pom_max_steps;      // e.g. 32

layout(location=0) in vec3 v_view_ts;   // view direction in tangent space (unnormalized)
layout(location=1) in vec2 v_uv;

// Compute number of steps based on viewing angle (fewer for steep views)
int num_steps(vec3 V_ts) {
    float NdotV = abs(dot(vec3(0,0,1), normalize(V_ts)));
    return int(mix(float(pom_max_steps), float(pom_min_steps), NdotV));
}

vec2 pom_uv(vec2 uv, vec3 V_ts) {
    int   steps    = num_steps(V_ts);
    vec3  V_norm   = normalize(V_ts);
    // Step direction in UV space (XY of view in tangent space / Z)
    vec2  uv_step  = V_norm.xy * pom_depth_scale / (float(steps) * V_norm.z);
    float h_step   = 1.0 / float(steps);

    float ray_h    = 1.0;   // ray height (1 = surface, 0 = deepest)
    float surf_h   = texture(height_map, uv).r;
    vec2  cur_uv   = uv;

    // Linear march until ray goes below surface
    for (int i = 0; i < steps; ++i) {
        if (ray_h < surf_h) break;
        cur_uv  -= uv_step;
        ray_h   -= h_step;
        surf_h   = texture(height_map, cur_uv).r;
    }

    // Binary refinement (4 iterations)
    vec2  prev_uv  = cur_uv + uv_step;
    float prev_h   = ray_h  + h_step;
    for (int b = 0; b < 4; ++b) {
        vec2  mid_uv = (cur_uv + prev_uv) * 0.5;
        float mid_h  = (ray_h  + prev_h)  * 0.5;
        float mid_s  = texture(height_map, mid_uv).r;
        if (mid_h < mid_s) { prev_uv = mid_uv; prev_h = mid_h; }
        else               { cur_uv  = mid_uv; ray_h  = mid_h; }
    }
    return (cur_uv + prev_uv) * 0.5;
}

void main() {
    vec2  disp_uv  = pom_uv(v_uv, v_view_ts);
    vec4  albedo   = texture(albedo_map, disp_uv);
    vec3  N_ts     = texture(normal_map, disp_uv).xyz * 2.0 - 1.0;
    // Continue with standard PBR lighting using displaced UV and normal
}
```

#### POM Self-Shadowing
After finding the displaced surface point, march a second ray toward the light in tangent space. If the ray goes above the surface height, the point is self-shadowed by the heightmap.

```glsl
float pom_shadow(vec2 disp_uv, vec3 L_ts, float surface_height) {
    vec3  L_norm   = normalize(L_ts);
    if (L_norm.z <= 0.0) return 0.0;  // light below surface tangent plane

    vec2  uv_step  = L_norm.xy * pom_depth_scale / (16.0 * L_norm.z);
    float h_step   = surface_height / 16.0;

    float shadow   = 0.0;
    float ray_h    = surface_height;
    vec2  cur_uv   = disp_uv;

    for (int i = 0; i < 16; ++i) {
        cur_uv  += uv_step;
        ray_h   += h_step;
        float h  = texture(height_map, cur_uv).r;
        shadow   = max(shadow, (h - ray_h) * (1.0 - float(i) / 16.0));
    }
    return clamp(shadow * 10.0, 0.0, 1.0);  // 0 = lit, 1 = fully shadowed
}
```

**Reference** — [Tatarchuk: Practical Parallax Occlusion Mapping for Highly Detailed Surface Rendering (SIGGRAPH 2006)](https://dl.acm.org/doi/10.1145/1185657.1185830); [Welsh: Parallax Mapping with Offset Limiting (2004)](https://www.infiscape.com/doc/parallax_mapping.pdf)

---

### Virtual Texturing and Sparse Virtual Textures

Virtual texturing (megatextures, sparse virtual textures) decouples texture coordinate space from GPU memory residency. Each mesh uses a large virtual UV space (e.g., 16k×16k per unique surface); the actual texture data is divided into fixed-size pages (128×128 or 256×256 texels) stored in a streaming atlas. Only pages visible in the current frame are resident; the rest remain on disk or host memory.

The pipeline has three components: (1) a **feedback pass** identifies which virtual pages are accessed; (2) a **streaming system** (CPU) loads missing pages and uploads them to the physical atlas; (3) a **final render pass** translates virtual UV to physical atlas UV using an indirection (page table) texture.

**Shader stage**: feedback writes in the **fragment shader** of a low-resolution prepass. Page table lookup and atlas sampling run in the **fragment shader** of the main render pass.

#### Page Table Texture (Indirection Texture)
A 2D texture where each texel represents one virtual page. The texel value encodes the physical atlas page coordinates where that virtual page's data lives. Resolution = `(virtual_size / page_size)²` — for 16k virtual / 256 page = 64×64 page table.

```glsl
layout(set=0, binding=0) uniform sampler2D page_table;    // RG8: physical page XY coords
layout(set=0, binding=1) uniform sampler2DArray phys_atlas; // tiled physical pages

uniform vec2 virtual_size;   // e.g. 16384.0
uniform vec2 page_size;      // e.g. 256.0
uniform vec2 atlas_size;     // physical atlas size in pages

vec4 virtual_texture_sample(vec2 virtual_uv) {
    // Which page does this UV fall in?
    vec2 page_uv    = virtual_uv * virtual_size / page_size;
    vec2 page_coord = floor(page_uv);
    vec2 texel_frac = fract(page_uv);   // position within page

    // Look up physical page location from indirection texture
    vec2 page_table_uv = (page_coord + 0.5) / (virtual_size / page_size);
    vec2 phys_page     = texture(page_table, page_table_uv).rg * 255.0; // decode RG8

    // Compute UV into physical atlas
    vec2 atlas_uv = (phys_page + texel_frac) / atlas_size;
    return texture(phys_atlas, vec3(atlas_uv, 0.0));
}
```

#### Feedback Buffer (Page Request)
A low-resolution framebuffer (1/4 or 1/8 resolution) where each fragment writes the virtual page coordinates it needs. The CPU reads this buffer asynchronously (with one frame latency via `vkGetQueryPoolResults` or persistent-mapped staging buffer) and schedules missing pages for streaming.

```glsl
// Feedback fragment shader — low-res prepass
layout(location=0) out uvec2 page_request;  // R16G16_UINT: virtual page XY

void main() {
    vec2  page_uv   = virtual_uv * virtual_size / page_size;
    uvec2 page_coord = uvec2(floor(page_uv));
    page_request    = page_coord;   // CPU reads this to know what to stream
}
```

**LOD selection**: incorporate the texture MIP level into the page request so coarser pages are streamed for distant surfaces. The MIP level maps directly to a coarser page table level.

#### Page Upload (CPU → GPU)
When the streaming system identifies a missing page, it copies from disk/host to a staging buffer and calls `vkCmdCopyBufferToImage` to upload the page into the correct slot in the physical atlas. The page table texture is then updated to point to the new location.

**Use cases** — id Tech 7 (Doom Eternal), UE5 Virtual Textures, Unity Adaptive Probe Volumes; any open-world game with unique per-surface texturing at scale.  
**Reference** — [id Software: MegaTexture (id Tech 4/5)](https://en.wikipedia.org/wiki/MegaTexture); [UE5 Virtual Texturing Documentation](https://docs.unrealengine.com/5.0/en-US/virtual-texturing-in-unreal-engine/)

---

### Sparse Texture and Tiled Resources

Standard textures allocate a contiguous block of GPU memory for every mip level — even levels that are never sampled at runtime (distant objects never need mip 0). **Sparse textures** (`VK_EXT_sparse_binding`, OpenGL `ARB_sparse_texture`, Direct3D tiled resources) allocate only the tiles that are actually accessed: 64KB tiles are committed on demand as the camera moves. This enables textures of arbitrary logical size (16k×16k or larger) backed by a physical memory budget orders of magnitude smaller, and is foundational to **Virtual Texturing** (§49) at the hardware level.

**Shader stage**: sparse texture sampling uses the **fragment shader** with the `sparseTextureSparseARB` (GLSL) or `Vulkan OpImageSparseSampleImplicitLod` (SPIR-V) instruction, which returns both the texel colour and a **residency code** indicating whether the tile was resident.

#### Vulkan Sparse Texture Setup (CPU)
Creating a sparse texture requires `VK_IMAGE_CREATE_SPARSE_RESIDENCY_BIT`. Memory is bound per-tile using `vkQueueBindSparse` rather than `vkBindImageMemory`.

```c
// Sparse image creation
VkImageCreateInfo sparse_ci = {
    .flags       = VK_IMAGE_CREATE_SPARSE_RESIDENCY_BIT |
                   VK_IMAGE_CREATE_SPARSE_BINDING_BIT,
    .format      = VK_FORMAT_R8G8B8A8_SRGB,
    .extent      = { 16384, 16384, 1 },
    .mipLevels   = 15,
    .usage       = VK_IMAGE_USAGE_SAMPLED_BIT | VK_IMAGE_USAGE_TRANSFER_DST_BIT,
};
vkCreateImage(device, &sparse_ci, NULL, &sparse_image);

// Bind individual tiles after the camera has determined which are needed
VkSparseImageMemoryBind tile_bind = {
    .subresource = { VK_IMAGE_ASPECT_COLOR_BIT, mip_level, 0 },
    .offset      = { tile_x * tile_width, tile_y * tile_height, 0 },
    .extent      = { tile_width, tile_height, 1 },
    .memory      = tile_memory_allocation,
    .memoryOffset= 0,
};
VkBindSparseInfo bind_info = {
    .imageBindCount = 1,
    .pImageBinds    = &(VkSparseImageMemoryBindInfo){ sparse_image, 1, &tile_bind },
};
vkQueueBindSparse(sparse_queue, 1, &bind_info, VK_NULL_HANDLE);
```

#### Residency Check in Fragment Shader
The `sparseTexelResidentARB` function tests the residency code returned by the sparse sample. Unresident tiles can fall back to a lower mip level (always fully resident) or a solid colour placeholder.

```glsl
#extension GL_ARB_sparse_texture2 : require

layout(set=0, binding=0) uniform sampler2D sparse_tex;

void main() {
    vec4  color;
    int   residency = sparseTexture2DARB(sparse_tex, uv, color);
    if (!sparseTexelResidentARB(residency)) {
        // Tile not resident — fall back to coarse mip (always resident)
        color = texture(sparse_tex, uv, max_mip_bias);
    }
    // Record the UV for feedback streaming (which tile needs to be loaded)
    if (!sparseTexelResidentARB(residency)) {
        imageStore(feedback_buf, ivec2(gl_FragCoord.xy >> 3), vec4(uv, mip_level, 1.0));
    }
    frag_color = color;
}
```

#### Mip Tail
The mip tail (smallest mip levels, below the tile granularity) is always packed into a single non-sparse allocation and committed at image creation time, guaranteeing a fallback for any unresident tile.

**Reference** — [Vulkan: Sparse Resources](https://registry.khronos.org/vulkan/specs/1.3/html/chap30.html#sparsememory); [Crassin & Green: Sparse Virtual Textures (GDC 2011)](https://silverspaceship.com/src/svt/)

---

### Bindless Texturing and Descriptor Indexing

Traditional Vulkan pipelines bind a fixed set of textures per draw call via descriptor sets — one `VkDescriptorSet` update per material, one `vkCmdBindDescriptorSets` per draw. With many unique materials this becomes the CPU bottleneck. `VK_EXT_descriptor_indexing` (core in Vulkan 1.2 as `descriptorIndexing`) allows a shader to index into an **unbounded array** of textures using a runtime integer — eliminating per-draw rebinding entirely and enabling the GPU-driven rendering (§32) and visibility buffer (§21) pipelines to work with heterogeneous material sets.

**Shader stage**: bindless arrays are declared in the **fragment shader** (or any shader stage). The index may be non-uniform (different across fragments in the same draw), which requires the `nonuniformEXT` qualifier to ensure correct GPU memory access.

#### Unbounded Descriptor Array
Declare a descriptor array with no fixed size using `[]`. The binding must use the `UPDATE_AFTER_BIND` flag and the array must be allocated with a large enough descriptor pool.

```glsl
#extension GL_EXT_nonuniform_qualifier : require

// Bindless texture array — all scene textures bound once, indexed by material ID
layout(set=0, binding=0) uniform sampler2D tex_array[];       // unbounded albedo/normal/etc.
layout(set=0, binding=1) uniform sampler2D normal_array[];
layout(set=0, binding=2) uniform sampler2D roughness_array[];

// Per-draw or per-instance material index (from InstanceSSBO or push constant)
layout(location=2) flat in uint v_material_id;

void main() {
    // nonuniformEXT: tells the GPU that material_id varies across fragments —
    // required for correct wave/quad execution when different pixels hit different textures
    vec4 albedo    = texture(tex_array[nonuniformEXT(v_material_id)],     uv);
    vec3 normal_ts = texture(normal_array[nonuniformEXT(v_material_id)],  uv).xyz * 2.0 - 1.0;
    float rough    = texture(roughness_array[nonuniformEXT(v_material_id)], uv).r;
    // ... continue with shading
}
```

**Without `nonuniformEXT`**: the GPU may read the texture index from lane 0 and use it for all lanes in the wave — producing wrong results when different fragments have different material IDs. This is a correctness issue, not a performance one.

#### Material SSBO + Bindless Index
In a GPU-driven renderer, the material index comes from the per-instance SSBO (§32) rather than a uniform. The fragment shader reads the instance data to find which textures to sample.

```glsl
struct MaterialRecord {
    uint albedo_idx;
    uint normal_idx;
    uint roughness_idx;
    uint emissive_idx;
    vec4 base_color_factor;
    float roughness_factor;
    float metallic_factor;
};
layout(set=1, binding=0) readonly buffer MaterialSSBO { MaterialRecord materials[]; };

layout(location=2) flat in uint v_instance_id;

void main() {
    MaterialRecord mat = materials[v_instance_id];
    vec4 albedo = texture(tex_array[nonuniformEXT(mat.albedo_idx)], uv)
                * mat.base_color_factor;
    // ...
}
```

#### Vulkan Setup for Bindless
On the CPU side, the descriptor pool and layout must be created with the correct flags:

```c
// Descriptor pool flag (allows descriptors updated after bind)
VkDescriptorPoolCreateInfo pool_info = {
    .flags = VK_DESCRIPTOR_POOL_CREATE_UPDATE_AFTER_BIND_BIT,
    ...
};
// Descriptor set layout binding flag
VkDescriptorBindingFlags binding_flags =
    VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT |
    VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT |
    VK_DESCRIPTOR_BINDING_VARIABLE_DESCRIPTOR_COUNT_BIT;
// Descriptor count: allocate the maximum number of textures in the scene
uint32_t max_textures = 65536;
```

**Use cases** — every GPU-driven renderer (Nanite, id Tech 7, Frostbite, Bevy); visibility buffer material evaluation (§21); any scene with > 16 unique textures.  
**Reference** — [Vulkan: VK_EXT_descriptor_indexing](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_descriptor_indexing.html); [Wicked Engine: Bindless Descriptors](https://wickedengine.net/2021/04/bindless-descriptors/)

---

### Multi-Channel SDF Font Rendering

GPU text rendering at arbitrary scale without rasterisation at each size requires precomputed **Signed Distance Field (SDF)** glyph atlases. A plain SDF (§2) produces rounded corners at high zoom; **Multi-Channel SDF (MSDF)**, developed by Viktor Chlumský (2016), encodes corner information in three independent colour channels, preserving sharp corners at all zoom levels. The fragment shader reconstructs the correct coverage by taking the median of the three channels.

**Shader stage**: atlas generation runs offline (CPU or compute). Runtime rendering runs in the **vertex shader** (billboard quad per glyph) and **fragment shader** (median SDF coverage evaluation).

#### MSDF Fragment Evaluation
```glsl
// MSDF glyph rendering — fragment shader
layout(set=0, binding=0) uniform sampler2D msdf_atlas;

uniform float px_range;      // distance range in pixels (e.g. 4.0 — set at atlas gen time)
uniform vec4  text_color;
uniform float gamma_correct; // 1.0 for sRGB output, 2.2 for linear

layout(location=0) in  vec2 v_uv;      // UV into glyph sub-rect of atlas
layout(location=1) in  float v_size;   // glyph rendered size in pixels
layout(location=0) out vec4 frag;

// Median of three values (MSDF corner reconstruction)
float median(float r, float g, float b) {
    return max(min(r, g), min(max(r, g), b));
}

void main() {
    vec3  msd     = texture(msdf_atlas, v_uv).rgb;
    float sd      = median(msd.r, msd.g, msd.b);  // signed distance (0.5 = on boundary)

    // Convert from [0,1] SDF value to pixel distance
    float px_dist = (sd - 0.5) * px_range * (v_size / float(textureSize(msdf_atlas, 0).x));

    // Anti-aliased coverage: smooth step over 1 pixel around the boundary
    float opacity  = clamp(px_dist + 0.5, 0.0, 1.0);

    // Optional: screen gamma for correct perceived weight
    opacity = pow(opacity, 1.0 / gamma_correct);

    frag = vec4(text_color.rgb, text_color.a * opacity);
}
```

#### Bold / Outline via SDF Bias
Adjust the threshold away from 0.5 to fatten or thin the glyph, or add a second threshold for an outline ring — all without regenerating the atlas.

```glsl
// Outline: inner edge at sd > outline_inner, outer at sd > outline_outer
float inner = clamp((sd - 0.5 + outline_width) * px_range + 0.5, 0.0, 1.0);
float outer = clamp((sd - 0.5)                 * px_range + 0.5, 0.0, 1.0);
vec4 out_col = mix(outline_color * inner, text_color, outer);
frag = out_col;
```

#### Glyph Atlas Layout
Each glyph occupies a rectangular cell in the atlas with a padding of `px_range/2` pixels around the tight bounding box. At runtime, a CPU-side text shaper (HarfBuzz) produces glyph IDs and advances; a vertex buffer is filled with one quad per glyph, each with UV coordinates into the atlas cell.

**Reference** — [Chlumský: Shape Decomposition for Multi-Channel Distance Fields (Master's Thesis, 2015)](https://github.com/Chlumsky/msdfgen); [msdfgen on GitHub](https://github.com/Chlumsky/msdfgen)

---


---

## XI. Transparency, Reflections, and Compositing

These techniques share a dependency on rendering order or on compositing multiple layers of geometry. Order-independent transparency and stochastic alpha solve the depth-sorting problem for transparent surfaces; planar and screen-space reflections require a second rendering pass or screen-space approximation; decals and outlines composite over already-rasterized geometry; octahedral impostors and 3D Gaussian Splatting represent geometry as screen-space splats. The unifying theme is that none of these fit cleanly into a single opaque forward or deferred pass — all require some form of multi-layer or order-sensitive compositing.

### Order-Independent Transparency

Alpha-blended transparency requires back-to-front draw order — the Porter-Duff OVER operator is not commutative. Sorting transparent objects on the CPU is expensive and incorrect for intersecting or overlapping geometry. OIT techniques eliminate the sort requirement entirely at the cost of additional memory or rendering passes.

**Shader stage**: The accumulation pass runs in the **fragment shader** writing to special render targets or SSBOs. The resolve pass runs as a full-screen **fragment** or **compute shader**.

#### Weighted Blended OIT (McGuire & Bavoil)
Accumulates alpha-weighted colour into one RGBA16F render target and alpha coverage into one R16F target in a single forward pass, then resolves by dividing. Approximates correct transparency for most scenes without a sort.

**Use cases** — the industry-standard OIT technique; used in Filament, Godot 4, Three.js. Correct for light-to-medium transparency overlap; breaks down for high-opacity stacked surfaces.  
**Limitations** — an approximation: very opaque or deep transparency stacks produce incorrect results; colour bleeding at object boundaries.  
**Reference** — [McGuire & Bavoil: Weighted Blended OIT (I3D 2013)](http://jcgt.org/published/0002/02/09/)

#### Per-Pixel Linked Lists (A-Buffer)
Each fragment atomically appends its depth, colour, and alpha to a per-pixel linked list in an SSBO, with a head pointer per pixel stored in an image. A resolve pass reads and sorts each pixel's list back-to-front and blends in correct order.

**Use cases** — exact OIT for any scene; used when WBOIT approximation is unacceptable (glass, coloured liquids).  
**Limitations** — unbounded SSBO memory (must over-provision); fragment atomic contention on hot pixels; resolve sort cost scales with overdraw depth.  
**Reference** — [Yang et al.: Real-Time Concurrent Linked List Construction on the GPU (EGSR 2010)](https://doi.org/10.1111/j.1467-8659.2010.01725.x)

#### Moment-Based OIT
Stores the first four power moments of the depth distribution per pixel (encoded compactly into two RGBA16F targets), then reconstructs a transmittance curve at resolve time using moment inversion — a principled approximation between WBOIT and per-pixel linked lists in both cost and accuracy.

**Use cases** — high-quality transparency where WBOIT fails but per-pixel linked lists are too expensive; hair rendering.  
**Reference** — [Münstermann et al.: Moment-Based OIT (I3D 2018)](http://jcgt.org/published/0007/03/02/)

---

### Stochastic Rendering

Stochastic rendering uses random or blue-noise per-pixel decisions that are perceptually smooth after temporal accumulation by TAA. The key insight: if per-pixel noise has a blue-noise spatial distribution and is uncorrelated across frames, TAA's temporal accumulation converges to the correct average faster and with fewer ghosting artefacts than correlated dithering patterns.

**Shader stage**: the random decision runs in the **vertex or fragment shader**; the result is smoothed by the TAA pass in post-processing.

#### Alpha Dithering (Stochastic Transparency)
Replaces alpha blending with a 1-bit per-pixel stochastic accept/reject: if `rand(px, frame) < alpha`, keep the fragment; otherwise discard. No blending, no sorting, full depth-correct transparency — at the cost of per-pixel noise that TAA must clean up. Used by Unreal Engine for foliage LOD crossfade and Nanite fallback geometry.

```glsl
// Stochastic transparency — fragment shader
layout(set=0, binding=0) uniform sampler2D blue_noise;
uniform float alpha;
uniform uint  frame_index;

void main() {
    ivec2 px = ivec2(gl_FragCoord.xy);
    float bn = texelFetch(blue_noise, px % 64, 0).r;
    // Temporal decorrelation via golden ratio
    bn = fract(bn + float(frame_index) * 0.61803398875);
    if (bn >= alpha) discard;  // stochastic discard — no blending needed
    // ... continue with opaque shading
}
```

**Use cases** — foliage LOD crossfade without z-sorting; alpha-tested hair with soft edges after TAA; cross-dissolve transitions.  
**Reference** — [Enderton et al.: Stochastic Transparency (I3D 2010)](https://dl.acm.org/doi/10.1145/1730804.1730838)

#### Stochastic LOD Selection
Randomly selects between discrete LOD levels with a probability proportional to the interpolation weight between levels, then lets TAA average across frames. Eliminates the sharp LOD pop without cross-fade geometry. The selection probability for LOD `n` ramps from 0 to 1 as the camera-to-object distance crosses the transition range.

```glsl
// In vertex shader or culling compute:
float lod_frac = smoothstep(lod_near, lod_far, distance_to_object);
float noise    = hash(object_id ^ frame_index);
uint  lod      = (noise < lod_frac) ? lod_high : lod_low;
// Submit the draw with the selected LOD mesh
```

**Use cases** — vegetation LOD, distant building LOD; requires TAA to be enabled; correct temporal statistics require blue-noise object-id-based hash.  
**Reference** — [Wihlidal: Stochastic LOD (GDC 2016)](https://advances.realtimerendering.com/s2016/)

#### Interleaved Sampling / Checkerboard Rendering
Renders only every other pixel each frame in a checkerboard pattern (or quarter-resolution for 2×2 tiles), reconstructing the missing pixels using a bilateral filter or the previous frame's reprojected pixels. Halves fragment shader cost; TAA accumulates to full resolution over two frames.

**Use cases** — half-rate AO, half-rate reflections, half-rate volumetric fog; standard technique for expensive per-pixel effects.  
**Reference** — [Drobot: Checkerboard Rendering (Digital Dragons 2017)](http://advances.realtimerendering.com/s2017/)

---

### Planar Reflections

A mirror or calm water surface reflects the world as seen from a camera position symmetrically reflected across the plane of the surface. The reflected image is rendered into a separate render target by rendering the entire scene from this reflected viewpoint, then sampled on the reflective surface using its clip-space coordinates. This produces pixel-perfect reflections including off-screen content, geometry detail, and correct parallax — at the cost of an additional scene render.

**Shader stage**: the reflection render uses the full scene render pipeline with the **view matrix replaced**. The sampling runs in the **fragment shader** of the reflective surface.

#### Reflected Camera Setup
The reflected view matrix is computed by reflecting the camera position and forward vector across the reflection plane and constructing a new look-at matrix. An oblique projection matrix clips geometry below the reflection plane to avoid rendering objects behind the mirror into the reflection.

```glsl
// CPU-side: compute reflected view matrix for plane (normal N, point P0)
// Reflect camera position: pos' = pos - 2 * (dot(pos - P0, N)) * N
// Reflect look direction: dir' = dir - 2 * dot(dir, N) * N
// Then standard lookAt(pos', pos' + dir', up')

// Oblique near-plane clipping (Lengyel 2007) — clip geometry behind the plane
// Modify projection matrix column 2 to match the reflection plane:
vec4 clip_plane = vec4(N, -dot(N, P0)); // in view space of reflected camera
// c = sign(clip_plane) * inv(proj)ᵀ row 3 / dot(clip_plane, that row)
// proj[2] = c - proj[3]  (replaces near plane with reflection plane)
```

#### Reflection Sampling in Fragment Shader
The reflective surface's fragment shader projects the fragment's world position into the reflected camera's clip space and samples the reflection render target using the resulting screen-space UV, adding a normal-map distortion for water ripples.

```glsl
layout(set=1, binding=0) uniform sampler2D reflection_rt;
layout(set=1, binding=1) uniform mat4 reflected_proj_view;
layout(set=1, binding=2) uniform sampler2D water_normal_map;

layout(location=0) in vec3 world_pos;
layout(location=0) out vec4 frag_color;

void main() {
    // Normal map distortion for water ripples
    vec2 ripple = texture(water_normal_map, world_pos.xz * 0.1).rg * 2.0 - 1.0;

    // Project world position into reflected camera clip space
    vec4 refl_clip = reflected_proj_view * vec4(world_pos, 1.0);
    vec2 refl_uv   = (refl_clip.xy / refl_clip.w) * 0.5 + 0.5;
    refl_uv       += ripple * 0.02;  // distort by ripple normal

    vec3 reflection = texture(reflection_rt, refl_uv).rgb;

    // Fresnel blend: more reflection at grazing angles
    float fresnel = pow(1.0 - max(0.0, dot(normalize(cam_pos - world_pos),
                                           vec3(0,1,0))), 5.0);
    frag_color = vec4(mix(water_color, reflection, fresnel), 1.0);
}
```

**Use cases** — calm water, mirrors, polished floors, portal surfaces; used in Source Engine, Godot, and Unity for water rendering; combined with SSR for rough surfaces.  
**Limitations** — doubles draw call count; geometry below the plane must be excluded or the render double-renders; does not work for curved reflectors.  
**Reference** — [Lengyel: Oblique View Frustum Depth Projection (JGTE 2007)](http://www.terathink.com/lengyel/GPUGems2/GPUGems2_ch42.pdf); [Valve: Water in Half-Life 2 (GDC 2006)]

---

### Screen-Space Reflections

Screen-Space Reflections (SSR) ray-marches the reflected ray in screen space — sampling the depth buffer to find intersections — then reads the reflected colour from the colour buffer. It is cheap for rough reflections where the approximation's artifacts (missing off-screen geometry, incorrect self-occlusion) are hidden by the blur, and it complements WSRC (§84) and IBL (§7) which lack high-frequency surface detail.

**Shader stage**: runs as a **compute** pass (one thread per pixel) on the G-buffer and previous-frame colour buffer.

#### Linear Ray March
March the reflected ray in clip space in fixed steps, comparing the ray depth against the depth buffer at each step.

```glsl
// SSR — linear ray march (basic)
layout(set=0, binding=0) uniform sampler2D depth_buf;
layout(set=0, binding=1) uniform sampler2D color_buf;     // previous frame
layout(set=0, binding=2) uniform sampler2D normal_buf;
layout(set=0, binding=3, rgba16f) uniform image2D ssr_out;

uniform mat4 proj;
uniform mat4 inv_proj;
uniform mat4 inv_view;
uniform int  num_steps;       // e.g. 64
uniform float step_size;      // NDC units per step, e.g. 0.02
uniform float thickness;      // depth tolerance (e.g. 0.1)

vec3 view_from_depth(vec2 uv, float depth) {
    vec4 ndc = vec4(uv * 2.0 - 1.0, depth * 2.0 - 1.0, 1.0);
    vec4 view = inv_proj * ndc;
    return view.xyz / view.w;
}

void main() {
    ivec2 px    = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv    = (vec2(px) + 0.5) / vec2(imageSize(ssr_out));

    float depth = texture(depth_buf, uv).r;
    vec3  N_vs  = texture(normal_buf, uv).xyz * 2.0 - 1.0;  // view-space normal
    vec3  P_vs  = view_from_depth(uv, depth);
    vec3  V_vs  = normalize(-P_vs);
    vec3  R_vs  = reflect(-V_vs, N_vs);

    vec4  hit_color = vec4(0.0);
    float hit_mask  = 0.0;

    vec3  ray_pos = P_vs;
    for (int i = 0; i < num_steps; ++i) {
        ray_pos += R_vs * step_size;

        // Project back to screen UV
        vec4  clip  = proj * vec4(ray_pos, 1.0);
        vec3  ndc   = clip.xyz / clip.w;
        vec2  s_uv  = ndc.xy * 0.5 + 0.5;
        if (any(lessThan(s_uv, vec2(0.0))) || any(greaterThan(s_uv, vec2(1.0)))) break;

        float s_depth   = texture(depth_buf, s_uv).r;
        vec3  s_pos_vs  = view_from_depth(s_uv, s_depth);

        float ray_depth = -ray_pos.z;   // view-space depth (positive forward)
        float surf_depth = -s_pos_vs.z;

        if (ray_depth > surf_depth && ray_depth < surf_depth + thickness) {
            // Fade at screen edges and steep angles
            float edge_fade = 1.0 - smoothstep(0.8, 1.0, max(abs(ndc.x), abs(ndc.y)));
            float angle_fade = max(0.0, dot(R_vs, -normalize(P_vs)));
            hit_color = texture(color_buf, s_uv) * edge_fade * angle_fade;
            hit_mask  = 1.0;
            break;
        }
    }
    imageStore(ssr_out, px, hit_color);
}
```

#### Hi-Z Ray March
Replace linear stepping with **hierarchical Z traversal**: advance the ray to the next tile boundary using the coarser mip levels of a pre-built Hi-Z pyramid, then step down to finer mips when an intersection is found. This finds the first hit in O(log N) steps instead of O(N), enabling far longer ray distances.

```glsl
// Hi-Z SSR step (one iteration)
float hiz_trace_step(vec3 ray_pos_vs, vec3 ray_dir_vs, inout int mip) {
    vec4 clip   = proj * vec4(ray_pos_vs, 1.0);
    vec2 uv     = (clip.xy / clip.w) * 0.5 + 0.5;
    float z_ray = -ray_pos_vs.z;
    float z_hiz = textureLod(hiz_pyramid, uv, float(mip)).r;  // min-Z at this mip

    if (z_ray > z_hiz) {
        mip = max(mip - 1, 0);  // potential hit: descend mip
    } else {
        // Advance ray to next tile boundary at this mip level
        vec2  tile_size = 1.0 / vec2(textureSize(hiz_pyramid, mip));
        vec2  t_max     = (floor(uv / tile_size + 0.5) * tile_size - uv)
                        / (ray_dir_vs.xy / z_ray + 1e-6);
        float t_step    = min(t_max.x, t_max.y) + 1e-4;
        mip = min(mip + 1, max_mip);
        return t_step;
    }
    return 0.0;
}
```

**Roughness integration**: for rough surfaces, jitter the reflected ray by the GGX VNDF lobe (§29) and blur the SSR result with a radius proportional to roughness, then composite with IBL at the same roughness level.

**Reference** — [Stachowiak: Stochastic Screen-Space Reflections (SIGGRAPH 2015)](https://www.gdcvault.com/play/1022260); [Uludag: Hi-Z Screen-Space Cone-Traced Reflections (GPU Pro 5)](https://gpupro5.com/)

---

### Screen-Space Techniques

Screen-space techniques exploit the depth buffer and G-buffer data already produced by the main render pass to approximate effects that would otherwise require expensive ray casts or multi-pass rendering. Their defining characteristic — and their fundamental limitation — is that they can only see what is on screen: any occluder or reflector that has been culled or is off-screen is invisible to them. Used correctly, they deliver high-quality reflections (SSR), contact shadows, and subsurface scattering at a fraction of the cost of ray-traced equivalents. Used incorrectly, they produce distracting edge cutoffs and artefacts that immediately signal "screen-space" to an experienced viewer.

**Shader stage**: SSR and contact shadows run as full-screen **compute shaders** reading from depth and G-buffer. Screen-space SSS runs as a full-screen **fragment shader** or **compute shader** blur pass on the deferred lighting buffer, masked by a skin stencil. All of these are post-geometry, post-deferred-lighting passes; they have no access to geometry or vertex attributes directly.

#### SSR (Screen-Space Reflections)
Marches rays in screen space (along the Hi-Z pyramid) to find intersections with existing screen-space geometry, producing plausible reflections that are dynamically consistent with the scene.

**Use cases** — wet floors, puddles, polished surfaces — anywhere accurate real-time reflections are needed within view.  
**Key variants** — linear depth march (simple); Hi-Z hierarchical march (fast empty-space skip); cone-traced rough reflections; hybrid SSR + IBL blend.  
**Limitations** — cannot reflect off-screen surfaces; breaks at screen edges; silhouette gap at reflection horizon.  
**Reference** — [Morgan McGuire & Michael Mara: Efficient GPU Screen-Space Ray Tracing (JCGT 2014)](https://jcgt.org/published/0003/04/04/)

#### Screen-Space Contact Shadows
Marches a short ray in depth buffer space along the light direction from each fragment, producing sharp contact shadows at geometry intersections that shadow maps miss due to limited resolution.

**Use cases** — small crevices, character feet on ground, detailed prop shadows; supplements shadow maps at close range.  
**Limitations** — only a few texels of march; misses shadows from occluders not directly adjacent in screen space.  
**Reference** — [NVIDIA: Contact Shadows](https://developer.nvidia.com/blog/contact-shadows/)

#### Screen-Space Subsurface Scattering
Applies a separable 2D Gaussian blur in screen space to a skin cluster mask, approximating the diffuse light transport inside the skin volume without a volumetric simulation.

**Use cases** — real-time skin rendering in games (Jimenez 2015); character close-ups; the standard approach in Unreal and Unity.  
**Reference** — [Jimenez et al.: Separable Subsurface Scattering (CGF 2015)](https://iryoku.com/separable-sss/)

---

### Decal Rendering

Decals project textures onto existing geometry without modifying meshes — bullet holes, dirt, scorch marks, puddles, graffiti. The two main implementations differ in when they interact with the G-Buffer and how they handle depth testing.

**Shader stage**: Screen-space and deferred decals use a **fragment shader** drawing an oriented bounding box mesh. Clustered decals are evaluated within the deferred **lighting fragment or compute shader** without additional draw calls.

#### Screen-Space Decals
Draw a decal-volume OBB after the geometry pass. The fragment shader reconstructs world position from the depth buffer, projects into decal UV space, discards fragments outside the volume, and blends the decal colour over the existing framebuffer.

```glsl
// Screen-space decal fragment shader
layout(set=0, binding=0) uniform sampler2D scene_depth;
layout(set=0, binding=1) uniform sampler2D decal_albedo;
layout(set=0, binding=2) uniform DecalUB {
    mat4 inv_decal_model;   // world → decal local space
    mat4 inv_proj_view;
};

layout(location=0) in vec4 v_clip_pos;
layout(location=0) out vec4 frag_color;

void main() {
    vec2 screen_uv = (v_clip_pos.xy / v_clip_pos.w) * 0.5 + 0.5;
    float depth    = texture(scene_depth, screen_uv).r;
    vec4  world_p  = inv_proj_view * vec4(screen_uv * 2.0 - 1.0, depth, 1.0);
    world_p       /= world_p.w;
    vec3  local_p  = (inv_decal_model * world_p).xyz;

    if (any(greaterThan(abs(local_p), vec3(0.5)))) discard; // outside OBB
    vec2 decal_uv  = local_p.xz + 0.5;
    frag_color     = texture(decal_albedo, decal_uv);
}
```

**Use cases** — impact decals, paint marks; simple to implement; artifacts at steep surface angles (decal stretches along the view vector).  
**Reference** — [Sousa: CryEngine Deferred Decals (GDC 2010)](https://advances.realtimerendering.com/s2010/)

#### Deferred Decals
Draw decal OBBs after the geometry pass but before the lighting pass, writing modified albedo/normal/roughness into G-Buffer MRTs. Requires a stencil mask or depth-bias to prevent writing over decal-volume backfaces.

**Use cases** — decals that must affect normal mapping (puddles on rough concrete, bullet holes with displaced normals); correct deferred shading interaction.  
**Limitations** — requires access to G-Buffer as both input (depth for world position) and output (albedo/normal MRTs); requires a depth-write pass or deferred decal render pass.

#### Clustered Decals
Assign decals to frustum clusters (same as clustered lights), then evaluate all decals in the lighting compute pass by iterating the per-cluster decal list. No extra OBB draw calls; decal count scales with clustering budget.

**Use cases** — high decal counts (dozens to hundreds simultaneously) where OBB draw call overhead is prohibitive.  
**Reference** — [Schulz: Clustered Deferred and Forward+ (SIGGRAPH 2012)](https://www.cse.chalmers.se/~uffe/clustered_shading_preprint.pdf)

---

### Outline and Selection Rendering

Object outline rendering is universal in games and editors — selection highlight in Blender, enemy detection outlines in stealth games, UI focus rings. The algorithms range from simple stencil tricks to full-screen distance field passes.

**Shader stage**: Stencil outlines use **vertex + fragment** for the scaled re-render. JFA uses a **compute** or **fragment** shader for each flooding step (O(log N) passes). Edge detection runs as a full-screen **fragment** or **compute** pass.

#### Stencil Buffer Outline
Render the selected object into the stencil buffer (write 1). Re-render the object with a slightly scaled model matrix (scale by 1 + outline_width in view space) using a solid colour shader, discarding fragments where stencil = 1. The ring outside the object but inside the scaled version is the outline.

**Use cases** — cheap selection highlight; correct depth testing; aliased at low outline widths; width is model-scale not screen-pixel-scale.  
**Limitations** — outline width is geometry-dependent (nonuniform for complex objects); no sub-pixel width; does not work with skinned meshes without re-binding the skinned vertex buffer.

#### Jump Flood Algorithm (JFA) Outline
Renders the selected object into a seed image (object pixels = screen coordinate, background = sentinel). JFA then runs log₂(max_dim) passes: each pass reads 8 neighbours at step size 2^(pass) and keeps the nearest seed coordinate. The resulting Voronoi distance field gives each pixel the distance to the nearest object pixel — threshold to produce the outline.

```glsl
// JFA step — fragment or compute shader
layout(set=0, binding=0) uniform sampler2D jfa_input;
layout(set=0, binding=1) writeonly uniform image2D jfa_output;
layout(set=0, binding=2) uniform JfaUB { ivec2 step_size; };

void main() {
    ivec2 px   = ivec2(gl_FragCoord.xy);
    vec2  best = texelFetch(jfa_input, px, 0).xy;
    float best_dist = best.x < 0.0 ? 1e9 : distance(vec2(px), best);

    for (int dy = -1; dy <= 1; ++dy)
    for (int dx = -1; dx <= 1; ++dx) {
        if (dx == 0 && dy == 0) continue;
        ivec2 nb   = px + ivec2(dx, dy) * step_size;
        vec2  seed = texelFetch(jfa_input, nb, 0).xy;
        if (seed.x < 0.0) continue;
        float d    = distance(vec2(px), seed);
        if (d < best_dist) { best = seed; best_dist = d; }
    }
    imageStore(jfa_output, px, vec4(best, 0.0, 0.0));
}
// Resolve: distance < outline_px → outline colour
```

**Use cases** — anti-aliased outlines at arbitrary pixel width; Blender's object selection uses a variant; works correctly with any geometry including skinned characters.  
**Reference** — [Rong & Tan: Jump Flooding in GPU with Applications to Voronoi Diagram and Distance Transform (I3D 2006)](https://doi.org/10.1145/1111411.1111431)

#### Screen-Space Edge Detection Outline
Applies a Sobel or Laplacian operator to the depth buffer (and optionally the normal buffer) to find geometric edges; writes an edge mask used as the outline colour. Detects all scene edges, not just selected objects — useful for a "blueprint" or "sketch" visualisation of the full scene.

**Use cases** — NPR full-scene edge rendering; technical visualisation; debug normal/depth display; combining with cel shading for a toon-shaded edge pass.  
**Reference** — [GPU Gems 2 Ch19: Generic Refraction Simulation](https://developer.nvidia.com/gpugems/gpugems2/part-ii-shading-antialiasing-and-shadows/chapter-19-generic-refraction-simulation) | [Roberts: Edge Detection (NVIDIA)](https://developer.nvidia.com/gpugems/gpugems/part-iv-image-processing/chapter-24-fast-fourth-order-partial-differential-equations)

#### Post-Process Halo (Convolution Outline)
Renders selected objects into an offscreen mask, blurs the mask with a Gaussian or box filter, subtracts the unblurred mask, and composites the resulting halo over the scene. Width and softness are controlled by the blur radius and the subtraction coefficient.

**Use cases** — soft selection glow; XP-style health aura; any effect needing a smooth width-controllable halo rather than a sharp edge.

---

### Octahedral Impostor LOD

At large distances, complex 3D objects (trees, rocks, vehicles) can be replaced with a **billboard** — a camera-facing quad — textured with a prerendered view of the object. **Octahedral impostors** generalise flat billboards: the impostor atlas contains views of the object sampled at directions distributed over the hemisphere (or full sphere) of an octahedron; at runtime the two nearest atlas views are blended based on the actual camera direction, giving correct silhouettes from all angles without 3D geometry.

**Shader stage**: impostor rendering uses a **vertex shader** (align billboard to camera) and **fragment shader** (atlas lookup + view blending).

#### Octahedral Atlas Layout
The 2D atlas is arranged so that each cell in an N×N grid corresponds to one view direction mapped from the octahedron (§35 octahedral mapping). A 16×16 atlas = 256 views covering the upper hemisphere, each rendered offline into RGBA8 (albedo+alpha) or GBuffer MRTs.

```glsl
// Convert camera direction to atlas UV (octahedral hemisphere map)
vec2 dir_to_impostor_uv(vec3 view_dir, int atlas_n) {
    // Project direction onto octahedron (upper hemisphere)
    vec3 d   = view_dir / (abs(view_dir.x) + abs(view_dir.y) + abs(view_dir.z));
    vec2 oct = d.xz;  // top-down hemisphere: use XZ plane
    if (d.y < 0.0) {  // fold lower hemisphere (mirror edges)
        oct = (1.0 - abs(oct.yx)) * sign(oct);
    }
    // Map [-1,1] to [0, atlas_n-1] grid cell (bilinear between 4 nearest cells)
    return oct * 0.5 + 0.5;
}
```

#### Impostor Fragment Shader
Look up the two nearest atlas frames (by quantising the view direction to the nearest octahedral grid point and its neighbour), blend between them, and discard transparent pixels to preserve the silhouette.

```glsl
// Impostor fragment shader — blend two nearest views
layout(set=0, binding=0) uniform sampler2DArray impostor_atlas;  // array of N*N cells
uniform int   atlas_n;        // views per axis (e.g. 16)
uniform vec3  impostor_scale; // world-space size of billboard

layout(location=0) in vec3 v_cam_dir;  // camera→impostor direction (world space)
layout(location=1) in vec2 v_uv;       // billboard surface UV [0,1]

layout(location=0) out vec4 frag;

void main() {
    // Quantise camera direction to nearest grid cell
    vec2  oct    = dir_to_impostor_uv(normalize(v_cam_dir), atlas_n) * float(atlas_n - 1);
    ivec2 cell0  = ivec2(floor(oct));
    ivec2 cell1  = ivec2(ceil(oct));
    vec2  frac   = fract(oct);

    // Map surface UV into each cell's sub-rect within the atlas
    vec2 atlas_uv0 = (v_uv + vec2(cell0)) / float(atlas_n);
    vec2 atlas_uv1 = (v_uv + vec2(cell1)) / float(atlas_n);

    vec4 s0 = texture(impostor_atlas, vec3(atlas_uv0, 0.0));
    vec4 s1 = texture(impostor_atlas, vec3(atlas_uv1, 0.0));

    // Blend based on fractional position between cells
    float blend_w = length(frac);
    frag = mix(s0, s1, blend_w);
    if (frag.a < 0.1) discard;
}
```

**LOD integration**: the impostor replaces the full mesh below a screen-coverage threshold (computed from projected bounding sphere radius vs. pixel coverage). A smooth transition can blend the impostor alpha with the mesh at the crossover distance.

**Reference** — [Loos & Sloan: Volumetric Billboards (CGF 2010)](https://dl.acm.org/doi/10.1111/j.1467-8659.2010.01663.x); [Shaderbits: Octahedral Impostors](https://shaderbits.com/blog/octahedral-impostors/)

---

### 3D Gaussian Splatting

3D Gaussian Splatting (3DGS, Kerbl et al. 2023) represents a scene as a set of M anisotropic 3D Gaussians — each described by a world-space position μ, a covariance matrix Σ (encoded as a rotation quaternion q and scale s), an opacity α, and spherical harmonic (SH) coefficients for view-dependent colour. At render time, the Gaussians are projected to 2D screen-space ellipses, sorted front-to-back by depth, and alpha-composited in a tile-based rasterizer. This pipeline replaces both the NeRF MLP inference (§46) and traditional geometry, achieving real-time novel-view synthesis quality.

**Shader stage**: projection and depth sorting run in **compute**; tile-based rasterization and alpha compositing run in a **compute** forward pass (one workgroup per screen tile).

#### Gaussian Projection to 2D
Each 3D Gaussian is projected to a 2D screen-space Gaussian using the Jacobian of the perspective projection.

```glsl
// Project one 3D Gaussian to screen-space ellipse params
// Input: position, quaternion rotation, scale (log), opacity, SH coefficients
// Output: 2D mean, 2D covariance inverse, colour, opacity

struct Gaussian3D {
    vec3  pos;
    vec4  rot;      // unit quaternion (w, x, y, z)
    vec3  scale;    // log scale per axis
    float opacity;  // before sigmoid
    float sh[48];   // SH coefficients, degree 3 = 16 * RGB
};

struct Gaussian2D {
    vec2  mean;     // screen UV
    mat2  cov_inv;  // inverse 2D covariance for Gaussian eval
    float opacity;
    vec3  color;
    float depth;    // for sort
};

Gaussian2D project_gaussian(Gaussian3D g, mat4 view, mat4 proj, vec2 resolution) {
    // Build 3D covariance: Σ = R * S * S^T * R^T
    mat3 R = quat_to_mat3(g.rot);
    vec3 sc = exp(g.scale);
    mat3 S = mat3(sc.x, 0, 0,  0, sc.y, 0,  0, 0, sc.z);
    mat3 Sigma3D = R * S * transpose(S) * transpose(R);

    // Project: Σ_2D = J * W * Σ3D * W^T * J^T (Zwicker 2002)
    vec4 pos_view = view * vec4(g.pos, 1.0);
    float tx = pos_view.x, ty = pos_view.y, tz = pos_view.z;
    float f = proj[1][1];   // focal length y from projection matrix
    // Jacobian of perspective projection (at this point)
    mat3 J = mat3(
        f / tz,     0.0,        -f * tx / (tz * tz),
        0.0,        f / tz,     -f * ty / (tz * tz),
        0.0,        0.0,         0.0
    );
    mat3 W = mat3(view);  // upper-left 3×3 of view matrix
    mat3 Sigma2D3 = J * W * Sigma3D * transpose(W) * transpose(J);
    mat2 cov2D = mat2(Sigma2D3[0][0] + 0.3, Sigma2D3[0][1],
                      Sigma2D3[1][0],        Sigma2D3[1][1] + 0.3);

    // Invert 2D covariance
    float det   = cov2D[0][0] * cov2D[1][1] - cov2D[0][1] * cov2D[1][0];
    mat2  cov_i = mat2( cov2D[1][1], -cov2D[0][1],
                       -cov2D[1][0],  cov2D[0][0]) / max(det, 1e-6);

    // Project mean to screen
    vec4  clip  = proj * pos_view;
    vec2  mean  = (clip.xy / clip.w * 0.5 + 0.5) * resolution;

    // Evaluate SH for view-dependent colour (degree 1 for brevity)
    vec3  dir   = normalize(g.pos - cam_pos);
    vec3  color = sh_l0(g.sh) + sh_l1(g.sh, dir);  // + higher SH degrees

    return Gaussian2D(mean, cov_i, sigmoid(g.opacity), color, pos_view.z);
}
```

#### Tile-Based Alpha Compositing
Divide the screen into 16×16 pixel tiles. Sort the Gaussians by depth within each tile. Each tile's workgroup composites all overlapping Gaussians front-to-back using Porter-Duff OVER.

```glsl
// Tile compositing — one workgroup per 16×16 tile
layout(local_size_x=16, local_size_y=16) in;
layout(set=0, binding=0, rgba8) uniform image2D output_image;
layout(set=0, binding=1) readonly buffer SortedGaussians { Gaussian2D gs[]; };
layout(set=0, binding=2) readonly buffer TileRanges { ivec2 tile_range[]; }; // [start, end]

void main() {
    ivec2 px    = ivec2(gl_GlobalInvocationID.xy);
    ivec2 tile  = ivec2(gl_WorkGroupID.xy);
    vec2  px_f  = vec2(px) + 0.5;

    ivec2 range = tile_range[tile.y * num_tiles_x + tile.x];
    vec3  accum = vec3(0.0);
    float T     = 1.0;   // transmittance, starts at 1

    for (int i = range.x; i < range.y && T > 0.001; ++i) {
        Gaussian2D g  = gs[i];
        vec2       d  = px_f - g.mean;

        // Gaussian power: G(d) = exp(-0.5 * d^T * cov_inv * d)
        float power = -0.5 * (g.cov_inv[0][0]*d.x*d.x
                            + 2.0*g.cov_inv[0][1]*d.x*d.y
                            + g.cov_inv[1][1]*d.y*d.y);
        if (power > 0.0) continue;

        float alpha = min(0.99, g.opacity * exp(power));
        accum      += T * alpha * g.color;
        T          *= (1.0 - alpha);
    }
    accum += T * background_color;
    imageStore(output_image, px, vec4(accum, 1.0));
}
```

**Use cases** — real-time novel-view synthesis from captured scenes; digital twins; NeRF-to-3DGS baking for game-ready assets; environment capture for VR.  
**Reference** — [Kerbl et al.: 3D Gaussian Splatting for Real-Time Novel View Synthesis (SIGGRAPH 2023)](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/)

---

### Neural Rendering Primitives

Neural rendering augments or replaces traditional texture/geometry lookups with small learned models evaluated at runtime on the GPU. The key enabler is **hash grid encoding** (Müller et al. 2022): instead of a large MLP mapping 3D position to colour, position is first encoded by a multi-resolution hash table of learned feature vectors, and only a tiny 2–4 layer MLP processes the encoded features. This makes inference fast enough for real-time use.

**Shader stage**: hash grid lookup and tiny MLP inference run in **compute shaders** (or fragment shaders for view-dependent effects). Weight SSBOs store the network parameters.

#### Instant-NGP Hash Grid Encoding
The scene is divided into L multi-resolution grids. For a query point `x`, each level `l` has a grid of resolution `N_l = ⌊N_min · b^l⌋` and a hash table of size `T` (typically 2^14 to 2^24). The 8 corners of the cell containing `x` are hashed to table indices; their feature vectors are trilinearly interpolated and concatenated across all L levels into a single feature vector.

```glsl
// Hash grid lookup — one level
layout(set=0, binding=0) buffer HashTable { vec2 features[]; };  // F=2 features per entry

uint grid_hash(ivec3 cell, uint T) {
    const uint p1 = 1u, p2 = 2654435761u, p3 = 805459861u;
    return (uint(cell.x)*p1 ^ uint(cell.y)*p2 ^ uint(cell.z)*p3) & (T - 1u);
}

vec2 hash_grid_lookup(vec3 x, float resolution, uint T) {
    vec3  scaled = x * resolution;
    ivec3 base   = ivec3(floor(scaled));
    vec3  frac   = fract(scaled);

    // 8-corner trilinear interpolation
    vec2 result = vec2(0.0);
    for (int dz=0; dz<=1; ++dz)
    for (int dy=0; dy<=1; ++dy)
    for (int dx=0; dx<=1; ++dx) {
        uint h = grid_hash(base + ivec3(dx,dy,dz), T);
        vec2 f = features[h];
        float w = (dx==0 ? 1.0-frac.x : frac.x)
                * (dy==0 ? 1.0-frac.y : frac.y)
                * (dz==0 ? 1.0-frac.z : frac.z);
        result += w * f;
    }
    return result;
}
// Concatenate L=16 levels → 32-dimensional feature vector → tiny MLP
```

**Reference** — [Müller et al.: Instant Neural Graphics Primitives (SIGGRAPH 2022)](https://nvlabs.github.io/instant-ngp/)

#### Tiny MLP Inference in Compute
A 2–4 layer fully-connected network with 64 neurons per layer fits entirely in registers on modern GPUs. Each layer is a matrix-vector product followed by a ReLU (or exponential for the final colour output). Weights are stored in a small SSBO read once per network query.

```glsl
// Tiny MLP inference — 2 hidden layers, width W
layout(set=1, binding=0) readonly buffer Weights0 { float w0[W * IN_DIM]; };
layout(set=1, binding=1) readonly buffer Weights1 { float w1[W * W]; };
layout(set=1, binding=2) readonly buffer Weights2 { float w2[OUT_DIM * W]; };
layout(set=1, binding=3) readonly buffer Biases0  { float b0[W]; };
// ... similar for b1, b2

vec4 mlp_infer(float features[IN_DIM]) {
    float h0[W], h1[W];
    // Layer 0: h0 = ReLU(W0 * features + b0)
    for (int j = 0; j < W; ++j) {
        float acc = b0[j];
        for (int i = 0; i < IN_DIM; ++i) acc += w0[j * IN_DIM + i] * features[i];
        h0[j] = max(0.0, acc);
    }
    // Layer 1: h1 = ReLU(W1 * h0 + b1)
    for (int j = 0; j < W; ++j) {
        float acc = b1[j];
        for (int i = 0; i < W; ++i) acc += w1[j * W + i] * h0[i];
        h1[j] = max(0.0, acc);
    }
    // Output layer
    vec4 out = vec4(0.0);
    for (int j = 0; j < OUT_DIM; ++j) {
        float acc = b2[j];
        for (int i = 0; i < W; ++i) acc += w2[j * W + i] * h1[i];
        out[j] = acc;
    }
    return out;
}
```

**Use cases** — neural radiance caching (NRC, §6); neural texture compression (trained per-texture decoder); view-dependent appearance model for baked scenes.  
**Reference** — [Müller et al.: Real-Time Neural Radiance Caching (SIGGRAPH 2021)](https://research.nvidia.com/publication/2021-06_real-time-neural-radiance-caching-path-tracing)

#### NeRF — Neural Radiance Fields (Conceptual)
A NeRF stores a scene as a neural function `f(x, d) → (RGB, σ)` mapping 3D position `x` and view direction `d` to colour and density. Rendering requires ray marching through the volume, sampling the network at each step to accumulate colour and opacity via volume rendering (Beer-Lambert integration over learned density). Real-time NeRF rendering uses the hash grid to replace positional encoding, reducing network width and enabling interactive rates.

**Relevance** — WebGPU/compute-capable browsers can run tiny NeRF inference; the technique is increasingly deployed for 3D content on the web (Google Immersive View, ARCore Geospatial).  
**Reference** — [Mildenhall et al.: NeRF (ECCV 2020)](https://www.matthewtancik.com/nerf); [instant-ngp project](https://nvlabs.github.io/instant-ngp/)

---


---

## XII. Post-Processing and Image Effects

Post-processing operates on a fully-rendered image buffer rather than on geometry or lights, running in fragment or compute shaders on fullscreen quads or compute dispatches. This category covers the canonical post-processing pipeline: temporal anti-aliasing and reconstruction (TAA, motion vectors for temporal reprojection), camera lens simulation (depth of field, motion blur, bloom, chromatic aberration, lens flare, vignette), image quality steps (contrast-adaptive sharpening, film grain, ordered dithering), and the colour pipeline (colour science transforms, 3D LUT grading, histogram-based auto-exposure).

### Post-Processing and Image Effects

Post-processing runs after the main scene is rendered, operating on the completed colour buffer (and optionally depth and motion vectors) as a sequence of full-screen passes. These passes are responsible for the final "look" of the image: tone mapping brings HDR linear-light values into display range; bloom, depth of field, and motion blur add photographic character; colour grading via 3D LUT decouples art direction from shader code. Sharpening (CAS) counteracts the blur introduced by TAA (§16). The order of these passes matters — tone mapping must occur after bloom to preserve HDR highlight bloom, and LUT grading must occur after tone mapping.

**Shader stage**: All post-processing passes are full-screen **compute shaders** (preferred, better occupancy for read-modify-write) or full-screen **fragment shaders** (a fullscreen triangle bound to a framebuffer attachment). Bloom's separable blur is almost universally implemented as two **compute shader** dispatches (horizontal then vertical). CAS is a single **compute shader** dispatch.

#### Tone Mapping
Maps HDR scene luminance (linear light) to the display's SDR range [0,1] or PQ/HLG for HDR displays, applying a characteristic S-curve that compresses highlights while preserving shadow detail.

**Use cases** — required final pass for all HDR rendering pipelines; the choice of curve affects the entire look of the image.  
**Key variants** — Reinhard (simple, preserves ratios, bright desaturation); Uncharted 2 / Hable (filmic toe and shoulder); ACES Filmic (industry standard, DCI-P3 gamut); Khronos PBR Neutral (2023, perceptual uniformity, open standard).  
**Reference** — [Hable: Filmic Tonemapping (GDC 2010)](https://gdcvault.com/play/1012351) | [Khronos PBR Neutral](https://github.com/KhronosGroup/ToneMapping)

#### Bloom
Extracts bright regions above a threshold, applies a multi-pass blur, and additively composites onto the original image to simulate lens glare from intense light sources.

**Use cases** — emissive surfaces, sun disc, neon signs, explosions; contributes significantly to perceived HDR quality.  
**Key variants** — dual Kawase blur (fast, separable); FFT convolution bloom (physically correct lens PSF, expensive); tiered threshold (luminance + colour channel blend).  
**Reference** — [Kawase: Frame Buffer Postprocessing Effects (GDC 2003)](https://www.daionet.gr.jp/~masa/column/2003-06-02.html)

#### Depth of Field
Simulates camera lens focus by blurring fragments at distances outside the focal plane, with blur radius (Circle of Confusion) proportional to distance from the focal plane.

**Use cases** — cinematic camera feels; product visualisation; cutscenes.  
**Key variants** — CoC-based Gaussian blur (cheap, incorrect); gather-based bokeh (Jimenez 2014, scatter-as-gather); physically accurate hexagonal or circular bokeh; next-frame reprojection.  
**Reference** — [Jimenez: Next Generation Post Processing (SIGGRAPH 2014)](https://advances.realtimerendering.com/s2014/)

#### Motion Blur
Blurs each fragment along its screen-space velocity vector to simulate camera shutter opening during fast motion.

**Use cases** — conveying speed; reducing temporal aliasing at low frame rates; matching cinematic look.  
**Key variants** — per-object velocity buffer; tile max velocity + neighbourhood max (McGuire 2012); camera-only blur (cheaper); exposure-weighted accumulation.  
**Reference** — [McGuire et al.: A Reconstruction Filter for Plausible Motion Blur (HPG 2012)](https://research.nvidia.com/publication/2012-08_reconstruction-filter-plausible-motion-blur)

#### Chromatic Aberration
Offsets the UV coordinates of each colour channel differently to simulate lens chromatic aberration (colour fringing at high-contrast edges).

**Use cases** — cinematic / lo-fi aesthetic; realistic lens simulation; impact/damage effect.  
**Limitations** — overuse is a common stylistic mistake; physically, aberration is concentrated at the image periphery not uniformly.

#### Color Grading via 3D LUT
Applies a pre-baked 3D lookup table (typically 33³ or 65³ RGB entries) to remap scene colours to a target grade; the LUT can encode any colour transform including CDL, film emulsion simulation, or artistic grade.

**Use cases** — every production renderer uses LUT-based grade; decouples colour science pipeline from shader code.  
**Reference** — [GPU Gems 2 Ch24: Using Lookup Tables](https://developer.nvidia.com/gpugems/gpugems2/part-iii-high-quality-rendering/chapter-24-using-lookup-tables-accelerate-color)

#### Screen-Space Lens Flares
Generates analytic ghost artefacts, starburst diffraction patterns, and haze halos in screen space by ray-marching from bright sources toward the camera's optical axis.

**Use cases** — sun, bright point lights, weapon effects; purely aesthetic, no physical accuracy required.  
**Reference** — [Eberly: Physically Based Lens Flares (GPU Pro 3)](https://gpupro.blogspot.com/)

#### CAS (Contrast-Adaptive Sharpening)
An AMD FidelityFX compute pass that detects locally low-contrast regions and applies a spatial sharpening kernel proportional to local contrast, avoiding over-sharpening of already sharp edges.

**Use cases** — post-TAA sharpening to counteract temporal blur; upscaling quality improvement.  
**Reference** — [AMD FidelityFX CAS](https://gpuopen.com/fidelityfx-cas/)

---

### Anti-Aliasing, TAA, and ML Upscaling

Aliasing arises wherever a continuous signal (geometry edges, specular highlights, fine texture) is sampled at discrete pixel positions. The solutions in this section operate at different levels: MSAA and SMAA work on geometric edges during or immediately after rasterisation; TAA accumulates sub-pixel samples across time and is the current industry standard for both geometric and shader aliasing; ML upscalers (DLSS, FSR, XeSS) extend TAA by rendering at a fraction of display resolution and reconstructing the full image with a neural network or spatial filter. Upscaling and anti-aliasing have converged: every modern upscaler includes temporal accumulation, and every modern TAA implementation includes a sharpening step to counteract its inherent blur.

**Shader stage**: MSAA is a fixed-function rasterisation setting; the resolve is a **fragment shader** or **compute shader** pass. FXAA and SMAA run as **fragment shaders** or **compute shaders** on the resolved colour buffer. TAA history accumulation and reprojection run as **compute shaders**. DLSS, FSR, and XeSS all run as one or more **compute shader** dispatches taking colour + depth + motion vectors as inputs.

#### MSAA (Multisample Anti-Aliasing)
Rasterises each pixel at multiple sub-pixel sample positions and averages coverage, reducing geometric edge aliasing without increasing the shading rate (using shading at centroid or one sample per pixel).

**Use cases** — forward rendering with transparent geometry; VR (high pixel-density screens make TAA ghosting distracting).  
**Limitations** — does not reduce shader aliasing (specular shimmer); MSAA resolve is expensive for deferred rendering.

#### FXAA (Fast Approximate Anti-Aliasing)
Detects high-contrast edges in the final colour buffer using local luminance gradients and blends along the detected edge direction sub-pixel.

**Use cases** — a single post-process pass that reduces aliasing at near-zero cost; still used as a fallback when TAA is unavailable.  
**Limitations** — blurs texture detail; cannot reduce sub-pixel specular shimmer; superseded by SMAA and TAA.  
**Reference** — [Lottes: FXAA (NVIDIA 2009)](https://developer.download.nvidia.com/assets/gamedev/files/sdk/11/FXAA_WhitePaper.pdf)

#### SMAA (Subpixel Morphological Anti-Aliasing)
A morphological technique that detects edge patterns in the image (using a pre-built pattern atlas), computes sub-pixel blend weights, and applies a higher-quality edge-aware blend than FXAA.

**Use cases** — 2× the quality of FXAA at 2× the cost; SMAA T2x adds temporal accumulation for sub-pixel detail.  
**Reference** — [Jimenez et al.: SMAA (2012)](https://iryoku.com/smaa/)

#### TAA (Temporal Anti-Aliasing)
Accumulates jittered sub-pixel samples across multiple frames using reprojected history, blending new and old samples with exponential moving average, providing high-quality AA including sub-pixel specular.

**Use cases** — the dominant AA technique in modern engines (Unreal, Unity HDRP, Godot); essential for stable specular.  
**Key variants** — basic TAA (Karis 2014); TAA with variance clipping (reduces ghosting); TAA with Catmull-Rom history filtering; Mitchell-Netravali reconstruction.  
**Limitations** — ghosting on fast-moving objects if velocity vectors are incomplete; blurring of fine geometry; requires motion vectors from all moving objects.  
**Reference** — [Karis: High-Quality Temporal Supersampling (SIGGRAPH 2014)](https://advances.realtimerendering.com/s2014/)

#### DLSS 3 (NVIDIA Deep Learning Super Sampling)
Uses a transformer-based neural network and hardware Tensor Cores to upscale frames from a lower resolution, with frame generation (optical-flow–interpolated intermediate frame) for additional frame rate multiplication.

**Use cases** — NVIDIA RTX GPU users; highest image quality upscaling; frame generation doubles apparent frame rate.  
**Limitations** — NVIDIA-only (Tensor Cores required); frame generation adds latency; frame-gen frames can't sample new input state.  
**Reference** — [NVIDIA DLSS Developer Documentation](https://developer.nvidia.com/rtx/dlss)

#### FSR 3 (AMD FidelityFX Super Resolution)
Open-source spatial upscaler (RCAS sharpening + Lanczos) with optional frame generation (Fluid Motion Frames, optical-flow–based); vendor-agnostic, runs on any GPU.

**Use cases** — cross-vendor upscaling; available on PC, consoles, and open-source projects.  
**Reference** — [AMD GPUOpen: FidelityFX Super Resolution](https://gpuopen.com/fidelityfx-super-resolution/)

#### XeSS (Intel Xe Super Sampling)
ML-based upscaler that uses XMX matrix units on Intel Arc GPUs for highest quality; falls back to DP4a instructions on non-Arc hardware for broad compatibility.

**Use cases** — Intel Arc users (XMX path); general cross-vendor upscaling on the DP4a path.  
**Reference** — [Intel XeSS Developer Guide](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-xe-super-sampling.html)

---

### Motion Vector Generation

Motion vectors (also called velocity vectors) store the 2D screen-space displacement of each pixel from the previous frame to the current frame. They are consumed by TAA (§17) for reprojection, SVGF denoising (§40) for temporal accumulation, and motion blur (§10). Without correct motion vectors, TAA ghosts on moving objects and SVGF fails to reuse history at dynamic surfaces.

**Shader stage**: motion vectors are written by a dedicated **fragment shader** output (or as an additional G-Buffer MRT) during the geometry pass. Static objects can use a depth-reprojection fallback; dynamic objects (skinned meshes, rigid bodies, particles) must write per-vertex motion using their previous-frame transform.

#### Static Object Reprojection (Depth-Based)
For geometry that does not move, the motion vector is computed entirely in the fragment shader from the current-frame world position (reconstructed from depth) reprojected through the previous frame's view-projection matrix. No additional vertex data required.

```glsl
// Fragment shader — motion vector pass for static geometry
layout(set=0, binding=0) uniform sampler2D g_depth;
layout(set=0, binding=1) uniform CameraUB {
    mat4 curr_inv_proj_view;   // current frame inverse VP
    mat4 prev_proj_view;       // previous frame VP
};
layout(location=0) out vec2 motion_vector;

void main() {
    ivec2 px      = ivec2(gl_FragCoord.xy);
    float depth   = texelFetch(g_depth, px, 0).r;

    // Reconstruct world position from current depth
    vec2  ndc     = (vec2(px) + 0.5) / vec2(textureSize(g_depth, 0)) * 2.0 - 1.0;
    vec4  clip    = vec4(ndc, depth, 1.0);
    vec4  world_h = curr_inv_proj_view * clip;
    vec3  world_p = world_h.xyz / world_h.w;

    // Reproject into previous frame clip space
    vec4 prev_clip = prev_proj_view * vec4(world_p, 1.0);
    vec2 prev_ndc  = prev_clip.xy / prev_clip.w;

    // Motion vector: current NDC − previous NDC (in [−1, 1] space)
    motion_vector  = ndc - prev_ndc;
}
```

#### Dynamic Object Motion Vectors (Per-Vertex)
Dynamic objects (skinned characters, rigid bodies, vehicles) must track their previous-frame world transform. The vertex shader computes both current and previous clip positions; the fragment shader writes the 2D difference.

```glsl
// Vertex shader — outputs current and previous clip position
layout(set=1, binding=0) uniform ObjectUB {
    mat4 curr_model;
    mat4 prev_model;   // model matrix from last frame
};
layout(set=0, binding=0) uniform CameraUB {
    mat4 curr_proj_view;
    mat4 prev_proj_view;
};

layout(location=0) in vec3 in_pos;
layout(location=0) out vec4 v_curr_clip;
layout(location=1) out vec4 v_prev_clip;

void main() {
    vec4 world_curr = curr_model * vec4(in_pos, 1.0);
    vec4 world_prev = prev_model * vec4(in_pos, 1.0);  // previous transform
    v_curr_clip     = curr_proj_view * world_curr;
    v_prev_clip     = prev_proj_view * world_prev;
    gl_Position     = v_curr_clip;
}
```

```glsl
// Fragment shader — write motion vector MRT
layout(location=0) out vec4 g_albedo;
layout(location=1) out vec2 g_motion;   // RG16F motion vector target

layout(location=0) in vec4 v_curr_clip;
layout(location=1) in vec4 v_prev_clip;

void main() {
    vec2 curr_ndc = v_curr_clip.xy / v_curr_clip.w;
    vec2 prev_ndc = v_prev_clip.xy / v_prev_clip.w;
    g_motion = curr_ndc - prev_ndc;
    // ... write other G-Buffer MRTs
}
```

**Skinned meshes**: run the compute skinning pre-pass (§23) twice — once for the current-frame skeleton and once for the previous-frame skeleton — producing two skinned vertex buffers. The vertex shader reads both and computes the motion vector from the position difference.

#### Disocclusion and History Invalidation
TAA and SVGF must detect pixels where the reprojected history is invalid: the surface has been newly revealed (disocclusion) or the material has changed. The standard test compares the reprojected depth against the current depth; a depth difference exceeding a threshold marks the history as invalid, forcing the temporal filter to fall back to the current frame only.

```glsl
// Disocclusion test — in TAA or SVGF temporal accumulation pass
float prev_depth_reproj = texture(prev_depth, prev_uv).r;
float curr_depth        = texture(curr_depth, curr_uv).r;
bool  disoccluded       = abs(prev_depth_reproj - curr_depth) > 0.01 * curr_depth;
float alpha = disoccluded ? 1.0 : 0.1;  // 1.0 = no history; 0.1 = 90% history
```

**Reference** — [Karis: High-Quality Temporal Supersampling (SIGGRAPH 2014)](https://advances.realtimerendering.com/s2014/); [Schied et al.: SVGF (HPG 2017)](https://research.nvidia.com/publication/2017-07_spatiotemporal-variance-guided-filtering-real-time-reconstruction-path-traced)

---

### Bloom — Dual Kawase and FFT Convolution

Bloom simulates the scattering of intense light by the camera lens and sensor, producing a soft glow around bright highlights. It is a mandatory HDR post-process: applied after tone mapping would clip the bright regions that drive it, so bloom operates on the pre-tone-map HDR buffer. Two implementations dominate: the **dual Kawase blur** (fast, separable, hardware-bilinear-exploiting) used by Unity and Godot, and **FFT convolution bloom** (physically accurate, captures any lens kernel shape) used in film and offline rendering.

**Shader stage**: all bloom passes are **compute or fragment** full-screen passes.

#### Threshold and Downsample
First, extract pixels above the bloom threshold into a half-resolution HDR buffer, then progressively downsample into a mip chain.

```glsl
// Bloom threshold pass — extract bright regions
layout(set=0, binding=0) uniform sampler2D hdr_scene;
layout(set=0, binding=1, rgba16f) uniform image2D bloom_0;  // half-res

uniform float threshold;      // e.g. 1.0 (in scene-linear units)
uniform float knee;           // soft knee width, e.g. 0.5

float bloom_knee(float x, float t, float k) {
    // Smooth knee: remap [t-k, t+k] quadratically
    float h = t - k;
    float s = 2.0 * k;
    if (x < h)       return 0.0;
    if (x > t + k)   return x - t;
    float v = x - h;
    return v * v / (4.0 * k);
}

void main() {
    ivec2 px   = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv   = (vec2(px) + 0.5) / vec2(imageSize(bloom_0));
    vec3  col  = texture(hdr_scene, uv).rgb;
    float lum  = dot(col, vec3(0.2126, 0.7152, 0.0722));
    float scale = bloom_knee(lum, threshold, knee) / max(lum, 1e-5);
    imageStore(bloom_0, px, vec4(col * scale, 1.0));
}
```

#### Dual Kawase Blur (Downsample + Upsample)
The dual Kawase blur performs iterative downsampling (each step takes 4 bilinear samples offset at ±0.5 texels) followed by upsampling (each step takes 8 bilinear samples). The two-pass structure avoids ring artifacts of single-direction Gaussian separability while being cheaper than a full Gaussian.

```glsl
// Dual Kawase downsample — sample 4 corners at half-texel offset
layout(set=0, binding=0) uniform sampler2D src;
layout(set=0, binding=1, rgba16f) uniform image2D dst;
uniform int iteration;  // controls offset size

void main() {
    ivec2 px  = ivec2(gl_GlobalInvocationID.xy);
    vec2  sz  = vec2(imageSize(dst));
    vec2  uv  = (vec2(px) + 0.5) / sz;
    vec2  off = 0.5 / vec2(textureSize(src, 0));

    vec4 col  = texture(src, uv + vec2(-off.x, -off.y)) * 0.25;
    col      += texture(src, uv + vec2( off.x, -off.y)) * 0.25;
    col      += texture(src, uv + vec2(-off.x,  off.y)) * 0.25;
    col      += texture(src, uv + vec2( off.x,  off.y)) * 0.25;
    imageStore(dst, px, col);
}

// Dual Kawase upsample — sample 8 neighbours
void main_upsample() {
    ivec2 px  = ivec2(gl_GlobalInvocationID.xy);
    vec2  sz  = vec2(imageSize(dst));
    vec2  uv  = (vec2(px) + 0.5) / sz;
    vec2  off = 0.5 / vec2(textureSize(src, 0));

    vec4 col  = vec4(0.0);
    col += texture(src, uv + vec2(-2.0*off.x, 0.0))         * (1.0/12.0);
    col += texture(src, uv + vec2(-off.x,  off.y))          * (2.0/12.0);
    col += texture(src, uv + vec2( 0.0,    2.0*off.y))      * (1.0/12.0);
    col += texture(src, uv + vec2( off.x,  off.y))          * (2.0/12.0);
    col += texture(src, uv + vec2( 2.0*off.x, 0.0))         * (1.0/12.0);
    col += texture(src, uv + vec2( off.x, -off.y))          * (2.0/12.0);
    col += texture(src, uv + vec2( 0.0,   -2.0*off.y))      * (1.0/12.0);
    col += texture(src, uv + vec2(-off.x, -off.y))          * (2.0/12.0);
    imageStore(dst, px, col);
}
```

The final pass additively composites all upsampled levels onto the HDR scene before tone mapping with a user-controlled `bloom_strength`.

#### FFT Convolution Bloom
Physically accurate bloom convolves the HDR buffer with a lens flare kernel (captured or designed as a 512×512 or 1024×1024 texture). The convolution is performed in frequency space: FFT the scene, FFT the kernel (precomputed), multiply complex spectra, inverse FFT. GPU FFT is implemented as a series of compute passes (Cooley-Tukey butterfly stages). Used in Unreal Engine's cinematic bloom mode and film compositing.

**Reference** — [Jimenez: Next Generation Post Processing in Call of Duty: Advanced Warfare (SIGGRAPH 2014)](https://www.iryoku.com/next-generation-post-processing-in-call-of-duty-advanced-warfare/); [Kawase: Frame Buffer Postprocessing Effects in DOUBLE-S.T.E.A.L. (GDC 2003)](https://www.daionet.gr.jp/~masa/archives/GDC2003_DSTEAL.ppt)

---

### Depth of Field — CoC and Scatter-as-Gather

Depth of Field (DoF) simulates camera lens defocus: objects outside the focal plane form a blurred **circle of confusion (CoC)** on the sensor whose radius is proportional to distance from the focus plane. Real-time DoF on GPU works in three stages: CoC computation per pixel, near/far field separation, and a gather-based reconstruction blur.

**Shader stage**: all passes are **compute or fragment**.

#### Circle of Confusion
Compute the CoC radius in pixels for each fragment from its linear depth and the lens parameters.

```glsl
// CoC computation — runs on depth buffer
uniform float focus_distance;   // focal plane distance (world units)
uniform float focus_range;      // half-width of in-focus zone
uniform float max_coc_pixels;   // maximum CoC radius in pixels (e.g. 16)
uniform float sensor_height;    // sensor height in world units (for physical model)
uniform float focal_length;     // lens focal length in world units
uniform float aperture;         // f-number (e.g. 1.4)

// Simplified linear model (game-style)
float compute_coc(float depth) {
    float dist_from_focus = abs(depth - focus_distance) - focus_range;
    return clamp(dist_from_focus / focus_distance * max_coc_pixels, -max_coc_pixels, max_coc_pixels);
    // positive CoC = far field (behind focus), negative = near field (in front)
}

// Physical thin-lens model
float compute_coc_physical(float depth) {
    float image_dist = focal_length * focus_distance / (focus_distance - focal_length);
    float image_dist_sample = focal_length * depth / (depth - focal_length);
    float coc_m = abs(aperture * focal_length * (image_dist_sample - image_dist))
                / (image_dist_sample * (focus_distance - focal_length));
    return coc_m / sensor_height * resolution_y;  // convert to pixels
}
```

#### Near/Far Field Separation and Gather Blur
The far field (objects behind focus) can simply be blurred; the near field (objects in front) bleeds over in-focus geometry and requires separate treatment. The standard approach: separate near and far CoC into two half-resolution buffers, blur each with a disc kernel using gather (loop over Poisson or Fibonacci disc samples), then composite back using a CoC-weighted alpha blend.

```glsl
// Scatter-as-gather DoF blur — far field
layout(set=0, binding=0) uniform sampler2D hdr_color;
layout(set=0, binding=1) uniform sampler2D coc_tex;       // R = far CoC in px, G = near CoC
layout(set=0, binding=2, rgba16f) uniform image2D dof_far;

// Fibonacci disc samples (25 taps)
const int   NUM_SAMPLES = 25;
const float GOLDEN = 2.399963229;  // 2π / φ²
vec2 fibonacci_disc(int i, int n) {
    float r   = sqrt(float(i) / float(n));
    float phi = float(i) * GOLDEN;
    return r * vec2(cos(phi), sin(phi));
}

void main() {
    ivec2 px     = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv     = (vec2(px) + 0.5) / vec2(imageSize(dof_far));
    float coc    = texture(coc_tex, uv).r;    // far CoC in pixels

    vec4  accum  = vec4(0.0);
    float weight = 0.0;
    for (int i = 0; i < NUM_SAMPLES; ++i) {
        vec2  off   = fibonacci_disc(i, NUM_SAMPLES) * coc / vec2(textureSize(hdr_color, 0));
        vec2  s_uv  = uv + off;
        float s_coc = texture(coc_tex, s_uv).r;
        // Accept sample only if its CoC covers this pixel (prevents leaking)
        float w     = step(length(off * vec2(textureSize(hdr_color, 0))), s_coc);
        accum      += texture(hdr_color, s_uv) * w;
        weight     += w;
    }
    imageStore(dof_far, px, (weight > 0.0) ? accum / weight : texture(hdr_color, uv));
}
```

#### Bokeh Shape
For artistic bokeh (hexagonal, star, anamorphic), replace the Fibonacci disc samples with samples from a precomputed bokeh shape texture. For physically accurate bokeh simulation, use the **scatter** approach: each bright pixel emits a sprite the size of its CoC — expensive but used in offline rendering and cinematic engines.

**Reference** — [Jimenez (2014), ibid.]; [Cantlay: Practical Post-Process Depth of Field (GPU Gems 3)](https://developer.nvidia.com/gpugems/gpugems3/part-iv-image-effects/chapter-28-practical-post-process-depth-field)

---

### Motion Blur — Per-Object Reconstruction Filter

Motion blur simulates camera shutter integration over time: objects moving during the exposure interval appear streaked. GPU implementations avoid the cost of temporal integration by reconstructing blur from per-pixel velocity vectors. The **McGuire reconstruction filter** (used in UE, DOOM, most AAA titles) operates in a single gather pass with velocity dilation to handle disocclusion.

**Shader stage**: the velocity buffer is written by the **vertex/geometry shader** (or rasterized in a separate pass). The blur reconstruction runs as a **compute or fragment** post-process.

#### Velocity Buffer Generation
Each vertex outputs both its current and previous clip-space positions. The fragment writes the screen-space velocity (in pixels per frame or NDC delta).

```glsl
// Vertex shader — output motion vectors
layout(location=0) in vec3 position;

uniform mat4 curr_mvp;
uniform mat4 prev_mvp;

layout(location=0) out vec4 v_curr_clip;
layout(location=1) out vec4 v_prev_clip;

void main() {
    v_curr_clip  = curr_mvp * vec4(position, 1.0);
    v_prev_clip  = prev_mvp * vec4(position, 1.0);
    gl_Position  = v_curr_clip;
}

// Fragment shader — write velocity in NDC space
layout(location=0) in  vec4 v_curr_clip;
layout(location=1) in  vec4 v_prev_clip;
layout(location=0) out vec2 velocity_out;

void main() {
    vec2 curr_ndc = v_curr_clip.xy / v_curr_clip.w;
    vec2 prev_ndc = v_prev_clip.xy / v_prev_clip.w;
    velocity_out  = (curr_ndc - prev_ndc) * 0.5;  // in [−1,1] NDC delta
}
```

#### Velocity Tile Dilation
Before the reconstruction gather, build a tile-max buffer: for each N×N tile (typically 20×20 pixels), store the velocity vector with the largest magnitude. This is the "neighbourhood max velocity" used to detect motion blur boundaries. Implemented as two passes: per-tile horizontal max, then vertical max.

```glsl
// Tile max — compute horizontal max velocity magnitude per tile row
layout(set=0, binding=0) uniform sampler2D velocity_buf;
layout(set=0, binding=1, rg16f) uniform image2D tile_max_h;

uniform int tile_size;  // e.g. 20

void main() {
    ivec2 tile = ivec2(gl_GlobalInvocationID.xy);
    vec2  max_vel = vec2(0.0);
    float max_len = 0.0;
    for (int x = 0; x < tile_size; ++x) {
        ivec2 px  = ivec2(tile.x * tile_size + x, tile.y);
        vec2  vel = texelFetch(velocity_buf, px, 0).rg;
        float l   = dot(vel, vel);
        if (l > max_len) { max_len = l; max_vel = vel; }
    }
    imageStore(tile_max_h, tile, vec4(max_vel, 0.0, 0.0));
}
```

#### Reconstruction Gather
For each pixel, gather samples along the dominant tile velocity direction, weighting by depth (foreground over background) and velocity similarity.

```glsl
// Motion blur reconstruction gather
layout(set=0, binding=0) uniform sampler2D hdr_color;
layout(set=0, binding=1) uniform sampler2D velocity_buf;
layout(set=0, binding=2) uniform sampler2D depth_buf;
layout(set=0, binding=3) uniform sampler2D tile_max;    // dilated tile velocity
layout(set=0, binding=4, rgba16f) uniform image2D mb_out;

uniform int   num_samples;   // typically 15
uniform float shutter_scale; // controls blur length, e.g. 0.5

void main() {
    ivec2 px   = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv   = (vec2(px) + 0.5) / vec2(imageSize(mb_out));
    vec2  tile_uv = uv;  // could subsample tile
    vec2  tile_vel = texture(tile_max, tile_uv).rg * shutter_scale;

    if (dot(tile_vel, tile_vel) < 0.5/resolution.x) {
        imageStore(mb_out, px, texture(hdr_color, uv));
        return;
    }

    vec4  accum  = vec4(0.0);
    float total  = 0.0;
    float depth0 = texture(depth_buf, uv).r;

    for (int i = 0; i < num_samples; ++i) {
        float t    = mix(-1.0, 1.0, (float(i) + 0.5) / float(num_samples));
        vec2  s_uv = uv + tile_vel * t;
        s_uv       = clamp(s_uv, vec2(0.0), vec2(1.0));

        float s_depth = texture(depth_buf, s_uv).r;
        vec2  s_vel   = texture(velocity_buf, s_uv).rg * shutter_scale;

        // Foreground weight: samples in front of receiver get full weight
        float fg_w    = float(s_depth <= depth0 + 0.001);
        // Velocity similarity: samples with velocity covering this pixel
        float vel_w   = clamp(1.0 - abs(dot(s_vel - tile_vel, tile_vel) /
                              max(dot(tile_vel, tile_vel), 1e-5)), 0.0, 1.0);
        float w       = fg_w * vel_w + 0.01;
        accum        += texture(hdr_color, s_uv) * w;
        total        += w;
    }
    imageStore(mb_out, px, accum / total);
}
```

**Reference** — [McGuire et al.: A Reconstruction Filter for Plausible Motion Blur (I3D 2012)](https://casual-effects.com/research/McGuire2012Blur/index.html)

---

### Chromatic Aberration

**Chromatic aberration** is the failure of a lens to focus all wavelengths at the same point — different colours are refracted by slightly different amounts, producing colour fringing at high-contrast edges. Two variants exist: **lateral** (transverse) CA, where fringing appears as RGB UV offset in the radial direction from the image centre, and **longitudinal** CA, where the blur circle varies per wavelength (requires per-channel defocus blur). In post-processing, lateral CA is simulated by per-channel radial UV offset, optionally combined with per-channel barrel distortion (§67).

**Shader stage**: runs as a **fragment or compute** full-screen pass, applied after tone mapping.

#### Lateral Chromatic Aberration (Per-Channel UV Offset)
```glsl
// Lateral chromatic aberration — radial RGB UV offset
layout(set=0, binding=0) uniform sampler2D ldr_scene;

uniform float ca_strength;   // e.g. 0.005 (fraction of image width)
uniform vec2  ca_centre;     // typically (0.5, 0.5)

layout(location=0) out vec3 ca_out;

void main() {
    vec2  uv     = gl_FragCoord.xy / resolution;
    vec2  offset = (uv - ca_centre);         // radial direction from centre
    float r2     = dot(offset, offset);       // distance² from centre

    // Per-channel UV offset: R outward, B inward (or both scaled differently)
    // Offset is proportional to r² (matches real lens behaviour)
    vec2  r_uv = uv + offset * ca_strength * r2 * 0.8;
    vec2  g_uv = uv;                                       // green channel unshifted
    vec2  b_uv = uv - offset * ca_strength * r2 * 1.2;

    ca_out = vec3(
        texture(ldr_scene, r_uv).r,
        texture(ldr_scene, g_uv).g,
        texture(ldr_scene, b_uv).b
    );
}
```

#### Spectral CA (Multi-Sample Wavelength)
For higher quality, sample at 5–8 wavelengths between 400 nm and 700 nm and blend the results using CIE colour matching functions, giving a physically plausible rainbow fringe that fades to white at low contrast.

```glsl
// Spectral chromatic aberration — 6 wavelength samples
const float wavelengths[6] = float[](400.0, 450.0, 500.0, 550.0, 600.0, 650.0);
const vec3  cmf_xyz[6] = vec3[](  // CIE 1931 colour matching (simplified)
    vec3(0.014, 0.000, 0.068), vec3(0.091, 0.004, 0.461),
    vec3(0.005, 0.323, 0.272), vec3(0.434, 0.995, 0.008),
    vec3(1.062, 0.631, 0.000), vec3(0.341, 0.175, 0.000)
);
uniform mat3 xyz_to_srgb;   // CIE XYZ → sRGB matrix

vec3 spectral_ca(sampler2D scene, vec2 uv, vec2 direction) {
    vec3 xyz = vec3(0.0);
    for (int i = 0; i < 6; ++i) {
        float dispersion = (wavelengths[i] - 550.0) / 300.0;  // −0.5 to +0.5
        vec2  s_uv = uv + direction * ca_strength * dispersion;
        float lum  = dot(texture(scene, s_uv).rgb, vec3(0.2126, 0.7152, 0.0722));
        xyz += cmf_xyz[i] * lum;
    }
    return max(xyz_to_srgb * xyz / 6.0, vec3(0.0));
}
```

**Reference** — [Pettineo: Physically-Based Lens Flares (Rastertek, 2017)](https://mynameismjp.wordpress.com/); [Jimenez (2014), ibid.]

---

### Lens Flare

Lens flares are optical artifacts produced when bright light sources (sun, headlights) interact with the multiple glass elements of a real camera lens: light bounces between element surfaces producing **ghosts** (circular or polygonal copies of the aperture), a **halo** (circular glow), and **streaks** (diffraction spikes from the aperture blades). In games and film compositing, these are replicated in screen space as a post-process applied after tone mapping.

**Shader stage**: runs as a **fragment or compute** full-screen pass. Ghosts and halos are rendered as 2D primitives blended additively into the output.

#### Ghost Generation
Project the sun/light position into screen space. For each ghost, place a scaled copy of the light's aperture shape at a position along the lens axis between the light and the screen centre (flipped reflection point).

```glsl
// Lens flare ghost — evaluated per fragment in ghost pass
uniform vec2  sun_uv;           // sun position in [0,1] screen UV
uniform vec4  ghost_params[8];  // per ghost: (offset, scale, falloff, unused)
uniform vec4  ghost_colors[8];  // per ghost colour tint

layout(set=0, binding=0) uniform sampler2D aperture_tex;  // hexagonal aperture mask
layout(set=0, binding=1) uniform sampler2D lens_color_tex;// spectral shift texture

layout(location=0) out vec4 flare_out;

void main() {
    vec2 uv     = gl_FragCoord.xy / resolution;
    vec3 result = vec3(0.0);

    // Flare axis: line from sun through screen centre
    vec2 axis   = sun_uv - 0.5;

    for (int g = 0; g < 8; ++g) {
        float offset = ghost_params[g].x;   // 0 = sun, 1 = opposite side
        float scale  = ghost_params[g].y;
        float falloff= ghost_params[g].z;

        // Ghost centre: reflected along the axis
        vec2  ghost_uv  = 0.5 + axis * (1.0 - 2.0 * offset);
        vec2  local_uv  = (uv - ghost_uv) / scale + 0.5;

        if (any(lessThan(local_uv, vec2(0.0))) || any(greaterThan(local_uv, vec2(1.0))))
            continue;

        float mask  = texture(aperture_tex, local_uv).r;
        float dist  = length(uv - ghost_uv);
        float fade  = exp(-dist * dist * falloff);

        // Chromatic tint from lens colour texture (spectral fringe)
        vec3  tint  = texture(lens_color_tex, vec2(float(g) / 8.0 + 0.5/8.0, 0.5)).rgb;
        result     += mask * fade * tint * ghost_colors[g].rgb;
    }

    flare_out = vec4(result, 1.0);
}
```

#### Halo and Streak
The halo is a Gaussian ring around the sun position: `exp(-abs(r - halo_radius)² / width²)`. Streaks are drawn as thin quads emanating from the sun position, with length proportional to sun luminance and a diffraction pattern envelope.

```glsl
// Halo contribution
float r         = length(uv - sun_uv);
float halo      = exp(-pow(abs(r - 0.12) / 0.05, 2.0)) * sun_luminance;

// Streak contribution (one streak, replicate for blade count)
float streak_dir = atan(uv.y - sun_uv.y, uv.x - sun_uv.x);
float streak_w   = exp(-pow(mod(streak_dir, 3.14159/3.0) / 0.08, 2.0));
float streak     = streak_w * exp(-r / 0.3) * sun_luminance;

result += vec3(halo * 0.8, halo * 0.9, halo) + vec3(streak);
```

**Occlusion**: multiply flare intensity by a sun visibility term computed from the shadow map or a ray cast to a screen-space depth sample at the sun UV, fading out when the sun is behind geometry.

**Reference** — [Flare Tool (Lens Flares in Unity HDRP)](https://docs.unity3d.com/Packages/com.unity.render-pipelines.high-definition@14.0/manual/lens-flares-reference.html); [Jimenez: Next-Gen Post Processing (SIGGRAPH 2014)](https://www.iryoku.com/next-generation-post-processing-in-call-of-duty-advanced-warfare/)

---

### Film Grain

Film grain simulates the silver halide crystal structure of analogue film: random brightness variation correlated across colour channels (luminance grain) with a characteristic frequency spectrum — high spatial frequency, slightly red-biased for ISO 800+ emulsions. Unlike simple white noise, authentic grain has a specific power spectrum (not flat), temporal variation, and is masked in the shadows and highlights (darkest and brightest areas show less grain than midtones).

**Shader stage**: runs as a **fragment or compute** pass applied to the LDR output after colour grading and tone mapping, before display encoding.

#### Procedural Grain via Blue-Noise Modulation
Tile a precomputed blue-noise texture (128×128 RGBA, each frame offset by a different RNG value) with temporal animation, then modulate by a grain strength that peaks at midtones.

```glsl
// Film grain — fragment shader
layout(set=0, binding=0) uniform sampler2D ldr_color;
layout(set=0, binding=1) uniform sampler2D blue_noise;  // 128×128 RGBA tileable

uniform float grain_strength;    // e.g. 0.04 for ISO 400, 0.1 for ISO 3200
uniform float grain_size;        // texel scale factor (> 1 = coarser grain)
uniform uint  frame_index;

layout(location=0) out vec3 grain_out;

void main() {
    vec2  uv      = gl_FragCoord.xy / resolution;
    vec3  color   = texture(ldr_color, uv).rgb;
    float lum     = dot(color, vec3(0.2126, 0.7152, 0.0722));

    // Temporal offset: shift blue noise each frame to avoid static pattern
    vec2  noise_uv  = gl_FragCoord.xy / (128.0 * grain_size);
    vec2  frame_off = vec2(
        fract(float(frame_index) * 0.61803),   // golden-ratio offset
        fract(float(frame_index) * 0.38197)
    );
    vec4  bn = texture(blue_noise, noise_uv + frame_off);

    // Grain amplitude: peaks at midtones (lum ~ 0.5), falls off in blacks and whites
    float mid_mask = 4.0 * lum * (1.0 - lum);   // parabola: 0 at 0 and 1, 1 at 0.5
    float amp      = grain_strength * mid_mask;

    // Chromatic grain: R/G/B channels use different noise samples (slight correlation)
    vec3  grain = vec3(bn.r, bn.g, bn.b) * 2.0 - 1.0;
    // Bias red channel slightly (film grain is more visible in luma/red)
    grain.r *= 1.2;
    grain.g *= 0.9;
    grain.b *= 0.8;

    grain_out = color + grain * amp;
}
```

#### Grain Power Spectrum (Frequency-Correct Grain)
For a more physically accurate grain spectrum, filter white noise through a Gaussian kernel that matches real film's power spectrum — coarser grain at higher ISO. In practice this is precomputed into the blue-noise texture; at runtime, additional resolution-dependent scaling of `grain_size` reproduces the correct spatial frequency for the output resolution.

```glsl
// Resolution-adaptive grain size: real grain is fixed physical size on sensor
// Higher render resolution = smaller grain in pixels = divide grain_size accordingly
float adaptive_grain_size = grain_size * (native_resolution.y / render_resolution.y);
vec2  noise_uv = gl_FragCoord.xy / (128.0 * adaptive_grain_size);
```

**Reference** — [Jimenez (2014), ibid. §Film Grain]; [Nosseir: Implementing a Practical Rendering Model for Neon Lights (2020)](https://bartwronski.com/2020/10/10/neon-neon-neon/); [Wronski: Good Random — Low Discrepancy Blue Noise in Temporal Domain (2022)](https://bartwronski.com/)

---

### Vignette and Lens Distortion

Two universal post-processing effects that complete the camera simulation pipeline: **vignette** darkens the frame edges to simulate real lens light falloff; **lens distortion** warps the image to match the barrel or pincushion distortion of a real or stylised optical system (required for VR optics compensation and cinematic fisheye looks).

**Shader stage**: both run as a **fragment shader** in the final post-processing pass, applied after tone mapping and before display output.

#### Vignette
Reduces brightness near the screen edges as a function of radial distance from the screen centre. The falloff exponent and radius control the effect strength.

```glsl
// Vignette — applied to tone-mapped LDR output
uniform float vignette_strength;  // 0.0 = none, 1.0 = heavy
uniform float vignette_radius;    // typically 0.75–0.85
uniform float vignette_softness;  // feather width, typically 0.45

vec3 apply_vignette(vec3 color, vec2 uv) {
    vec2  center = uv - 0.5;          // centre at (0,0)
    float dist   = length(center);
    float vign   = smoothstep(vignette_radius,
                              vignette_radius - vignette_softness,
                              dist);
    return color * mix(1.0 - vignette_strength, 1.0, vign);
}
```

**Chromatic vignette variant**: apply different vignette strengths per colour channel (stronger on blue), mimicking lens colour falloff.

#### Barrel / Pincushion Distortion
Warps texture coordinates before sampling the rendered scene to compensate for or simulate lens distortion. Barrel distortion (outward bulge, negative k) is produced by wide-angle and fisheye lenses; pincushion (inward pull, positive k) by telephoto lenses. VR headsets apply barrel distortion to pre-compensate for the headset lens's pincushion.

```glsl
// Radial lens distortion — Brown-Conrady model (k1, k2 coefficients)
// k1 < 0: barrel (wide-angle, VR pre-distortion)
// k1 > 0: pincushion (telephoto)
uniform float k1;   // primary distortion coefficient
uniform float k2;   // secondary (quartic) coefficient
uniform vec2  lens_centre;   // typically (0.5, 0.5)

vec2 distort_uv(vec2 uv) {
    vec2  d  = uv - lens_centre;
    float r2 = dot(d, d);
    float factor = 1.0 + k1 * r2 + k2 * r2 * r2;
    return lens_centre + d * factor;
}

// Sample the rendered scene through the distorted UV
vec3 apply_lens_distortion(sampler2D scene, vec2 uv) {
    vec2 distorted = distort_uv(uv);
    if (any(lessThan(distorted, vec2(0.0))) ||
        any(greaterThan(distorted, vec2(1.0))))
        return vec3(0.0);   // outside frame — show black
    return texture(scene, distorted).rgb;
}
```

#### Chromatic Aberration via Per-Channel Distortion
Apply slightly different distortion strengths per colour channel, simulating longitudinal chromatic aberration (colour fringing at high-contrast edges). Combine with the main distortion pass for efficiency.

```glsl
// Per-channel distortion for chromatic aberration + barrel distortion
vec3 distort_chromatic(sampler2D scene, vec2 uv) {
    vec2 r_uv = distort_uv_k(uv, k1 * 1.00, k2 * 1.00);  // red: base distortion
    vec2 g_uv = distort_uv_k(uv, k1 * 1.02, k2 * 1.02);  // green: slightly more
    vec2 b_uv = distort_uv_k(uv, k1 * 1.04, k2 * 1.04);  // blue: most distorted
    return vec3(
        texture(scene, r_uv).r,
        texture(scene, g_uv).g,
        texture(scene, b_uv).b
    );
}
```

**VR usage**: each eye's final blit applies a per-eye distortion mesh (precomputed by the VR runtime, e.g., via OpenXR `xrGetSwapchainImage`) rather than a polynomial approximation, achieving sub-pixel accurate optics compensation.

**Reference** — [Valve: VR Distortion (SteamVR SDK)](https://github.com/ValveSoftware/openvr); [Brown: Decentering Distortion of Lenses (1966)](https://www.semanticscholar.org/paper/Decentering-distortion-of-lenses-Brown/7a2b4b22e94d29196e4c9f2c8a07636ca5af527)

---

### Contrast Adaptive Sharpening

**Contrast Adaptive Sharpening (CAS)**, released by AMD in FidelityFX, applies a spatially-adaptive sharpening filter that increases edge contrast without amplifying noise — sharpening is proportional to the local contrast minimum, so flat regions (already sharp, or just noise) receive little sharpening while low-contrast edges (genuinely blurred by temporal AA or upscaling) are boosted. CAS is universally used as a post-pass after DLSS, FSR, or TAA.

**Shader stage**: runs as a **compute shader** (one thread per output pixel, or 8×8 tiles for cache efficiency).

```glsl
// Contrast Adaptive Sharpening — compute shader (AMD FidelityFX CAS)
layout(local_size_x=8, local_size_y=8) in;
layout(set=0, binding=0) uniform  sampler2D input_tex;
layout(set=0, binding=1, rgba8)   uniform image2D output_img;

uniform float sharpness;   // 0.0 = full sharpening, 1.0 = none (counter-intuitive: 0 = max)

void main() {
    ivec2 px = ivec2(gl_GlobalInvocationID.xy);
    vec2  sz = vec2(textureSize(input_tex, 0));
    vec2  uv = (vec2(px) + 0.5) / sz;
    vec2  d  = 1.0 / sz;

    // Sample 3×3 neighbourhood (5-tap cross only: cheaper, sufficient)
    vec3  a  = texture(input_tex, uv + vec2(-d.x, -d.y)).rgb;  // top-left
    vec3  b  = texture(input_tex, uv + vec2( 0.0, -d.y)).rgb;  // top
    vec3  c  = texture(input_tex, uv + vec2( d.x, -d.y)).rgb;  // top-right
    vec3  d_ = texture(input_tex, uv + vec2(-d.x,  0.0)).rgb;  // left
    vec3  e  = texture(input_tex, uv).rgb;                       // centre
    vec3  f  = texture(input_tex, uv + vec2( d.x,  0.0)).rgb;  // right
    vec3  g  = texture(input_tex, uv + vec2(-d.x,  d.y)).rgb;  // bottom-left
    vec3  h  = texture(input_tex, uv + vec2( 0.0,  d.y)).rgb;  // bottom
    vec3  i_ = texture(input_tex, uv + vec2( d.x,  d.y)).rgb;  // bottom-right

    // Local contrast: min/max of cross neighbours (b, d, f, h) + centre
    vec3  mn = min(min(min(b, d_), min(f, h)), e);
    vec3  mx = max(max(max(b, d_), max(f, h)), e);

    // Adaptive sharpening weight: inversely proportional to contrast
    // (more contrast = less sharpening needed = lower weight)
    vec3  rcp_mx = vec3(1.0) / max(mx, vec3(1e-4));
    vec3  amp    = clamp(min(mn, 2.0 - mx) * rcp_mx, 0.0, 1.0);
    // Map sharpness [0,1] to weight: 0 -> sqrt(0.125) (max), 1 -> 0 (no sharpening)
    amp = sqrt(amp);
    float peak = -3.0 / 8.0 * sharpness - 1.0 / 8.0 * (1.0 - sharpness);
    // peak is in range [-0.125, -0.5]
    vec3  w    = amp * peak;  // negative kernel weight (sharpening via unsharp mask)

    // Sharpen: centre boosted, cross neighbours reduced by weight
    // Normalised: sum of weights = 1 + 4*w
    vec3  result = (e * (1.0 - 4.0 * w) + (b + d_ + f + h) * w)
                 / max(1.0 - 4.0 * w + 4.0 * w, 1e-4);
    // Clamp to [mn, mx] to prevent ringing
    result = clamp(result, mn, mx);

    imageStore(output_img, px, vec4(result, 1.0));
}
```

**Use cases** — mandatory post-pass after any spatial or temporal upscaler (DLSS, FSR 1/2, XeSS); also useful standalone after TAA to recover perceived sharpness lost to temporal blending.  
**Reference** — [AMD FidelityFX CAS source (GPUOpen)](https://github.com/GPUOpen-Effects/FidelityFX-CAS); [Drobot: FidelityFX CAS (GDC 2019)](https://gpuopen.com/fidelityfx-cas/)

---

### Color Science in Shaders

HDR rendering and wide-color-gamut display output require correct color space transforms at several points in the pipeline: on texture read (decode), during lighting accumulation (scene-linear), before display output (encode). Getting any transform wrong results in crushed shadows, blown highlights, or wrong hue on wide-gamut displays.

**Shader stage**: color space transforms appear in the **fragment shader** (texture decode, final tone-map and encode pass) and in **compute** (post-processing chain).

#### sRGB ↔ Linear Conversion
Most albedo textures are stored in sRGB; the GPU can linearise on read via `VK_FORMAT_R8G8B8A8_SRGB`, but manual conversion is needed for data stored in non-SRGB formats or for output encoding.

```glsl
// sRGB → linear (manual, for formats without hardware decode)
vec3 srgb_to_linear(vec3 c) {
    return mix(c / 12.92,
               pow((c + 0.055) / 1.055, vec3(2.4)),
               step(0.04045, c));
}

// Linear → sRGB (for manual output encoding)
vec3 linear_to_srgb(vec3 c) {
    c = clamp(c, 0.0, 1.0);
    return mix(c * 12.92,
               1.055 * pow(c, vec3(1.0 / 2.4)) - 0.055,
               step(0.0031308, c));
}
```

**When needed** — output framebuffer is `VK_FORMAT_R8G8B8A8_UNORM` (not SRGB): encode manually. Input texture is `VK_FORMAT_R8G8B8A8_UNORM` but contains sRGB data: decode manually.

#### BT.709 ↔ BT.2020 Color Space Transform
Rec.2020 is the wide-color-gamut space used by HDR10 displays. Converting scene-linear BT.709 primaries to BT.2020 uses a fixed 3×3 matrix (derived from the primaries' xy chromaticities).

```glsl
// BT.709 scene-linear → BT.2020 scene-linear (for HDR10 output)
const mat3 BT709_TO_BT2020 = mat3(
    0.6274040,  0.0690970,  0.0163916,
    0.3292820,  0.9195400,  0.0880132,
    0.0433136,  0.0113612,  0.8955950
);
vec3 c_2020 = BT709_TO_BT2020 * c_709;
```

#### ACEScg and ACES Tone Mapping
ACEScg is the ACES scene-linear working space used in film and HDR game pipelines. The ACES RRT (Reference Rendering Transform) + ODT (Output Display Transform) maps ACEScg scene linear to display-referred BT.709 or BT.2020 with a perceptually pleasing tone curve including a subtle S-curve in shadows and a roll-off in highlights.

```glsl
// Narkowicz ACES approximation (fast, BT.709 output)
vec3 aces_approx(vec3 x) {
    const float a = 2.51, b = 0.03, c = 2.43, d = 0.59, e = 0.14;
    return clamp((x*(a*x+b))/(x*(c*x+d)+e), 0.0, 1.0);
}

// Full ACES RRT+ODT: apply input transform, then RRT, then ODT
// Requires the ACES matrices (BT.709→ACEScg, ACEScg→BT.709)
const mat3 AP1_TO_AP0 = mat3(...); // ACEScg → ACES2065-1
const mat3 AP0_TO_AP1 = mat3(...); // ACES2065-1 → ACEScg
```

**Reference** — [ACES GitHub: aces-dev](https://github.com/ampas/aces-dev); [Narkowicz: ACES Filmic Tone Mapping Curve](https://knarkowicz.wordpress.com/2016/01/06/aces-filmic-tone-mapping-curve/)

#### PQ (ST 2084) Transfer Function — HDR10 Output
The Perceptual Quantizer EOTF maps scene-linear luminance (in nits) to a 0–1 PQ signal for HDR10 displays. The display reconstructs the nit value from the PQ signal and drives the panel at that brightness. Correct PQ encoding is required for HDR10 Vulkan swapchain output (`VK_COLOR_SPACE_HDR10_ST2084_EXT`).

```glsl
// Linear luminance (nits) → PQ signal [0,1]
vec3 linear_to_pq(vec3 L_nits) {
    const float m1 = 2610.0 / 16384.0;
    const float m2 = 2523.0 / 4096.0 * 128.0;
    const float c1 = 3424.0 / 4096.0;
    const float c2 = 2413.0 / 4096.0 * 32.0;
    const float c3 = 2392.0 / 4096.0 * 32.0;
    vec3 Lm = pow(L_nits / 10000.0, vec3(m1));
    return pow((c1 + c2 * Lm) / (1.0 + c3 * Lm), vec3(m2));
}
```

**Reference** — [SMPTE ST 2084 Specification](https://ieeexplore.ieee.org/document/7291452); [Wayland color-management-v1 protocol](https://gitlab.freedesktop.org/wayland/wayland-protocols)

#### HLG (Hybrid Log-Gamma)
HLG is the HDR broadcast standard (BBC/NHK) that is backward-compatible with SDR displays. Unlike PQ, HLG is a relative transfer function (no absolute nit mapping); useful for live video pipelines.

```glsl
vec3 linear_to_hlg(vec3 L) {
    const float a = 0.17883277, b = 0.28466892, c_hlg = 0.55991073;
    return mix(sqrt(3.0 * L),
               a * log(12.0 * L - b) + c_hlg,
               step(vec3(1.0 / 12.0), L));
}
```

**Reference** — [BBC R&D: Hybrid Log-Gamma](https://www.bbc.co.uk/rd/publications/whitepaper309)

---

### Color Grading and 3D LUT

Color grading maps the scene-linear HDR values (after tone mapping to SDR or HDR display output) through a 3D lookup table (LUT) that encodes the creative colour grade: saturation, contrast, colour casts, per-channel curves, and any film print emulation. The 3D LUT is a 32³ or 64³ cube of RGB output values indexed by (R, G, B) input; the GPU samples it with trilinear interpolation in a single texture fetch per pixel.

**Shader stage**: 3D LUT application is a **fragment or compute** post-process, the last colour-space operation before display encoding.

#### 3D LUT Application
```glsl
// 3D LUT application — fragment shader
layout(set=0, binding=0) uniform sampler2D ldr_scene;  // tone-mapped [0,1]
layout(set=0, binding=1) uniform sampler3D color_lut;  // 32³ or 64³ RGBA8/16F

uniform float lut_size;    // e.g. 32.0 — number of entries per axis
uniform float lut_scale;   // (lut_size - 1.0) / lut_size
uniform float lut_offset;  // 0.5 / lut_size

layout(location=0) out vec3 graded;

void main() {
    vec2  uv    = gl_FragCoord.xy / resolution;
    vec3  col   = texture(ldr_scene, uv).rgb;

    // Map [0,1] input to LUT texel centre coordinates
    vec3  lut_uv = col * lut_scale + lut_offset;
    graded = texture(color_lut, lut_uv).rgb;
}
```

#### LUT Baking on CPU / Offline
The 3D LUT is precomputed by applying the full colour grade (ASC CDL, film curve, per-channel splines, CST) to every texel of an identity LUT. The result is exported as a `.cube` file (DaVinci Resolve, Nuke standard) and uploaded as a `VK_FORMAT_R16G16B16A16_SFLOAT` 3D texture at load time.

#### Neutral/Identity LUT Generation
At content load time, generate an identity LUT (no grade applied) to verify the pipeline is correct before injecting the creative grade.

```glsl
// Identity LUT generation — compute shader
layout(set=0, binding=0, rgba16f) uniform image3D identity_lut;
uniform int lut_size;  // 32 or 64

void main() {
    ivec3 px  = ivec3(gl_GlobalInvocationID.xyz);
    vec3  col = (vec3(px) + 0.5) / float(lut_size);
    imageStore(identity_lut, px, vec4(col, 1.0));
}
```

#### Per-Channel Curve (In-Shader Grade)
For run-time-adjustable grades (e.g., in a game with day/night colour temperature shifts), apply a simple per-channel lift/gamma/gain before the LUT:

```glsl
// Lift-Gamma-Gain colour grade
vec3 lift_gamma_gain(vec3 col, vec3 lift, vec3 gamma, vec3 gain) {
    col = col * gain + lift;
    return pow(max(col, vec3(0.0)), 1.0 / max(gamma, vec3(0.01)));
}
```

**Reference** — [Hable: Filmic Tonemapping (2010)](http://filmicworlds.com/blog/filmic-tonemapping-operators/); [ACES: Academy Color Encoding System](https://acescentral.com/)

---

### Histogram-Based Auto-Exposure

Auto-exposure (AE) adapts the scene's exposure value (EV) so the average luminance of the rendered frame maps to a target midtone, mimicking the eye's adaptation to changing light levels. GPU AE uses a luminance histogram computed from the full HDR frame each frame, filters it to a stable average, and smoothly adapts the exposure over time using the adaptation speed parameter.

**Shader stage**: histogram build and reduce run as **compute**. The adapted EV is stored in a 1×1 `R32F` buffer read by the tone-mapping pass.

#### Luminance Histogram Build
Build a 256-bin histogram of log-luminance values in a single compute dispatch using shared-memory atomics.

```glsl
// Histogram build — one thread per pixel, shared histogram per workgroup
layout(local_size_x=16, local_size_y=16) in;
layout(set=0, binding=0) uniform sampler2D hdr_scene;
layout(set=0, binding=1) buffer HistogramBuffer { uint bins[256]; };

shared uint s_hist[256];

uniform float lum_min_log2;   // e.g. -10.0 (minimum log2 luminance)
uniform float lum_max_log2;   // e.g.   4.0 (maximum log2 luminance)
uniform float lum_range_inv;  // 1.0 / (lum_max_log2 - lum_min_log2)

void main() {
    uint lid = gl_LocalInvocationIndex;
    if (lid < 256u) s_hist[lid] = 0u;
    barrier();

    ivec2 px = ivec2(gl_GlobalInvocationID.xy);
    if (all(lessThan(px, textureSize(hdr_scene, 0)))) {
        vec3  col = texelFetch(hdr_scene, px, 0).rgb;
        float lum = dot(col, vec3(0.2126, 0.7152, 0.0722));
        if (lum > 0.0) {
            float log_lum = log2(lum);
            float t       = clamp((log_lum - lum_min_log2) * lum_range_inv, 0.0, 1.0);
            uint  bin     = uint(t * 255.0);
            atomicAdd(s_hist[bin], 1u);
        }
    }
    barrier();

    // Accumulate workgroup histogram into global histogram
    if (lid < 256u) atomicAdd(bins[lid], s_hist[lid]);
}
```

#### Histogram Average and EV Adaptation
Compute the weighted average log-luminance from the histogram (excluding the darkest and brightest pixels), then adapt the current EV toward the target using a time-weighted blend.

```glsl
// Histogram reduce and EV adaptation — single-thread compute (256 threads total)
layout(local_size_x=256) in;
layout(set=0, binding=0) buffer HistogramBuffer { uint bins[256]; };
layout(set=0, binding=1) buffer ExposureBuffer  { float ev_current; };

uniform float lum_min_log2;
uniform float lum_range_inv;
uniform float low_cutoff;      // fraction of pixels to discard from dark end (e.g. 0.6)
uniform float high_cutoff;     // fraction to discard from bright end (e.g. 0.95)
uniform float adaptation_speed;  // seconds to adapt half-way (e.g. 1.0)
uniform float delta_time;
uniform float ev_compensation; // artist bias (stops), e.g. 0.0
uniform uint  pixel_count;

shared float s_accum[256];

void main() {
    uint bin    = gl_LocalInvocationIndex;
    float count = float(bins[bin]);

    // Compute cumulative count for cutoff
    // (simplified: accumulate in shared memory then reduce)
    s_accum[bin] = count * (float(bin) / 255.0 / lum_range_inv + lum_min_log2);
    barrier();

    // Parallel reduction — sum weighted log luminances
    for (uint stride = 128u; stride > 0u; stride >>= 1u) {
        if (bin < stride) s_accum[bin] += s_accum[bin + stride];
        barrier();
    }

    if (bin == 0u) {
        float avg_log_lum = s_accum[0] / float(pixel_count);
        float target_ev   = avg_log_lum + ev_compensation;
        // Smooth adaptation: exponential blend
        float adapt_factor = 1.0 - exp(-delta_time / max(adaptation_speed, 0.001));
        ev_current = mix(ev_current, target_ev, adapt_factor);
        // Clear histogram for next frame
        for (uint i = 0u; i < 256u; ++i) bins[i] = 0u;
    }
}
```

**EV to exposure scalar**: in the tone-mapping pass, `exposure = exp2(-ev_current)` (more positive EV = darker image). The EV buffer is read as a storage buffer or via a descriptor update.

**Reference** — [Lagarde & de Rousiers: Moving Frostbite to PBR (SIGGRAPH 2014, §4 Exposure)](https://media.contentapi.ea.com/content/dam/eacom/frostbite/files/course-notes-moving-frostbite-to-pbr-v32.pdf); [Karis: Automatic Exposure (2013)](https://placeholderart.wordpress.com/2014/11/21/implementing-a-physically-based-camera-manual-exposure/)

---

### Ordered Dithering and Output Quantization

When a smooth gradient is written to an 8-bit render target (`VK_FORMAT_R8G8B8A8_UNORM`), the 8-bit quantization step (1/255 ≈ 0.004) is visible as discrete colour bands — posterisation. **Dithering** adds a carefully chosen noise pattern to the signal before quantization, distributing the quantization error spatially rather than temporally, making it perceptually invisible. This is required whenever an HDR or high-precision value is written to an 8-bit surface: the final output stage, lightmap encoding, and screen-door transparency thresholds.

**Shader stage**: dithering is applied in the **fragment shader** immediately before writing the final output value, as a one-line addition before the implicit or explicit quantization.

#### Bayer Matrix Ordered Dithering
A Bayer matrix `M_n` is a spatially tiled threshold matrix whose entries are distributed to maximally spread quantization error across pixels. An 8×8 Bayer matrix is common; it is typically stored as a 64-entry `uint` lookup or computed analytically.

```glsl
// 4×4 Bayer matrix (normalised to [0,1) range)
float bayer4x4(ivec2 px) {
    const float M[16] = float[](
         0.0/16.0,  8.0/16.0,  2.0/16.0, 10.0/16.0,
        12.0/16.0,  4.0/16.0, 14.0/16.0,  6.0/16.0,
         3.0/16.0, 11.0/16.0,  1.0/16.0,  9.0/16.0,
        15.0/16.0,  7.0/16.0, 13.0/16.0,  5.0/16.0
    );
    return M[(px.y % 4) * 4 + (px.x % 4)];
}

// Apply dithering before writing to 8-bit target
vec3 dither_output(vec3 color, ivec2 px) {
    float threshold = bayer4x4(px) - 0.5;          // centre around zero
    color += threshold / 255.0;                      // one quantization step
    return color;                                     // will be rounded on write
}
```

#### Blue-Noise Dithered Quantization
Blue-noise dithering (using the precomputed blue noise texture from §29) spreads error at high spatial frequencies rather than the grid-aligned pattern of Bayer dithering. Preferred for organic gradients; Bayer is preferred for geometric patterns.

```glsl
layout(set=0, binding=0) uniform sampler2D blue_noise;  // 64×64 void-and-cluster
uniform uint frame_index;

vec3 dither_blue_noise(vec3 color, ivec2 px) {
    float noise = texelFetch(blue_noise, px % 64, 0).r;
    // Temporal decorrelation
    noise = fract(noise + float(frame_index) * 0.61803398875);
    color += (noise - 0.5) / 255.0;
    return color;
}
```

#### Temporal Dithering
Alternates dither thresholds across frames so that TAA's temporal accumulation removes the dithered noise entirely, leaving a clean gradient. Each pixel receives a different threshold each frame (using frame_index modulo the dither matrix period). The average over N frames has zero dither error.

```glsl
// Temporal Bayer — shift spatial phase by frame index
float bayer_temporal(ivec2 px, uint frame) {
    ivec2 shifted = (px + ivec2(frame * 17u, frame * 31u)) % 4;  // coprime shifts
    return bayer4x4(shifted) - 0.5;
}
```

**Applications beyond output**: dithered alpha thresholds for stochastic transparency (§44); dithered LOD crossfade thresholds (§44); dithered AO quantization when storing to 8-bit G-Buffer.  
**Reference** — [Roberts: The Unreasonable Effectiveness of Quasirandom Sequences (2018)](http://extremelearning.com.au/unreasonable-effectiveness-of-quasirandom-sequences/); [Ulichney: Digital Halftoning (MIT Press 1987)]

---


---

## XIII. GPU Compute Primitives

These are the low-level algorithmic building blocks that appear inside many of the higher-level techniques above. Radix sort and bitonic sort underpin BVH construction, depth-sorted OIT linked-list management, and particle simulation. Wave/subgroup intrinsics enable prefix sums, reductions, and ballot operations without shared-memory round-trips. Cooperative matrices expose Tensor Core throughput for ML inference inside shaders. Sampling theory and low-discrepancy sequences are the mathematical foundation of every Monte Carlo renderer. Reversed-Z is a depth-buffer precision technique that every rasterisation renderer should apply. Understanding these primitives is prerequisite to implementing or optimising any of the algorithms in the preceding categories.

### GPU Compute Algorithm Primitives

These are the fundamental data-parallel algorithms that underpin GPU-driven rendering, simulation, and post-processing — not visual effects, but computational building blocks. Parallel prefix sum (scan) enables stream compaction; radix sort enables depth-sorted transparency and BVH construction; histogram enables auto-exposure; Hi-Z generation enables occlusion culling. They are invoked from compute shaders (`layout(local_size_x = N) in`) and operate on structured buffers rather than render targets. Correctness requires careful attention to wave-level synchronisation (`barrier()`, `memoryBarrierBuffer()`) and the distinction between wave-uniform and non-uniform execution paths.

#### Parallel Prefix Sum (Scan)
Computes the running sum of an array in O(log N) parallel steps using an up-sweep and down-sweep tree reduction, making every thread's output depend on all prior elements.

**Use cases** — stream compaction (visible draw list from GPU culling); histogram cumulation; memory allocation for variable-length outputs.  
**Reference** — [Blelloch: Prefix Sums and Their Applications (CMU 1990)](https://www.cs.cmu.edu/~guyb/papers/Ble93.pdf) | [GPU Gems 3 Ch39: Parallel Prefix Sum](https://developer.nvidia.com/gpugems/gpugems3/part-vi-gpu-computing/chapter-39-parallel-prefix-sum-scan-cuda)

#### Parallel Reduction
Reduces an array to a single value (sum, max, min, average) by repeatedly halving the active thread count until one value remains; requires two-phase execution for arrays larger than one thread block.

**Use cases** — auto-exposure (luminance average); shadow frustum AABB (max depth); GPU occlusion query aggregation.  
**Reference** — [Harris: Optimising Parallel Reduction in CUDA (NVIDIA)](https://developer.download.nvidia.com/assets/cuda/files/reduction.pdf)

#### Bitonic Sort
An oblivious comparison network that sorts N elements in O(N log² N) comparisons, all of which are data-independent and thus suitable for GPU SIMD execution.

**Use cases** — transparency sort (back-to-front order for alpha); BVH leaf sort; small-to-medium arrays where radix sort setup cost is too high.  
**Reference** — [GPU Gems 2 Ch46: Improved GPU Sorting](https://developer.nvidia.com/gpugems/gpugems2/part-vi-simulation-and-numerical-algorithms/chapter-46-improved-gpu-sorting)

#### Radix Sort
Sorts by digit (typically 8-bit passes) using a histogram + prefix sum + scatter pattern, achieving O(kN) where k is the number of passes; the fastest GPU sort for large integer or float keys.

**Use cases** — particle systems (sort by depth); BVH construction; any sort over 64k+ elements where bitonic is too slow.  
**Key variants** — CUB's DeviceRadixSort (CUDA); FidelityFX Sort (Vulkan, AMD GPUOpen); onesweep variant (single-pass, 2022).  
**Reference** — [AMD GPUOpen: FidelityFX Sort](https://gpuopen.com/fidelityfx-sort/)

#### Stream Compaction
Removes rejected elements from an array using a parallel prefix sum of a boolean predicate array, writing only accepted elements to a packed output — the GPU equivalent of `std::remove_if`.

**Use cases** — GPU culling output (write only visible draw commands); particle death (remove dead particles); any GPU-generated variable-length list.  
**Reference** — [GPU Gems 3 Ch39](https://developer.nvidia.com/gpugems/gpugems3/part-vi-gpu-computing/chapter-39-parallel-prefix-sum-scan-cuda)

#### GPU Histogram
Accumulates per-bin counts using atomic operations or wave-level intrinsics; the naive approach (global atomics) is bandwidth-limited and requires local shared-memory staging.

**Use cases** — auto-exposure from scene luminance; colour grading LUT generation; HDR tonemap curve fitting.  
**Reference** — [Bartlomiej Wronski: Automatic Exposure](https://bartwronski.com/2016/10/07/automatic-exposure-using-a-tonemapping-curve/)

#### Hi-Z Mip Generation
Builds a mip chain of the depth buffer where each mip stores the maximum (farthest) depth over a 2×2 region of the level above, enabling conservative occlusion testing at coarse resolution.

**Use cases** — two-phase GPU occlusion culling (Ch154); SSR Hi-Z ray march; software occlusion queries.  
**Reference** — [Andersson: Parallel Graphics in Frostbite (SIGGRAPH 2009)](https://advances.realtimerendering.com/s2009/)

---

### Wave and Subgroup Intrinsics

Vulkan subgroup operations (`VK_KHR_shader_subgroup_extended_types`, GLSL `GL_KHR_shader_subgroup_*`) expose direct warp/wavefront communication between threads executing in lockstep on the GPU's SIMD unit. This eliminates the need for shared memory round-trips for many common reductions, prefix sums, and compaction patterns. On NVIDIA hardware a subgroup is 32 threads (a warp); on AMD it is 32 or 64 (a wavefront); on mobile it is typically 4 or 8.

**Shader stage**: subgroup intrinsics are available in **compute, vertex, fragment, and mesh shaders** wherever `gl_SubgroupSize` > 1.

#### Subgroup Reduction
Computes a reduction (sum, min, max, AND, OR) across all active lanes in a subgroup in a single instruction — no shared memory, no barrier.

```glsl
#extension GL_KHR_shader_subgroup_arithmetic : require

layout(local_size_x=256) in;
shared float s_partial[8];  // one slot per subgroup

void main() {
    float val = load_data(gl_GlobalInvocationID.x);

    // Subgroup-level reduction — single instruction, no barrier
    float sub_sum = subgroupAdd(val);

    // Only one thread per subgroup writes to shared memory
    if (subgroupElect())
        s_partial[gl_SubgroupID] = sub_sum;

    barrier();

    // Final reduction across subgroups by first subgroup
    if (gl_SubgroupID == 0) {
        float partial = (gl_SubgroupInvocationID < gl_NumSubgroups) ?
            s_partial[gl_SubgroupInvocationID] : 0.0;
        float total = subgroupAdd(partial);
        if (subgroupElect()) output_sum = total;
    }
}
```

**Speedup vs shared-memory-only**: 2–4× on typical workloads; eliminates one `barrier()` per reduction level.

#### Subgroup Ballot and Compaction
`subgroupBallot(condition)` returns a bitmask of which lanes satisfy the condition. `subgroupBallotBitCount` counts set bits; `subgroupBallotExclusiveBitCount` gives each lane its compacted output index — the building block of warp-efficient stream compaction.

```glsl
#extension GL_KHR_shader_subgroup_ballot : require

bool active = test_condition(gl_GlobalInvocationID.x);
uvec4 ballot = subgroupBallot(active);
uint  count  = subgroupBallotBitCount(ballot);
uint  idx    = subgroupBallotExclusiveBitCount(ballot);

if (active) output_buf[base + idx] = compute_result();
if (subgroupElect()) atomicAdd(total_count, count);
```

**Use cases** — GPU-driven rendering task shader lane compaction (see §32), particle death compaction, sparse light list building.

#### Subgroup Shuffle
Reads a value from an arbitrary lane within the subgroup without shared memory. Enables butterfly networks for prefix sums, efficient transpose, and warp-level matrix operations.

```glsl
#extension GL_KHR_shader_subgroup_shuffle : require

// Parallel prefix sum within a subgroup (4 steps for 16-wide subgroup)
float v = in_val;
for (uint offset = 1; offset < gl_SubgroupSize; offset <<= 1) {
    float neighbor = subgroupShuffleUp(v, offset);
    if (gl_SubgroupInvocationID >= offset) v += neighbor;
}
// v now contains inclusive prefix sum for this lane
```

#### Quad Operations
`dFdx`/`dFdy` operate on 2×2 pixel quads; subgroup quad ops expose the same mechanism explicitly: `subgroupQuadSwapHorizontal`, `subgroupQuadSwapVertical`, `subgroupQuadBroadcast`. Useful for computing finite-difference normals or depth derivatives in compute shaders that process the framebuffer tile.

**Reference** — [Vulkan Subgroup Tutorial (Khronos Blog)](https://www.khronos.org/blog/vulkan-subgroup-tutorial); [GLSL_KHR_shader_subgroup spec](https://registry.khronos.org/OpenGL/extensions/KHR/KHR_shader_subgroup.txt)

---

### GPU Radix Sort

Sorting is a fundamental GPU primitive needed by many rendering algorithms: LBVH Morton code sorting (§42), particle Z-sort for transparency, shadow map priority, visibility buffer triangle ID sorting. **Radix sort** is the fastest comparison-free GPU sort: it makes `K` passes over the data (one per `b`-bit digit), each computing a prefix-sum histogram to determine each element's output position. On GPU, the most efficient variant uses **single-pass decoupled lookback** (Merrill & Garland 2016) which avoids inter-pass global synchronisation barriers.

**Shader stage**: all radix sort passes run as **compute shaders**. A typical 32-bit key sort with 4-bit digits requires 8 passes.

#### Per-Workgroup Histogram (Upsweep)
Each workgroup counts how many elements fall in each digit bucket for its assigned input range, writing partial histograms to a global histogram buffer.

```glsl
// Radix sort upsweep — per-workgroup histogram (4-bit digit, 16 buckets)
layout(local_size_x=256) in;
layout(set=0, binding=0) readonly  buffer Keys       { uint keys_in[]; };
layout(set=0, binding=1) writeonly buffer Histograms { uint histo[]; };  // [num_wg * 16]

shared uint s_hist[16];

uniform uint num_elements;
uniform uint digit_shift;  // 0, 4, 8, 12, 16, 20, 24, 28 for 8 passes

void main() {
    uint lid  = gl_LocalInvocationIndex;
    uint gid  = gl_GlobalInvocationID.x;
    uint wg   = gl_WorkGroupID.x;

    if (lid < 16u) s_hist[lid] = 0u;
    barrier();

    if (gid < num_elements) {
        uint digit = (keys_in[gid] >> digit_shift) & 0xFu;
        atomicAdd(s_hist[digit], 1u);
    }
    barrier();

    if (lid < 16u)
        histo[wg * 16u + lid] = s_hist[lid];
}
```

#### Global Prefix Sum (Scan)
A separate pass computes an exclusive prefix sum across all workgroup histograms for each bucket — the output position offset for each (workgroup, bucket) pair.

```glsl
// Global prefix scan over histogram buckets — one thread per bucket per workgroup
// (simplified: in practice use a 2-level tree reduction for large workgroup counts)
layout(local_size_x=256) in;
layout(set=0, binding=0) buffer Histograms { uint histo[]; };  // in-place scan
uniform uint num_workgroups;

shared uint s_val[256];

void main() {
    uint bucket = gl_GlobalInvocationID.x;  // 0..15
    // Scan across workgroup counts for this bucket
    uint accum = 0u;
    for (uint wg = 0u; wg < num_workgroups; ++wg) {
        uint idx = wg * 16u + bucket;
        uint cnt = histo[idx];
        histo[idx] = accum;  // exclusive prefix
        accum += cnt;
    }
}
```

#### Scatter (Downsweep)
Each element reads its digit's scanned position offset, adds its local rank within the workgroup, and writes to the output array.

```glsl
// Radix sort scatter — reorder keys from input to output using prefix sums
layout(local_size_x=256) in;
layout(set=0, binding=0) readonly  buffer KeysIn     { uint keys_in[]; };
layout(set=0, binding=1) writeonly buffer KeysOut    { uint keys_out[]; };
layout(set=0, binding=2) readonly  buffer ValuesIn   { uint vals_in[]; };
layout(set=0, binding=3) writeonly buffer ValuesOut  { uint vals_out[]; };
layout(set=0, binding=4) readonly  buffer Histograms { uint histo[]; };

shared uint s_rank[256];  // local rank within workgroup per thread

uniform uint num_elements;
uniform uint digit_shift;

void main() {
    uint lid = gl_LocalInvocationIndex;
    uint gid = gl_GlobalInvocationID.x;
    uint wg  = gl_WorkGroupID.x;

    uint key    = (gid < num_elements) ? keys_in[gid] : 0xFFFFFFFFu;
    uint digit  = (key >> digit_shift) & 0xFu;

    // Local rank: count how many earlier threads have the same digit (ballot prefix)
    uint rank = 0u;
    for (uint d = 0u; d < 16u; ++d) {
        uvec4 ballot = subgroupBallot(digit == d);
        if (digit == d) rank = subgroupBallotExclusiveBitCount(ballot);
    }

    if (gid < num_elements) {
        uint dst = histo[wg * 16u + digit] + rank;
        keys_out[dst]  = key;
        vals_out[dst]  = vals_in[gid];
    }
}
```

A full implementation alternates ping-pong between two key/value buffers for each of the 8 digit passes. The result is a fully sorted array in O(n) GPU time, typically outperforming GPU comparison sorts for n > 100k.

**Reference** — [Merrill & Garland: Single-pass Parallel Prefix Scan with Decoupled Lookback (NVIDIA TR 2016)](https://research.nvidia.com/publication/2016-03_single-pass-parallel-prefix-scan-decoupled-look-back); [CUB: cub::DeviceRadixSort](https://nvlabs.github.io/cub/)

---

### Parallel Bitonic Sort

**Bitonic sort** is a comparison network that sorts N elements using O(N log² N) compare-and-swap operations. Unlike radix sort (§80) it works for any data type with a comparison operator and is simple to implement in a single compute shader dispatch: all compare-and-swap pairs are independent within each pass and can execute in parallel. It is the standard choice for sorting within a single workgroup (particle depth sort, draw call reordering by material, small N < 4096).

**Shader stage**: runs as a **compute shader**, either within a single dispatch for small N (using shared memory) or across multiple dispatches for large N.

#### Shared-Memory Bitonic Sort (N ≤ workgroup size × 2)
```glsl
// Bitonic sort — single workgroup, shared memory (N must be power of 2, N ≤ 2048)
layout(local_size_x=1024) in;

layout(set=0, binding=0) buffer Keys   { float keys[]; };
layout(set=0, binding=1) buffer Values { uint  vals[]; };

shared float s_keys[2048];
shared uint  s_vals[2048];

uniform uint N;  // number of elements, power of 2

void main() {
    uint lid = gl_LocalInvocationIndex;

    // Load into shared memory
    if (lid < N)       { s_keys[lid]     = keys[lid];     s_vals[lid]     = vals[lid]; }
    if (lid + 1024 < N){ s_keys[lid+1024]= keys[lid+1024];s_vals[lid+1024]= vals[lid+1024]; }
    barrier();

    // Bitonic sort network
    for (uint k = 2u; k <= N; k <<= 1u) {
        for (uint j = k >> 1u; j >= 1u; j >>= 1u) {
            uint i = lid;
            uint l = i ^ j;
            if (l > i) {
                bool ascending = ((i & k) == 0u);
                if ((ascending && s_keys[i] > s_keys[l]) ||
                    (!ascending && s_keys[i] < s_keys[l])) {
                    // Swap keys and values
                    float tmp_k = s_keys[i]; s_keys[i] = s_keys[l]; s_keys[l] = tmp_k;
                    uint  tmp_v = s_vals[i]; s_vals[i] = s_vals[l]; s_vals[l] = tmp_v;
                }
            }
            barrier();
        }
    }

    // Write back
    if (lid < N)        { keys[lid]      = s_keys[lid];      vals[lid]      = s_vals[lid]; }
    if (lid + 1024 < N) { keys[lid+1024] = s_keys[lid+1024]; vals[lid+1024] = s_vals[lid+1024]; }
}
```

#### Multi-Pass Bitonic Sort (Large N)
For N > workgroup capacity, split into multiple dispatches — each dispatch executes one (k, j) pair of the bitonic network using global memory. The compare-and-swap pattern is identical; the index calculation uses `gl_GlobalInvocationID`.

```glsl
// Multi-pass bitonic sort — one dispatch per (k, j) pair
layout(local_size_x=256) in;
layout(set=0, binding=0) buffer Keys { float keys[]; };
layout(set=0, binding=1) buffer Vals { uint  vals[]; };

uniform uint k_uniform;   // current k (outer loop, set via push constant each dispatch)
uniform uint j_uniform;   // current j (inner loop)

void main() {
    uint i = gl_GlobalInvocationID.x;
    uint l = i ^ j_uniform;
    if (l <= i) return;
    bool ascending = ((i & k_uniform) == 0u);
    if ((ascending && keys[i] > keys[l]) || (!ascending && keys[i] < keys[l])) {
        float tk = keys[i]; keys[i] = keys[l]; keys[l] = tk;
        uint  tv = vals[i]; vals[i] = vals[l]; vals[l] = tv;
    }
}
```

**Use cases** — particle Z-sort (§24); transparent draw call reordering; cluster light list ordering; any per-frame sort of N < 100k where radix sort setup cost dominates.  
**Reference** — [Batcher: Sorting Networks and Their Applications (AFIPS 1968)](https://dl.acm.org/doi/10.1145/1468075.1468121); [NVIDIA GPU Gems 2: Fast Sorting (2005)](https://developer.nvidia.com/gpugems/gpugems2/part-vi-simulation-and-numerical-algorithms/chapter-46-improved-gpu-sorting)

---

### Cooperative Matrix and Tensor Core Operations

Modern GPUs (NVIDIA Turing+, AMD RDNA3+, Intel Arc) contain dedicated matrix-multiply-accumulate (MMA) hardware — tensor cores on NVIDIA, matrix cores on AMD. `VK_KHR_cooperative_matrix` exposes these as GLSL/SPIR-V operations: a cooperative matrix multiply `C += A × B` executes as a single SPIR-V instruction across an entire subgroup (warp), completing a 16×16 half-precision matrix multiply in ~4 clock cycles rather than ~256.

**Shader stage**: cooperative matrix operations run in **compute shaders** using the `GL_KHR_cooperative_matrix` GLSL extension (maps to `OpCooperativeMatrixMulAddKHR` in SPIR-V). The subgroup must be the correct size (typically 32 threads on NVIDIA) and must all execute the cooperative instruction together.

**Why it matters**: the tiny MLP inference in §46 uses scalar loops — O(W²) multiply-adds per layer, executed serially in registers. With cooperative matrices, an entire 16×16 weight layer executes in one instruction, achieving peak tensor core throughput (e.g., 1000 TFLOPS on RTX 4090 at FP16 vs ~80 TFLOPS scalar).

#### Cooperative Matrix Multiply (GEMM)
Computes `C = A × B + C` where `A`, `B`, `C` are cooperative matrix types loaded from shared memory or buffers. Each cooperative matrix is distributed across the subgroup — no single thread holds the full matrix.

```glsl
#extension GL_KHR_cooperative_matrix : require
#extension GL_KHR_shader_subgroup_basic : require

layout(local_size_x = 32) in;  // one subgroup per workgroup

layout(set=0, binding=0) readonly buffer MatA { float16_t A[]; };
layout(set=0, binding=1) readonly buffer MatB { float16_t B[]; };
layout(set=0, binding=2) buffer          MatC { float      C[]; };

uniform uint M, N, K;   // matrix dimensions: A is M×K, B is K×N, C is M×N

void main() {
    // Each workgroup computes one 16×16 tile of C
    uint tile_row = gl_WorkGroupID.y;
    uint tile_col = gl_WorkGroupID.x;

    // Declare cooperative matrix types (16×16 tiles)
    coopmat<float16_t, gl_ScopeSubgroup, 16, 16, gl_MatrixUseA> cmat_a;
    coopmat<float16_t, gl_ScopeSubgroup, 16, 16, gl_MatrixUseB> cmat_b;
    coopmat<float,     gl_ScopeSubgroup, 16, 16, gl_MatrixUseAccumulator> cmat_c;

    // Zero accumulator
    coopMatFillKHR(cmat_c, 0.0);

    // Accumulate over K dimension in 16-wide tiles
    for (uint k = 0; k < K; k += 16) {
        uint a_offset = tile_row * 16 * K + k;
        uint b_offset = k * N + tile_col * 16;

        coopMatLoadKHR(cmat_a, A, a_offset, K, gl_CooperativeMatrixLayoutRowMajorKHR);
        coopMatLoadKHR(cmat_b, B, b_offset, N, gl_CooperativeMatrixLayoutRowMajorKHR);
        coopMatMulAddKHR(cmat_c, cmat_a, cmat_b, cmat_c);  // C += A × B — tensor core
    }

    // Store result tile
    uint c_offset = tile_row * 16 * N + tile_col * 16;
    coopMatStoreKHR(cmat_c, C, c_offset, N, gl_CooperativeMatrixLayoutRowMajorKHR);
}
```

#### Tiny MLP Inference with Cooperative Matrices
Replace the scalar loops in the tiny MLP inference (§46) with cooperative matrix multiplications. Batch multiple pixel queries into a `M×IN_DIM` input matrix, execute one `coopMatMulAddKHR` per layer, then scatter output rows back to pixels. The cooperative matrix tiles the work across the subgroup automatically.

**Sizing**: cooperative matrix requires M, N, K to be multiples of 16. Pad the feature vector and layer width to the next multiple of 16; pad the pixel batch to the next multiple of 16.

**Use cases** — accelerated tiny MLP inference for neural textures and NeRF; neural radiance cache query throughput; any GPU compute workload involving dense matrix products.  
**Reference** — [Vulkan: VK_KHR_cooperative_matrix specification](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_cooperative_matrix.html); [NVIDIA: Programming Tensor Cores in CUDA](https://developer.nvidia.com/blog/programming-tensor-cores-cuda-9/)

---

### GPU Spatial Data Structures

Neighbour queries ("find all particles within radius h of particle i") are the inner loop of SPH, cloth collision, and particle interaction. On the CPU, k-d trees and BVHs handle this; on the GPU, cache-coherent alternatives are required: spatial hash maps and Linear BVHs built via Morton codes.

#### GPU Spatial Hashing
Hashes each particle's grid cell coordinate into a fixed-size table of bucket lists. Construction: one compute pass writes (cell_hash, particle_id) pairs; a sort (radix sort, §19) puts same-bucket particles contiguous; a scan writes per-bucket start offsets. Query: hash the query cell, walk the bucket list.

```glsl
// Spatial hash: 3D cell → table index
uint spatial_hash(ivec3 cell, uint table_size) {
    const uint p1 = 73856093u, p2 = 19349663u, p3 = 83492791u;
    return (uint(cell.x)*p1 ^ uint(cell.y)*p2 ^ uint(cell.z)*p3) % table_size;
}

// Query: enumerate neighbours of particle i within radius h
void find_neighbors(uint i, float h) {
    vec3  pi   = positions[i];
    ivec3 cell = ivec3(floor(pi / h));
    for (int dz = -1; dz <= 1; ++dz)
    for (int dy = -1; dy <= 1; ++dy)
    for (int dx = -1; dx <= 1; ++dx) {
        uint bucket = spatial_hash(cell + ivec3(dx,dy,dz), TABLE_SIZE);
        uint start  = bucket_start[bucket];
        uint end    = bucket_start[bucket + 1];
        for (uint k = start; k < end; ++k) {
            uint j = particle_ids[k];
            if (length(positions[j] - pi) < h)
                process_neighbor(i, j);
        }
    }
}
```

**Use cases** — SPH neighbour search, cloth self-collision, crowd simulation, particle-particle interaction.  
**Reference** — [Ihmsen et al.: A Parallel SPH Implementation on Multi-Core CPUs (CGF 2011)](https://doi.org/10.1111/j.1467-8659.2010.01832.x)

#### LBVH — Linear BVH via Morton Codes
Builds a BVH over dynamic geometry (for TLAS rebuild of deformable objects) without the CPU. Three compute passes: (1) compute 30-bit Morton code for each primitive by interleaving its centroid's quantised x, y, z bits; (2) radix sort primitives by Morton code — spatially nearby primitives become contiguous; (3) build internal BVH nodes bottom-up using the radix tree structure implicit in sorted Morton codes (Karras 2012), computing AABBs with a parallel reduction.

```glsl
// Morton code — interleave 10-bit quantised coordinates
uint expand_bits(uint v) {
    v = (v * 0x00010001u) & 0xFF0000FFu;
    v = (v * 0x00000101u) & 0x0F00F00Fu;
    v = (v * 0x00000011u) & 0xC30C30C3u;
    v = (v * 0x00000005u) & 0x49249249u;
    return v;
}
uint morton3D(float x, float y, float z) {
    x = clamp(x * 1024.0, 0.0, 1023.0);
    y = clamp(y * 1024.0, 0.0, 1023.0);
    z = clamp(z * 1024.0, 0.0, 1023.0);
    return expand_bits(uint(x)) | (expand_bits(uint(y)) << 1) | (expand_bits(uint(z)) << 2);
}
```

**Use cases** — per-frame TLAS rebuild for deformable meshes (skinned characters, cloth) without CPU involvement; dynamic ray tracing scenes.  
**Reference** — [Karras: Maximizing Parallelism in the Construction of BVHs, Octrees, and k-d Trees (HPG 2012)](https://dl.acm.org/doi/10.2312/EGGH/HPG12/033-037)

---

### Sampling Techniques

**What sampling is and why it matters.** Every rendered pixel represents a question: "what is the radiance arriving at this point on the sensor from this direction?" The exact answer requires integrating over all light paths connecting the scene to that pixel — an integral over a high-dimensional space of possible paths that has no closed form for real scenes. *Sampling* is the strategy for choosing which paths to evaluate so that their weighted average converges to the true integral as efficiently as possible.

Sampling problems appear throughout the rendering pipeline:

- **Direct lighting** — which point on an area light should we test for shadow? A hemisphere of directions must be sampled.
- **Indirect/global illumination** — what secondary direction should a ray reflect or scatter into?
- **Ray tracing denoising** — how do we distribute 1 ray/pixel so the noise pattern denoises well?
- **TAA/upscaling** — which sub-pixel offset should we jitter the camera by this frame?
- **Texture filtering** — which texel neighbours should we blend, and by how much?
- **Lightmap baking** — how many rays per texel, and in which directions?

The core trade-off is *variance* (how much the estimate fluctuates from frame to frame or pixel to pixel) versus *cost* (how many samples we can afford). Better sampling strategies reduce variance for the same sample count — they are free performance gains at the algorithm level, invisible to the rasterizer and memory subsystem.

**Key sampling vocabulary:**

| Term | Meaning |
|------|---------|
| **PDF** (probability density function) | The probability of choosing a given sample direction or point; must integrate to 1 over the domain. |
| **Unbiased** | Expected value equals the true integral regardless of sample count. |
| **Consistent** | Converges to the true integral as sample count → ∞. |
| **Importance sampling** | Draw samples proportional to the integrand so high-contribution directions are sampled more. |
| **MC estimator** | `(1/N) Σ f(xᵢ)/pdf(xᵢ)` — the unbiased estimator for ∫f(x)dx when samples are drawn from pdf. |
| **Low-discrepancy** | Samples that fill the domain more uniformly than pseudorandom; faster convergence. |
| **Blue noise** | Noise whose energy is concentrated at high spatial frequencies; errors are perceptually small. |

---

### Texture Sampling and Filtering

Texture sampling is the most frequent sampling operation on the GPU: the fixed-function texture unit interpolates between neighbouring texels using the surface's UV coordinates and the screen-space derivatives of those coordinates.

**Bilinear filtering** — interpolates the four nearest texels weighted by fractional UV position. Blurs the texture at close range; free on all hardware.

**Trilinear filtering** — blends between two adjacent mip levels in addition to bilinear, eliminating the mip-seam pop. Costs one extra bilinear sample per access.

**Anisotropic filtering** — samples along the anisotropy direction (the elongated texture-space footprint of a near-grazing surface) using multiple bilinear samples at a single mip level. Eliminates the blurring of trilinear on slanted surfaces. AF×8 or AF×16 is standard in modern games at near-zero performance cost.

```glsl
// Manual LOD bias — sharpen a texture by biasing the mip selection
layout(set=0, binding=0) uniform sampler2D albedo;
float lod_bias = -1.0; // sample one mip level finer than default
vec4  color    = texture(albedo, uv, lod_bias);

// Explicit LOD — compute level manually (e.g. in compute, where dFdx is unavailable)
float lod = textureQueryLod(albedo, uv).y; // .x = accessed level, .y = computed level
vec4  c2  = textureLod(albedo, uv, lod);
```

**EWA (Elliptical Weighted Average)** filtering uses an elliptical Gaussian footprint matched to the true screen-space anisotropy, giving higher quality than the hardware AF approximation. Implemented in software (rarely in real-time, more common in offline renderers).

**Reference** — [OpenGL 4.6 Core Profile §8.14: Texture Minification](https://registry.khronos.org/OpenGL/specs/gl/glspec46.core.pdf)

---

### Monte Carlo Hemisphere Sampling

To compute diffuse irradiance at a point, we integrate incoming radiance over the hemisphere above the surface normal — an integral of the form `∫_Ω L(ω) (N·ω) dω`. Monte Carlo estimation draws random samples from a distribution covering the hemisphere and averages the weighted result.

#### Uniform Hemisphere Sampling
Draws samples uniformly over the hemisphere; pdf = `1/(2π)`. Simple but suboptimal: samples near the horizon (where `cos θ` is small) contribute little radiance.

```glsl
// Map a 2D uniform random pair (u1, u2) ∈ [0,1)² to a hemisphere direction
vec3 sample_hemisphere_uniform(float u1, float u2) {
    float phi      = 2.0 * PI * u1;
    float cos_theta = u2;             // uniform in [0,1]
    float sin_theta = sqrt(1.0 - cos_theta * cos_theta);
    return vec3(cos(phi) * sin_theta, sin(phi) * sin_theta, cos_theta);
}
// pdf = 1.0 / (2.0 * PI)
```

#### Cosine-Weighted Hemisphere Sampling (Malley's Method)
Draws samples proportional to `cos θ`; pdf = `cos θ / π`. Eliminates the cosine weighting from the MC estimator for Lambertian diffuse — the integrand simplifies to L(ω), reducing variance.

```glsl
vec3 sample_hemisphere_cosine(float u1, float u2) {
    float phi       = 2.0 * PI * u1;
    float cos_theta = sqrt(u2);       // cosine-weighted: CDF inversion gives sqrt(u)
    float sin_theta = sqrt(1.0 - u2);
    return vec3(cos(phi) * sin_theta, sin(phi) * sin_theta, cos_theta);
}
// pdf = cos_theta / PI  (i.e. dot(sample, N) / PI)
```

**Use cases** — diffuse irradiance estimation for RTAO, lightmap baking, DDGI probe tracing. Reduces variance by ~2–3× compared to uniform sampling for Lambertian surfaces.  
**Reference** — [Shirley: Realistic Ray Tracing §12](https://www.realtimerendering.com/)

#### Stratified Sampling
Divides the sample domain into N equal strata and draws one sample per stratum, preventing clumping. Provides O(N^(d+1)/d) convergence in d dimensions versus O(√N) for random.

**Use cases** — lightmap baking where sample count per texel is small; 2D stratification with N = k² samples per pixel.

---

### Low-Discrepancy Sequences

Pseudorandom sampling produces clumps and gaps that slow convergence. Low-discrepancy (LD) sequences fill the sample domain more uniformly, giving deterministically faster convergence — called *quasi-Monte Carlo* (QMC). LD sequences are particularly effective in 1–4 dimensions (single-pixel path tracing, per-frame jitter).

**Convergence comparison**: random MC converges at O(1/√N); QMC sequences converge at O((log N)^d / N) in d dimensions — asymptotically much faster.

#### Halton Sequence
Generates d-dimensional points by interleaving radical inverses in d coprime bases. Base-2 for x, base-3 for y, base-5 for z, etc. The first few hundred samples are well-distributed; quality degrades in high dimensions (>6).

```glsl
// Halton base-b radical inverse
float halton(uint index, uint base) {
    float result = 0.0;
    float f      = 1.0;
    uint  i      = index;
    while (i > 0u) {
        f      /= float(base);
        result += f * float(i % base);
        i      /= base;
    }
    return result;
}
// Sample index: combine frame index with pixel index to avoid repetition
float jitter_x = halton(frame_index * pixel_count + pixel_index, 2u);
float jitter_y = halton(frame_index * pixel_count + pixel_index, 3u);
```

**Use cases** — TAA jitter pattern (frame index as sequence index, base 2/3); per-frame 2D jitter in path tracers; shadow soft sampling.  
**Reference** — [Keller: Quasi-Monte Carlo Methods in Computer Graphics (SIGGRAPH Course 2003)](https://www.cs.dartmouth.edu/~wjarosz/publications/dissertation/appendixA.pdf)

#### Sobol Sequence
Generates samples using bit-reversal and direction numbers, providing very low discrepancy in 2D and reasonable quality up to ~10 dimensions. Standard in production path tracers (Cycles, Arnold, Mitsuba 3).

```glsl
// Sobol 2D — dimension 0 (base 2), dimension 1 (Sobol direction numbers)
// Scrambled Sobol via Owen scrambling for decorrelated pixel estimates:
uint sobol_sample(uint index, uint dimension, uint scramble) {
    // See: Burley 2020, Practical Hash-Based Owen Scrambling
    // Implementation: bit-reverse(index) XOR with direction-number accumulation
    // [abbreviated; production implementations use precomputed direction tables]
    return sobol_unscrambled(index, dimension) ^ scramble;
}
```

**Use cases** — path tracer primary sample sequence; lightmap baking; denoiser-friendly noise patterns (Sobol's blue-noise-like spectrum after Owen scrambling).  
**Reference** — [Burley: Practical Hash-Based Owen Scrambling (JCGT 2020)](http://jcgt.org/published/0009/04/01/)

#### R2 Sequence (Roberts)
A 2D additive recurrence sequence using irrational golden-ratio-like constants: `x_{n+1} = (x_n + α) mod 1` where α = `(√5-1)/2`, `(√3-1)/2` for x and y. The simplest LD sequence: one addition per dimension, no lookup tables.

```glsl
const float R2_X = 0.7548776662466927;  // 1/φ², φ = golden ratio
const float R2_Y = 0.5698402909980532;  // 1/φ
vec2 r2_sample(uint n) {
    return fract(vec2(0.5) + float(n) * vec2(R2_X, R2_Y));
}
```

**Use cases** — TAA sub-pixel jitter (1 addition per frame), blue noise approximation, soft shadow kernel.  
**Reference** — [Roberts: The Unreasonable Effectiveness of Quasirandom Sequences (2018)](http://extremelearning.com.au/unreasonable-effectiveness-of-quasirandom-sequences/)

---

### Blue Noise Sampling

Blue noise masks contain energy only at high spatial frequencies — their Fourier spectrum is dark (zero energy) at low frequencies and bright (non-zero) at high frequencies. Errors from blue-noise-distributed samples are small, high-frequency, and perceptually invisible to the human visual system; they denoise better than white noise at equal sample count.

**Why it matters for denoising**: a denoiser's kernel suppresses low-frequency error well but struggles with low-frequency (correlated) noise. Blue noise pushes all error to high frequencies where the denoiser performs best.

#### Precomputed Blue Noise Texture
Store a 64×64 (or 128×128) tileable blue noise texture generated by the void-and-cluster algorithm. Each frame, combine pixel coordinates and frame index to index the texture, producing temporally stable blue-noise jitter without sequence generation cost.

```glsl
layout(set=0, binding=5) uniform sampler2D blue_noise;  // 64x64 R8 void-and-cluster

// Per-pixel, per-frame blue noise sample in [0,1)
float bn = texture(blue_noise, (vec2(px) + 0.5) / 64.0).r;
// Temporal decorrelation: XOR frame index into the tile address
bn = fract(bn + float(frame_index % 64) * 0.61803398875);  // golden ratio increment
```

**Use cases** — per-pixel RTAO ray direction jitter, soft shadow sampling offset, dithered transparency, per-sample TAA jitter.  
**Reference** — [Christoph Peters: Free Blue Noise Textures (2016)](http://momentsingraphics.de/BlueNoise.html); [Heitz & Belcour: A Low-Discrepancy Sampler Using a Blue Noise Mask](https://belcour.github.io/blog/slides/2019-sampling-bluenoise/index.html)

#### Temporal Blue Noise (TBN)
Animates a blue noise pattern over time such that the combined spatio-temporal distribution is also blue. Each frame, shift the lookup by a golden ratio increment: errors that are spatially blue-noise become also temporally uncorrelated, ideal for TAA denoising.

**Use cases** — RTAO, RTGI, screen-space effects where TAA accumulates the result; the de facto standard in production deferred engines post-2020.  
**Reference** — [Wolfe et al.: Temporal Blue Noise (EGSR 2022)](https://doi.org/10.1111/cgf.14616)

---

### Importance Sampling

Plain Monte Carlo draws samples uniformly and weights by the integrand value, leading to high variance when the integrand has a sharply peaked distribution (e.g., a specular BRDF lobe). *Importance sampling* instead draws samples proportionally to the integrand — or an approximation of it — so each sample contributes approximately equal weight. The estimator is still `f(x)/pdf(x)`, but with pdf ≈ f, the ratio is nearly constant and variance collapses.

**When to use**: any Monte Carlo integral where the integrand is concentrated in a small fraction of the domain — specular BRDF evaluation, area light sampling, environment map lighting.

#### GGX VNDF Importance Sampling
Samples the visible normal distribution function (VNDF) of the GGX/Trowbridge-Reitz microfacet model — the distribution of microfacet normals visible from the view direction. Produces candidate half-vectors `H` from which the reflection direction `L = reflect(-V, H)` is computed. The VNDF-weighted pdf converges dramatically faster than sampling the full NDF.

```glsl
// Heitz 2018: Sampling the GGX Distribution of Visible Normals
vec3 sample_GGX_VNDF(vec3 Ve, float alpha_x, float alpha_y, float u1, float u2) {
    // Transform to hemispherical configuration
    vec3 Vh = normalize(vec3(alpha_x * Ve.x, alpha_y * Ve.y, Ve.z));
    // Orthonormal basis around Vh
    float lensq = Vh.x * Vh.x + Vh.y * Vh.y;
    vec3 T1     = lensq > 0.0 ? vec3(-Vh.y, Vh.x, 0.0) / sqrt(lensq) : vec3(1.0, 0.0, 0.0);
    vec3 T2     = cross(Vh, T1);
    // Sample point on disk
    float r   = sqrt(u1);
    float phi = 2.0 * PI * u2;
    float t1  = r * cos(phi);
    float t2  = r * sin(phi);
    float s   = 0.5 * (1.0 + Vh.z);
    t2        = mix(sqrt(1.0 - t1 * t1), t2, s);
    // Back to ellipsoid and sphere
    vec3 Nh   = t1 * T1 + t2 * T2 + sqrt(max(0.0, 1.0 - t1*t1 - t2*t2)) * Vh;
    return normalize(vec3(alpha_x * Nh.x, alpha_y * Nh.y, max(0.0, Nh.z)));
}
// Usage: H = sample_GGX_VNDF(V, roughness², roughness², u1, u2);
//        L = reflect(-V, H);  then evaluate BRDF/pdf and accumulate
```

**Use cases** — primary specular sampling in path tracers; RT reflection ray direction selection; environment map specular convolution.  
**Reference** — [Heitz: Sampling the GGX Distribution of Visible Normals (JCGT 2018)](http://jcgt.org/published/0007/04/01/)

#### Cosine-Weighted Light Sampling
For area lights, draws a sample point on the light surface weighted by the geometric term `cos θ_L / dist²` rather than uniformly by surface area, concentrating samples toward the centre of the light as seen from the receiver.

**Use cases** — disk and sphere area light sampling in Monte Carlo renderers; correctness requires matching pdf in the MC estimator.  
**Reference** — [Shirley et al.: Monte Carlo Techniques for Direct Lighting Calculations (TOGS 1996)](https://doi.org/10.1145/226150.226155)

#### Environment Map Importance Sampling
Builds a 2D CDF pyramid over the environment map HDR texels (sum-area table or hierarchical marginal/conditional 1D CDFs). Samples draw directions proportional to the environment luminance — sunrise/sunset and bright sky regions are sampled more; dark regions less.

```glsl
// At precompute time: build 1D marginal PDF over rows (Y), 1D conditional PDF over cols (X)
// At sample time: invert both CDFs to get (u,v) → spherical direction with pdf
// pdf(θ,φ) = luminance(u,v) / total_luminance * (width * height) / (2π² sin θ)
```

**Use cases** — HDRI-lit path tracing; environment map specular in hybrid rasterization+RT pipelines; denoising quality improves dramatically with IS vs uniform env sampling.  
**Reference** — [Pharr, Jakob, Humphreys: PBRT-v4 §12.6 — Infinite Area Light Sampling](https://pbrt.org/chapters/pbrt-4ed-chapters.pdf)

---

### Multiple Importance Sampling (MIS)

A scene has two strategies for sampling a direct lighting integral: sample the BRDF (good for specular; bad for large area lights) or sample the light (good for area lights; bad for mirror-like BRDFs). Neither dominates the other in all cases. MIS combines both strategies in a single unbiased estimator:

```
L_direct ≈ (1/N) Σ [ f(x_BRDF) * L(x_BRDF) / (w_BRDF * pdf_BRDF + w_light * pdf_light_for_x_BRDF) ]
          + (1/M) Σ [ f(x_light) * L(x_light) / (w_BRDF * pdf_BRDF_for_x_light + w_light * pdf_light) ]
```

The balance heuristic weights are `w_k = pdf_k / Σ pdf_i`; the power heuristic uses `pdf_k^β / Σ pdf_i^β` with β=2, reducing variance further.

```glsl
// Power heuristic with beta=2
float mis_power(float pdf_a, float pdf_b) {
    float a2 = pdf_a * pdf_a;
    float b2 = pdf_b * pdf_b;
    return a2 / (a2 + b2);
}

// One MIS direct light estimate: N=1 BRDF sample + M=1 light sample
vec3 direct = vec3(0.0);
// BRDF sample
{
    vec3 L_s   = sample_BRDF(V, N, roughness, u1, u2);
    float p_s  = pdf_BRDF(V, N, L_s, roughness);
    float p_l  = pdf_light(P, L_s);   // chance the light sampler would have chosen L_s
    float w_s  = mis_power(p_s, p_l);
    direct    += w_s * eval_BRDF(V, L_s, N) * sample_light_radiance(P, L_s) / p_s;
}
// Light sample
{
    vec3 L_l   = sample_light(P, u3, u4);
    float p_l  = pdf_light(P, L_l);
    float p_s  = pdf_BRDF(V, N, L_l, roughness);
    float w_l  = mis_power(p_l, p_s);
    direct    += w_l * eval_BRDF(V, L_l, N) * light_radiance(P, L_l) / p_l;
}
```

**Use cases** — any scene mixing specular surfaces and area lights; standard in all production path tracers (Cycles, Arnold, V-Ray, PBRT). Reduces direct lighting variance by 5–50× over single-strategy sampling in typical scenes.  
**Reference** — [Veach & Guibas: Optimally Combining Sampling Techniques for Monte Carlo Rendering (SIGGRAPH 1995)](https://dl.acm.org/doi/10.1145/218380.218498)

---

### ReSTIR — Reservoir-Based Spatiotemporal Importance Resampling

ReSTIR is the dominant technique for real-time many-light GI, enabling sampling from thousands of virtual point lights (VPLs) or path-traced light samples at 1–4 samples/pixel with low variance. It combines reservoir sampling (Weighted Reservoir Sampling, WRS) with temporal and spatial reuse of past-frame and neighbouring-pixel reservoirs.

**Why this matters**: environment and emissive GI with thousands of lights would require hundreds of samples/pixel to converge with standard MC. ReSTIR achieves near-converged results at 1 sample/pixel by sharing reservoirs across pixels and frames, effectively borrowing sample quality from neighbours and past frames.

#### Weighted Reservoir Sampling (WRS)
Maintains a reservoir `(y, W_sum, M)`: a single selected candidate `y`, accumulated weight `W_sum`, and sample count `M`. To stream in a new candidate `x` with weight `w(x)`:

```glsl
struct Reservoir {
    uint  y;       // selected sample (e.g., light index)
    float W_sum;   // sum of weights seen so far
    float W;       // unbiased contribution weight = W_sum / (M * p_hat(y))
    uint  M;       // number of samples seen
};

void reservoir_update(inout Reservoir r, uint x, float w, float u) {
    r.W_sum += w;
    r.M     += 1u;
    if (u < w / r.W_sum)   // accept x with probability proportional to w
        r.y = x;
}
```

After streaming N candidates: the selected `y` is drawn with probability proportional to its weight. The unbiased contribution weight `W = W_sum / (M * p_hat(y))` corrects for the resampling bias when combining reservoirs from different pixels.

#### ReSTIR DI — Direct Illumination
Each pixel generates K candidate lights (K = 32 is common), builds a reservoir from them, then reuses reservoirs from temporal (same pixel, previous frame) and spatial (8 random neighbours) reservoirs. Result: effective sample count of `K × N_temporal × N_spatial` at 1 final shadow ray per pixel per frame.

**Use cases** — many-light direct illumination in deferred engines; Cyberpunk 2077 RT mode (2022); all major DXR/Vulkan RT titles post-2022.  
**Reference** — [Bitterli et al.: Spatiotemporal Reservoir Resampling for Real-Time Ray Tracing with Dynamic Direct Lighting (SIGGRAPH 2020)](https://research.nvidia.com/publication/2020-07_spatiotemporal-reservoir-resampling-real-time-ray-tracing-dynamic-direct)

#### ReSTIR GI — Global Illumination
Extends ReSTIR to indirect light paths: each reservoir stores a complete path (hit point, direction, incident radiance) rather than a light index. Spatial and temporal reservoir reuse applies the same WRS mechanism across indirect paths, enabling real-time diffuse+specular GI without a probe grid.

**Use cases** — real-time RTGI in AAA titles; replaces or supplements DDGI probe grids for dynamic geometry; high quality with 1 indirect path/pixel + denoising.  
**Reference** — [Ouyang et al.: ReSTIR GI: Path Resampling for Real-Time Path Tracing (EGSR 2021)](https://research.nvidia.com/publication/2021-06_restir-gi-path-resampling-real-time-path-tracing)

---

### Reversed-Z and Logarithmic Depth

Standard depth buffers map the near plane to `z = 0` and the far plane to `z = 1` in NDC. With a standard 24-bit depth buffer and typical near/far ratios (0.1m / 1000m = 1:10000), IEEE floating-point precision is extremely unequal: 50% of all representable depth values fall within the first 0.1% of the depth range. The result is severe Z-fighting on distant geometry.

**Shader stage**: reversed-Z requires a projection matrix change (CPU-side) and `VK_COMPARE_OP_GREATER_OR_EQUAL` in the depth compare op. Logarithmic depth requires a **vertex shader** modification. Both are orthogonal and composable.

#### Reversed-Z (Reversed Depth Buffer)
Flips the depth range: near plane maps to `z = 1.0`, far plane maps to `z = 0.0`. Because IEEE 32-bit floats have their highest density near zero, reversed-Z gives maximum precision to the most important region: the far field. Near-plane precision is largely wasted in standard depth (near objects are already close to `z = 0`); reversed-Z redirects that precision budget to where Z-fighting actually occurs.

```glsl
// Reversed-Z projection matrix (right-hand, Vulkan NDC convention)
// Standard: near → 0, far → 1. Reversed: near → 1, far → 0.
// Change the matrix on the CPU:
float fov_y_rad = ..., aspect = ..., near = 0.1;  // NO far plane needed!
float f = 1.0 / tan(fov_y_rad * 0.5);
mat4 proj_reversed_z = mat4(
    f / aspect,  0.0,  0.0,   0.0,
    0.0,          f,   0.0,   0.0,
    0.0,         0.0,  0.0,  -1.0,   // z = -near * w  (maps near → w, far → 0)
    0.0,         0.0, near,   0.0
);
// With infinite far plane: no far-plane Z-fighting at all — far clips to 0 exactly
```

**API changes required**:
- Vulkan `VkPipelineDepthStencilStateCreateInfo::depthCompareOp = VK_COMPARE_OP_GREATER_OR_EQUAL`
- Clear depth to `0.0` (was `1.0`)
- Viewport `minDepth = 0.0`, `maxDepth = 1.0` unchanged

**Use cases** — every modern game engine uses reversed-Z: UE4+, Unity HDRP, Godot 4, Bevy, id Tech 7. The infinite far plane variant eliminates far-plane clipping entirely.  
**Reference** — [Everitt & Rákos: Depth Precision Visualized (NVIDIADevBlog 2015)](https://developer.nvidia.com/blog/depth-precision-visualized/)

#### Logarithmic Depth
For extremely large scale differences (planetary rendering, flight simulators, space games: near = 0.1m, far = 6400km), even reversed-Z floats cannot provide enough precision. Logarithmic depth remaps NDC depth to a logarithmic scale, distributing precision evenly in log space.

```glsl
// Logarithmic depth — applied in vertex shader after standard transform
uniform float log_depth_C;   // typically 1.0; controls precision curve

void main() {
    vec4 clip = standard_proj_view * vec4(in_pos, 1.0);
    // Overwrite clip.z with log-depth: log(C * clip.w + 1) / log(C * far + 1) * clip.w
    gl_Position = clip;
    float Fcoef = 2.0 / log2(far_plane * log_depth_C + 1.0);
    gl_Position.z = (log2(max(1e-6, clip.w * log_depth_C + 1.0)) * Fcoef - 1.0) * clip.w;
}
```

**Fragment shader**: optionally recompute for higher precision (GPU interpolates `gl_FragCoord.z` linearly in clip space by default).

**Use cases** — Kerbal Space Program 2, Space Engine, any game with 6+ orders of magnitude depth range.  
**Reference** — [Brano Kemen: Logarithmic Depth Buffer (2009)](https://outerra.blogspot.com/2009/08/logarithmic-z-buffer.html); [Thatcher Ulrich: Continuous LOD Terrains (Game Programming Gems, 2000)]

---


---

## References

### A. Freely Readable Books and Long-Form References

| Resource | URL | Coverage |
|---|---|---|
| **Physically Based Rendering 4th ed.** (Pharr, Jakob, Humphreys, 2023) | [pbr-book.org](https://pbr-book.org/4ed/) | Monte Carlo, BSDFs, volume scattering, sampling, spectral rendering, GPU wavefront. CC BY-NC-ND |
| **GPU Gems 1** (NVIDIA, 2004) | [developer.nvidia.com/gpugems/gpugems](https://developer.nvidia.com/gpugems/gpugems/) | Natural effects, lighting, materials, image processing, vertex programs |
| **GPU Gems 2** (NVIDIA, 2005) | [developer.nvidia.com/gpugems/gpugems2](https://developer.nvidia.com/gpugems/gpugems2/) | Geometric complexity, shading, shadows, GPGPU, simulation |
| **GPU Gems 3** (NVIDIA, 2007) | [developer.nvidia.com/gpugems/gpugems3](https://developer.nvidia.com/gpugems/gpugems3/) | Geometry, GI, AO, physics simulation, GPU computing |
| **Advances in Real-Time Rendering** (SIGGRAPH course notes, 2006–2026) | [advances.realtimerendering.com](https://advances.realtimerendering.com/) | Production engine techniques; Lumen, Nubis clouds, neural GI, hair, skin |
| **Real-Time Rendering 4th ed. bibliography** (Akenine-Möller et al.) | [realtimerendering.com](https://www.realtimerendering.com/) | 4,000+ paper bibliography covering all rendering topics |

### B. Algorithm and Technique Sites

| Resource | URL | Speciality |
|---|---|---|
| **Inigo Quilez articles** | [iquilezles.org/articles](https://iquilezles.org/articles/) | SDFs, raymarching, procedural noise, shader math — 138 articles, MIT-licensed GLSL |
| **Shadertoy** | [shadertoy.com](https://www.shadertoy.com/) | Live GLSL shader sandbox; community-contributed examples for virtually every technique |
| **Christoph Peters: Moments in Graphics** | [momentsingraphics.de](https://momentsingraphics.de/) | Moment shadow maps, blue noise textures, signal processing |
| **Bart Wronski: Graphics/ML blog** | [bartwronski.com](https://bartwronski.com/) | Auto-exposure, temporal AA, noise, sampling theory |
| **Self Shadow (SIGGRAPH Shading courses)** | [blog.selfshadow.com/publications](https://blog.selfshadow.com/publications/) | Curated SIGGRAPH shading course PDFs — Disney BRDF, skin, hair, Lumen, and more |
| **LearnOpenGL** | [learnopengl.com](https://learnopengl.com/) | Tutorial-level coverage of ~40 rendering techniques with source code |

### C. Peer-Reviewed Short-Form

| Resource | URL | Notes |
|---|---|---|
| **JCGT** (Journal of Computer Graphics Techniques) | [jcgt.org](https://jcgt.org/) | Diamond open access; peer-reviewed practical techniques; 2012–present |
| **HPG** (High Performance Graphics proceedings) | [highperformancegraphics.org](https://www.highperformancegraphics.org/) | GPU algorithms, ray tracing, rendering performance |
| **EGSR** (Eurographics Symposium on Rendering) | [egsr.eu](https://egsr.eu/) | Offline rendering, light transport, material models |
| **ACM Digital Library** (SIGGRAPH) | [dl.acm.org](https://dl.acm.org/) | Complete SIGGRAPH papers; many authors also host free PDFs |

### D. Vendor Developer Resources

| Resource | URL | Focus |
|---|---|---|
| **NVIDIA Developer Blog** | [developer.nvidia.com/blog](https://developer.nvidia.com/blog/) | RTX techniques, DLSS, NRC, OptiX, CUDA |
| **AMD GPUOpen / FidelityFX** | [gpuopen.com](https://gpuopen.com/) | FSR, CAS, FidelityFX Sort, CACAO AO, open-source effects |
| **Intel Arc graphics dev** | [intel.com/content/www/us/en/developer/topic-technology/graphics](https://www.intel.com/content/www/us/en/developer/topic-technology/graphics.html) | XeSS, XeGTAO, Arc GPU developer guides |
| **ARM Mali developer hub** | [developer.arm.com/documentation](https://developer.arm.com/documentation/) | Mobile GPU optimisation, tile-based rendering, bandwidth |

### E. Open Source Reference Implementations

| Project | URL | Why notable |
|---|---|---|
| **Filament** (Google) | [github.com/google/filament](https://github.com/google/filament) | Production PBR reference: IBL, materials, exposure, TAA — fully documented |
| **Khronos glTF-IBL-Sampler** | [github.com/KhronosGroup/glTF-IBL-Sampler](https://github.com/KhronosGroup/glTF-IBL-Sampler) | Reference prefiltered environment + BRDF LUT generation |
| **Khronos glTF-Sample-Viewer** | [github.com/KhronosGroup/glTF-Sample-Viewer](https://github.com/KhronosGroup/glTF-Sample-Viewer) | WebGL PBR reference renderer; Khronos PBR Neutral tone map |
| **XeGTAO** (Intel) | [github.com/GameTechDev/XeGTAO](https://github.com/GameTechDev/XeGTAO) | Open-source optimised GTAO implementation |
| **iryoku/separable-sss** | [iryoku.com/separable-sss](https://iryoku.com/separable-sss/) | Jimenez separable SSS reference implementation |
| **wgpu examples** | [github.com/gfx-rs/wgpu/tree/trunk/examples](https://github.com/gfx-rs/wgpu/tree/trunk/examples) | Rust/WGSL implementations of standard techniques |
| **OpenSimplex2** | [github.com/KdotJPG/OpenSimplex2](https://github.com/KdotJPG/OpenSimplex2) | Patent-free simplex noise; GLSL port included |

### F. Curated Reading Lists and Aggregators

| Resource | URL | Notes |
|---|---|---|
| **Ke-Sen Huang's rendering paper list** | [kesen.realtimerendering.com](https://kesen.realtimerendering.com/) | Comprehensive per-venue list of rendering papers with free-PDF links |
| **Graphics Programming Weekly** | [jendrikillner.com/article_database](https://www.jendrikillner.com/article_database/) | Weekly newsletter archive; curated links to new shader techniques and papers |
| **Alain Galvan's blog** | [alain.xyz/blog](https://alain.xyz/blog/) | Vulkan/GPU tutorial articles with implementation |

---


## Integrations

- **Ch135 (Vulkan Ray Tracing)** — the BRDF models in §3, GI algorithms in §6, and shadow techniques in §4 describe the algorithms that ray tracing hardware accelerates; Ch135 covers the Vulkan RT API that dispatches them
- **Ch154 (GPU-Driven Rendering)** — the GPU compute primitives in §18 (scan, compaction, radix sort, Hi-Z) are the building blocks of the GPU culling pipeline described in Ch154's indirect draw and meshlet sections
- **Ch84 (bgfx / Filament / render graphs)** — Filament implements the IBL pipeline (§7) and PBR BRDFs (§3) described here; its source is the reference implementation for this chapter's §7 entries
- **Ch82 (VMA / Vulkan Helpers)** — the post-processing passes in §9 and screen-space techniques in §10 are implemented as render passes with VMA-managed framebuffer attachments
- **Ch152 (Rust GPU Ecosystem)** — WGSL implementations of noise (§1), IBL (§7), and post-processing (§9) shaders are authoured via naga and rust-gpu using the same algorithms catalogued here
- **Ch25 (GPU Compute)** — the compute primitives in §18 are Vulkan compute shader workloads; Ch25 covers dispatching and synchronising them

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*

