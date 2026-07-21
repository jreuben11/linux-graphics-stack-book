# Chapter 208: GPU Geometry Algorithms — Subdivision, Implicit Surfaces, Skinning, and IK

**Audiences:** Graphics application developers implementing character animation, procedural geometry, or physics-driven deformation pipelines on Linux using Vulkan or OpenGL compute.

**CPU vs. GPU split.** Roughly 70% of the material in this chapter is GPU-centric: algorithms that are data-parallel over vertices, triangles, or voxels map directly to Vulkan compute shaders and run entirely on the GPU after an initial dispatch. The remaining 30% is CPU-centric or hybrid, falling into two categories:

*CPU-only:* Mature production libraries whose APIs are inherently serial or topology-dependent — OpenSubdiv's Far layer (`TopologyRefiner`, `StencilTableFactory`), OpenCASCADE's `BRepMesh_IncrementalMesh`, meshoptimizer's simplification and encode/decode passes, and VHACD convex decomposition. These run on the CPU and upload their results (index buffers, stencil tables, convex hull vertex sets) as Vulkan buffers for GPU consumption.

*Hybrid:* Operations with a fast parallel component and a slower serial or data-dependent component. Examples: OpenSubdiv (CPU Far → GPU Osd evaluator), Poisson surface reconstruction (CPU reference library vs. GPU splat/solve/MC pipeline), mesh Boolean operations (GPU BVH broad-phase and clipping kernel, CPU global inside/outside classification), and constrained Delaunay triangulation (CPU Triangle library for constraint insertion, GPU for Ruppert refinement). In all hybrid cases the chapter shows the GPU-accelerable portion in compute shader code and notes which steps remain on CPU.

The sections below are ordered by geometry domain rather than CPU/GPU split. Where a section covers a CPU library, the practical GPU integration pattern (data layout for upload, barrier placement, descriptor set structure) is always shown alongside the library API.

---

## Table of Contents

1. [Subdivision Surfaces on the GPU](#1-subdivision-surfaces-on-the-gpu)
   - 1.1 [Catmull-Clark Subdivision: Theory](#11-catmull-clark-subdivision-theory)
   - 1.2 [Catmull-Clark Compute Shaders](#12-catmull-clark-compute-shaders)
   - 1.3 [Loop Subdivision for Triangle Meshes](#13-loop-subdivision-for-triangle-meshes)
   - 1.4 [OpenSubdiv 3.6.0: Far and Osd Layers](#14-opensubdiv-360-far-and-osd-layers)
   - 1.5 [Custom Vulkan Stencil Evaluator](#15-custom-vulkan-stencil-evaluator)
   - 1.6 [Subdivision + Skinning Interaction](#16-subdivision--skinning-interaction)
2. [NURBS and B-Spline Surfaces](#2-nurbs-and-b-spline-surfaces)
   - 2.1 [Cox-de Boor Recursion and De Boor's Algorithm](#21-cox-de-boor-recursion-and-de-boors-algorithm)
   - 2.2 [GPU Tessellation of Bézier Patches (TCS/TES)](#22-gpu-tessellation-of-bézier-patches-tcstes)
   - 2.3 [OpenCASCADE NURBS Tessellation for GPU Upload](#23-opencascade-nurbs-tessellation-for-gpu-upload)
   - 2.4 [Displacement Mapping on Tessellated Surfaces](#24-displacement-mapping-on-tessellated-surfaces)
   - 2.5 [PN Triangles and Phong Tessellation](#25-pn-triangles-and-phong-tessellation)
3. [Metaballs and Implicit Surfaces](#3-metaballs-and-implicit-surfaces)
   - 3.1 [Field Functions](#31-field-functions)
   - 3.2 [Marching Cubes on the GPU](#32-marching-cubes-on-the-gpu)
   - 3.3 [Dual Contouring with QEF Minimization](#33-dual-contouring-with-qef-minimization)
   - 3.4 [SDF Construction: Jump Flooding on the GPU](#34-sdf-construction-jump-flooding-on-the-gpu)
   - 3.5 [Procedural Noise Density Fields](#35-procedural-noise-density-fields)
   - 3.6 [NanoVDB: Sparse GPU Volumes](#36-nanovdb-sparse-gpu-volumes)
   - 3.7 [SDF CSG Operations](#37-sdf-csg-operations)
   - 3.8 [SPH Particle Fluid Surface Extraction](#38-sph-particle-fluid-surface-extraction)
   - 3.9 [GPU Delaunay Triangulation and Voronoi](#39-gpu-delaunay-triangulation-and-voronoi)
   - 3.10 [Marching Tetrahedra](#310-marching-tetrahedra)
   - 3.11 [Constrained Delaunay and Terrain-Quality Meshes](#311-constrained-delaunay-and-terrain-quality-meshes)
   - 3.12 [Poisson Surface Reconstruction](#312-poisson-surface-reconstruction)
4. [Skeletal Animation: Skinning](#4-skeletal-animation-skinning)
   - 4.1 [Forward Kinematics on GPU](#41-forward-kinematics-on-gpu)
   - 4.2 [Linear Blend Skinning (LBS)](#42-linear-blend-skinning-lbs)
   - 4.3 [Dual Quaternion Skinning (DQS)](#43-dual-quaternion-skinning-dqs)
   - 4.4 [GPU Compute Skinning Pre-Pass](#44-gpu-compute-skinning-pre-pass)
   - 4.5 [Blend Shapes (Morph Targets)](#45-blend-shapes-morph-targets)
   - 4.6 [PBD Cloth and Soft-Body Simulation](#46-pbd-cloth-and-soft-body-simulation)
5. [Inverse Kinematics on the GPU](#5-inverse-kinematics-on-the-gpu)
   - 5.1 [CCD: Cyclic Coordinate Descent](#51-ccd-cyclic-coordinate-descent)
   - 5.2 [FABRIK: Forward and Backward Reaching IK](#52-fabrik-forward-and-backward-reaching-ik)
   - 5.3 [Jacobian-Based IK and Damped Least Squares](#53-jacobian-based-ik-and-damped-least-squares)
6. [Spline Curves: GPU Tessellation](#6-spline-curves-gpu-tessellation)
   - 6.1 [Catmull-Rom Splines](#61-catmull-rom-splines)
   - 6.2 [Cubic Hermite and B-Spline Curves](#62-cubic-hermite-and-b-spline-curves)
   - 6.3 [Tube Geometry via Compute](#63-tube-geometry-via-compute)
   - 6.4 [Arc-Length Reparameterization](#64-arc-length-reparameterization)
7. [GPU Mesh Simplification and LOD](#7-gpu-mesh-simplification-and-lod)
   - 7.1 [Quadric Error Metric (Garland-Heckbert)](#71-quadric-error-metric-garland-heckbert)
   - 7.2 [meshoptimizer](#72-meshoptimizer)
   - 7.3 [Vertex Clustering on GPU](#73-vertex-clustering-on-gpu)
   - 7.4 [LOD Selection in Mesh Shaders](#74-lod-selection-in-mesh-shaders)
   - 7.5 [Smooth Normals Post-Pass](#75-smooth-normals-post-pass)
   - 7.6 [Procedural GPU Instancing: Grass, Hair, Fur](#76-procedural-gpu-instancing-grass-hair-fur)
   - 7.7 [Billboard and Impostor LOD Atlases](#77-billboard-and-impostor-lod-atlases)
   - 7.8 [Bent Normals Precomputation](#78-bent-normals-precomputation)
   - 7.9 [Geometry Compression and Quantization](#79-geometry-compression-and-quantization)
   - 7.10 [Mesh Stitching and Seam Welding](#710-mesh-stitching-and-seam-welding)
   - 7.11 [Mesh Boolean Operations: Polygon Clipping on GPU](#711-mesh-boolean-operations-polygon-clipping-on-gpu)
   - 7.12 [Harmonic and LSCM UV Parameterization](#712-harmonic-and-lscm-uv-parameterization)
   - 7.13 [GPU Parametric Texture Baking](#713-gpu-parametric-texture-baking)
   - 7.14 [GPU Mesh Repair](#714-gpu-mesh-repair)
   - 7.15 [Sphere Packing and Farthest-Point Sampling](#715-sphere-packing-and-farthest-point-sampling)
8. [GPU BVH Construction](#8-gpu-bvh-construction)
   - 8.1 [Morton Codes and LBVH](#81-morton-codes-and-lbvh)
   - 8.2 [LBVH Tree Construction (Karras 2012)](#82-lbvh-tree-construction-karras-2012)
   - 8.3 [AABB Propagation](#83-aabb-propagation)
   - 8.4 [GPU Narrow-Phase Collision: SAT and GJK](#84-gpu-narrow-phase-collision-sat-and-gjk)
   - 8.5 [Convex Hull Generation on GPU](#85-convex-hull-generation-on-gpu)
9. [Screen-Space Geometry Techniques](#9-screen-space-geometry-techniques)
   - 9.1 [Screen-Space Tessellation and Dynamic LOD](#91-screen-space-tessellation-and-dynamic-lod)
   - 9.2 [Parallax Occlusion Mapping](#92-parallax-occlusion-mapping)
   - 9.3 [Temporal Geometry Reprojection](#93-temporal-geometry-reprojection)
10. [GPU-Driven Rendering and Virtual Geometry](#10-gpu-driven-rendering-and-virtual-geometry)
    - 10.1 [Meshlets and Cluster Hierarchies](#101-meshlets-and-cluster-hierarchies)
    - 10.2 [Amplification/Task Shader Culling Pipeline](#102-amplificationtask-shader-culling-pipeline)
    - 10.3 [Hierarchical Z-Buffer Occlusion Culling](#103-hierarchical-z-buffer-occlusion-culling)
    - 10.4 [Two-Phase Occlusion and Indirect Draw Compaction](#104-two-phase-occlusion-and-indirect-draw-compaction)
    - 10.5 [Virtual Geometry: Cluster DAG and Error-Metric LOD](#105-virtual-geometry-cluster-dag-and-error-metric-lod)
11. [Fluid Simulation on the GPU](#11-fluid-simulation-on-the-gpu)
    - 11.1 [MAC Grid and Semi-Lagrangian Advection](#111-mac-grid-and-semi-lagrangian-advection)
    - 11.2 [Pressure Projection via Conjugate Gradient](#112-pressure-projection-via-conjugate-gradient)
    - 11.3 [Position-Based Fluids (PBF) on GPU](#113-position-based-fluids-pbf-on-gpu)
    - 11.4 [FLIP: Hybrid Particle-Grid](#114-flip-hybrid-particle-grid)
    - 11.5 [Lattice Boltzmann Method (LBM)](#115-lattice-boltzmann-method-lbm)
12. [Deformable Bodies and Continuum Mechanics](#12-deformable-bodies-and-continuum-mechanics)
    - 12.1 [Mass-Spring Systems on GPU](#121-mass-spring-systems-on-gpu)
    - 12.2 [Corotational FEM on GPU](#122-corotational-fem-on-gpu)
    - 12.3 [Material Point Method (MPM) on GPU](#123-material-point-method-mpm-on-gpu)
13. [Procedural Geometry Generation](#13-procedural-geometry-generation)
    - 13.1 [GPU Terrain: Fractional Brownian Motion and Erosion](#131-gpu-terrain-fractional-brownian-motion-and-erosion)
    - 13.2 [L-Systems in Compute Shaders](#132-l-systems-in-compute-shaders)
    - 13.3 [Wave Function Collapse for Tile Geometry](#133-wave-function-collapse-for-tile-geometry)
    - 13.4 [Task Shader Procedural Instancing](#134-task-shader-procedural-instancing)
14. [Geodesics and Discrete Differential Geometry](#14-geodesics-and-discrete-differential-geometry)
    - 14.1 [Cotangent Laplacian](#141-cotangent-laplacian)
    - 14.2 [Heat Method for Geodesic Distance](#142-heat-method-for-geodesic-distance)
    - 14.3 [Mean Curvature Flow and Mesh Smoothing](#143-mean-curvature-flow-and-mesh-smoothing)
    - 14.4 [Principal Curvature Estimation on GPU](#144-principal-curvature-estimation-on-gpu)
15. [Shape Deformation Beyond Skinning](#15-shape-deformation-beyond-skinning)
    - 15.1 [As-Rigid-As-Possible (ARAP) Deformation](#151-as-rigid-as-possible-arap-deformation)
    - 15.2 [Cage-Based Deformation: Mean Value Coordinates](#152-cage-based-deformation-mean-value-coordinates)
    - 15.3 [Radial Basis Function Deformation](#153-radial-basis-function-deformation)
16. [3D Gaussian Splatting and Neural Geometry](#16-3d-gaussian-splatting-and-neural-geometry)
    - 16.1 [3DGS Representation and Projection](#161-3dgs-representation-and-projection)
    - 16.2 [Tile Rasterizer: Sort-by-Depth and Alpha Compositing](#162-tile-rasterizer-sort-by-depth-and-alpha-compositing)
    - 16.3 [Mesh Extraction from 3DGS and NeRF](#163-mesh-extraction-from-3dgs-and-nerf)
17. [Spatial Data Structures Beyond BVH](#17-spatial-data-structures-beyond-bvh)
    - 17.1 [GPU k-d Trees](#171-gpu-k-d-trees)
    - 17.2 [Uniform Grids and Cell-List Hashing](#172-uniform-grids-and-cell-list-hashing)
    - 17.3 [Sparse Voxel Octrees (SVO)](#173-sparse-voxel-octrees-svo)
18. [Spectral Geometry Processing](#18-spectral-geometry-processing)
    - 18.1 [Graph Laplacian Eigenbasis](#181-graph-laplacian-eigenbasis)
    - 18.2 [Bilateral Mesh Filtering](#182-bilateral-mesh-filtering)
    - 18.3 [Anisotropic Diffusion and Curvature Flow](#183-anisotropic-diffusion-and-curvature-flow)
19. [Navigation Meshes and GPU Pathfinding](#19-navigation-meshes-and-gpu-pathfinding)
    - 19.1 [Voxelisation for NavMesh Generation (Recast-style)](#191-voxelisation-for-navmesh-generation-recast-style)
    - 19.2 [Parallel Region Labelling via GPU BFS](#192-parallel-region-labelling-via-gpu-bfs)
    - 19.3 [GPU Flow Fields for Navigation](#193-gpu-flow-fields-for-navigation)
20. [Ray Tracing Geometry Pipeline](#20-ray-tracing-geometry-pipeline)
21. [Point Cloud Processing](#21-point-cloud-processing)
22. [Rigid Body Dynamics on GPU](#22-rigid-body-dynamics-on-gpu)
23. [Crowd and Multi-Agent Simulation](#23-crowd-and-multi-agent-simulation)
24. [GPU Remeshing](#24-gpu-remeshing)
25. [Fracture and Destruction](#25-fracture-and-destruction)
26. [Terrain LOD and Streaming](#26-terrain-lod-and-streaming)
27. [Geometric Deep Learning](#27-geometric-deep-learning)
28. [2D Vector Graphics on GPU](#28-2d-vector-graphics-on-gpu)
29. [Scientific Visualization Geometry](#29-scientific-visualization-geometry)
30. [Hair and Strand Geometry](#30-hair-and-strand-geometry)
31. [Level Set Methods](#31-level-set-methods)
32. [Differentiable Rendering and Inverse Geometry](#32-differentiable-rendering-and-inverse-geometry)
33. [GPU Particle Systems and Ribbon Geometry](#33-gpu-particle-systems-and-ribbon-geometry)
34. [Precomputed Visibility and Lightmap Baking](#34-precomputed-visibility-and-lightmap-baking)
35. [UV Seam Optimization and Atlas Packing](#35-uv-seam-optimization-and-atlas-packing)
36. [Swept Volumes and Minkowski Sums](#36-swept-volumes-and-minkowski-sums)
37. [Geometric Shape Morphing and Functional Maps](#37-geometric-shape-morphing-and-functional-maps)
38. [Decal and Surface Projection Geometry](#38-decal-and-surface-projection-geometry)
39. [Visibility Queries and GPU Occlusion](#39-visibility-queries-and-gpu-occlusion)
40. [Library Landscape](#40-library-landscape)
41. [Performance Reference](#41-performance-reference)
42. [Integrations](#42-integrations)

---

## 1. Subdivision Surfaces on the GPU

Subdivision surfaces refine a coarse control mesh toward a smooth limit surface through repeated topological splitting and vertex averaging. Two schemes dominate GPU workloads: Catmull-Clark (quadrilateral input, used in film VFX and CAD) and Loop (triangle input, used in game engines and real-time rendering).

### 1.1 Catmull-Clark Subdivision: Theory

Catmull-Clark (1978) takes an arbitrary polygon mesh and produces a quad-dominant mesh at each level. Each face gains a **face point**, each edge gains an **edge point**, and each vertex is moved to an updated **vertex point**.

**Face point** — centroid of all face corners:

```
F = (1/n) * Σ(corner_vertices)
```

**Edge point** — average of the two endpoints and the two adjacent face points:

```
E = (e₀ + e₁ + F₀ + F₁) / 4
```

**Vertex point** (interior, valence n):

```
P' = (F_avg + 2·R_avg + (n−3)·P) / n
```

where `F_avg` is the average of all adjacent face points and `R_avg` is the average of all adjacent edge midpoints.

**Boundary vertex** (no adjacent face on one side):

```
P' = (6·P + P_L + P_R) / 8
```

**Semi-sharp creases** are expressed as a crease weight s ≥ 0. For s ≥ 1, the edge is blended toward the sharp rule (edge midpoint only); for s < 1, the sharp and smooth positions are linearly blended, and s decrements by 1 per subdivision level. Infinitely sharp creases (s = ∞) use the boundary rule unconditionally.

[Source: Catmull & Clark (1978), "Recursively Generated B-Spline Surfaces on Arbitrary Topological Meshes", CAD]

### 1.2 Catmull-Clark Compute Shaders

One-level Catmull-Clark requires access to the mesh topology (adjacency lists) and can be decomposed into three sequential compute passes: face points, edge points, and vertex points.

**Data layout.** Store the quad-mesh in CSR (Compressed Sparse Row) format: a `faceOffsets[]` array mapping face index → offset into `faceVerts[]`, plus `vertFaceOffsets[]` mapping vertex index → offset into `vertFaces[]` for the 1-ring.

**Pass 1 — face point computation:**

```glsl
// face_points.comp
#version 450
layout(local_size_x = 64) in;

layout(set=0, binding=0) readonly  buffer Positions   { vec4 pos[];        };
layout(set=0, binding=1) readonly  buffer FaceVerts    { uint faceVerts[];  };
layout(set=0, binding=2) readonly  buffer FaceOffsets  { uint faceOff[];    };
layout(set=0, binding=3) writeonly buffer FacePoints   { vec4 facePoints[]; };
layout(push_constant) uniform PC { uint numFaces; };

void main() {
    uint f = gl_GlobalInvocationID.x;
    if (f >= numFaces) return;

    uint start = faceOff[f];
    uint end   = faceOff[f + 1u];
    uint n     = end - start;

    vec3 sum = vec3(0.0);
    for (uint i = start; i < end; ++i)
        sum += pos[faceVerts[i]].xyz;

    facePoints[f] = vec4(sum / float(n), 1.0);
}
```

**Pass 3 — vertex point computation (interior, valence n):**

```glsl
// vertex_points.comp
#version 450
layout(local_size_x = 64) in;

layout(set=0, binding=0) readonly buffer Positions   { vec4 pos[];         };
layout(set=0, binding=1) readonly buffer FacePoints  { vec4 facePoints[];  };
layout(set=0, binding=2) readonly buffer EdgeMids    { vec4 edgeMids[];    };
layout(set=0, binding=3) readonly buffer VertFaces   { uint vertFaces[];   };
layout(set=0, binding=4) readonly buffer VertFaceOff { uint vertFaceOff[]; };
layout(set=0, binding=5) readonly buffer VertEdges   { uint vertEdges[];   };
layout(set=0, binding=6) readonly buffer VertEdgeOff { uint vertEdgeOff[]; };
layout(set=0, binding=7) writeonly buffer NewPositions { vec4 newPos[];    };
layout(push_constant) uniform PC { uint numVerts; };

void main() {
    uint v = gl_GlobalInvocationID.x;
    if (v >= numVerts) return;

    uint fStart = vertFaceOff[v], fEnd = vertFaceOff[v + 1u];
    uint eStart = vertEdgeOff[v], eEnd = vertEdgeOff[v + 1u];
    uint n = fEnd - fStart;  // valence

    vec3 F_avg = vec3(0.0);
    for (uint i = fStart; i < fEnd; ++i)
        F_avg += facePoints[vertFaces[i]].xyz;
    F_avg /= float(n);

    vec3 R_avg = vec3(0.0);
    for (uint i = eStart; i < eEnd; ++i)
        R_avg += edgeMids[vertEdges[i]].xyz;
    R_avg /= float(n);

    vec3 P   = pos[v].xyz;
    newPos[v] = vec4((F_avg + 2.0*R_avg + float(n - 3)*P) / float(n), 1.0);
}
```

Between passes, insert `VkMemoryBarrier` with `srcAccessMask = VK_ACCESS_SHADER_WRITE_BIT` and `dstAccessMask = VK_ACCESS_SHADER_READ_BIT`, both in `VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT`. Each level of subdivision quadruples the face count, so memory requirements grow as 4^level.

### 1.3 Loop Subdivision for Triangle Meshes

Loop subdivision (Charles Loop, 1987) refines triangle meshes. Each triangle edge is split at a new **edge midpoint**, and each original vertex is moved to an **updated vertex point**.

**Interior edge midpoint:**

```
E = (3/8)·(v₀ + v₁) + (1/8)·(v₂ + v₃)
```

where v₀, v₁ are the edge endpoints and v₂, v₃ are the opposite vertices of the two adjacent triangles.

**Updated interior vertex (Warren's β, valence n):**

```
β(n) = 3/(8n)   for n > 3
β(3) = 3/16

P' = (1 − n·β)·P + β·Σ(neighbors)
```

**Boundary vertex:**

```
P' = (3/4)·P + (1/8)·(P_L + P_R)
```

Warren's β (1995) is the standard smooth variant; Loop's original formula used `β = (1/n)·(5/8 − (3/8 + cos(2π/n)/4)²)`.

[Source: Loop (1987), "Smooth Subdivision Surfaces Based on Triangles", MS Thesis, Utah]

### 1.4 OpenSubdiv 3.6.0: Far and Osd Layers

OpenSubdiv ([GitHub](https://github.com/PixarAnimationStudios/OpenSubdiv), Apache 2.0) is the industry-standard subdivision library. Version 3.6.0 (March 2024) is the current stable release.

The library separates topology processing (the `Far` layer) from GPU evaluation (the `Osd` layer).

**Far layer — topology refinement and stencil generation:**

```cpp
#include <opensubdiv/far/topologyDescriptor.h>
#include <opensubdiv/far/topologyRefinerFactory.h>
#include <opensubdiv/far/stencilTableFactory.h>
#include <opensubdiv/far/patchTableFactory.h>

// Build topology descriptor from mesh data
Far::TopologyDescriptor desc;
desc.numVertices        = numBaseVerts;
desc.numFaces           = numBaseFaces;
desc.numVertsPerFace    = faceSizes.data();    // 4 per quad
desc.vertIndicesPerFace = faceVerts.data();

Sdc::SchemeType type    = Sdc::SCHEME_CATMARK;
Sdc::Options    sdcOpts;
sdcOpts.SetVtxBoundaryInterpolation(Sdc::Options::VTX_BOUNDARY_EDGE_ONLY);

// Create topology refiner
Far::TopologyRefinerFactory<Far::TopologyDescriptor>::Options
    refinerOpts(type, sdcOpts);
Far::TopologyRefiner *refiner =
    Far::TopologyRefinerFactory<Far::TopologyDescriptor>::Create(
        desc, refinerOpts);

// Adaptive refinement (isolates extraordinary vertices)
Far::TopologyRefiner::AdaptiveOptions adaptOpts(/*maxIsolationLevel=*/3);
adaptOpts.useInfSharpPatch = true;  // exact crease patches, no refinement needed
refiner->RefineAdaptive(adaptOpts);

// Build stencil table (maps refined vertices → base-mesh linear combinations)
Far::StencilTableFactory::Options stencilOpts;
stencilOpts.generateOffsets    = true;
stencilOpts.generateIntermediateLevels = false;  // only finest level
const Far::StencilTable *stencilTable =
    Far::StencilTableFactory::Create(*refiner, stencilOpts);

// Build patch table for rendering (Gregory basis end-caps)
Far::PatchTableFactory::Options patchOpts;
patchOpts.SetEndCapType(Far::PatchTableFactory::Options::ENDCAP_GREGORY_BASIS);
const Far::PatchTable *patchTable =
    Far::PatchTableFactory::Create(*refiner, patchOpts);

// Append local point stencils (for Gregory patches)
const Far::StencilTable *localStencils = patchTable->GetLocalPointStencilTable();
const Far::StencilTable *combined =
    Far::StencilTableFactory::AppendLocalPointStencilTable(
        *refiner, stencilTable, localStencils);
```

**Osd layer — GPU evaluation with GLComputeEvaluator:**

```cpp
#include <opensubdiv/osd/glComputeEvaluator.h>
#include <opensubdiv/osd/glVertexBuffer.h>
#include <opensubdiv/osd/glPatchTable.h>

// Convert CPU stencil table to GPU representation
Osd::GLStencilTableSSBO *gpuStencils =
    Osd::GLStencilTableSSBO::Create(combined, nullptr);

// Create GL vertex buffers (base verts + refined verts)
int numTotalVerts = combined->GetNumControlVertices()
                  + combined->GetNumStencils();
Osd::GLVertexBuffer *vbo =
    Osd::GLVertexBuffer::Create(/*numElements=*/3, numTotalVerts);

// Upload base mesh positions
vbo->UpdateData(basePositions.data(),
                /*startElement=*/0, combined->GetNumControlVertices());

// Run stencil evaluation on GPU (dispatches a compute shader)
Osd::GLComputeEvaluator::EvalStencils(
    vbo,    // source vertex buffer
    Osd::BufferDescriptor(0, 3, 3),   // base positions: offset=0, length=3, stride=3
    vbo,    // destination vertex buffer (refined positions written here)
    Osd::BufferDescriptor(combined->GetNumControlVertices() * 3, 3, 3),
    gpuStencils,
    nullptr);   // evaluator instance (null = use default instance)

// For rendering: build GL patch table
Osd::GLPatchTable *glPatchTable = Osd::GLPatchTable::Create(patchTable, nullptr);
// Bind patch index buffer and draw — see OpenSubdiv glViewer example for full draw loop
```

The `GLComputeEvaluator` dispatches OpenGL 4.3+ compute shaders stored in `opensubdiv/osd/glComputeEvaluator.cpp` in the OpenSubdiv source tree. There is no `Osd::VkComputeEvaluator` in OpenSubdiv 3.6.0; the available evaluators are: `CpuEvaluator`, `GLComputeEvaluator`, `GLXFBEvaluator`, `CudaEvaluator`, `CLEvaluator`, `MTLComputeEvaluator` (Metal, added in 3.5.1).

**GPU porting status.** The Far layer (`TopologyRefiner`, `StencilTableFactory`, `PatchTableFactory`) remains CPU-only and is unlikely to be ported. Topology analysis requires pointer-chasing over irregular adjacency graphs — valence lookup, half-edge traversal, crease propagation — which are data-dependent serial traversals that map poorly to GPU SIMD. The OpenSubdiv team's explicit position is that the topology phase belongs on CPU; GPU parallelism begins at stencil evaluation (Osd). A `Osd::VkComputeEvaluator` has been discussed in OpenSubdiv GitHub issues (e.g., #1116) but has not been implemented as of 3.6.0. The custom Vulkan stencil evaluator in §1.5 is the current recommended workaround for Vulkan-native pipelines.

### 1.5 Custom Vulkan Stencil Evaluator

For pure Vulkan pipelines, replicate the stencil evaluation in a Vulkan compute shader. The `Far::StencilTable` can be serialized as three arrays:

- `sizes[]` — number of control-vertex contributions per refined vertex
- `offsets[]` — prefix sum of sizes
- `indices[]` — source control-vertex indices
- `weights[]` — corresponding blend weights

```glsl
// vulkan_stencil_eval.comp
#version 450
layout(local_size_x = 64) in;

layout(set=0, binding=0) readonly buffer SrcVerts { vec3 src[]; };   // base mesh
layout(set=0, binding=1) readonly buffer Sizes    { uint sizes[]; };
layout(set=0, binding=2) readonly buffer Offsets  { uint offsets[]; };
layout(set=0, binding=3) readonly buffer Indices  { uint indices[]; };
layout(set=0, binding=4) readonly buffer Weights  { float weights[]; };
layout(set=0, binding=5) writeonly buffer DstVerts { vec3 dst[]; };  // refined verts
layout(push_constant) uniform PC { uint numRefined; };

void main() {
    uint v = gl_GlobalInvocationID.x;
    if (v >= numRefined) return;

    vec3 result  = vec3(0.0);
    uint off     = offsets[v];
    uint n       = sizes[v];
    for (uint i = 0u; i < n; ++i)
        result += weights[off + i] * src[indices[off + i]];
    dst[v] = result;
}
```

Run one dispatch per primvar channel (positions, normals, UV). This pattern also applies to evaluating the `LocalPointStencilTable` for Gregory patch control points. Upload the four CSR arrays from `Far::StencilTable` member accessors: `GetSizes()`, `GetOffsets()`, `GetControlIndices()`, `GetWeights()`.

### 1.6 Subdivision + Skinning Interaction

A practical dilemma in character rendering: should you skin the coarse control mesh and then evaluate the subdivision limit surface, or refine to a target level first and then apply per-vertex skinning?

**Option A — Skin control cage, then evaluate limit surface (OpenSubdiv's recommended path):**

Apply LBS or DQS to the N_base control cage vertices each frame, then run stencil evaluation on the deformed cage to produce the refined positions:

```cpp
// 1. Compute skinned_base_positions[] via GPU compute pre-pass (§4.3)
// 2. Upload skinned_base to the OpenSubdiv source vertex buffer
vbo->UpdateData(skinned_base_positions.data(), 0, numControlVerts);
// 3. Stencil evaluation: deformed cage → refined limit positions
Osd::GLComputeEvaluator::EvalStencils(vbo, srcDesc, vbo, dstDesc, gpuStencils);
// 4. Render from refined positions
```

Cost: `N_base × 4 bone influences + stencil evaluation`. Stencil weights are computed in bind pose and remain valid for arbitrary deformations since the subdivision rules are topological. Under extreme twist poses, extraordinary vertex normals may show minor artifacts; the `Far::LimitStencilTable` (which evaluates first and second derivatives at the limit surface directly) reduces this.

**Option B — Subdivide bind pose, then skin refined mesh:**

Refine the bind-pose mesh offline to target level L. Each frame, run LBS/DQS on all 4^L × N_base refined vertices. This produces the highest quality but is prohibitively expensive for L ≥ 2 on character meshes.

**UV seam handling:** Face-varying (UV) stencil tables are topology-dependent and must be re-evaluated after position stencils using the *original* UV control values — UVs do not deform with the skeleton. Always evaluate position and UV stencils as separate dispatches with separate source/destination descriptors.

```cpp
// Position primvar
Osd::GLComputeEvaluator::EvalStencils(vbo, posDescSrc, vbo, posDescDst, posStencils);
// UV primvar (uses same topology stencils but different source/dest offsets)
Osd::GLComputeEvaluator::EvalStencils(vbo, uvDescSrc, vbo, uvDescDst, fvarStencils);
```

---

## 2. NURBS and B-Spline Surfaces

### 2.1 Cox-de Boor Recursion and De Boor's Algorithm

A B-spline basis function of degree p is defined by the Cox-de Boor recursion:

```
N_{i,0}(t) = 1  if t_{i} ≤ t < t_{i+1}, else 0

N_{i,p}(t) =  (t − t_i) / (t_{i+p} − t_i)      · N_{i,p−1}(t)
            + (t_{i+p+1} − t) / (t_{i+p+1} − t_{i+1}) · N_{i+1,p−1}(t)
```

Division by zero is treated as 0/0 = 0.

De Boor's algorithm evaluates a B-spline curve at parameter t without explicitly constructing the basis functions. Given a knot span [t_r, t_{r+1}) containing t, and control points P_{r-p}, …, P_r:

```glsl
// De Boor's algorithm in GLSL (degree p = 3, cubic)
vec3 deboor(vec3 ctrlPts[64], float knots[68], int r, float t, int p) {
    // Copy the p+1 relevant control points into a local array
    vec3 d[4];
    for (int j = 0; j <= p; ++j)
        d[j] = ctrlPts[r - p + j];

    // Triangular reduction
    for (int k = 1; k <= p; ++k) {
        for (int j = p; j >= k; --j) {
            float ti    = knots[r - p + j];
            float tip1  = knots[r + 1 + j - k];
            float denom = tip1 - ti;
            float alpha = (denom < 1e-8) ? 0.0 : (t - ti) / denom;
            d[j] = mix(d[j-1], d[j], alpha);
        }
    }
    return d[p];
}
```

**Rational NURBS** use homogeneous coordinates (wx, wy, wz, w) and perform De Boor's algorithm in 4D:

```
P_hom(t) = deboor_4d(weighted_control_pts, knots, r, t, p)
P(t) = P_hom.xyz / P_hom.w
```

[Source: de Boor (1972), "On calculating with B-splines", JAMA; Piegl & Tiller, "The NURBS Book", 2nd ed. 1997]

### 2.2 GPU Tessellation of Bézier Patches (TCS/TES)

A bicubic Bézier patch with 4×4 = 16 control points is the standard GPU tessellation input. Convert a NURBS surface to Bézier form via knot insertion (Bézier extraction), then submit each patch to the tessellation pipeline.

**Tessellation Control Shader — adaptive LOD based on screen-space projected edge length:**

```glsl
#version 450
layout(vertices = 16) out;

layout(location = 0) in  vec3 in_pos[];
layout(location = 0) out vec3 tc_pos[];

layout(set=0, binding=0) uniform Camera { mat4 view_proj; vec2 viewport; float tess_scale; };

float screenLen(vec3 a, vec3 b) {
    vec4 sa = view_proj * vec4(a, 1.0);
    vec4 sb = view_proj * vec4(b, 1.0);
    vec2 nda = (sa.xy / sa.w) * 0.5 * viewport;
    vec2 ndb = (sb.xy / sb.w) * 0.5 * viewport;
    return length(ndb - nda);
}

void main() {
    tc_pos[gl_InvocationID] = in_pos[gl_InvocationID];

    if (gl_InvocationID == 0) {
        // Edge midpoints for LOD computation (4x4 patch, 4 outer edges)
        float l0 = screenLen(in_pos[0],  in_pos[12]) * tess_scale / 32.0;
        float l1 = screenLen(in_pos[0],  in_pos[3])  * tess_scale / 32.0;
        float l2 = screenLen(in_pos[3],  in_pos[15]) * tess_scale / 32.0;
        float l3 = screenLen(in_pos[12], in_pos[15]) * tess_scale / 32.0;

        gl_TessLevelOuter[0] = clamp(l0, 1.0, 64.0);
        gl_TessLevelOuter[1] = clamp(l1, 1.0, 64.0);
        gl_TessLevelOuter[2] = clamp(l2, 1.0, 64.0);
        gl_TessLevelOuter[3] = clamp(l3, 1.0, 64.0);
        gl_TessLevelInner[0] = max(gl_TessLevelOuter[1], gl_TessLevelOuter[3]);
        gl_TessLevelInner[1] = max(gl_TessLevelOuter[0], gl_TessLevelOuter[2]);
    }
}
```

**Tessellation Evaluation Shader — cubic Bernstein product:**

```glsl
#version 450
layout(quads, fractional_even_spacing, ccw) in;

layout(location = 0) in  vec3 tc_pos[];
layout(location = 0) out vec3 te_pos_world;
layout(location = 1) out vec3 te_normal;

layout(push_constant) uniform PC { mat4 view_proj; mat4 model; };

vec4 bernstein3(float t) {
    float t2 = t*t, t3 = t2*t;
    float s  = 1.0 - t;
    float s2 = s*s, s3 = s2*s;
    return vec4(s3, 3.0*s2*t, 3.0*s*t2, t3);
}

vec4 bernstein3_deriv(float t) {
    float s = 1.0 - t;
    return vec4(-3.0*s*s,
                 3.0*s*s - 6.0*s*t,
                 6.0*s*t - 3.0*t*t,
                 3.0*t*t);
}

vec3 bilerp_patch(vec4 Bu, vec4 Bv) {
    vec3 pos = vec3(0.0);
    for (int j = 0; j < 4; ++j)
        for (int i = 0; i < 4; ++i)
            pos += Bu[i] * Bv[j] * tc_pos[j*4 + i];
    return pos;
}

void main() {
    float u = gl_TessCoord.x, v = gl_TessCoord.y;
    vec4 Bu  = bernstein3(u),       Bv  = bernstein3(v);
    vec4 dBu = bernstein3_deriv(u), dBv = bernstein3_deriv(v);

    vec3 pos  = bilerp_patch(Bu,  Bv);
    vec3 dpdu = bilerp_patch(dBu, Bv);
    vec3 dpdv = bilerp_patch(Bu,  dBv);

    te_normal    = normalize(cross(dpdu, dpdv));
    te_pos_world = (model * vec4(pos, 1.0)).xyz;
    gl_Position  = view_proj * vec4(te_pos_world, 1.0);
}
```

### 2.3 OpenCASCADE NURBS Tessellation for GPU Upload

OpenCASCADE (OCCT 8.0.0p1, LGPL 2.1+) provides production-quality NURBS handling under `TKMesh`. After tessellation, retrieve triangle geometry and upload to Vulkan:

```cpp
#include <BRepMesh_IncrementalMesh.hxx>
#include <BRep_Tool.hxx>
#include <TopExp_Explorer.hxx>
#include <BRepGProp_Face.hxx>

BRepMesh_IncrementalMesh mesher(shape,
    /*linearDeflection=*/0.1,
    /*isRelative=*/false,
    /*angularDeflection=*/0.5);
mesher.Perform();

std::vector<float>    positions, normals;
std::vector<uint32_t> indices;
uint32_t vertBase = 0;

TopExp_Explorer faceExp(shape, TopAbs_FACE);
for (; faceExp.More(); faceExp.Next()) {
    TopoDS_Face face = TopoDS::Face(faceExp.Current());
    TopLoc_Location loc;
    Handle(Poly_Triangulation) tri = BRep_Tool::Triangulation(face, loc);
    if (tri.IsNull()) continue;

    for (int i = 1; i <= tri->NbNodes(); ++i) {
        gp_Pnt pt = tri->Node(i).Transformed(loc);
        positions.push_back((float)pt.X());
        positions.push_back((float)pt.Y());
        positions.push_back((float)pt.Z());
    }

    BRepGProp_Face gpropFace(face);
    for (int i = 1; i <= tri->NbNodes(); ++i) {
        gp_Pnt2d uv = tri->UVNode(i);
        gp_Pnt P; gp_Vec N;
        gpropFace.Normal(uv.X(), uv.Y(), P, N);
        normals.push_back((float)N.X());
        normals.push_back((float)N.Y());
        normals.push_back((float)N.Z());
    }

    for (int t = 1; t <= tri->NbTriangles(); ++t) {
        int n1, n2, n3;
        tri->Triangle(t).Get(n1, n2, n3);
        if (face.Orientation() == TopAbs_REVERSED) std::swap(n2, n3);
        indices.push_back(vertBase + n1 - 1);
        indices.push_back(vertBase + n2 - 1);
        indices.push_back(vertBase + n3 - 1);
    }
    vertBase += tri->NbNodes();
}
// Upload positions/normals/indices to VkBuffer via staging
```

[Source: OCCT `src/ModelingAlgorithms/TKMesh/BRepMesh/BRepMesh_IncrementalMesh.cxx`]

**GPU porting status.** `BRepMesh_IncrementalMesh` is CPU-only and no GPU port is in progress. The tessellator is tightly coupled to OpenCASCADE's BRep topology kernel (half-edge structures, `TopoDS_Shape` ownership graphs), making extraction into a data-parallel GPU kernel impractical without redesigning the data model. For real-time use, the recommended alternative is to bypass OpenCASCADE tessellation entirely and evaluate NURBS surfaces directly via TCS/TES (§2.2), at the cost of losing exact BRep topology. CAD pipelines that must preserve topology (e.g., engineering simulation, CNC toolpath generation) have no GPU alternative and must run tessellation on CPU at asset-load time.

### 2.4 Displacement Mapping on Tessellated Surfaces

The §2.2 TCS/TES pipeline produces a smooth mathematical surface. In practice, hardware tessellation is almost always paired with a displacement map that pushes each tessellated vertex outward along its normal, adding high-frequency geometric detail without increasing base-mesh complexity.

**T-junction and crack prevention.** Before applying displacement, the TCS must ensure adjacent patches share identical outer tessellation levels on their shared edges. If patch A sets `gl_TessLevelOuter[1] = 12` and its neighbor sets `gl_TessLevelOuter[3] = 8` for the shared edge, a T-junction crack opens. The fix: store per-patch LOD values in a shared SSBO, and in the TCS read the neighbor's level for each shared edge and take the minimum:

```glsl
// In TCS, before writing gl_TessLevelOuter:
float myLevel    = computeScreenSpaceLOD(patchCenter);
float neighborL  = patchLOD[neighborPatchID_left];
float neighborR  = patchLOD[neighborPatchID_right];
gl_TessLevelOuter[0] = min(myLevel, neighborL);
gl_TessLevelOuter[2] = min(myLevel, neighborR);
gl_TessLevelOuter[1] = gl_TessLevelOuter[3] = myLevel;
gl_TessLevelInner[0] = gl_TessLevelInner[1]  = myLevel;
```

Use `fractional_even_spacing` in the TES `layout()` declaration to smooth visual popping when levels change between frames; this does not eliminate cracks on its own but makes the remaining visual transitions less abrupt.

**Displacement in the TES.** After computing the smooth surface position `pos` and normal `te_normal`, sample a heightmap and push the vertex:

```glsl
// Extended TES from §2.2, with displacement
layout(set=0, binding=1) uniform sampler2D displacementMap;
layout(push_constant) uniform PC {
    mat4  view_proj;
    mat4  model;
    float dispScale;
    float dispBias;    // negative for inward displacement
};

void main() {
    float u = gl_TessCoord.x, v = gl_TessCoord.y;

    // Smooth patch position and normal (as in §2.2)
    vec4 Bu = bernstein3(u), Bv = bernstein3(v);
    vec3 pos   = bilerp_patch(Bu, Bv);
    vec3 dpdu  = bilerp_patch(bernstein3_deriv(u), Bv);
    vec3 dpdv  = bilerp_patch(Bu, bernstein3_deriv(v));
    vec3 N_geo = normalize(cross(dpdu, dpdv));

    // UV from a bilinear blend of corner UVs passed via a second patch array
    vec2 uv    = mix(mix(tc_uv[0], tc_uv[3], u),
                     mix(tc_uv[12], tc_uv[15], u), v);

    // Sample and apply displacement
    float disp    = texture(displacementMap, uv).r * dispScale + dispBias;
    vec3  displaced = pos + disp * N_geo;

    // Recompute shading normal from displaced neighbours (central difference)
    float eps  = 0.001;
    float du_p = texture(displacementMap, uv + vec2(eps, 0)).r * dispScale + dispBias;
    float dv_p = texture(displacementMap, uv + vec2(0, eps)).r * dispScale + dispBias;
    vec3  dPos_du = dpdu + (du_p - disp) / eps * N_geo;
    vec3  dPos_dv = dpdv + (dv_p - disp) / eps * N_geo;
    te_normal     = normalize(cross(dPos_du, dPos_dv));

    te_pos_world = (model * vec4(displaced, 1.0)).xyz;
    gl_Position  = view_proj * vec4(te_pos_world, 1.0);
}
```

In production, a precomputed tangent-space normal map (baked from a high-poly mesh) in the fragment shader replaces the real-time TES normal derivation. The TES provides the coarse displaced position; the normal map provides the per-pixel shading normal. This combination — hardware tessellation + displacement + tangent-space normals — is the standard terrain pipeline in Unreal Engine's Landscape system, Godot's `ShaderMaterial` terrain nodes, and id Tech 7's surface subdivision.

### 2.5 PN Triangles and Phong Tessellation

Hardware TCS/TES requires OpenGL 4.0 / Vulkan 1.0 and has non-trivial pipeline complexity. For mobile and integrated GPUs where tessellation is slow or unsupported, **PN Triangles** (Alex Vlachos et al., GDC 2001) and **Phong Tessellation** (Boubekeur & Alexa 2008) achieve smooth silhouettes using only a vertex shader — no tessellation stage — by generating curved geometry in a geometry shader or through mesh-shader subdivision.

**PN Triangles** replace each input triangle with a cubic Bézier patch defined entirely by the triangle's three vertex positions and normals. The 10 Bézier control points are:

```glsl
// Compute PN triangle control points from 3 vertices (positions p0-p2, normals n0-n2)
// Corner points: b300=p0, b030=p1, b003=p2
vec3 b300 = p0, b030 = p1, b003 = p2;

// Edge points: project neighbour along normal plane
vec3 b210 = (2.0*p0 + p1 - dot(p1-p0, n0)*n0) / 3.0;
vec3 b120 = (2.0*p1 + p0 - dot(p0-p1, n1)*n1) / 3.0;
vec3 b021 = (2.0*p1 + p2 - dot(p2-p1, n1)*n1) / 3.0;
vec3 b012 = (2.0*p2 + p1 - dot(p1-p2, n2)*n2) / 3.0;
vec3 b102 = (2.0*p2 + p0 - dot(p0-p2, n2)*n2) / 3.0;
vec3 b201 = (2.0*p0 + p2 - dot(p2-p0, n0)*n0) / 3.0;

// Centre point: average of edge midpoints, pulled toward triangle interior
vec3 E    = (b210+b120+b021+b012+b102+b201) / 6.0;
vec3 V    = (p0+p1+p2) / 3.0;
vec3 b111 = E + (E - V) * 0.5;

// Normal quadratic blending: linear interpolation of normals
vec3 n200 = n0, n020 = n1, n002 = n2;
vec3 n110 = normalize(n0 + n1 - 2.0*dot(p1-p0, n0+n1)/dot(p1-p0, p1-p0)*(p1-p0));
vec3 n011 = normalize(n1 + n2 - 2.0*dot(p2-p1, n1+n2)/dot(p2-p1, p2-p1)*(p2-p1));
vec3 n101 = normalize(n2 + n0 - 2.0*dot(p0-p2, n2+n0)/dot(p0-p2, p0-p2)*(p0-p2));
```

In the TES (or equivalent), evaluate at barycentric coordinate `(u, v, w)` where `u+v+w=1`:

```glsl
// Evaluate cubic Bernstein on PN control points
vec3 pn_position(float u, float v, float w,
    vec3 b300, vec3 b030, vec3 b003,
    vec3 b210, vec3 b120, vec3 b021,
    vec3 b012, vec3 b102, vec3 b201, vec3 b111) {
    return b300*u*u*u + b030*v*v*v + b003*w*w*w
         + b210*3.0*u*u*v + b120*3.0*u*v*v
         + b021*3.0*v*v*w + b012*3.0*v*w*w
         + b102*3.0*u*w*w + b201*3.0*u*u*w
         + b111*6.0*u*v*w;
}

vec3 pn_normal(float u, float v, float w,
    vec3 n200, vec3 n020, vec3 n002,
    vec3 n110, vec3 n011, vec3 n101) {
    return normalize(n200*u*u + n020*v*v + n002*w*w
                   + n110*u*v + n011*v*w + n101*u*w);
}
```

**Phong Tessellation** (simpler, slightly lower quality) linearly interpolates the flat-shaded position, then projects each interpolated point onto the tangent plane of the nearest corner vertex normal:

```glsl
// Phong tessellation in TES — α controls blend between flat and curved
vec3 phong_tessellate(vec3 p_interp, vec3 p0, vec3 p1, vec3 p2,
                      vec3 n0, vec3 n1, vec3 n2,
                      float u, float v, float w, float alpha) {
    // Project interpolated point onto each vertex's tangent plane
    vec3 proj0 = p_interp - dot(p_interp - p0, n0) * n0;
    vec3 proj1 = p_interp - dot(p_interp - p1, n1) * n1;
    vec3 proj2 = p_interp - dot(p_interp - p2, n2) * n2;
    vec3 curved = u*proj0 + v*proj1 + w*proj2;
    return mix(p_interp, curved, alpha);  // alpha=0: flat, alpha=1: full curve
}
```

**Mobile and no-TES path.** Without hardware tessellation, emit subdivided triangles from a compute shader pre-pass: each input triangle fans into 4 or 16 sub-triangles, with PN control points evaluated per-vertex. The output writes to a vertex/index buffer consumed by a standard draw call. This produces the same visual result with zero pipeline complexity. [Source: Vlachos et al. "Curved PN Triangles", GDC 2001; Boubekeur & Alexa "Phong Tessellation", SIGGRAPH Asia 2008]

---

## 3. Metaballs and Implicit Surfaces

### 3.1 Field Functions

An implicit surface is the zero-set (or iso-set) of a scalar field F(p) = T. Metaballs compose field functions from multiple blob sources.

**Wyvill soft object (1986)** — C²-continuous with compact support, computationally preferable:

```
f(r) = (1 − r²/R²)³    for r ≤ R
f(r) = 0                for r > R
```

**Blinn metaball (1982):**

```
f(r) = e^(−b·r²)    (blobbyness parameter b)
```

**Total field for n blobs (threshold T = 0.5):**

```
F(p) = Σᵢ f(|p − cᵢ|)
Isosurface: F(p) = T
```

[Source: Blinn (1982), "A Generalization of Algebraic Surface Drawing", TOG; Wyvill et al. (1986), "Data Structure for Soft Objects", The Visual Computer]

### 3.2 Marching Cubes on the GPU

Lorensen and Cline (1987) classify each voxel cube by the sign of its 8 corners against the isovalue, producing a 256-case lookup table for triangle topology.

**Two-pass GPU approach.** Pass 1 counts triangles per cell (with `atomicAdd` to a global counter). After a prefix-scan pass (exclusive scan over per-cell triangle counts for compacted output), Pass 2 emits the actual triangles.

**Pass 1 — triangle counting:**

```glsl
// mc_count.comp
#version 450
layout(local_size_x = 8, local_size_y = 8, local_size_z = 8) in;

layout(set=0, binding=0) readonly  buffer Density    { float density[]; };
layout(set=0, binding=1) readonly  buffer EdgeLUT    { int edgeLUT[256]; };   // bitmask
layout(set=0, binding=2) readonly  buffer TriLUT     { int triLUT[256*16]; };
layout(set=0, binding=3)           buffer TriCount   { uint triCount; };      // atomic
layout(set=0, binding=4)           buffer CellTriCt  { uint cellTriCts[]; };

layout(push_constant) uniform PC { uvec3 gridDim; float isovalue; };

uint flatCell(uvec3 c) {
    return c.z * gridDim.y * gridDim.x + c.y * gridDim.x + c.x;
}

float sampleDensity(uvec3 c) {
    return density[flatCell(c)];
}

void main() {
    uvec3 cell = gl_GlobalInvocationID;
    if (any(greaterThanEqual(cell, gridDim - uvec3(1)))) return;

    const uvec3 offsets[8] = uvec3[8](
        uvec3(0,0,0), uvec3(1,0,0), uvec3(1,1,0), uvec3(0,1,0),
        uvec3(0,0,1), uvec3(1,0,1), uvec3(1,1,1), uvec3(0,1,1));

    float vals[8];
    uint  cubeIdx = 0u;
    for (int i = 0; i < 8; ++i) {
        vals[i] = sampleDensity(cell + offsets[i]);
        if (vals[i] < isovalue) cubeIdx |= (1u << i);
    }

    if (edgeLUT[cubeIdx] == 0) { cellTriCts[flatCell(cell)] = 0u; return; }

    uint nTris = 0u;
    for (int i = 0; i < 16 && triLUT[cubeIdx*16 + i] != -1; i += 3) ++nTris;
    cellTriCts[flatCell(cell)] = nTris;
    atomicAdd(triCount, nTris);
}
```

**Pass 2 — triangle emission** (after prefix scan into `cellOffsets[]`):

```glsl
// mc_emit.comp
layout(set=0, binding=5) readonly  buffer CellOffsets { uint cellOff[]; };
layout(set=0, binding=6) writeonly buffer Triangles   { vec4 tris[]; };  // interleaved pos+normal

const uvec2 edgePairs[12] = uvec2[12](
    uvec2(0,1),uvec2(1,2),uvec2(3,2),uvec2(0,3),
    uvec2(4,5),uvec2(5,6),uvec2(7,6),uvec2(4,7),
    uvec2(0,4),uvec2(1,5),uvec2(2,6),uvec2(3,7));

vec3 interpEdge(vec3 p0, vec3 p1, float v0, float v1, float iso) {
    float t = clamp((iso - v0) / (v1 - v0), 0.0, 1.0);
    return mix(p0, p1, t);
}

void main() {
    // Recompute cubeIdx, vals, offsets per cell (same as pass 1)
    // ...
    vec3 edgeVerts[12];
    for (int e = 0; e < 12; ++e) {
        uint i0 = edgePairs[e].x, i1 = edgePairs[e].y;
        edgeVerts[e] = interpEdge(cornerPos(cell, offsets[i0]),
                                  cornerPos(cell, offsets[i1]),
                                  vals[i0], vals[i1], isovalue);
    }

    uint outBase = cellOff[flatCell(cell)];
    for (int i = 0; triLUT[cubeIdx*16 + i] != -1; i += 3) {
        vec3 A = edgeVerts[triLUT[cubeIdx*16 + i + 0]];
        vec3 B = edgeVerts[triLUT[cubeIdx*16 + i + 1]];
        vec3 C = edgeVerts[triLUT[cubeIdx*16 + i + 2]];
        vec3 N = normalize(cross(B-A, C-A));
        uint t = outBase + uint(i/3);
        tris[t*6+0] = vec4(A, 0); tris[t*6+1] = vec4(N, 0);
        tris[t*6+2] = vec4(B, 0); tris[t*6+3] = vec4(N, 0);
        tris[t*6+4] = vec4(C, 0); tris[t*6+5] = vec4(N, 0);
    }
}
```

A `VkMemoryBarrier` is required between the count pass, the prefix-scan pass, and the emit pass. Insert `vkCmdPipelineBarrier` with `srcStageMask = VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT` and `dstStageMask = VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT`.

**Shared memory optimization.** For better density cache performance, load an 8³ tile + 1-cell border (10³ = 1000 threads) into shared memory cooperatively, then process the inner 8³ cells:

```glsl
layout(local_size_x = 10, local_size_y = 10, local_size_z = 10) in;
shared float shared_density[10][10][10];

void main() {
    uvec3 localId  = gl_LocalInvocationID;
    uvec3 tileBase = gl_WorkGroupID * 8u;
    uvec3 gid      = tileBase + localId - uvec3(1);
    if (all(lessThan(gid, gridDim)))
        shared_density[localId.z][localId.y][localId.x] = sampleDensity(gid);
    else
        shared_density[localId.z][localId.y][localId.x] = 0.0;
    barrier();
    // Only interior 8³ threads process cells (localId in [1,8] on each axis)
}
```

[Source: Lorensen & Cline (1987), "Marching Cubes", SIGGRAPH; NVIDIA Marching Cubes sample, https://developer.download.nvidia.com/compute/cuda/1.1-Beta/x86_64_website/projects/marchingCubes/doc/marchingCubes.pdf]

### 3.3 Dual Contouring with QEF Minimization

Dual contouring (Ju et al., 2002) places one vertex per cell at the minimizer of a Quadratic Error Function (QEF), using Hermite data (surface intersection point + gradient) at each edge crossing. This eliminates marching cubes' staircase artifacts near sharp features.

For a cell with k edge crossings, each providing point pᵢ and outward normal nᵢ:

```
E(x) = Σᵢ (nᵢ · (x − pᵢ))²
     = xᵀ AᵀA x − 2 bᵀ Ax + c

where A is k×3 (rows = nᵢ), bᵢ = nᵢ · pᵢ
Solve: (AᵀA) x = Aᵀb  — a 3×3 SPD system
```

```glsl
// QEF accumulation and solve in a compute shader

void accumulate_qef(inout float AtA[6], inout float Atb[3],
                    vec3 n, vec3 p) {
    // AtA stores upper triangle: [00, 01, 02, 11, 12, 22]
    AtA[0] += n.x*n.x; AtA[1] += n.x*n.y; AtA[2] += n.x*n.z;
    AtA[3] += n.y*n.y; AtA[4] += n.y*n.z; AtA[5] += n.z*n.z;
    float dp = dot(n, p);
    Atb[0] += n.x*dp; Atb[1] += n.y*dp; Atb[2] += n.z*dp;
}

vec3 solve_qef(float AtA[6], float Atb[3]) {
    mat3 M = mat3(
        AtA[0], AtA[1], AtA[2],
        AtA[1], AtA[3], AtA[4],
        AtA[2], AtA[4], AtA[5]);
    float det = determinant(M);
    if (abs(det) < 1e-8) {
        // Mass point fallback: average of edge crossing positions
        return vec3(Atb[0], Atb[1], Atb[2]) / float(max(1u, accumCount));
    }
    return inverse(M) * vec3(Atb[0], Atb[1], Atb[2]);
}
```

Dual contouring requires the gradient of the density field at edge crossings (not needed by marching cubes). For analytical fields (metaballs), compute the gradient analytically. For sampled fields, use central differencing.

[Source: Ju et al. (2002), "Dual Contouring of Hermite Data", SIGGRAPH]

### 3.4 SDF Construction: Jump Flooding on the GPU

The Jump Flooding Algorithm (JFA, Rong & Tan 2006) computes a 3D Voronoi/distance transform in O(log N) passes, each sweeping the 3D grid with step sizes N/2, N/4, …, 1.

```glsl
// jfa_step.comp — one pass of 3D JFA
layout(local_size_x = 4, local_size_y = 4, local_size_z = 4) in;

layout(set=0, binding=0) readonly  uniform image3D jfaIn;
layout(set=0, binding=1) writeonly uniform image3D jfaOut;
layout(push_constant) uniform PC { int step; };

void main() {
    ivec3 px = ivec3(gl_GlobalInvocationID);
    vec4  best = imageLoad(jfaIn, px);

    for (int dz = -1; dz <= 1; ++dz)
    for (int dy = -1; dy <= 1; ++dy)
    for (int dx = -1; dx <= 1; ++dx) {
        if (dx == 0 && dy == 0 && dz == 0) continue;
        ivec3 nb = px + ivec3(dx, dy, dz) * step;
        vec4  s  = imageLoad(jfaIn, nb);
        if (s.w < 0.0) continue;  // sentinel: no seed
        float d2 = dot(vec3(px) - s.xyz, vec3(px) - s.xyz);
        if (best.w < 0.0 || d2 < best.w)
            best = vec4(s.xyz, d2);
    }
    imageStore(jfaOut, px, best);
}
```

After ceil(log₂(maxDim)) passes, `sqrt(best.w)` gives the unsigned distance, and `normalize(vec3(px) - best.xyz)` gives the gradient. Initialize JFA by writing seed voxels as `vec4(seed_pos, 0.0)` and non-seeds as `vec4(0, 0, 0, -1.0)`.

### 3.5 Procedural Noise Density Fields

The most common marching cubes workload in games is procedural terrain: a 3D noise function generates a density field evaluated directly on the GPU, and marching cubes extracts the isosurface per frame or per chunk.

**3D gradient noise (Perlin-style) in GLSL:**

```glsl
uint hash3(uvec3 p) {
    uint x = p.x ^ (p.y * 2654435761u) ^ (p.z * 805459861u);
    x ^= x >> 17; x *= 0xbf324c81u;
    x ^= x >> 11; x *= 0x68bc6ad9u;
    x ^= x >> 16;
    return x;
}

vec3 gradient(uvec3 p) {
    uint h = hash3(p) & 15u;
    const vec3 grads[16] = vec3[16](
        vec3( 1, 1, 0), vec3(-1, 1, 0), vec3( 1,-1, 0), vec3(-1,-1, 0),
        vec3( 1, 0, 1), vec3(-1, 0, 1), vec3( 1, 0,-1), vec3(-1, 0,-1),
        vec3( 0, 1, 1), vec3( 0,-1, 1), vec3( 0, 1,-1), vec3( 0,-1,-1),
        vec3( 1, 1, 0), vec3(-1, 1, 0), vec3( 0,-1, 1), vec3( 0,-1,-1));
    return grads[h];
}

float fade(float t) { return t*t*t*(t*(t*6.0 - 15.0) + 10.0); }

float perlin3(vec3 p) {
    vec3 i = floor(p), f = fract(p);
    vec3 u = vec3(fade(f.x), fade(f.y), fade(f.z));

    float v000 = dot(gradient(uvec3(i)),              f);
    float v100 = dot(gradient(uvec3(i+vec3(1,0,0))),  f - vec3(1,0,0));
    float v010 = dot(gradient(uvec3(i+vec3(0,1,0))),  f - vec3(0,1,0));
    float v110 = dot(gradient(uvec3(i+vec3(1,1,0))),  f - vec3(1,1,0));
    float v001 = dot(gradient(uvec3(i+vec3(0,0,1))),  f - vec3(0,0,1));
    float v101 = dot(gradient(uvec3(i+vec3(1,0,1))),  f - vec3(1,0,1));
    float v011 = dot(gradient(uvec3(i+vec3(0,1,1))),  f - vec3(0,1,1));
    float v111 = dot(gradient(uvec3(i+vec3(1,1,1))),  f - vec3(1,1,1));

    return mix(mix(mix(v000,v100,u.x), mix(v010,v110,u.x), u.y),
               mix(mix(v001,v101,u.x), mix(v011,v111,u.x), u.y), u.z);
}

// Fractal Brownian Motion: octave sum at increasing frequency, decreasing amplitude
float fbm3(vec3 p, int octaves) {
    float val = 0.0, amp = 0.5, freq = 1.0;
    for (int i = 0; i < octaves; ++i) {
        val  += amp * perlin3(p * freq);
        amp  *= 0.5;
        freq *= 2.0;
    }
    return val;
}
```

**Terrain density function** (positive = solid, negative = air) for a Minecraft-style chunk system:

```glsl
// density_gen.comp — evaluated per voxel in a 32³ chunk at world position chunkOrigin
layout(local_size_x = 4, local_size_y = 4, local_size_z = 4) in;
layout(set=0, binding=0) writeonly buffer Density { float density[]; };
layout(push_constant) uniform PC { vec3 chunkOrigin; float voxelSize; };

float terrainDensity(vec3 worldPos) {
    // Base height field: solid below y=32, air above
    float d = 32.0 - worldPos.y;
    // Large-scale hills (4 octaves, ~200 unit wavelength)
    d += 20.0 * fbm3(worldPos * 0.005, 4);
    // Cave carving: 3D noise tunnel network
    float cave = fbm3(worldPos * 0.03 + vec3(17.3, 5.1, 31.7), 3);
    d -= 8.0 * max(0.0, cave - 0.25);
    // Ore vein density boost (metaball-style)
    d += 3.0 * max(0.0, 0.5 - length(fbm3(worldPos * 0.08 + vec3(91.4), 2)));
    return d;
}

void main() {
    uvec3 local  = gl_GlobalInvocationID;
    vec3  world  = chunkOrigin + vec3(local) * voxelSize;
    uint  idx    = local.z * 33u * 33u + local.y * 33u + local.x;
    density[idx] = terrainDensity(world);
}
```

Each 32³ chunk evaluates a 33³ density grid (one extra voxel on each face for edge connectivity), runs marching cubes (§3.2), and uploads the resulting triangle buffer to the GPU once per generation. Re-generation on terrain modification replaces only the affected chunks.

### 3.6 NanoVDB: Sparse GPU Volumes

NanoVDB ([GitHub](https://github.com/AcademySoftwareFoundation/openvdb), Apache 2.0, included in OpenVDB 9.0+) is a read-only, GPU-native representation of sparse VDB volumes. It serializes the OpenVDB tree — a 5-level hierarchy (root → 32³ internal nodes → 16³ internal nodes → 8³ leaf nodes) — into a single contiguous flat buffer suitable for Vulkan SSBO or CUDA upload. Inactive voxel regions are pruned and replaced by a background constant.

**CPU build and GPU transfer:**

```cpp
#include <openvdb/openvdb.h>
#include <nanovdb/NanoVDB.h>
#include <nanovdb/util/OpenToNanoVDB.h>

openvdb::initialize();
// Build or load an OpenVDB float grid (e.g., a signed-distance level set)
openvdb::FloatGrid::Ptr grid = openvdb::FloatGrid::create(/*background=*/1.0f);
// ... populate grid via openvdb::tools::levelSetSphere or fluid simulation export

// Convert to NanoVDB host buffer
auto handle = nanovdb::openToNanoVDB(*grid);

// handle.data() is a flat byte array; handle.size() is its byte count
// Upload to Vulkan SSBO via staging buffer:
VkBuffer nanoVDBBuf;
VmaAllocation nanoVDBAlloc;
// VmaCreateBuffer(..., handle.size(), VK_BUFFER_USAGE_STORAGE_BUFFER_BIT, ...)
// copyToGPU(stagingBuf, handle.data(), handle.size())
```

**GLSL accessor (NanoVDB float level-set):**

The NanoVDB GLSL utility header (`nanovdb/util/NanoVDB.glsl`, part of the OpenVDB repo) defines tree-descent accessors without branching using precomputed bitmask lookups. Include it via a `#include` directive enabled by `GL_GOOGLE_include_directive` or copy the relevant accessor functions manually.

```glsl
layout(set=0, binding=0, std430) readonly buffer NanoVDBGrid { uint nanoVDB[]; };

// After including NanoVDB.glsl (or equivalent):
// nanovdb_readFloat(buffer, worldPos, worldToIndex) → float SDF value

float sdfSample(vec3 worldPos, mat4 worldToIndex) {
    return nanovdb_readFloat(nanoVDB, worldPos, worldToIndex);
}

// Sphere tracing through the NanoVDB level set
vec3 sphereTrace(vec3 ro, vec3 rd, mat4 worldToIndex, float tMax) {
    float t = 0.0;
    for (int i = 0; i < 256; ++i) {
        float d = sdfSample(ro + rd * t, worldToIndex);
        if (abs(d) < 1e-3) break;
        t += d * 0.9;  // safety under-step factor
        if (t > tMax) break;
    }
    return ro + rd * t;
}
```

Use `VK_EXT_scalar_block_layout` (or `std430`) to match NanoVDB's internal memory packing. The `worldToIndex` matrix transforms world-space coordinates to NanoVDB index space (voxel units), read from the `GridData` metadata at the start of the buffer.

**Applications:** Blender's Cycles GPU renderer uses NanoVDB for smoke and fire volume rendering. Houdini's game baker exports NanoVDB grids for real-time game engine import. The VFX pipeline pattern is: fluid sim (Houdini/PhysBAM) → OpenVDB level set → `openToNanoVDB` → Vulkan SSBO → sphere-trace or density-integrate in fragment/compute shader.

---

### 3.7 SDF CSG Operations

Signed distance functions compose algebraically without topology overhead. The core Boolean operators are:

```glsl
float sdfUnion(float a, float b)     { return min(a, b); }
float sdfIntersect(float a, float b) { return max(a, b); }
float sdfSubtract(float a, float b)  { return max(a, -b); }
```

These are exact for convex primitives and approximate (but visually correct) for concave shapes. Inigo Quilez's **polynomial smooth union** blends a region of width `k` around the join seam:

```glsl
float smin(float a, float b, float k) {
    float h = max(k - abs(a - b), 0.0) / k;
    return min(a, b) - h * h * h * k * (1.0 / 6.0);
}
float smax(float a, float b, float k) {
    return -smin(-a, -b, k);
}
```

`smin` is C¹-continuous at the join boundary; for C²-continuity use the cubic or quartic variants from [Quilez 2018 SDF operations](https://iquilezles.org/articles/sdfops/).

**Primitive SDFs:**

```glsl
float sdSphere(vec3 p, float r)          { return length(p) - r; }
float sdBox(vec3 p, vec3 b)              { vec3 q = abs(p) - b; return length(max(q,0.)) + min(max(q.x,max(q.y,q.z)),0.); }
float sdCapsule(vec3 p, vec3 a, vec3 b, float r) {
    vec3 ab = b-a, ap = p-a;
    float t = clamp(dot(ap,ab)/dot(ab,ab), 0., 1.);
    return length(ap - t*ab) - r;
}
float sdTorus(vec3 p, vec2 t)            { vec2 q = vec2(length(p.xz)-t.x, p.y); return length(q)-t.y; }
```

**Domain operations** transform the query point `p` before evaluation, effectively deforming the shape without changing the SDF function:

```glsl
vec3 opRepeat(vec3 p, vec3 c) { return mod(p + 0.5*c, c) - 0.5*c; }
vec3 opRepeatLimited(vec3 p, float c, vec3 l) {
    return p - c * clamp(round(p/c), -l, l);
}
vec3 opTwist(vec3 p, float k) {
    float c = cos(k * p.y), s = sin(k * p.y);
    return vec3(c*p.x - s*p.z, p.y, s*p.x + c*p.z);
}
vec3 opBend(vec3 p, float k) {
    float c = cos(k * p.x), s = sin(k * p.x);
    return vec3(c*p.x - s*p.y, s*p.x + c*p.y, p.z);
}
```

**CSG scene function and sphere tracer:**

```glsl
float sceneSDF(vec3 p) {
    // A box with a cylindrical hole, smoothly unioned with a sphere
    float box      = sdBox(p - vec3(0,0,0), vec3(1.0));
    vec3  cylP     = p;
    float cyl      = sdCapsule(cylP, vec3(0,-2,0), vec3(0,2,0), 0.4);
    float hollowed = sdfSubtract(box, cyl);      // punch hole in box
    float ball     = sdSphere(p - vec3(2,0,0), 0.8);
    return smin(hollowed, ball, 0.3);            // smooth union with sphere
}

vec3 calcNormal(vec3 p) {                        // central difference gradient
    const float e = 0.001;
    return normalize(vec3(
        sceneSDF(p + vec3(e,0,0)) - sceneSDF(p - vec3(e,0,0)),
        sceneSDF(p + vec3(0,e,0)) - sceneSDF(p - vec3(0,e,0)),
        sceneSDF(p + vec3(0,0,e)) - sceneSDF(p - vec3(0,0,e))));
}

// Fragment shader sphere trace (ray origin ro, direction rd)
vec3 sphereTrace(vec3 ro, vec3 rd) {
    float t = 0.0;
    for (int i = 0; i < 128; ++i) {
        float d = sceneSDF(ro + rd * t);
        if (d < 0.001) return ro + rd * t;       // hit
        t += d;
        if (t > 100.0) break;                    // miss
    }
    return vec3(1e10);                           // sentinel
}
```

The sphere-trace loop converges in O(log(1/ε)) steps for well-formed SDFs. The dominant cost is `sceneSDF` evaluations — each primitive call is a handful of ALU instructions, so deeply nested CSG trees with 20–30 primitives run at 60 fps in a fragment shader at 1080p on modern GPUs.

**Compute-shader SDF grid baking.** For scenes with many or expensive primitives, bake the SDF into a 3D texture:

```glsl
// bake_sdf.comp  — one thread per voxel
layout(local_size_x=8, local_size_y=8, local_size_z=8) in;
layout(set=0, binding=0, r32f) uniform image3D sdfVolume;
layout(push_constant) uniform PC { vec3 origin; vec3 cellSize; };

void main() {
    ivec3 id   = ivec3(gl_GlobalInvocationID);
    vec3  pos  = origin + vec3(id) * cellSize;
    float dist = sceneSDF(pos);
    imageStore(sdfVolume, id, vec4(dist));
}
```

The baked volume is then sphere-traced via a 3D texture lookup at runtime with trilinear interpolation — much faster than re-evaluating the full scene function per frame, at the cost of discretization error proportional to `cellSize`.

### 3.8 SPH Particle Fluid Surface Extraction

Smoothed Particle Hydrodynamics (SPH) simulations output per-particle positions and densities each frame. Converting this particle cloud to a renderable mesh requires reconstructing a continuous scalar density field and then extracting the isosurface — connecting the SPH simulator to the marching cubes pipeline of §3.2.

**SPH kernel.** The standard cubic-spline kernel in GLSL: given a particle at `xⱼ` with smoothing radius `h`, its contribution to the density field at point `x` is:

```glsl
float sph_kernel(float r, float h) {
    float q = r / h;
    float sigma = 8.0 / (3.14159265 * h * h * h);
    if (q < 0.5) return sigma * (6.0*(q*q*q - q*q) + 1.0);
    if (q < 1.0) return sigma * 2.0 * pow(1.0 - q, 3.0);
    return 0.0;
}
```

**Density grid bake (`sph_density.comp`).** Each thread evaluates the density at one grid voxel by summing contributions from nearby particles. A spatial hash grid (bucket sort by cell) accelerates the neighbour query to O(1) average cost:

```glsl
// sph_density.comp — one thread per voxel
layout(local_size_x=8, local_size_y=8, local_size_z=8) in;

struct Particle { vec3 pos; float mass; };
layout(set=0, binding=0) readonly  buffer Particles { Particle particles[]; };
layout(set=0, binding=1) readonly  buffer HashCells { uint cellStart[];  };  // prefix sum
layout(set=0, binding=2) readonly  buffer HashParts { uint partIdx[];    };  // sorted particle IDs
layout(set=0, binding=3, r32f)     uniform image3D   densityGrid;

layout(push_constant) uniform PC {
    vec3  gridOrigin;
    float cellSize;      // voxel size in world units
    float h;             // SPH smoothing radius
    uint  gridDim;       // grid dimension (cubic)
};

void main() {
    ivec3 id   = ivec3(gl_GlobalInvocationID);
    vec3  x    = gridOrigin + (vec3(id) + 0.5) * cellSize;

    float rho  = 0.0;
    // Iterate over 3x3x3 neighbourhood in the hash grid
    ivec3 ci   = ivec3(floor((x - gridOrigin) / h));
    for (int dz=-1; dz<=1; ++dz)
    for (int dy=-1; dy<=1; ++dy)
    for (int dx=-1; dx<=1; ++dx) {
        ivec3 nc = ci + ivec3(dx, dy, dz);
        if (any(lessThan(nc, ivec3(0))) || any(greaterThanEqual(nc, ivec3(gridDim)))) continue;
        uint cell = nc.x + nc.y*gridDim + nc.z*gridDim*gridDim;
        for (uint p = cellStart[cell]; p < cellStart[cell+1]; ++p) {
            Particle pk = particles[partIdx[p]];
            float r = length(x - pk.pos);
            rho += pk.mass * sph_kernel(r, h);
        }
    }
    imageStore(densityGrid, id, vec4(rho));
}
```

Once the density grid is written, it feeds directly into the two-pass marching cubes pipeline from §3.2 (replace the metaball field with a 3D image sampler bound to `densityGrid`). The isosurface level is typically set to the SPH rest density `ρ₀` of the fluid.

**Frame pipeline.** The per-frame GPU pipeline for fluid rendering is:
1. **SPH simulation** — pressure-force compute → velocity integrate → position update
2. **Spatial hash rebuild** — count, prefix-scan, scatter by cell
3. **Density grid bake** — `sph_density.comp`
4. **Marching cubes** — count + emit (§3.2) using density grid as input field
5. **Smooth normals** — §7.5 post-pass on the extracted mesh
6. **Render** — PBR shading with refraction and foam opacity

This pipeline is used in real-time fluid demos on NVIDIA Flex (GPU PhysX), AMD's fluid demo for RDNA3, and Houdini's Karma GPU renderer for SPH particle renders. [Source: Müller et al. "Particle-based Fluid Simulation for Interactive Applications", SCA 2003; NVIDIA Flex SDK documentation]

### 3.9 GPU Delaunay Triangulation and Voronoi

Delaunay triangulation maximizes the minimum angle of all triangles, producing meshes with no degenerate slivers — ideal for point-cloud surface reconstruction, procedural terrain meshing from scattered elevation samples, and fluid surface refinement. Its dual, the Voronoi diagram, partitions space into nearest-neighbour regions useful for level-of-detail seeding and tiling.

**Jump flooding Voronoi (JFA revisited).** The JFA pass from §3.4 also computes the 2D/3D Voronoi diagram: each cell stores the ID (and position) of its nearest seed. On the GPU, JFA runs in O(log N) passes, each of O(N) thread cost, where N is the grid dimension — far faster than CPU Fortune's sweep for large grids.

For 2D terrain Voronoi (procedural biome regions, tiling):

```glsl
// voronoi_jfa.comp — 2D version; each texel stores nearest seed pos + ID
layout(local_size_x=16, local_size_y=16) in;
layout(set=0, binding=0)        uniform sampler2D inputMap;   // (seedX, seedY, seedID, dist)
layout(set=0, binding=1, rgba32f) uniform image2D  outputMap;
layout(push_constant) uniform PC { int step; ivec2 dims; };

void main() {
    ivec2 id  = ivec2(gl_GlobalInvocationID.xy);
    vec4  best = texelFetch(sampler2D(inputMap, ...), id, 0);
    float bestDist = (best.z >= 0.0) ? length(vec2(id) - best.xy) : 1e30;

    for (int dy = -1; dy <= 1; ++dy)
    for (int dx = -1; dx <= 1; ++dx) {
        ivec2 nc = id + ivec2(dx, dy) * step;
        if (any(lessThan(nc, ivec2(0))) || any(greaterThanEqual(nc, dims))) continue;
        vec4 cand = texelFetch(sampler2D(inputMap, ...), nc, 0);
        if (cand.z < 0.0) continue;
        float d = length(vec2(id) - cand.xy);
        if (d < bestDist) { bestDist = d; best = cand; }
    }
    imageStore(outputMap, id, best);
}
```

**Bowyer-Watson incremental Delaunay on GPU.** Full GPU Delaunay triangulation is algorithmically complex (circumcircle conflict sets form data-dependent dependency chains). The practical GPU approach is:

1. **Rasterize Voronoi cones** — render an N-sided pyramid per seed (apex at seed position, slope = 1.0) into a depth buffer; the depth test produces the exact Voronoi partition in O(N) rasterization work.
2. **Extract edges from Voronoi image** — identify adjacent region boundaries (pixels where the seed ID changes between neighbours) via a compute pass.
3. **Connect Delaunay edges** — each Voronoi edge corresponds to a Delaunay edge between the two seed IDs sharing that boundary; emit into an edge list.
4. **Fan-triangulate** — sort edges by seed, form Delaunay triangles from each seed's neighbour list.

The rasterized Voronoi approach runs in O(N) GPU time for N seeds, producing the exact Voronoi diagram and an approximate Delaunay triangulation (exact up to rasterization resolution). For terrain meshing at 100k sample points, this runs in <2 ms on a modern GPU.

**Applications.** GPU Voronoi/Delaunay is used in: procedural biome generation (each Voronoi cell = one biome with coherent terrain parameters), fluid mesh reconstruction (Poisson surface reconstruction seeds from SPH particles), and GPU-side LOD meshing where scattered high-frequency samples (from adaptive tessellation) need triangulation before mesh simplification (§7.1). [Source: Hoff et al. "Fast Computation of Generalized Voronoi Diagrams Using Graphics Hardware", SIGGRAPH 1999; Rong & Tan "Jump Flooding in GPU with Applications to Voronoi Diagram and Distance Transform", I3D 2006]

### 3.10 Marching Tetrahedra

Marching Cubes (§3.2) decomposes each cell ambiguously when two diagonally opposite corners have different signs — 256 case configurations reduce to 15 canonical patterns, but 6 of those patterns have two valid interpretations that can produce topological inconsistencies (holes or T-junctions) at cell boundaries. **Marching Tetrahedra** eliminates all ambiguity by subdividing each cube into 5 or 6 tetrahedra, each with only 2⁴ = 16 unambiguous sign configurations producing at most one triangle.

**Tetrahedral decomposition of a cube.** A unit cube can be subdivided into 5 tetrahedra (asymmetric) or 6 tetrahedra (symmetric, preferred for consistent orientation):

```glsl
// 6-tetrahedra decomposition: cube vertices v0..v7
// Each tet listed as 4 corner indices into the cube
const ivec4 TET_TABLE[6] = ivec4[6](
    ivec4(0, 1, 3, 7),
    ivec4(0, 1, 5, 7),
    ivec4(0, 2, 3, 7),
    ivec4(0, 4, 5, 7),
    ivec4(0, 2, 6, 7),
    ivec4(0, 4, 6, 7)
);
```

**Triangle extraction.** Each tetrahedron has 4 vertices; with sign assignments `{-,-,-,+}` through `{+,+,+,-}`, exactly 1 or 2 triangles are produced per configuration. The 16 cases reduce to 3 distinct topologies (0 triangles: all same sign; 1 triangle: one corner differs; 2 triangles: two corners differ):

```glsl
// mt_emit.comp — one thread per tetrahedron
layout(local_size_x=64) in;
layout(set=0, binding=0, r32f) uniform image3D field;       // scalar field
layout(set=0, binding=1) buffer TriOut { vec3 triVerts[]; };// output triangle soup
layout(set=0, binding=2) buffer Counter{ uint count; };
layout(push_constant) uniform PC { vec3 origin; float cellSize; float isoLevel; };

// Interpolate edge crossing
vec3 edgeLerp(vec3 p0, float v0, vec3 p1, float v1, float iso) {
    float t = (iso - v0) / (v1 - v0);
    return mix(p0, p1, t);
}

void main() {
    uint cellIdx = gl_GlobalInvocationID.x;
    ivec3 c = ivec3(cellIdx % 64, (cellIdx/64) % 64, cellIdx/4096);

    // Load 8 cube-corner field values
    float val[8];
    vec3  pos[8];
    for (int k=0; k<8; ++k) {
        ivec3 off = ivec3(k&1, (k>>1)&1, (k>>2)&1);
        val[k] = imageLoad(field, c + off).r;
        pos[k] = origin + vec3(c + off) * cellSize;
    }

    // Iterate over 6 tetrahedra per cell
    for (int t = 0; t < 6; ++t) {
        ivec4 tet  = TET_TABLE[t];
        uint  mask = 0u;
        for (int v=0; v<4; ++v)
            if (val[tet[v]] < isoLevel) mask |= (1u << v);

        if (mask == 0u || mask == 15u) continue; // all inside or all outside

        // 1-triangle cases: single vertex on minority side
        int flip = (mask == 1||mask==2||mask==4||mask==8) ? 0 : 1;
        // (emit triangle by finding the 3 or 6 edge crossings for 1 or 2 tris)
        // ... (full case table elided for brevity; 14 non-trivial cases)
        uint slot = atomicAdd(count, 3u);
        // triVerts[slot..slot+2] = computed edge crossings
    }
}
```

**Quality comparison with Marching Cubes:**

| Property | Marching Cubes | Marching Tetrahedra |
|---|---|---|
| Topological consistency | No (6 ambiguous cases) | Yes (unambiguous) |
| Triangle count | Lower (~2 tris/cell) | Higher (~3–4 tris/cell) |
| Triangle quality | Moderate | Moderate |
| Manifold output | Not guaranteed | Guaranteed for closed surfaces |
| GPU implementation | Simpler (15 cases) | Moderate (16 cases × 6 tets) |

Marching Tetrahedra is preferred in medical imaging (CT/MRI segmentation) and any application requiring a guaranteed manifold mesh — e.g., as input to a Laplacian smoothing pass or a mesh repair pipeline. Libraries: libigl `igl::marching_tets` (CPU reference), TetWild for quality tetrahedral mesh generation from triangle soups. [Source: Shirley & Tuchman "A Polygonal Approximation to Find the Isosurface in Implicit Surfaces", SIGGRAPH 1990; Bloomenthal "Polygonization of Implicit Surfaces", CAGD 1988]

### 3.11 Constrained Delaunay and Terrain-Quality Meshes

The unconstrained Delaunay triangulation from §3.9 maximizes minimum angles but ignores hard boundary edges (cliff edges, coastlines, road borders) that must appear exactly in the output mesh. **Constrained Delaunay Triangulation** (CDT) inserts edges that must be preserved, then re-triangulates the interior to remain as Delaunay as possible while respecting those constraints.

**Practical GPU approach.** Full CDT construction is difficult to parallelize. The practical GPU pipeline is:

1. **Seed placement** — scatter seeds from a heightmap or point cloud; high-curvature regions (detected via second derivative of height) receive denser seeds via a compute adaptive sampling pass.
2. **Voronoi/Delaunay** (§3.9) — build the unconstrained triangulation on GPU.
3. **Constraint insertion** (CPU, fast for 1000s of segments) — insert road/coastline/cliff polylines into the CDT using the Shewchuk CDT algorithm (Triangle library).
4. **Ruppert refinement** — insert Steiner points to bound minimum angle ≥ 20°; iterate until no more violations. On GPU this is a parallel edge-flip + point-insertion loop.

**GPU Ruppert refinement pass.** Identify all triangles with minimum angle < threshold via a compute classify pass, insert circumcentres as new vertices in a parallel scatter pass, then flip non-Delaunay edges:

```glsl
// ruppert_classify.comp — one thread per triangle
layout(local_size_x=64) in;
layout(set=0, binding=0) readonly  buffer Triangles { uvec3 tris[]; };
layout(set=0, binding=1) readonly  buffer Vertices  { vec2 verts[]; };
layout(set=0, binding=2) writeonly buffer BadTris   { uint bad[]; };
layout(set=0, binding=3) buffer    Counter          { uint badCount; };
layout(push_constant) uniform PC { float minAngleCos; };  // cos(20°) = 0.940

void main() {
    uint tid = gl_GlobalInvocationID.x;
    uvec3 t  = tris[tid];
    vec2 a=verts[t.x], b=verts[t.y], c=verts[t.z];

    // Compute all three angles via dot products
    float cosA = dot(normalize(b-a), normalize(c-a));
    float cosB = dot(normalize(a-b), normalize(c-b));
    float cosC = dot(normalize(a-c), normalize(b-c));
    float minCos = max(max(cosA, cosB), cosC);  // largest cos = smallest angle

    if (minCos > minAngleCos) {                 // minimum angle < threshold
        uint slot = atomicAdd(badCount, 1u);
        bad[slot] = tid;
    }
}
```

The circumcentre of each bad triangle is computed and inserted as a new vertex, respecting constraint segments (discard insertions that would violate a constraint). Ruppert's algorithm terminates for minimum angles ≤ 20.7° with no input angle sharper than 60°.

**Applications.** Constrained Delaunay + Ruppert refinement is the standard meshing pipeline for: FEM terrain simulation (seismic, hydrology), game terrain LOD transition zones, road/river network embedding into height-field meshes. The `Triangle` library (Shewchuk, public domain) provides the CPU CDT/Ruppert reference; libigl wraps it. For fully GPU terrain meshing pipelines, the CDT step typically runs on CPU for constraint insertion (milliseconds for road networks) while the refinement iterations run on GPU. [Source: Shewchuk "Triangle: Engineering a 2D Quality Mesh Generator and Delaunay Triangulator", WACG 1996; Ruppert "A Delaunay Refinement Algorithm for Quality 2-Dimensional Mesh Generation", J. Algorithms 1995]

**GPU porting status.** Unconstrained GPU Delaunay is largely solved (§3.9 JFA/rasterized-cone approach). The hard part — constraint edge insertion — remains CPU-only in all production tools. Constraint insertion requires locating the existing triangles that cross each constraint segment and re-triangulating the cavity, a data-dependent traversal that serializes easily. Academic GPU CDT work exists (Qi et al. "Computing Two-Dimensional Delaunay Triangulation Using Graphics Hardware", I3D 2012; rGPU-CDT) but none handles general constraint polylines robustly enough for production terrain pipelines. The GPU Ruppert refinement shown above (parallel bad-triangle detection + circumcentre scatter) is itself a GPU contribution — the angle classification and point insertion can run on GPU, while the CPU only manages the outer convergence loop.

### 3.12 Poisson Surface Reconstruction

Poisson surface reconstruction (Kazhdan & Hoppe 2013, "Screened Poisson Surface Reconstruction") converts an oriented point cloud — positions plus normals — into a watertight triangle mesh. It is the standard algorithm for reconstructing meshes from LiDAR scans, photogrammetry point clouds, SPH simulation particles (§3.8), and SDF-sampled surfaces.

**The core insight.** Define an indicator function χ that is 1 inside the surface and 0 outside. The gradient ∇χ equals the inward surface normal field (zero everywhere except at the surface). Given a set of oriented points `{(pᵢ, nᵢ)}`, construct a vector field **V** that approximates ∇χ, then solve the Poisson equation ∇²χ = ∇·**V** for χ. Extract the isosurface at χ = 0.5.

**GPU-accelerable Poisson pipeline:**

1. **Octree construction** — build an adaptive octree over the point cloud; deeper cells near high-point-density regions. One approach: Morton-sort points (§8.1 pattern), then split cells that exceed a count threshold.

2. **Vector field splatting (`poisson_splat.comp`)** — splat each oriented point's normal into the octree's function values at surrounding nodes:

```glsl
// poisson_splat.comp — scatter point normals into function grid
layout(local_size_x=64) in;
layout(set=0,binding=0) readonly  buffer Points { vec4 ptNorm[]; };  // xyz=pos, w unused
layout(set=0,binding=1) readonly  buffer Normals{ vec4 ptN[];    };  // xyz=normal
layout(set=0,binding=2) coherent  buffer VField { vec4 vfield[]; }; // 3D grid, xyz=V
layout(push_constant) uniform PC { vec3 origin; float cellSize; uint gridDim; };

// Trilinear splat of normal into surrounding 8 grid nodes
void main() {
    uint pi   = gl_GlobalInvocationID.x;
    vec3 p    = ptNorm[pi].xyz;
    vec3 n    = ptN[pi].xyz;
    vec3 lp   = (p - origin) / cellSize;   // local grid coords
    ivec3 c   = ivec3(floor(lp));
    vec3  f   = fract(lp);

    for (int dz=0; dz<2; ++dz)
    for (int dy=0; dy<2; ++dy)
    for (int dx=0; dx<2; ++dx) {
        float w  = (dx==0?(1-f.x):f.x) * (dy==0?(1-f.y):f.y) * (dz==0?(1-f.z):f.z);
        uint  gi = (c.x+dx) + (c.y+dy)*gridDim + (c.z+dz)*gridDim*gridDim;
        atomicAdd(vfield[gi].x, w * n.x);  // requires VK_EXT_shader_atomic_float
        atomicAdd(vfield[gi].y, w * n.y);
        atomicAdd(vfield[gi].z, w * n.z);
    }
}
```

3. **Divergence computation** — compute ∇·**V** at each grid node via central differences:

```glsl
// divergence.comp — one thread per grid node
layout(local_size_x=8, local_size_y=8, local_size_z=8) in;
layout(set=0,binding=0) readonly  buffer VField { vec4 vfield[]; };
layout(set=0,binding=1) writeonly buffer Div    { float div[];   };

void main() {
    ivec3 id = ivec3(gl_GlobalInvocationID);
    uint  N  = gridDim;
    uint  gi = id.x + id.y*N + id.z*N*N;

    float dvx = (vfield[gi+1].x    - vfield[gi-1].x)   / (2.0*cellSize);
    float dvy = (vfield[gi+N].y    - vfield[gi-N].y)    / (2.0*cellSize);
    float dvz = (vfield[gi+N*N].z  - vfield[gi-N*N].z)  / (2.0*cellSize);
    div[gi]   = dvx + dvy + dvz;
}
```

4. **Poisson solve** — solve ∇²χ = div on the grid. On GPU, a Conjugate Gradient solver (same pattern as §7.12 CG step) handles the sparse 7-point stencil Laplacian. For large grids (256³+), a multigrid V-cycle reduces iteration count from O(N²) to O(N log N):

```glsl
// poisson_jacobi.comp — one Jacobi relaxation step (simpler than CG, more iterations)
layout(local_size_x=8, local_size_y=8, local_size_z=8) in;
// Read chi from pingBuffer, write to pongBuffer
// chi[i,j,k] = (chi[i±1] + chi[j±1] + chi[k±1] - h²*div[i,j,k]) / 6.0
```

5. **Isosurface extraction** — once χ converges, run the two-pass Marching Cubes pipeline from §3.2 on the χ grid with isolevel 0.5.

**Screened Poisson.** The 2013 "screened" variant adds a data-fitting term that pulls the solution toward the original point positions (not just their normals), improving reconstruction accuracy near sparse regions and at open boundaries. The screening modifies the linear system: `(L + αI)χ = b - α·w` where `w` is a point-value constraint vector and α controls the screening weight.

**Library.** The reference implementation `PoissonRecon` (Kazhdan, MIT) runs on CPU in 10–60 s for 1M-point clouds; the GPU-accelerated steps (splat, divergence, CG solve, MC extract) reduce this to 1–5 s. CloudCompare and MeshLab expose PoissonRecon as a plugin. [Source: Kazhdan & Hoppe "Screened Poisson Surface Reconstruction", TOG 2013; Kazhdan et al. "Poisson Surface Reconstruction", SGP 2006]

**GPU porting status.** This is the most complete GPU port among the CPU-dominant sections. The pipeline shown above — normal splatting, divergence, Poisson solve, isosurface extraction — maps directly to GPU compute, and all four steps have GPU implementations. What does not yet exist is a production-quality open-source library that integrates all steps under a single GPU-native API. NVIDIA cuSPARSE handles the sparse CG solve; the splat and divergence steps are straightforward compute shaders as shown. The octree management and adaptive refinement (used in the full screened Poisson for non-uniform point densities) are harder to GPU-port because they require dynamic tree modification, which is the remaining CPU work in a hybrid pipeline. For uniform-grid Poisson (fixed voxel resolution), a fully GPU implementation is straightforward from the code in this section.

---

## 4. Skeletal Animation: Skinning

### 4.1 Forward Kinematics on GPU

A skeleton is a directed tree of joints; each joint's world transform depends on its parent's. Forward kinematics (FK) computes every joint's global transform by traversing the tree root-to-leaf. On the CPU this is a serial depth-first walk; on the GPU, the tree is processed one depth level per compute dispatch, so all joints at the same depth execute in parallel.

**Data layout.** Pack the skeleton into three SSBOs: parent indices, local transforms (as quaternion + translation, 8 floats per joint), and output global transforms (mat4, 16 floats per joint).

```glsl
// fk_level.comp  — one thread per joint at a given depth level
layout(local_size_x=64) in;

struct Joint { vec4 rot; vec3 trans; uint parentIdx; };  // 8 floats + 1 uint

layout(set=0, binding=0) readonly  buffer Joints  { Joint joints[]; };
layout(set=0, binding=1) readonly  buffer LocalXF { mat4 localXF[];  };   // local transform per joint
layout(set=0, binding=2) writeonly buffer GlobalXF{ mat4 globalXF[]; };   // written this dispatch
layout(push_constant)    uniform PC { uint jointCount; };

mat4 quatToMat4(vec4 q, vec3 t) {
    float x=q.x, y=q.y, z=q.z, w=q.w;
    return mat4(
        1-2*(y*y+z*z),   2*(x*y+w*z),   2*(x*z-w*y), 0,
          2*(x*y-w*z), 1-2*(x*x+z*z),   2*(y*z+w*x), 0,
          2*(x*z+w*y),   2*(y*z-w*x), 1-2*(x*x+y*y), 0,
        t.x,             t.y,             t.z,         1
    );
}

void main() {
    uint idx = gl_GlobalInvocationID.x;
    if (idx >= jointCount) return;

    mat4 local  = quatToMat4(joints[idx].rot, joints[idx].trans);
    uint parent = joints[idx].parentIdx;
    // Parent's global transform was written by the previous dispatch
    mat4 global = (parent == 0xFFFFFFFF) ? local : globalXF[parent] * local;
    globalXF[idx] = global;
}
```

The host dispatches once per depth level, inserting a `VkMemoryBarrier` (`srcAccess = SHADER_WRITE`, `dstAccess = SHADER_READ`) between dispatches so each level reads fully-committed parent transforms:

```c
for (uint32_t d = 0; d <= maxDepth; ++d) {
    // push level's joint list via a per-level indirect dispatch buffer
    vkCmdDispatch(cmd, (levelJointCounts[d] + 63) / 64, 1, 1);
    if (d < maxDepth) {
        VkMemoryBarrier bar = { VK_STRUCTURE_TYPE_MEMORY_BARRIER,
            .srcAccessMask = VK_ACCESS_SHADER_WRITE_BIT,
            .dstAccessMask = VK_ACCESS_SHADER_READ_BIT };
        vkCmdPipelineBarrier(cmd,
            VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT,
            VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT,
            0, 1, &bar, 0, NULL, 0, NULL);
    }
}
```

**Animation blending (NLerp).** To blend between two animation clips, NLerp the per-joint quaternions on the GPU before the FK dispatch:

```glsl
// nlerp_blend.comp
layout(set=0, binding=0) readonly  buffer ClipA { vec4 rotA[]; };
layout(set=0, binding=1) readonly  buffer ClipB { vec4 rotB[]; };
layout(set=0, binding=2) writeonly buffer BlendedRot { vec4 blended[]; };
layout(push_constant) uniform PC { float t; uint jointCount; };

void main() {
    uint i = gl_GlobalInvocationID.x;
    if (i >= jointCount) return;
    vec4 qa = rotA[i], qb = rotB[i];
    // Antipodality correction: ensure shorter-arc interpolation
    if (dot(qa, qb) < 0.0) qb = -qb;
    blended[i] = normalize(mix(qa, qb, t));
}
```

NLerp is not constant-speed (unlike SLerp) but runs with no transcendental instructions and is indistinguishable from SLerp for blending weights below 0.5 angular distance. For game characters with 50–200 joints, both the NLerp blend and the FK dispatch complete in well under 0.1 ms on discrete GPUs, making full GPU skeletal pipelines practical. [Source: ARM Mali GPU Best Practices, skeletal animation chapter; CDLOD2 paper §5.2 for level-of-detail FK scheduling]

### 4.2 Linear Blend Skinning (LBS)

Linear Blend Skinning blends vertex positions across bone influences: 

```
v' = Σᵢ wᵢ · Mᵢ · v

Mᵢ = globalTransform(joint[i]) × inverseBindMatrix[i]
```

Bone weights wᵢ sum to 1.0 and are typically limited to 4 influences per vertex (8 for high-fidelity characters).

**Vertex shader (Vulkan GLSL, 4 influences):**

```glsl
#version 450

layout(location = 0) in vec3  in_position;
layout(location = 1) in vec3  in_normal;
layout(location = 2) in vec2  in_uv;
layout(location = 3) in uvec4 in_joints;   // JOINTS_0 — bone indices
layout(location = 4) in vec4  in_weights;  // WEIGHTS_0 — sum to 1.0

layout(set=0, binding=0) uniform CameraUB { mat4 view_proj; };
layout(set=1, binding=0) readonly buffer SkinSSBO {
    mat4 joint_matrices[];  // globalTransform * inverseBindMatrix, pre-computed CPU-side
};

layout(location = 0) out vec3 out_world_pos;
layout(location = 1) out vec3 out_world_normal;
layout(location = 2) out vec2 out_uv;

void main() {
    mat4 skin =
        in_weights.x * joint_matrices[in_joints.x] +
        in_weights.y * joint_matrices[in_joints.y] +
        in_weights.z * joint_matrices[in_joints.z] +
        in_weights.w * joint_matrices[in_joints.w];

    vec4 world_pos    = skin * vec4(in_position, 1.0);
    // mat3(skin) is valid for rigid-body joints; use transpose(inverse(mat3(skin))) for non-uniform scale
    vec3 world_normal = normalize(mat3(skin) * in_normal);

    out_world_pos    = world_pos.xyz;
    out_world_normal = world_normal;
    out_uv           = in_uv;
    gl_Position      = view_proj * world_pos;
}
```

LBS suffers from the "candy-wrapper" artifact at high-twist rotations (the mesh collapses toward the bone axis). Dual Quaternion Skinning corrects this.

### 4.3 Dual Quaternion Skinning (DQS)

A dual quaternion encodes rotation and translation as `DQ = q_r + ε·q_d` where `ε² = 0` and `q_d = 0.5·t̂·q_r` (t̂ = translation as pure quaternion).

**Converting joint matrix to dual quaternion (CPU, per frame):**

```cpp
glm::quat q_r = glm::quat_cast(glm::mat3(M));
glm::vec3 t   = glm::vec3(M[3]);
glm::quat q_d = 0.5f * glm::quat(0.0f, t.x, t.y, t.z) * q_r;
// Upload as two vec4: [q_r.xyzw], [q_d.xyzw]
```

**Vertex shader (DQS with antipodality correction):**

```glsl
layout(set=1, binding=0) readonly buffer DQBuffer {
    vec4 dq_real[];  // q_r per joint
    vec4 dq_dual[];  // q_d per joint
};

vec3 dqs_apply(uvec4 joints, vec4 weights, vec3 pos) {
    vec4 q_r = weights.x * dq_real[joints.x];
    vec4 q_d = weights.x * dq_dual[joints.x];

    for (int i = 1; i < 4; ++i) {
        vec4 qr_i = dq_real[joints[i]];
        vec4 qd_i = dq_dual[joints[i]];
        // Antipodality: negate if the dot product with the pivot is negative
        float s  = (dot(qr_i, q_r) < 0.0) ? -1.0 : 1.0;
        q_r += s * weights[i] * qr_i;
        q_d += s * weights[i] * qd_i;
    }

    float len = length(q_r);
    q_r /= len; q_d /= len;

    // Apply dual quaternion: position + rotation + translation
    vec3 r  = q_r.xyz; float rw = q_r.w;
    vec3 d  = q_d.xyz; float dw = q_d.w;

    vec3 rotated = pos + 2.0*rw*cross(r, pos) + 2.0*cross(r, cross(r, pos));
    vec3 trans   = 2.0*(rw*d - dw*r + cross(r, d));
    return rotated + trans;
}
```

[Source: Kavan et al. (2007), "Skinning with Dual Quaternions", I3D. DOI: 10.1145/1230100.1230107]

### 4.4 GPU Compute Skinning Pre-Pass

Computing skinning in a compute pass and writing to a pre-skinned vertex buffer allows all subsequent rendering passes (opaque, shadow, reflections) to reuse the result without re-executing the vertex shader skinning math:

```glsl
// skin_prepass.comp
#version 450
layout(local_size_x = 64) in;

layout(set=0, binding=0) readonly buffer SrcPositions { vec4 src_pos[]; };
layout(set=0, binding=1) readonly buffer SrcNormals   { vec4 src_nor[]; };
layout(set=0, binding=2) readonly buffer SrcTangents  { vec4 src_tan[]; };
layout(set=0, binding=3) readonly buffer SrcJoints    { uvec4 joints[]; };
layout(set=0, binding=4) readonly buffer SrcWeights   { vec4  weights[]; };
layout(set=0, binding=5) readonly buffer JointMats    { mat4  joint_mats[]; };
layout(set=0, binding=6) writeonly buffer DstPositions { vec4 dst_pos[]; };
layout(set=0, binding=7) writeonly buffer DstNormals   { vec4 dst_nor[]; };
layout(set=0, binding=8) writeonly buffer DstTangents  { vec4 dst_tan[]; };
layout(push_constant) uniform PC { uint numVerts; };

void main() {
    uint i = gl_GlobalInvocationID.x;
    if (i >= numVerts) return;

    mat4 skin = weights[i].x * joint_mats[joints[i].x]
              + weights[i].y * joint_mats[joints[i].y]
              + weights[i].z * joint_mats[joints[i].z]
              + weights[i].w * joint_mats[joints[i].w];
    mat3 rot  = mat3(skin);

    dst_pos[i] = skin * vec4(src_pos[i].xyz, 1.0);
    dst_nor[i] = vec4(normalize(rot * src_nor[i].xyz), 0.0);
    dst_tan[i] = vec4(normalize(rot * src_tan[i].xyz), src_tan[i].w);
}
```

This pattern matches Unreal Engine's GPU Skin Cache (`r.SkinCache.Mode 1`). After the compute dispatch, bind `DstPositions`/`DstNormals`/`DstTangents` as vertex buffers for all render passes in the frame.

**glTF skinning data layout:**

The glTF 2.0 specification defines JOINTS_0 as `UNSIGNED_BYTE` or `UNSIGNED_SHORT` VEC4, and WEIGHTS_0 as normalized `FLOAT` VEC4. The per-frame joint matrix is:

```
jointMatrix[i] = globalNodeTransform(skin.joints[i]) × inverseBindMatrix[i]
```

where `inverseBindMatrix[i]` is stored as accessor type `MAT4` in the glTF binary.
[Source: https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html#skins]

### 4.5 Blend Shapes (Morph Targets)

Morph targets store per-vertex position deltas from the base mesh. The vertex shader interpolates among targets using weight values animated on the CPU:

```glsl
layout(location = 0) in vec3 base_pos;
layout(location = 1) in vec3 target0_delta;
layout(location = 2) in vec3 target1_delta;

layout(set=0, binding=0) uniform MorphWeights { float weights[MAX_TARGETS]; };

void main() {
    vec3 morphed_pos = base_pos
        + weights[0] * target0_delta
        + weights[1] * target1_delta;
    // When combined with skinning: morph first, then skin
    gl_Position = view_proj * vec4(morphed_pos, 1.0);
}
```

For large numbers of targets (100+ FACS blend shapes for facial animation), store deltas as a 2D texture atlas indexed by vertex and target index, and evaluate in a compute pre-pass analogous to the skinning pre-pass above.

### 4.6 PBD Cloth and Soft-Body Simulation

Position-Based Dynamics (Müller et al. 2007, Macklin et al. 2016 XPBD) simulates cloth by iteratively projecting constraint violations rather than solving forces. Every vertex carries position `p`, predicted position `p̃`, and velocity `v`. A GPU PBD frame consists of:

1. **Integration pass** — apply external forces (gravity, wind), predict new positions
2. **Constraint projection** — iteratively correct positions to satisfy distance and bending constraints
3. **Velocity update** — derive velocity from position delta; apply damping

**Integration pass (`pbd_integrate.comp`):**

```glsl
layout(local_size_x=64) in;
layout(set=0, binding=0) buffer Pos      { vec4 pos[];  };   // xyz=position, w=invMass
layout(set=0, binding=1) buffer PosOld   { vec4 posOld[];};
layout(push_constant) uniform PC { float dt; vec3 gravity; float damping; };

void main() {
    uint i = gl_GlobalInvocationID.x;
    vec4  p   = pos[i];
    float w   = p.w;
    if (w == 0.0) return;  // pinned vertex

    vec3  v   = (p.xyz - posOld[i].xyz) / dt;  // Verlet velocity
    v        += gravity * dt;
    v        *= pow(damping, dt);               // velocity damping
    posOld[i] = p;
    pos[i].xyz = p.xyz + v * dt;
}
```

**Stretch constraint projection (`pbd_stretch.comp`).** Each constraint is an edge (i, j) with rest length `d0`. For parallel GPU evaluation, edges are graph-colored into independent sets; each dispatch processes one color class:

```glsl
layout(local_size_x=64) in;
layout(set=0, binding=0) buffer Pos       { vec4 pos[]; };
layout(set=0, binding=1) readonly buffer Edges {
    uint   edgeA[];    // vertex index A
    uint   edgeB[];    // vertex index B
    float  restLen[];  // rest length d0
};
layout(push_constant) uniform PC { uint edgeCount; float stiffness; };

void main() {
    uint e = gl_GlobalInvocationID.x;
    if (e >= edgeCount) return;

    uint  i  = edgeA[e], j = edgeB[e];
    vec3  pi = pos[i].xyz, pj = pos[j].xyz;
    float wi = pos[i].w,   wj = pos[j].w;
    float d  = length(pi - pj);
    float C  = d - restLen[e];                  // constraint violation
    if (abs(C) < 1e-6) return;

    vec3  n  = (pi - pj) / (d + 1e-10);
    float s  = -C / (wi + wj) * stiffness;      // XPBD: include compliance term here
    pos[i].xyz += s * wi * n;
    pos[j].xyz -= s * wj * n;
}
```

Graph coloring (Welsh-Powell or Jones-Plassman on GPU via independent-set stripping) groups edges so no two edges in a color class share a vertex, making the per-color dispatch race-free.

**Dihedral bending constraint.** A bending constraint acts on a quad formed by two adjacent triangles sharing edge (p1, p2), with tips at p0 and p3. The constraint is `C = θ − θ₀` where `θ` is the current dihedral angle. The gradient computation follows Müller 2007 Appendix A:

```glsl
// Simplified dihedral bend for one shared edge
float dihedralConstraint(vec3 p0, vec3 p1, vec3 p2, vec3 p3, float theta0) {
    vec3 n1 = normalize(cross(p1-p0, p2-p0));
    vec3 n2 = normalize(cross(p1-p0, p3-p0));
    float cosTheta = clamp(dot(n1, n2), -1.0, 1.0);
    return acos(cosTheta) - theta0;
}
```

**Capsule collision.** For character cloth–body interaction, resolve each vertex against a set of capsules (a simplified body collider) in a post-projection pass:

```glsl
vec3 capsuleConstraint(vec3 p, vec3 capA, vec3 capB, float r) {
    vec3 ab = capB - capA, ap = p - capA;
    float t  = clamp(dot(ap, ab) / dot(ab, ab), 0.0, 1.0);
    vec3 closest = capA + t * ab;
    float dist = length(p - closest);
    if (dist < r) return closest + normalize(p - closest) * r;
    return p;
}
```

**Iteration count and stiffness.** PBD convergence improves with more solver iterations (typically 4–16 per frame). XPBD (Macklin 2016) introduces a compliance parameter `α = 1/(k·dt²)` that makes stiffness iteration-count-independent, allowing coarser iteration schedules with physically meaningful material parameters. [Source: Müller et al. "Position Based Dynamics", VRIPHYS 2007; Macklin et al. "XPBD", MIG 2016]

---

## 5. Inverse Kinematics on the GPU

IK solvers within a single chain are inherently sequential, but multiple independent chains (different characters, tentacles, ragdoll limbs) parallelize trivially: one workgroup per chain.

### 5.1 CCD: Cyclic Coordinate Descent

CCD iterates from the end-effector's parent joint back to the root, rotating each joint to bring the end-effector closer to the target:

```
For j from (n−2) down to 0:
  e_vec = normalize(worldPos(end_effector) − worldPos(joint[j]))
  t_vec = normalize(target − worldPos(joint[j]))
  axis  = normalize(cross(e_vec, t_vec))
  angle = acos(clamp(dot(e_vec, t_vec), −1, 1))
  joint[j].rotation = rotationAround(axis, angle) · joint[j].rotation
  // Apply joint limits; recompute FK from j+1 onward
```

```glsl
// ccd.comp — one workgroup per chain
layout(local_size_x = 1) in;
layout(set=0, binding=0) buffer Chains  { ChainData chains[]; };
layout(set=0, binding=1) buffer Joints  { JointData joints[]; };
layout(set=0, binding=2) readonly buffer Targets { vec3 targets[]; };
layout(push_constant) uniform PC { uint maxIter; float tolerance; };

void main() {
    uint chainIdx = gl_WorkGroupID.x;
    ChainData chain = chains[chainIdx];
    vec3 target = targets[chainIdx];

    for (uint iter = 0u; iter < maxIter; ++iter) {
        // FK pass: compute worldPos[] for all joints in this chain
        // ...
        if (length(worldPos[chain.numJoints-1] - target) < tolerance) break;

        for (int j = int(chain.numJoints)-2; j >= 0; --j) {
            vec3 e = normalize(worldPos[chain.numJoints-1] - worldPos[j]);
            vec3 t = normalize(target - worldPos[j]);
            vec3 axis  = cross(e, t);
            float sinA = length(axis);
            float cosA = dot(e, t);
            if (sinA < 1e-6) continue;
            float angle = atan(sinA, cosA);
            quat dq = quat_from_axis_angle(normalize(axis), angle);
            joints[chain.jointStart + j].rotation =
                quat_mul(dq, joints[chain.jointStart + j].rotation);
            // Re-run FK from j+1 onward
        }
    }
}
```

[Source: Kenwright (2012), "Inverse Kinematics — CCD", JCGT 1(1)]

### 5.2 FABRIK: Forward and Backward Reaching IK

FABRIK (Aristidou & Lasenby, 2011) operates directly on positions and avoids explicit rotation computations, converging faster than CCD for 3D chains.

```
Forward pass (reach toward target):
  positions[n−1] = target
  for i from n−2 down to 0:
    r = |positions[i] − positions[i+1]|
    λ = boneLengths[i] / r
    positions[i] = (1−λ)·positions[i+1] + λ·positions[i]

Backward pass (anchor at root):
  positions[0] = rootPosition
  for i from 1 to n−1:
    r = |positions[i] − positions[i−1]|
    λ = boneLengths[i−1] / r
    positions[i] = (1−λ)·positions[i−1] + λ·positions[i]

Repeat until |positions[n−1] − target| < ε
```

```glsl
// fabrik.comp — one workgroup per chain; thread 0 runs serial passes
layout(local_size_x = MAX_JOINTS) in;
shared vec3 pos[MAX_JOINTS];

void main() {
    uint chainIdx = gl_WorkGroupID.x;
    uint j        = gl_LocalInvocationID.x;
    ChainData chain = chains[chainIdx];

    if (j < chain.numJoints) pos[j] = computeWorldPos(chain, j);
    barrier();

    vec3 root_pos = pos[0];
    vec3 target   = targets[chainIdx];

    for (uint iter = 0u; iter < maxIter; ++iter) {
        if (j == 0u) {
            // Forward pass
            pos[chain.numJoints-1] = target;
            for (int i = int(chain.numJoints)-2; i >= 0; --i) {
                float l = chain.boneLens[i];
                vec3  d = pos[i] - pos[i+1];
                float r = length(d);
                pos[i] = pos[i+1] + (r > 1e-8 ? l/r : 1.0) * d;
            }
            // Backward pass
            pos[0] = root_pos;
            for (int i = 1; i < int(chain.numJoints); ++i) {
                float l = chain.boneLens[i-1];
                vec3  d = pos[i] - pos[i-1];
                float r = length(d);
                pos[i] = pos[i-1] + (r > 1e-8 ? l/r : 1.0) * d;
            }
        }
        barrier();
        if (length(pos[chain.numJoints-1] - target) < tolerance) break;
    }

    // Convert final positions back to joint rotations via local bind frame
    if (j < chain.numJoints-1) {
        vec3 dir = normalize(pos[j+1] - pos[j]);
        joints[chain.jointStart + j].rotation =
            dir_to_quat(dir, chain.bindDirs[j]);
    }
}
```

[Source: Aristidou & Lasenby (2011), "FABRIK: A fast, iterative solver for the Inverse Kinematics problem", Graphical Models 73(5)]

### 5.3 Jacobian-Based IK and Damped Least Squares

For revolute joint i with world-space rotation axis aᵢ and joint position pᵢ, the Jacobian column for position IK is:

```
Jᵢ = aᵢ × (p_e − pᵢ)    (cross product)
```

Full Jacobian J is 3×n. The Damped Least Squares (DLS) update:

```
Δθ = Jᵀ (JJᵀ + λ²I)⁻¹ Δe
```

For small n (≤ 6 DOF), `JJᵀ` is a 3×3 matrix solved analytically in GLSL:

```glsl
mat3 JJT = J * transpose(J);
JJT[0][0] += lambda2; JJT[1][1] += lambda2; JJT[2][2] += lambda2;
mat3 invJJT = inverse(JJT);   // GLSL mat3 inverse
vec3 b      = invJJT * delta_e;
// Δθ[i] = dot(J_col[i], b)
```

For 6-DOF pose IK (position + orientation), `JJᵀ` becomes 6×6; implement Gaussian elimination in shared memory across workgroup threads.

**Position-Based Dynamics (PBD) soft IK** formulates joint constraints as distance-preserving projections:

```
Δp_a = −(1/2)·(|p_a − p_b| − d)·normalize(p_a − p_b)
Δp_b = +(1/2)·(|p_a − p_b| − d)·normalize(p_a − p_b)
```

Each substep projects all constraints (after graph coloring to identify independent sets) in parallel — an efficient GPU approach for soft-body and cloth-coupled IK.

[Source: Buss (2004), "Introduction to Inverse Kinematics with Jacobian Transpose, Pseudoinverse and Damped Least Squares methods"]

---

## 6. Spline Curves: GPU Tessellation

### 6.1 Catmull-Rom Splines

Catmull-Rom interpolates through control points (unlike B-splines, which only approximate them). The uniform parameterization with tension τ = 0.5:

```
P(t) = 0.5·[ (2·Pᵢ)
           + (−Pᵢ₋₁ + Pᵢ₊₁)·t
           + (2·Pᵢ₋₁ − 5·Pᵢ + 4·Pᵢ₊₁ − Pᵢ₊₂)·t²
           + (−Pᵢ₋₁ + 3·Pᵢ − 3·Pᵢ₊₁ + Pᵢ₊₂)·t³ ]
```

Basis weights as a vec4 for t ∈ [0,1]:

```
h.x =  0.5·(−t³ + 2t² − t)        // weight for Pᵢ₋₁
h.y =  0.5·( 3t³ − 5t² + 2)       // weight for Pᵢ
h.z =  0.5·(−3t³ + 4t² + t)       // weight for Pᵢ₊₁
h.w =  0.5·( t³ − t²)             // weight for Pᵢ₊₂
```

**Centripetal Catmull-Rom** (α = 0.5) avoids loops and cusps at sharp corners through chord-length parameterization: `tᵢ₊₁ = tᵢ + |Pᵢ₊₁ − Pᵢ|^α`.

**Domain shader for isolines (curve tessellation):**

```glsl
#version 450
layout(isolines, equal_spacing, point_mode) in;
layout(location = 0) in  vec3 tc_pos[];  // 4 control points per patch
layout(location = 0) out vec3 out_pos;
layout(push_constant) uniform PC { mat4 view_proj; };

void main() {
    float t = gl_TessCoord.x;
    float t2 = t*t, t3 = t2*t;

    vec4 h;
    h.x =  0.5*(-t3 + 2.0*t2 - t);
    h.y =  0.5*( 3.0*t3 - 5.0*t2 + 2.0);
    h.z =  0.5*(-3.0*t3 + 4.0*t2 + t);
    h.w =  0.5*( t3 - t2);

    vec3 p = h.x*tc_pos[0] + h.y*tc_pos[1] + h.z*tc_pos[2] + h.w*tc_pos[3];
    out_pos     = p;
    gl_Position = view_proj * vec4(p, 1.0);
}
```

[Source: Catmull & Rom (1974), "A class of local interpolating splines", Computer Aided Geometric Design]

### 6.2 Cubic Hermite and B-Spline Curves

**Cubic Hermite spline** — interpolates endpoints P₀, P₁ with explicit tangent vectors T₀, T₁:

```
P(t) = (2t³−3t²+1)·P₀ + (t³−2t²+t)·T₀ + (−2t³+3t²)·P₁ + (t³−t²)·T₁
```

The tangent relationship to Catmull-Rom: `T₀ = (P₂ − P₋₁)/2`, `T₁ = (P₃ − P₀)/2`.

**Uniform cubic B-spline** — approximates (does not interpolate) control points:

```
M_B = (1/6) · | 1   4   1  0 |
              |−3   0   3  0 |   applied to [Pᵢ₋₁, Pᵢ, Pᵢ₊₁, Pᵢ₊₂]ᵀ
              | 3  −6   3  0 |
              |−1   3  −3  1 |
```

**OpenSubdiv B-spline patch evaluation** (from `opensubdiv/osd/glslPatchBSpline.glsl`):

```glsl
vec4 OsdBSplineBasis(float t) {
    float t2 = t*t, t3 = t2*t;
    return (1.0/6.0) * vec4(
        −t3 + 3.0*t2 − 3.0*t + 1.0,
         3.0*t3 − 6.0*t2 + 4.0,
        −3.0*t3 + 3.0*t2 + 3.0*t + 1.0,
         t3);
}
```

For **non-uniform B-splines**, apply De Boor's algorithm (§2.1) with a general knot vector.

### 6.3 Tube Geometry via Compute

A curve tube is generated by evaluating the spline tangent, building a Frenet-Serret frame (or parallel-transport frame to avoid singularities at inflection points), and emitting a circle of vertices at each sampled point:

```glsl
// tube_gen.comp
layout(local_size_x = 32) in;

layout(set=0, binding=0) readonly  buffer CtrlPts  { vec3 ctrl[];  };
layout(set=0, binding=1) writeonly buffer TubeVerts { vec4 verts[]; };
layout(push_constant) uniform PC {
    uint numSegments;
    float radius;
    uint circleSubdivisions;
};

vec3 catmull_rom(vec3 P[4], float t) {
    float t2 = t*t, t3 = t2*t;
    return 0.5 * ((-t3+2*t2-t)*P[0] + (3*t3-5*t2+2)*P[1]
                + (-3*t3+4*t2+t)*P[2] + (t3-t2)*P[3]);
}

vec3 catmull_rom_tangent(vec3 P[4], float t) {
    float t2 = t*t;
    return 0.5*((-3*t2+4*t-1)*P[0] + (9*t2-10*t)*P[1]
              + (-9*t2+8*t+1)*P[2] + (3*t2-2*t)*P[3]);
}

void main() {
    uint sampleIdx = gl_GlobalInvocationID.x;
    uint segment   = sampleIdx / 32u;
    float t        = float(sampleIdx % 32u) / 31.0;

    vec3 P[4];
    P[0] = ctrl[max(segment,   1u) - 1u];
    P[1] = ctrl[segment];
    P[2] = ctrl[segment + 1u];
    P[3] = ctrl[min(segment+2u, numSegments)];

    vec3 center  = catmull_rom(P, t);
    vec3 tangent = normalize(catmull_rom_tangent(P, t));

    // Parallel-transport normal (avoids Frenet-Serret singularities at inflections)
    vec3 up     = (abs(tangent.y) < 0.9) ? vec3(0,1,0) : vec3(1,0,0);
    vec3 normal = normalize(cross(tangent, up));
    vec3 bitan  = cross(tangent, normal);

    const float PI = 3.14159265358979;
    for (uint c = 0u; c < circleSubdivisions; ++c) {
        float angle  = 2.0*PI*float(c)/float(circleSubdivisions);
        vec3 offset  = radius * (cos(angle)*normal + sin(angle)*bitan);
        verts[sampleIdx * circleSubdivisions + c] = vec4(center + offset, 1.0);
    }
}
```

Generate triangle indices (quad strips around the tube) on the CPU once, since the topology is static for a given number of segments and circle subdivisions.

### 6.4 Arc-Length Reparameterization

Uniform parameter t gives non-uniform spacing along a spline: the curve moves faster through near-linear sections and slower through tight bends. For constant-speed animation (camera paths, particle rails, conveyor belts), reparameterize by arc length s ∈ [0, S_total].

**Two-pass GPU approach:**

*Pass 1* — evaluate chord lengths at M fine samples and build a cumulative arc-length table via prefix sum.

```glsl
// arc_chord.comp — one thread per sample
layout(local_size_x = 256) in;
layout(set=0, binding=0) readonly  buffer CtrlPts  { vec3 ctrl[]; };
layout(set=0, binding=1) writeonly buffer ChordLens { float chords[]; };
layout(push_constant) uniform PC { uint numSamples; uint numSegments; };

vec3 catmull_rom(vec3 P[4], float t) {
    float t2 = t*t, t3 = t2*t;
    return 0.5*( (-t3+2*t2-t)*P[0] + (3*t3-5*t2+2)*P[1]
               + (-3*t3+4*t2+t)*P[2] + (t3-t2)*P[3] );
}

void main() {
    uint  idx  = gl_GlobalInvocationID.x;
    if (idx >= numSamples - 1u) return;
    float t0 = float(idx)   / float(numSamples - 1u);
    float t1 = float(idx+1u)/ float(numSamples - 1u);

    // Map global t to segment + local parameter
    float s0 = t0 * float(numSegments), s1 = t1 * float(numSegments);
    uint  seg0 = min(uint(s0), numSegments-1u);
    float tl0  = s0 - float(seg0);
    uint  seg1 = min(uint(s1), numSegments-1u);
    float tl1  = s1 - float(seg1);

    vec3 P0[4], P1[4];
    for (int k = 0; k < 4; ++k) {
        P0[k] = ctrl[clamp(int(seg0)+k-1, 0, int(numSegments))];
        P1[k] = ctrl[clamp(int(seg1)+k-1, 0, int(numSegments))];
    }
    chords[idx] = length(catmull_rom(P1, tl1) - catmull_rom(P0, tl0));
}
// After dispatch: run exclusive prefix sum over chords[] → cumArc[]
// cumArc[0] = 0, cumArc[M] = totalArcLength
```

*Pass 2* — binary-search the cumulative table to find uniform-distance sample positions:

```glsl
// arc_query.comp — one thread per animation frame
layout(set=0, binding=2) readonly  buffer CumArc   { float cumArc[]; };
layout(set=0, binding=3) writeonly buffer FramePos { vec4 framePos[]; };
layout(push_constant) uniform PC2 { uint numFrames; uint numSamples; float totalArc; };

void main() {
    uint  frame   = gl_GlobalInvocationID.x;
    if (frame >= numFrames) return;
    float targetS = (float(frame) / float(numFrames - 1u)) * totalArc;

    // Binary search for the interval [k, k+1] in cumArc[] containing targetS
    uint lo = 0u, hi = numSamples - 2u;
    while (lo < hi) {
        uint mid = (lo + hi) / 2u;
        if (cumArc[mid + 1u] <= targetS) lo = mid + 1u;
        else                             hi = mid;
    }
    float tLocal = float(lo) / float(numSamples - 1u);
    float span   = cumArc[lo+1u] - cumArc[lo];
    if (span > 1e-8)
        tLocal += (targetS - cumArc[lo]) / span / float(numSamples - 1u);

    // Evaluate spline at tLocal (same as §6.1 TES logic)
    // ...
    framePos[frame] = vec4(evaluatedPos, 1.0);
}
```

The cumulative arc-length table is computed once (or whenever control points change) and reused for all frame queries. For real-time camera paths, M = 1024 samples per segment gives sub-millimeter accuracy at typical scene scales.

---

## 7. GPU Mesh Simplification and LOD

GPU subdivision (§1) adds geometric detail; the inverse — removing detail for distant objects — is equally critical for real-time budgets. Two complementary approaches exist: quadric-error-metric (QEM) edge collapse for quality-preserving offline LOD generation, and vertex clustering for GPU-parallel online simplification.

### 7.1 Quadric Error Metric (Garland-Heckbert)

The QEM (Garland & Heckbert, 1997) assigns each vertex v a 4×4 error matrix Q = ΣᵢKᵢ where Kᵢ = ppᵀ for each adjacent face plane p = [nₓ, nᵧ, nᵤ, d]ᵀ (plane equation as a 4-vector). The cost of collapsing edge (v₁, v₂) to optimal position v̄:

```
Q_combined = Q₁ + Q₂
v̄ = argmin v̄ᵀ Q_combined v̄   (solve the 3×3 linear subsystem)
cost = v̄ᵀ Q_combined v̄
```

Collapse edges in order of cost using a min-heap; after each collapse, update the affected neighbor quadrics. The algorithm is inherently sequential due to topological dependencies between collapses.

### 7.2 meshoptimizer

The dominant production simplifier is `meshoptimizer` by Arseny Kapoulkine ([GitHub](https://github.com/zeux/meshoptimizer), MIT), used in Godot, Unreal, Bevy, and most modern game engines. It runs on the CPU and produces LOD index buffers ready for GPU upload:

```cpp
#include <meshoptimizer.h>

// Build LOD chain: target 50% triangles per level
size_t numIndices    = numTris * 3;
size_t targetCount   = numIndices / 2;
float  targetError   = 0.01f;   // relative QEF error bound
float  resultError   = 0.0f;

std::vector<uint32_t> lod(numIndices);
size_t lodCount = meshopt_simplify(
    lod.data(), indices, numIndices,
    (float*)positions, numVerts, sizeof(glm::vec3),
    targetCount, targetError,
    meshopt_SimplifyLockBorder,   // preserve mesh boundary edges
    &resultError);
lod.resize(lodCount);

// Optimize vertex cache and fetch efficiency for the LOD
meshopt_optimizeVertexCache(lod.data(), lod.data(), lodCount, numVerts);
meshopt_optimizeOverdraw(lod.data(), lod.data(), lodCount,
    (float*)positions, numVerts, sizeof(glm::vec3), /*threshold=*/1.05f);

// Upload LOD indices to a VkBuffer; store per-level offsets and counts
// for LOD selection at draw time
```

`meshopt_simplify` also supports attribute-weighted simplification (`meshopt_simplifyWithAttributes`) to preserve UV seams, normal discontinuities, and vertex colors during edge collapse.

**Vertex cache optimization — why it matters.** Modern GPUs have a post-transform vertex cache (PTVC) of 16–32 entries. If a vertex is transformed once and cached, a second triangle referencing it within the FIFO window pays no vertex-shader cost. A pathological index order (e.g., sequential insertion order from a DCC tool) may achieve ACMR (average cache miss ratio) above 1.5 — far from the theoretical minimum near 0.5 for a typical mesh. `meshopt_optimizeVertexCache` reorders indices using the Forsyth algorithm (linear scan with a per-vertex score function combining cache position and vertex valence), reducing ACMR to ~0.6 on dense triangle meshes.

```cpp
// Vertex cache optimization — reorders index buffer in-place
meshopt_optimizeVertexCache(indices, indices, indexCount, vertexCount);

// Measure ACMR before and after (cache_size = 16 simulates typical GPU PTVC)
meshopt_VertexCacheStatistics stats =
    meshopt_analyzeVertexCache(indices, indexCount, vertexCount,
                               /*cache_size=*/16, /*warp_size=*/0, /*prim_group_size=*/0);
// stats.acmr < 0.7 is good; < 0.55 is excellent
// stats.atvr (average transformed vertex ratio) should be close to 1.0
```

After vertex cache reordering, `meshopt_optimizeVertexFetch` remaps vertex data to match the new access pattern, improving L2 cache hit rate during vertex fetch:

```cpp
// Remap vertices to match the cache-optimized index order
std::vector<uint32_t> remap(vertexCount);
size_t uniqueVerts = meshopt_optimizeVertexFetchRemap(
    remap.data(), indices, indexCount, vertexCount);
meshopt_remapVertexBuffer(vertices, vertices, vertexCount,
                          sizeof(Vertex), remap.data());
meshopt_remapIndexBuffer(indices, indices, indexCount, remap.data());
```

For meshlet-based rendering (§7.4), meshoptimizer's `meshopt_buildMeshlets` already incorporates vertex cache awareness at the meshlet granularity: each meshlet's local index buffer is a tight 64-vertex / 124-triangle unit that fits entirely within the mesh shader's per-workgroup registers, making PTVC optimization less critical at draw time but more important within the meshlet build step.

**GPU porting status — simplification.** `meshopt_simplify` (QEM edge collapse) is CPU-only. GPU simplification is an active area: Unreal Engine 5's Nanite performs cluster-level hierarchical simplification on the GPU as part of its virtual geometry system, but this is not a direct port of QEM — it uses a custom cluster-DAG structure built offline. There is no production GPU port of the meshoptimizer simplification API. For real-time LOD generation (e.g., runtime terrain or streaming), GPU vertex clustering (§7.3) is the practical GPU alternative at the cost of lower quality than QEM.

**GPU porting status — encode/decode.** `meshopt_decodeIndexBuffer` and `meshopt_decodeVertexBuffer` are explicitly designed to be portable to GPU compute. The decode algorithms are tight loops with no irregular branching, and Kapoulkine has noted that a GLSL/HLSL compute shader decode pass is a stated roadmap goal. The encode step (delta coding, bit packing) remains CPU-only. A GPU decode pass would enable streaming compressed geometry directly from a staging buffer into a vertex/index buffer in a single compute dispatch, eliminating CPU decompression latency.

### 7.3 Vertex Clustering on GPU

Vertex clustering groups all vertices falling within a spatial cell and replaces each group with a representative (centroid or QEM-minimizer). Unlike QEM edge collapse, it requires no adjacency data — each vertex independently maps to its cell, making it GPU-friendly:

```glsl
// vertex_cluster.comp — assign each vertex to a grid cell
layout(local_size_x = 64) in;
layout(set=0, binding=0) readonly  buffer Positions { vec4 pos[]; };
layout(set=0, binding=1) writeonly buffer ClusterID { uint clusterID[]; };
layout(push_constant) uniform PC {
    vec3  gridMin;
    float cellSize;
    uvec3 gridDim;
};

void main() {
    uint  v    = gl_GlobalInvocationID.x;
    uvec3 cell = uvec3(clamp((pos[v].xyz - gridMin) / cellSize,
                              vec3(0), vec3(gridDim - uvec3(1))));
    clusterID[v] = cell.z * gridDim.y * gridDim.x
                 + cell.y * gridDim.x
                 + cell.x;
}
```

After clustering: radix-sort vertices by `clusterID`, compute per-cluster centroid via segmented reduction, and rebuild connectivity (triangles whose all three vertices map to the same cluster are degenerate and dropped). The resulting mesh has at most one vertex per grid cell. Cell size controls the LOD level — halving cell size doubles vertex count.

### 7.4 LOD Selection in Mesh Shaders

With `VK_EXT_mesh_shader` (Ch127), the amplification (task) shader can select per-meshlet LOD based on projected screen-space size, without any CPU round-trip:

```glsl
// task.glsl — amplification shader selects LOD per cluster
layout(local_size_x = 32) in;

struct TaskPayload { uint meshletIndices[32]; uint lodLevel[32]; };
taskPayloadSharedEXT TaskPayload payload;

void main() {
    uint clusterID = gl_WorkGroupID.x * 32u + gl_LocalInvocationID.x;

    // Project cluster bounding sphere to screen space
    vec4  center     = view_proj * vec4(clusterBounds[clusterID].center, 1.0);
    float screenDiam = clusterBounds[clusterID].radius / center.w * viewport.y;
    uint  lod        = uint(clamp(log2(max(screenDiam / lodPixelThreshold, 1.0)),
                                  0.0, float(MAX_LOD)));

    payload.meshletIndices[gl_LocalInvocationID.x] = clusterID;
    payload.lodLevel[gl_LocalInvocationID.x]       = lod;

    EmitMeshTasksEXT(lodMeshletCounts[clusterID][lod], 1u, 1u);
}
```

This pattern (per-cluster LOD in the amplification shader) is the basis of Unreal Engine's Nanite virtual geometry system and Mesa's experimental Vulkan mesh-shader LOD pass.

[Source: Garland & Heckbert (1997), "Surface Simplification Using Quadric Error Metrics", SIGGRAPH; https://github.com/zeux/meshoptimizer]

### 7.5 Smooth Normals Post-Pass

Meshes imported from CAD or sculpting tools may have per-face normals, producing faceted shading. Recomputing angle-weighted per-vertex smooth normals on the GPU requires atomic accumulation — each triangle contributes its face normal (scaled by vertex angle) to all three of its vertices' accumulators, then a normalize pass finalizes.

Direct float atomicAdd is available via `VK_EXT_shader_atomic_float`; without that extension, accumulate into scaled integers and divide in the normalize pass:

```glsl
// normal_accum.comp — one thread per triangle
layout(local_size_x=64) in;
layout(set=0, binding=0) readonly  buffer Positions { vec3 positions[]; };
layout(set=0, binding=1) readonly  buffer Indices   { uint indices[];   };
layout(set=0, binding=2) coherent  buffer NormAccum { int accum[];      };
// accum is 3 ints (x,y,z) per vertex, scaled by SCALE

const float SCALE = 1e6;

void main() {
    uint tri = gl_GlobalInvocationID.x;
    uint i0  = indices[tri*3+0], i1 = indices[tri*3+1], i2 = indices[tri*3+2];

    vec3 p0  = positions[i0], p1 = positions[i1], p2 = positions[i2];
    vec3 e01 = p1-p0, e02 = p2-p0;
    vec3 faceN = cross(e01, e02);   // magnitude = 2 * triangle area (area weighting)

    // Angle-weighted contribution: compute angle at each vertex
    vec3 an0 = faceN * acos(clamp(dot(normalize(e01), normalize(e02)), -1., 1.));
    vec3 an1 = faceN * acos(clamp(dot(normalize(p0-p1), normalize(p2-p1)), -1., 1.));
    vec3 an2 = faceN * acos(clamp(dot(normalize(p0-p2), normalize(p1-p2)), -1., 1.));

    atomicAdd(accum[i0*3+0], int(an0.x*SCALE));
    atomicAdd(accum[i0*3+1], int(an0.y*SCALE));
    atomicAdd(accum[i0*3+2], int(an0.z*SCALE));
    atomicAdd(accum[i1*3+0], int(an1.x*SCALE));
    atomicAdd(accum[i1*3+1], int(an1.y*SCALE));
    atomicAdd(accum[i1*3+2], int(an1.z*SCALE));
    atomicAdd(accum[i2*3+0], int(an2.x*SCALE));
    atomicAdd(accum[i2*3+1], int(an2.y*SCALE));
    atomicAdd(accum[i2*3+2], int(an2.z*SCALE));
}
```

```glsl
// normal_normalize.comp — one thread per vertex
layout(local_size_x=64) in;
layout(set=0, binding=0) readonly buffer NormAccum { int accum[]; };
layout(set=0, binding=1) writeonly buffer Normals  { vec4 normals[]; };

const float SCALE = 1e6;

void main() {
    uint i  = gl_GlobalInvocationID.x;
    vec3 n  = vec3(accum[i*3+0], accum[i*3+1], accum[i*3+2]) / SCALE;
    normals[i] = vec4(normalize(n), 0.0);
}
```

With `VK_EXT_shader_atomic_float`, replace the integer accum buffer with a `float` SSBO and use `atomicAdd` on `float` directly (declared with `layout(buffer_reference, std430)` or just standard SSBO binding). The integer-scaled approach runs on all Vulkan 1.2+ hardware without extensions.

### 7.6 Procedural GPU Instancing: Grass, Hair, Fur

Dense surface detail (grass fields, fur, hair cards) cannot be stored as conventional meshes — a 100m² grass field at 64 blades/m² is 6.4 million blades. Mesh shaders solve this by generating geometry on-chip from a compact procedural description, with the amplification (task) shader culling and LOD-selecting clusters before the mesh shader emits blade geometry.

**Amplification/Task shader (`grass_task.glsl`):**

```glsl
// grass_task.glsl  — EXT_mesh_shader task shader
layout(local_size_x=32) in;

layout(set=0, binding=0) uniform sampler2D densityMap;   // grass density 0..1
layout(set=0, binding=1) uniform sampler2D heightMap;
layout(push_constant) uniform PC {
    mat4  viewProj;
    vec3  cameraPos;
    float patchSize;    // world-space size of one cluster patch
    uint  patchesX;
    uint  patchesZ;
};

taskPayloadSharedEXT struct GrassPayload {
    uint patchIDs[32];
    uint count;
} payload;

void main() {
    uint id     = gl_GlobalInvocationID.x;
    uint pX     = id % patchesX, pZ = id / patchesX;
    vec2 uv     = vec2(pX, pZ) / vec2(patchesX, patchesZ);
    float dens  = texture(densityMap, uv).r;

    // Frustum + density cull
    vec3 patchCenter = vec3(pX * patchSize, 0, pZ * patchSize);
    patchCenter.y    = texture(heightMap, uv).r;
    float dist       = length(cameraPos - patchCenter);
    bool  visible    = (dens > 0.05) && (dist < 200.0);

    if (visible) {
        uint slot = atomicAdd(payload.count, 1u);
        if (slot < 32) payload.patchIDs[slot] = id;
    }

    barrier();
    if (gl_LocalInvocationID.x == 0)
        EmitMeshTasksEXT(payload.count, 1, 1);
}
```

**Mesh shader (`grass_mesh.glsl`)** emits one tapered grass blade (5 vertices, 3 triangles) per invocation:

```glsl
// grass_mesh.glsl  — EXT_mesh_shader
layout(local_size_x=32) in;
layout(triangles, max_vertices=160, max_primitives=96) out;  // 5 verts × 32 blades, 3 tris × 32

taskPayloadSharedEXT struct GrassPayload { uint patchIDs[32]; uint count; } payload;
layout(location=0) out vec3 fragNormal[];
layout(location=1) out vec2 fragUV[];

// Pseudo-random from uint seed
float rng(uint s) { s ^= s<<13; s ^= s>>7; s ^= s<<17; return float(s) / float(0xFFFFFFFFu); }

void main() {
    uint blade    = gl_LocalInvocationID.x;
    uint patchID  = payload.patchIDs[gl_WorkGroupID.x];
    // Derive blade root position from patchID + blade index
    uint seed     = patchID * 1237 + blade;
    float bx      = rng(seed)   * patchSize;
    float bz      = rng(seed+1) * patchSize;
    // ... (read height, compute world position, Hermite-style bend by wind)

    float height  = 0.4 + rng(seed+2) * 0.4;
    float lean    = rng(seed+3) * 0.3;             // lean angle
    float windSway = sin(time * 1.8 + bx * 0.5) * 0.12;

    // 5 control points: root → tip with Hermite bend
    vec3 root = worldBase + vec3(bx, 0, bz);
    vec3 ctrl = root + vec3(lean + windSway, height * 0.6, 0);
    vec3 tip  = root + vec3(lean * 2.0 + windSway * 1.5, height, 0);

    // Emit 5 vertices (3-segment blade: 4 quads collapsed at tip)
    uint vBase = blade * 5;
    // ... (set gl_MeshVerticesEXT[vBase..vBase+4].gl_Position)

    uint pBase = blade * 3;
    SetMeshOutputsEXT(160, 96);
    gl_PrimitiveTriangleIndicesEXT[pBase+0] = uvec3(vBase+0, vBase+2, vBase+1);
    gl_PrimitiveTriangleIndicesEXT[pBase+1] = uvec3(vBase+1, vBase+2, vBase+3);
    gl_PrimitiveTriangleIndicesEXT[pBase+2] = uvec3(vBase+2, vBase+4, vBase+3);
}
```

**LOD.** The task shader can switch to a billboard card (2 triangles per blade) beyond ~50m and suppress blades entirely past ~150m. For hair, replace the 5-vertex blade with a 10–16 vertex strand following a Catmull-Rom spline through simulation control points output by a PBD pass (§4.6). Fur uses a shorter, denser strand with a random length distribution driven by a density-map alpha channel.

[Source: Acton (2021) "Procedural Grass in 'Ghost of Tsushima'" GDC; Epic Games "UE5 Nanite Foliage" tech blog]

### 7.7 Billboard and Impostor LOD Atlases

At extreme camera distances (>200m for a tree, >50m for a shrub), even a single-lod low-poly mesh wastes vertex processing on subpixel geometry. **Impostors** replace the 3D mesh entirely with a camera-facing quad whose texture captures the mesh rendered from multiple discrete angles — a camera-view-dependent atlas lookup that is visually identical to the mesh at distances where the angular delta between atlas samples is below the pixel error threshold.

**Atlas baking pipeline (offline GPU pass):**

```glsl
// impostor_bake.comp — renders the mesh from N×N hemisphere directions
// into a texture atlas, one tile per direction
// Typical: 8×4=32 tiles, each 128×128px for small to medium foliage
layout(push_constant) uniform PC {
    mat4   model;
    uint   tileX, tileY;         // which tile to render (direction index)
    uint   atlasW, atlasH;       // full atlas dimensions
    uint   tileSize;             // pixels per tile
    float  hemiFov;              // in radians, 0..PI/2
};
// Render call: draw the full mesh to a sub-region of the atlas using
// an offset viewport/scissor for each (tileX, tileY)
// Store albedo+normal in RGBA8 GBuffer tiles; alpha = coverage mask
```

**Runtime billboard rendering.** The mesh shader amplification stage selects `IMPOSTOR_LOD` when the projected object diameter falls below a threshold:

```glsl
// In the amplification/task shader LOD decision:
float projectedDiam = objectRadius * 2.0 / (dist * tan(halfFov));
if (projectedDiam < IMPOSTOR_THRESHOLD) {
    // Encode view direction as atlas tile index
    vec3  viewDir = normalize(objectCenter - cameraPos);
    float azimuth = atan(viewDir.x, viewDir.z);          // -PI..PI
    float elevation = asin(clamp(viewDir.y, -1.0, 1.0)); // -PI/2..PI/2
    uint  tileX  = uint((azimuth   / (2.0*PI) + 0.5) * float(atlasGridX)) % atlasGridX;
    uint  tileY  = uint((elevation / (PI)      + 0.5) * float(atlasGridY)) % atlasGridY;
    payload.impostorTile = tileX + tileY * atlasGridX;
    EmitMeshTasksEXT(1, 1, 1);  // emit one impostor quad
} else {
    EmitMeshTasksEXT(meshletCount, 1, 1);
}
```

**Octahedral impostors** (Baker 2021) parameterize the hemisphere with an octahedral projection, achieving uniform angular sample density with fewer tiles than spherical grids. Each tile stores albedo, normal (in octahedral encoding), and a depth silhouette for parallax-corrected reprojection.

**Transition blending.** Cross-fade between mesh LOD and impostor by alpha-blending the two draw calls during a distance hysteresis band. Unreal Engine's `HISM` (Hierarchical Instanced Static Mesh) uses exactly this pattern for foliage at >200k instances. [Source: Baker "Octahedral Impostor Rendering", GPU Gems 3 Ch13; Klauder "Efficient Impostors in UE4", GDC 2019]

### 7.8 Bent Normals Precomputation

A **bent normal** at a surface point is the average unoccluded hemisphere direction — pointing toward the open sky rather than toward nearby occluders. When used to look up an irradiance probe or environment map, bent normals dramatically reduce light leaking under overhangs, inside concave surfaces, and near contact shadows, at zero fragment-shader cost beyond the texture lookup.

**Baking pass (`bent_normal_bake.comp`):** For each surface vertex, cast N rays in a cosine-weighted hemisphere and accumulate the directions of unoccluded rays. The BVH built in §8 provides the ray-cast primitive:

```glsl
// bent_normal_bake.comp — one thread per surface vertex
layout(local_size_x=64) in;

layout(set=0, binding=0) readonly  buffer Positions { vec4 positions[]; };
layout(set=0, binding=1) readonly  buffer Normals   { vec4 normals[];   };
layout(set=0, binding=2) readonly  buffer BVH       { /* LBVH nodes from §8 */ };
layout(set=0, binding=3) writeonly buffer BentNorms { vec4 bentNormals[]; };
layout(push_constant) uniform PC { uint vertCount; uint raysPerVertex; uint frameSeed; };

// Cosine-weighted hemisphere sample (Malley's method)
vec3 cosineSampleHemisphere(vec2 xi) {
    float r   = sqrt(xi.x);
    float phi = 2.0 * 3.14159265 * xi.y;
    vec2  d   = r * vec2(cos(phi), sin(phi));
    float z   = sqrt(max(0.0, 1.0 - dot(d, d)));
    return vec3(d.x, d.y, z);
}

void main() {
    uint  vi  = gl_GlobalInvocationID.x;
    if (vi >= vertCount) return;
    vec3  pos = positions[vi].xyz + normals[vi].xyz * 0.001;  // bias off surface
    vec3  N   = normals[vi].xyz;

    // Build TBN frame from N
    vec3 T  = abs(N.x) < 0.9 ? vec3(1,0,0) : vec3(0,1,0);
    vec3 B  = normalize(cross(N, T));
    T       = cross(B, N);

    vec3 bentSum = vec3(0.0);
    for (uint r = 0; r < raysPerVertex; ++r) {
        vec2 xi  = vec2(hashFloat(vi*raysPerVertex+r + frameSeed*65537u),
                        hashFloat(vi*raysPerVertex+r + frameSeed*131071u));
        vec3 ls  = cosineSampleHemisphere(xi);
        vec3 dir = T*ls.x + B*ls.y + N*ls.z;

        // BVH ray cast (shadow ray, any-hit)
        float tHit = bvh_ray_cast(pos, dir, 2.0);  // max AO distance = 2m
        if (tHit > 2.0) bentSum += dir;             // unoccluded
    }
    bentNormals[vi] = vec4(normalize(bentSum), 0.0);
}
```

For 64 rays per vertex on a 100k-vertex mesh, the bake dispatches ~6.4M ray tests — approximately 50–200 ms on a midrange GPU. Accumulate across multiple frames (each frame uses a different `frameSeed` for stratified sampling) and EMA-blend results for a progressive bake. Store the baked bent normal as an `SNORM16` vertex attribute or bake into a UV-mapped texture for static geometry.

**Runtime usage.** In the PBR fragment shader, replace the surface normal for the irradiance probe lookup:

```glsl
vec3 bentN   = normalize(texture(bentNormalMap, uv).xyz * 2.0 - 1.0);
vec3 irrDir  = TBN * bentN;   // transform from tangent to world space
vec3 irr     = texture(irradianceCube, irrDir).rgb;
vec3 diffuse = albedo * irr;
```

[Source: Landis "Production-Ready Global Illumination", SIGGRAPH 2002; UE4 "Ambient Occlusion and Bent Normal" documentation]

### 7.9 Geometry Compression and Quantization

Memory bandwidth is the dominant cost in geometry-heavy rendering. Compressing vertex attributes before upload to VRAM reduces bandwidth on every draw call and meshlet dispatch.

**Position quantization.** Store positions as `SNORM16` (3 × 16-bit signed normalized integers) relative to an object bounding-box:

```cpp
// CPU-side quantization: map [bbox.min, bbox.max] → [-32767, 32767]
glm::vec3 scale  = 2.0f / (bbox.max - bbox.min);
glm::vec3 offset = -(bbox.min + bbox.max) * 0.5f;
for (auto& v : vertices) {
    glm::vec3 q = (v.pos + offset) * scale;
    v.posQ[0] = int16_t(clamp(q.x, -1.f, 1.f) * 32767.f);
    v.posQ[1] = int16_t(clamp(q.y, -1.f, 1.f) * 32767.f);
    v.posQ[2] = int16_t(clamp(q.z, -1.f, 1.f) * 32767.f);
}
// In Vulkan: VK_FORMAT_R16G16B16_SNORM vertex attribute
// In vertex shader: pos = vec3(posQ) / 32767.0 * halfExtent + center;
```

This reduces position bandwidth from 12 bytes to 6 bytes per vertex (50% saving) with sub-millimetre error on a 10m bounding box.

**Normal quantization.** Octahedral encoding packs a unit normal into two `SNORM8` values with uniform angular error (< 0.1°):

```glsl
// Encode (in CPU preprocessing)
vec2 octEncode(vec3 n) {
    vec2 p = n.xy * (1.0 / (abs(n.x) + abs(n.y) + abs(n.z)));
    return (n.z <= 0.0) ? (1.0 - abs(p.yx)) * sign(p) : p;
}
// Decode (in vertex shader)
vec3 octDecode(vec2 e) {
    vec3 v = vec3(e.xy, 1.0 - abs(e.x) - abs(e.y));
    if (v.z < 0.0) v.xy = (1.0 - abs(v.yx)) * sign(v.xy);
    return normalize(v);
}
```

Store as `VK_FORMAT_R8G8_SNORM`. Combined with `VK_FORMAT_R16G16_SNORM` for UVs, a full vertex (position + normal + UV) compresses from 32 bytes to 12 bytes — a 62% bandwidth saving.

**Index buffer compression.** meshoptimizer's `meshopt_encodeIndexBuffer` / `meshopt_decodeIndexBuffer` compresses an index buffer to ~1.5 bits per index (vs. 4 bytes = 32 bits for uint32): a 20× reduction for streaming. At load time, the decode runs on CPU in ~0.1 ns per index, and for GPU-resident streaming, a compute shader can decode a compressed chunk directly into a VkBuffer:

```cpp
// Encode at asset build time
std::vector<uint8_t> compressed(meshopt_encodeIndexBufferBound(indexCount, vertexCount));
size_t compSize = meshopt_encodeIndexBuffer(
    compressed.data(), compressed.size(),
    indices, indexCount);

// Decode at runtime (CPU)
meshopt_decodeIndexBuffer(indices, indexCount, sizeof(uint32_t),
    compressed.data(), compSize);
```

**Vertex buffer compression.** `meshopt_encodeVertexBuffer` / `meshopt_decodeVertexBuffer` compress arbitrary vertex streams using delta coding and bit-packing, achieving ~3–5× compression on typical meshes. Combined with GPU-side `meshopt_decodeVertexBuffer` (C code compilable to a compute shader with minor adaptation), full mesh streaming from network/disk to GPU can proceed without intermediate CPU decompression.

[Source: Kapoulkine "meshoptimizer: Mesh Optimization Library", GitHub README; Meyer et al. "Octahedral Normal Vector Encoding", ShaderX3]

### 7.10 Mesh Stitching and Seam Welding

DCC tools and glTF exporters produce meshes with split vertices at UV seams, hard-normal boundaries, and part joints — positions that are geometrically coincident but stored as separate vertices with different attributes. This produces visual cracks in subdivision (§1), incorrect smooth normals (§7.5), and doubles the vertex count along every seam. GPU-side stitching merges these before any geometry processing.

**Vertex deduplication with `meshopt_generateVertexRemap`.** meshoptimizer's remap pass compares vertices by a caller-supplied equality predicate and produces a remap table:

```cpp
// Define per-vertex equality (merge on position only, ignore UV/normal)
struct PositionVertex { float x, y, z; };

std::vector<uint32_t> remap(vertexCount);
size_t uniqueCount = meshopt_generateVertexRemap(
    remap.data(),
    nullptr, vertexCount,              // no index buffer: dense input
    positionData, vertexCount, sizeof(PositionVertex));

// Apply remap to all streams: positions, normals, UVs, joint indices, etc.
meshopt_remapVertexBuffer(outPos,   inPos,   vertexCount, sizeof(PositionVertex), remap.data());
meshopt_remapVertexBuffer(outNorm,  inNorm,  vertexCount, sizeof(glm::vec3),      remap.data());
meshopt_remapVertexBuffer(outUV,    inUV,    vertexCount, sizeof(glm::vec2),      remap.data());
meshopt_remapIndexBuffer( outIdx,   inIdx,   indexCount,  remap.data());

// Result: uniqueCount vertices, seams closed for position-only identity
```

For tolerance-based welding (merge vertices within ε of each other), sort vertices by a spatial hash bucket, then merge within each bucket — a two-pass GPU pipeline:

```glsl
// weld_hash.comp — one thread per vertex: assign spatial hash bucket
layout(local_size_x=64) in;
layout(set=0,binding=0) readonly  buffer Positions{ vec3 pos[]; };
layout(set=0,binding=1) writeonly buffer Buckets  { uint bucket[]; };
layout(push_constant) uniform PC { float cellSize; uint vertCount; };

uint spatialHash(ivec3 c) { return uint(c.x*73856093 ^ c.y*19349663 ^ c.z*83492791); }

void main() {
    uint i  = gl_GlobalInvocationID.x;
    if (i >= vertCount) return;
    ivec3 c = ivec3(floor(pos[i] / cellSize));
    bucket[i] = spatialHash(c);
}
```

After GPU sort-by-bucket, a merge pass within each bucket identifies canonical representatives and writes the remap array, which then feeds `meshopt_remapIndexBuffer` or a custom index rewrite.

**Seam handling for UV/normal attributes.** Welding on position alone discards UV and normal discontinuities needed for correct texturing. The standard approach: weld to a per-vertex record `{position, roundedNormal, roundedUV}` where the normal and UV are rounded to a tolerance that preserves hard edges (angle > 30°) while merging smooth neighbours. `meshopt_generateVertexRemapMulti` accepts multiple streams and per-stream equality tolerances. [Source: meshoptimizer `generateVertexRemap` API documentation; Turk "Re-Tiling Polygonal Surfaces", SIGGRAPH 1992]

### 7.11 Mesh Boolean Operations: Polygon Clipping on GPU

SDF Boolean operations (§3.7) approximate the result via a distance field. **Polygon clipping** produces exact Boolean operations on triangle meshes by clipping each triangle of mesh A against each triangle of mesh B at their intersection contour, then classifying and keeping the desired inside/outside fragments.

**Sutherland-Hodgman polygon clipping on GPU.** For each candidate pair of triangles (A_i, B_j) from the BVH broad-phase (§8.1–8.3), clip triangle A_i against the plane of B_j:

```glsl
// clip_tri_against_plane.glsl — called per candidate pair
// Returns clipped polygon (0–6 vertices) into shared memory output

int clipPolygon(vec3 poly[], int n, vec3 planeN, float planeD, vec3 out[]) {
    int outN = 0;
    for (int i = 0; i < n; ++i) {
        vec3 a = poly[i], b = poly[(i+1)%n];
        float da = dot(a, planeN) - planeD;
        float db = dot(b, planeN) - planeD;
        if (da >= 0.0) out[outN++] = a;              // a inside
        if ((da > 0.0) != (db > 0.0)) {              // edge crosses
            float t = da / (da - db);
            out[outN++] = mix(a, b, t);              // intersection point
        }
    }
    return outN;
}

// Boolean union: keep polys outside B, and B polys outside A, plus boundary
// Boolean subtract: keep A polys outside B, inverted B polys inside A
// Boolean intersect: keep only polys inside both A and B
```

**GPU pipeline for mesh Boolean:**

1. **BVH broad-phase** — find all (A_i, B_j) AABB pairs that overlap (§8.1–8.3)
2. **Triangle-triangle intersection** (compute) — for each pair, compute the intersection line segment; discard non-intersecting pairs
3. **Sutherland-Hodgman clipping** (compute) — clip each intersecting triangle against the other's plane; emit sub-triangles into output buffer
4. **Inside/outside classification** — ray-cast from each sub-triangle centroid to classify interior/exterior using the opposite mesh's BVH
5. **Triangle soup assembly** — merge kept sub-triangles from both meshes into output index buffer

Full mesh Boolean on GPU is an active research area. Production tools (Blender's `bmesh`, OpenCASCADE `BRepAlgoAPI`) run on CPU, but GPU broad-phase acceleration can reduce step 1–2 from minutes to seconds for complex meshes. The GPU pipeline above handles the intersection kernel; CPU coordinates the global classification. [Source: Greiner & Hormann "Efficient Clipping of Arbitrary Polygons", TOG 1998; Bernstein & Fussell "Fast, Exact, Linear Booleans", SGP 2009]

### 7.12 Harmonic and LSCM UV Parameterization

UV parameterization maps a 3D mesh surface to a 2D texture domain with minimal distortion. Two methods dominate production use: **Harmonic maps** (minimize Dirichlet energy, angle-preserving near boundaries) and **Least Squares Conformal Maps** (LSCM, Lévy 2002 — minimizes angle distortion globally, requires only 2 pinned vertices).

**Laplacian system setup.** Both methods solve a sparse linear system `L · u = b` where `L` is the cotangent-weight Laplacian. Each interior vertex `i` with neighbours `j₁..jₙ` contributes:

```
L[i,i] = Σⱼ cot(αᵢⱼ) + cot(βᵢⱼ)    (sum of cotangent weights)
L[i,j] = -(cot(αᵢⱼ) + cot(βᵢⱼ))    (negative weight for each neighbour)
```

where αᵢⱼ and βᵢⱼ are the angles opposite edge (i,j) in the two adjacent triangles. The cotangent weights ensure the solution is angle-preserving at interior vertices.

**GPU Laplacian assembly (`uv_laplacian.comp`):**

```glsl
// uv_laplacian.comp — one thread per triangle, atomic scatter into sparse L
layout(local_size_x=64) in;
layout(set=0,binding=0) readonly  buffer Positions{ vec3 pos[]; };
layout(set=0,binding=1) readonly  buffer Indices  { uvec3 tris[]; };
layout(set=0,binding=2) coherent  buffer LValues  { float lval[]; }; // CSR values
layout(set=0,binding=3) readonly  buffer LOffsets { uint loff[]; };  // CSR row offsets

void main() {
    uint ti  = gl_GlobalInvocationID.x;
    uvec3 t  = tris[ti];
    vec3  p0=pos[t.x], p1=pos[t.y], p2=pos[t.z];

    // Cotangent weights for the three edges
    float w01 = 0.5 * dot(p2-p0, p2-p1) / length(cross(p2-p0, p2-p1));  // cot at p2
    float w12 = 0.5 * dot(p0-p1, p0-p2) / length(cross(p0-p1, p0-p2));  // cot at p0
    float w20 = 0.5 * dot(p1-p2, p1-p0) / length(cross(p1-p2, p1-p0));  // cot at p1

    // Scatter-add into CSR (each thread needs to find the CSR column entries)
    // atomicAdd into lval at the pre-computed CSR positions for (t.x,t.y), etc.
    // (CSR structure pre-built from mesh topology on CPU)
    atomicAdd(lval[findEntry(loff, t.x, t.y)], -w01);
    atomicAdd(lval[findEntry(loff, t.y, t.x)], -w01);
    atomicAdd(lval[findEntry(loff, t.x, t.x)],  w01);
    atomicAdd(lval[findEntry(loff, t.y, t.y)],  w01);
    // ... repeat for (t.y,t.z) with w12 and (t.z,t.x) with w20
}
```

**Solving on GPU.** The assembled `L` is a sparse symmetric positive semi-definite matrix. Solve with Conjugate Gradient:

```glsl
// cg_step.comp — one CG iteration (sparse matrix-vector product + update)
// x: current UV solution, r: residual, p: search direction, Ap: L*p
// Standard CG: α = (r·r)/(p·Ap), x += α*p, r -= α*Ap, β = (r·r)/rr_old, p = r + β*p
```

For LSCM specifically, pin two boundary vertices (e.g., the two vertices of the longest boundary edge) and solve the 2×2 block system simultaneously for u and v coordinates. LSCM minimizes the complex Dirichlet energy `‖∂f/∂z̄‖²` which is equivalent to conformal (angle-preserving) mapping.

**Practical output.** For a 100k-triangle mesh, GPU Laplacian assembly takes ~5 ms; CG convergence (50–200 iterations) takes ~10–50 ms. The result feeds directly into the baking pipeline (§7.13). Production tools: xatlas (MIT, CPU-only) for automatic atlas layout with seam optimization; libigl `igl::lscm` + `igl::harmonic` (CPU); uvatlas (DirectX SDK, CPU). [Source: Lévy et al. "Least Squares Conformal Maps for Automatic Texture Atlas Generation", SIGGRAPH 2002; Desbrun et al. "Intrinsic Parameterizations of Surface Meshes", Eurographics 2002]

### 7.13 GPU Parametric Texture Baking

Texture baking transfers high-frequency surface detail from a high-poly mesh to a low-poly mesh via UV projection. The GPU pipeline renders the high-poly mesh once per baked channel (normal, AO, curvature, thickness, albedo) using the low-poly UV layout as the render target coordinate system.

**Cage projection pipeline.** For each texel in the low-poly UV atlas:
1. Compute the 3D position on the low-poly surface corresponding to this texel's UV
2. Cast a ray from this position along the low-poly surface normal (or inward-offset cage normal)
3. Find the first intersection with the high-poly mesh
4. Sample whatever attribute is being baked at that intersection point

```glsl
// bake_normal.comp — one thread per texel in the UV atlas
layout(local_size_x=16, local_size_y=16) in;
layout(set=0,binding=0, rgba8snorm) uniform image2D normalMap;   // output
layout(set=0,binding=1) readonly buffer LowPolyPos { vec3 lpPos[]; };
layout(set=0,binding=2) readonly buffer LowPolyUV  { vec2 lpUV[];  };
layout(set=0,binding=3) readonly buffer LowPolyIdx { uvec3 lpTri[];};
layout(set=0,binding=4) readonly buffer HiPolyBVH  { /* LBVH §8 */ };
layout(set=0,binding=5) readonly buffer HiPolyNorm { vec3 hpNorm[];};
layout(push_constant) uniform PC { ivec2 atlasSize; uint triCount; float cageOffset; };

void main() {
    ivec2 texel  = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv     = (vec2(texel) + 0.5) / vec2(atlasSize);

    // Find which low-poly triangle owns this texel (UV rasterization)
    // (pre-rasterized coverage map can accelerate this)
    uint  triID  = findTriangleForUV(uv, lpUV, lpTri, triCount);
    if (triID == 0xFFFFFFFF) { imageStore(normalMap, texel, vec4(0)); return; }

    // Compute 3D position + normal on the low-poly surface at this UV
    uvec3 t      = lpTri[triID];
    vec3  bary   = uvToBarycentric(uv, lpUV[t.x], lpUV[t.y], lpUV[t.z]);
    vec3  surfPos= bary.x*lpPos[t.x] + bary.y*lpPos[t.y] + bary.z*lpPos[t.z];
    vec3  surfN  = ...; // interpolated low-poly normal

    // Ray-cast against high-poly BVH from cage offset position
    vec3  rayO   = surfPos + surfN * cageOffset;   // start outside cage
    vec3  rayD   = -surfN;                         // shoot inward
    float tHit;  uint hitTri;
    if (!bvh_intersect(rayO, rayD, HiPolyBVH, tHit, hitTri)) {
        imageStore(normalMap, texel, vec4(0,0,1,0)); // fallback: surface normal
        return;
    }

    // Sample high-poly normal at hit point, transform to tangent space
    vec3  hiN    = interpolateNormal(hitTri, rayO + tHit*rayD, HiPolyNorm);
    vec3  tsN    = worldToTangentSpace(hiN, surfN, surfPos); // TBN matrix
    imageStore(normalMap, texel, vec4(tsN * 0.5 + 0.5, 1.0));
}
```

**Baked channels.** The same pipeline handles:
- **Normal map** — high-poly surface normal in low-poly tangent space
- **AO / bent normals** — §7.8 bent normal bake using BVH ray casting
- **Curvature** — mean curvature `H = ½ tr(shape operator)` at the hit point; sign indicates convex/concave; used for edge highlights and cavity shading
- **Thickness** — bidirectional ray: shoot forward and backward along normal, record total distance inside the mesh; used for subsurface scattering approximation
- **Albedo transfer** — sample the high-poly diffuse texture at the hit UV

**Atlas dilation.** After baking, texels at UV island boundaries have no valid low-poly coverage. A post-pass dilates island pixels into the gutter (2–4 texels outward) by JFA (§3.4) or a simple push-fill pass, preventing seam artifacts from bilinear filtering:

```glsl
// dilate.comp — flood valid texels outward into empty gutter regions
layout(local_size_x=16, local_size_y=16) in;
layout(set=0,binding=0) uniform sampler2D input;   // baked atlas with holes
layout(set=0,binding=1, rgba8) uniform image2D output;
void main() {
    ivec2 p = ivec2(gl_GlobalInvocationID.xy);
    vec4  c = texelFetch(input, p, 0);
    if (c.a > 0.5) { imageStore(output, p, c); return; }  // already filled
    // Sample 4 neighbours; fill from first valid one
    for each neighbour n: if texelFetch(input, n).a > 0.5: imageStore(output, p, texelFetch(n)); return;
}
```

Production bakers: xNormal (GPU-accelerated), Marmoset Toolbag (GPU), Blender Cycles bake (GPU compute via HLBVH). xatlas handles the UV layout step preceding bake. [Source: Cignoni et al. "Metro: Measuring Error on Simplified Surfaces", Eurographics 1998; xNormal v3 GPU baking whitepaper]

### 7.14 GPU Mesh Repair

Meshes from external sources — CAD exports, photogrammetry reconstruction, physics simulation output, format converters — routinely contain: degenerate triangles (zero-area, collinear vertices), T-junctions (a vertex of one triangle lying on the edge of another without a shared vertex), non-manifold edges (three or more triangles sharing one edge), and holes (boundary edge loops). GPU compute can detect and partially repair these at asset-load time.

**Degenerate triangle removal.** A triangle is degenerate if its area falls below a threshold. Classify and compact in a two-pass GPU pipeline:

```glsl
// degen_classify.comp — one thread per triangle
layout(local_size_x=64) in;
layout(set=0,binding=0) readonly  buffer Positions{ vec3 pos[];   };
layout(set=0,binding=1) readonly  buffer Indices  { uvec3 tris[]; };
layout(set=0,binding=2) writeonly buffer Keep     { uint keep[];  };  // 1=valid, 0=degen
layout(push_constant) uniform PC { uint triCount; float areaEps; };

void main() {
    uint ti   = gl_GlobalInvocationID.x;
    if (ti >= triCount) return;
    uvec3 t   = tris[ti];
    vec3  e01 = pos[t.y] - pos[t.x];
    vec3  e02 = pos[t.z] - pos[t.x];
    float area = 0.5 * length(cross(e01, e02));
    keep[ti]  = (area > areaEps) ? 1u : 0u;
}
```

Prefix-scan the `keep` array, then scatter surviving triangles into a compacted index buffer — the standard two-pass GPU stream-compaction pattern used in Marching Cubes (§3.2).

**T-junction detection.** A T-junction exists when a vertex `v` lies on an edge `(a, b)` of another triangle (within tolerance ε) but there is no triangle that shares the pair `(a, v)` or `(v, b)`. Detect via a GPU hash: for each edge, insert into a hash map keyed by the two vertex indices; for each vertex, check if it lies on any stored edge:

```glsl
// tjunction_detect.comp — one thread per vertex
// For each vertex, test against nearby edges from the BVH edge structure
// If dot(v - a, b - a) in [eps, len-eps] and dist(v, line(a,b)) < eps: T-junction
float t   = dot(v - a, b - a) / dot(b - a, b - a);
float dist = length(v - (a + clamp(t,0.,1.)*(b-a)));
bool isTJ = (t > EPS && t < 1.0-EPS) && (dist < WELD_EPS);
```

T-junction repair inserts the T-vertex into the edge by splitting the adjacent triangle `(a, b, c)` into two triangles `(a, v, c)` and `(v, b, c)`. This is straightforward when done per-T-junction but requires a variable-length output (number of new triangles = number of T-junctions per edge); use `atomicAdd` into an output triangle list.

**Non-manifold edge detection.** An edge is non-manifold if it is shared by ≠ 2 triangles (boundary = 1 triangle, manifold interior = 2 triangles, non-manifold = 3+). Count edge-to-triangle incidences via a GPU hash map or by sorting edges:

```glsl
// edge_valence.comp — count triangle-per-edge incidence
// Each thread processes one triangle, emits 3 half-edges sorted by (min,max) vertex
// After sort+groupBy, count group sizes: 1=boundary, 2=manifold, 3+=non-manifold
uint eKey = min(t.x,t.y)*maxV + max(t.x,t.y);   // canonical undirected edge key
atomicAdd(edgeCount[eKey], 1u);
```

Non-manifold edges require human-guided repair (split vertex, fill hole, or delete); the GPU pass only detects and emits a list.

**Hole filling (advancing front).** For each boundary loop (a cycle of boundary edges), an advancing-front pass fills the hole by triangulating inward. A simple fan triangulation from the loop centroid works for convex holes; for concave holes, use Ear Clipping on the boundary polygon, parallelized across multiple holes:

```glsl
// hole_fill.comp — one workgroup per boundary loop
// Ear clipping: repeatedly remove the vertex whose triangle has smallest area
// and is entirely inside the polygon (no other boundary vertex inside)
// Emit one triangle per removed ear into output triangle buffer
```

Production mesh repair tools: MeshFix (Attene 2010, C++), Open3D `mesh.remove_degenerate_triangles()` (CPU), Blender's `Clean Up` operators. The GPU pipeline above handles the detection and simple repairs; complex topology repair (genus changes, overlapping shells) remains a CPU task. [Source: Attene "A lightweight approach to repairing digitized polygon meshes", VC 2010; Ju "Fixing Geometric Errors on Polygonal Models", JCAM 2009]

### 7.15 Sphere Packing and Farthest-Point Sampling

**Farthest-point sampling (FPS)** selects a subset of N points from a larger set M such that each selected point is as far as possible from all previously selected points. It produces a well-distributed, coverage-maximizing sample set — ideal for: light probe placement (maximize coverage of the scene), LOD seed selection (§7.1 QEM clustering), impostor view-direction sampling (§7.7), and point-cloud downsampling before Poisson reconstruction (§3.12).

**Sequential FPS is inherently serial** (each new point depends on all previously selected). The GPU-parallel approximation maintains a per-point `minDist` buffer — the distance from each unselected point to its nearest already-selected point — and performs a reduce-max to find the next sample:

```glsl
// fps_update.comp — after selecting point s, update minDist for all remaining points
layout(local_size_x=64) in;
layout(set=0,binding=0) readonly  buffer Points  { vec3 pts[];     };
layout(set=0,binding=1) buffer    MinDist        { float minD[];   };  // +inf initially
layout(set=0,binding=2) readonly  buffer Selected{ uint selected;  };  // index of s
layout(push_constant) uniform PC { uint ptCount; };

void main() {
    uint i  = gl_GlobalInvocationID.x;
    if (i >= ptCount) return;
    vec3  s = pts[selected];
    float d = length(pts[i] - s);
    if (d < minD[i]) minD[i] = d;
}
```

Then a parallel reduce-max over `minD` (using the prefix-scan pattern from §3.2) finds the next farthest point. For M=100k points and N=1k samples, the N × O(M/W) GPU work equals ~1.6M threads total, completing in ~5 ms on a midrange GPU — 50× faster than CPU FPS on the same data.

**Poisson-disk sampling.** For uniform surface coverage with a guaranteed minimum inter-sample distance `r` (Poisson-disk property), use a GPU dart-throwing approach:

```glsl
// poisson_disk.comp — parallel dart throwing with spatial hash rejection
layout(local_size_x=64) in;
layout(set=0,binding=0) readonly  buffer Candidates{ vec3 cands[];   };
layout(set=0,binding=1) coherent  buffer Accepted  { vec3 accepted[];};
layout(set=0,binding=2) buffer    Counter          { uint count;     };
layout(set=0,binding=3) readonly  buffer SpatialHash{ uint hashGrid[];};
layout(push_constant) uniform PC { float minDist; uint candCount; };

void main() {
    uint ci   = gl_GlobalInvocationID.x;
    if (ci >= candCount) return;
    vec3  c   = cands[ci];

    // Check spatial hash for any accepted point within minDist
    ivec3 cell = ivec3(floor(c / minDist));
    bool  ok   = true;
    for (int dz=-2; dz<=2 && ok; ++dz)
    for (int dy=-2; dy<=2 && ok; ++dy)
    for (int dx=-2; dx<=2 && ok; ++dx) {
        uint h = hashCell(cell + ivec3(dx,dy,dz));
        // Check all points in that hash bucket
        for (uint k=hashStart[h]; k<hashEnd[h]; ++k)
            if (length(accepted[k] - c) < minDist) { ok=false; break; }
    }
    if (ok) {
        uint slot = atomicAdd(count, 1u);
        accepted[slot] = c;
        // Update spatial hash (requires a separate synchronization pass)
    }
}
```

Parallel dart throwing has race conditions (two threads may both accept points that are too close). The mitigation is to process candidates in waves with a barrier between each wave, or to use a conflict-detection post-pass that removes any accepted pair with `dist < minDist`.

**Sphere packing.** For 3D space-filling sphere placement (used for light probe grids and particle initial conditions), maintain a set of non-overlapping spheres of radius `r`. The GPU Poisson-disk pass generalizes directly to 3D: replace surface sampling with volumetric candidate generation (random points inside the volume) and apply the same spatial-hash rejection.

**Applications summary:**

| Use case | Algorithm | Typical N | GPU time |
|---|---|---|---|
| Light probe placement (scene) | FPS on surface samples | 64–256 | <1 ms |
| QEM cluster seeds | Poisson-disk on vertices | 1k–10k | ~5 ms |
| Impostor view directions | FPS on hemisphere | 16–64 | <0.1 ms |
| Point cloud downsampling | FPS before PoissonRecon | 10k–100k | 5–20 ms |
| SPH initial conditions | Poisson-disk 3D | 10k–100k | 10–50 ms |

[Source: Eldar et al. "The Farthest Point Strategy for Progressive Image Sampling", IEEE IP 1997; Bridson "Fast Poisson Disk Sampling in Arbitrary Dimensions", SIGGRAPH 2007]

---

## 8. GPU BVH Construction

A Bounding Volume Hierarchy (BVH) is the spatial acceleration structure underlying Vulkan ray tracing (Ch135), GPU collision detection, and ray-cast inverse kinematics. For dynamic geometry (skinned characters, deforming cloth), rebuilding the BVH every frame on the GPU eliminates the CPU bottleneck.

### 8.1 Morton Codes and LBVH

The Linear BVH (LBVH) algorithm (Lauterbach et al., 2009; Karras, 2012) builds a complete BVH in O(n log n) via three stages: (1) map primitive centroids to 30-bit Morton codes, (2) GPU radix sort by code, (3) build binary tree from the sorted code sequence using longest-common-prefix analysis.

**Morton code computation (30 bits, 10 per axis):**

```glsl
// morton_codes.comp
layout(local_size_x = 64) in;
layout(set=0, binding=0) readonly  buffer PrimCentroids { vec4 centroids[]; };
layout(set=0, binding=1) writeonly buffer MortonCodes   { uint codes[]; };
layout(push_constant) uniform PC { vec3 sceneMin; float invSceneExtent; uint numPrims; };

// Spread 10-bit value into 30 bits by inserting two zeros after each bit
uint expandBits(uint v) {
    v = (v * 0x00010001u) & 0xFF0000FFu;
    v = (v * 0x00000101u) & 0x0F00F00Fu;
    v = (v * 0x00000011u) & 0xC30C30C3u;
    v = (v * 0x00000005u) & 0x49249249u;
    return v;
}

uint morton3(vec3 p) {
    uint x = uint(clamp(p.x * 1024.0, 0.0, 1023.0));
    uint y = uint(clamp(p.y * 1024.0, 0.0, 1023.0));
    uint z = uint(clamp(p.z * 1024.0, 0.0, 1023.0));
    return expandBits(x) | (expandBits(y) << 1u) | (expandBits(z) << 2u);
}

void main() {
    uint i = gl_GlobalInvocationID.x;
    if (i >= numPrims) return;
    vec3 norm = (centroids[i].xyz - sceneMin) * invSceneExtent;
    codes[i]  = morton3(norm);
}
```

After this pass, run a GPU radix sort (e.g., the 4-pass 8-bit-per-pass radix sort from [GPUSorting](https://github.com/b0nes164/GPUSorting)) to produce a sorted index array. The radix sort is the dominant cost: O(n) passes of 256-bin histogram + scatter.

### 8.2 LBVH Tree Construction (Karras 2012)

Karras's insight: given n primitives sorted by Morton code, the binary tree structure is fully determined by the position of the most-significant differing bit between adjacent codes. Each internal node i (there are n−1 of them) covers a range [first_i, last_i] in the sorted array, found via the `clz` (count-leading-zeros) of adjacent code XORs:

```glsl
// lbvh_build.comp — one thread per internal node (indices 0..n-2)
layout(local_size_x = 64) in;
layout(set=0, binding=0) readonly  buffer SortedCodes { uint codes[]; };
layout(set=0, binding=1) writeonly buffer InternalNodes { BVHNode nodes[]; };
layout(push_constant) uniform PC { uint n; };  // n = number of leaf primitives

int delta(int i, int j) {
    if (j < 0 || uint(j) >= n) return -1;
    if (codes[i] == codes[j])
        // Tie-break with index XOR (guarantees unique values)
        return 32 + int(findMSB(uint(i) ^ uint(j)));
    return int(findMSB(uint(0u))) - int(findMSB(codes[i] ^ codes[j]));
}

void main() {
    int i = int(gl_GlobalInvocationID.x);
    if (i >= int(n) - 1) return;

    // Direction: sign of (delta(i,i+1) - delta(i,i-1))
    int d = (delta(i, i+1) > delta(i, i-1)) ? 1 : -1;
    int dMin = delta(i, i - d);

    // Find upper bound for node range (binary search by doubling)
    int lMax = 2;
    while (delta(i, i + d*lMax) > dMin) lMax *= 2;

    int l = 0;
    for (int t = lMax/2; t >= 1; t /= 2)
        if (delta(i, i + d*(l+t)) > dMin) l += t;
    int j = i + d*l;

    // Find split position (gamma) within [min(i,j), max(i,j)]
    int dNode = delta(i, j);
    int s = 0;
    for (int t = max((l+1)/2, 1); ; t = max(t/2, 1)) {
        if (delta(i, i + d*(s+t)) > dNode) s += t;
        if (t == 1) break;
    }
    int gamma = i + d*s + min(d, 0);

    // Assign children: leaf nodes are at indices [n-1 .. 2n-2]
    bool leftIsLeaf  = (min(i,j) == gamma);
    bool rightIsLeaf = (max(i,j) == gamma + 1);
    nodes[i].leftChild  = leftIsLeaf  ? (int(n) - 1 + gamma)     : gamma;
    nodes[i].rightChild = rightIsLeaf ? (int(n) - 1 + gamma + 1) : gamma + 1;
    nodes[i].isLeaf     = 0u;
    // Set parent index for children (needed for bottom-up AABB propagation)
    nodes[nodes[i].leftChild ].parentIdx = uint(i);
    nodes[nodes[i].rightChild].parentIdx = uint(i);
}
```

### 8.3 AABB Propagation

After the tree topology is built, propagate bounding boxes from leaves (initialized from primitive AABBs) up to the root. Use an atomic counter per node: each node's second arriving child computes the merged AABB and continues upward:

```glsl
// aabb_propagate.comp — one thread per leaf primitive
layout(local_size_x = 64) in;
layout(set=0, binding=1) buffer   InternalNodes { BVHNode nodes[]; };  // r/w
layout(set=0, binding=2) readonly buffer PrimAABBs { vec4 aabbMin[]; vec4 aabbMax[]; };
layout(set=0, binding=3) buffer   AtomicCounts   { uint childCount[]; };
layout(push_constant) uniform PC { uint n; };

void main() {
    uint leafIdx  = gl_GlobalInvocationID.x;
    if (leafIdx >= n) return;
    uint nodeIdx  = n - 1u + leafIdx;  // leaf node index in flat array

    // Initialize leaf AABB from primitive data
    nodes[nodeIdx].aabbMin = aabbMin[leafIdx];
    nodes[nodeIdx].aabbMax = aabbMax[leafIdx];

    uint parent = nodes[nodeIdx].parentIdx;
    while (parent != 0xFFFFFFFFu) {
        // First child to arrive just increments the counter and exits
        if (atomicAdd(childCount[parent], 1u) == 0u) return;

        // Second child merges AABBs from both children
        uint lc = nodes[parent].leftChild;
        uint rc = nodes[parent].rightChild;
        nodes[parent].aabbMin = min(nodes[lc].aabbMin, nodes[rc].aabbMin);
        nodes[parent].aabbMax = max(nodes[lc].aabbMax, nodes[rc].aabbMax);
        parent = nodes[parent].parentIdx;
    }
}
```

The `atomicAdd` introduces a data-dependent execution path, but each node is visited at most twice (once per child), so total work is O(n) with no warp divergence bottleneck beyond the first barrier.

**Applications.** The GPU-built LBVH feeds directly into `VkAccelerationStructureBuildGeometryInfoKHR` for Vulkan ray tracing (Ch135) via `VK_ACCELERATION_STRUCTURE_BUILD_MODE_UPDATE_KHR` — the LBVH topology is provided as a host-side hint and the driver refines it. The same structure serves as a broad-phase collision structure for compute-side ragdoll and IK ray-casting (§5) without leaving the GPU.

[Source: Lauterbach et al. (2009), "Fast BVH Construction on GPUs", Eurographics; Karras (2012), "Maximizing Parallelism in the Construction of BVHs, Octrees, and k-d Trees", HPG]

### 8.4 GPU Narrow-Phase Collision: SAT and GJK

The BVH (§8.1–8.3) is a broad-phase structure: it quickly eliminates non-overlapping bounding box pairs. Candidate pairs that pass the broad-phase need a **narrow-phase** test to determine actual geometric contact — position, normal, and penetration depth — for physics constraint generation. Two algorithms dominate GPU narrow-phase: SAT (Separating Axis Theorem) for OBB pairs, and GJK (Gilbert-Johnson-Keerthi) for general convex hulls.

**OBB SAT (`sat_obb.comp`).** Two oriented bounding boxes are separated if a separating axis exists among the 15 candidates (3 face normals per box + 9 cross-product axes). Each candidate axis generates an interval overlap test:

```glsl
// sat_obb.comp — one workgroup per candidate OBB pair
layout(local_size_x=15) in;  // one thread per SAT axis

struct OBB { vec3 center; vec3 half;    // half-extents
             mat3 axes; };              // columns = local X,Y,Z in world space

layout(set=0, binding=0) readonly  buffer OBBs    { OBB obbs[]; };
layout(set=0, binding=1) readonly  buffer Pairs   { uvec2 pairs[]; };   // (idA, idB)
layout(set=0, binding=2) writeonly buffer Contacts{ vec4 contacts[]; }; // (normal, depth), sentinel if no contact
layout(set=0, binding=3) coherent  buffer Counter { uint count; };

shared float minPen;    // minimum penetration across all 15 axes
shared vec3  minAxis;

void main() {
    uint pairIdx = gl_WorkGroupID.x;
    uint axisIdx = gl_LocalInvocationID.x;   // 0..14

    uvec2 ids = pairs[pairIdx];
    OBB A = obbs[ids.x], B = obbs[ids.y];

    // Build 15 test axes
    vec3 axes[15];
    axes[0]=A.axes[0]; axes[1]=A.axes[1]; axes[2]=A.axes[2];
    axes[3]=B.axes[0]; axes[4]=B.axes[1]; axes[5]=B.axes[2];
    for (int i=0; i<3; ++i)
    for (int j=0; j<3; ++j)
        axes[6+i*3+j] = cross(A.axes[i], B.axes[j]);

    vec3 T   = B.center - A.center;
    vec3 ax  = normalize(axes[axisIdx] + vec3(1e-10)); // degenerate guard

    // Project both OBBs onto ax
    float rA = abs(dot(A.half.x * A.axes[0], ax))
             + abs(dot(A.half.y * A.axes[1], ax))
             + abs(dot(A.half.z * A.axes[2], ax));
    float rB = abs(dot(B.half.x * B.axes[0], ax))
             + abs(dot(B.half.y * B.axes[1], ax))
             + abs(dot(B.half.z * B.axes[2], ax));
    float pen = rA + rB - abs(dot(T, ax));

    // Reduce: find minimum penetration
    if (axisIdx == 0) { minPen = 1e30; }
    barrier(); memoryBarrierShared();

    atomicMin(floatBitsToUint(minPen), floatBitsToUint(pen > 0.0 ? pen : -1.0));
    barrier();

    if (pen == uintBitsToFloat(atomicOr(floatBitsToUint(minPen), 0u)) && pen > 0.0) {
        minAxis = dot(T, ax) < 0.0 ? -ax : ax;
    }
    barrier();

    if (axisIdx == 0) {
        float p = uintBitsToFloat(atomicOr(floatBitsToUint(minPen), 0u));
        if (p > 0.0) {
            uint slot = atomicAdd(count, 1u);
            contacts[slot] = vec4(minAxis, p);
        }
    }
}
```

**GJK for convex hulls.** GJK iteratively finds the closest point between two convex shapes by building a simplex (point, line, triangle, tetrahedron) in Minkowski difference space and walking toward the origin. If the origin is inside the simplex, the shapes overlap.

```glsl
// support() returns the point in Minkowski difference furthest along dir
vec3 support(vec3[] hullA, uint nA, vec3[] hullB, uint nB, vec3 dir) {
    // Furthest point in A along dir
    float maxA = -1e30; vec3 ptA;
    for (uint i=0; i<nA; ++i) { float d=dot(hullA[i],dir); if(d>maxA){maxA=d;ptA=hullA[i];} }
    // Furthest point in B along -dir
    float maxB = -1e30; vec3 ptB;
    for (uint i=0; i<nB; ++i) { float d=dot(hullB[i],-dir); if(d>maxB){maxB=d;ptB=hullB[i];} }
    return ptA - ptB;
}

bool gjk(vec3[] hullA, uint nA, vec3[] hullB, uint nB, out float dist) {
    vec3 d = vec3(1,0,0);
    vec3 simplex[4]; uint n = 0;
    simplex[n++] = support(hullA, nA, hullB, nB, d);
    d = -simplex[0];
    for (int iter = 0; iter < 64; ++iter) {
        vec3 a = support(hullA, nA, hullB, nB, d);
        if (dot(a, d) < 0.0) { dist = length(d); return false; }  // no intersection
        simplex[n++] = a;
        if (doSimplex(simplex, n, d)) { dist = 0.0; return true; } // origin inside
    }
    dist = length(d); return false;
}
```

`doSimplex` selects the voronoi region of the simplex closest to the origin and updates `d` accordingly — the standard 2/3/4-vertex cases from van den Bergen 2003.

**GPU dispatch pattern.** Assign one workgroup per candidate pair from the BVH broad-phase output. For 1000 candidate pairs, a single dispatch of 1000 workgroups (one per pair) completes narrow-phase in ~0.05 ms on a midrange GPU. Contact results feed into a physics constraint solver (PBD §4.6 or a rigid-body impulse solver) in the next compute pass.

[Source: Gilbert, Johnson, Keerthi "A Fast Procedure for Computing the Distance Between Complex Objects in Three-Dimensional Space", IEEE RA 1988; van den Bergen "Collision Detection in Interactive 3D Environments", 2003]

### 8.5 Convex Hull Generation on GPU

A convex hull is the tightest convex polytope enclosing a point set. It is required as: (a) a preprocessing step before GJK (§8.4) — GJK requires convex inputs; (b) the physics collider for rigid bodies before VHACD convex decomposition; (c) the initial seed polytope for EPA (Expanding Polytope Algorithm) penetration depth computation.

**GPU QuickHull.** The classic QuickHull algorithm adapts to GPU with a wave-front parallelism model:

1. **Extreme points** — reduce to find the 6 axis-aligned extremes (min/max X, Y, Z) in O(N/W) parallel reductions to form the initial simplex.
2. **Point-face assignment** — for each unprocessed point, test against all current horizon faces and assign to the furthest face it can see (positive dot product with face normal).
3. **Horizon ridge finding** — for each face's assigned point set, find the furthest point; mark visible faces from this point.
4. **Face update** — remove visible faces, emit new faces from the horizon ridges to the furthest point.

Steps 2–4 iterate until no points remain outside. The GPU-parallel bottleneck is step 2 (assign N points to F faces, O(NF) work) and step 4 (variable-length output):

```glsl
// hull_assign.comp — assign points to faces
layout(local_size_x=64) in;
layout(set=0,binding=0) readonly  buffer Points   { vec3 pts[]; };
layout(set=0,binding=1) readonly  buffer Faces    { vec4 planes[];  };  // (normal, d)
layout(set=0,binding=2) writeonly buffer Assignment{ uint faceID[]; };  // per point
layout(set=0,binding=3) writeonly buffer MaxDist  { float maxD[];   };  // per point
layout(push_constant) uniform PC { uint ptCount; uint faceCount; };

void main() {
    uint pi  = gl_GlobalInvocationID.x;
    if (pi >= ptCount) return;
    vec3 p   = pts[pi];
    uint best = 0xFFFFFFFF; float bestD = 0.0;
    for (uint fi = 0; fi < faceCount; ++fi) {
        vec4 pl  = planes[fi];
        float d  = dot(pl.xyz, p) + pl.w;    // signed distance to face
        if (d > bestD) { bestD = d; best = fi; }
    }
    faceID[pi] = best;
    maxD[pi]   = bestD;
}
```

A per-face reduction (atomic max over `maxD` within each face's assigned points) identifies the furthest point per face, then a serial CPU step updates the face list (or a GPU indirect dispatch handles this for fully GPU-resident hulls). Convergence is typically 5–20 iterations for smooth point clouds.

**VHACD (V-HACD 4.0) — Convex Decomposition.** Non-convex meshes require decomposition into a union of convex pieces before GJK. V-HACD (Volumetric Hierarchical Approximate Convex Decomposition) is the de-facto standard:

```cpp
// CPU: V-HACD decomposes a mesh into convex hulls for physics
VHACD::IVHACD::Parameters params;
params.m_maxNumVerticesPerCH = 64;
params.m_maxConvexHulls      = 32;
params.m_resolution          = 100000;   // voxel resolution

auto* vhacd = VHACD::CreateVHACD();
vhacd->Compute(vertices, vertCount, indices, triCount, params);

for (uint32_t i = 0; i < vhacd->GetNConvexHulls(); ++i) {
    VHACD::IVHACD::ConvexHull ch;
    vhacd->GetConvexHull(i, ch);
    // Upload ch.m_points as the convex hull vertex set for GJK §8.4
}
```

V-HACD runs on CPU in 0.1–2 s depending on resolution; the resulting convex pieces are uploaded once as static SSBOs for GJK dispatch at runtime. [Source: Mamou & Ghazali "A Simple and Efficient Approach for 3D Mesh Approximate Convex Decomposition", ICIP 2009; V-HACD v4.0 GitHub https://github.com/kmammou/v-hacd]

**GPU porting status.** No GPU port of VHACD exists or is in active development. The VHACD algorithm's inner loop — iterative voxelization, PCA convex hull approximation, and concavity-driven splitting — involves adaptive tree decomposition with data-dependent termination that does not parallelize efficiently. The more significant shift is that NVIDIA PhysX 5 (Ch69) and GPU-native physics engines increasingly use SDF-based collision geometry rather than convex decomposition: the SDF is computed once on GPU and ray-cast for collision queries, entirely bypassing the need for convex decomposition. For engines that still require explicit convex hulls (e.g., for GJK §8.4 or network-replicated physics shapes), VHACD remains a CPU-only offline preprocessing step.

---

## 9. Screen-Space Geometry Techniques

Screen-space methods approximate geometry operations using the depth buffer, G-buffer, or reprojected history — achieving results close to full geometric solutions at a fraction of the cost by operating in the 2D screen domain rather than the 3D object domain.

### 9.1 Screen-Space Tessellation and Dynamic LOD

Screen-space tessellation drives triangle subdivision level from the current depth buffer rather than a pre-authored LOD chain. Each frame, a compute pass reads the depth buffer, estimates the projected edge length (in pixels) for each triangle edge, and emits a variable-resolution mesh via a mesh shader.

**Depth-buffer LOD estimation:**

```glsl
// ss_tess_lod.glsl — in amplification/task shader
float screenEdgeLength(vec3 posA, vec3 posB, mat4 viewProj, vec2 screenSize) {
    vec4 cA = viewProj * vec4(posA, 1.0);
    vec4 cB = viewProj * vec4(posB, 1.0);
    vec2 sA = (cA.xy / cA.w * 0.5 + 0.5) * screenSize;
    vec2 sB = (cB.xy / cB.w * 0.5 + 0.5) * screenSize;
    return length(sA - sB);
}

// Target: ~8 pixels per edge segment
float tessLevel = clamp(screenEdgeLength(p0, p1, viewProj, screenSize) / 8.0,
                        1.0, 64.0);
gl_TessLevelOuter[0] = tessLevel;
```

**Screen-space crack prevention.** Adjacent patches at different screen depths use different tessellation levels. The shared-SSBO neighbour-level approach from §2.4 applies here too — read neighbour tessellation level and take the minimum on shared edges. Alternatively, use `fractional_odd_spacing` + level-match SSBO for seamless cracks at any level combination.

**Depth-based displacement.** A depth pre-pass in the vertex shader outputs only depth; a subsequent pass reads the depth buffer to estimate per-vertex curvature (depth Laplacian in screen space) and applies proportional tessellation — high curvature areas receive more triangles. This is the algorithm behind Unreal Engine's "Runtime Virtual Texture" terrain tessellation and Godot's terrain LOD shader.

### 9.2 Parallax Occlusion Mapping

Parallax Occlusion Mapping (POM) simulates geometric displacement entirely in the fragment shader by ray-marching through a heightmap, producing self-shadowing, depth-correct silhouettes, and contact hardening — without any tessellated geometry.

**POM fragment shader:**

```glsl
// pom_fragment.glsl
uniform sampler2D heightMap;
uniform float     heightScale;   // e.g. 0.05 (5 cm)
uniform int       numSteps;      // 16..64 depending on distance

vec2 parallaxOcclusionMapping(vec2 uv, vec3 viewDirTS, float heightScale) {
    float stepSize   = 1.0 / float(numSteps);
    float curHeight  = 0.0;
    vec2  dtex       = viewDirTS.xy / viewDirTS.z * heightScale * stepSize;

    // Linear ray march
    float prevH = 0.0, curSample = 1.0;
    vec2  prevUV = uv;
    for (int i = 0; i < numSteps; ++i) {
        curHeight  += stepSize;
        uv         -= dtex;
        curSample   = texture(heightMap, uv).r;
        if (curSample >= curHeight) {
            // Binary search refinement between prevUV and uv
            float weight = (curSample - curHeight) /
                           (curSample - prevH + stepSize);
            return mix(uv, prevUV, weight);
        }
        prevH = curSample; prevUV = uv;
    }
    return uv;
}

// In main():
vec3  viewDirTS = normalize(TBN_inverse * (cameraPos - fragPos));
vec2  displaced = parallaxOcclusionMapping(texCoord, viewDirTS, heightScale);
vec3  normal    = texture(normalMap, displaced).rgb * 2.0 - 1.0;
// ... rest of PBR shading
```

**Self-shadowing.** March from the displaced UV back toward the light direction in tangent space; if any heightmap sample occludes the light ray before it exits, apply a shadow factor. This produces soft contact shadows at geometry-height transitions at near-zero additional GPU cost.

**POM vs. tessellation displacement.** POM works entirely in the fragment shader (no geometry overhead) and is effective for viewing angles <60° from the normal. At grazing angles, the silhouette reveals the flat underlying geometry — tessellated displacement (§2.4) is required for correct silhouettes. The standard production hybrid: POM for mid-range surfaces, tessellation+displacement for hero assets at <5m camera distance.

### 9.3 Temporal Geometry Reprojection

Temporal reprojection caches per-pixel geometry computations from previous frames and reprojects them into the current frame using per-pixel motion vectors (from a dedicated motion vector pass or velocity G-buffer). This amortizes expensive per-frame geometry costs across multiple frames.

**Motion vector pass (`motion_vectors.glsl`):**

```glsl
// In vertex shader: output current and previous clip-space positions
layout(location=4) out vec4 curClip;
layout(location=5) out vec4 prevClip;

void main() {
    vec4 curr = viewProj     * model     * vec4(pos, 1.0);
    vec4 prev = prevViewProj * prevModel * vec4(pos, 1.0);
    curClip   = curr;
    prevClip  = prev;
    gl_Position = curr;
}

// In fragment shader: write motion vector
vec2 curNDC  = curClip.xy  / curClip.w;
vec2 prevNDC = prevClip.xy / prevClip.w;
outMotionVec = (curNDC - prevNDC) * 0.5;   // in [0,1] UV space delta
```

**Geometry reprojection for bent normals / AO.** Progressive bakes (§7.8) use temporal reprojection to accumulate AO samples across frames: each frame adds N new ray samples per vertex; the EMA blend `result = mix(cached, newSample, alpha)` converges to the full-sample result over 1/alpha frames. Motion vectors from the motion-vector pass reproject the cached AO from last frame's vertex positions:

```glsl
// reproject_ao.comp — one thread per vertex
vec2  prevUV    = uv + texture(motionVectors, uv).rg;   // reprojected UV
float cachedAO  = texture(prevAOMap, prevUV).r;
float newAO     = computeAOSample(pos, normal, bvh);    // N rays this frame
outAO[i]        = mix(cachedAO, newAO, 1.0/16.0);      // 16-frame EMA
```

**Temporal geometry caching.** For static environment geometry, cached G-buffer attributes (normals, albedo, depth) from the previous frame can fill pixels that were occluded this frame but visible last frame (disocclusion holes from camera movement). The reprojection confidence is weighted by motion-vector magnitude and depth difference — discard cache hits where the depth delta exceeds a threshold (indicating a different surface is now visible). This is the basis of Unreal Engine's Lumen global illumination cache and DXR "Restir" temporal reuse for ray-traced shadows. [Source: Karis "High Quality Temporal Supersampling", SIGGRAPH 2014; Schied et al. "Spatiotemporal Variance-Guided Filtering", HPG 2017]

---

## 10. GPU-Driven Rendering and Virtual Geometry

*Audience: graphics application developers, systems developers.*

Modern GPU-driven pipelines invert the CPU↔GPU command relationship: the GPU selects what to draw, computes per-object transforms, culls occluded geometry, and emits indirect draw commands — the CPU only submits one or a handful of dispatch calls. This section covers the core GPU data structures and algorithms that make this possible, from meshlet decomposition through cluster-DAG LOD selection.

### 10.1 Meshlets and Cluster Hierarchies

A **meshlet** is a small, cache-coherent patch of mesh triangles — typically 64 or 128 vertices and 64 or 126 triangles — chosen to maximise vertex reuse within the cluster. The vertex count limit is set so the meshlet's vertex data fits in the mesh shader's per-workgroup register file, and the triangle count limit matches the maximum primitive output the mesh shader can emit in one invocation.

**meshoptimizer 0.22** generates meshlets from any indexed triangle mesh:

```cpp
#include <meshoptimizer.h>

std::vector<meshopt_Meshlet> meshlets(max_meshlets);
std::vector<uint32_t> meshlet_verts(max_meshlets * 128);
std::vector<uint8_t>  meshlet_tris (max_meshlets * 126 * 3);

size_t count = meshopt_buildMeshlets(
    meshlets.data(), meshlet_verts.data(), meshlet_tris.data(),
    indices.data(), indices.size(), positions.data(),
    vertex_count, sizeof(glm::vec3),
    /*max_vertices=*/128, /*max_triangles=*/126, /*cone_weight=*/0.5f);
// cone_weight > 0 enables backface cone culling in the amplification shader
```

[Source: meshoptimizer `src/meshlet.cpp`](https://github.com/zeux/meshoptimizer/blob/master/src/meshlet.cpp)

Each `meshopt_Meshlet` carries a bounding sphere (center + radius) and a backface cone (axis + cutoff angle) encoded as `meshopt_Bounds`:

```cpp
meshopt_Bounds b = meshopt_computeMeshletBounds(
    meshlet_verts.data() + m.vertex_offset,
    meshlet_tris.data()  + m.triangle_offset,
    m.triangle_count,
    positions.data(), vertex_count, sizeof(glm::vec3));
// b.center, b.radius — frustum/occlusion cull sphere
// b.cone_axis[3], b.cone_cutoff — backface cone cull
```

Upload meshlet data to GPU SSBOs:

```c
// Binding layout (set = 0)
layout(set=0, binding=0) readonly buffer Meshlets   { MeshletData meshlets[]; };
layout(set=0, binding=1) readonly buffer MeshVerts  { uint meshlet_verts[]; };
layout(set=0, binding=2) readonly buffer MeshTris   { uint8_t meshlet_tris[]; };
layout(set=0, binding=3) readonly buffer Positions  { vec3 positions[]; };
layout(set=0, binding=4) readonly buffer DrawCmds   { DrawIndexedIndirectCommand cmds[]; };
```

### 10.2 Amplification/Task Shader Culling Pipeline

The Vulkan mesh shading pipeline (VK_EXT_mesh_shader, Vulkan 1.3+) uses a two-stage dispatch: an **amplification (task) shader** runs one workgroup per meshlet and either discards it or emits a mesh shader workgroup via `EmitMeshTasksEXT`:

```glsl
// task.glsl — frustum + backface cone cull per meshlet
#version 460
#extension GL_EXT_mesh_shader : require

layout(local_size_x = 32) in;

layout(push_constant) uniform PC { mat4 vp; vec4 frustum[6]; vec3 cam; } pc;

struct MeshletBounds { vec3 center; float radius; vec3 coneAxis; float coneCutoff; };
layout(set=0,binding=5) readonly buffer Bounds { MeshletBounds bounds[]; };

taskPayloadSharedEXT uint visibleMeshlets[32];
shared uint visCount;

void main() {
    uint mid = gl_GlobalInvocationID.x;
    if (gl_LocalInvocationID.x == 0) visCount = 0;
    barrier();

    MeshletBounds b = bounds[mid];

    // Frustum cull — sphere vs 6 planes
    bool visible = true;
    for (int i = 0; i < 6; i++)
        if (dot(pc.frustum[i].xyz, b.center) + pc.frustum[i].w < -b.radius)
            visible = false;

    // Backface cone cull
    vec3 toMeshlet = normalize(b.center - pc.cam);
    if (dot(toMeshlet, b.coneAxis) >= b.coneCutoff)
        visible = false;

    if (visible) {
        uint slot = atomicAdd(visCount, 1);
        visibleMeshlets[slot] = mid;
    }
    barrier();
    if (gl_LocalInvocationID.x == 0)
        EmitMeshTasksEXT(visCount, 1, 1);
}
```

The mesh shader receives the `visibleMeshlets` payload, fetches vertex and triangle data for the one assigned meshlet, and emits the final primitive list:

```glsl
// mesh.glsl
#version 460
#extension GL_EXT_mesh_shader : require
layout(local_size_x = 128) in;
layout(triangles, max_vertices=128, max_primitives=126) out;

taskPayloadSharedEXT uint visibleMeshlets[32];

void main() {
    uint mid = visibleMeshlets[gl_WorkGroupID.x];
    MeshletData m = meshlets[mid];

    // Emit vertices
    uint v = gl_LocalInvocationID.x;
    if (v < m.vertex_count) {
        uint vi = meshlet_verts[m.vertex_offset + v];
        gl_MeshVerticesEXT[v].gl_Position = pc.vp * vec4(positions[vi], 1.0);
    }

    // Emit triangles (3 threads per triangle)
    uint t = gl_LocalInvocationID.x;
    if (t < m.triangle_count) {
        uint base = m.triangle_offset + t * 3;
        gl_PrimitiveTriangleIndicesEXT[t] = uvec3(
            meshlet_tris[base], meshlet_tris[base+1], meshlet_tris[base+2]);
    }
    SetMeshOutputsEXT(m.vertex_count, m.triangle_count);
}
```

[Source: Vulkan Mesh Shading spec §14.2; Wihlidal "Optimizing the Graphics Pipeline with Compute", GDC 2016]

### 10.3 Hierarchical Z-Buffer Occlusion Culling

A **Hierarchical Z-Buffer (HZB)** is a mip-chain of the depth buffer where each texel stores the *maximum* (farthest) depth of its children — the safe conservative bound for occlusion testing. An object is occluded if its projected bounding sphere falls within a texel whose max-depth is nearer than the object's near depth.

Build the HZB with a compute shader that reduces 4×4 texel blocks taking the `max`:

```glsl
// hzb_reduce.comp
layout(set=0,binding=0) uniform sampler2D srcDepth;
layout(set=0,binding=1, r32f) uniform writeonly image2D dstHZB;
layout(local_size_x=8, local_size_y=8) in;

void main() {
    ivec2 dst = ivec2(gl_GlobalInvocationID.xy);
    ivec2 src = dst * 2;
    float d = max(max(texelFetch(srcDepth, src,           0).r,
                      texelFetch(srcDepth, src+ivec2(1,0),0).r),
                  max(texelFetch(srcDepth, src+ivec2(0,1),0).r,
                      texelFetch(srcDepth, src+ivec2(1,1),0).r));
    imageStore(dstHZB, dst, vec4(d));
}
```

Dispatch one pass per mip level until the chain reaches 1×1. Occlusion test in the task shader samples the appropriate mip level for the projected sphere radius:

```glsl
float projRadius = b.radius / (tan(fovY * 0.5) * max(screenH, screenW));
float mipLevel   = log2(projRadius * screenH);
float hzbDepth   = textureLod(hzb, projCenter.xy * 0.5 + 0.5, mipLevel).r;
if (projCenter.z - b.radius > hzbDepth)
    visible = false;  // entirely behind the depth surface
```

[Source: Uludag "Hi-Z GPU Occlusion Culling", GPU Pro 2; Schied "A Hierarchical Depth Buffer", ShaderX6]

### 10.4 Two-Phase Occlusion and Indirect Draw Compaction

A single-frame HZB test has a bootstrap problem: the depth buffer is empty at frame start. **Two-phase occlusion** resolves this using the previous frame's HZB:

- **Phase 1:** Test all objects against last frame's HZB. Objects passing are rendered; their depth contributions update the current HZB.
- **Phase 2:** Test Phase-1 rejects against the now-populated current HZB. Newly visible objects are rendered; their depth contributions extend the HZB.

Both phases use `vkCmdDrawIndexedIndirectCount` driven by a GPU compaction pass:

```glsl
// compact.comp — pack visible meshlet draw commands into an indirect buffer
layout(set=0,binding=0) readonly  buffer Visibility { uint visible[]; };
layout(set=0,binding=1) writeonly buffer IndirectBuf { DrawIndexedIndirectCommand cmds[]; };
layout(set=0,binding=2) buffer    DrawCount { uint count; };

layout(local_size_x=64) in;
void main() {
    uint id = gl_GlobalInvocationID.x;
    if (visible[id] == 1) {
        uint slot = atomicAdd(count, 1);
        cmds[slot] = DrawIndexedIndirectCommand(
            meshlets[id].triangle_count * 3,
            1, meshlets[id].triangle_offset, 0, id);
    }
}
```

[Source: Wihlidal "Cluster Culling", SIGGRAPH 2015; Karis "Nanite: A Deep Dive", SIGGRAPH 2021]

### 10.5 Virtual Geometry: Cluster DAG and Error-Metric LOD

Nanite-style virtual geometry organises all mesh clusters into a directed acyclic graph (DAG) by simplifying and re-clustering iteratively. Each cluster group has a **parent error** and **self error** metric (QEM-based screen-space error approximation). At runtime a single GPU traversal selects the finest cluster whose projected screen-space error falls below a pixel threshold:

```
for each cluster c in DAG (bottom-up BFS):
    parentError_screen = project(c.parentError, distance)
    selfError_screen   = project(c.selfError,   distance)
    if selfError_screen < threshold and parentError_screen >= threshold:
        draw c  // finest cluster satisfying the error bound
```

The traversal runs as a persistent compute kernel reading the cluster DAG from an SSBO and writing visible cluster IDs via `atomicAdd` into the indirect draw buffer. The key invariant is that every visible surface is covered by exactly one cluster at the finest acceptable resolution — no cracks, no overdraw between LOD levels. [Source: Karis et al. "Nanite Virtualised Geometry", SIGGRAPH 2021 Advances in Real-Time Rendering course notes]

---

## 11. Fluid Simulation on the GPU

*Audience: graphics application developers, systems developers.*

Fluid simulation divides into two major families: **Eulerian** (fixed grid, track field values at cell centers/faces) and **Lagrangian** (particles follow the fluid). Hybrid schemes (FLIP, APIC) combine both. GPU implementations exploit the data-parallelism in per-cell and per-particle updates, with the bottleneck usually being the pressure Poisson solve.

### 11.1 MAC Grid and Semi-Lagrangian Advection

The **Marker-and-Cell (MAC) grid** stores pressure at cell centers and velocity components at face centers (staggered). For an N³ grid a Vulkan compute layout maps naturally: one thread per cell for pressure, one thread per axis-face pair for velocity.

**Semi-Lagrangian advection** traces each grid point backward along the velocity field for one time step and samples the field at the traced position:

```glsl
// advect.comp
layout(set=0,binding=0) uniform sampler3D velX, velY, velZ;
layout(set=0,binding=1, r32f) uniform writeonly image3D outField;
layout(set=0,binding=2) uniform sampler3D srcField;
layout(push_constant) uniform PC { float dt; float invDx; vec3 origin; } pc;
layout(local_size_x=8, local_size_y=8, local_size_z=8) in;

void main() {
    ivec3 cell = ivec3(gl_GlobalInvocationID);
    vec3  pos  = vec3(cell) + 0.5;            // cell center in grid coords
    vec3  vel  = vec3(texture(velX, pos*pc.invDx).r,
                      texture(velY, pos*pc.invDx).r,
                      texture(velZ, pos*pc.invDx).r);
    vec3  back = (pos - vel * pc.dt) * pc.invDx;  // back-trace
    imageStore(outField, cell, textureLod(srcField, back, 0));
}
```

Third-order BFECC (Back and Forth Error Compensation and Correction) halves the dissipation with two additional advection passes and an error correction term. [Source: Kim et al. "Advections With Significantly Reduced Diffusion and Diffusion", SCA 2005]

### 11.2 Pressure Projection via Conjugate Gradient

After advection the velocity field is not divergence-free. The **pressure Poisson solve** enforces incompressibility:

```
∇²p = (1/Δt) ∇·u*     (Poisson equation)
u   = u* − Δt ∇p       (divergence-free correction)
```

On a uniform grid ∇²p is a sparse symmetric positive-definite matrix amenable to **conjugate gradient (CG)** with an incomplete-Cholesky (IC) or Jacobi preconditioner. Each CG iteration is two sparse matrix-vector products (SpMV) and a few dot products — all O(N³) parallel reductions:

```glsl
// jacobi_iter.comp — one Gauss-Seidel/Jacobi step for ∇²p = rhs
layout(set=0,binding=0, r32f) uniform image3D pressure;
layout(set=0,binding=1) uniform sampler3D rhs;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;

void main() {
    ivec3 c  = ivec3(gl_GlobalInvocationID);
    float r  = texelFetch(rhs, c, 0).r;
    float p  = (imageLoad(pressure, c+ivec3(1,0,0)).r +
                imageLoad(pressure, c-ivec3(1,0,0)).r +
                imageLoad(pressure, c+ivec3(0,1,0)).r +
                imageLoad(pressure, c-ivec3(0,1,0)).r +
                imageLoad(pressure, c+ivec3(0,0,1)).r +
                imageLoad(pressure, c-ivec3(0,0,1)).r - r) / 6.0;
    imageStore(pressure, c, vec4(p));
}
```

For production, replace Jacobi with a **multigrid V-cycle**: restrict residual to a coarser grid, smooth there, then prolongate the correction back. Each level halves the grid in each dimension; total work is O(N³) rather than O(N³ log N). [Source: Fedkiw, Stam, Jensen "Visual Simulation of Smoke", SIGGRAPH 2001; McAdams et al. "A Parallel Multigrid Poisson Solver", SCA 2010]

### 11.3 Position-Based Fluids (PBF) on GPU

**Position-Based Fluids** (Macklin & Müller 2013) treats each particle as a density constraint rather than a force, solving for position corrections that keep density at ρ₀:

```
C_i(x) = ρ_i / ρ_0 − 1 = 0
ρ_i = Σ_j m_j W(||x_i − x_j||, h)   // SPH density kernel
```

Per-iteration Lagrange multiplier λᵢ = −Cᵢ / (Σ_k |∇_{x_k} Cᵢ|² + ε):

```glsl
// pbf_lambda.comp
layout(set=0,binding=0) readonly  buffer Pos     { vec4 pos[]; };
layout(set=0,binding=1) readonly  buffer Cells   { int grid[]; };  // spatial hash
layout(set=0,binding=2) writeonly buffer Lambda  { float lambda[]; };

float W_poly6(float r, float h) {
    float x = max(0.0, h*h - r*r);
    return 315.0 / (64.0 * 3.14159 * pow(h,9)) * x*x*x;
}
vec3  W_spiky_grad(vec3 r, float h) {
    float rlen = length(r);
    if (rlen < 1e-6 || rlen > h) return vec3(0);
    float x = h - rlen;
    return -45.0 / (3.14159 * pow(h,6)) * x*x * normalize(r);
}
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    // accumulate density and gradient norm from neighbor cells
    float rho = 0.0, gradNorm = 0.0;
    // (neighbor iteration over spatial hash cells omitted for brevity)
    float Ci = rho / RHO0 - 1.0;
    lambda[i] = -Ci / (gradNorm + EPSILON_RELAXATION);
}
```

After computing λ, position corrections Δpᵢ = (1/ρ₀) Σⱼ (λᵢ + λⱼ) ∇W accumulate via a scatter pass (atomic floats or separate reduction). Surface tension, vorticity confinement, and XPBD compliance extend PBF without structural changes to the GPU pipeline. [Source: Macklin & Müller "Position Based Fluids", SIGGRAPH 2013; Macklin et al. "Unified Particle Physics for Real-Time Applications", SIGGRAPH 2014]

### 11.4 FLIP: Hybrid Particle-Grid

**FLIP** (Fluid Implicit Particles) transfers velocity between particles and a MAC grid each step, combining the detail of Lagrangian particles with the stability of an Eulerian pressure solve:

```
P2G (splat):   u_grid[cell] += w_ip * u_particle[i]   // weight by trilinear kernel
Pressure solve: u_grid = project(u_grid)
G2P (gather):  u_particle[i] += α * (u_grid_new[cell] - u_grid_old[cell])
               (α=0.95: mostly FLIP, 5% PIC damping to suppress noise)
Position update: x_particle += dt * u_particle
```

The P2G scatter pass uses `atomicAdd` on float SSBOs (requires `VK_EXT_shader_atomic_float`). The G2P gather pass reads from the post-solve grid and is fully parallel per particle.

The key GPU dispatch sequence per frame:

```
vkCmdDispatch(P2G_pipeline,    ceil(N_particles/64), 1, 1)
// pressure solve dispatches (CG iterations) …
vkCmdDispatch(G2P_pipeline,    ceil(N_particles/64), 1, 1)
vkCmdDispatch(advect_pipeline, ceil(N_grid/512),     1, 1)
```

[Source: Bridson "Fluid Simulation for Computer Graphics" §7; Zhu & Bridson "Animating Sand as a Fluid", SIGGRAPH 2005]

### 11.5 Lattice Boltzmann Method (LBM)

LBM replaces the Navier-Stokes equations with a kinetic model on a regular lattice (D2Q9 in 2D, D3Q19 or D3Q27 in 3D). Each cell stores 19 distribution functions fᵢ that represent particle populations moving in 19 directions. One step = **collision** (relax toward equilibrium) + **stream** (shift each fᵢ to its neighbor):

```glsl
// lbm_step.comp — BGK collision + stream (D2Q9 shown for brevity)
layout(set=0,binding=0) buffer F    { float f[9 * NCELLS]; };
layout(set=0,binding=1) buffer FNew { float fn[9 * NCELLS]; };
layout(local_size_x=64) in;

const vec2 e[9] = vec2[9](vec2(0,0),vec2(1,0),vec2(-1,0),
                            vec2(0,1),vec2(0,-1),vec2(1,1),
                            vec2(-1,1),vec2(1,-1),vec2(-1,-1));
const float w[9] = float[9](4./9.,1./9.,1./9.,1./9.,1./9.,
                              1./36.,1./36.,1./36.,1./36.);

void main() {
    uint c = gl_GlobalInvocationID.x;
    ivec2 xy = ivec2(c % WIDTH, c / WIDTH);

    // Compute macroscopic density and velocity
    float rho = 0.0; vec2 u = vec2(0);
    for (int i = 0; i < 9; i++) { float fi = f[i*NCELLS+c]; rho += fi; u += fi*e[i]; }
    u /= rho;

    float omega = 1.0 / (3.0*VISCOSITY + 0.5);  // relaxation: tau = 1/omega
    for (int i = 0; i < 9; i++) {
        float eu  = dot(e[i], u);
        float feq = w[i] * rho * (1.0 + 3.0*eu + 4.5*eu*eu - 1.5*dot(u,u));
        float fi  = f[i*NCELLS+c];
        // Stream: write to neighbor cell
        ivec2 dst = xy + ivec2(e[i]);
        if (dst.x >= 0 && dst.x < WIDTH && dst.y >= 0 && dst.y < HEIGHT)
            fn[i * NCELLS + dst.y*WIDTH+dst.x] = fi + omega*(feq - fi);
    }
}
```

LBM is embarrassingly parallel: every cell updates independently. It handles complex boundary conditions naturally and couples to GPU rendering by writing velocity magnitude as a texture. [Source: Körner et al. "Lattice Boltzmann Simulation of Flow in Anisotropic Porous Media", 2016; Lehmann "Esoteric-Pull and Esoteric-Push: An Two-in-One Single-Step Streaming Scheme", Computation 2022]

---

## 12. Deformable Bodies and Continuum Mechanics

*Audience: graphics application developers, systems developers.*

PBD cloth (§4.6) handles thin surfaces via distance and bending constraints. Volumetric deformable bodies require a continuum mechanics formulation — FEM for stiff materials, MPM for materials that fracture, flow, or undergo large deformation. Both decompose naturally into per-element parallel local steps and a global linear solve.

### 12.1 Mass-Spring Systems on GPU

The simplest volumetric deformable body discretises the mesh into a spring network: each edge becomes a damped spring with rest length ℓ₀, stiffness k, and damping d. Forces integrate via explicit Verlet:

```glsl
// spring_force.comp
layout(set=0,binding=0) readonly buffer Pos  { vec4 pos[]; };
layout(set=0,binding=1) readonly buffer Vel  { vec4 vel[]; };
layout(set=0,binding=2) readonly buffer Edges{ uvec2 edges[]; float rest[]; };
layout(set=0,binding=3) buffer Force { vec4 force[]; };
layout(local_size_x=64) in;

void main() {
    uint e    = gl_GlobalInvocationID.x;
    uvec2 ij  = edges[e];
    vec3  d   = pos[ij.y].xyz - pos[ij.x].xyz;
    float len = length(d);
    vec3  axis = d / max(len, 1e-6);
    float fs  = SPRING_K * (len - rest[e]);
    vec3  fd  = DAMPING  * dot(vel[ij.y].xyz - vel[ij.x].xyz, axis) * axis;
    vec3  f   = (fs + length(fd)) * axis;
    atomicAdd(force[ij.x].x,  f.x);   // VK_EXT_shader_atomic_float
    atomicAdd(force[ij.y].x, -f.x);
    // (y,z components similarly)
}
```

Stiff springs require implicit integration (backward Euler, sparse linear solve) to avoid instability at large time steps.

### 12.2 Corotational FEM on GPU

**Corotational FEM** factors out rigid-body rotation from the deformation so the elastic energy is measured relative to the rotated rest configuration. For a tetrahedral mesh:

1. **Per-tet deformation gradient** F = ∂x/∂X (3×3 matrix, one per tet — fully parallel).
2. **Polar decomposition** F = RS (R rotation, S symmetric stretch). Computed via iterative SVD or QR.
3. **Corotational elastic force** on each tet: f = −k_vol · Rᵀ(F − R)

The polar decomposition for 3×3 matrices converges in 4–6 iterations of the cyclic Jacobi method, parallelisable per tet:

```glsl
// corot_tet.comp
layout(local_size_x=64) in;
layout(set=0,binding=0) readonly buffer RestPos { vec3 X[]; };
layout(set=0,binding=1) readonly buffer CurPos  { vec3 x[]; };
layout(set=0,binding=2) readonly buffer Tets    { uvec4 tets[]; };
layout(set=0,binding=3) buffer Force { vec4 f[]; };

mat3 polarRotation(mat3 F) {
    mat3 R = mat3(1.0);
    for (int iter = 0; iter < 6; iter++) {
        vec3 omega = (cross(R[0],F[0])+cross(R[1],F[1])+cross(R[2],F[2])) /
                     (abs(dot(R[0],F[0])+dot(R[1],F[1])+dot(R[2],F[2]))+1e-6);
        float w = length(omega);
        if (w < 1e-8) break;
        R = mat3(cos(w)*R + (1-cos(w))/w/w*outerProd(omega,omega)*R +
                 sin(w)/w*cross(omega, R));  // Rodrigues update
    }
    return R;
}

void main() {
    uint t = gl_GlobalInvocationID.x;
    uvec4 idx = tets[t];
    mat3 Ds = mat3(x[idx.y]-x[idx.x], x[idx.z]-x[idx.x], x[idx.w]-x[idx.x]);
    mat3 Dm = mat3(X[idx.y]-X[idx.x], X[idx.z]-X[idx.x], X[idx.w]-X[idx.x]);
    mat3 F  = Ds * inverse(Dm);
    mat3 R  = polarRotation(F);
    mat3 P  = LAMBDA * (F-R);  // first Piola-Kirchhoff (simplified)
    // distribute forces to vertices via scatter (atomicAdd)
}
```

[Source: Müller & Gross "Interactive Virtual Materials", GI 2004; Irving et al. "Invertible Finite Elements for Robust Simulation", SCA 2004]

### 12.3 Material Point Method (MPM) on GPU

MPM represents material as particles carrying mass, velocity, and deformation gradient, and transfers momentum through a background grid each step. The GPU pipeline alternates between particle-centric and grid-centric passes:

```
P2G (transfer to grid):
    for each particle p:
        for each nearby grid node i:
            grid_mass[i]     += w_ip * mass_p
            grid_momentum[i] += w_ip * mass_p * (vel_p + B_p * (xi - xp))
                                 // APIC affine momentum term
Grid update:
    grid_vel[i] = grid_momentum[i] / grid_mass[i]
    apply gravity, boundary conditions
G2P (transfer back to particles):
    for each particle p:
        vel_p = Σ_i w_ip * grid_vel[i]
        x_p  += dt * vel_p
        F_p   = (I + dt * Σ_i grid_vel[i] * ∇w_ip^T) * F_p
```

The cubic B-spline weight function wᵢₚ = N(xᵢ − xₚ/dx) covers a 3×3×3 stencil, giving each particle 27 grid-node interactions. P2G uses `atomicAdd` on grid mass and momentum (VK_EXT_shader_atomic_float); G2P is a fully parallel gather. MPM handles snow compaction, sand, and fracture by choosing the appropriate constitutive model for the deformation gradient update. [Source: Stomakhin et al. "A Material Point Method for Snow Simulation", SIGGRAPH 2013; Hu et al. "A Moving Least Squares Material Point Method", SIGGRAPH 2018]

---

## 13. Procedural Geometry Generation

*Audience: graphics application developers.*

Procedural generation creates geometry analytically rather than from stored mesh data, enabling effectively infinite variation at GPU-native speed. The common pattern: a compute shader writes vertex/index data into a pre-allocated SSBO that is then consumed directly by the rasterizer — no CPU roundtrip.

### 13.1 GPU Terrain: Fractional Brownian Motion and Erosion

A terrain heightmap can be generated entirely in a compute shader using layered noise (fBm):

```glsl
// terrain_gen.comp
layout(set=0,binding=0, r32f) uniform writeonly image2D heightmap;
layout(local_size_x=16, local_size_y=16) in;

float hash(vec2 p) { return fract(sin(dot(p, vec2(127.1,311.7))) * 43758.5453); }
float noise(vec2 p) {
    vec2 i = floor(p), f = fract(p);
    f = f*f*(3.0-2.0*f);
    return mix(mix(hash(i), hash(i+vec2(1,0)), f.x),
               mix(hash(i+vec2(0,1)), hash(i+vec2(1,1)), f.x), f.y);
}

float fbm(vec2 p) {
    float v=0.0, a=0.5;
    for (int i=0; i<8; i++) { v += a*noise(p); p*=2.1; a*=0.5; }
    return v;
}

void main() {
    ivec2 coord = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv    = vec2(coord) / vec2(imageSize(heightmap));
    imageStore(heightmap, coord, vec4(fbm(uv * 4.0)));
}
```

**Hydraulic erosion** (Mei et al. 2007) iterates across the heightmap: each cell receives rain, water flows to lower neighbors driven by height differences, sediment is eroded proportional to flow velocity, and deposited where flow slows:

```glsl
// erosion_step.comp — simplified single-pass thermal + hydraulic
layout(set=0,binding=0) buffer Height   { float h[]; };
layout(set=0,binding=1) buffer Water    { float w[]; };
layout(set=0,binding=2) buffer Sediment { float s[]; };
layout(local_size_x=64) in;

void main() {
    uint c = gl_GlobalInvocationID.x;
    ivec2 xy = ivec2(c % W, c / W);
    // compute outflow to 4 neighbors by height difference
    float totalH = h[c] + w[c];
    float d[4]; float totalFlow = 0.0;
    for (int i = 0; i < 4; i++) {
        ivec2 n = xy + OFFSETS[i];
        float hn = h[n.y*W+n.x] + w[n.y*W+n.x];
        d[i] = max(0.0, totalH - hn);
        totalFlow += d[i];
    }
    float scale = min(w[c], totalFlow) / max(totalFlow, 1e-6);
    for (int i = 0; i < 4; i++) {
        uint nc = uint((xy + OFFSETS[i]).y*W + (xy + OFFSETS[i]).x);
        float flow = scale * d[i];
        atomicAdd_f(w, nc, +flow);
        atomicAdd_f(w, c,  -flow);
        atomicAdd_f(s, nc, +flow * EROSION_K * h[c]);
        atomicAdd_f(h, c,  -flow * EROSION_K);
    }
}
```

100–200 iterations produce visually plausible erosion channels and alluvial fans. [Source: Mei et al. "Fast Hydraulic Erosion Simulation", 2007; Olsen "Realtime Procedural Terrain Generation", 2004]

### 13.2 L-Systems in Compute Shaders

L-systems expand a string of symbols by applying production rules iteratively. GPU parallelisation uses a **parallel prefix scan** to determine output positions before writing:

1. Each symbol in the current string is one thread; it reads its character and looks up the production rule length.
2. Prefix scan over rule lengths gives each symbol its output offset.
3. Each thread writes its expanded string into the output buffer at its computed offset.
4. Repeat for N generations.

```glsl
// lsystem_expand.comp
layout(set=0,binding=0) readonly  buffer InStr  { uint8_t in_str[]; };
layout(set=0,binding=1) writeonly buffer OutStr { uint8_t out_str[]; };
layout(set=0,binding=2) readonly  buffer Offsets{ uint offsets[]; };  // prefix scan result
layout(set=0,binding=3) readonly  buffer Rules  { uint8_t rules[512]; uint ruleLen[256]; };
layout(local_size_x=64) in;

void main() {
    uint i    = gl_GlobalInvocationID.x;
    uint sym  = in_str[i];
    uint base = offsets[i];
    uint len  = ruleLen[sym];
    for (uint j = 0; j < len; j++)
        out_str[base + j] = rules[sym * MAX_RULE_LEN + j];
}
```

A turtle interpretation pass then converts the symbol string to vertex data: `F` → emit cylinder segment, `+`/`−` → rotate, `[`/`]` → push/pop transform stack. Branch stacks require serialization but can be pre-computed in a first pass. [Source: Prusinkiewicz & Lindenmayer "The Algorithmic Beauty of Plants" §1; Lienhard et al. "Fast and Simple Creation of GPU Tree Models", TPCG 2010]

### 13.3 Wave Function Collapse for Tile Geometry

Wave Function Collapse (WFC) generates tile-map geometry consistent with local adjacency constraints extracted from an example. The GPU implementation adapts the AC-3 arc-consistency propagation as a BFS wavefront:

```glsl
// wfc_propagate.comp — one pass of constraint propagation
layout(set=0,binding=0) buffer CellMask { uint mask[]; }; // bitfield of allowed tiles per cell
layout(set=0,binding=1) buffer Changed  { uint changed; };
layout(local_size_x=64) in;

layout(push_constant) uniform PC { uint W, H; } pc;
// adjacency_table[tile * 4 + direction] = allowed-neighbor bitfield
layout(set=0,binding=2) readonly buffer AdjTable { uint adj[]; };

void main() {
    uint c   = gl_GlobalInvocationID.x;
    ivec2 xy = ivec2(c % pc.W, c / pc.W);
    uint m   = mask[c];
    uint newm = 0xFFFFFFFF;
    for (int dir = 0; dir < 4; dir++) {
        ivec2 n = xy + OFFSETS[dir];
        if (n.x < 0 || n.x >= int(pc.W) || n.y < 0 || n.y >= int(pc.H)) continue;
        uint nm = mask[n.y*pc.W + n.x];
        // For each allowed tile in neighbor, OR its allowed-neighbor bits for direction
        uint allowed = 0u;
        uint bits = nm;
        while (bits != 0) {
            int t = findLSB(bits); bits &= bits-1;
            allowed |= adj[t * 4 + (dir ^ 1)];
        }
        newm &= allowed;
    }
    newm &= m;
    if (newm != m) { mask[c] = newm; atomicOr(changed, 1u); }
}
```

Iterate until `changed == 0`. A CPU-side collapse step picks the lowest-entropy cell, collapses its mask to one tile, and re-runs the propagation kernel. [Source: Gumin "Wave Function Collapse" (2016); Merrell "Example-Based Model Synthesis", I3D 2007]

### 13.4 Task Shader Procedural Instancing

For buildings, rocks, and foliage, the amplification/task shader is an ideal procedural generator: given a sparse set of placement points in a SSBO, the task shader evaluates placement rules (slope, altitude, biome mask), generates per-instance transforms, and emits mesh shader workgroups only for visible, rule-passing instances:

```glsl
// proc_instance.task.glsl
layout(set=0,binding=0) readonly buffer PlacementPoints { vec4 pts[]; };
layout(set=0,binding=1) readonly buffer BiomeMask       { float biome[]; };
layout(local_size_x=32) in;
taskPayloadSharedEXT InstancePayload { mat4 transforms[32]; uint count; } payload;

void main() {
    uint id = gl_GlobalInvocationID.x;
    if (gl_LocalInvocationID.x == 0) payload.count = 0;
    barrier();

    vec4  pt    = pts[id];
    float slope = computeSlope(pt.xy);
    float b     = biome[id];

    bool emit = slope < MAX_SLOPE && b > MIN_BIOME_WEIGHT;
    // frustum cull
    if (emit) emit = sphereInFrustum(pt.xyz, INST_RADIUS);

    if (emit) {
        uint slot = atomicAdd(payload.count, 1);
        payload.transforms[slot] = buildTransform(pt.xyz, slope);
    }
    barrier();
    if (gl_LocalInvocationID.x == 0)
        EmitMeshTasksEXT(payload.count, 1, 1);
}
```

[Source: Wihlidal "GPU-Driven Rendering Pipelines", SIGGRAPH 2015; Graham "Procedural Generation in Game Development", GDC 2019]

---

## 14. Geodesics and Discrete Differential Geometry

*Audience: graphics application developers, systems developers.*

Discrete differential geometry provides a rigorous framework for computing smooth differential quantities (curvature, Laplacian, geodesic distance) on triangle meshes. All core operations reduce to sparse linear algebra on the mesh adjacency graph — well-suited for GPU conjugate gradient solvers.

### 14.1 Cotangent Laplacian

The **cotangent Laplacian** is the discrete analogue of the smooth Laplace-Beltrami operator. For an edge (i,j) adjacent to triangles with opposite angles αᵢⱼ and βᵢⱼ:

```
L_ij = (cot αᵢⱼ + cot βᵢⱼ) / 2    (off-diagonal)
L_ii = −Σⱼ L_ij                     (diagonal)
```

Construct the sparse CSR matrix on GPU: one thread per edge writes two symmetric off-diagonal entries; one thread per vertex sums its row for the diagonal:

```glsl
// cotan_laplacian.comp
layout(set=0,binding=0) readonly buffer Verts  { vec3 verts[]; };
layout(set=0,binding=1) readonly buffer HalfEdges { HalfEdge he[]; }; // opposite vert idx
layout(set=0,binding=2) buffer CSRValues { float vals[]; };
layout(local_size_x=64) in;

void main() {
    uint e = gl_GlobalInvocationID.x;
    uint i = he[e].vert, j = he[he[e].next].vert;
    uint opp_a = he[he[e].twin ? he[he[e].twin].prev : e].vert;
    uint opp_b = he[he[e].prev].vert;
    vec3 va = verts[opp_a]-verts[i], vb = verts[j]-verts[i];
    vec3 vc = verts[opp_b]-verts[j], vd = verts[i]-verts[j];
    float cotA = dot(va,vb) / max(length(cross(va,vb)), 1e-8);
    float cotB = dot(vc,vd) / max(length(cross(vc,vd)), 1e-8);
    float w    = 0.5 * (cotA + cotB);
    vals[he[e].csr_idx] = w;
    atomicAdd(vals[he[e].diag_idx], -w);
}
```

[Source: Botsch et al. "Polygon Mesh Processing" §3; Meyer et al. "Discrete Differential-Geometry Operators for Triangulated 2-Manifolds", 2003]

### 14.2 Heat Method for Geodesic Distance

The **heat method** (Crane, Weischedel, Wardetzky 2013) computes exact geodesic distances on triangle meshes in three GPU steps:

**Step 1 — Heat flow:** Solve `(M − δt L) u = δ` where M is the mass matrix, L is the cotangent Laplacian, δ is a heat source at the query vertex, and δt = h² (h = mean edge length). One sparse CG solve (∼20 iterations):

```glsl
// Preconditioned CG loop (one kernel per iteration, barrier between)
// A = M - dt*L,  b = delta_source,  solve A*u = b
```

**Step 2 — Normalize gradient:** Per-triangle compute ∇u, then normalize: X = −∇u / ‖∇u‖. This is fully parallel (one thread per triangle).

**Step 3 — Poisson solve:** Solve `L φ = ∇·X` (divergence of the normalized gradient field). Another sparse CG solve. The result φ is the geodesic distance field, shifted to zero at the source vertex.

All three steps use the same cotangent Laplacian built once per mesh. Total GPU time for a 100k-vertex mesh: ∼5 ms on modern hardware. [Source: Crane et al. "Geodesics in Heat", ACM TOG 2013; code at https://github.com/nmwsharp/geometry-central]

### 14.3 Mean Curvature Flow and Mesh Smoothing

**Mean curvature flow** moves each vertex in the direction of the mean curvature normal H n = L x (Laplacian of position). Implicit integration avoids instability:

```
(M − δt L) x_{t+1} = M x_t     →  sparse CG solve
```

One dispatch per CG iteration. The mass matrix M uses "mixed Voronoi area" weights for the vertex lumping. For pure isotropic smoothing the Laplacian operator above suffices; for **feature-preserving smoothing** replace L with an anisotropic Laplacian that suppresses diffusion across sharp edges (dihedral angle > threshold).

**Taubin λ/μ smoothing** avoids volume shrinkage by alternating a diffuse step (λ > 0) with an anti-diffuse step (μ < 0, |μ| > |λ|):

```glsl
// taubin.comp — one diffusion step
layout(local_size_x=64) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    vec3 Li = vec3(0); float wsum = 0.0;
    for each neighbor j of i:
        float w = cotanWeight(i, j);
        Li += w * (pos[j] - pos[i]); wsum += w;
    pos_out[i] = pos[i] + LAMBDA * Li / max(wsum, 1e-8);
}
// Second pass replaces LAMBDA with MU (negative)
```

[Source: Taubin "A Signal Processing Approach to Fair Surface Design", SIGGRAPH 1995; Desbrun et al. "Implicit Fairing of Irregular Meshes using Diffusion and Curvature Flow", SIGGRAPH 1999]

### 14.4 Principal Curvature Estimation on GPU

Principal curvatures κ₁, κ₂ and their directions e₁, e₂ are the eigenvalues/vectors of the shape operator at each vertex. GPU estimation uses the per-triangle curvature tensor method (Rusinkiewicz 2004):

1. Per-triangle: fit a 2×2 curvature tensor from position and normal differences at the triangle's vertices (parallel).
2. Per-vertex: scatter-accumulate (atomicAdd) weighted triangle tensors onto vertex SSBO.
3. Per-vertex: 2×2 symmetric eigendecomposition (closed-form, one thread per vertex) → κ₁, κ₂, e₁, e₂.

Principal curvatures feed downstream algorithms: anisotropic remeshing (align quads to curvature directions), feature line extraction (ridges where κ₁ is locally maximal), and shading. [Source: Rusinkiewicz "Estimating Curvatures and Their Derivatives on Triangle Meshes", 3DPVT 2004]

---

## 15. Shape Deformation Beyond Skinning

*Audience: graphics application developers.*

Skeletal skinning (§4) deforms a mesh by blending bone transforms. More general deformation methods impose different structural constraints: ARAP preserves local rigidity cell-by-cell, cage deformation provides artistic control via a coarse proxy, and RBF deformation scatters influence from a sparse set of handles.

### 15.1 As-Rigid-As-Possible (ARAP) Deformation

ARAP (Sorkine & Alexa 2007) minimises the distortion energy:

```
E(p) = Σᵢ Σⱼ∈N(i) wᵢⱼ ‖(pᵢ − pⱼ) − Rᵢ(pᵢ₀ − pⱼ₀)‖²
```

where pᵢ₀ are rest positions, pᵢ are deformed positions, and Rᵢ is a per-vertex rotation. Minimisation alternates:

**Local step** (parallel per vertex — GPU): Given current p, compute optimal Rᵢ via SVD of the 3×3 covariance matrix Sᵢ = Σⱼ wᵢⱼ (pᵢ₀ − pⱼ₀)(pᵢ − pⱼ)ᵀ. Extract Rᵢ = V Uᵀ from SVD(Sᵢ) = U Σ Vᵀ.

**Global step** (sparse CG on GPU): Solve L p = b where b = Σⱼ wᵢⱼ Rᵢ(pᵢ₀ − pⱼ₀) + Rⱼ(pⱼ₀ − pᵢ₀) (fixed-boundary RHS).

```glsl
// arap_local.comp — per-vertex optimal rotation
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    mat3 S = mat3(0.0);
    for each neighbor j of i:
        float w = cotanWeight(i, j);
        vec3 pij0 = restPos[i] - restPos[j];
        vec3 pij  = curPos[i]  - curPos[j];
        S += w * outerProduct(pij0, pij);  // 3x3 accumulate
    mat3 U, V; vec3 sigma;
    svd3x3(S, U, sigma, V);               // inline cyclic Jacobi SVD
    R[i] = V * transpose(U);
}
```

3–5 local+global iterations converge to a near-optimal deformation. [Source: Sorkine & Alexa "As-Rigid-As-Possible Surface Modeling", SGP 2007; implementation in libigl `igl::arap()`]

### 15.2 Cage-Based Deformation: Mean Value Coordinates

A **cage** is a coarse triangulated hull enclosing the high-resolution mesh. Each mesh vertex is expressed as a weighted combination of cage vertices via **Mean Value Coordinates** (MVC):

```
x = Σⱼ wⱼ(x₀) · cⱼ / Σⱼ wⱼ(x₀)
```

where wⱼ(x₀) = (tan(θⱼ₋₁/2) + tan(θⱼ/2)) / ‖cⱼ − x₀‖ and θᵢ is the angle subtended at x₀ by the triangle edge opposite to cage vertex j.

Precompute MVC weights at rest (CPU, one per mesh-vertex × cage-vertex pair); store as a sparse matrix SSBO. Deformation is a single sparse matrix-vector product per frame — ideal for GPU:

```glsl
// mvc_deform.comp
layout(set=0,binding=0) readonly buffer MVCWeights { float w[]; uint cageIdx[]; uint rowStart[]; };
layout(set=0,binding=1) readonly buffer CagePos    { vec3 cage[]; };
layout(set=0,binding=2) writeonly buffer MeshPos   { vec3 mesh[]; };
layout(local_size_x=64) in;
void main() {
    uint v   = gl_GlobalInvocationID.x;
    vec3 pos = vec3(0); float wsum = 0.0;
    for (uint k = rowStart[v]; k < rowStart[v+1]; k++) {
        float wk = w[k];
        pos += wk * cage[cageIdx[k]];
        wsum += wk;
    }
    mesh[v] = pos / max(wsum, 1e-8);
}
```

[Source: Floater "Mean Value Coordinates", CAGD 2003; Ju et al. "Mean Value Coordinates for Closed Triangular Meshes", SIGGRAPH 2005]

### 15.3 Radial Basis Function Deformation

RBF deformation places N control points (handles) in space. Moving a handle from xᵢ to xᵢ' induces a displacement field:

```
d(x) = Σᵢ αᵢ φ(‖x − xᵢ‖)    where φ(r) = r³  (polyharmonic spline)
```

Coefficients αᵢ are solved offline when handles move (N×N linear system). At runtime, evaluate d(x) at every mesh vertex — O(N · V) operations, parallelised per vertex:

```glsl
// rbf_eval.comp
layout(set=0,binding=0) readonly buffer Handles { vec4 handles[]; };   // xyz pos, w unused
layout(set=0,binding=1) readonly buffer Coeffs  { vec3 alpha[]; };
layout(set=0,binding=2) buffer MeshPos { vec3 pos[]; };
layout(local_size_x=64) in;
void main() {
    uint v  = gl_GlobalInvocationID.x;
    vec3 d  = vec3(0);
    for (uint i = 0; i < NUM_HANDLES; i++) {
        float r = length(pos[v] - handles[i].xyz);
        d += alpha[i] * r * r * r;   // φ(r) = r³
    }
    pos[v] += d;
}
```

For many handles (N > 64), use a spatial partition to evaluate only nearby handles — reduces O(NV) to O(kV) where k is the number of handles within the RBF support radius. [Source: Botsch et al. "Deformation Transfer for Triangle Meshes", SIGGRAPH 2006; Sumner et al. "Mesh-Based Inverse Kinematics", SIGGRAPH 2005]

---

## 16. 3D Gaussian Splatting and Neural Geometry

*Audience: graphics application developers.*

3DGS (Kerbl et al. 2023) replaces triangle meshes with a set of 3D Gaussian primitives, each parameterised by position, covariance, opacity, and view-dependent colour encoded in spherical harmonics. Rendering is a differentiable rasterization pipeline that has overtaken NeRF for real-time novel-view synthesis. The GPU pipeline is fully Vulkan/compute-compatible.

### 16.1 3DGS Representation and Projection

Each Gaussian g has:

- **Position** μ ∈ ℝ³
- **Covariance** Σ = R S Sᵀ Rᵀ (rotation R from unit quaternion q, diagonal scale S)
- **Opacity** σ ∈ (0,1)
- **SH coefficients** for RGB colour (up to degree 3 → 48 floats)

Project to screen-space covariance Σ' = J W Σ Wᵀ Jᵀ, where W is the view transform and J is the Jacobian of the perspective projection at μ:

```glsl
// project_gaussians.comp
layout(set=0,binding=0) readonly buffer Gaussians { GaussianData g[]; };
layout(set=0,binding=1) writeonly buffer Projected { ProjectedG pg[]; };
layout(push_constant) uniform PC { mat4 view, proj; vec2 focal; } pc;
layout(local_size_x=64) in;

void main() {
    uint i   = gl_GlobalInvocationID.x;
    vec4 mu4 = pc.view * vec4(g[i].pos, 1.0);
    float tx = mu4.x, ty = mu4.y, tz = mu4.z;

    // Jacobian of perspective projection
    mat3 J = mat3(pc.focal.x/tz, 0, -pc.focal.x*tx/(tz*tz),
                  0, pc.focal.y/tz, -pc.focal.y*ty/(tz*tz),
                  0, 0, 0);
    mat3 W3 = mat3(pc.view);
    mat3 cov3d = buildCov3D(g[i].scale, g[i].rot);
    mat3 cov2d = J * W3 * cov3d * transpose(W3) * transpose(J);

    pg[i].pos2d  = (pc.proj * mu4).xy / mu4.w * 0.5 + 0.5;
    pg[i].cov2d  = vec3(cov2d[0][0], cov2d[0][1], cov2d[1][1]);
    pg[i].depth  = tz;
    pg[i].color  = evalSH(g[i].sh, normalize(-mu4.xyz));
    pg[i].opacity = g[i].opacity;
}
```

[Source: Kerbl et al. "3D Gaussian Splatting for Real-Time Novel View Synthesis", SIGGRAPH 2023; https://github.com/graphdeco-inria/gaussian-splatting]

### 16.2 Tile Rasterizer: Sort-by-Depth and Alpha Compositing

Rendering proceeds in four steps:

**Step 1 — Tile assignment.** Each Gaussian is assigned to one or more 16×16 screen tiles based on its 2D bounding box (3σ ellipse). A compact list of (tile_id << 32 | depth) sort keys is emitted — one per Gaussian-tile pair — via prefix scan:

```glsl
// tile_assign.comp — count tiles per Gaussian, prefix scan, then scatter keys
uint nTiles = countTilesInBbox(pg[i].pos2d, pg[i].cov2d);
uint base   = scanOffset[i];
for (uint t = 0; t < nTiles; t++)
    keys[base + t] = (tileID(t) << 32) | floatBitsToUint(pg[i].depth);
```

**Step 2 — Sort.** Radix sort keys by tile_id (high 32 bits) then depth (low 32 bits). Use the VkRadixSort library or a custom 4-pass GPU radix sort.

**Step 3 — Tile range.** A single scan over the sorted key array marks the start/end index in the sorted list for each tile.

**Step 4 — Alpha composite per tile.** Each tile gets one compute workgroup; it iterates its Gaussian list front-to-back and blends:

```glsl
// splat_forward.comp
layout(local_size_x=16, local_size_y=16) in;
void main() {
    ivec2 px = ivec2(gl_GlobalInvocationID.xy);
    vec3 col = vec3(0); float T = 1.0;   // transmittance
    for (uint k = tileStart[tileID]; k < tileEnd[tileID]; k++) {
        uint i     = sortedIdx[k];
        vec2 delta = vec2(px) - pg[i].pos2d * screenSize;
        float power = evalGaussian2D(delta, pg[i].cov2d);
        float alpha = min(0.99, pg[i].opacity * exp(-power));
        if (alpha < 1.0/255.0) continue;
        col += T * alpha * pg[i].color;
        T   *= (1.0 - alpha);
        if (T < 1e-4) break;
    }
    imageStore(outImage, px, vec4(col, 1.0 - T));
}
```

[Source: Kerbl et al. 2023 §4; GPU rasterizer implementation at https://github.com/graphdeco-inria/diff-gaussian-rasterization]

### 16.3 Mesh Extraction from 3DGS and NeRF

Converting a trained 3DGS scene to a triangle mesh enables traditional rendering pipelines and physics simulation. Two approaches:

**SuGaR** (Guédon & Lepetit 2023) adds a regularisation term during training that aligns Gaussians to surface sheets, then extracts a mesh by Poisson reconstruction (§3.12) from Gaussian centers weighted by opacity.

**NeRF → mesh** (using Instant NGP): marching cubes on the NeRF density field at a threshold σ_surface. The density grid is evaluated at each voxel center in a compute dispatch, then piped into the two-pass GPU MC pipeline from §3.2. Texture maps are baked by computing the NeRF colour at each surface point (§7.13 GPU texture baking pipeline).

**Occupancy networks** predict binary occupancy o(x) ∈ {0,1} for any query point. Run inference on a 256³ voxel grid in parallel (batched MLP evaluation in compute), then extract the isosurface with GPU marching cubes. [Source: Guédon & Lepetit "SuGaR: Surface-Aligned Gaussian Splatting", CVPR 2024; Müller et al. "Instant Neural Graphics Primitives", SIGGRAPH 2022]

---

## 17. Spatial Data Structures Beyond BVH

*Audience: systems developers, graphics application developers.*

§8 covers GPU LBVH construction for ray-surface intersection. Other spatial data structures serve different access patterns: k-d trees for nearest-neighbour queries, uniform grids for short-range interactions (SPH, collision), and sparse voxel octrees for multi-resolution volume traversal.

### 17.1 GPU k-d Trees

A **left-balanced k-d tree** stored as a flat array supports GPU traversal without pointer chasing: node i has left child 2i and right child 2i+1. Construction sorts points by each split dimension using a radix sort, picks the median as the root, and recurses on the two halves:

```glsl
// kd_query.comp — nearest-neighbour search, stackless traversal
layout(set=0,binding=0) readonly buffer KDTree { vec4 nodes[]; }; // xyz + splitAxis
layout(set=0,binding=1) readonly buffer Queries{ vec3 queries[]; };
layout(set=0,binding=2) writeonly buffer Result { uint nearestIdx[]; };
layout(local_size_x=64) in;

void main() {
    uint q   = gl_GlobalInvocationID.x;
    vec3 qpt = queries[q];
    uint best = 0; float bestDist = 1e30;
    // Iterative traversal with implicit stack (depth limited)
    uint stack[32]; int sp = 0;
    stack[sp++] = 1;
    while (sp > 0) {
        uint node = stack[--sp];
        if (node >= NODE_COUNT) continue;
        vec4 n  = nodes[node];
        float d = length(n.xyz - qpt);
        if (d < bestDist) { bestDist = d; best = node; }
        int axis = int(n.w);
        float diff = qpt[axis] - n.xyz[axis];
        uint near = (diff < 0) ? node*2 : node*2+1;
        uint far  = (diff < 0) ? node*2+1 : node*2;
        stack[sp++] = far;
        if (diff*diff < bestDist) stack[sp++] = near;
    }
    nearestIdx[q] = best;
}
```

GPU k-d trees suit static point clouds. For dynamic scenes (particles, agents) the rebuild cost at each frame makes uniform grids preferable.

### 17.2 Uniform Grids and Cell-List Hashing

A **uniform grid** (cell-list / spatial hash) divides space into axis-aligned cells of side length h. For each particle, its cell index is `floor((p - origin) / h)`. A two-pass GPU algorithm builds the structure:

**Pass 1 — Count:** One thread per particle atomically increments `cellCount[cellIndex(p)]`.

**Pass 2 — Prefix scan:** Exclusive scan over `cellCount` → `cellStart`.

**Pass 3 — Scatter:** One thread per particle writes `sortedParticle[atomicAdd(cellOffset[ci], 1)] = pid`.

Neighbour queries iterate the 3×3×3 cells surrounding the query cell — at most 27 cells, bounded iteration count:

```glsl
// neighbor_iter.comp
layout(local_size_x=64) in;
void main() {
    uint i   = gl_GlobalInvocationID.x;
    vec3 pi  = pos[i].xyz;
    ivec3 ci = ivec3(floor((pi - ORIGIN) / H));
    for (int dz=-1; dz<=1; dz++)
    for (int dy=-1; dy<=1; dy++)
    for (int dx=-1; dx<=1; dx++) {
        uint cellIdx = cellHash(ci + ivec3(dx,dy,dz));
        for (uint k = cellStart[cellIdx]; k < cellStart[cellIdx+1]; k++) {
            uint j  = sortedParticle[k];
            vec3 r  = pi - pos[j].xyz;
            float d = dot(r, r);
            if (d < H*H) processNeighbor(i, j, r, d);
        }
    }
}
```

Rebuild cost is O(N) and fits within a single frame budget for N < 1M particles. [Source: Green "Particle Simulation using CUDA", NVIDIA 2010]

### 17.3 Sparse Voxel Octrees (SVO)

An SVO stores only non-empty octree nodes, indexed by Morton (Z-curve) codes. GPU construction (Laine & Karras 2010):

1. **Compute Morton codes** for all triangles (centroid → 30-bit Morton code). Radix sort.
2. **Bottom-up merge:** Threads at level L examine adjacent codes; if they share a common ancestor at level L, mark as one node.
3. **Compact** non-empty nodes into a flat SSBO with parent/child pointers stored as integer offsets.

GPU ray traversal follows the standard octree DDA: descend to the deepest non-empty child overlapping the ray, march forward, ascend when the ray exits a node. A **contour** field at each SDF leaf enables sub-voxel geometry detail. NanoVDB (§3.6) implements a production-quality version of this layout optimised for GPU cache coherence. [Source: Laine & Karras "Efficient Sparse Voxel Octrees", I3D 2010; Crassin et al. "GigaVoxels: Ray-Guided Streaming for Efficient and Detailed Voxel Rendering", I3D 2009]

---

## 18. Spectral Geometry Processing

*Audience: systems developers, graphics application developers.*

Spectral geometry processing applies signal-processing concepts (filtering, frequency decomposition) to triangle meshes by treating the cotangent Laplacian as the mesh's "frequency operator". Low-frequency eigenvectors correspond to smooth, large-scale shape variations; high-frequency eigenvectors capture fine surface detail.

### 18.1 Graph Laplacian Eigenbasis

The graph Laplacian L = D − A (degree matrix minus adjacency) has V eigenvectors Φ = [φ₁, …, φ_V] and eigenvalues Λ = diag(λ₁, …, λ_V). Any per-vertex scalar function f decomposes as f̂ = Φᵀ f. Filtered reconstruction: f' = Φ h(Λ) f̂ where h is the frequency filter (e.g., low-pass: h(λ) = exp(−t λ)).

Full eigenbasis computation (O(V³)) is impractical for large meshes. **Chebyshev polynomial approximation** avoids explicit eigenvectors: any spectral filter h(L) f ≈ Σₖ cₖ Tₖ(L) f, where Tₖ are Chebyshev polynomials evaluated via the recurrence Tₖ(L) f = 2L·Tₖ₋₁(L)f − Tₖ₋₂(L)f. Each step is one sparse matrix-vector product — directly GPU-parallelisable.

[Source: Shuman et al. "The Emerging Field of Signal Processing on Graphs", IEEE Signal Processing Magazine 2013; Defferrard et al. "Convolutional Neural Networks on Graphs with Fast Localized Spectral Filtering", NIPS 2016]

### 18.2 Bilateral Mesh Filtering

Bilateral filtering on meshes (Jones et al. 2003) weights the neighbour contribution by both spatial proximity and normal similarity, preserving sharp features while smoothing noise:

```glsl
// bilateral_filter.comp
layout(set=0,binding=0) readonly buffer InPos  { vec3 inPos[]; };
layout(set=0,binding=1) readonly buffer Normals{ vec3 normals[]; };
layout(set=0,binding=2) writeonly buffer OutPos { vec3 outPos[]; };
layout(set=0,binding=3) readonly buffer Adj    { uint adjStart[]; uint adjList[]; };
layout(local_size_x=64) in;

void main() {
    uint i   = gl_GlobalInvocationID.x;
    vec3 pi  = inPos[i], ni = normals[i];
    vec3 sum = vec3(0); float wsum = 0.0;

    for (uint k = adjStart[i]; k < adjStart[i+1]; k++) {
        uint j  = adjList[k];
        vec3 pj = inPos[j], nj = normals[j];
        float ws = exp(-dot(pi-pj, pi-pj) / (2.0*SIGMA_S*SIGMA_S));
        float wr = exp(-pow(1.0-dot(ni,nj), 2.0) / (2.0*SIGMA_R*SIGMA_R));
        float w  = ws * wr;
        sum  += w * pj; wsum += w;
    }
    outPos[i] = sum / max(wsum, 1e-8);
}
```

3–10 iterations converge. `SIGMA_S` controls spatial spread (typically 2–5× mean edge length), `SIGMA_R` controls feature sharpness (0.1–0.5 radians). [Source: Jones et al. "Non-iterative, Feature-Preserving Mesh Smoothing", SIGGRAPH 2003]

### 18.3 Anisotropic Diffusion and Curvature Flow

**Anisotropic diffusion** (Clarenz et al. 2000) generalises the Laplacian smoother by introducing a per-vertex diffusion tensor D(∇n) that suppresses diffusion across edges (where the normal gradient is large) while amplifying diffusion along edges:

```
dx/dt = div(D(∇n) · ∇x)
```

On a triangle mesh, the per-edge weight replaces the cotangent weight in the Laplacian with w_ij · g(‖nᵢ − nⱼ‖) where g is a decreasing edge-stopping function (e.g., g(s) = 1/(1 + s²/K²)). One CG solve per time step.

**Willmore flow** minimises the total squared mean curvature ∫ H² dA, producing globally fair surfaces. The gradient flow involves fourth-order differential operators (biharmonic) — requires solving L Δx = f where L is the Laplacian squared, achievable as two cascaded CG solves. [Source: Clarenz et al. "Anisotropic Geometric Diffusion in Surface Processing", VIS 2000; Crane et al. "Robust Fairing via Conformal Curvature Flow", SIGGRAPH 2013]

---

## 19. Navigation Meshes and GPU Pathfinding

*Audience: graphics application developers.*

Navigation mesh (navmesh) generation and pathfinding are geometry algorithms in disguise: voxelisation is a rasterization problem, region labelling is a BFS problem on a graph, and flow field computation is a parallel shortest-path problem on a grid — all map to Vulkan compute naturally.

### 19.1 Voxelisation for NavMesh Generation (Recast-style)

Recast's CPU navmesh generation begins with a **solid heightfield**: rasterize each triangle onto a voxel column grid, producing spans of solid voxels. The GPU equivalent uses depth-peeling compute:

```glsl
// voxelise.comp — rasterize triangles into column heightfield
layout(set=0,binding=0) readonly buffer Tris   { TriangleData tris[]; };
layout(set=0,binding=1) buffer Heightfield { Span spans[MAX_CELLS * MAX_SPANS]; uint spanCount[]; };
layout(local_size_x=64) in;

void main() {
    uint t   = gl_GlobalInvocationID.x;
    // Project triangle to XZ grid, find overlapping columns
    vec3 v0 = tris[t].v0, v1 = tris[t].v1, v2 = tris[t].v2;
    ivec2 minCell = ivec2(floor(min(min(v0.xz, v1.xz), v2.xz) / CELL_SIZE));
    ivec2 maxCell = ivec2(ceil(max(max(v0.xz, v1.xz), v2.xz) / CELL_SIZE));
    for (int cz = minCell.y; cz <= maxCell.y; cz++)
    for (int cx = minCell.x; cx <= maxCell.x; cx++) {
        // compute min/max Y of triangle within this column via ray-tri intersection
        float yMin, yMax;
        if (columnTriangleClip(v0, v1, v2, cx, cz, yMin, yMax)) {
            uint sc = atomicAdd(spanCount[cz*GRID_W+cx], 1u);
            spans[(cz*GRID_W+cx)*MAX_SPANS + sc] = Span(yMin, yMax, walkable(v0,v1,v2));
        }
    }
}
```

After voxelisation, a walkability filter checks each span's clearance (vertical space above), slope (triangle normal vs. up), and step height — fully parallel per span.

### 19.2 Parallel Region Labelling via GPU BFS

Recast clusters walkable spans into **regions** for polygon extraction. The GPU equivalent is a parallel iterative BFS flood-fill: each thread checks its 4-connected neighbours and updates its label to the minimum neighbour label if smaller:

```glsl
// region_label.comp
layout(set=0,binding=0) buffer Labels  { uint label[]; };   // init: label[i] = i
layout(set=0,binding=1) buffer Changed { uint changed; };
layout(local_size_x=64) in;
void main() {
    uint i   = gl_GlobalInvocationID.x;
    uint myL = label[i];
    uint minL = myL;
    for each walkable neighbour j of i:
        minL = min(minL, label[j]);
    if (minL < myL) {
        label[i] = minL;
        atomicOr(changed, 1u);
    }
}
```

Iterate until `changed == 0`. Convergence requires O(diameter) iterations (typically 20–50 for game-scale meshes). A path-compression step (like union-find) can accelerate convergence. [Source: Recast navigation: https://github.com/recastnavigation/recastnavigation; parallel label propagation: Soman et al. "Fast Transitive Closure using Graph Partitioning", SC 2011]

### 19.3 GPU Flow Fields for Navigation

A **flow field** precomputes for every cell a steering direction towards the goal, avoiding obstacles. Construction is a parallel BFS from the goal cell, computing a distance field, then differencing adjacent cells to obtain the gradient (steering direction):

```glsl
// flow_field_step.comp — one BFS wavefront expansion
layout(set=0,binding=0) buffer Dist   { uint dist[]; };   // init: goal=0, others=UINT_MAX
layout(set=0,binding=1) buffer Changed{ uint changed; };
layout(push_constant) uniform PC { uint W, H; } pc;
layout(local_size_x=64) in;

void main() {
    uint c = gl_GlobalInvocationID.x;
    if (!walkable[c]) return;
    uint myD = dist[c];
    for each cardinal neighbour n of c:
        if (walkable[n] && dist[n] < myD - 1) {
            atomicMin(dist[c], dist[n] + 1);
            atomicOr(changed, 1u);
        }
}
```

After convergence, a separate pass computes the steering direction at each cell as the gradient of `dist` (direction towards the lowest-distance neighbour). Agents sample their current cell's direction vector to steer without per-agent A* queries.

Flow fields scale to thousands of agents at constant cost per frame (one texture read per agent). Recompute only when the goal changes or obstacles move. [Source: Elijah Emerson "Crowd Pathfinding and Steering Using Flow Field Tiles", Game AI Pro 2013; https://github.com/recastnavigation/recastnavigation detour for reference path finding]

---

## 20. Ray Tracing Geometry Pipeline

*Audience: graphics application developers, systems developers.*

Vulkan ray tracing (VK_KHR_ray_tracing_pipeline, core in Vulkan 1.2+ with KHR extensions) organises geometry into a two-level hierarchy. §8 covered LBVH *construction* as a compute algorithm; this section covers the Vulkan RT *pipeline* as a geometry-consumption interface — acceleration structure management, procedural geometry via intersection shaders, and animated geometry update strategies.

### 20.1 Acceleration Structure Fundamentals: BLAS and TLAS

- **BLAS (Bottom-Level Acceleration Structure):** Contains actual geometry — triangle meshes or AABBs for procedural geometry. Built once per unique mesh shape.
- **TLAS (Top-Level Acceleration Structure):** Contains instances of BLASes with per-instance 3×4 transforms, visibility masks, and shader binding table offsets. Rebuilt each frame for animated scenes.

Queries traverse TLAS → BLAS → geometry test. RT cores (NVIDIA) and BVH hardware (AMD RDNA3+) handle traversal; application provides shader callbacks at each intersection event.

### 20.2 Building BLAS from Vertex/Index Buffers

```cpp
VkAccelerationStructureGeometryKHR geom{};
geom.sType        = VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_GEOMETRY_KHR;
geom.geometryType = VK_GEOMETRY_TYPE_TRIANGLES_KHR;
geom.flags        = VK_GEOMETRY_OPAQUE_BIT_KHR;
auto& tri = geom.geometry.triangles;
tri.sType         = VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_GEOMETRY_TRIANGLES_DATA_KHR;
tri.vertexFormat  = VK_FORMAT_R32G32B32_SFLOAT;
tri.vertexData.deviceAddress = vertexBufferDeviceAddress;
tri.vertexStride  = sizeof(Vertex);
tri.maxVertex     = vertexCount - 1;
tri.indexType     = VK_INDEX_TYPE_UINT32;
tri.indexData.deviceAddress  = indexBufferDeviceAddress;

VkAccelerationStructureBuildGeometryInfoKHR buildInfo{};
buildInfo.sType         = VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_BUILD_GEOMETRY_INFO_KHR;
buildInfo.type          = VK_ACCELERATION_STRUCTURE_TYPE_BOTTOM_LEVEL_KHR;
buildInfo.flags         = VK_BUILD_ACCELERATION_STRUCTURE_PREFER_FAST_TRACE_BIT_KHR;
buildInfo.geometryCount = 1;
buildInfo.pGeometries   = &geom;

VkAccelerationStructureBuildSizesInfoKHR sizeInfo{};
uint32_t primCount = triangleCount;
vkGetAccelerationStructureBuildSizesKHR(device,
    VK_ACCELERATION_STRUCTURE_BUILD_TYPE_DEVICE_KHR,
    &buildInfo, &primCount, &sizeInfo);
// Allocate result and scratch buffers, create AS handle, then:
buildInfo.mode = VK_BUILD_ACCELERATION_STRUCTURE_MODE_BUILD_KHR;
buildInfo.dstAccelerationStructure  = blas;
buildInfo.scratchData.deviceAddress = scratchAddress;
VkAccelerationStructureBuildRangeInfoKHR range{triangleCount, 0, 0, 0};
const auto* pRange = &range;
vkCmdBuildAccelerationStructuresKHR(cmd, 1, &buildInfo, &pRange);
```

For multiple submeshes in one BLAS, pass an array of `VkAccelerationStructureGeometryKHR` and corresponding `VkAccelerationStructureBuildRangeInfoKHR` — one per submesh, different hit group offsets for different materials. [Source: Vulkan Spec §37.9; NVIDIA Vulkan Ray Tracing Tutorial https://nvpro-samples.github.io/vk_raytracing_tutorial_KHR/]

### 20.3 TLAS and Instance Transforms

```cpp
VkAccelerationStructureInstanceKHR inst{};
memcpy(inst.transform.matrix, glm::value_ptr(glm::transpose(modelMatrix)),
       sizeof(inst.transform.matrix));          // row-major 3×4
inst.instanceCustomIndex                    = objectID;          // gl_InstanceCustomIndexEXT
inst.mask                                   = 0xFF;
inst.instanceShaderBindingTableRecordOffset = hitGroupIndex;
inst.flags = VK_GEOMETRY_INSTANCE_TRIANGLE_FACING_CULL_DISABLE_BIT_KHR;
inst.accelerationStructureReference         = blasDeviceAddress;
```

For dynamic scenes, rebuild the TLAS each frame (instances are just a flat buffer of 64-byte structs — fast) while BLASes stay fixed unless mesh topology changes. For skinned meshes use BLAS refit (§20.5) then TLAS rebuild.

### 20.4 Procedural Geometry via Intersection Shaders

Geometry that cannot be represented as triangle soups (analytic spheres, SDF primitives, Bézier patches) uses AABB-based BLASes and a custom intersection shader. The AABB defines the bounding volume; the intersection shader computes the exact hit point:

```glsl
// sphere.rint — analytic sphere intersection
#version 460
#extension GL_EXT_ray_tracing : require

struct SphereData { vec3 center; float radius; };
layout(set=0,binding=2) readonly buffer Spheres { SphereData spheres[]; };

void main() {
    SphereData s = spheres[gl_PrimitiveID];
    vec3  oc = gl_WorldRayOriginEXT - s.center;
    float a  = dot(gl_WorldRayDirectionEXT, gl_WorldRayDirectionEXT);
    float b  = 2.0 * dot(oc, gl_WorldRayDirectionEXT);
    float c  = dot(oc, oc) - s.radius * s.radius;
    float disc = b*b - 4.0*a*c;
    if (disc < 0.0) return;
    float t = (-b - sqrt(disc)) / (2.0 * a);
    if (t < gl_RayTminEXT || t > gl_RayTmaxEXT) {
        t = (-b + sqrt(disc)) / (2.0 * a);
        if (t < gl_RayTminEXT || t > gl_RayTmaxEXT) return;
    }
    reportIntersectionEXT(t, 0u);
}
```

SDF sphere tracing (§3.7) runs as a hybrid: coarse BVH traversal from RT hardware, fine sphere-march inside the custom intersection shader — combining hardware AS traversal efficiency with analytical SDF accuracy. [Source: Vulkan Spec §14.7 Ray Tracing Shaders; Quilez "Raymarching Signed Distance Fields"]

### 20.5 BLAS Refit for Animated Geometry

When vertex positions change (skinning, cloth, fluid surface) but triangle topology stays constant, **refit** updates the BLAS tree in O(N) rather than rebuilding in O(N log N):

```cpp
// First frame: build with ALLOW_UPDATE flag
buildInfo.flags = VK_BUILD_ACCELERATION_STRUCTURE_PREFER_FAST_BUILD_BIT_KHR |
                  VK_BUILD_ACCELERATION_STRUCTURE_ALLOW_UPDATE_BIT_KHR;
buildInfo.mode  = VK_BUILD_ACCELERATION_STRUCTURE_MODE_BUILD_KHR;
// ...build...

// Each subsequent frame after skinning pre-pass (§4.4):
buildInfo.mode    = VK_BUILD_ACCELERATION_STRUCTURE_MODE_UPDATE_KHR;
buildInfo.srcAccelerationStructure = blas;   // refit in place
buildInfo.dstAccelerationStructure = blas;
vkCmdBuildAccelerationStructuresKHR(cmd, 1, &buildInfo, &pRange);
```

Refit traverses the existing BLAS bottom-up recomputing AABBs from updated vertex positions without re-sorting primitives. Quality degrades over many frames as the tree drifts from optimal. A common strategy: refit for K frames, then full rebuild every K+1th frame. [Source: Wyman et al. "Introduction to DirectX Raytracing" §3; NVIDIA Vulkan RT best practices]

---

## 21. Point Cloud Processing

*Audience: systems developers, graphics application developers.*

Point clouds are the direct output of depth cameras (structured light, LiDAR, stereo), photogrammetry pipelines, and 3DGS/NeRF density fields. GPU point cloud processing bridges raw sensor data and the mesh/SDF representations used throughout this chapter.

### 21.1 GPU k-NN via Tiled Brute-Force

For N ≤ 100k points and k ≤ 32, a **tiled brute-force** approach exploiting shared memory beats tree traversal by eliminating branch divergence:

```glsl
// knn_brute.comp — k=16 nearest neighbours, TILE=64
layout(set=0,binding=0) readonly buffer Points { vec4 pts[]; };
layout(set=0,binding=1) writeonly buffer KNN   { uint knn[16 * N_POINTS]; };
layout(local_size_x=64) in;
shared vec4 tile[64];

void main() {
    uint i = gl_GlobalInvocationID.x;
    vec3 pi = pts[i].xyz;
    float dist[16]; uint idx[16];
    for (int j=0; j<16; j++) { dist[j]=1e30; idx[j]=0; }

    for (uint base=0; base<N_POINTS; base+=64) {
        tile[gl_LocalInvocationID.x] = pts[base + gl_LocalInvocationID.x];
        barrier();
        for (uint t=0; t<64; t++) {
            float d = length(pi - tile[t].xyz);
            if (d < dist[15]) {
                dist[15]=d; idx[15]=base+t;
                for (int s=14; s>=0 && dist[s]>dist[s+1]; s--)
                    { float td=dist[s]; dist[s]=dist[s+1]; dist[s+1]=td;
                      uint ti=idx[s];  idx[s]=idx[s+1];   idx[s+1]=ti; }
            }
        }
        barrier();
    }
    for (int j=0; j<16; j++) knn[i*16+j]=idx[j];
}
```

For larger clouds use the GPU k-d tree (§17.1) or uniform grid radius search (§17.2). [Source: Garcia et al. "Fast k Nearest Neighbor Search Using GPU", CVPR Workshop 2008]

### 21.2 Normal Estimation from k-NN

Per-point normals are the smallest eigenvector of the 3×3 neighbourhood covariance matrix. Each thread computes its local covariance and solves the 3×3 symmetric eigenproblem via cyclic Jacobi iteration:

```glsl
// normal_est.comp
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    vec3 pi = pts[i].xyz;
    vec3 mean = vec3(0);
    for (int j=0; j<K; j++) mean += pts[knn[i*K+j]].xyz;
    mean /= float(K);
    mat3 C = mat3(0.0);
    for (int j=0; j<K; j++) {
        vec3 d = pts[knn[i*K+j]].xyz - mean;
        C += outerProduct(d,d);
    }
    // 3x3 symmetric Jacobi eigensolver (5-6 sweeps)
    mat3 V; vec3 lambda;
    jacobiEigen3x3(C, V, lambda);  // lambda[0] ≤ lambda[1] ≤ lambda[2]
    normals[i] = vec4(V[0], 0.0);  // normal = eigenvector of smallest eigenvalue
}
```

Orientation consistency (pointing toward the sensor) requires a minimum spanning tree propagation — a CPU post-pass for static clouds. [Source: Rusu "Semantic 3D Object Maps for Everyday Manipulation", PhD 2009]

### 21.3 GPU ICP Registration

**Iterative Closest Point** (Besl & McKay 1992) aligns source cloud P to target Q. Each iteration: (1) GPU k-NN finds nearest neighbours; (2) GPU reduction accumulates the 3×3 cross-covariance H = Σ(pᵢ−μₚ)(qᵢ−μ_q)ᵀ; (3) CPU SVD of H gives R = V Uᵀ; (4) apply R,t and repeat.

```glsl
// icp_reduce.comp — accumulate cross-covariance and means per workgroup
layout(set=0,binding=0) readonly buffer Src { vec4 src[]; };
layout(set=0,binding=1) readonly buffer NN  { uint nn[]; };
layout(set=0,binding=2) readonly buffer Tgt { vec4 tgt[]; };
layout(set=0,binding=3) buffer Reduction { float H[9]; float meanP[3]; float meanQ[3]; };
layout(local_size_x=64) in;
shared mat3 sH[64]; shared vec3 sMp[64], sMq[64];
void main() {
    uint i=gl_GlobalInvocationID.x;
    vec3 p=src[i].xyz, q=tgt[nn[i]].xyz;
    sH[gl_LocalInvocationID.x]=outerProduct(p,q);
    sMp[gl_LocalInvocationID.x]=p; sMq[gl_LocalInvocationID.x]=q;
    barrier();
    for (uint s=32; s>0; s>>=1) {
        if (gl_LocalInvocationID.x<s) {
            sH[gl_LocalInvocationID.x]  += sH[gl_LocalInvocationID.x+s];
            sMp[gl_LocalInvocationID.x] += sMp[gl_LocalInvocationID.x+s];
            sMq[gl_LocalInvocationID.x] += sMq[gl_LocalInvocationID.x+s];
        }
        barrier();
    }
    if (gl_LocalInvocationID.x==0) {
        for (int r=0;r<3;r++) for (int c=0;c<3;c++)
            atomicAdd(Reduction.H[r*3+c], sH[0][r][c]);
        for (int r=0;r<3;r++) {
            atomicAdd(Reduction.meanP[r], sMp[0][r]);
            atomicAdd(Reduction.meanQ[r], sMq[0][r]);
        }
    }
}
```

Point-to-plane ICP (project residual onto surface normal) converges 3× faster and has a closed-form 6×6 linear system solve per iteration. [Source: Besl & McKay "A Method for Registration of 3-D Shapes", PAMI 1992; Chen & Medioni "Object Modeling by Registration of Multiple Range Images", IVC 1992]

### 21.4 GPU RANSAC Primitive Fitting

RANSAC scores each candidate model against all points in parallel. Generate N candidates (plane, sphere, cylinder) from random minimal subsets; dispatch one kernel to count inliers for all N simultaneously:

```glsl
// ransac_score.comp — score N candidate planes against M points
layout(set=0,binding=0) readonly buffer Points     { vec4 pts[]; };     // M points
layout(set=0,binding=1) readonly buffer Candidates { vec4 planes[]; };  // N (n,d) planes
layout(set=0,binding=2) buffer InlierCounts { uint counts[]; };          // N counters
layout(push_constant) uniform PC { float threshold; } pc;
layout(local_size_x=64) in;
void main() {
    uint p=gl_GlobalInvocationID.x % M_POINTS;
    uint c=gl_GlobalInvocationID.x / M_POINTS;
    float dist = abs(dot(pts[p].xyz, Candidates[c].xyz) + Candidates[c].w);
    if (dist < pc.threshold) atomicAdd(counts[c], 1u);
}
```

Dispatch with `(N*M/64, 1, 1)` workgroups — all N×M scores evaluate in one pass. Read back the winning candidate index. [Source: Fischler & Bolles "Random Sample Consensus", CACM 1981; Schnabel et al. "Efficient RANSAC for Point-Cloud Shape Detection", CGF 2007]

### 21.5 Euclidean Segmentation via Label Propagation

Cluster points within distance δ using the same iterative label-propagation pattern from §19.2 and §22.2, but in 3D over a spatial hash grid:

```glsl
// seg_label.comp
layout(local_size_x=64) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    uint lm = label[i];
    ivec3 ci = gridCell(pts[i].xyz);
    for (int dz=-1;dz<=1;dz++) for(int dy=-1;dy<=1;dy++) for(int dx=-1;dx<=1;dx++) {
        uint cell = hashCell(ci+ivec3(dx,dy,dz));
        for (uint k=gridStart[cell]; k<gridEnd[cell]; k++) {
            uint j = gridPts[k];
            if (length(pts[i].xyz-pts[j].xyz) < DELTA)
                lm = min(lm, label[j]);
        }
    }
    if (lm < label[i]) { label[i]=lm; atomicOr(changed,1u); }
}
```

Iterate until `changed==0`. Compact to dense cluster IDs via prefix scan. [Source: Rusu et al. "Towards 3D Point Cloud Based Object Maps", Robotics and Autonomous Systems 2008]

---

## 22. Rigid Body Dynamics on GPU

*Audience: graphics application developers, systems developers.*

PhysX 5 (§30 Library Landscape Table B) provides a production GPU rigid body solver; this section covers the algorithms so implementors can build Vulkan-native physics or understand what PhysX does under the hood.

### 22.1 Broadphase: SAP Pair Generation

Sweep-and-prune (SAP) sorts AABB extents along one axis; overlapping intervals on that axis are candidate pairs, checked against the other two axes:

```glsl
// sap_pairs.comp — overlapping AABB pairs from sorted X-extents
layout(set=0,binding=0) readonly buffer SortedMin { float minX[]; uint objID[]; };
layout(set=0,binding=1) readonly buffer MaxX      { float maxX[]; };
layout(set=0,binding=2) buffer Pairs     { uvec2 pairs[]; };
layout(set=0,binding=3) buffer PairCount { uint count; };
layout(local_size_x=64) in;
void main() {
    uint i=gl_GlobalInvocationID.x;
    uint a=objID[i];
    for (uint j=i+1; j<N_OBJECTS && minX[j]<=maxX[a]; j++) {
        uint b=objID[j];
        if (aabbOverlap(aabb[a], aabb[b])) {
            uint slot=atomicAdd(count,1u);
            pairs[slot]=uvec2(a,b);
        }
    }
}
```

Pairs feed the island detection and narrowphase (§8.4 SAT/GJK). [Source: Tonge "Iterative Rigid Body Simulation", GDC 2013]

### 22.2 Island Detection via GPU Label Propagation

Rigid body simulation islands (connected components of the contact graph) must be solved together. Parallel label propagation over active contact pairs:

```glsl
// island_label.comp
layout(set=0,binding=0) readonly buffer Contacts { uvec2 contacts[]; };
layout(set=0,binding=1) buffer Labels  { uint label[]; };
layout(set=0,binding=2) buffer Changed { uint changed; };
layout(local_size_x=64) in;
void main() {
    uint c=gl_GlobalInvocationID.x;
    uint a=contacts[c].x, b=contacts[c].y;
    uint lm=min(label[a],label[b]);
    if (label[a]!=lm) { atomicMin(label[a],lm); atomicOr(changed,1u); }
    if (label[b]!=lm) { atomicMin(label[b],lm); atomicOr(changed,1u); }
}
```

After convergence, islands with disjoint label values dispatch independently in parallel.

### 22.3 Sequential Impulse Solver with Graph Colouring

The sequential impulse (SI) solver applies corrective velocity impulses at each contact. GPU parallelism requires contacts sharing no body to run in the same pass — enforced by graph colouring (4–8 colours suffice for most scenes):

```glsl
// si_solve.comp — one colour pass
layout(set=0,binding=0) buffer Contacts { ContactData c[]; };
layout(set=0,binding=1) buffer BodyVel  { BodyVelocity vel[]; };
layout(local_size_x=64) in;
void main() {
    uint ci=gl_GlobalInvocationID.x;
    ContactData co=c[ci];
    vec3 vrel=linearVelAtPoint(vel[co.bodyA],co.rA)
             -linearVelAtPoint(vel[co.bodyB],co.rB);
    float vn=dot(vrel,co.normal);
    float dL=-(vn+co.bias)/co.effectiveMass;
    dL=max(co.lambda+dL,0.0)-co.lambda;
    c[ci].lambda+=dL;
    applyImpulse(vel[co.bodyA],+dL,co.normal,co.rA,co.invMassA,co.invInertiaA);
    applyImpulse(vel[co.bodyB],-dL,co.normal,co.rB,co.invMassB,co.invInertiaB);
}
```

Run 4–20 iterations; each iteration dispatches one kernel per colour. [Source: Catto "Iterative Dynamics with Temporal Coherence", GDC 2005; Tonge "Solving Rigid Body Contacts", GDC 2013]

### 22.4 Featherstone Articulated Body Algorithm

For articulated bodies (ragdolls, robot arms) the Articulated Body Algorithm (Featherstone 1987) propagates joint quantities in three O(n)-per-chain passes. Independent chains (multiple ragdolls) execute in parallel; joints within one chain are serialised by their topological depth:

```glsl
// aba_pass1.comp — root-to-leaf velocity propagation (one joint per thread)
layout(local_size_x=64) in;
void main() {
    uint j=gl_GlobalInvocationID.x;
    uint p=parent[j];
    if (p==INVALID) { v[j]=externalVel[j]; return; }
    // 6D spatial velocity: v[j] = X[j]*v[p] + S[j]*qdot[j]
    v[j]=spatialTransform(X[j],v[p]) + motionSubspace[j]*qdot[j];
}
// Pass 2 (leaves→root): articulated inertia recursion (reverse topological)
// Pass 3 (root→leaves): acceleration propagation
// Each pass is one dispatch with a barrier before the next.
```

[Source: Featherstone "Rigid Body Dynamics Algorithms", 2008; GPU implementation in NVIDIA PhysX 5 articulation solver]

---

## 23. Crowd and Multi-Agent Simulation

*Audience: graphics application developers.*

Navigation meshes (§19) compute flow fields telling each agent which direction to move. This section covers the per-agent algorithms that consume those fields and avoid collisions with other agents at run time.

### 23.1 ORCA: Optimal Reciprocal Collision Avoidance

ORCA (van den Berg et al. 2011) computes a collision-free velocity for each agent by solving a small 2D linear programme over velocity-obstacle half-planes from nearby neighbours:

```glsl
// orca.comp
layout(set=0,binding=0) readonly buffer Agents { AgentData a[]; };
layout(set=0,binding=1) writeonly buffer NewVel { vec2 nv[]; };
layout(local_size_x=64) in;

struct HalfPlane { vec2 point, normal; };

void main() {
    uint i=gl_GlobalInvocationID.x;
    vec2 pi=a[i].pos, vi=a[i].vel; float ri=a[i].radius;
    HalfPlane planes[MAX_NEIGHBOURS]; uint nPlanes=0;

    // iterate nearby agents via spatial hash (§17.2)
    ivec2 ci=ivec2(floor(pi/CELL_SIZE));
    for (int dy=-1;dy<=1;dy++) for(int dx=-1;dx<=1;dx++) {
        uint cell=hash2D(ci+ivec2(dx,dy));
        for (uint k=gridStart[cell];k<gridEnd[cell];k++) {
            uint j=gridAgents[k]; if(j==i) continue;
            vec2 relPos=a[j].pos-pi, relVel=vi-a[j].vel;
            float combR=ri+a[j].radius;
            // compute velocity obstacle boundary and closest-point offset u
            vec2 w=relVel-relPos/ORCA_TAU;
            float wlen=length(w);
            vec2 u=(combR/ORCA_TAU-wlen)*w/max(wlen,1e-6);
            planes[nPlanes++]=HalfPlane(vi+0.5*u, normalize(u));
            if(nPlanes>=MAX_NEIGHBOURS) break;
        }
    }
    nv[i]=solveLP2D(planes, nPlanes, a[i].prefVel, a[i].maxSpeed);
}
```

`solveLP2D` is an incremental 2D LP running entirely in registers — O(k) expected per agent. [Source: van den Berg et al. "Reciprocal n-Body Collision Avoidance", ISRR 2011; RVO2 https://gamma.cs.unc.edu/RVO2/]

### 23.2 Reynolds Boid Flocking

Three-rule boid steering (separation, alignment, cohesion) over a spatial-hash neighbourhood:

```glsl
// boids.comp
layout(local_size_x=64) in;
void main() {
    uint i=gl_GlobalInvocationID.x;
    vec3 pi=pos[i], vi=vel[i];
    vec3 sep=vec3(0), ali=vec3(0), coh=vec3(0); uint n=0;
    // iterate neighbours within BOID_RADIUS via §17.2 grid
    for each neighbour j: {
        vec3 d=pi-pos[j];
        sep+=normalize(d)/max(length(d),0.01);
        ali+=vel[j]; coh+=pos[j]; n++;
    }
    if(n>0){ali=normalize(ali/float(n)); coh=normalize(coh/float(n)-pi);}
    vec3 steer=W_SEP*sep+W_ALI*ali+W_COH*coh+W_GOAL*normalize(goal-pi);
    vel[i]=normalize(vi+steer*DT)*BOID_SPEED;
    pos[i]=pi+vel[i]*DT;
}
```

[Source: Reynolds "Flocks, Herds, and Schools", SIGGRAPH 1987]

### 23.3 Agent Steering over Flow Fields

Blend ORCA collision-avoidance velocity with flow-field global direction:

```glsl
// steer_integrate.comp
layout(local_size_x=64) in;
void main() {
    uint i=gl_GlobalInvocationID.x;
    ivec2 ci=ivec2(floor(pos[i].xz/CELL_SIZE));
    vec2 flow=flowField[ci.y*GRID_W+ci.x].dir;   // §19.3
    vec2 orca=newVel[i];                            // §23.1
    vec2 steer=normalize(mix(flow,orca,ORCA_BLEND))*MAX_SPEED;
    vel[i].xz=steer;
    pos[i]+=vel[i]*DT;
}
```

### 23.4 GPU Crowd Rendering via Task Shaders

Each agent selects a pre-skinned LOD meshlet cluster; the task shader culls and emits only visible agents:

```glsl
// crowd_task.glsl
layout(local_size_x=32) in;
taskPayloadSharedEXT struct { uint agentIdx[32]; uint count; } payload;
void main() {
    uint i=gl_GlobalInvocationID.x;
    if(gl_LocalInvocationID.x==0) payload.count=0;
    barrier();
    AgentData ag=agents[i];
    bool visible=sphereInFrustum(ag.pos, AGENT_RADIUS)
              && length(ag.pos-cameraPos)<CULL_DIST;
    if(visible){ uint slot=atomicAdd(payload.count,1u); payload.agentIdx[slot]=i; }
    barrier();
    if(gl_LocalInvocationID.x==0) EmitMeshTasksEXT(payload.count,1,1);
}
```

Distant agents use billboard impostors (§7.7); close agents use full animated meshlets (§10.1–10.2). [Source: Reynolds 1987; van den Berg et al. 2011]

---

## 24. GPU Remeshing

*Audience: graphics application developers, systems developers.*

Remeshing reshapes mesh geometry — changing edge lengths, valences, and alignment — without changing the underlying surface. It sits between simplification (§7, which reduces vertex count) and repair (§7.14, which fixes pathological topology).

### 24.1 Isotropic Remeshing

Botsch–Kobbelt isotropic remeshing (2004) iterates five operations targeting a uniform edge length ℓ. Operations 1–3 modify topology and require graph-coloured parallel dispatch; operations 4–5 are purely per-vertex:

**Operation 4 — Tangential relaxation** (fully parallel):

```glsl
// tangential_relax.comp
layout(set=0,binding=0) readonly buffer Positions { vec3 pos[]; };
layout(set=0,binding=1) readonly buffer Normals   { vec3 nor[]; };
layout(set=0,binding=2) readonly buffer Adj       { uint adjStart[]; uint adjList[]; };
layout(set=0,binding=3) writeonly buffer NewPos   { vec3 newpos[]; };
layout(local_size_x=64) in;
void main() {
    uint i=gl_GlobalInvocationID.x;
    vec3 c=vec3(0); uint n=0;
    for (uint k=adjStart[i];k<adjStart[i+1];k++) { c+=pos[adjList[k]]; n++; }
    c/=float(n);
    vec3 ni=nor[i];
    // project centroid onto tangent plane
    newpos[i]=pos[i]+(c-pos[i])-dot(c-pos[i],ni)*ni;
}
```

**Operations 1–3** (edge split/collapse/flip) use graph-coloured dispatches: colour edges so no two same-colour edges share a vertex. Split and collapse use `atomicAdd` into a free-list counter for new vertex/index slots. [Source: Botsch & Kobbelt "A Remeshing Approach to Multiresolution Modeling", SGP 2004]

### 24.2 Anisotropic Curvature-Aligned Remeshing

Aligns quad/triangle edges to principal curvature directions (§14.4). Pipeline:

1. Compute principal curvature directions per vertex (§14.4 GPU kernel).
2. Smooth a cross-field (4-RoSy) by solving ∇²θ=0 on the mesh (sparse CG, same Laplacian as §14.1).
3. Trace streamlines along cross-field directions via RK4 on-surface integration.
4. Extract quad mesh from streamline network intersections (CPU topology extraction).

Steps 1–3 run on GPU; step 4 is CPU-side. [Source: Bommes et al. "Mixed-Integer Quadrangulation", SIGGRAPH 2009]

### 24.3 Incremental Delaunay Refinement for FEM Meshes

Generating volumetric tet meshes with quality guarantees (minimum dihedral angle > θ) uses Delaunay refinement (Shewchuk 1998): identify poor-quality tets (circumradius/shortest-edge > threshold), insert circumcenter, re-triangulate the star cavity. GPU parallelism: independently bad tets whose circumcenters do not spatially conflict process simultaneously; the §17.2 grid detects conflicts. [Source: Shewchuk "Tetrahedral Mesh Generation by Delaunay Refinement", SoCG 1998]

---

## 25. Fracture and Destruction

*Audience: graphics application developers.*

Fracture combines geometry processing (§3, §7, §12) with physics (§22) to produce visually plausible breaking and crumbling. GPU implementations pre-fracture offline for performance-critical paths and use runtime SDF crack propagation for interactive destruction.

### 25.1 Voronoi Pre-Fracture

Pre-computed Voronoi fracture assigns each voxel to its nearest fracture seed using 3D JFA (§3.9 extended to 3D). Each Voronoi cell becomes a convex fragment:

```glsl
// voronoi_fracture_3d.comp — 3D JFA step
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
layout(set=0,binding=0) buffer SeedMap { ivec4 seedMap[]; };  // xyz=nearest seed, w=unused
void main() {
    ivec3 vox=ivec3(gl_GlobalInvocationID);
    ivec4 best=seedMap[flatIdx(vox)]; float bestDist=1e30;
    for(int dz=-1;dz<=1;dz++) for(int dy=-1;dy<=1;dy++) for(int dx=-1;dx<=1;dx++) {
        ivec3 n=clamp(vox+ivec3(dx,dy,dz)*JFA_STEP, ivec3(0), GRID_SIZE-1);
        ivec4 s=seedMap[flatIdx(n)];
        float d=length(vec3(vox-s.xyz));
        if(d<bestDist){bestDist=d; best=s;}
    }
    seedMap[flatIdx(vox)]=best;
}
```

After JFA, run GPU marching cubes (§3.2) on boundaries between adjacent seed regions to extract per-fragment surface meshes. Each fragment becomes an independent BLAS (§20.2) and rigid body (§22). [Source: Müller et al. "Real Time Dynamic Fracture with Volumetric Approximate Convex Decompositions", SIGGRAPH 2013]

### 25.2 Runtime Brittle Fracture via SDF Crack Propagation

Crack fronts advance as an SDF in a 3D grid, driven by a stress threshold:

```glsl
// crack_advance.comp
layout(set=0,binding=0) buffer CrackSDF { float sdf[]; };
layout(set=0,binding=1) readonly buffer Stress { float stress[]; };
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    uint v=flatIdx(ivec3(gl_GlobalInvocationID));
    if(sdf[v]>0.0 && sdf[v]<CRACK_SEARCH_RADIUS && stress[v]>FRACTURE_THRESHOLD)
        sdf[v]=-1.0;  // crack this voxel
}
```

Re-extract crack surface with GPU MC after each advance step; add new triangle geometry to the BLAS. [Source: Parker & O'Brien "Real-Time Deformation and Fracture in a Game Context", SCA 2009]

### 25.3 FEM Ductile Fracture

Per-element failure criterion on the corotational FEM from §12.2:

```glsl
// fem_fracture_check.comp
layout(set=0,binding=0) readonly buffer Stress { mat3 cauchyStress[]; };
layout(set=0,binding=1) buffer Failed { uint failedBits[]; };
layout(local_size_x=64) in;
void main() {
    uint t=gl_GlobalInvocationID.x;
    float maxEig=maxEigenvalue3x3sym(cauchyStress[t]);
    if(maxEig>FRACTURE_STRESS)
        atomicOr(failedBits[t/32], 1u<<(t%32));
}
```

Compact failed elements, patch crack surface (dual MC on failure boundary), add patch to BLAS. [Source: Irving et al. "Invertible Finite Elements for Robust Simulation", SCA 2004]

---

## 26. Terrain LOD and Streaming

*Audience: graphics application developers.*

§13 covered procedural terrain *generation*. This section covers runtime terrain *rendering*: quadtree LOD selection, geomorphing to prevent popping, virtual texture streaming, and GPU heightfield collision.

### 26.1 CDLOD: Continuous Distance-Dependent Level of Detail

CDLOD (Strugar 2010) traverses a heightmap quadtree in compute, selecting per-node LOD based on projected screen-space error:

```glsl
// cdlod_traverse.comp
struct QuadNode { vec2 min, max; int level; };
layout(set=0,binding=0) readonly buffer QuadTree { QuadNode nodes[]; };
layout(set=0,binding=1) writeonly buffer DrawList { uint ids[]; };
layout(set=0,binding=2) buffer DrawCount { uint count; };
layout(local_size_x=64) in;

float screenError(QuadNode n, vec3 cam) {
    float dist=max(0.0,length(cam.xz-(n.min+n.max)*0.5)-length(n.max-n.min)*0.5);
    return ERROR_SCALE*float(1<<n.level)/max(dist,0.001);
}
void main() {
    uint id=gl_GlobalInvocationID.x;
    QuadNode n=nodes[id];
    if(!nodeInFrustum(n)) return;
    if(n.level==0 || screenError(n,camPos)<SCREEN_ERROR_THRESHOLD) {
        uint slot=atomicAdd(count,1u); ids[slot]=id;
    }
}
```

Selected nodes drive `vkCmdDrawIndirect`. [Source: Strugar "Continuous Distance-Dependent Level of Detail for Rendering Heightmaps", JGT 2010]

### 26.2 Geomorphing to Prevent LOD Popping

Vertices blend between fine and coarse grid positions as a node enters/leaves its LOD range:

```glsl
// terrain.vert
float morphFactor(float dist, float mStart, float mEnd) {
    return clamp((dist-mStart)/(mEnd-mStart), 0.0, 1.0);
}
void main() {
    vec2 uv      = (aPos.xz - nodeMin) / nodeSize;
    float hFine  = texture(heightmap, uv).r * HEIGHT_SCALE;
    float hCoarse= sampleSnappedHeight(uv);  // nearest even-grid sample
    float mf     = morphFactor(length(aPos-camPos), morphStart, morphEnd);
    float h      = mix(hFine, hCoarse, mf);
    gl_Position  = vp * vec4(aPos.x, h, aPos.z, 1.0);
}
```

### 26.3 Virtual Terrain Texturing: Sparse Clipmaps

A clipmap stores terrain texture as a stack of annular rings centred on the camera, each ring at a progressively coarser resolution. When the camera moves, GPU compute fills new tiles into the ring boundary:

```glsl
// clipmap_update.comp
layout(set=0,binding=0, rgba8) uniform writeonly image2DArray clipmap;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec3 coord=ivec3(gl_GlobalInvocationID.xy, CURRENT_LEVEL);
    vec2  uv=(vec2(coord.xy)+clipmapOffset[CURRENT_LEVEL])*texelScale[CURRENT_LEVEL];
    imageStore(clipmap, coord, sampleTerrainTex(uv));
}
```

[Source: Tanner et al. "The Clipmap: A Virtual Mipmap", SIGGRAPH 1998; GPU Gems 2 §2]

### 26.4 GPU Heightfield Ray Intersection

Binary search along the ray's XZ projection finds the terrain hit point for physics queries (vehicle wheels, character stepping):

```glsl
// heightfield_ray.comp
layout(local_size_x=64) in;
void main() {
    uint r=gl_GlobalInvocationID.x;
    vec3 o=rayOrigin[r], d=rayDir[r];
    float tLo=0.0, tHi=MAX_T;
    for(int i=0; i<20; i++) {
        float tMid=(tLo+tHi)*0.5;
        vec3 p=o+tMid*d;
        float h=texture(heightmap, p.xz/TERRAIN_SIZE).r*HEIGHT_SCALE;
        if(p.y<h) tHi=tMid; else tLo=tMid;
    }
    hitT[r]=(tLo+tHi)*0.5;
    hitNorm[r]=terrainNormal(o+hitT[r]*d);
}
```

---

## 27. Geometric Deep Learning

*Audience: graphics application developers, systems developers.*

Geometric deep learning runs neural network inference directly on point clouds and triangle meshes, enabling semantic segmentation, shape completion, and neural implicit surface extraction on the same GPU that renders the scene.

### 27.1 PointNet on GPU

PointNet (Qi et al. 2017) applies a shared MLP per point then aggregates via symmetric max-pool. The per-point MLP is fully parallel — one thread per point per layer:

```glsl
// pointnet_mlp.comp — one fully-connected layer
layout(set=0,binding=0) readonly buffer InFeat  { float inF[N_POINTS*IN_DIM]; };
layout(set=0,binding=1) readonly buffer Weights { float W[IN_DIM*OUT_DIM]; float B[OUT_DIM]; };
layout(set=0,binding=2) writeonly buffer OutFeat { float outF[N_POINTS*OUT_DIM]; };
layout(local_size_x=64) in;
void main() {
    uint p=gl_GlobalInvocationID.x;
    for(uint o=0; o<OUT_DIM; o++) {
        float acc=B[o];
        for(uint i=0; i<IN_DIM; i++) acc+=inF[p*IN_DIM+i]*W[i*OUT_DIM+o];
        outF[p*OUT_DIM+o]=max(acc,0.0);  // ReLU
    }
}
```

Global max-pool over all points aggregates per-point features into a global descriptor via a parallel reduce (same pattern as §11.3). [Source: Qi et al. "PointNet", CVPR 2017; https://github.com/charlesq34/pointnet]

### 27.2 Graph Convolution on Mesh Edge Graphs

Graph neural network message passing on mesh edges — structurally identical to the cotangent Laplacian evaluation (§14.1) with learned weights:

```glsl
// graph_conv.comp — one GNN message-passing layer
layout(set=0,binding=0) readonly buffer NodeFeat { float nf[N_VERTS*FEAT_DIM]; };
layout(set=0,binding=1) readonly buffer Edges    { uvec2 edges[N_EDGES]; };
layout(set=0,binding=2) readonly buffer EdgeW    { float ew[N_EDGES*FEAT_DIM*FEAT_DIM]; };
layout(set=0,binding=3) buffer AggFeat { float agg[N_VERTS*FEAT_DIM]; };
layout(local_size_x=64) in;
void main() {
    uint e=gl_GlobalInvocationID.x;
    uint i=edges[e].x, j=edges[e].y;
    for(uint f=0; f<FEAT_DIM; f++) {
        float msg=0.0;
        for(uint g=0; g<FEAT_DIM; g++)
            msg+=ew[e*FEAT_DIM*FEAT_DIM+f*FEAT_DIM+g]*nf[j*FEAT_DIM+g];
        atomicAdd(agg[i*FEAT_DIM+f], msg);   // VK_EXT_shader_atomic_float
    }
}
```

Supports ChebNet, GAT (Graph Attention Networks), and DGCNN with minor modifications. [Source: Kipf & Welling "Semi-Supervised Classification with GCNs", ICLR 2017; Wang et al. "Dynamic Graph CNN for Learning on Point Clouds", SIGGRAPH 2019]

### 27.3 Neural SDF Inference on GPU

Evaluate a two-layer MLP at each voxel of a 256³ grid in parallel, then extract the isosurface with GPU MC (§3.2):

```glsl
// neural_sdf.comp
layout(set=0,binding=0) readonly buffer W1 { float w1[IN_DIM*H_DIM]; float b1[H_DIM]; };
layout(set=0,binding=1) readonly buffer W2 { float w2[H_DIM];         float b2;        };
layout(set=0,binding=2, r32f) uniform writeonly image3D sdfGrid;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 vox=ivec3(gl_GlobalInvocationID);
    vec3  x=(vec3(vox)/vec3(GRID_SIZE))*2.0-1.0;
    float h[H_DIM];
    for(int j=0; j<H_DIM; j++) {
        float acc=b1[j];
        acc+=w1[j*3+0]*x.x+w1[j*3+1]*x.y+w1[j*3+2]*x.z;
        h[j]=max(acc,0.0);
    }
    float sdf=b2;
    for(int j=0; j<H_DIM; j++) sdf+=w2[j]*h[j];
    imageStore(sdfGrid, vox, vec4(sdf));
}
```

[Source: Mescheder et al. "Occupancy Networks", CVPR 2019; Müller et al. "Instant NGP", SIGGRAPH 2022]

### 27.4 Feature Distillation from 3DGS to Mesh Atlas

Bake semantic/appearance features from a trained 3DGS scene (§16) into a mesh UV atlas (§7.13) for downstream editing or segmentation:

```glsl
// feature_bake.comp
layout(set=0,binding=0) readonly buffer AtlasPos  { vec4 worldPos[];  };
layout(set=0,binding=1) readonly buffer Gaussians { GaussianData g[]; };
layout(set=0,binding=2, rgba32f) uniform writeonly image2D featureAtlas;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 texel=ivec2(gl_GlobalInvocationID.xy);
    vec3  p=worldPos[texel.y*ATLAS_W+texel.x].xyz;
    vec4  feat=vec4(0);
    for(uint i=0; i<N_GAUSSIANS; i++) {
        float d=length(g[i].pos-p);
        float alpha=g[i].opacity*exp(-0.5*d*d/(g[i].scale*g[i].scale));
        feat+=alpha*g[i].semanticFeature;
    }
    imageStore(featureAtlas, texel, feat);
}
```

[Source: Kerbl et al. SIGGRAPH 2023; Lerf "Language Embedded Radiance Fields", ICCV 2023]

---

## 28. 2D Vector Graphics on GPU

*Audience: graphics application developers.*

GPU geometry algorithms are not exclusive to 3D. 2D vector graphics — Bézier curves, fonts, strokes, filled paths — have their own GPU-native rendering techniques that avoid tessellation entirely.

### 28.1 Loop-Blinn Bézier Rendering

Loop & Blinn (2005) cover each cubic Bézier with one triangle; the fragment shader evaluates the implicit polynomial (k³ − lm) and discards pixels outside the fill:

```glsl
// bezier_fill.frag
in vec3 klm;  // (k, l, m) per-fragment via barycentric interpolation
out vec4 fragColor;
void main() {
    float f=klm.x*klm.x*klm.x - klm.y*klm.z;
    float fw=fwidth(f);
    float alpha=smoothstep(fw,-fw,f);
    fragColor=vec4(color.rgb, color.a*alpha);
}
```

Vertex shader assigns (k,l,m) from the Bézier canonical form (serpentine, cusp, loop, quadratic — classified by the curve's discriminant). No tessellation; correct at any resolution. [Source: Loop & Blinn "Resolution Independent Curve Rendering", SIGGRAPH 2005]

### 28.2 MSDF Font Atlas Rendering

Multi-channel SDF (Chlumský 2017) encodes edge orientation into RGB channels to restore sharp corners lost by single-channel SDF:

```glsl
// msdf.frag
uniform sampler2D msdfAtlas;
uniform float pxRange;
in vec2 uv;
float median(float r,float g,float b){ return max(min(r,g),min(max(r,g),b)); }
void main() {
    vec3  s=texture(msdfAtlas,uv).rgb;
    float sd=median(s.r,s.g,s.b);
    float px=pxRange*(sd-0.5);
    float a=clamp(px+0.5,0.0,1.0);
    fragColor=vec4(textColor.rgb, textColor.a*a);
}
```

Generate the atlas offline with `msdfgen` ([source](https://github.com/Chlumsky/msdfgen)). [Source: Chlumský "Shape Decomposition for Multi-Channel Distance Fields", master's thesis 2017]

### 28.3 Stroke Expansion in Compute

Expand a 2D polyline to a quad strip with miter joins in a compute shader:

```glsl
// stroke_expand.comp
layout(set=0,binding=0) readonly buffer Poly { vec2 pts[]; };
layout(set=0,binding=1) writeonly buffer Strip{ vec2 strip[]; };
layout(push_constant) uniform PC { float hw; } pc;  // half-width
layout(local_size_x=64) in;
void main() {
    uint i=gl_GlobalInvocationID.x;
    if(i>=N_SEGS) return;
    vec2 a=pts[i], b=pts[i+1];
    vec2 dir=normalize(b-a), perp=vec2(-dir.y,dir.x)*pc.hw;
    vec2 miter=perp;
    if(i>0){
        vec2 pd=normalize(a-pts[i-1]);
        vec2 tang=normalize(dir+pd);
        miter=vec2(-tang.y,tang.x)*pc.hw/max(dot(vec2(-tang.y,tang.x),perp/pc.hw),0.1);
    }
    strip[i*4+0]=a+miter; strip[i*4+1]=a-miter;
    strip[i*4+2]=b+perp;  strip[i*4+3]=b-perp;
}
```

[Source: Rougier "Antialiased 2D Grid, Marker, and Arrow Shaders", JCGT 2014]

### 28.4 Stencil-Then-Cover for Filled Paths

Fill arbitrary paths without tessellation: (1) stencil pass renders a triangle fan from the path origin, incrementing/decrementing stencil per edge crossing direction; (2) cover pass draws a bounding quad with stencil test — only non-zero stencil pixels get filled. Implemented via `vkCmdSetStencilOp` and two draw calls. [Source: Kilgard & Bolz "GPU-accelerated Path Rendering", SIGGRAPH Asia 2012]

---

## 29. Scientific Visualization Geometry

*Audience: systems developers, graphics application developers.*

Simulation output — velocity fields, scalar fields, tensor fields — requires geometry algorithms to extract, integrate, and render meaningful visual structures. All compute patterns here reuse infrastructure from earlier sections: RK4 integration, marching cubes, mesh shaders.

### 29.1 Streamline and Pathline Integration

Streamlines trace tangent curves of a vector field v(x); pathlines follow time-varying fields. 4th-order Runge-Kutta in compute — one thread per seed:

```glsl
// streamline.comp
layout(set=0,binding=0) uniform sampler3D velField;
layout(set=0,binding=1) writeonly buffer Lines { vec4 pts[MAX_SEEDS*MAX_STEPS]; };
layout(set=0,binding=2) writeonly buffer Counts{ uint len[]; };
layout(push_constant) uniform PC { float dt; uint maxSteps; vec3 fMin,fScale; } pc;
layout(local_size_x=64) in;

vec3 sample(vec3 p){ return texture(velField,(p-pc.fMin)/pc.fScale).xyz; }
vec3 rk4(vec3 p,float dt){
    vec3 k1=sample(p), k2=sample(p+0.5*dt*k1),
         k3=sample(p+0.5*dt*k2), k4=sample(p+dt*k3);
    return p+dt/6.0*(k1+2*k2+2*k3+k4);
}
void main() {
    uint lid=gl_GlobalInvocationID.x;
    vec3 pos=seedPoints[lid];
    uint base=lid*MAX_STEPS, step;
    for(step=0; step<pc.maxSteps && !outsideDomain(pos); step++){
        pts[base+step]=vec4(pos,0.0); pos=rk4(pos,pc.dt);
    }
    len[lid]=step;
}
```

Tube geometry (§6.3) converts streamlines to renderable ribbons. [Source: Jobard & Lefer "Creating Evenly-Spaced Streamlines", EuroVis 1997]

### 29.2 Empty-Space Skipping for Volume Ray Casting

Volume rendering casts one ray per pixel accumulating colour and opacity via front-to-back compositing. A min-max mipmap skips empty regions:

```glsl
// volume_cast.frag
uniform sampler3D vol; uniform sampler3D minMaxMip;
in vec3 entryPos, exitPos;
void main() {
    vec3 dir=normalize(exitPos-entryPos);
    float tMax=length(exitPos-entryPos);
    vec3 col=vec3(0); float T=1.0, t=0.0;
    while(t<tMax && T>0.01) {
        vec3 p=entryPos+t*dir;
        float maxD=textureLod(minMaxMip,p,SKIP_MIP).g;
        if(maxD<ISO_THRESHOLD){ t+=SKIP_STEP; continue; }
        float density=texture(vol,p).r;
        vec4 c=transferFunction(density);
        col+=(1.0-T)*c.a*c.rgb; T*=(1.0-c.a);
        t+=FINE_STEP;
    }
    fragColor=vec4(col,1.0-T);
}
```

[Source: Krueger & Westermann "Acceleration Techniques for GPU-based Volume Rendering", VIS 2003]

### 29.3 Tensor Field Glyphs in Mesh Shaders

Superquadric tensor glyphs (Kindlmann 2004) encode tensor eigenvalues/eigenvectors in glyph shape and orientation. Generated entirely in the mesh shader — no precomputed glyph mesh — one workgroup per grid point:

```glsl
// tensor_glyph.mesh.glsl
layout(triangles, max_vertices=96, max_primitives=64) out;
layout(local_size_x=64) in;
taskPayloadSharedEXT struct { uint count; uint glyphIdx[32]; } payload;
void main() {
    uint g=payload.glyphIdx[gl_WorkGroupID.x];
    vec3 evals=eigenvalues[g]; mat3 evecs=eigenvectors[g];
    // sample superquadric on (theta,phi) parametric grid
    // scale by eigenvalues, apply superquadric nonlinearity, rotate by evecs
    // emit vertices and triangle indices
    SetMeshOutputsEXT(96, 64);
}
```

[Source: Kindlmann "Superquadric Tensor Glyphs", EuroVis 2004]

### 29.4 Time-Varying Isosurface Extraction

For animated scalar fields, limit GPU marching cubes (§3.2) to **dirty voxels** where the field changed:

```glsl
// dirty_mark.comp — mark voxels changed between frames
layout(set=0,binding=0) readonly buffer FieldOld { float fOld[]; };
layout(set=0,binding=1) readonly buffer FieldNew { float fNew[]; };
layout(set=0,binding=2) buffer DirtyBits { uint dirty[]; };
layout(local_size_x=64) in;
void main() {
    uint v=gl_GlobalInvocationID.x;
    if(abs(fNew[v]-fOld[v])>CHANGE_THRESHOLD)
        atomicOr(dirty[v/32], 1u<<(v%32));
}
```

The MC dispatch skips voxels with `dirty[v/32] & (1<<(v%32)) == 0`. For slowly evolving simulations the dirty fraction is typically < 5% per frame, reducing MC compute by ~20×. [Source: Lorensen & Cline SIGGRAPH 1987; temporal delta technique from Kazhdan et al. "Streaming Multigrid", SIGGRAPH 2008]

---

## 30. Hair and Strand Geometry

*Audience: graphics application developers.*

Hair simulation and rendering sit at the intersection of geometry generation, physics simulation, and LOD management. Unlike mesh-based geometry, strand geometry is one-dimensional (a piecewise curve) but must interact with a 3D collision surface, self-collide at high density, and transition between strand, shell (hair card), and volume representations across LOD levels.

### 30.1 Strand Representation and Cosserat Rods

Each hair strand is a sequence of N control points {p₀, p₁, …, p_{N-1}} plus a reference frame per segment (bishop frame or material frame). The Cosserat rod model tracks both centerline position and cross-section orientation, enabling simulation of bending, twisting, and stretching:

```glsl
// strand_step.comp — one integration step per strand segment
layout(set=0,binding=0) buffer StrandPos  { vec4 pos[];  };   // xyz=position, w=invMass
layout(set=0,binding=1) buffer StrandVel  { vec3 vel[];  };
layout(set=0,binding=2) readonly buffer StrandRest { vec4 rest[]; };
layout(push_constant) uniform PC { float dt; float kStretch; float kBend; float gravity; } pc;
layout(local_size_x=64) in;

void main() {
    uint i = gl_GlobalInvocationID.x;   // global segment index
    uint strand = i / SEGS_PER_STRAND;
    uint seg    = i % SEGS_PER_STRAND;
    if (seg == 0) return;               // root is pinned to scalp

    // Semi-implicit Euler: gravity + damping
    vel[i] += vec3(0, -pc.gravity, 0) * pc.dt;
    vel[i] *= 0.98;                     // linear damping
    pos[i].xyz += vel[i] * pc.dt;

    // Stretch constraint: enforce rest length between consecutive control points
    uint prev = i - 1;
    vec3 diff  = pos[i].xyz - pos[prev].xyz;
    float len  = length(diff);
    float rest = length(rest[i].xyz - rest[prev].xyz);
    vec3  corr = 0.5 * (len - rest) * normalize(diff);
    if (pos[i].w > 0.0)    pos[i].xyz   -= corr * pc.kStretch;
    if (pos[prev].w > 0.0) pos[prev].xyz += corr * pc.kStretch;
}
```

[Source: Müller et al. "Position Based Dynamics", J. Visual Communication 2007; AMD TressFX https://github.com/GPUOpen-Effects/TressFX]

### 30.2 Bending and Torsion Constraints

Bending is resisted by a dihedral angle constraint between consecutive segment pairs. Per strand, thread i handles the joint at control point i:

```glsl
// strand_bend.comp — per-strand bending constraint projection
layout(local_size_x=SEGS_PER_STRAND) in;
void main() {
    uint strand = gl_WorkGroupID.x;
    uint seg    = gl_LocalInvocationID.x;
    uint base   = strand * SEGS_PER_STRAND;
    if (seg == 0 || seg >= SEGS_PER_STRAND - 1) return;

    vec3 p0 = pos[base+seg-1].xyz;
    vec3 p1 = pos[base+seg  ].xyz;
    vec3 p2 = pos[base+seg+1].xyz;

    vec3 e0  = normalize(p1 - p0);
    vec3 e1  = normalize(p2 - p1);
    float cosA = dot(e0, e1);
    float target = rest_cos_angle[base+seg];  // cosine of rest bend angle
    float diff   = cosA - target;

    // Gradient projection (XPBD formulation)
    vec3 grad = (e0 + e1 - cosA*(e0 + e1)) / max(length(p2-p0), 1e-4);
    float lambda = -diff / (dot(grad, grad) + XPBD_COMPLIANCE);
    if (pos[base+seg].w > 0.0) pos[base+seg].xyz += lambda * grad * BEND_WEIGHT;
}
```

### 30.3 Strand Self-Collision via Spatial Hashing

At hair densities of 50–200k strands, brute-force self-collision is infeasible. Uniform grid hashing (§17.2 pattern) detects strand segment pairs within a collision radius:

```glsl
// strand_selfcol.comp — push apart overlapping segments
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    ivec3 ci = ivec3(floor(pos[i].xyz / HAIR_CELL));
    for (int dz=-1;dz<=1;dz++) for(int dy=-1;dy<=1;dy++) for(int dx=-1;dx<=1;dx++) {
        uint cell = hashCell3(ci + ivec3(dx,dy,dz));
        for (uint k = gridStart[cell]; k < gridEnd[cell]; k++) {
            uint j = gridSegs[k];
            if (j / SEGS_PER_STRAND == i / SEGS_PER_STRAND) continue;  // same strand
            vec3 d = pos[i].xyz - pos[j].xyz;
            float dist = length(d);
            if (dist < HAIR_RADIUS * 2.0 && dist > 1e-5) {
                vec3 push = normalize(d) * (HAIR_RADIUS * 2.0 - dist) * 0.5;
                if (pos[i].w > 0.0) pos[i].xyz += push;
                if (pos[j].w > 0.0) pos[j].xyz -= push;
            }
        }
    }
}
```

[Source: TressFX 4.0 source, GPUOpen; Daviet et al. "A Semi-Implicit Material Point Method for the Continuum Simulation of Granular Materials", SIGGRAPH 2011]

### 30.4 LOD: Strand-to-Shell-to-Volume Transitions

At distance d:
- **Close** (d < d₁): render individual strands as triangle strips (2 triangles per segment via geometry expansion in mesh shader).
- **Medium** (d₁ < d < d₂): render hair cards — pre-baked atlas strips approximating clusters of 10–50 strands.
- **Far** (d > d₂): render as volume (shell mesh with transparency and noise-driven alpha).

The LOD selection runs in the task shader per strand cluster; the mesh shader expands selected strands into quad strips:

```glsl
// hair_mesh.glsl — expand strand control points to quad strip
layout(triangles, max_vertices=SEGS_PER_STRAND*2, max_primitives=SEGS_PER_STRAND) out;
void main() {
    uint strand = payload.strandIdx[gl_WorkGroupID.x];
    uint base   = strand * SEGS_PER_STRAND;
    SetMeshOutputsEXT(SEGS_PER_STRAND*2, SEGS_PER_STRAND-1);
    for (uint s=0; s<SEGS_PER_STRAND; s++) {
        vec3 p   = pos[base+s].xyz;
        vec3 cam = normalize(cameraPos - p);
        vec3 tan = (s+1 < SEGS_PER_STRAND)
                 ? normalize(pos[base+s+1].xyz - p)
                 : normalize(p - pos[base+s-1].xyz);
        vec3 side = normalize(cross(tan, cam)) * HAIR_WIDTH * 0.5;
        gl_MeshVerticesEXT[s*2+0].gl_Position = vp * vec4(p + side, 1.0);
        gl_MeshVerticesEXT[s*2+1].gl_Position = vp * vec4(p - side, 1.0);
    }
    for (uint s=0; s<SEGS_PER_STRAND-1; s++) {
        gl_PrimitiveTriangleIndicesEXT[s] = uvec3(s*2,s*2+1,s*2+2);
    }
}
```

[Source: AMD TressFX; NVIDIA HairWorks; Yuksel & Tariq "Advanced Techniques in Real-time Hair Rendering", SIGGRAPH Course 2010]

---

## 31. Level Set Methods

*Audience: systems developers, graphics application developers.*

Level set methods (Osher & Sethian 1988) represent surfaces implicitly as the zero crossing of a scalar field φ(x,t) stored on a grid. Unlike JFA (§3.4), which constructs SDFs from point sources, LSM *evolves* the SDF forward in time using a PDE. Key applications: fluid free surfaces (§11 uses MAC grid + LSM hybrid), phase-field fracture (§25), topology-changing surface animation.

### 31.1 Level Set Advection

The fundamental LSM PDE for a velocity field **v**:

∂φ/∂t + **v** · ∇φ = 0

Discretise using semi-Lagrangian advection (same stencil as §11.1 MAC advection, but applied to the scalar SDF field):

```glsl
// ls_advect.comp — semi-Lagrangian level set advection
layout(set=0,binding=0) uniform sampler3D phiIn;
layout(set=0,binding=1) uniform sampler3D velField;
layout(set=0,binding=2, r32f) uniform writeonly image3D phiOut;
layout(push_constant) uniform PC { float dt; vec3 gridMin; float dx; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 idx = ivec3(gl_GlobalInvocationID);
    vec3  x   = pc.gridMin + (vec3(idx) + 0.5) * pc.dx;
    vec3  v   = texture(velField, (x - pc.gridMin) / (pc.dx * GRID_SIZE)).xyz;
    vec3  xp  = x - v * pc.dt;  // back-trace particle position
    vec3  uvw = (xp - pc.gridMin) / (pc.dx * GRID_SIZE);
    imageStore(phiOut, idx, vec4(texture(phiIn, uvw).r));
}
```

### 31.2 SDF Reinitialization (Redistancing)

After advection, φ is no longer a true SDF (|∇φ| ≠ 1). Sussman et al. (1994) solve:

∂φ/∂τ = sign(φ₀)(1 − |∇φ|)

until steady state (|∇φ| ≈ 1 near the zero crossing). GPU implementation: one compute dispatch per pseudo-time step τ, applying the PDE only within a narrow band of ±k·dx around φ=0:

```glsl
// ls_reinit.comp — Sussman redistancing, narrow-band
layout(set=0,binding=0) readonly buffer PhiIn  { float phiIn[];  };
layout(set=0,binding=1) writeonly buffer PhiOut { float phiOut[]; };
layout(push_constant) uniform PC { float dtau; float dx; float bandWidth; } pc;
layout(local_size_x=64) in;

float godunovGrad(uint v) {
    float dxp = (phiIn[v+1]        - phiIn[v]) / pc.dx;
    float dxm = (phiIn[v]          - phiIn[v-1]) / pc.dx;
    float dyp = (phiIn[v+GRID_X]   - phiIn[v]) / pc.dx;
    float dym = (phiIn[v]          - phiIn[v-GRID_X]) / pc.dx;
    float dzp = (phiIn[v+GRID_XY]  - phiIn[v]) / pc.dx;
    float dzm = (phiIn[v]          - phiIn[v-GRID_XY]) / pc.dx;
    float sp   = sign(phiIn[v]);
    float Dp = (sp > 0.0)
        ? sqrt(max(dxm,0)*max(dxm,0)+max(-dxp,0)*max(-dxp,0)+max(dym,0)*max(dym,0)+max(-dyp,0)*max(-dyp,0)+max(dzm,0)*max(dzm,0)+max(-dzp,0)*max(-dzp,0))
        : sqrt(max(-dxm,0)*max(-dxm,0)+max(dxp,0)*max(dxp,0)+max(-dym,0)*max(-dym,0)+max(dyp,0)*max(dyp,0)+max(-dzm,0)*max(-dzm,0)+max(dzp,0)*max(dzp,0));
    return Dp;
}

void main() {
    uint v = gl_GlobalInvocationID.x;
    if (abs(phiIn[v]) > pc.bandWidth) { phiOut[v] = phiIn[v]; return; }
    float S = sign(phiIn[v]);
    phiOut[v] = phiIn[v] - pc.dtau * S * (godunovGrad(v) - 1.0);
}
```

[Source: Osher & Sethian "Fronts Propagating with Curvature-Dependent Speed", J. Comput. Phys. 1988; Sussman et al. "A Level Set Approach for Computing Solutions to Incompressible Two-Phase Flow", J. Comput. Phys. 1994]

### 31.3 GPU Fast Marching Method (FMM)

FMM builds the SDF outward from an initialised narrow band using a heap-like wavefront. GPU parallelism: Jeong & Whitaker (2008) parallel FMM tiles each grid block and uses a local heap per workgroup, merging wavefronts at block boundaries:

```glsl
// fmm_sweep.comp — one FMM sweep pass (Godunov upwind stencil)
layout(set=0,binding=0) buffer SDF    { float sdf[]; };
layout(set=0,binding=1) buffer State  { uint  state[]; };  // 0=unknown,1=trial,2=known
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    uint v = flatIdx(ivec3(gl_GlobalInvocationID));
    if (state[v] == 2u) return;  // already frozen
    // Godunov upwind: take minimum of ±x, ±y, ±z known neighbours
    float ax = min(sdf[v-1], sdf[v+1]);
    float ay = min(sdf[v-GRID_X], sdf[v+GRID_X]);
    float az = min(sdf[v-GRID_XY], sdf[v+GRID_XY]);
    // solve quadratic (ax-t)²+(ay-t)²+(az-t)²=dx² for t
    // ... Godunov quadratic solve ...
    float t = godunovSolve(ax, ay, az, DX);
    if (t < sdf[v]) { sdf[v] = t; state[v] = 1u; }
}
```

Iterate until no trial nodes remain. [Source: Jeong & Whitaker "A Fast Iterative Method for Eikonal Equations", SIAM J. Sci. Comput. 2008]

### 31.4 Narrow-Band Level Sets

Only voxels within a band of width ±W·dx of φ=0 need updating. Compact active voxels with prefix scan (§25 dirty-bitmask pattern) before each sweep to avoid dispatching over the full grid. For a 256³ grid with W=3, this limits active voxels to ~5% of the grid at typical surface areas.

---

## 32. Differentiable Rendering and Inverse Geometry

*Audience: graphics application developers, systems developers.*

Differentiable rendering computes the gradient ∂L/∂θ of a scalar loss L (e.g., image reconstruction error) with respect to scene geometry parameters θ (vertex positions, SDF values, Gaussian means). This enables gradient-based geometry optimisation: mesh fitting to photographs, geometry parameter tuning, neural inverse rendering.

### 32.1 Differentiable Rasterization: Soft Visibility

Standard rasterisation is not differentiable at triangle edges (step-function coverage). SoftRas (Liu et al. 2019) and NVDiffRast (Laine et al. 2020) smooth the triangle edge boundary:

```glsl
// soft_rast.frag — soft triangle coverage for differentiable rendering
in vec3 bary;   // barycentric coordinates
uniform float sigma;  // softness parameter (typ. 1e-4)
out float coverage;
void main() {
    float d = min(min(bary.x, bary.y), bary.z);  // signed distance to nearest edge
    coverage = 1.0 / (1.0 + exp(-d / sigma));    // sigmoid soft boundary
}
```

The backward pass accumulates ∂coverage/∂vertex_position via the chain rule; each edge contributes a gradient proportional to the sigmoid derivative. [Source: Liu et al. "Soft Rasterizer", ICCV 2019; Laine et al. "Modular Primitives for High-Performance Differentiable Rendering", SIGGRAPH Asia 2020]

### 32.2 Differentiable Ray Tracing: Geometry Gradients

For ray-traced images, the gradient of a pixel colour L w.r.t. a vertex position p is:

∂L/∂p = ∂L/∂x_hit · ∂x_hit/∂p

where x_hit is the intersection point. The second term comes from the barycentric interpolation:

```glsl
// grad_rchit.glsl — accumulate vertex gradient in ray tracing closest-hit shader
hitAttributeEXT vec2 bary;
layout(set=0,binding=3) buffer GradBuffer { vec3 vertGrad[]; };

void main() {
    vec3 grad_L = gradFromPayload();    // ∂L/∂shading from miss/recursion
    uint i0 = indices[3*gl_PrimitiveID+0];
    uint i1 = indices[3*gl_PrimitiveID+1];
    uint i2 = indices[3*gl_PrimitiveID+2];
    float w0 = 1.0 - bary.x - bary.y;
    // ∂x_hit/∂v_k = w_k * I (identity), so ∂L/∂v_k = w_k * grad_L
    atomicAdd_vec3(vertGrad[i0], grad_L * w0);
    atomicAdd_vec3(vertGrad[i1], grad_L * bary.x);
    atomicAdd_vec3(vertGrad[i2], grad_L * bary.y);
}
```

[Source: Laine et al. 2020; Li et al. "Differentiable Monte Carlo Ray Tracing through Edge Sampling", SIGGRAPH Asia 2018]

### 32.3 3DGS Gradient Flow

For 3D Gaussian splatting (§16), the backward pass of the tile rasterizer propagates image-space gradients back to per-Gaussian mean μ, covariance Σ, opacity α, and spherical harmonic coefficients. The forward-pass tile rasterizer stores per-pixel transmittance T for the backward pass:

```glsl
// gs_backward.comp — gradient w.r.t. Gaussian means and opacity
layout(set=0,binding=0) readonly buffer Transmittance { float T[];    };   // per-pixel, per-depth
layout(set=0,binding=1) readonly buffer GradImg       { vec3  dL_dC[]; };  // ∂L/∂pixel color
layout(set=0,binding=2) buffer GradMeans { vec3 dL_dmu[]; };
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    // iterate sorted Gaussians front-to-back, accumulate gradient
    for (int k = tileGaussEnd[pix] - 1; k >= tileGaussStart[pix]; k--) {
        uint g     = sortedIdx[k];
        float dL_dalpha = dot(dL_dC[pix.y*WIDTH+pix.x], color[g]) * T[pix.y*WIDTH+pix.x];
        // ∂alpha/∂mu via projected Gaussian PDF gradient
        vec2 d     = projPos[g] - vec2(pix);
        vec2 dmu2d = -alpha[g] * dL_dalpha * (cov2dInv[g] * d);
        atomicAdd_vec2(dL_dmu2d[g], dmu2d);
    }
}
```

[Source: Kerbl et al. "3D Gaussian Splatting for Real-Time Radiance Field Rendering", SIGGRAPH 2023 (supplemental training code); https://github.com/graphdeco-inria/gaussian-splatting]

### 32.4 Geometry Optimisation Loop

The full loop: forward render → compute loss → backward pass → gradient descent step on vertex positions/SDF values:

```python
# Host-side loop (pseudo-code; actual dispatch via Vulkan compute)
for iteration in range(MAX_ITERS):
    vkCmdDispatch(forward_render)      # §32.1 soft rasteriser or §20 RT
    vkCmdDispatch(loss_compute)        # MSE / perceptual loss
    vkCmdDispatch(backward_pass)       # §32.2 or §32.3 gradient accumulation
    vkCmdDispatch(adam_update)         # Adam: m = β1*m + (1-β1)*g; ...
    # After convergence: extract mesh from optimised SDF via §3.2 GPU MC
```

Adam per-parameter momentum/variance update also runs in compute — one thread per vertex/voxel/Gaussian. [Source: Kingma & Ba "Adam: A Method for Stochastic Optimization", ICLR 2015]

---

## 33. GPU Particle Systems and Ribbon Geometry

*Audience: graphics application developers.*

Particle systems are the standard GPU geometry for effects (fire, smoke, sparks, rain, magic). Unlike §11 physics-driven SPH, this section covers the *rendering geometry pipeline*: how particles are born, evolved, compacted, and converted into drawable primitives each frame.

### 33.1 Particle Lifecycle: Emit, Age, Kill

Particles are stored in a flat SSBO pool. Each frame: (1) increment age and kill dead particles; (2) emit new particles into freed slots. Stream compaction (prefix scan) packs live particles to a dense range for rendering:

```glsl
// particle_update.comp
layout(set=0,binding=0) buffer Particles { ParticleData p[]; };
layout(set=0,binding=1) buffer AliveOut  { uint alive[]; };
layout(set=0,binding=2) buffer AliveCount{ uint count; };
layout(push_constant) uniform PC { float dt; } pc;
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    p[i].age  += pc.dt;
    p[i].pos  += p[i].vel * pc.dt + vec3(0,-9.8,0)*pc.dt*pc.dt*0.5;
    p[i].vel  += vec3(0,-9.8,0) * pc.dt;
    p[i].vel  *= 0.99;  // drag
    if (p[i].age < p[i].lifetime) {
        uint slot = atomicAdd(count, 1u);
        alive[slot] = i;
    }
}
```

Emission fills slots after compaction using a free-list counter. [Source: Harris "GPU Gems 3 §39: Parallel Prefix Sum"; Fatahalian et al. "Sequentially Consistent Atomic Operations in GLSL"]

### 33.2 Billboard Expansion in Mesh Shaders

Each live particle becomes a screen-aligned quad. The mesh shader generates 4 vertices and 2 triangles per particle, avoiding geometry shader overhead:

```glsl
// particle_mesh.glsl
layout(triangles, max_vertices=128, max_primitives=64) out;
layout(local_size_x=64) in;
taskPayloadSharedEXT struct { uint idx[64]; uint count; } payload;
out vec2 vUV[];
out vec4 vColor[];
void main() {
    SetMeshOutputsEXT(payload.count * 4, payload.count * 2);
    for (uint t = gl_LocalInvocationID.x; t < payload.count; t += 64) {
        ParticleData pa = particles[payload.idx[t]];
        float sz = pa.size * (1.0 - pa.age / pa.lifetime);  // shrink on death
        vec3 right = cameraRight * sz;
        vec3 up    = cameraUp   * sz;
        gl_MeshVerticesEXT[t*4+0].gl_Position = vp * vec4(pa.pos - right - up, 1);
        gl_MeshVerticesEXT[t*4+1].gl_Position = vp * vec4(pa.pos + right - up, 1);
        gl_MeshVerticesEXT[t*4+2].gl_Position = vp * vec4(pa.pos + right + up, 1);
        gl_MeshVerticesEXT[t*4+3].gl_Position = vp * vec4(pa.pos - right + up, 1);
        vUV[t*4+0]=vec2(0,0); vUV[t*4+1]=vec2(1,0);
        vUV[t*4+2]=vec2(1,1); vUV[t*4+3]=vec2(0,1);
        gl_PrimitiveTriangleIndicesEXT[t*2+0] = uvec3(t*4+0,t*4+1,t*4+2);
        gl_PrimitiveTriangleIndicesEXT[t*2+1] = uvec3(t*4+0,t*4+2,t*4+3);
    }
}
```

### 33.3 Ribbon / Trail Geometry via Rolling Ring Buffer

A ribbon is the surface swept by a particle along its path — N recent positions stored in a ring buffer, converted to a quad strip each frame:

```glsl
// ribbon_expand.comp — ring buffer → quad strip
layout(set=0,binding=0) readonly buffer RingBuf { vec4 history[N_PARTICLES * TRAIL_LEN]; };
layout(set=0,binding=1) readonly buffer Head    { uint head[N_PARTICLES]; };
layout(set=0,binding=2) writeonly buffer Strip  { vec4 verts[]; };
layout(local_size_x=64) in;
void main() {
    uint p = gl_GlobalInvocationID.x;
    uint h = head[p];
    for (uint s = 0; s < TRAIL_LEN - 1; s++) {
        uint cur  = (h + s)          % TRAIL_LEN;
        uint next = (h + s + 1)      % TRAIL_LEN;
        vec3 a    = history[p*TRAIL_LEN + cur ].xyz;
        vec3 b    = history[p*TRAIL_LEN + next].xyz;
        vec3 dir  = normalize(b - a);
        vec3 perp = normalize(cross(dir, cameraPos - a));
        float w   = mix(WIDTH_TIP, WIDTH_BASE, float(s) / float(TRAIL_LEN));
        verts[(p*(TRAIL_LEN-1) + s)*2 + 0] = vec4(a - perp*w, 1);
        verts[(p*(TRAIL_LEN-1) + s)*2 + 1] = vec4(a + perp*w, 1);
    }
}
```

### 33.4 Soft Particles via Depth Comparison

Soft particles fade out where they intersect scene geometry, avoiding hard silhouettes. The fragment shader reads the depth buffer and blends by proximity:

```glsl
// soft_particle.frag
uniform sampler2D depthBuffer;
in vec2 screenUV;
in float particleDepth;
void main() {
    float sceneZ = texture(depthBuffer, screenUV).r;
    float sceneLinear = linearize(sceneZ);
    float partLinear  = linearize(particleDepth);
    float softness    = clamp((sceneLinear - partLinear) / SOFT_DISTANCE, 0.0, 1.0);
    fragColor.a *= softness;
}
```

Requires the depth buffer to be readable as a texture (`VK_IMAGE_LAYOUT_DEPTH_READ_ONLY_STENCIL_ATTACHMENT_OPTIMAL` or a depth copy pre-pass). [Source: Vlachos "Soft Particles", GDC 2007]

---

## 34. Precomputed Visibility and Lightmap Baking

*Audience: graphics application developers.*

Lightmap baking pre-integrates indirect illumination into per-texel texture samples, allowing high-quality global illumination at zero runtime cost. The GPU-accelerated bake pipeline dispatches one ray (or hemisphere of rays) per atlas texel using the VK_KHR_ray_tracing_pipeline from §20.

### 34.1 Texel-World Position Atlas

The bake begins by rasterising the mesh into UV space to produce per-texel world-space position and normal. A G-buffer pass with the UV as clip-space position fills this atlas:

```glsl
// atlas_gbuf.vert — render mesh in UV space
layout(location=0) in vec3 position;
layout(location=1) in vec3 normal;
layout(location=2) in vec2 uv;
layout(location=0) out vec3 vWorldPos;
layout(location=1) out vec3 vNormal;
void main() {
    vWorldPos   = (model * vec4(position,1)).xyz;
    vNormal     = normalize(mat3(model) * normal);
    gl_Position = vec4(uv * 2.0 - 1.0, 0.5, 1.0);  // UV → NDC
}
```

### 34.2 Hemisphere Ray Batching for AO

Each texel dispatches N ray queries against the scene TLAS (§20.3) in a hemisphere centred on the surface normal. Uses `VK_KHR_ray_query` (inline ray queries in compute) to avoid full RT pipeline overhead for AO:

```glsl
// ao_bake.comp
#extension GL_EXT_ray_query : require
layout(set=0,binding=0) readonly buffer AtlasPos { vec4 pos[];  };   // world pos per texel
layout(set=0,binding=1) readonly buffer AtlasNor { vec4 nor[];  };
layout(set=0,binding=2) uniform accelerationStructureEXT tlas;
layout(set=0,binding=3, r32f) uniform writeonly image2D aoAtlas;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 texel = ivec2(gl_GlobalInvocationID.xy);
    vec3  p     = pos[texel.y*ATLAS_W + texel.x].xyz;
    vec3  n     = nor[texel.y*ATLAS_W + texel.x].xyz;
    if (length(n) < 0.5) { imageStore(aoAtlas, texel, vec4(1)); return; }

    uint  hits  = 0;
    for (uint s = 0; s < AO_SAMPLES; s++) {
        vec3 dir = cosineSampleHemisphere(n, haltonSample(s, texel));
        rayQueryEXT rq;
        rayQueryInitializeEXT(rq, tlas, gl_RayFlagsTerminateOnFirstHitEXT,
            0xFF, p + n*0.001, 0.001, dir, AO_MAX_DIST);
        rayQueryProceedEXT(rq);
        if (rayQueryGetIntersectionTypeEXT(rq, true) != gl_RayQueryCommittedIntersectionNoneEXT)
            hits++;
    }
    imageStore(aoAtlas, texel, vec4(1.0 - float(hits)/float(AO_SAMPLES)));
}
```

### 34.3 Irradiance Baking with Spherical Harmonics

For indirect light, accumulate SH coefficients per texel from hemisphere samples. Each sample contributes to the 9 L2 SH basis functions:

```glsl
// irr_bake.comp — per-texel SH irradiance accumulation
layout(set=0,binding=3) readonly buffer Lights { SHLight lights[]; };
layout(set=0,binding=4, rgba32f) uniform writeonly image2D shAtlas;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 texel = ivec2(gl_GlobalInvocationID.xy);
    vec3  p     = atlasPos[texel.y*ATLAS_W+texel.x].xyz;
    vec3  n     = atlasNor[texel.y*ATLAS_W+texel.x].xyz;
    float sh[9] = {0,0,0,0,0,0,0,0,0};
    for (uint s=0; s<SH_SAMPLES; s++) {
        vec3 dir   = cosineSampleHemisphere(n, haltonSample(s, texel));
        vec3 color = tracePath(p + n*0.001, dir);  // one bounce
        float shBasis[9]; evalSH9(dir, shBasis);
        for (int k=0; k<9; k++) sh[k] += shBasis[k] * dot(color, vec3(0.2126,0.7152,0.0722));
    }
    // Store first 4 coefficients in one RGBA texel (9 total → 3 texels or 1 RGBA32F×3)
    imageStore(shAtlas, texel, vec4(sh[0],sh[1],sh[2],sh[3]) / float(SH_SAMPLES));
}
```

[Source: Ramamoorthi & Hanrahan "An Efficient Representation for Irradiance Environment Maps", SIGGRAPH 2001; Loos & Sloan "Volumetric Obscurance", GDC 2010]

### 34.4 Lightmap Dilation and Padding

After baking, un-baked texels (seams, invalid UV coverage) must be dilated from their neighbours to prevent bleeding artefacts. A simple iterative 4-connectivity dilation in compute:

```glsl
// dilate.comp
layout(set=0,binding=0, rgba32f) uniform image2D lm;
layout(set=0,binding=1) readonly buffer Mask { uint valid[]; };
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 c = ivec2(gl_GlobalInvocationID.xy);
    if ((valid[c.y*ATLAS_W/32 + c.x/32] >> (c.x%32)) & 1u) return;
    ivec2 dirs[4] = {ivec2(1,0),ivec2(-1,0),ivec2(0,1),ivec2(0,-1)};
    for (int d=0; d<4; d++) {
        ivec2 n = c + dirs[d];
        if (n.x<0||n.y<0||n.x>=ATLAS_W||n.y>=ATLAS_H) continue;
        if ((valid[n.y*ATLAS_W/32 + n.x/32] >> (n.x%32)) & 1u) {
            imageStore(lm, c, imageLoad(lm, n)); return;
        }
    }
}
```

Iterate 4–8 times to fill gaps up to 8 texels wide. [Source: Bjorke "High Quality Global Illumination Rendering Using Rasterization", GPU Gems 2]

---

## 35. UV Seam Optimization and Atlas Packing

*Audience: graphics application developers.*

§7.12 covered LSCM and harmonic UV parameterization algorithms. This section covers the upstream problem (where to cut seams to minimise distortion) and the downstream problem (how to pack resulting UV charts into an atlas with minimal wasted space).

### 35.1 Seam Placement via Graph Cuts

Optimal seam placement minimises total parameterization distortion. Model as a minimum-cut problem on the dual graph: edge weight = penalty for cutting that mesh edge (e.g., proportional to curvature or visibility). GPU-accelerated max-flow for mesh graphs is still an open problem; in practice, a heuristic spanning tree approach runs on CPU (Levy 2002 autoseams) and the resulting topology is uploaded for GPU parameterization.

For real-time use, a GPU approximation: score each edge by `|∇UV| × edgeLength` from an initial parameterization, mark high-distortion edges as seams, re-parameterize. One iteration in compute:

```glsl
// seam_score.comp — score edge distortion for seam candidates
layout(set=0,binding=0) readonly buffer UVs   { vec2 uv[]; };
layout(set=0,binding=1) readonly buffer Edges { uvec2 edges[]; };
layout(set=0,binding=2) writeonly buffer Score{ float score[]; };
layout(local_size_x=64) in;
void main() {
    uint e  = gl_GlobalInvocationID.x;
    uint v0 = edges[e].x, v1 = edges[e].y;
    float stretch = length(uv[v0] - uv[v1]) / length(pos[v0] - pos[v1]);
    score[e] = abs(stretch - 1.0);  // deviation from isometry
}
```

[Source: Levy et al. "Least Squares Conformal Maps for Automatic Texture Atlas Generation", SIGGRAPH 2002]

### 35.2 SLIM: Scalable Locally Injective Mappings on GPU

SLIM (Rabinovich et al. 2017) minimises a distortion energy (symmetric Dirichlet) via local-global ARAP-style iterations. Each local step computes a per-triangle optimal transformation; each global step solves a linear system. The local step is fully parallel:

```glsl
// slim_local.comp — per-triangle optimal rotation
layout(set=0,binding=0) readonly buffer UVs    { vec2 uv[]; };
layout(set=0,binding=1) readonly buffer Pos3D  { vec3 pos[]; };
layout(set=0,binding=2) writeonly buffer OptR  { mat2 R[]; };
layout(local_size_x=64) in;
void main() {
    uint t  = gl_GlobalInvocationID.x;
    uint v0 = tris[t].x, v1 = tris[t].y, v2 = tris[t].z;
    // 2D Jacobian J of UV map at triangle t
    mat2 J  = computeJacobian2D(uv[v0],uv[v1],uv[v2], pos[v0],pos[v1],pos[v2]);
    // SVD of J → nearest rotation R (closest to J in Frobenius norm with det > 0)
    mat2 U, Vt; vec2 sigma; svd2x2(J, U, sigma, Vt);
    R[t] = U * Vt;
}
```

The global step is a sparse linear system (same Laplacian structure as §14.1) solved by conjugate gradient on CPU or GPU sparse CG (§11.2). [Source: Rabinovich et al. "SLIM: Scalable Locally Injective Mappings", SIGGRAPH Asia 2017]

### 35.3 GPU Atlas Bin-Packing: Skyline Algorithm

After parameterization, pack UV chart rectangles (bounding boxes) into the atlas texture. The skyline packing algorithm places each chart at the lowest possible y in the skyline contour:

```glsl
// pack_skyline.comp — one chart placement attempt
layout(set=0,binding=0) buffer Skyline { uint height[ATLAS_W]; };  // column heights
layout(set=0,binding=1) readonly buffer Charts { ChartRect charts[]; };
layout(set=0,binding=2) buffer Placement { ivec2 place[]; };
layout(local_size_x=1) in;  // serial – one chart at a time
void main() {
    uint c   = gl_GlobalInvocationID.x;
    uint cw  = charts[c].w, ch = charts[c].h;
    uint bestX = 0, bestY = ATLAS_H;
    for (uint x=0; x+cw <= ATLAS_W; x++) {
        uint y = 0;
        for (uint dx=0; dx<cw; dx++) y = max(y, height[x+dx]);
        if (y + ch <= ATLAS_H && y < bestY) { bestY=y; bestX=x; }
    }
    place[c] = ivec2(bestX, bestY);
    for (uint dx=0; dx<cw; dx++) height[bestX+dx] = bestY + ch;
}
```

Chart placement is inherently serial (each placement modifies the skyline for the next). GPU acceleration is used for the per-chart scoring pass when many candidate positions are evaluated. [Source: Jylänki "A Thousand Ways to Pack the Bin — A Practical Approach to Two-Dimensional Rectangle Bin Packing", 2010]

---

## 36. Swept Volumes and Minkowski Sums

*Audience: systems developers, graphics application developers.*

The swept volume of an object O moving along a trajectory T is the union of all instantaneous placements O(t). GPU computation is used in robot workspace analysis, CNC toolpath collision avoidance, and dynamic broad-phase proximity queries (§22.1 SAP needs swept AABBs for fast-moving objects).

### 36.1 Swept AABB for Broadphase

For a rigid body moving from pose A to pose B in one frame, the swept AABB bounds all intermediate positions. Compute per-object:

```glsl
// swept_aabb.comp
layout(set=0,binding=0) readonly buffer ObjOld { AABB old_aabb[]; };
layout(set=0,binding=1) readonly buffer ObjNew { AABB new_aabb[]; };
layout(set=0,binding=2) writeonly buffer Swept  { AABB swept[]; };
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    swept[i].min = min(old_aabb[i].min, new_aabb[i].min);
    swept[i].max = max(old_aabb[i].max, new_aabb[i].max);
}
```

Feed swept AABBs into the SAP broadphase (§22.1) for CCD (Continuous Collision Detection).

### 36.2 Minkowski Sum of Convex Polyhedra

The Minkowski sum A ⊕ B has vertices at {a + b : a ∈ A, b ∈ B}. For convex polytopes, the exact sum is computed by merging the Gauss maps: sort all face normals of A and B on the Gauss sphere, merge-sort by angle, emit a new face for each angular range. GPU parallelism on the Gauss map merge:

```glsl
// gauss_map_sort.comp — sort face normals by longitude on Gauss sphere
layout(set=0,binding=0) readonly buffer FaceNormals { vec4 normals[]; };  // xyz normal, w=faceID
layout(set=0,binding=1) writeonly buffer Sorted { vec4 sorted[]; };
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    float lon = atan(normals[i].y, normals[i].x);  // longitude for Gauss map sort
    sorted[i] = vec4(normals[i].xyz, lon);
}
// Follow with GPU radix sort on w (longitude), then CPU merge of A and B sorted arrays
```

For non-convex objects: decompose via V-HACD (§30 Table A) then sum each convex part pair and union the resulting meshes. [Source: Dobkin et al. "Computing the Minkowski Sum of Convex Polytopes", Discrete & Computational Geometry 1993]

### 36.3 CNC Toolpath Swept Volume via SDF Dilation

In CNC verification, the swept volume of a tool along a path equals the SDF of the path dilated by the tool radius. GPU implementation: rasterise the toolpath as a polyline SDF (JFA, §3.4) then dilate by the tool radius as a post-process (morphological dilation on 3D SDF voxel grid):

```glsl
// sdf_dilate.comp — Minkowski dilation by radius r (offset SDF)
layout(set=0,binding=0, r32f) uniform readonly  image3D sdfIn;
layout(set=0,binding=1, r32f) uniform writeonly image3D sdfOut;
layout(push_constant) uniform PC { float radius; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 v   = ivec3(gl_GlobalInvocationID);
    float val = imageLoad(sdfIn, v).r - pc.radius;  // offset = dilation
    imageStore(sdfOut, v, vec4(val));
}
```

The Minkowski sum A ⊕ B_sphere(r) equals offsetting A's SDF by −r everywhere — O(1) per voxel. [Source: Jones et al. "3D Distance Fields: A Survey of Techniques and Applications", TVCG 2006]

---

## 37. Geometric Shape Morphing and Functional Maps

*Audience: graphics application developers.*

Shape morphing creates a smooth geometric transition between two shapes with potentially different topology. Unlike blend shapes (§4.5, which require identical topology), functional maps establish point-to-point correspondence between different meshes via their spectral representations, enabling morphing between a human hand and a cat paw, or completing a partial scan.

### 37.1 Functional Maps: Spectral Correspondence

A functional map C ∈ ℝ^{k×k} represents a map between two shapes M and N in the basis of their first k Laplacian eigenfunctions (§18.1). Given descriptor functions on M (e.g., WKS, HKS), solve for C minimising:

‖C · Φ_M^T · F_M − Φ_N^T · F_N‖²_F

where Φ are the eigenbases and F are descriptor function values. This is a small (k×k, k≈50) least-squares problem. The GPU contribution is evaluating descriptors at all vertices:

```glsl
// wks_descriptor.comp — Wave Kernel Signature per vertex
layout(set=0,binding=0) readonly buffer Eigenvalues { float lambda[K_EIGS]; };
layout(set=0,binding=1) readonly buffer Eigenfuncs  { float phi[N_VERTS * K_EIGS]; };
layout(set=0,binding=2) writeonly buffer WKS         { float wks[N_VERTS * N_SCALES]; };
layout(local_size_x=64) in;
void main() {
    uint v = gl_GlobalInvocationID.x;
    for (uint s=0; s<N_SCALES; s++) {
        float t   = exp(logEMin + float(s) * (logEMax - logEMin) / float(N_SCALES));
        float acc = 0.0;
        float Z   = 0.0;
        for (uint k=0; k<K_EIGS; k++) {
            float e = exp(-pow(t - log(max(lambda[k], 1e-6)), 2.0) / (2.0 * SIGMA * SIGMA));
            acc += e * phi[v*K_EIGS+k] * phi[v*K_EIGS+k];
            Z   += e;
        }
        wks[v*N_SCALES+s] = acc / max(Z, 1e-10);
    }
}
```

[Source: Ovsjanikov et al. "Functional Maps", SIGGRAPH 2012; Sun et al. "A Concise and Provably Informative Multi-Scale Signature", SGP 2009]

### 37.2 Point-to-Point Map from Functional Map

Given C, recover point-to-point correspondence: for each vertex v on M, find its image on N by nearest-neighbour search in the transformed spectral embedding:

```glsl
// recover_ptp.comp — nearest neighbour in spectral embedding
layout(set=0,binding=0) readonly buffer PhiM { float phiM[N_M * K]; };
layout(set=0,binding=1) readonly buffer PhiN { float phiN[N_N * K]; };
layout(set=0,binding=2) readonly buffer C    { float C[K * K]; };
layout(set=0,binding=3) writeonly buffer Corr { uint corr[N_M]; };
layout(local_size_x=64) in;
void main() {
    uint v = gl_GlobalInvocationID.x;
    // embed v under functional map: x_hat = C * phi_M[v]
    float xhat[K];
    for (uint k=0; k<K; k++) {
        xhat[k]=0.0;
        for (uint j=0; j<K; j++) xhat[k]+=C[k*K+j]*phiM[v*K+j];
    }
    // NN search in phiN
    float best=1e30; uint bestJ=0;
    for (uint u=0; u<N_N; u++) {
        float d=0.0;
        for (uint k=0; k<K; k++) d+=(phiN[u*K+k]-xhat[k])*(phiN[u*K+k]-xhat[k]);
        if(d<best){best=d; bestJ=u;}
    }
    corr[v]=bestJ;
}
```

### 37.3 Vertex-Interpolated Morphing

Given correspondence, interpolate vertex positions frame-by-frame. Unmatched vertices (different counts) use barycentric interpolation on the target mesh:

```glsl
// morph.comp — interpolate between source and target via correspondence
layout(set=0,binding=0) readonly buffer SrcPos  { vec3 srcPos[]; };
layout(set=0,binding=1) readonly buffer TgtPos  { vec3 tgtPos[]; };
layout(set=0,binding=2) readonly buffer Corr    { uint corr[]; };
layout(set=0,binding=3) writeonly buffer OutPos { vec3 outPos[]; };
layout(push_constant) uniform PC { float t; } pc;
layout(local_size_x=64) in;
void main() {
    uint v = gl_GlobalInvocationID.x;
    outPos[v] = mix(srcPos[v], tgtPos[corr[v]], pc.t);
}
```

[Source: Ovsjanikov et al. 2012; Rustamov et al. "Map-Based Exploration of Intrinsic Shape Differences and Variability", SIGGRAPH 2013]

---

## 38. Decal and Surface Projection Geometry

*Audience: graphics application developers.*

Decals project surface detail (bullet holes, dirt, emblems) onto arbitrary geometry without modifying the underlying mesh. GPU decal systems fall into two families: *deferred decals* (project in screen space using the deferred G-buffer) and *mesh decals* (clip decal polygon against scene triangles on GPU to generate a coplanar mesh).

### 38.1 Deferred Screen-Space Decals

A decal volume is an axis-aligned box in world space. In the deferred pass, each decal dispatches a full-screen quad; the fragment shader tests whether each G-buffer pixel falls inside the decal box and blends the decal texture:

```glsl
// decal.frag — deferred screen-space decal
uniform sampler2D depthBuffer;
uniform sampler2D decalTex;
uniform mat4 decalWorldToLocal;   // transforms world position into decal UVW [0,1]³
uniform mat4 invView, invProj;
in vec2 screenUV;
void main() {
    float z       = texture(depthBuffer, screenUV).r;
    vec4  ndc     = vec4(screenUV*2.0-1.0, z*2.0-1.0, 1.0);
    vec4  worldH  = invView * invProj * ndc;
    vec3  worldP  = worldH.xyz / worldH.w;
    vec3  local   = (decalWorldToLocal * vec4(worldP,1)).xyz;
    if (any(lessThan(local, vec3(0))) || any(greaterThan(local, vec3(1)))) discard;
    vec4 col = texture(decalTex, local.xz);
    fragColor = col;   // blend into albedo / normal GBuffer layer
}
```

For normal-map decals, also unproject the G-buffer normal and blend in tangent space. [Source: Nota "Rendering Wounds: Decal Projection in 'The Last of Us'", SIGGRAPH 2014; Valient "Killzone Shadow Fall: Deferred Rendering", GDC 2014]

### 38.2 Mesh Decal: GPU Polygon Clipping

For masked decals that must respect depth discontinuities or work in forward rendering, clip the decal polygon against each scene triangle using the Sutherland-Hodgman algorithm on GPU:

```glsl
// decal_clip.comp — clip decal quad against one scene triangle
layout(set=0,binding=0) readonly buffer SceneTris { vec3 triVerts[]; };
layout(set=0,binding=1) writeonly buffer DecalTris { vec3 outVerts[]; };
layout(set=0,binding=2) buffer OutCount { uint count; };
layout(local_size_x=64) in;

// Sutherland-Hodgman clip polygon against one half-space
uint clipAgainstPlane(vec3 poly[], uint n, vec4 plane, vec3 out[]) {
    uint outN=0;
    for (uint i=0; i<n; i++) {
        vec3 a=poly[i], b=poly[(i+1)%n];
        float da=dot(vec4(a,1),plane), db=dot(vec4(b,1),plane);
        if(da>=0.0) out[outN++]=a;
        if((da>=0.0)!=(db>=0.0)) out[outN++]=mix(a,b,-da/(db-da));
    }
    return outN;
}
void main() {
    uint t   = gl_GlobalInvocationID.x;
    vec3 tri[3] = {triVerts[t*3], triVerts[t*3+1], triVerts[t*3+2]};
    vec3 poly[8]; memcpy(poly, decalQuad, 4*12);
    uint n   = 4;
    // clip against all 6 decal box planes
    n = clipAgainstPlane(poly, n, decalPlane[0], poly);
    // ... repeat for planes 1-5 ...
    if (n >= 3) {
        uint slot = atomicAdd(count, n-2);
        for (uint i=0; i<n-2; i++) {
            outVerts[(slot+i)*3+0]=poly[0];
            outVerts[(slot+i)*3+1]=poly[i+1];
            outVerts[(slot+i)*3+2]=poly[i+2];
        }
    }
}
```

[Source: Sutherland & Hodgman "Reentrant Polygon Clipping", CACM 1974; Tatarchuk "Practical Parallax Occlusion Mapping with Self-Shadowing", GDC 2006]

### 38.3 Stencil-Volume Projection

A third approach uses the stencil buffer to mask the decal projection volume: draw back faces incrementing stencil, draw front faces decrementing. Pixels with stencil=1 are inside the volume. Then draw the decal with stencil test = equal(1):

```cpp
// Vulkan pseudo-code — stencil volume decal setup
VkStencilOpState backFace{VK_STENCIL_OP_KEEP, VK_STENCIL_OP_INCREMENT_AND_CLAMP, ...};
VkStencilOpState frontFace{VK_STENCIL_OP_KEEP, VK_STENCIL_OP_DECREMENT_AND_CLAMP, ...};
// Pass 1: fill stencil volume
// Pass 2: draw decal with VK_COMPARE_OP_EQUAL, reference=1
```

[Source: McGuire & Hughes "Fast, Practical and Robust Shadows", 2003 (stencil volume technique applied to decals)]

---

## 39. Visibility Queries and GPU Occlusion

*Audience: graphics application developers, systems developers.*

§10.3 covered Hierarchical Z-Buffer (HZB) occlusion culling within the GPU-Driven Rendering pipeline. This section covers the broader family of GPU visibility query mechanisms: hardware occlusion queries, software occlusion rasterisers in compute, and portal/sector GPU culling for indoor scenes.

### 39.1 Hardware Occlusion Queries

Vulkan's `VK_QUERY_TYPE_OCCLUSION` counts fragments that pass depth test. Typical use: render a cheap proxy (AABB) for each object, query the count, skip full rendering if zero:

```cpp
vkCmdBeginQuery(cmd, queryPool, objectID, 0);
vkCmdDraw(cmd, proxyVertexCount, 1, proxyFirst, 0);
vkCmdEndQuery(cmd, queryPool, objectID);
// Later frame: retrieve results and build indirect draw buffer for non-zero objects
vkGetQueryPoolResults(device, queryPool, 0, N_OBJECTS,
    sizeof(uint64_t)*N_OBJECTS, results, sizeof(uint64_t),
    VK_QUERY_RESULT_64_BIT | VK_QUERY_RESULT_WAIT_BIT);
```

The GPU–CPU readback latency (typically 1–2 frames) means hardware OQ is best for large objects with stable visibility. [Source: Bittner et al. "Coherent Hierarchical Culling", EuroGraphics 2004; Vulkan Spec §18.2]

### 39.2 Software Occlusion Rasteriser in Compute

A software occlusion rasteriser (SOR) in compute shaders allows the same-frame query results needed for GPU-driven pipelines (§10.4 two-phase occlusion). The SOR builds a low-resolution depth buffer (e.g., 256×128) from occluder proxies in compute, then tests occludee AABBs against it:

```glsl
// sor_raster.comp — rasterise occluder triangle into software depth buffer
layout(set=0,binding=0) buffer SoftDepth { float sdepth[SOR_W * SOR_H]; };
layout(local_size_x=64) in;
void main() {
    uint t = gl_GlobalInvocationID.x;
    vec4 v0 = occluderTris[t*3+0], v1 = occluderTris[t*3+1], v2 = occluderTris[t*3+2];
    // Project to SOR NDC, rasterise with conservative depth
    ivec2 bbMin = max(ivec2(0), ivec2(floor(min(v0.xy,min(v1.xy,v2.xy)) * SOR_SIZE)));
    ivec2 bbMax = min(ivec2(SOR_SIZE-1), ivec2(ceil(max(v0.xy,max(v1.xy,v2.xy)) * SOR_SIZE)));
    float zMax  = max(v0.z, max(v1.z, v2.z));   // conservative (near face)
    for (int y=bbMin.y; y<=bbMax.y; y++) for (int x=bbMin.x; x<=bbMax.x; x++) {
        if (pointInTriangle2D(vec2(x,y)/SOR_SIZE, v0.xy, v1.xy, v2.xy))
            atomicMin_float(sdepth[y*SOR_W+x], zMax);
    }
}
// sor_test.comp — test occludee AABB against SOR depth
// If all 8 AABB corners project behind SOR depth → cull
```

[Source: Hasselgren & Akenine-Möller "An Efficient Multi-View Rasterizer", EGSR 2006; Intel ISPC Sample "Masked Software Occlusion Culling" https://github.com/GameTechDev/OcclusionCulling]

### 39.3 Portal and Sector Visibility Culling

Indoor scenes (buildings, caves) achieve near-perfect culling by modelling rooms as convex sectors connected by portal polygons. GPU visibility: starting from the camera sector, BFS through visible portals, clipping the portal frustum at each step:

```glsl
// portal_cull.comp — one-level portal frustum clip
layout(set=0,binding=0) readonly buffer Portals { PortalData portals[]; };
layout(set=0,binding=1) buffer VisibleSectors { uint visible[]; };
layout(set=0,binding=2) buffer VisibleCount   { uint count; };
layout(local_size_x=64) in;
void main() {
    uint p = gl_GlobalInvocationID.x;
    PortalData pd = portals[p];
    if ((visible[pd.srcSector/32] & (1u << (pd.srcSector%32))) == 0u) return;
    // Check if portal frustum (from camera to portal rectangle) intersects camera frustum
    if (frustumPortalOverlap(cameraFrustum, pd.polygon, pd.nVerts)) {
        atomicOr(visible[pd.dstSector/32], 1u << (pd.dstSector%32));
        uint slot = atomicAdd(count, 1u);
        VisibleSectors[slot] = pd.dstSector;
    }
}
```

Iterate BFS until no new sectors are added. Objects in non-visible sectors are excluded from the indirect draw buffer (§10.4). [Source: Luebke & Georges "Portals and Mirrors: Simple, Fast Evaluation of Potentially Visible Sets", I3D 1995]

### 39.4 Conditional Rendering via Predication

`VK_EXT_conditional_rendering` allows draw calls to be skipped based on a buffer value without CPU readback, closing the loop between hardware OQ results (§39.1) and draw culling in a single frame:

```cpp
VkConditionalRenderingBeginInfoEXT cond{};
cond.sType  = VK_STRUCTURE_TYPE_CONDITIONAL_RENDERING_BEGIN_INFO_EXT;
cond.buffer = queryResultBuffer;   // populated from vkCmdCopyQueryPoolResults
cond.offset = objectID * sizeof(uint32_t);
cond.flags  = 0;
vkCmdBeginConditionalRenderingEXT(cmd, &cond);
vkCmdDrawIndexed(cmd, ...);        // skipped if queryResultBuffer[objectID] == 0
vkCmdEndConditionalRenderingEXT(cmd);
```

This avoids the 1–2 frame readback latency of §39.1 at the cost of conservative culling (last frame's results drive this frame's draws). [Source: Vulkan Spec VK_EXT_conditional_rendering]

---

## 40. Library Landscape

**Coverage gap.** No single open-source library covers more than a narrow slice of the algorithms in this chapter. The table below maps the available libraries against the chapter's topic areas; empty cells represent functionality that must be implemented directly in shader/compute code using the patterns shown in the preceding sections. This fragmentation is the norm in GPU geometry programming: the field is young enough that GPU-native algorithmic libraries have not yet consolidated around a common abstraction layer comparable to what BLAS/LAPACK provide for linear algebra.

The closest approximation to a broad GPU geometry toolkit is **NVIDIA's** ecosystem when CUDA is available: cuSPARSE (sparse solvers for Poisson/LSCM), Thrust (prefix scans, radix sort, stream compaction), and NVCC-compiled versions of CPU libraries. On non-NVIDIA hardware the pattern in this chapter — implementing each algorithm as a self-contained Vulkan compute shader from first principles — remains the only portable path.

Two tables follow — one per group of nine sections — because a single 22-column table is unreadable. — means no coverage in that section.

**Table A — §1–§9 coverage**

| Library | Ver | GPU Backend | §1 Subdiv | §2 NURBS | §3 Implicit | §4 Skeletal | §5 IK | §6 Splines | §7 Mesh | §8 BVH/Coll | §9 SS | Best Use |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| [OpenSubdiv](https://github.com/PixarAnimationStudios/OpenSubdiv) | 3.6.0 | GL compute, CUDA, Metal | ✓ CC/Loop | — | — | — | — | — | — | — | — | Production subdivision |
| [CGAL](https://www.cgal.org) | 6.1 | None (CPU + TBB) | ✓ CC/Loop/DS | — | ✓ MC/DC | — | — | — | ✓ CDT/repair | ✓ AABB tree | — | CPU preprocessing |
| [GeometricTools](https://github.com/davideberly/GeometricTools) | 6.x | None | ✓ CC/DS | ✓ NURBS | partial | — | — | ✓ | — | ✓ BVH | — | CPU offline reference |
| [libigl](https://github.com/libigl/libigl) | 2.5.0 | None (CPU + Eigen) | ✓ | — | ✓ MC/DC/MT | ✓ LBS/DQS | — | — | ✓ UV/Bool | — | — | Skinning weights, UV |
| [OpenCASCADE](https://www.opencascade.com/) | 8.0.0p1 | None (OpenGL vis only) | — | ✓ BRep | — | — | — | — | ✓ tess | — | — | CAD NURBS → VkBuffer |
| [meshoptimizer](https://github.com/zeux/meshoptimizer) | 0.22 | None (CPU → GPU output) | — | — | — | — | — | — | ✓ QEM/cache | — | — | LOD chain, compression |
| [NanoVDB](https://github.com/AcademySoftwareFoundation/openvdb) | OVDBv9 | CUDA / Vulkan SSBO | — | — | ✓ sparse vol | — | — | — | — | — | — | Sparse volume GPU |
| [xatlas](https://github.com/jpcy/xatlas) | 2.x | None | — | — | — | — | — | — | ✓ UV atlas | — | — | UV layout for baking |
| [PoissonRecon](https://github.com/mkazhdan/PoissonRecon) | 13.x | None (CPU MT) | — | — | ✓ recon | — | — | — | — | — | — | Point cloud → mesh |
| [V-HACD](https://github.com/kmammou/v-hacd) | 4.0 | None (CPU) | — | — | — | — | — | — | — | ✓ decomp | — | Convex hull for GJK |
| [Triangle](https://www.cs.cmu.edu/~quake/triangle.html) | 1.6 | None | — | — | — | — | — | — | ✓ CDT | — | — | 2D CDT/quality meshing |

**Table B — §10–§19 coverage**

| Library | Ver | GPU Backend | §10 GPU-Driven | §11 Fluid | §12 Deform | §13 Proc | §14 Geo | §15 Shape | §16 3DGS | §17 Spatial | §18 Spectral | §19 NavMesh | Best Use |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| [meshoptimizer](https://github.com/zeux/meshoptimizer) | 0.22 | None (CPU → GPU output) | ✓ meshlets | — | — | — | — | — | — | — | — | — | Meshlet build + bounds |
| [NanoVDB](https://github.com/AcademySoftwareFoundation/openvdb) | OVDBv9 | CUDA / Vulkan SSBO | — | — | — | — | — | — | — | ✓ SVO | — | — | Sparse voxel GPU traversal |
| [CGAL](https://www.cgal.org) | 6.1 | None (CPU + TBB) | — | — | — | — | partial cotan | — | — | ✓ k-d tree | partial | — | CPU k-d tree, Laplacian |
| [libigl](https://github.com/libigl/libigl) | 2.5.0 | None (CPU + Eigen) | — | — | partial FEM | — | ✓ cotan/geo | ✓ ARAP/MVC | — | — | ✓ spectral | — | Geodesics, deformation, spectral |
| [SPlisHSPlasH](https://github.com/InteractiveComputerGraphics/SPlisHSPlasH) | 2.13 | CUDA | — | ✓ SPH/PBF/FLIP | — | — | — | — | — | — | — | — | GPU fluid simulation |
| [PhysX 5](https://github.com/NVIDIA-Omniverse/PhysX) | 5.4 | CUDA | — | ✓ PBD fluid | ✓ FEM/cloth | — | — | — | — | — | — | — | GPU rigid + soft body + fluid |
| [geometry-central](https://github.com/nmwsharp/geometry-central) | 0.x | None (CPU) | — | — | — | — | ✓ heat/cotan | partial | — | — | ✓ spectral | — | Geodesics, DDG reference |
| [gsplat](https://github.com/nerfstudio-project/gsplat) | 1.x | CUDA / ROCm | — | — | — | — | — | — | ✓ 3DGS raster | — | — | — | GPU Gaussian splatting |
| [FastNoise2](https://github.com/Auburn/FastNoise2) | 0.10 | SIMD / CUDA node graph | — | — | — | ✓ GPU noise | — | — | — | — | — | — | Procedural terrain noise |
| [Recast/Detour](https://github.com/recastnavigation/recastnavigation) | 1.6 | None (CPU) | — | — | — | — | — | — | — | — | — | ✓ (CPU ref) | NavMesh voxelise + pathfind |

**OpenSubdiv** ([source](https://github.com/PixarAnimationStudios/OpenSubdiv)) is the right choice for any production subdivision pipeline. The lack of a Vulkan evaluator requires the custom SSBO stencil approach described in §1.5.

**CGAL** ([source](https://www.cgal.org), [subdivision](https://doc.cgal.org/latest/Subdivision_method_3/index.html)) provides `CGAL::Subdivision_method_3::CatmullClark_subdivision()` and `Loop_subdivision()` as single-call in-place operations on a `Surface_mesh`. It also covers 2D/3D Delaunay triangulation, constrained Delaunay, Ruppert refinement, a CPU k-d tree (`CGAL::Kd_tree<>`), and a partial cotangent Laplacian via `CGAL::Polygon_mesh_processing::compute_vertex_normals()` — the broadest algorithmic span of any library here, though entirely CPU-only:

```cpp
#include <CGAL/subdivision_method_3.h>
namespace Sub = CGAL::Subdivision_method_3;
Sub::CatmullClark_subdivision(mesh,
    CGAL::parameters::number_of_iterations(2));
```

**libigl** ([source](https://github.com/libigl/libigl)) provides `igl::lbs_matrix()` and `igl::dqs()` for skinning, `igl::bbw()` for bounded biharmonic weights, `igl::lscm()` and `igl::harmonic()` for UV parameterization, `igl::marching_tets()` for isosurface extraction, `igl::heat_geodesics()` and `igl::cotmatrix()` for geodesics and the cotangent Laplacian, `igl::arap()` for ARAP deformation, `igl::mvc()` for mean value coordinates, and `igl::eigs()` for spectral geometry. All CPU-side using Eigen; upload results to GPU as vertex buffers. libigl has the widest algorithmic breadth of any portable library in this chapter — it spans §1, §3, §4, §7, §14, §15, and §18 — but has no GPU execution path.

**GeometricTools** ([source](https://github.com/davideberly/GeometricTools), Boost license) provides `GTL::NURBSSurface<float, 3>` and `GTL::CatmullClarkSurface` as header-only CPU implementations. Use for offline asset preprocessing.

**meshoptimizer** ([source](https://github.com/zeux/meshoptimizer), MIT) provides `meshopt_simplify()`, `meshopt_optimizeVertexCache()`, and `meshopt_buildMeshlets()` with `meshopt_computeMeshletBounds()` — the last two are §10 GPU-Driven Rendering directly. Generate the full LOD chain and meshlet data offline, upload to a single `VkBuffer`, and dispatch via the task/mesh shader pipeline.

**NanoVDB** ([source](https://github.com/AcademySoftwareFoundation/openvdb), Apache 2.0, OpenVDB 9.0+) converts OpenVDB trees to flat GPU buffers via `nanovdb::openToNanoVDB()`. The companion GLSL header provides tree-descent accessors matching §17.3 (SVO traversal). Requires `VK_EXT_scalar_block_layout`.

**xatlas** ([source](https://github.com/jpcy/xatlas), MIT) generates UV atlases as a preprocessing step for texture baking (§7.13). CPU-only; outputs UV coordinates and a remapped index buffer ready for GPU upload.

**PoissonRecon** ([source](https://github.com/mkazhdan/PoissonRecon), MIT) is the reference screened Poisson surface reconstruction (§3.12). CPU multi-threaded (TBB); 10–60 s for 1M-point clouds.

**V-HACD 4.0** ([source](https://github.com/kmammou/v-hacd), BSD-3) decomposes non-convex meshes into convex hull approximations for physics (§8.5). CPU-only.

**Triangle** ([source](https://www.cs.cmu.edu/~quake/triangle.html), Shewchuk, public domain) is the reference 2D CDT and Ruppert quality-mesh generator (§3.11). Single-file C library.

**SPlisHSPlasH** ([source](https://github.com/InteractiveComputerGraphics/SPlisHSPlasH), MIT) is the most complete open-source GPU fluid simulation library. It implements SPH, PBF (Macklin 2013), DFSPH (divergence-free SPH), IISPH, PBD, and FLIP variants, all with CUDA backends. Spatial hashing for neighbour search runs on GPU; pressure and density solvers run fully on-device. The library outputs particle positions each step — couple to the SPH surface extraction pipeline from §3.8 to produce renderable meshes in real time.

**PhysX 5** ([source](https://github.com/NVIDIA-Omniverse/PhysX), BSD-3) adds GPU-native FEM soft bodies (corotational linear elasticity on tet meshes, §12.2) and GPU cloth (Position-Based Dynamics, matching §4.6 and §12.1) alongside GPU rigid body simulation and PBD fluid. The soft body solver runs entirely on CUDA; results are available as CUDA device pointers that map directly to Vulkan external memory via `VK_KHR_external_memory` + `VK_EXT_external_memory_host`. No Vulkan-native API; requires interop.

**geometry-central** ([source](https://github.com/nmwsharp/geometry-central), MIT) is the research reference implementation for discrete differential geometry: `HeatMethodDistance` (§14.2), cotangent Laplacian assembly, `StripePatterns` for seamless parameterization, vector heat method, and spectral basis computation (§18.1). CPU-only; results upload to GPU. Essential for verifying custom Vulkan compute DDG implementations against a known-correct baseline.

**gsplat** ([source](https://github.com/nerfstudio-project/gsplat), Apache 2.0) is a production-quality GPU 3DGS rasterizer supporting CUDA and ROCm backends. It implements the full pipeline from §16 — Gaussian projection, tile assignment, radix sort, and per-tile alpha compositing — with forward and backward passes for training. For Vulkan integration, render to a CUDA surface then copy via external memory, or port the tile rasterizer kernels to Vulkan compute using the same algorithm (§16.2).

**FastNoise2** ([source](https://github.com/Auburn/FastNoise2), MIT) provides a SIMD node-graph noise system supporting Perlin, OpenSimplex2, Cellular, Domain Warp, and fractal combinations (§13.1 terrain fBm). A CUDA backend is available for GPU-side evaluation. On CPU, AVX2/AVX-512 SIMD makes it fast enough for real-time heightmap generation.

**Recast/Detour** ([source](https://github.com/recastnavigation/recastnavigation), Zlib) is the canonical navmesh library used by most game engines. Recast's voxelisation and region-labelling steps (§19.1–19.2) are CPU-only; no GPU port exists. The flow-field construction in §19.3 is implemented directly in compute shaders without a library equivalent. Recast is the correct preprocessing tool for static navmesh geometry; Detour handles the runtime pathfinding query API.

**Table C — §20–§29 coverage**

| Library | Ver | GPU Backend | §20 RT Geom | §21 Point Cloud | §22 Rigid Body | §23 Crowd | §24 Remesh | §25 Fracture | §26 Terrain | §27 Geo DL | §28 2D Vector | §29 Sci Vis | Best Use |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| [Vulkan RT Extensions](https://www.khronos.org/registry/vulkan/) | 1.3 | Vulkan | ✓ BLAS/TLAS | — | — | — | — | — | — | — | — | — | Ray tracing AS build |
| [PCL](https://pointclouds.org) | 1.14 | None (CPU) | — | ✓ k-NN/ICP | — | — | — | — | — | — | — | ✓ vis | Point cloud CPU pipeline |
| [Open3D](https://www.open3d.org) | 0.18 | CUDA (partial) | — | ✓ k-NN/ICP/RANSAC | — | — | — | — | — | ✓ PointNet | — | ✓ vis | GPU point cloud + DL |
| [Bullet Physics](https://github.com/bulletphysics/bullet3) | 3.25 | OpenCL (btGpu) | — | — | ✓ SAP/SI/ABA | — | — | ✓ Voronoi | — | — | — | — | Physics + fracture |
| [RVO2](https://gamma.cs.unc.edu/RVO2/) | 2.0 | None (CPU MT) | — | — | — | ✓ ORCA | — | — | — | — | — | — | ORCA reference impl |
| [OpenMesh](https://www.graphics.rwth-aachen.de/OpenMesh/) | 10.0 | None (CPU) | — | — | — | — | ✓ isotropic | — | — | — | — | — | Remeshing, halfedge ops |
| [msdfgen](https://github.com/Chlumsky/msdfgen) | 1.12 | None (CPU) | — | — | — | — | — | — | — | — | ✓ MSDF atlas | — | Font atlas generation |
| [PyTorch Geometric](https://pyg.org) | 2.5 | CUDA / ROCm | — | ✓ k-NN | — | — | — | — | — | ✓ GNN/PointNet | — | — | Geometric deep learning |
| [VTK](https://vtk.org) | 9.3 | OpenGL compute | — | — | — | — | — | — | — | — | — | ✓ streams/glyphs | Scientific vis |
| [Recast/Detour](https://github.com/recastnavigation/recastnavigation) | 1.6 | None (CPU) | — | — | — | — | — | — | ✓ terrain voxel | — | — | — | Terrain navmesh preprocessing |

**Vulkan RT Extensions** (KHR, Vulkan 1.2+) are the only "library" for §20 — AS build/update APIs are in the driver. VulkanMemoryAllocator ([source](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator)) simplifies the scratch/result buffer lifecycle for BLAS/TLAS operations.

**PCL** ([source](https://pointclouds.org), BSD-3) provides `pcl::KdTreeFLANN<pcl::PointXYZ>` for CPU k-NN, `pcl::IterativeClosestPoint<>` for ICP, `pcl::SACSegmentation<>` for RANSAC plane/sphere fitting, and `pcl::visualization::PCLVisualizer` for scientific vis. No GPU backend; use for preprocessing before uploading point clouds to Vulkan SSBOs.

**Open3D** ([source](https://www.open3d.org), MIT) adds a CUDA k-NN search (`open3d.core.nns.NearestNeighborSearch`), GPU ICP variants (point-to-plane, colored ICP), RANSAC registration, and a PointNet++ training pipeline via PyTorch. The `open3d.visualization.Visualizer` covers §29 scientific vis. Open3D is the most capable GPU point cloud toolkit outside CUDA-only paths.

**Bullet 3** ([source](https://github.com/bulletphysics/bullet3), Zlib) implements SAP broadphase, sequential impulse (Catto solver), and Featherstone ABA for articulated bodies (§22). A partial OpenCL backend (`btGpuBroadphaseProxy`) parallelises the broadphase; the narrow-phase and solver remain on CPU. The `HACD` component provides Voronoi-style approximate convex decomposition for pre-fracture (§25.1).

**RVO2** ([source](https://gamma.cs.unc.edu/RVO2/), Apache 2.0) is the reference CPU implementation of ORCA (van den Berg 2011) for crowd simulation (§23.1). For GPU port use the compute shader pattern from §23.1 with the §17.2 spatial hash for neighbourhood queries.

**OpenMesh** ([source](https://www.graphics.rwth-aachen.de/OpenMesh/), LGPL) provides a halfedge-based mesh data structure with `OpenMesh::Subdivider::Uniform::CatmullClarkT<>` and edge split/collapse/flip operations for isotropic remeshing (§24.1). CPU-only; use for offline remesh passes before GPU upload.

**msdfgen** ([source](https://github.com/Chlumsky/msdfgen), MIT) generates multi-channel SDF atlases from font outlines or SVG paths — the offline preprocessing step for §28.2 MSDF font rendering. Produces a `VkImage` data asset; no runtime GPU component.

**PyTorch Geometric** ([source](https://pyg.org), MIT) provides `torch_geometric.nn.PointNetConv`, graph convolution layers, and batched k-NN search (`torch_cluster.knn_graph`) all running on CUDA or ROCm. The trained model weights (§27) can be extracted and run as a Vulkan compute shader MLP inference pass.

**VTK** ([source](https://vtk.org), BSD-3) covers §29 scientific visualization: `vtkStreamTracer` for streamlines, `vtkVolume` with GPU ray casting, `vtkTensorGlyph` for superquadrics, and `vtkMarchingCubes` for isosurface extraction. The OpenGL compute backend (`vtkOpenGLGPUVolumeRayCastMapper`) uses GL compute shaders; Vulkan integration requires the VTK-m ([source](https://m.vtk.org)) parallel framework with a Kokkos backend.

**Table D — §30–§39 coverage**

| Library | Ver | GPU Backend | §30 Hair | §31 LSM | §32 DiffRender | §33 Particles | §34 Lightmap | §35 UV Pack | §36 Swept Vol | §37 Morphing | §38 Decals | §39 Occlusion | Best Use |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| [TressFX](https://github.com/GPUOpen-Effects/TressFX) | 4.1 | Vulkan / DX12 | ✓ sim+render | — | — | — | — | — | — | — | — | — | GPU hair simulation |
| [NvCloth](https://github.com/NVIDIAGameWorks/NvCloth) | 1.1 | CUDA | partial strand | — | — | — | — | — | — | — | — | — | Cloth+strand PBD |
| [NvDiffRast](https://github.com/NVIDIAGameWorks/nvdiffrast) | 0.3 | CUDA | — | — | ✓ diff rast | — | — | — | — | — | — | — | Differentiable rasterisation |
| [Pytorch3D](https://github.com/facebookresearch/pytorch3d) | 0.7 | CUDA | — | — | ✓ diff render | — | — | — | — | ✓ functional maps | — | — | Differentiable rendering + shape |
| [xatlas](https://github.com/jpcy/xatlas) | 2.x | None (CPU) | — | — | — | — | — | ✓ atlas pack | — | — | — | — | UV atlas generation |
| [Intel OSR](https://github.com/GameTechDev/OcclusionCulling) | ISPC | CPU SIMD | — | — | — | — | — | — | — | — | — | ✓ SW occlusion | Software occlusion rasteriser |
| [Embree](https://www.embree.org) | 4.3 | AVX-512 / CPU | — | — | ✓ ray query | — | ✓ AO bake | — | — | ✓ correspondence | — | — | CPU ray tracing for bake |
| [CGAL](https://www.cgal.org) | 6.1 | None | — | — | — | — | — | ✓ seam (CPU) | ✓ Minkowski | ✓ functional maps (CPU) | — | — | Geometry preprocessing |
| [Radeon Rays](https://github.com/GPUOpen-LibrariesAndSDKs/RadeonRays_SDK) | 4.1 | Vulkan / HIP | — | — | partial | — | ✓ AO GPU | — | — | — | — | — | GPU ray casting for bake |
| [LightBaker](https://github.com/ands/lightmapper) | 1.x | OpenGL | — | — | — | — | ✓ hemicube | — | — | — | — | — | CPU/GPU lightmap bake |

**TressFX 4.1** ([source](https://github.com/GPUOpen-Effects/TressFX), MIT) is the most complete open-source GPU hair library, implementing the full §30 pipeline: position-based strand simulation (stretch, bending, torsion constraints), strand self-collision via spatial hashing, collision with scene geometry, and LOD-controlled strand-to-shell transitions. Vulkan and DX12 backends. Integrates with the engine's TLAS via `VK_KHR_ray_tracing_pipeline` for strand shadow rays.

**NvDiffRast** ([source](https://github.com/NVIDIAGameWorks/nvdiffrast), Apache 2.0) provides CUDA-accelerated differentiable rasterisation (§32.1): forward rasteriser with anti-aliased edge gradients, texture sample backward pass, and interpolation gradients. Includes a `torch.autograd.Function` wrapper. For Vulkan integration, run the forward pass in nvdiffrast (CUDA) and extract gradients into a Vulkan-mapped buffer via `VK_KHR_external_memory`.

**PyTorch3D** ([source](https://github.com/facebookresearch/pytorch3d), BSD) extends §27 (geometric deep learning) with differentiable mesh rendering (§32), functional map utilities (§37), and point cloud operations. The `pytorch3d.renderer.MeshRasterizer` implements the soft rasterisation from §32.1.

**Embree 4** ([source](https://www.embree.org), Apache 2.0) provides CPU ray tracing via AVX-512/AVX2 SIMD — the standard reference backend for offline lightmap baking (§34). `rtcOccluded1M()` batches occlusion queries for AO hemisphere sampling. Pair with a GPU denoiser (OIDN) for high-quality baked lighting.

**Radeon Rays 4** ([source](https://github.com/GPUOpen-LibrariesAndSDKs/RadeonRays_SDK), MIT) is the GPU counterpart: Vulkan and HIP ray query acceleration for AO baking (§34) and differentiable rendering (§32) on AMD hardware. Complements the `VK_KHR_ray_query` inline queries in §34.2 with a higher-level batch API.

**What is missing.** Spanning all four tables, no single library covers more than five of the thirty-nine algorithm sections. Hair simulation (§30) is the best-served new section (TressFX covers it end-to-end). Level set methods (§31), GPU particle systems (§33), swept volumes (§36), decals (§38), and portal occlusion (§39) have no portable Vulkan-native open-source library and must be implemented from first principles using the compute shader patterns in this chapter. The CUDA ecosystem (PhysX 5 + SPlisHSPlasH + gsplat + Open3D + PyTorch Geometric + NvDiffRast + NanoVDB + TressFX) now covers sixteen of the thirty-nine sections for NVIDIA hardware, but every result requires CUDA–Vulkan interop. On non-NVIDIA hardware or with a Vulkan-only constraint, the compute shader patterns across §10–§39 remain the only portable path.

---

## 41. Performance Reference

**Subdivision** (OpenSubdiv stencil evaluation, RTX 4080):

- 400k refined vertices (3 floats each), `GLComputeEvaluator`: ~0.3 ms, limited by SSBO scatter-gather bandwidth
- Custom Vulkan stencil kernel (same data): ~0.28 ms (comparable; pipeline setup overhead differs)
- Adaptive tessellation + hardware tessellator vs. uniform full-resolution subdivision: 3–5× faster at typical view distances

**Marching Cubes** (two-pass GPU MC, RTX 3070):

- 256³ voxel grid (density update + MC + output): ~2 ms total
- Prefix scan overhead: ~0.3 ms (eliminates atomic contention in Pass 2)
- Dynamic metaballs (moving centers, full density recompute): ~1.7 ms density + ~0.3 ms MC

**Skinning** (GPU compute pre-pass):

- 100k vertices, 4 bone influences, joint matrix fetch from SSBO: ~0.1 ms (RTX 3070)
- Bottleneck: scattered reads into `joint_mats[]`; packing as `mat3x4` (12 floats) instead of `mat4` (16 floats) reduces bandwidth by 25%

**IK** (1000 independent chains, 10 bones, 10 iterations):

- FABRIK: ~0.1 ms (RTX 4070)
- CCD: ~0.15 ms (FK recomputation overhead)
- Jacobian DLS (6-DOF, 6 joints): dominated by 3×3 solve; negligible per-chain, trivially parallelizable

**Mesh simplification** (meshoptimizer, CPU, single thread):

- 1M-triangle mesh, 50% target: ~30 ms (QEM edge collapse with heap)
- GPU vertex clustering (32³ grid, 500k input vertices): ~0.5 ms dispatch + ~1 ms radix sort (RTX 3070)

**LBVH construction** (GPU, RTX 3070):

- Morton code generation + 30-bit radix sort: ~0.8 ms for 500k triangles
- Tree construction (Karras algorithm): ~0.3 ms
- AABB propagation (bottom-up atomic): ~0.2 ms
- Total: ~1.3 ms for a fully rebuilt BVH on 500k triangles — suitable for dynamic geometry updated every frame

**NanoVDB sphere tracing** (fragment shader, 1080p, RTX 3070):

- 128 iterations per pixel, 256³ grid, ~4 ms (scene complexity dependent)
- NanoVDB SSBO accessor: ~2× slower than CUDA __ldg path due to lack of texture cache, but within acceptable budget for volume preview rendering

---

## 42. Integrations

- **Ch24 (Vulkan/EGL)** — Vulkan resource allocation patterns for SSBO-based subdivision and skinning buffers; pipeline barrier placement between compute passes.
- **Ch25 (GPU Compute)** — Compute shader dispatch setup, shared memory usage for marching cubes tile optimization, prefix-scan patterns for compaction, and radix sort for Morton-code-based LBVH (§8).
- **Ch127 (Mesh Shaders and VRS)** — Mesh shaders replace the tessellation pipeline for subdivision patches; the amplification shader's LOD selection (§7.4) uses cluster bounding spheres from the same meshlet data structures described in Ch127.
- **Ch133 (Vulkan Compute Queues)** — Skinning pre-pass, IK compute, and BVH rebuild should run on the async compute queue to overlap with graphics rendering; see Ch133 for synchronization between compute and graphics queues.
- **Ch135 (Vulkan Ray Tracing)** — The GPU-built LBVH (§8) feeds into `VkAccelerationStructureBuildGeometryInfoKHR` via `VK_ACCELERATION_STRUCTURE_BUILD_MODE_UPDATE_KHR` for dynamic geometry; Ch135 covers the full AS build and GLSL ray-tracing shader integration. The BLAS/TLAS pipeline in §20 of this chapter provides the geometry layer that Ch135's ray tracing shaders query.
- **Ch141 (Cooperative Matrices)** — QEF matrix accumulation in dual contouring (§3.3) and Jacobian construction in IK (§5.3) benefit from cooperative matrix operations when matrix sizes exceed the warp-level primitive dimensions. The MLP inference in §27 (PointNet, neural SDF) maps directly to cooperative matrix multiply-accumulate patterns.
- **Ch154 (GPU-Driven Rendering)** — GPU-driven pipelines require that geometry (subdivision patches, skinned meshes, LOD index buffers) be finalized in compute before indirect draw commands are issued; see Ch154's indirect dispatch patterns. The task-shader crowd rendering in §23.4 uses the same indirect draw compaction flow as Ch154's two-phase occlusion pass.
- **Ch204 (Shader Algorithm Catalog)** — Reference for quaternion arithmetic primitives (`quat_mul`, `quat_from_axis_angle`) used in CCD and DQS shaders; also covers RK4 integration primitives reused in §26.4 heightfield collision and §29.1 streamline tracing.
- **Ch209 (GPU Simulation Pipelines)** — The fluid (§11), deformable body (§12), rigid body (§22), fracture (§25), and crowd (§23) sections of this chapter provide the algorithm kernels; Ch209 covers how to assemble these kernels into a full simulation pipeline with shared memory layouts, double-buffering, and cross-queue synchronization. Hair strand simulation (§30) and level set evolution (§31) connect to the same pipeline architecture.
- **Ch210 (Terrain and Streaming)** — The CDLOD quadtree (§26.1), sparse clipmap (§26.3), and heightfield ray intersection (§26.4) in this chapter are the GPU geometry algorithms; Ch210 covers the full terrain system including heightmap streaming, virtual texture feedback, and integration with the Vulkan sparse binding API.
- **Ch135 (Vulkan Ray Tracing, lightmap baking)** — §34 uses `VK_KHR_ray_query` inline queries for per-texel AO baking; Ch135 covers the full RT pipeline, acceleration structure lifetime management, and shader binding table layout needed to build the bake pipeline.
- **Ch152 (Rust GPU)** — Differentiable rendering (§32) and geometric deep learning (§27) are active areas for GPU-accelerated training in Rust; the neural SDF inference kernel from §27.3 maps directly to rust-gpu compute shaders for Vulkan-native ML inference.
- **Ch154 (GPU-Driven Rendering, visibility)** — The software occlusion rasteriser (§39.2) and portal BFS (§39.3) feed into the two-phase occlusion pass (§10.4) described in Ch154; §39 provides the culling query algorithms, Ch154 provides the indirect draw compaction that acts on their results.
