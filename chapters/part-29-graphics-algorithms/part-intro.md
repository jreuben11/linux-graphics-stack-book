# Part XXIX — Graphics Algorithms

Parts I–XXVIII establish the Linux graphics stack from kernel DRM internals through Wayland compositor architecture, Vulkan API layers, browser rendering engines, terminal emulators, multimedia frameworks, and display hardware. Part XXIX steps back from the stack itself to examine what runs *on* it: the algorithms and mathematical techniques that GPU programs execute once the pipeline is correctly constructed. The nine chapters here are reference catalogs, not tutorials — they collect in one place the named techniques that graphics, geometry, and compute programmers reach for repeatedly, with enough description to select the right approach and enough references to implement it correctly.

These chapters address three audiences simultaneously. **Graphics application developers** selecting a shading or geometry technique before committing to an implementation will find the trade-off analysis most useful. **Systems and driver developers** performance-budgeting a GPU workload need to know what class of algorithm is running — register pressure, memory access pattern, divergence profile — and the catalog entries surface those characteristics directly. **Browser and web platform engineers** mapping WebGPU or WebGL workloads onto Linux GPU capabilities will find the per-algorithm extension and API-support notes relevant.

The unifying theme is **GPU-resident data parallelism**: every algorithm here is either already data-parallel or is expressed in a parallel form that maps onto Vulkan compute dispatches, vertex/fragment shader invocations, or hardware fixed-function stages. Where a CPU-centric library remains the practical production path (OpenSubdiv's topology refiner, meshoptimizer's encoder, OpenCASCADE's B-rep mesher), the relevant chapter explains the GPU upload and integration pattern so the CPU pre-pass feeds a GPU-resident pipeline.

## Chapter List

**Shader Algorithm Catalog**
- [Ch 204: Shader Algorithm Catalog — Rendering Pipeline, Lighting, and Shadows](ch204-shader-algorithm-catalog.md)
- [Ch 205: Shader Algorithm Catalog — Global Illumination and Materials](ch205-shader-gi-and-materials.md)
- [Ch 206: Shader Algorithm Catalog — Ray Tracing and Procedural Content](ch206-shader-raytracing-and-procedural.md)
- [Ch 207: Shader Algorithm Catalog — Visual Effects, Post-Processing, and GPU Compute](ch207-shader-vfx-postprocess-compute.md)

**GPU Geometry Algorithms**
- [Ch 208: GPU Geometry Algorithms — Surface Representation and Mesh Processing](ch208-gpu-geometry-algorithms.md)
- [Ch 209: GPU Geometry Algorithms — Spatial Structures, Differential Geometry, and Animation](ch209-gpu-spatial-differential-animation.md)
- [Ch 210: GPU Geometry Algorithms — Physics Simulation and Volumetric Methods](ch210-gpu-physics-and-volumetric.md)
- [Ch 211: GPU Geometry Algorithms — Terrain, Ray Tracing Geometry, and Point Clouds](ch211-gpu-terrain-raytracing-pointcloud.md)
- [Ch 212: GPU Geometry Algorithms — Neural Geometry, Specialized Techniques, and GPU Primitives](ch212-gpu-neural-specialized-primitives.md)

## Chapters in This Part

### Shader Algorithm Catalog (Ch 204–207)

The Shader Algorithm Catalog spans four volumes covering 108 named techniques across 13 categories. Every entry follows the same structure: what the algorithm computes (one sentence), when to use it, named variants, costs and limitations, and a primary reference URL. The catalog is intended as a decision tool at the whiteboard stage — before a line of shader code is written — and as a vocabulary reference for reading GPU-intensive codebases and papers.

**Chapter 204 — Rendering Pipeline, Lighting, and Shadows** covers the three foundational categories: rendering architecture (GPU-driven indirect, Clustered Forward+, Deferred/G-Buffer, Visibility Buffer, VRS, Shader Execution Reordering, Foveated/Multiview, Meshlet Generation), direct and area lighting (punctual/IES, LTC area lights, capsule/tube lights, volumetric shafts, caustics), and shadows and occlusion (shadow maps, CSM, omnidirectional, PCSS, ESM, shadow atlas, SSAO, bent normal/AO baking). These three categories are prerequisites for every technique in the subsequent volumes.

**Chapter 205 — Global Illumination and Materials** covers the algorithms that determine surface radiance: global illumination (GI survey, IBL, dynamic env map, lightmapping, stochastic GPU path tracing, LPV, SSGI, world-space radiance cache, spherical harmonics, spherical Gaussians), material models and BRDF (PBR Cook-Torrance, anisotropic, clear coat/sheen, glint/sparkle, iridescence/thin-film, translucency, SSS, eye rendering, NPR), and character and organic rendering (skin/hair, fur/shell, cloth, skeletal animation/skinning).

**Chapter 206 — Ray Tracing and Procedural Content** covers non-rasterisation techniques and world building: ray tracing (shader patterns, complete path tracer, ReSTIR PT, RT denoising SVGF/NRD, ray differentials, software BVH traversal), procedural content and geometry (noise, SDF sphere marching, 3D SDF voxelization, isosurface extraction, Catmull-Clark GPU, displacement via compute, phong/adaptive tessellation, mesh shader patterns), and environment and simulation (terrain/vegetation, terrain splatting, water/ocean, fluid SPH/Eulerian, GPU particles, fog, participating media, direct volume rendering).

**Chapter 207 — Visual Effects, Post-Processing, and GPU Compute** is the final Shader Catalog volume. It covers screen-space surface detail (virtual texturing, bindless, POM, MSDF font), transparency and compositing (OIT, SSR, decals, 3D Gaussian Splatting, neural rendering primitives), post-processing and image effects (TAA, DLSS/FSR/XeSS, bloom, DoF, motion blur, colour grading, 3D LUT, auto-exposure, dithering), and GPU compute primitives (wave/subgroup intrinsics, radix sort, bitonic sort, cooperative matrices, spatial hash structures, sampling). Chapter 207 contains the full References section for the Ch 204–207 series.

### GPU Geometry Algorithms (Ch 208–212)

The GPU Geometry Algorithms series spans five volumes covering 107 algorithms across 13 categories. **Chapter 208 contains the Algorithm Selection Guide** — decision tables by problem type and runtime constraint covering all 13 geometry categories — which should be read before any of the five volumes when the right category is not yet known.

**Chapter 208 — Surface Representation and Mesh Processing** covers the Algorithm Selection Guide for all 13 geometry categories, surface representation (subdivision surfaces/OpenSubdiv/Catmull-Clark/Loop, NURBS/B-Spline, metaballs/implicit surfaces, spline tessellation, 2D vector graphics on GPU, CAD curve rendering, GPU Boolean CSG, SDF blending, Poisson surface reconstruction), and mesh processing and topology (simplification/LOD, remeshing, UV seam/atlas, voxelisation, progressive meshes, compression/streaming, UV unwrapping, GPU Delaunay, smoothing/fairing, segmentation, repair/waterproofing, constrained Delaunay).

**Chapter 209 — Spatial Structures, Differential Geometry, and Animation** covers acceleration and indexing structures (GPU BVH construction/traversal, screen-space geometry, GPU-driven virtual geometry, octree/k-d tree/hash grid, precomputed visibility, occlusion queries, sparse voxel DAG, SDF baking/3D jump flooding, virtual geometry cluster LOD), differential geometry (geodesics, spectral geometry, shape morphing/functional maps, heat method, spectral mesh processing), and animation (skeletal LBS/DQS, IK solvers, blend shapes, hair/strand geometry, secondary dynamics, mesh retargeting, soft-body FEM, cage-based deformation, Laplacian mesh editing).

**Chapter 210 — Physics Simulation and Volumetric Methods** covers GPU simulation dynamics (fluid SPH/FLIP/Eulerian, deformable FEM/projective dynamics, rigid body, crowd/multi-agent, fracture/destruction, rope/cable, GPU cloth, fluid-solid coupling, granular DEM, hydraulic erosion, fluid surface extraction, CCD, Minkowski sum/motion planning) and SDF and volumetric methods (level set evolution, swept volumes, TSDF fusion/3D reconstruction, SDF collision, isosurface extraction Marching Cubes/Dual Contouring, GPU AO, volumetric ray marching, sparse narrow-band level-set).

**Chapter 211 — Terrain, Ray Tracing Geometry, and Point Clouds** covers terrain and environment geometry (procedural geometry, nav mesh/GPU pathfinding, terrain LOD/streaming, GPU particles/ribbon, ocean/wave simulation, geospatial terrain), ray tracing and optical geometry (RT geometry pipeline TLAS/BLAS, shadow geometry, IBL/env map preprocessing, GI probe geometry, geometric optics/caustics/beam tracing, SSR, GPU radiosity, path tracing geometry, atmospheric scattering, SSS geometry, OIT, polygon/frustum clipping), and point cloud processing (3D Gaussian Splatting, point splatting, structure-from-motion/MVS, 3D feature descriptors/pose estimation, ICP registration).

**Chapter 212 — Neural Geometry, Specialized Techniques, and GPU Primitives** is the final Geometry Algorithms volume. It covers neural and learned geometry (3D Gaussian Splatting, geometric deep learning, differentiable rendering, NeRF/neural implicit surfaces), specialized and cross-domain applications (scientific visualization, decal projection, procedural texture synthesis, acoustic ray tracing, VR/XR reprojection/foveation, silhouette detection, micro-polygon displacement, SDF font rendering, molecular surface), and GPU algorithm primitives (Poisson disk/blue noise sampling, GPU convex hull, GPU radix sort). Chapter 212 also contains §108 (Library Landscape — production geometry library survey), §109 (Performance Reference), and §110 (Integrations cross-reference map for Ch208–212).

## How the Series Interrelates

The Shader Catalog (Ch 204–207) and Geometry Catalog (Ch 208–212) are complementary references operating at different stages of the GPU pipeline. The geometry chapters describe how meshes are **constructed, deformed, indexed, and queried** before rasterisation or ray-test; the shader chapters describe what the GPU **computes for each pixel or sample** once geometry is in clip space.

In practice most workloads combine both series: a character is skinned (Ch209 §39–47), its meshlets are culled and dispatched by GPU-driven rendering (Ch204 Cat I), its surface shaded with PBR (Ch205 Cat V), and ambient occlusion computed from a BVH acceleration structure (Ch211 Cat IX combined with Ch205 Cat IV). The Algorithm Selection Guide in Ch208 is the recommended entry point when the problem category is unknown.

Cross-references to the rest of the book are dense. The Shader Catalog references Ch135 (Vulkan Ray Tracing API) for ray tracing implementations, Ch154 (GPU-Driven Rendering) for indirect draw and meshlet patterns, and Ch127 (Mesh Shaders and VRS) for variable-rate shading. The Geometry Catalog references Ch135 for BVH acceleration structure construction, Ch133 (Vulkan Compute Queues) for async compute geometry passes, and Ch25 (GPU Compute) for the broader compute API landscape. Both series presuppose the Vulkan pipeline foundation established in Ch24 and the shader compilation pipeline covered in Ch152.
