# Chapter 204: Shader Algorithm Catalog — Recipes, References, and Use Cases

**Target audiences**: Graphics application developers selecting a technique before implementation; systems and driver developers understanding what the GPU is computing for performance budgeting; browser and web platform engineers mapping WebGPU/WGSL support onto algorithm requirements.

This chapter is a **no-code reference catalog**. Every entry follows the same structure: what the algorithm computes (one sentence), when to use it, key named variants worth knowing, limitations and costs, and a primary reference link. For implementation details, follow the reference. For theoretical background, see Ch135 (ray tracing and the rendering equation) and Ch154 (GPU-driven rendering).

### Why the catalog is weighted toward fragment and compute shaders

The distribution of sections in this catalog mirrors the actual distribution of algorithmic work across the GPU pipeline, not an editorial gap.

**Fragment shaders dominate because that is where visual variety lives.** Every "what colour is this pixel?" decision — lighting, shadowing, material evaluation, post-processing — executes at the per-pixel stage. The combinatorial space of visual effects is effectively unbounded: each new shading model, shadow technique, or post-process is another fragment shader recipe. No other stage has an equivalent explosion of algorithmic variety.

**Compute shaders dominate because they are the general escape hatch from the rasterisation pipeline.** Any algorithm that does not fit the vertex→primitive→rasterise→fragment model — physics, path tracing, neural inference, prefix sums, FFTs, image processing — is expressed as a compute dispatch. The breadth of compute algorithms matches the breadth of GPU computing itself.

**Vertex shaders have narrow algorithmic variety by design.** Their job is fundamentally fixed: transform a position from object space to clip space and pass along interpolated attributes. The entire recipe space reduces to matrix math variants (MVP, skinning, morph targets, instancing). All meaningful vertex shader patterns fit on roughly one page; fragment shaders would fill a book.

**Task and mesh shaders have few recipes because the problem domain is deliberately narrow.** Task shaders do one thing: cull meshlets and emit survivors, with optional LOD selection. Mesh shaders emit meshlet vertices and primitives, with some procedural geometry as a secondary use. The stage was introduced to solve a specific GPU-driven rendering bottleneck, not to express general algorithmic variety. These patterns are covered with full depth in Ch154 (GPU-Driven Rendering), where the architectural context justifies the detail.

**Tessellation (TCS/TES)** is similarly bounded: adaptive LOD, PN triangles, displacement mapping. Three patterns, well-understood, rarely evolving since the fixed-function era.

---

## Table of Contents

