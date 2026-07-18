# Chapter 204: Shader Algorithm Catalog — Recipes, References, and Use Cases

**Target audiences**: Graphics application developers selecting a technique before implementation; systems and driver developers understanding what the GPU is computing for performance budgeting; browser and web platform engineers mapping WebGPU/WGSL support onto algorithm requirements.

This chapter is a **no-code reference catalog**. Every entry follows the same structure: what the algorithm computes (one sentence), when to use it, key named variants worth knowing, limitations and costs, and a primary reference link. For implementation details, follow the reference. For theoretical background, see Ch135 (ray tracing and the rendering equation) and Ch154 (GPU-driven rendering).

---

## Table of Contents

1. [Procedural Generation and Noise](#procedural-generation-and-noise)
2. [Signed Distance Functions and Sphere Marching](#signed-distance-functions-and-sphere-marching)
3. [Physically Based Shading — BRDF Models](#physically-based-shading--brdf-models)
4. [Shadow Rendering Techniques](#shadow-rendering-techniques)
5. [Ambient Occlusion](#ambient-occlusion)
6. [Global Illumination](#global-illumination)
7. [Image-Based Lighting](#image-based-lighting)
8. [Volumetric Rendering and Participating Media](#volumetric-rendering-and-participating-media)
9. [Post-Processing and Image Effects](#post-processing-and-image-effects)
10. [Screen-Space Techniques](#screen-space-techniques)
11. [Texture Mapping and Surface Detail](#texture-mapping-and-surface-detail)
12. [Water, Ocean, and Fluid Surface](#water-ocean-and-fluid-surface)
13. [Terrain, Vegetation, and Foliage](#terrain-vegetation-and-foliage)
14. [Skin, Hair, and Character Rendering](#skin-hair-and-character-rendering)
15. [Non-Photorealistic Rendering (NPR)](#non-photorealistic-rendering-npr)
16. [Anti-Aliasing, TAA, and ML Upscaling](#anti-aliasing-taa-and-ml-upscaling)
17. [Geometry and Mesh Shader Patterns](#geometry-and-mesh-shader-patterns)
18. [GPU Compute Algorithm Primitives](#gpu-compute-algorithm-primitives)
19. [References](#references)

---

## Procedural Generation and Noise

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

## Signed Distance Functions and Sphere Marching

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

## Physically Based Shading — BRDF Models

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

## Shadow Rendering Techniques

Shadows are the primary visual cue for spatial relationships between objects. The field divides into two concerns: correctness (does a fragment lie in shadow?) and softness (how wide is the penumbra?). Shadow mapping answers the correctness question by depth-testing from the light's viewpoint; every technique that follows is an elaboration of this idea — PCF and PCSS add softness, VSM/MSM add filterability, CSM adds resolution scalability for directional lights, and Virtual Shadow Maps add on-demand paging for scenes with many local lights. Ray-traced shadows are the ground truth but require a denoiser for acceptable sample counts.

**Shader stage**: Shadow map generation is a **vertex shader** + depth-only pass (no fragment shader needed for depth-only). Shadow lookup and PCF filtering happen in the **fragment shader** of the main lighting pass. VSM/MSM blur passes run as **compute shaders** or full-screen **fragment shaders**. PCSS blockers search also runs in the **fragment shader**. Ray-traced shadows use the **ray generation** and **miss shaders**.

#### Shadow Mapping
Renders the scene depth from the light's perspective into a depth texture; in the main pass, compares each fragment's light-space depth against the stored value.

**Use cases** — the baseline shadow algorithm for all real-time rendering.  
**Key variants** — omnidirectional (cube shadow map); cascaded (CSM); perspective warping (PSM, LiSPSM).  
**Limitations** — aliasing (self-shadowing acne, Peter Panning); resolution limits maximum shadow quality; perspective aliasing requires cascades.  
**Reference** — [GPU Gems Ch11: Shadow Map Antialiasing](https://developer.nvidia.com/gpugems/gpugems/part-ii-lighting-and-shadows/chapter-11-shadow-map-antialiasing)

#### PCF (Percentage-Closer Filtering)
Filters shadow visibility (not depth values) across a neighbourhood of shadow map texels to produce soft shadow edges.

**Use cases** — the standard soft shadow technique for directional and spot lights in all real-time renderers.  
**Key variants** — fixed-width kernel; Poisson disk kernel; random rotated kernel; PCSS-driven variable width.  
**Limitations** — does not produce distance-correct penumbrae (light size is not modelled); wider kernels are expensive.  
**Reference** — [Reeves et al. (1987) SIGGRAPH](https://doi.org/10.1145/37402.37429)

#### PCSS (Percentage-Closer Soft Shadows)
Estimates the correct penumbra width at each pixel by first finding the average blocker depth (blocker search pass), then using that to drive a variable-width PCF kernel.

**Use cases** — physically plausible soft shadows that harden near the caster and soften with distance; games, film, archviz.  
**Limitations** — 2× the shadow map sample cost of PCF; blocker search requires a larger kernel than the final PCF; noisy at low sample counts.  
**Reference** — [Fernando: Percentage-Closer Soft Shadows (GDC 2005)](https://developer.download.nvidia.com/shaderlibrary/docs/shadow_PCSS.pdf)

#### VSM (Variance Shadow Maps)
Stores the mean and mean-squared depth in the shadow map, then applies Chebyshev's inequality to compute an upper bound on the shadow probability without per-sample depth tests.

**Use cases** — soft shadows that can be filtered/blurred (unlike PCF); real-time blur via separable Gaussian; hardware bilinear.  
**Limitations** — light bleeding (incorrect bright regions in shadow) when multiple occluders are at different depths; not suitable for transparent shadow casters.  
**Reference** — [Donnelly & Lauritzen (2006): VSM](https://jankautz.com/publications/vsm_gi06.pdf)

#### MSM (Moment Shadow Maps)
Extends VSM to four statistical moments of the depth distribution (using the Hamburger recurrence), reducing light bleeding while maintaining filterability.

**Use cases** — high-quality soft shadows where VSM light bleeding is unacceptable; production rendering.  
**Key variants** — 4-moment (balanced quality/cost); 6-moment (higher quality, more storage).  
**Reference** — [Hamburger et al.: Moment Shadow Maps (JCGT 2015)](https://jcgt.org/published/0004/05/01/)

#### Cascaded Shadow Maps (CSM)
Partitions the view frustum into depth slices, rendering a separate shadow map for each slice at a resolution matched to the slice's angular subtended area.

**Use cases** — directional lights (sun/moon) in outdoor scenes; the universal solution to shadow map perspective aliasing.  
**Key variants** — logarithmic split (more cascades near camera); practical split (blend of log and uniform); PSSM (Parallel-Split Shadow Maps).  
**Limitations** — cascade count × shadow map render cost; blend bands at cascade boundaries; moving sun requires per-frame re-render.  
**Reference** — [GPU Gems 3 Ch10: Parallel-Split Shadow Maps](https://developer.nvidia.com/gpugems/gpugems3/part-ii-light-and-shadows/chapter-10-parallel-split-shadow-maps-programmable-gpus)

#### Virtual Shadow Maps (VSMs)
Implements a massive-resolution clipmap (e.g. 16k×16k) backed by on-demand physical pages, rendering only shadow map pages that overlap the camera's visible geometry.

**Use cases** — Unreal Engine 5's shadow solution; eliminates cascade count choice; handles large scenes with many local lights.  
**Limitations** — requires a page table and physical page pool management; first-frame miss cost when camera moves.  
**Reference** — [Unreal Engine 5: Virtual Shadow Maps Documentation](https://docs.unrealengine.com/5.0/en-US/virtual-shadow-maps-in-unreal-engine/)

---

## Ambient Occlusion

Ambient occlusion approximates how much of the sky hemisphere is blocked by surrounding geometry at each surface point, darkening crevices and contact zones where indirect light cannot reach. It is not physically exact — true AO integrates over all light directions — but it is perceptually powerful: the human visual system strongly associates darkened concavities with spatial depth. The progression in this section runs from the cheapest screen-space approximations (SSAO, HBAO) through physically motivated formulations (GTAO, which also produces bent normals for correct IBL lookup) to the reference-quality ray-traced path (RTAO). Bent normals, produced as a byproduct of GTAO, feed back into §7 (Image-Based Lighting) to improve diffuse IBL accuracy.

**Shader stage**: All screen-space AO techniques (SSAO, HBAO, GTAO) run as full-screen **compute shaders** or full-screen **fragment shaders** reading from the depth buffer and G-buffer normals. RTAO uses the **ray generation shader** to cast hemisphere rays and the **miss shader** (no hit = unoccluded). A separate **compute shader** blur/denoise pass is always required after the raw AO signal.

#### SSAO (Screen-Space Ambient Occlusion)
Estimates local occlusion from nearby geometry by sampling the depth buffer in a hemisphere around each fragment and counting how many samples fall inside geometry.

**Use cases** — the baseline real-time AO technique; available in virtually every modern engine.  
**Key variants** — Alchemy AO / SAO (McGuire 2012): spatially-aware sample count scaling; SSAO+ (NVIDIA): higher quality sampling pattern.  
**Limitations** — only sees on-screen geometry; radius parameter causes halos; misses large-scale occlusion.  
**Reference** — [McGuire et al.: Scalable Ambient Occlusion (HPG 2012)](https://jcgt.org/published/0001/01/01/)

#### HBAO / HBAO+
Horizon-Based Ambient Occlusion samples depth along multiple directions in screen space and integrates the horizon angle to estimate the visible solid angle, producing a physically motivated AO approximation.

**Use cases** — higher-quality AO than SSAO at similar cost; used in NVIDIA GameWorks.  
**Limitations** — requires depth + normal buffers; tangent-plane approximation breaks on thin surfaces; proprietary implementation in HBAO+.  
**Reference** — [Bavoil & Sainz: Screen Space Ambient Occlusion (NVIDIA 2008)](https://developer.download.nvidia.com/SDK/10.5/direct3d/Source/ScreenSpaceAO/doc/ScreenSpaceAO.pdf)

#### GTAO (Ground Truth Ambient Occlusion)
Integrates the ambient occlusion analytically over the hemisphere using the full bent-normal and visibility function, producing significantly more accurate results than SSAO or HBAO.

**Use cases** — modern engines (Unreal 5, Unity HDRP); also produces bent normals as a byproduct for directional IBL lookup.  
**Key variants** — Jimenez (2016) original; optimised GTAO in XeGTAO (Intel, open source).  
**Reference** — [Jimenez et al.: GTAO (SIGGRAPH 2016)](https://blog.selfshadow.com/publications/s2016-shading-course/) | [XeGTAO](https://github.com/GameTechDev/XeGTAO)

#### RTAO (Ray Traced Ambient Occlusion)
Casts one or more shadow rays per pixel against the full scene geometry using hardware ray tracing, accumulates over time using temporal filtering.

**Use cases** — highest-quality AO for ray-tracing–capable hardware; reference for comparing screen-space methods.  
**Limitations** — even one ray per pixel requires a denoiser (SVGF, ReLAX) for smooth results; latency of denoising pipeline.  
**Reference** — [NVIDIA: Ray Traced Ambient Occlusion](https://developer.nvidia.com/rtx/raytracing)

#### Bent Normals
The average direction of unoccluded sky hemisphere, computed per fragment as a byproduct of AO; used to fetch the correct irradiance direction from an IBL environment map rather than the geometric normal.

**Use cases** — improves IBL diffuse quality in crevices; reduces over-darkening artefact from combining AO scalar with IBL.  
**Reference** — [Jimenez et al.: GTAO (SIGGRAPH 2016)](https://blog.selfshadow.com/publications/s2016-shading-course/)

---

## Global Illumination

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

## Image-Based Lighting

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

## Volumetric Rendering and Participating Media

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

---

## Post-Processing and Image Effects

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

## Screen-Space Techniques

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

## Texture Mapping and Surface Detail

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

## Water, Ocean, and Fluid Surface

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

## Terrain, Vegetation, and Foliage

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

---

## Skin, Hair, and Character Rendering

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

## Non-Photorealistic Rendering (NPR)

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

## Anti-Aliasing, TAA, and ML Upscaling

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

## Geometry and Mesh Shader Patterns

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

## GPU Compute Algorithm Primitives

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
