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
39. [Isosurface Extraction — Marching Cubes and Surface Nets](#isosurface-extraction--marching-cubes-and-surface-nets)
40. [RT Denoising — SVGF and NRD](#rt-denoising--svgf-and-nrd)
41. [Fluid Simulation — SPH and Eulerian Grids](#fluid-simulation--sph-and-eulerian-grids)
42. [GPU Spatial Data Structures](#gpu-spatial-data-structures)
43. [Planar Reflections](#planar-reflections)
44. [Stochastic Rendering](#stochastic-rendering)
45. [Terrain Material Splatting](#terrain-material-splatting)
46. [Neural Rendering Primitives](#neural-rendering-primitives)
47. [Omnidirectional Shadow Maps](#omnidirectional-shadow-maps)
48. [Motion Vector Generation](#motion-vector-generation)
49. [Virtual Texturing and Sparse Virtual Textures](#virtual-texturing-and-sparse-virtual-textures)
50. [Screen-Space Global Illumination (SSGI)](#screen-space-global-illumination-ssgi)
51. [Phong Tessellation and Adaptive Tessellation](#phong-tessellation-and-adaptive-tessellation)
52. [Exponential and Height Fog](#exponential-and-height-fog)
53. [Cooperative Matrix and Tensor Core Operations](#cooperative-matrix-and-tensor-core-operations)
54. [3D SDF Voxelization from Mesh](#3d-sdf-voxelization-from-mesh)
55. [Complete Path Tracer Pipeline](#complete-path-tracer-pipeline)
56. [Spherical Harmonics Lighting](#spherical-harmonics-lighting)
57. [Direct Volume Rendering](#direct-volume-rendering)
58. [Dynamic Environment Map Capture](#dynamic-environment-map-capture)
59. [Caustics](#caustics)
60. [Spherical Gaussians](#spherical-gaussians)
61. [Ordered Dithering and Output Quantization](#ordered-dithering-and-output-quantization)
62. [Meshlet Generation](#meshlet-generation)
63. [Shadow Atlas and Cached Shadows](#shadow-atlas-and-cached-shadows)
64. [Bindless Texturing and Descriptor Indexing](#bindless-texturing-and-descriptor-indexing)
65. [Normal Map Blending](#normal-map-blending)
66. [Volumetric Light Shafts and God Rays](#volumetric-light-shafts-and-god-rays)
67. [Vignette and Lens Distortion](#vignette-and-lens-distortion)
68. [Exponential Shadow Maps](#exponential-shadow-maps)
69. [Bent Normal and AO Baking](#bent-normal-and-ao-baking)
70. [Light Propagation Volumes](#light-propagation-volumes)
71. [Ray Differentials](#ray-differentials)
72. [Bloom — Dual Kawase and FFT Convolution](#bloom--dual-kawase-and-fft-convolution)
73. [Depth of Field — CoC and Scatter-as-Gather](#depth-of-field--coc-and-scatter-as-gather)
74. [Motion Blur — Per-Object Reconstruction Filter](#motion-blur--per-object-reconstruction-filter)
75. [Color Grading and 3D LUT](#color-grading-and-3d-lut)
76. [Histogram-Based Auto-Exposure](#histogram-based-auto-exposure)
77. [Subsurface Scattering](#subsurface-scattering)
78. [Capsule and Tube Area Lights](#capsule-and-tube-area-lights)
79. [Octahedral Impostor LOD](#octahedral-impostor-lod)
80. [GPU Radix Sort](#gpu-radix-sort)
81. [Sparse Texture and Tiled Resources](#sparse-texture-and-tiled-resources)
82. [Variable Rate Shading](#variable-rate-shading)
83. [Shader Execution Reordering](#shader-execution-reordering)
84. [World-Space Radiance Cache](#world-space-radiance-cache)
85. [References](#references)

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

## Isosurface Extraction — Marching Cubes and Surface Nets

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

## RT Denoising — SVGF and NRD

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

## Fluid Simulation — SPH and Eulerian Grids

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

## GPU Spatial Data Structures

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

## Planar Reflections

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

## Stochastic Rendering

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

## Terrain Material Splatting

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

## Neural Rendering Primitives

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

## Omnidirectional Shadow Maps

Point lights cast shadows in all directions — a spotlight's cone-bounded shadow map does not apply. The standard solution is a **cube shadow map**: render the scene six times from the light's position into the six faces of a cube texture, then during shading sample the cube map with the light-to-fragment vector to retrieve the stored depth and compare. A cheaper alternative is **dual paraboloid shadow mapping** (DPSM), which projects the scene onto two paraboloid surfaces (front/back hemisphere) in a single geometry pass using a mesh shader.

**Shader stage**: shadow map generation uses the standard **vertex + fragment** pipeline (or geometry shader for single-pass cube rendering). Shadow sampling runs in the **fragment shader** of the lighting pass.

#### Cube Shadow Map
Renders the scene six times with projection matrices pointing along ±X, ±Y, ±Z from the light position. Stores linear depth `length(frag_pos - light_pos)` (not projected NDC depth) to avoid distortion artefacts at cube face edges.

```glsl
// Shadow cube map generation — fragment shader (one face per render pass)
layout(location=0) in vec3 frag_world_pos;
layout(location=0) out float out_depth;

uniform vec3  light_pos;
uniform float far_plane;

void main() {
    out_depth = length(frag_world_pos - light_pos) / far_plane;  // [0, 1]
}
```

```glsl
// Shadow cube map sampling — deferred lighting fragment shader
layout(set=1, binding=0) uniform samplerCube shadow_cube;

float point_shadow(vec3 frag_pos, vec3 light_pos, float far_plane) {
    vec3  L    = frag_pos - light_pos;
    float dist = length(L) / far_plane;
    float closest = texture(shadow_cube, L).r;  // samplerCube auto-selects face
    return (dist - 0.005 > closest) ? 0.0 : 1.0;  // 0.005 = bias
}
```

**PCF for cube maps**: sample `textureCubeShadow` with a `samplerCubeShadow` for hardware PCF support, or manually sample neighbouring cube texels and average.

```glsl
layout(set=1, binding=0) uniform samplerCubeShadow shadow_cube_pcf;

float point_shadow_pcf(vec3 L, float compare_depth) {
    // Hardware PCF: returns interpolated result
    return texture(shadow_cube_pcf, vec4(L, compare_depth)).r;
}
```

**Use cases** — point lights (street lights, torches, explosions); any omni-directional shadow caster. Single-face rendering per frame (rotating through faces over multiple frames) can amortize the 6× cost for static scenes.  
**Reference** — [de Vries: LearnOpenGL — Point Shadows](https://learnopengl.com/Advanced-Lighting/Shadows/Point-Shadows)

#### Dual Paraboloid Shadow Mapping (DPSM)
Projects the scene onto two paraboloid surfaces (forward and backward hemispheres) using a non-linear vertex transform, producing two 2D shadow maps instead of six. A geometry shader or mesh shader routes each triangle to the correct hemisphere. Shadow lookup transforms the light-to-fragment vector to paraboloid UV space.

```glsl
// DPSM vertex transform — forward hemisphere (back hemisphere uses -pos.z)
layout(location=0) in vec3 in_pos;
uniform mat4 model;
uniform vec3 light_pos;
uniform float near, far;

void main() {
    vec3 P = (model * vec4(in_pos, 1.0)).xyz - light_pos;
    float L = length(P);
    P /= L;  // normalize
    // Paraboloid projection: map hemisphere to [-1,1]² disk
    float k = P.z + 1.0;
    gl_Position = vec4(P.x / k, P.y / k, (L - near) / (far - near), 1.0);
}
```

**Use cases** — point lights with tight shadow budget; mobile/VR where six render passes are too expensive.  
**Limitations** — triangles crossing the paraboloid boundary must be clipped (handled by the geometry shader); lower quality than cube maps at face boundaries.  
**Reference** — [Brabec et al.: Single Sample Soft Shadows Using Depth Maps (VMV 2002)](https://cg.cs.uni-bonn.de/publication/brabec-2002-single/); [GPU Gems Ch12: Omnidirectional Shadow Mapping](https://developer.nvidia.com/gpugems/gpugems/part-ii-lighting-and-shadows/chapter-12-omnidirectional-shadow-mapping)

---

## Motion Vector Generation

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

## Virtual Texturing and Sparse Virtual Textures

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

## Screen-Space Global Illumination (SSGI)

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

## Phong Tessellation and Adaptive Tessellation

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

## Exponential and Height Fog

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

## Cooperative Matrix and Tensor Core Operations

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

## 3D SDF Voxelization from Mesh

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

## Complete Path Tracer Pipeline

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

## Spherical Harmonics Lighting

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

## Direct Volume Rendering

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

## Dynamic Environment Map Capture

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

## Caustics

Caustics are the bright concentrated patterns of light formed when light refracts through a transparent medium (glass, water) or reflects off a curved specular surface and focuses onto a diffuse receiver. They are absent from standard rasterization pipelines (which cannot trace refracted paths) and require dedicated techniques.

**Shader stage**: photon splatting uses a **vertex shader** to project photon hit positions into screen space and a **fragment shader** to blend the photon contribution. Screen-space caustics use a **compute or fragment shader** ray-marching from the surface. RT caustics use a `rchit` shader accumulating onto a caustic buffer.

#### Photon Splatting (GPU Caustic Map)
Trace photons from the light through the refracting surface (CPU or compute), recording where each photon hits the receiver surface. Splat each photon hit as an additive point sprite or small Gaussian into a caustic irradiance map aligned to the receiver, then apply the caustic map as an additive light contribution during shading.

```glsl
// Photon splat vertex shader — one vertex per photon hit point
layout(location=0) in vec3 photon_world_pos;   // where photon hit the diffuse receiver
layout(location=1) in vec3 photon_flux;         // RGB power of this photon

layout(location=0) out vec3 v_flux;
layout(location=1) out vec2 v_offset;           // offset within point sprite

uniform mat4  proj_view;          // caustic map projection (light POV or world-aligned)
uniform float splat_radius;       // Gaussian radius in caustic map texels

void main() {
    gl_Position  = proj_view * vec4(photon_world_pos, 1.0);
    gl_PointSize = splat_radius * 2.0;
    v_flux       = photon_flux;
    v_offset     = vec2(0.0);  // filled by rasterizer for point sprite
}
```

```glsl
// Photon splat fragment shader — additive Gaussian splat
layout(location=0) in vec3 v_flux;
layout(location=0) out vec4 caustic_out;  // additive blend enabled

void main() {
    vec2  d = gl_PointCoord - 0.5;
    float r = dot(d, d) * 4.0;
    float w = exp(-r * 2.0);               // Gaussian kernel
    caustic_out = vec4(v_flux * w, 1.0);  // additive — accumulate photon contributions
}
```

**Apply caustic map in deferred lighting**: read the caustic irradiance map at the surface UV and add to the diffuse component.

#### Screen-Space Caustics (Wyman 2005)
For underwater scenes: each water surface fragment traces a refracted ray to find where it would hit the receiver. Splat the estimated flux concentration (inversely proportional to the refracted beam divergence) into a screen-space caustic buffer.

```glsl
// Screen-space caustic accumulation — fragment shader on the water surface
uniform vec3  light_dir;
uniform float ior;   // index of refraction (water ≈ 1.333)
layout(location=0) in vec3 world_pos;
layout(location=1) in vec3 world_normal;

layout(location=0) out vec4 caustic_contribution;  // additive

void main() {
    vec3 refracted = refract(-light_dir, world_normal, 1.0 / ior);
    // Trace to receiver plane (y = receiver_y)
    float t   = (receiver_y - world_pos.y) / refracted.y;
    vec3  hit = world_pos + t * refracted;

    // Estimate flux concentration from Jacobian of refraction map
    vec3 N_dx  = dFdx(world_normal);
    vec3 N_dy  = dFdy(world_normal);
    float jacobian = abs(1.0 / (1.0 + dot(N_dx, N_dx) + dot(N_dy, N_dy)));

    // Project hit to screen and write additive contribution
    vec4 hit_clip = proj_view * vec4(hit, 1.0);
    gl_FragDepth  = hit_clip.z / hit_clip.w;
    caustic_contribution = vec4(light_color * jacobian * 0.1, 1.0);
}
```

**RT Caustics**: with `VK_KHR_ray_tracing_pipeline`, trace refracted/reflected paths from light sources; accumulate hit radiance into a caustic buffer; denoise with SVGF. Correct but expensive.

**Use cases** — underwater swimming pool caustics, wine glass shadows, gemstone sparkle, aquarium walls.  
**Reference** — [Wyman: Interactive Image-Space Caustics (Eurographics 2005)](https://www.cs.uiowa.edu/~cwyman/publications/files/caustics2005.pdf); [Jensen: A Practical Guide to Global Illumination using Photon Mapping (SIGGRAPH 2002 Course)](https://graphics.ucsd.edu/~henrik/papers/book/)

---

## Spherical Gaussians

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

## Ordered Dithering and Output Quantization

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

## Meshlet Generation

The mesh shader pipeline (§32) dispatches one task shader workgroup per **meshlet** — a small cluster of ≤64 vertices and ≤126 primitives with a precomputed bounding sphere and normal cone for efficient culling. Meshlets are not a GPU runtime algorithm; they are generated offline (or at asset-load time) by a preprocessing step that partitions the mesh. This section explains the algorithm so that the mesh shader cluster data format is fully understood.

**Tool**: the reference implementation is [meshoptimizer](https://github.com/zeux/meshoptimizer) (`meshopt_buildMeshlets`). The algorithm described here matches its output format.

#### Meshlet Cluster Data Layout
Each meshlet stores:
- An index into a shared **vertex buffer** (64 entries max): `uint8_t vertices[64]` — local vertex indices into the mesh's global vertex array.
- A packed **triangle list** (126 entries max): `uint8_t indices[378]` — triplets of indices into the local `vertices[]` array (3 bytes per triangle).
- A **bounding sphere** (centre + radius) for frustum culling.
- A **normal cone** (apex, axis direction, cutoff angle) for backface cluster culling.

```c
// meshoptimizer meshlet structure (CPU-side, matches GPU SSBO layout)
struct Meshlet {
    uint32_t vertex_offset;    // offset into global meshlet_vertices[] array
    uint32_t triangle_offset;  // offset into global meshlet_triangles[] array
    uint32_t vertex_count;     // ≤ 64
    uint32_t triangle_count;   // ≤ 126

    // Bounding sphere (for frustum cull in task shader)
    float    center[3];
    float    radius;

    // Normal cone (for backface cull in task shader)
    int8_t   cone_axis[3];     // quantised normalised axis
    int8_t   cone_cutoff;      // cos(half_angle) × 127
    float    cone_apex[3];
};
```

#### Greedy Flood-Fill Partitioning
The core algorithm grows meshlet clusters by a greedy flood-fill over the mesh adjacency graph:

1. Start with an unused triangle as the seed.
2. For each candidate adjacent triangle: if adding it keeps vertex count ≤ 64 and triangle count ≤ 126, add it to the current meshlet.
3. When no adjacent triangle fits, close the meshlet, record it, start a new one.
4. Score candidates by the number of vertices they share with the current meshlet (maximising vertex cache reuse and minimising meshlet count).

```glsl
// This runs CPU-side or in a compute preprocessing shader
// Pseudocode for the greedy partitioner:
//   for each unused_triangle t:
//     if current_meshlet.try_add(t): continue
//     else: finalise(current_meshlet); current_meshlet = new Meshlet(); current_meshlet.add(t)
//   finalise(current_meshlet)
// try_add: check if all 3 vertices of t are already in vertex_map OR
//   vertex_count + new_vertices ≤ MAX_VERTICES AND triangle_count < MAX_TRIANGLES
```

#### Normal Cone Computation
For each meshlet, compute the tightest cone enclosing all triangle normals. In the task shader, if the cone's apex direction (negated) has a dot product with the view direction greater than `cos(cone_angle + 90°)`, the entire meshlet is backfacing and can be discarded without invoking the mesh shader.

```glsl
// Task shader backface cone culling — from §32, explained here
// cone_axis: average normal direction of meshlet's triangles
// cone_cutoff: cos(max deviation of any triangle normal from cone_axis)
bool meshlet_is_backface(Meshlet m, vec3 cam_pos) {
    vec3 apex      = vec3(m.cone_apex);
    vec3 axis      = vec3(m.cone_axis) / 127.0;  // dequantise
    float cutoff   = float(m.cone_cutoff) / 127.0;
    vec3  view_dir = normalize(apex - cam_pos);
    // If all normals face away from camera: dot(view, axis) > cutoff
    return dot(view_dir, axis) >= cutoff;
}
```

**Performance impact**: meshlet generation enables the task shader to cull 40–70% of clusters on typical scene geometry before any triangle rasterization occurs, as measured in Nanite and id Tech 7 presentations.  
**Reference** — [Wihlidal: GPU-Driven Rendering (GDC 2016)](https://advances.realtimerendering.com/s2016/); [meshoptimizer: zeux/meshoptimizer](https://github.com/zeux/meshoptimizer); [Kubisch: Turing Mesh Shaders (NVIDIA 2018)](https://developer.nvidia.com/blog/introduction-turing-mesh-shaders/)

---

## Shadow Atlas and Cached Shadows

Individual shadow maps for point, spot, and directional lights are described in §4 and §47. In a scene with dozens of shadow-casting lights, allocating a separate framebuffer per light is impractical. A **shadow atlas** packs all lights' shadow maps into a single large texture, with a region allocated per light proportional to its screen-space influence. **Shadow caching** avoids re-rendering shadow faces whose content (static geometry only) has not changed, dramatically reducing shadow rendering cost.

**Shader stage**: the shadow atlas sampler lookup uses standard `texture(shadow_atlas, uv)` with per-light UV transform. Atlas management and cache invalidation run on the **CPU** or in **compute** (dirty-flag evaluation, region assignment).

#### Shadow Atlas Layout
A `4096×4096` or `8192×8192` depth texture subdivided into tiles. Each light is assigned a tile whose size is proportional to the light's projected screen area (larger tiles for nearby lights, smaller for distant). The assignment is recalculated each frame as lights move or appear.

```glsl
// Per-light atlas metadata (in a SSBO)
struct ShadowAtlasEntry {
    vec4  uv_rect;       // (x, y, width, height) in [0,1] atlas UV space
    mat4  proj_view;     // light's shadow projection-view matrix
    float near, far;     // for linear depth reconstruction
    uint  is_cached;     // 1 if static-only content is still valid
};
layout(set=0, binding=0) readonly buffer AtlasSSBO { ShadowAtlasEntry entries[]; };
layout(set=0, binding=1) uniform sampler2DShadow shadow_atlas;

// Shadow lookup for light i
float atlas_shadow(uint light_idx, vec3 world_pos) {
    ShadowAtlasEntry e = entries[light_idx];
    vec4  light_clip   = e.proj_view * vec4(world_pos, 1.0);
    vec3  ndc          = light_clip.xyz / light_clip.w;
    vec2  atlas_uv     = e.uv_rect.xy + (ndc.xy * 0.5 + 0.5) * e.uv_rect.zw;
    float compare      = ndc.z;
    return texture(shadow_atlas, vec3(atlas_uv, compare));  // hardware PCF
}
```

#### Shadow Map Caching (Static Geometry)
Shadow maps containing only static geometry can be rendered once and cached indefinitely, re-rendered only when:
- A dynamic object enters the light's view frustum (invalidates the affected region).
- The light parameters (position, angle, range) change.
- Static geometry is modified (level streaming, destruction).

The cache maintains a dirty flag per atlas tile. Each frame, only dirty tiles are rendered. Static tiles are rendered in a separate, persistent shadow pass; dynamic objects are overlaid in a second pass.

```glsl
// Two-pass cached shadow rendering:
// Pass 1 (conditional): render static geometry into cached tile if dirty
//   → clear dirty flag, retain tile content across frames
// Pass 2 (always):      render dynamic objects into a temporary tile
//   → composited with the cached static tile during shadow lookup
//   result: shadow = static_shadow * dynamic_shadow

float cached_shadow_lookup(uint light_idx, vec3 world_pos) {
    float s_static  = atlas_shadow_layer(light_idx, world_pos, LAYER_STATIC);
    float s_dynamic = atlas_shadow_layer(light_idx, world_pos, LAYER_DYNAMIC);
    return min(s_static, s_dynamic);  // both must be unshadowed to be lit
}
```

#### Atlas Region Assignment Heuristics
- **Screen-space footprint**: compute the projected area of the light's influence sphere in screen pixels; assign a tile of proportional size (e.g., 512² for lights covering > 30% of screen, 128² for < 5%).
- **LRU eviction**: when the atlas is full and a new light requests a tile, evict the least-recently-used tile.
- **Stable assignment**: avoid reassigning tiles across frames (causes one-frame shadow pop); prefer keeping the same tile between frames unless the light has moved significantly.

**Use cases** — Frostbite many-light shadow atlas, id Tech 7 cached shadows, Unity HDRP shadow atlas; used whenever more than 4 lights cast shadows simultaneously.  
**Reference** — [Valient: Rendering of COD:AW (SIGGRAPH 2014)](https://advances.realtimerendering.com/s2014/); [Ubisoft: Cascaded Shadow Map Caching in For Honor (GDC 2018)](https://advances.realtimerendering.com/s2018/)

---

## Bindless Texturing and Descriptor Indexing

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

## Normal Map Blending

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

## Volumetric Light Shafts and God Rays

Volumetric light shafts — visible beams of sunlight through foliage, dust motes in a cathedral, fog illuminated by headlights — require computing how much light from a directional source reaches each point in the atmosphere before it reaches the camera. Two implementations exist at very different cost points.

**Shader stage**: screen-space god rays run as a full-screen **fragment or compute** post-process. Ray-marched volumetric shafts run as a **compute** pass producing a half-resolution volume texture that is upsampled and composited.

#### Screen-Space Radial Blur (Shaft Approximation)
Projects the sun position into screen space, then radially blurs the scene colour (or an occlusion mask) from each pixel toward the sun position. Fast and plausible for outdoor daylight; breaks for occluders not at screen centre and does not capture correct in-scattering physics.

```glsl
// Screen-space god ray — radial blur from sun screen position
layout(set=0, binding=0) uniform sampler2D scene_color;
layout(set=0, binding=1) uniform sampler2D occlusion_mask;  // 1 = sky, 0 = occluded

uniform vec2  sun_screen_pos;   // sun position in [0,1] UV
uniform float decay;            // attenuation per step (e.g. 0.97)
uniform float density;          // step scale (e.g. 0.9)
uniform float weight;           // light weight (e.g. 0.5)
uniform int   num_samples;      // typically 64–100

layout(location=0) out vec3 god_ray_out;

void main() {
    vec2  uv      = gl_FragCoord.xy / resolution;
    vec2  delta   = (uv - sun_screen_pos) * (density / float(num_samples));
    vec2  sample_uv = uv;
    float illum_decay = 1.0;
    vec3  accum   = vec3(0.0);

    for (int i = 0; i < num_samples; ++i) {
        sample_uv   -= delta;
        vec3  s      = texture(scene_color, sample_uv).rgb
                     * texture(occlusion_mask, sample_uv).r;
        accum       += s * illum_decay * weight;
        illum_decay *= decay;
    }
    god_ray_out = accum;
}
```

**Composite pass**: add the god ray buffer additively to the HDR scene before tone mapping.

#### Ray-Marched Volumetric Shafts
For each screen pixel, march a ray from the camera through the scene, sampling the light visibility at each step (shadow map lookup for the directional light) and accumulating in-scattered light using the Henyey-Greenstein phase function (§9). This is the physically correct approach used in UE4/UE5's volumetric atmosphere.

```glsl
// Volumetric shaft compute — one thread per pixel, half resolution
layout(set=0, binding=0) uniform sampler2DShadow shadow_csm;
layout(set=0, binding=1, rgba16f) uniform image2D shaft_out;

uniform mat4  light_proj_view;
uniform vec3  light_dir;
uniform vec3  light_color;
uniform float g;              // HG asymmetry (e.g. 0.7 for forward scatter)
uniform float extinction;     // σ_t (density × extinction coeff)
uniform int   num_steps;      // 16–32 for half-res

float hg_phase(float cos_theta, float g_val) {
    float g2 = g_val * g_val;
    return (1.0 - g2) / (4.0 * 3.14159 * pow(1.0 + g2 - 2.0*g_val*cos_theta, 1.5));
}

void main() {
    ivec2 px      = ivec2(gl_GlobalInvocationID.xy);
    vec3  ray_dir = reconstruct_ray_dir(px);
    float cos_theta = dot(ray_dir, -light_dir);
    float phase   = hg_phase(cos_theta, g);

    vec3  accum   = vec3(0.0);
    float transmit = 1.0;

    for (int i = 0; i < num_steps; ++i) {
        float t   = scene_depth * float(i) / float(num_steps);
        vec3  pos = cam_pos + ray_dir * t;

        // Shadow map visibility at this position
        vec4  lclip  = light_proj_view * vec4(pos, 1.0);
        vec3  lndc   = lclip.xyz / lclip.w;
        float shadow = texture(shadow_csm, vec3(lndc.xy*0.5+0.5, lndc.z));

        float step_ext = extinction * (scene_depth / float(num_steps));
        accum    += transmit * shadow * phase * light_color * step_ext;
        transmit *= exp(-step_ext);
    }
    imageStore(shaft_out, px, vec4(accum, 1.0 - transmit));
}
```

**Use cases** — outdoor sunlight shafts through trees, interior cathedral beams, vehicle headlights in fog, spot-lit stage haze.  
**Reference** — [Hoffman & Preetham: Rendering Outdoor Light Scattering in Real Time (GDC 2002)](https://www.terathink.com/lengyel/GPUGems2/GPUGems2_ch16.pdf); [Crassin: Cascaded Light Propagation Volumes (I3D 2011)](https://research.nvidia.com/publication/cascaded-light-propagation-volumes-real-time-indirect-illumination)

---

## Vignette and Lens Distortion

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

## Exponential Shadow Maps

Variance Shadow Maps (VSM, §4) filter the shadow map with a Gaussian blur and reconstruct a probability-of-being-lit using Chebyshev's inequality. A known failure mode of VSM is **light bleeding**: when a fully-lit receiver near a shadowed one receives incorrect bright light due to the variance inequality's looseness. **Exponential Shadow Maps (ESM)** avoid this by storing `exp(c · depth)` in the shadow map instead of raw depth, enabling exact multiplicative compositing without the Chebyshev approximation. **EVSM** (Exponential Variance Shadow Maps) combines ESM and VSM for both filtering quality and reduced light bleeding.

**Shader stage**: ESM generation uses the **fragment shader** during the shadow pass (writes `exp(c · depth)`). ESM lookup uses the **fragment shader** of the lighting pass (compares via division/clamping rather than depth compare).

#### ESM Generation and Lookup
Store `exp(c · z_linear)` where `c` is a positive constant (typically 40–100; larger = harder shadow edge, less light bleeding but more precision issues).

```glsl
// ESM shadow map generation — fragment shader in shadow pass
uniform float esm_c;       // exponent constant, typically 80.0

layout(location=0) out float esm_depth;

void main() {
    float z_linear = gl_FragCoord.z;   // or linearise from NDC if needed
    esm_depth = exp(esm_c * z_linear);
}
```

```glsl
// ESM shadow map lookup — lighting fragment shader
layout(set=1, binding=0) uniform sampler2D esm_map;   // pre-blurred with Gaussian

float esm_shadow(vec3 world_pos, mat4 light_proj_view, float esm_c) {
    vec4  lclip  = light_proj_view * vec4(world_pos, 1.0);
    vec3  lndc   = lclip.xyz / lclip.w;
    vec2  uv     = lndc.xy * 0.5 + 0.5;
    float z_recv = lndc.z;   // depth of receiver in light space

    float esm_occluder = texture(esm_map, uv).r;  // exp(c * z_occluder), blurred
    // Receiver is lit if exp(c * z_recv) ≤ exp(c * z_occluder)
    // Visibility = clamp(exp(c * (z_occluder - z_recv)), 0, 1)
    float visibility = clamp(esm_occluder * exp(-esm_c * z_recv), 0.0, 1.0);
    return visibility;
}
```

**Key property**: because `exp(a) · exp(b) = exp(a+b)`, ESM shadow maps can be convolved with a Gaussian using standard texture filtering hardware — no PCF needed, full hardware bilinear works correctly.

#### EVSM — Exponential Variance Shadow Maps
Stores four moments: `(exp(c·z), exp(2c·z), -exp(-c·z), exp(-2c·z))` in an RGBA16F shadow map. The positive exponent handles standard receiver depths; the negative exponent handles receivers behind the occluder (reducing light bleeding). Chebyshev's inequality is applied separately to the positive and negative exponent distributions, and the minimum is taken.

```glsl
// EVSM moment generation — fragment shader
layout(location=0) out vec4 evsm_moments;
uniform float pos_c;   // positive exponent, e.g. 40.0
uniform float neg_c;   // negative exponent, e.g. 40.0

void main() {
    float z  = gl_FragCoord.z;
    float ep = exp( pos_c * z);
    float en = exp(-neg_c * z);
    evsm_moments = vec4(ep, ep*ep, en, en*en);
}
```

**Reference** — [Lauritzen: Rendering Antialiased Shadows with Variance Shadow Maps (GDC 2008)](https://developer.nvidia.com/gpugems/gpugems3/part-ii-light-and-shadows/chapter-8-summed-area-variance-shadow-maps); [Salvi: Rendering Filtered Shadows with Exponential Shadow Maps (ShaderX6, 2008)](https://dl.acm.org/doi/10.5555/1407073.1407086)

---

## Bent Normal and AO Baking

Real-time RTAO (§5) and SSAO compute per-frame approximate AO; for static geometry the same result can be baked offline into UV-mapped textures, giving exact occlusion at zero runtime cost. **Bent normals** extend this: instead of just storing a scalar AO factor, they store the average unoccluded hemisphere direction — the direction from which the most light arrives unobstructed. The bent normal replaces the geometric normal for diffuse IBL lookups, giving correct ambient shadowing including directional bias toward open sky.

**Shader stage**: baking runs as a **compute shader** dispatched over UV-space texels of the target texture, sampling rays from the mesh surface reconstructed from the UV layout.

#### AO Baking via Hemisphere Ray Sampling
For each texel, reconstruct the world-space surface position and normal from the UV parameterisation (using the same chart-to-world transform used during lightmapping). Then trace N cosine-weighted rays and count the fraction that do not intersect the mesh within a maximum distance.

```glsl
// AO bake compute shader — one thread per UV texel
layout(local_size_x=8, local_size_y=8) in;
layout(set=0, binding=0) uniform accelerationStructureEXT tlas;
layout(set=0, binding=1, r16f) uniform image2D ao_out;
layout(set=0, binding=2, rgba16f) uniform image2D bent_normal_out;

layout(set=0, binding=3) uniform sampler2D position_atlas;   // world-space XYZ
layout(set=0, binding=4) uniform sampler2D normal_atlas;     // world-space normal
layout(set=0, binding=5) uniform sampler2D blue_noise;

uniform int   num_samples;    // typically 64–256
uniform float max_distance;   // AO ray max length (e.g. 0.5 m)

layout(location=0) rayPayloadEXT bool ray_hit;

void main() {
    ivec2 px   = ivec2(gl_GlobalInvocationID.xy);
    vec3  P    = texelFetch(position_atlas, px, 0).xyz;
    vec3  N    = normalize(texelFetch(normal_atlas, px, 0).xyz);
    if (length(P) < 1e-6) return;  // empty texel

    float ao       = 0.0;
    vec3  bent_sum = vec3(0.0);
    uint  rng      = pcg_init(uvec2(px), 0u);

    for (int i = 0; i < num_samples; ++i) {
        vec3 L = cosine_hemisphere_sample(N, pcg_float(rng), pcg_float(rng));
        ray_hit = false;
        traceRayEXT(tlas,
            gl_RayFlagsTerminateOnFirstHitEXT | gl_RayFlagsSkipClosestHitShaderEXT,
            0xFF, 0, 0, 0,
            P + N * 0.001,   // offset to avoid self-intersection
            0.0,
            L,
            max_distance,
            0);
        if (!ray_hit) {
            ao       += 1.0;
            bent_sum += L;
        }
    }
    float ao_factor   = ao / float(num_samples);
    vec3  bent_normal = length(bent_sum) > 0.0 ? normalize(bent_sum) : N;

    imageStore(ao_out,          px, vec4(ao_factor, 0.0, 0.0, 0.0));
    imageStore(bent_normal_out, px, vec4(bent_normal * 0.5 + 0.5, 1.0));
}
```

#### Using Baked Bent Normals at Runtime
Replace the geometric normal with the baked bent normal when sampling diffuse IBL or SH irradiance. The bent normal points toward the most unoccluded sky direction, giving correct shadowing for recessed surfaces (cavities, under overhangs) without any runtime ray tracing.

```glsl
// Runtime bent normal application — fragment shader
layout(set=2, binding=0) uniform sampler2D ao_map;
layout(set=2, binding=1) uniform sampler2D bent_normal_map;

vec2 lm_uv = v_lightmap_uv;
float ao          = texture(ao_map, lm_uv).r;
vec3  bent_normal = normalize(texture(bent_normal_map, lm_uv).rgb * 2.0 - 1.0);

// Use bent normal for diffuse IBL — captures directional occlusion
vec3 diffuse_irr  = sh_irradiance(bent_normal) * ao;   // §56 SH evaluation
vec3 diffuse      = albedo * diffuse_irr;
```

**Use cases** — arch-viz, static game environments, character skin pre-baked occlusion; combined with lightmapping for complete static surface lighting.  
**Reference** — [Landis: Production-Ready Global Illumination (SIGGRAPH 2002 Course)](https://graphics.pixar.com/library/ProductionReadyGI/); [UE4: Baked Bent Normals](https://docs.unrealengine.com/4.27/en-US/RenderingAndGraphics/LightingAndShadows/BentNormalMaps/)

---

## Light Propagation Volumes

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

## Ray Differentials

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

## Bloom — Dual Kawase and FFT Convolution

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

## Depth of Field — CoC and Scatter-as-Gather

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

## Motion Blur — Per-Object Reconstruction Filter

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

## Color Grading and 3D LUT

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

## Histogram-Based Auto-Exposure

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

## Subsurface Scattering

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

## Capsule and Tube Area Lights

Analytical area light shading (§30 LTC for polygonal area lights) has a cheaper analytic solution for **line/tube** light sources: fluorescent tubes, neon signs, lightsabers, streetlights viewed close-up. The key observation is that the closest point on a line segment to the reflection ray can be computed analytically, enabling a specular highlight that correctly stretches into a streak along the tube. For diffuse, the irradiance from an infinite line has a closed-form solution; for a finite segment a simple integral approximation suffices.

**Shader stage**: evaluated in the **fragment shader** per light per lit pixel, or in a deferred lighting **compute** pass.

#### Specular — Representative Point Method (Karis 2013)
Find the point on the tube line segment closest to the reflected ray direction, clamp to the segment endpoints, use it as the light direction for specular evaluation. Correct the normalisation so energy is preserved as the tube radius decreases to a point.

```glsl
// Tube area light specular — representative point (UE4 method)
vec3 tube_light_specular(vec3 P, vec3 N, vec3 V, vec3 tube_start, vec3 tube_end,
                          float tube_radius, float roughness, vec3 light_color) {
    vec3  R         = reflect(-V, N);
    vec3  L0        = tube_start - P;
    vec3  L1        = tube_end   - P;
    vec3  Ld        = L1 - L0;
    float len2      = dot(Ld, Ld);

    // Closest point on segment to reflection ray
    float t = dot(R, L0) * dot(R, Ld) - dot(L0, Ld);
    t = t / (len2 - dot(R, Ld) * dot(R, Ld) + 1e-6);
    t = clamp(t, 0.0, 1.0);

    vec3  closest   = L0 + t * Ld;
    vec3  centre_to_ray = closest - dot(closest, R) * R;
    // Move to tube surface if within radius
    vec3  L_spec    = normalize(closest + centre_to_ray *
                        clamp(tube_radius / length(centre_to_ray + vec3(1e-6)), 0.0, 1.0));

    // Effective roughness: GGX roughness blown up by tube solid angle
    float alpha     = roughness * roughness;
    float sphere_angle = clamp(tube_radius / length(closest), 0.0, 1.0);
    float alpha_tube = clamp(alpha + sphere_angle / (2.0 * clamp(dot(N, L_spec), 0.01, 1.0)), 0.0, 1.0);

    float NdotL     = max(dot(N, L_spec), 0.0);
    return GGX_specular(N, V, L_spec, alpha_tube) * NdotL * light_color;
}
```

#### Diffuse — Segment Irradiance Integral
Integrate the Lambertian irradiance from a finite line segment. The closed-form integral of `max(0, dot(N, L(t)))` over a segment of uniform radiance reduces to an arctan formula.

```glsl
// Tube area light diffuse — closed-form line integral (Picott 1992)
vec3 tube_light_diffuse(vec3 P, vec3 N, vec3 tube_start, vec3 tube_end, vec3 light_color) {
    vec3  L0   = tube_start - P;
    vec3  L1   = tube_end   - P;
    float d0   = length(L0); L0 /= d0;
    float d1   = length(L1); L1 /= d1;
    float NdotL0 = dot(N, L0);
    float NdotL1 = dot(N, L1);
    float LdotL  = dot(L0, L1);

    // Irradiance from a line segment using the solid-angle subtended formula
    vec3  Ld   = normalize(L1 - L0);
    float sin_theta0 = sqrt(max(0.0, 1.0 - LdotL * LdotL));
    float irr  = (NdotL0 + NdotL1) / (d0 + d1 + length(tube_start - tube_end))
               * max(0.0, sin_theta0);

    return max(irr, 0.0) * light_color;
}
```

**Reference** — [Karis: Real Shading in Unreal Engine 4 (SIGGRAPH 2013)](https://cdn2.unrealengine.com/Resources/files/2013SiggraphPresentationsNotes-26915738.pdf)

---

## Octahedral Impostor LOD

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

## GPU Radix Sort

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

## Sparse Texture and Tiled Resources

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

## Variable Rate Shading

Variable Rate Shading (VRS) decouples the shading rate from the pixel rate: rather than running one fragment shader invocation per pixel, the GPU can shade a 2×2, 4×2, or 4×4 tile of pixels with a single invocation and fill the whole tile with that result. VRS reduces fragment shader cost in image regions where full shading rate is wasteful — low-frequency regions, peripheral VR field of view, motion-blurred areas, or any region that will be upscaled. In Vulkan, VRS is provided by `VK_KHR_fragment_shading_rate` (core in Vulkan 1.3 on supporting hardware).

**Shader stage**: the shading rate can be set **per-draw** (uniform), **per-primitive** (vertex/geometry shader output), or **per-tile** (a shading-rate image read before rasterisation). Evaluation of the actual shading happens in the **fragment shader**, which sees the tile as a single invocation.

#### Per-Draw Uniform Rate
Set a single shading rate for an entire draw call — useful for secondary geometry (decals, particles, foliage) where full rate is unnecessary.

```c
// Vulkan: set per-draw fragment shading rate
VkFragmentShadingRateAttachmentInfoKHR sri = {
    .fragmentShadingRateAttachment       = NULL,
    .shadingRateAttachmentTexelSize      = { 1, 1 },
};
vkCmdSetFragmentShadingRateKHR(cmd,
    &(VkExtent2D){ 2, 2 },   // 2×2 pixel tile per shading invocation
    (VkFragmentShadingRateCombinerOpKHR[]){
        VK_FRAGMENT_SHADING_RATE_COMBINER_OP_KEEP_KHR,
        VK_FRAGMENT_SHADING_RATE_COMBINER_OP_KEEP_KHR,
    });
```

#### Shading Rate Image (Per-Tile)
Build a 2D image where each texel covers an N×N tile (typically 16×16 pixels) of the render target and contains the packed shading rate for that tile. Content-adaptive VRS reads luminance variance, velocity magnitude, or foveation radius to determine per-tile rates.

```glsl
// Build shading rate image — compute shader, one thread per tile
layout(set=0, binding=0) uniform sampler2D luma_buffer;   // previous-frame luminance
layout(set=0, binding=1) uniform sampler2D velocity_buf;
layout(set=0, binding=2, r8ui) uniform uimage2D shading_rate_img;

uniform float variance_threshold;  // e.g. 0.01 — below this: use 4×4
uniform float velocity_threshold;  // e.g. 0.003 — above this: use 1×1

void main() {
    ivec2 tile  = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv    = (vec2(tile) + 0.5) / vec2(imageSize(shading_rate_img));

    // Sample 2×2 region of luminance and compute variance
    float l00   = texture(luma_buffer, uv + vec2(-0.5, -0.5) / resolution).r;
    float l10   = texture(luma_buffer, uv + vec2( 0.5, -0.5) / resolution).r;
    float l01   = texture(luma_buffer, uv + vec2(-0.5,  0.5) / resolution).r;
    float l11   = texture(luma_buffer, uv + vec2( 0.5,  0.5) / resolution).r;
    float mean  = (l00 + l10 + l01 + l11) * 0.25;
    float var   = dot(vec4(l00,l10,l01,l11)-mean, vec4(l00,l10,l01,l11)-mean) * 0.25;

    float vel   = length(texture(velocity_buf, uv).rg);

    // VK_KHR_fragment_shading_rate packed rates:
    // 0 = 1×1, 1 = 1×2, 4 = 2×1, 5 = 2×2, 10 = 4×2, etc.
    uint rate;
    if (vel > velocity_threshold || var > variance_threshold) {
        rate = 0u;   // 1×1 full rate
    } else if (var > variance_threshold * 0.1) {
        rate = 5u;   // 2×2
    } else {
        rate = 10u;  // 4×2 (check hardware support)
    }
    imageStore(shading_rate_img, tile, uvec4(rate));
}
```

#### Per-Primitive Rate (VRS from Vertex Shader)
The vertex shader can output a primitive shading rate as a built-in:

```glsl
// Vertex shader — set per-primitive shading rate for this triangle
layout(location=5) out gl_PerVertex {
    vec4 gl_Position;
};
// VK_KHR_fragment_shading_rate built-in (SPIR-V PrimitiveShadingRateKHR)
layout(location=1) out int gl_PrimitiveShadingRateEXT;

void main() {
    // Dynamic objects always shaded at 1×1; distant static at 2×2
    gl_PrimitiveShadingRateEXT = is_dynamic ? 0 : 5;
}
```

**Use cases** — VR foveated rendering where 4×4 is used at the periphery (§27); TAA passes where full rate is wasted; particle/foliage secondary passes; always combined with TAA or DLSS to restore temporal detail at reduced rates.  
**Reference** — [Vulkan: VK_KHR_fragment_shading_rate](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_fragment_shading_rate.html); [Turner: Variable Rate Shading (NVIDIA DX12/Vulkan)](https://developer.nvidia.com/variable-rate-shading)

---

## Shader Execution Reordering

In ray tracing, the GPU executes shaders in the order that rays hit geometry — determined by the BVH traversal hardware, not the programmer. This causes **wavefront divergence**: rays in the same wave hit different materials, execute different shader code, and access different textures, resulting in poor SIMD utilisation and cache thrash. **Shader Execution Reordering (SER)**, introduced by NVIDIA on Ada Lovelace (RTX 4000), defers the closest-hit shader invocation until similar shaders are grouped together, improving SIMD coherence by 2–3× for complex scenes.

SER is exposed through `VK_NV_ray_tracing_invocation_reorder` (Vulkan) and GLSL `GL_NV_shader_invocation_reorder`.

**Shader stage**: SER is applied in the **closest-hit shader** (and optionally miss shader) by wrapping the shader body with `hitObjectNV` construction and a `reorderThreadNV` call before the actual shading.

#### HitObject Construction and Reorder
```glsl
#extension GL_NV_shader_invocation_reorder : require

layout(set=0, binding=0) uniform accelerationStructureEXT tlas;

// Payload
layout(location=0) rayPayloadNV vec3 payload_radiance;

void main() {
    hitObjectNV hit;

    // Trace ray without invoking any shaders — capture the hit in a hitObject
    hitObjectTraceRayNV(hit,
        tlas,
        gl_RayFlagsNoneNV,
        0xFF,           // cull mask
        0, 1, 0,        // SBT offsets
        gl_WorldRayOriginNV,
        0.001,          // tmin
        gl_WorldRayDirectionNV,
        1e27,           // tmax
        0               // payload location
    );

    // Provide a hint to the hardware scheduler: group threads by material/shader
    // The 5-bit coherence hint encodes material type (affects SIMD grouping)
    uint material_hint = hitObjectGetInstanceIdNV(hit) & 0x1Fu;
    reorderThreadNV(hit, material_hint, 5 /*hint bits*/);

    // After reordering, invoke the closest-hit shader
    hitObjectExecuteShaderNV(hit, 0);
}
```

#### Coherence Hint Design
The quality of SER depends on the coherence hint accurately predicting shader divergence. A good hint encodes: material shader index, whether the surface uses alpha masking (potentially invokes any-hit), and BRDF type. A bad hint (all-zeros) still produces valid output but provides no reordering benefit.

```glsl
// Build a coherence hint from instance data
uint build_coherence_hint(hitObjectNV hit) {
    int inst_id   = hitObjectGetInstanceIdNV(hit);
    int geom_id   = hitObjectGetGeometryIndexNV(hit);
    // Encode shader type in high bits, geometry variation in low bits
    uint shader_type = (inst_id >> 8) & 0x0Fu;  // from instance custom index
    uint geom_var    = geom_id & 0x0Fu;
    return (shader_type << 1u) | (geom_var & 0x1u);
}
```

**Performance impact**: for scenes with many material types (e.g., an urban environment with glass, metal, painted surfaces, vegetation), SER can improve closest-hit throughput by 2–3×. For scenes with a single dominant material (e.g., a terrain with one BRDF), the benefit is minimal. SER is transparent to correctness — the output is identical whether or not reordering occurs.

**Reference** — [NVIDIA: Shader Execution Reordering (GTC 2022)](https://developer.nvidia.com/blog/improve-shader-performance-and-in-game-frame-rates-with-shader-execution-reordering/); [Vulkan: VK_NV_ray_tracing_invocation_reorder](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_NV_ray_tracing_invocation_reorder.html)

---

## World-Space Radiance Cache

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