1. [Procedural Generation and Noise](#procedural-generation-and-noise)
2. [Signed Distance Functions and Sphere Marching](#signed-distance-functions-and-sphere-marching)
3. [Physically Based Shading — BRDF Models](#physically-based-shading--brdf-models)
4. [Shadow Rendering Techniques](#shadow-rendering-techniques)
5. [Ambient Occlusion](#ambient-occlusion)
6. [Global Illumination](#global-illumination)
7. [Image-Based Lighting](#image-based-lighting)
8. [Deferred Shading and G-Buffer Rendering](#deferred-shading-and-g-buffer-rendering)
9. [Volumetric Rendering and Participating Media](#volumetric-rendering-and-participating-media)
10. [Post-Processing and Image Effects](#post-processing-and-image-effects)
11. [Screen-Space Techniques](#screen-space-techniques)
12. [Texture Mapping and Surface Detail](#texture-mapping-and-surface-detail)
13. [Water, Ocean, and Fluid Surface](#water-ocean-and-fluid-surface)
14. [Terrain, Vegetation, and Foliage](#terrain-vegetation-and-foliage)
15. [Skin, Hair, and Character Rendering](#skin-hair-and-character-rendering)
16. [Non-Photorealistic Rendering (NPR)](#non-photorealistic-rendering-npr)
17. [Anti-Aliasing, TAA, and ML Upscaling](#anti-aliasing-taa-and-ml-upscaling)
18. [Geometry and Mesh Shader Patterns](#geometry-and-mesh-shader-patterns)
19. [GPU Compute Algorithm Primitives](#gpu-compute-algorithm-primitives)
20. [Order-Independent Transparency](#order-independent-transparency)
21. [Visibility Buffer and Deferred Texturing](#visibility-buffer-and-deferred-texturing)
22. [Ray Tracing Shader Patterns](#ray-tracing-shader-patterns)
23. [Skeletal Animation and Vertex Skinning](#skeletal-animation-and-vertex-skinning)
24. [GPU Particle Systems](#gpu-particle-systems)
25. [Decal Rendering](#decal-rendering)
26. [Lightmapping and Baked GI](#lightmapping-and-baked-gi)
27. [Foveated and Multiview Rendering](#foveated-and-multiview-rendering)
28. [Outline and Selection Rendering](#outline-and-selection-rendering)
29. [Sampling Techniques](#sampling-techniques)
30. [LTC Area Lights](#ltc-area-lights)
31. [Punctual Light Attenuation and IES Profiles](#punctual-light-attenuation-and-ies-profiles)
32. [GPU-Driven Rendering](#gpu-driven-rendering)
33. [Wave and Subgroup Intrinsics](#wave-and-subgroup-intrinsics)
34. [Color Science in Shaders](#color-science-in-shaders)
35. [Normal and HDR Encoding](#normal-and-hdr-encoding)
36. [Translucency and Backlit Thin-Surface Materials](#translucency-and-backlit-thin-surface-materials)
37. [Cloth and Soft-Body Simulation](#cloth-and-soft-body-simulation)
38. [Reversed-Z and Logarithmic Depth](#reversed-z-and-logarithmic-depth)
39. [References](#references)

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

## Deferred Shading and G-Buffer Rendering

Deferred shading is a rendering pipeline architecture that decouples geometry rasterization from lighting evaluation. Instead of computing the full lighting equation for every fragment as it is drawn (forward rendering), a geometry pass writes per-pixel surface attributes — the **G-Buffer** — into multiple render targets, and a subsequent lighting pass reads those attributes and evaluates all light contributions once per pixel. See §7.0 of ch84 for the architectural overview; this section contains the shader-level recipes.

**Shader stage**: The geometry pass runs in a standard **vertex + fragment shader** pair with multiple render target outputs. The lighting pass runs as a full-screen **fragment shader** (one triangle covering the screen) or a **compute shader** dispatched over the framebuffer grid. Compute is preferred when per-tile light culling is implemented (tiled deferred / clustered deferred), since compute threads naturally map to screen tiles and can share a tile-local light list via shared memory.

#### G-Buffer Write (Geometry Pass)

Writes per-pixel surface attributes into multiple render targets in a single draw call. The fragment shader encodes material properties rather than computing lighting.

```glsl
// Geometry pass fragment shader — writes G-Buffer MRTs
// Vulkan GLSL, layout locations match VkRenderPass attachment indices

layout(location = 0) out vec4 g_albedo_rough;   // RGB = albedo, A = roughness
layout(location = 1) out vec4 g_normal_metal;   // RGB = world normal (oct-encoded), A = metallic
layout(location = 2) out vec4 g_emissive;       // RGB = emissive, A = unused

layout(set = 1, binding = 0) uniform sampler2D albedo_tex;
layout(set = 1, binding = 1) uniform sampler2D normal_map;
layout(set = 1, binding = 2) uniform sampler2D arm_tex; // R=AO, G=roughness, B=metallic

layout(location = 0) in vec3 v_world_normal;
layout(location = 1) in vec3 v_world_tangent;
layout(location = 2) in vec2 v_uv;

void main() {
    vec4  albedo_smp = texture(albedo_tex, v_uv);
    vec3  arm        = texture(arm_tex, v_uv).rgb;
    vec3  ts_normal  = texture(normal_map, v_uv).rgb * 2.0 - 1.0;

    // TBN transform: tangent-space normal → world space
    vec3 T  = normalize(v_world_tangent);
    vec3 N  = normalize(v_world_normal);
    vec3 B  = cross(N, T);
    vec3 wN = normalize(mat3(T, B, N) * ts_normal);

    g_albedo_rough = vec4(albedo_smp.rgb, arm.g);      // roughness in alpha
    g_normal_metal = vec4(wN * 0.5 + 0.5, arm.b);     // pack normal to [0,1]
    g_emissive     = vec4(albedo_smp.rgb * arm.r, 0.0); // emissive driven by AO channel
}
```

**Use cases** — any renderer with >~8 dynamic lights per scene; standard architecture for Unreal Engine, Unity (URP deferred), Godot, Blender EEVEE Next.  
**Key variants** — packed G-Buffer (fit all attributes into 2 render targets using oct-encoding for normals); thin G-Buffer (albedo + packed normal only, reconstruct roughness from SSAO).  
**Reference** — [LearnOpenGL: Deferred Shading](https://learnopengl.com/Advanced-Lighting/Deferred-Shading) | [Filament deferred shading design](https://google.github.io/filament/Filament.html)

#### Deferred Lighting Pass (Full-Screen Fragment or Compute)

Reads the G-Buffer and evaluates all analytical lights, one thread per pixel. The lighting accumulation loop iterates over visible lights — either all lights globally or a tile-culled subset.

```glsl
// Deferred lighting pass — full-screen fragment shader
// Reconstruct position from depth; evaluate PBR per pixel.
// Vulkan GLSL

layout(set = 0, binding = 0) uniform sampler2D g_albedo_rough;
layout(set = 0, binding = 1) uniform sampler2D g_normal_metal;
layout(set = 0, binding = 2) uniform sampler2D g_depth;

layout(set = 0, binding = 3) uniform CameraUB {
    mat4 inv_proj;
    mat4 inv_view;
    vec3 cam_pos;
};

struct PointLight { vec3 pos; float radius; vec3 color; float intensity; };
layout(set = 0, binding = 4) readonly buffer LightBuf { PointLight lights[]; };
layout(set = 0, binding = 5) uniform LightCount { uint num_lights; };

layout(location = 0) in  vec2 v_uv;
layout(location = 0) out vec4 frag_color;

// Reconstruct world-space position from depth and inverse projection
vec3 reconstruct_position(vec2 uv, float depth) {
    vec4 clip = vec4(uv * 2.0 - 1.0, depth, 1.0);
    vec4 view = inv_proj * clip;
    view /= view.w;
    return (inv_view * vec4(view.xyz, 1.0)).xyz;
}

void main() {
    float depth  = texture(g_depth,        v_uv).r;
    vec4  alb_r  = texture(g_albedo_rough, v_uv);
    vec4  nor_m  = texture(g_normal_metal, v_uv);

    vec3 albedo    = alb_r.rgb;
    float roughness = alb_r.a;
    vec3 N          = normalize(nor_m.rgb * 2.0 - 1.0);
    float metallic  = nor_m.a;
    vec3 P          = reconstruct_position(v_uv, depth);
    vec3 V          = normalize(cam_pos - P);

    vec3 Lo = vec3(0.0);
    for (uint i = 0u; i < num_lights; ++i) {
        PointLight l   = lights[i];
        vec3  L        = normalize(l.pos - P);
        float dist     = length(l.pos - P);
        float atten    = max(0.0, 1.0 - dist / l.radius);
        vec3  radiance = l.color * l.intensity * atten * atten;
        Lo += brdf_cook_torrance(albedo, metallic, roughness, N, V, L) * radiance;
    }
    frag_color = vec4(Lo, 1.0);
}
```

**Use cases** — standard deferred renderer lighting accumulation; combined with shadow map sampling and IBL contribution added after the loop.  
**Key variants** — tiled deferred: dispatch compute with one workgroup per screen tile, load a tile-local light list into shared memory, iterate only those lights; clustered deferred: extends tiling into depth slices for correct lighting at varying depths.  
**Reference** — [Olsson & Assarsson: Tiled Shading (JCGT 2011)](https://jcgt.org/published/0001/01/03/) | [Harada et al.: A 2.5D Culling for Forward+ (SIGGRAPH Asia 2012)](https://dl.acm.org/doi/10.1145/2407746.2407780)

#### Tiled/Clustered Light Culling (Compute Prepass)

Before the lighting pass, a compute shader divides the screen into tiles (typically 16×16 pixels) and builds a per-tile list of lights whose bounding sphere overlaps the tile's frustum slice. The lighting pass reads the per-tile list, avoiding global light iteration.

**Use cases** — scenes with >32 analytical lights; required for Forward+ (applies the same tile list to a forward renderer) and tiled deferred.  
**Key variants** — clustered: extends tiles into *z* slices (froxels) for correct behaviour with wide depth ranges (outdoor scenes).  
**Limitations** — the prepass itself costs ~0.1–0.3 ms; breakeven versus full-loop deferred is around 50 lights depending on scene geometry complexity.  
**Reference** — [Dufresne: Adventures in Clustered Shading (GDC 2018)](https://advances.realtimerendering.com/s2018/) | [Filament clustered light implementation](https://github.com/google/filament/blob/main/shaders/src/light_indirect.fs)

#### Subpass-Merged G-Buffer on TBDR (Mobile)

On TBDR mobile GPUs (Adreno, Mali, PowerVR) the geometry pass and lighting pass can be expressed as Vulkan subpasses within a single render pass. The driver merges them into one tile pass: the G-Buffer attachments live in on-chip tile SRAM throughout, never written to DRAM. The lighting subpass reads G-Buffer data via `inputAttachment` samplers (or `VK_EXT_shader_tile_image` framebuffer fetch), which the hardware routes from tile SRAM rather than issuing DRAM reads.

**Use cases** — mobile deferred renderers (Unity mobile deferred, Unreal Mobile deferred); the only way to make deferred rendering bandwidth-competitive with forward on TBDR hardware.  
**Limitations** — subpass input attachments can only read the current fragment's texel (no neighbouring pixels); screen-space effects that need neighbour access (SSAO, SSR) must still break the tile pass.  
**Reference** — [ARM Mali GPU Best Practices — Deferred shading](https://developer.arm.com/documentation/101897/latest/) | ch86 §10 and §10b of this book

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

#### Octahedral Impostor
Bakes N×M orthographic views of a 3D object distributed over an octahedral sphere into an atlas texture. At runtime, selects the two nearest baked views based on the view direction and blends between them, producing a convincing representation of the 3D object at a fraction of its polygon cost.

**Use cases** — distant trees, rocks, and complex props where full LOD geometry is too expensive; the technique behind SpeedTree and UE5 Nanite foliage impostors.  
**Key variants** — alpha-clipped 8-frame impostors (cheap, coarse); full octahedral 64-frame impostors (smooth, higher memory); animated impostors (multiple frames baked per octant for e.g. water).  
**Reference** — [Shaderbits: Octahedral Impostors (2018)](https://shaderbits.com/blog/octahedral-impostors)

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

## Order-Independent Transparency

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

## Visibility Buffer and Deferred Texturing

The G-Buffer's geometry pass writes 4–8 full-resolution render targets. The visibility buffer replaces this with a single 64-bit render target: high bits = draw call / material ID, low bits = primitive ID. A subsequent compute pass reconstructs every material attribute on demand by looking up the mesh and sampling textures. Unreal Engine 5's Nanite uses this approach.

**Shader stage**: The triangle ID pass uses a standard **vertex + fragment** pair with a single integer MRT. The material evaluation pass uses a **compute shader** dispatched over the framebuffer.

#### Triangle ID Pass
Writes a packed `uvec2` per pixel: `x` = draw ID (encodes material and mesh), `y` = primitive ID within that draw. No texture sampling, no interpolated attributes beyond position.

```glsl
// Visibility buffer write — fragment shader
layout(location = 0) out uvec2 vis_buf;  // single R32G32_UINT render target

layout(location = 0) flat in uint v_draw_id;

void main() {
    vis_buf = uvec2(v_draw_id, gl_PrimitiveID);
}
```

**Use cases** — first pass of a visibility buffer renderer; ~4–8× lower geometry pass bandwidth than a full G-Buffer; enables heterogeneous material sets without a fixed G-Buffer schema.  
**Reference** — [Burns & Hunt: The Visibility Buffer: A Cache-Friendly Approach to Deferred Shading (JCGT 2013)](http://jcgt.org/published/0002/02/04/)

#### Barycentric Attribute Reconstruction
Given a primitive ID, fetches the three vertex positions from the index buffer and vertex buffer, computes screen-space partial derivatives, and derives correct perspective-correct barycentric coordinates — enabling any per-vertex attribute (UV, normal, colour) to be reconstructed in compute.

**Use cases** — material evaluation pass in a visibility buffer renderer; correct UV derivatives (`ddx`/`ddy` equivalents) from compute without rasterizer interpolation.  
**Reference** — [Wihlidal: Optimising the Graphics Pipeline with Compute (GDC 2016)](https://frostbite-wp-prd.s3.amazonaws.com/wp-content/uploads/2016/03/29204330/GDC_2016_Compute.pdf)

#### Deferred Texturing Compute Pass
One compute thread per pixel reads the visibility buffer, looks up the draw and primitive records from GPU-side mesh and material data, reconstructs UVs via barycentric interpolation, samples material textures, and writes the fully shaded result — or writes a surrogate G-Buffer for a standard lighting pass.

**Use cases** — complete visibility buffer pipeline; enables arbitrary material diversity at constant geometry pass cost; required for Nanite-style micro-polygon rendering.  
**Reference** — [UE5 Nanite: A Deep Dive (SIGGRAPH 2021)](https://advances.realtimerendering.com/s2021/Karis_Nanite_SIGGRAPH_Advances_2021_final.pdf)

---

## Ray Tracing Shader Patterns

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

## Skeletal Animation and Vertex Skinning

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

## GPU Particle Systems

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

## Decal Rendering

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

## Lightmapping and Baked GI

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

## Foveated and Multiview Rendering

VR/XR rendering must produce two views per frame at high resolution and high frame rate. Two Vulkan techniques reduce this cost: multiview renders geometry once for both eyes simultaneously; variable-rate shading coarsens shading in the peripheral visual field where the eye has low resolution.

**Shader stage**: Multiview uses a standard **vertex shader** that reads `gl_ViewIndex` to select the correct transform. FSR writes a shading rate image from **compute** before the main render pass. ATW reprojection runs as a **compute** or **fragment** shader.

#### VK_KHR_multiview
Renders a single draw call into a 2D array render target with two layers. The vertex shader uses `gl_ViewIndex` to index into an array of view matrices, writing `gl_Position` for both eyes from one geometry submission.

```glsl
#extension GL_EXT_multiview : require
layout(set=0, binding=0) uniform ViewUB { mat4 view_proj[2]; };

layout(location=0) in vec3 in_pos;

void main() {
    gl_Position = view_proj[gl_ViewIndex] * vec4(in_pos, 1.0);
}
```

**Use cases** — stereo VR rendering; reduces geometry processing and draw call overhead by up to 50%; mandatory for high-performance VR on mobile XR hardware (Quest).  
**Reference** — [Khronos: VK_KHR_multiview specification](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_multiview.html)

#### Variable Rate Shading — Fixed Foveated (VK_KHR_fragment_shading_rate)
Writes a shading rate image (one tile per N×N pixels) before the main render pass. Tiles in the peripheral screen region are marked 2×2 or 4×4 (one shader invocation per 4 or 16 pixels); the central fovea region is marked 1×1 (full rate). The rasterizer applies the coarser rate automatically during fragment dispatch.

**Use cases** — fixed foveated rendering on Quest, PSVR2, and PC VR headsets; 30–50% fragment shader cost reduction with minimal perceptual quality loss.  
**Reference** — [Qualcomm: Fixed Foveated Rendering Best Practices](https://developer.qualcomm.com/software/adreno-gpu-sdk/tools)

#### Eye-Tracked Foveated Rendering
Updates the shading rate image every frame from eye tracker coordinates: full-rate 1×1 region is centred on the gaze point rather than the screen centre. Requires per-frame compute dispatch to write the rate image before the main render pass begins.

**Use cases** — high-end VR headsets with eye tracking (Apple Vision Pro, Varjo, Pico); 40–60% fragment shader reduction; requires latency-hiding to avoid smearing artefacts.  
**Reference** — [Foveated Rendering Overview — Khronos Blog](https://www.khronos.org/blog/vulkan-variable-rate-shading)

#### Asynchronous TimeWarp / SpaceWarp (ATW/ASW)
Reprojects the last rendered frame to the current head pose using depth and motion vectors when a new frame is not ready in time, synthesising a valid display frame with minimal latency. The reprojection runs as a high-priority compute or graphics pass on a separate GPU queue.

**Use cases** — VR compositor reprojection fallback; prevents judder when the application misses its frame deadline; implemented in Oculus, OpenVR, and Monado runtimes.  
**Reference** — [Oculus: Asynchronous TimeWarp](https://developer.oculus.com/documentation/native/android/mobile-timewarp-overview/)

---

## Outline and Selection Rendering

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

## Sampling Techniques

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

## LTC Area Lights

Area lights — spheres, disks, rectangles, tubes — emit radiance from a surface rather than a point. Evaluating their contribution analytically (without Monte Carlo sampling) requires integrating the BRDF over the solid angle subtended by the light, which has no closed form for rough GGX specular.

**Linearly Transformed Cosines** (Heitz & de Dreu 2016) solve this by finding a 3×3 linear transformation `M` that maps the clamped cosine distribution into the GGX BRDF lobe for the current roughness and view angle. The area light integral in the transformed space reduces to the integral of a clamped cosine over a polygon — which has an exact analytic formula. `M` and its normalisation are stored in two RGBA16F lookup textures indexed by `(NdotV, roughness)` and fetched once per fragment.

**Shader stage**: evaluated in the **fragment shader** (deferred lighting pass or forward shading) after the BRDF LUT lookup. No ray tracing required.

```glsl
// LTC area light evaluation — rectangle light example
// Heitz et al. 2016: Real-Time Polygonal-Light Shading with LTC
// Two precomputed textures: ltc_mat (inverse transform matrix) and ltc_amp (scale factor)
layout(set=1, binding=0) uniform sampler2D ltc_mat;   // RGBA → columns of M⁻¹
layout(set=1, binding=1) uniform sampler2D ltc_amp;   // RG  → amplitude, fresnel scale

const float LUT_SIZE  = 64.0;
const float LUT_SCALE = (LUT_SIZE - 1.0) / LUT_SIZE;
const float LUT_BIAS  = 0.5 / LUT_SIZE;

// Look up LTC matrix for this (roughness, NdotV)
vec2 uv      = vec2(roughness, sqrt(1.0 - NdotV));
uv           = uv * LUT_SCALE + LUT_BIAS;
vec4 ltc     = texture(ltc_mat, uv);
mat3 Minv    = mat3(
    vec3(ltc.x, 0.0, ltc.y),
    vec3(  0.0, 1.0,   0.0),
    vec3(ltc.z, 0.0, ltc.w)
);

// Transform rectangle light corners into LTC space and integrate
vec3 L[4];
L[0] = normalize(Minv * (rect[0] - P));
L[1] = normalize(Minv * (rect[1] - P));
L[2] = normalize(Minv * (rect[2] - P));
L[3] = normalize(Minv * (rect[3] - P));

// Polygon irradiance — exact analytic integral of clamped cosine over polygon
float irr = 0.0;
for (int i = 0; i < 4; ++i) {
    vec3 a = L[i], b = L[(i+1)%4];
    irr += acos(clamp(dot(a, b), -1.0, 1.0)) * normalize(cross(a, b)).z;
}
irr = abs(irr) / (2.0 * PI);

// Scale by BRDF amplitude and Fresnel term from second LUT
vec2 amp     = texture(ltc_amp, uv).rg;
vec3 specular = light_color * irr * (amp.x + (1.0 - F0) * amp.y);
```

**Diffuse term**: use the same polygon irradiance formula with `Minv = identity` (clamped cosine is already the correct diffuse kernel).

**Disk and sphere lights**: approximate the shape by a quad or use the sphere solid-angle formula; the LTC matrix encodes roughness response regardless of light shape.

**Use cases** — rectangle lights (windows, monitors, LEDs), disk lights (ceiling panels, headlights), tube lights (fluorescent strips); standard in Frostbite, Filament, Three.js, and Unity HDRP.  
**Reference** — [Heitz et al.: Real-Time Polygonal-Light Shading with Linearly Transformed Cosines (SIGGRAPH 2016)](https://eheitzresearch.wordpress.com/415-2/); [Eric Heitz's LTC demo and LUT generator](https://github.com/selfshadow/ltc_code)

---

## Punctual Light Attenuation and IES Profiles

Punctual lights (point, spot, directional) are the simplest lighting primitives, but physically correct attenuation and realistic beam profiles require a few non-obvious details.

**Shader stage**: evaluated in the **fragment shader** (deferred lighting pass or forward pass) once per light per fragment.

#### Inverse-Square Attenuation
Physical light intensity falls off as `1/distance²`. A bare `1/d²` diverges at zero distance; the standard fix is a small epsilon bias or a windowed falloff that smoothly goes to zero at the light's influence radius.

```glsl
// UE4 smooth windowed inverse-square attenuation
float point_attenuation(float dist, float inv_radius) {
    float d_norm = dist * inv_radius;
    float falloff = 1.0 / (dist * dist + 1.0);           // avoid singularity at d=0
    float window  = pow(max(0.0, 1.0 - d_norm * d_norm * d_norm * d_norm), 2.0);
    return falloff * window;
}
```

**Reference** — [Karis: Real Shading in Unreal Engine 4 (SIGGRAPH 2013)](https://cdn2.unrealengine.com/Resources/files/2013SiggraphPresentationsNotes-26915738.pdf)

#### Spot Light Angular Attenuation
A spotlight's cone is characterised by inner and outer half-angles. Attenuation falls from 1 inside the inner cone to 0 at the outer cone, typically with a smooth Hermite curve.

```glsl
float spot_attenuation(vec3 L, vec3 spot_dir, float cos_inner, float cos_outer) {
    float cos_angle = dot(-L, spot_dir);
    float t = clamp((cos_angle - cos_outer) / (cos_inner - cos_outer), 0.0, 1.0);
    return t * t;   // smoothstep-equivalent; use t*t*(3-2t) for C1 continuity
}
```

#### IES Photometric Profiles
Real luminaires (streetlights, theatre spots, LEDs) have complex asymmetric emission distributions measured and distributed as IES files. The 2D distribution (vertical angle, horizontal angle → candela) is baked into a 2D `LUMINANCE` texture at content-load time and sampled in the fragment shader using the light-to-fragment direction.

```glsl
layout(set=2, binding=0) uniform sampler2D ies_profile;  // (θ, φ) → luminous intensity

// direction L = normalize(fragment_pos - light_pos), in light local space
float theta = acos(clamp(-L.z, -1.0, 1.0));          // vertical: 0=up, π=down
float phi   = atan(L.y, L.x) / (2.0 * PI) + 0.5;    // horizontal: [0,1)
float ies_scale = texture(ies_profile, vec2(theta / PI, phi)).r;
// Multiply point_attenuation() by ies_scale before BRDF integration
```

**Use cases** — architectural visualisation, cinematic lighting, any scene using real fixture data; standard in Unreal, Blender Cycles, Filament.  
**Reference** — [IES LM-63 Standard](https://www.iesna.org/); [Lagarde: Moving Frostbite to PBR (SIGGRAPH 2014)](https://seblagarde.wordpress.com/2015/07/14/siggraph-2014-moving-frostbite-to-physically-based-rendering/)

---

## GPU-Driven Rendering

Traditional rendering submits one draw call per mesh per frame from the CPU, which bottlenecks at CPU-GPU command throughput and CPU-side visibility testing. **GPU-driven rendering** moves all per-draw decisions (culling, LOD selection, draw count) into compute shaders that write `VkDrawIndexedIndirectCommand` structs into a buffer, then submits a single `vkCmdDrawIndexedIndirectCount` that executes only the surviving draws.

**Why this matters**: at 10,000 draw calls/frame, CPU submission overhead is significant. GPU-driven rendering makes draw call count a GPU-compute cost (cheap) rather than a CPU cost (expensive), enabling scenes with 100k+ draw calls at constant CPU overhead.

**Shader stage**: culling and LOD selection run in a **compute shader**. Drawing uses standard **vertex + fragment** shaders reading from per-draw SSBO arrays instead of per-draw uniform updates.

#### Per-Instance Data SSBO
Replace per-draw uniform buffers with a single large SSBO of per-instance data (transform, material index, bounds). All instances are always resident on the GPU; the cull pass selects which ones draw.

```glsl
struct InstanceData {
    mat4  model;
    uint  material_id;
    uint  mesh_id;
    vec3  aabb_min;
    float _pad0;
    vec3  aabb_max;
    float _pad1;
};
layout(set=0, binding=0) readonly buffer InstanceSSBO { InstanceData instances[]; };
layout(set=0, binding=1) writeonly buffer DrawCmdSSBO { VkDrawIndexedIndirectCommand cmds[]; };
layout(set=0, binding=2) buffer DrawCount { uint draw_count; };
```

#### GPU Frustum Culling Compute
One thread per instance tests the instance AABB against the six frustum planes (in view space). Surviving instances atomically increment the draw count and write their draw command.

```glsl
layout(local_size_x=64) in;
void main() {
    uint id = gl_GlobalInvocationID.x;
    if (id >= instance_count) return;

    InstanceData inst = instances[id];
    // Test all 8 AABB corners against 6 frustum planes
    bool visible = frustum_cull(inst.aabb_min, inst.aabb_max, frustum_planes);
    if (!visible) return;

    uint cmd_idx = atomicAdd(draw_count, 1u);
    cmds[cmd_idx].indexCount    = mesh_index_count[inst.mesh_id];
    cmds[cmd_idx].instanceCount = 1u;
    cmds[cmd_idx].firstIndex    = mesh_first_index[inst.mesh_id];
    cmds[cmd_idx].vertexOffset  = mesh_vertex_offset[inst.mesh_id];
    cmds[cmd_idx].firstInstance = id;   // instance ID → indexes into InstanceSSBO
}
```

#### Two-Pass Hi-Z Occlusion Culling
A more powerful variant uses the previous frame's Hi-Z (hierarchical Z) mip pyramid to also cull occluded instances. Pass 1 culls against the previous frame's Hi-Z (fast, approximate). Pass 2 re-renders occluders to update Hi-Z, then a second cull pass tests instances that failed pass 1 against the updated Hi-Z.

```
Pass 1: cull with prev-frame Hi-Z → indirect draw surviving instances
Build Hi-Z mip from current depth buffer (compute)
Pass 2: test previously-rejected instances against new Hi-Z → draw late survivors
```

**Use cases** — standard in UE5 Nanite front-end, Frostbite, id Tech 7. Reduces GPU vertex work by 30–80% in dense urban scenes.  
**Reference** — [Wihlidal: Optimising the Graphics Pipeline with Compute (GDC 2016)](https://frostbite-wp-prd.s3.amazonaws.com/wp-content/uploads/2016/03/29204330/GDC_2016_Compute.pdf); [Aaltonen: GPU-Driven Rendering Pipelines (SIGGRAPH 2015)](https://advances.realtimerendering.com/s2015/)

#### Meshlet / Mesh Shader Cluster Pipeline
With `VK_EXT_mesh_shader`, the vertex-processing stage is replaced by a **task shader** (one threadgroup per meshlet cluster, does frustum and cone culling, emits surviving meshlets) and a **mesh shader** (one threadgroup per meshlet, reads ~64 vertices and ~126 triangles, writes `gl_MeshPrimitivesEXT`). No index buffer rasterization; all vertex amplification and culling happen in the shader.

```glsl
// Task shader — one workgroup per meshlet cluster
layout(local_size_x=32) in;
taskPayloadSharedEXT MeshPayload payload;

void main() {
    uint meshlet_id = gl_WorkGroupID.x * 32 + gl_LocalInvocationID.x;
    bool visible = cone_cull(meshlets[meshlet_id]) && frustum_cull(meshlets[meshlet_id]);
    uvec4 vote   = subgroupBallot(visible);
    uint count   = subgroupBallotBitCount(vote);
    uint idx     = subgroupBallotExclusiveBitCount(vote);
    if (visible) payload.meshlet_ids[idx] = meshlet_id;
    if (gl_LocalInvocationID.x == 0) EmitMeshTasksEXT(count, 1, 1);
}
```

**Use cases** — Nanite meshlet rasterization, GPU-driven LOD without CPU involvement, efficient backface cluster culling.  
**Reference** — [Kubisch: Turing Mesh Shaders (NVIDIA 2018)](https://developer.nvidia.com/blog/introduction-turing-mesh-shaders/); [Vulkan Mesh Shader Spec](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_mesh_shader.html)

---

## Wave and Subgroup Intrinsics

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

## Color Science in Shaders

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

## Normal and HDR Encoding

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

## Translucency and Backlit Thin-Surface Materials

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

## Cloth and Soft-Body Simulation

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

## Reversed-Z and Logarithmic Depth

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
