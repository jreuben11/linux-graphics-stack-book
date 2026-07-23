# Chapter 208: GPU Geometry Algorithms — Subdivision, Implicit Surfaces, Skinning, and IK

**Audiences:** Graphics application developers implementing character animation, procedural geometry, or physics-driven deformation pipelines on Linux using Vulkan or OpenGL compute.

**CPU vs. GPU split.** Roughly 70% of the material in this chapter is GPU-centric: algorithms that are data-parallel over vertices, triangles, or voxels map directly to Vulkan compute shaders and run entirely on the GPU after an initial dispatch. The remaining 30% is CPU-centric or hybrid, falling into two categories:

*CPU-only:* Mature production libraries whose APIs are inherently serial or topology-dependent — OpenSubdiv's Far layer (`TopologyRefiner`, `StencilTableFactory`), OpenCASCADE's `BRepMesh_IncrementalMesh`, meshoptimizer's simplification and encode/decode passes, and VHACD convex decomposition. These run on the CPU and upload their results (index buffers, stencil tables, convex hull vertex sets) as Vulkan buffers for GPU consumption.

*Hybrid:* Operations with a fast parallel component and a slower serial or data-dependent component. Examples: OpenSubdiv (CPU Far → GPU Osd evaluator), Poisson surface reconstruction (CPU reference library vs. GPU splat/solve/MC pipeline), mesh Boolean operations (GPU BVH broad-phase and clipping kernel, CPU global inside/outside classification), and constrained Delaunay triangulation (CPU Triangle library for constraint insertion, GPU for Ruppert refinement). In all hybrid cases the chapter shows the GPU-accelerable portion in compute shader code and notes which steps remain on CPU.

The sections below are ordered by geometry domain rather than CPU/GPU split. Where a section covers a CPU library, the practical GPU integration pattern (data layout for upload, barrier placement, descriptor set structure) is always shown alongside the library API.

---

## Table of Contents

- **[I. Surface Representation and Modeling](#i-surface-representation-and-modeling)**
  - [1. Subdivision Surfaces on the GPU](#1-subdivision-surfaces-on-the-gpu)
  - [2. NURBS and B-Spline Surfaces](#2-nurbs-and-b-spline-surfaces)
  - [3. Metaballs and Implicit Surfaces](#3-metaballs-and-implicit-surfaces)
  - [4. Spline Curves: GPU Tessellation](#4-spline-curves-gpu-tessellation)
  - [5. 2D Vector Graphics on GPU](#5-2d-vector-graphics-on-gpu)
  - [6. GPU Curve Rendering for CAD and Precision Graphics](#6-gpu-curve-rendering-for-cad-and-precision-graphics)
  - [7. GPU Boolean Mesh Operations (CSG)](#7-gpu-boolean-mesh-operations-csg)
  - [8. Implicit Surface Blending: MetaBalls and SDF Primitives](#8-implicit-surface-blending-metaballs-and-sdf-primitives)
  - [9. Poisson Surface Reconstruction](#9-poisson-surface-reconstruction)
- **[II. Mesh Processing and Topology](#ii-mesh-processing-and-topology)**
  - [10. GPU Mesh Simplification and LOD](#10-gpu-mesh-simplification-and-lod)
  - [11. GPU Remeshing](#11-gpu-remeshing)
  - [12. UV Seam Optimization and Atlas Packing](#12-uv-seam-optimization-and-atlas-packing)
  - [13. Mesh Voxelisation](#13-mesh-voxelisation)
  - [14. Morphological Operations and Medial Axis](#14-morphological-operations-and-medial-axis)
  - [15. Progressive Meshes and Geometry Streaming](#15-progressive-meshes-and-geometry-streaming)
  - [16. Mesh Compression and Streaming Codecs](#16-mesh-compression-and-streaming-codecs)
  - [17. Mesh Parameterization and UV Unwrapping](#17-mesh-parameterization-and-uv-unwrapping)
  - [18. GPU Delaunay Triangulation](#18-gpu-delaunay-triangulation)
  - [19. Mesh Fairing and Geometric Smoothing](#19-mesh-fairing-and-geometric-smoothing)
  - [20. GPU-Accelerated Remeshing](#20-gpu-accelerated-remeshing)
  - [21. GPU Mesh Segmentation and Part Decomposition](#21-gpu-mesh-segmentation-and-part-decomposition)
  - [22. GPU Mesh Repair and Waterproofing](#22-gpu-mesh-repair-and-waterproofing)
  - [23. Constrained Delaunay Refinement](#23-constrained-delaunay-refinement)
- **[III. Spatial Data Structures and Visibility](#iii-spatial-data-structures-and-visibility)**
  - [24. GPU BVH Construction](#24-gpu-bvh-construction)
  - [25. Screen-Space Geometry Techniques](#25-screen-space-geometry-techniques)
  - [26. GPU-Driven Rendering and Virtual Geometry](#26-gpu-driven-rendering-and-virtual-geometry)
  - [27. Spatial Data Structures Beyond BVH](#27-spatial-data-structures-beyond-bvh)
  - [28. Precomputed Visibility and Lightmap Baking](#28-precomputed-visibility-and-lightmap-baking)
  - [29. Visibility Queries and GPU Occlusion](#29-visibility-queries-and-gpu-occlusion)
  - [30. Sparse Voxel DAG (SVDAG)](#30-sparse-voxel-dag-svdag)
  - [31. GPU BVH Construction](#31-gpu-bvh-construction)
  - [32. SDF Baking and 3D Jump Flooding](#32-sdf-baking-and-3d-jump-flooding)
  - [33. Virtual Geometry: GPU-Driven Cluster LOD](#33-virtual-geometry-gpu-driven-cluster-lod)
- **[IV. Differential Geometry and Analysis](#iv-differential-geometry-and-analysis)**
  - [34. Geodesics and Discrete Differential Geometry](#34-geodesics-and-discrete-differential-geometry)
  - [35. Spectral Geometry Processing](#35-spectral-geometry-processing)
  - [36. Geometric Shape Morphing and Functional Maps](#36-geometric-shape-morphing-and-functional-maps)
  - [37. Geodesic Distance: Heat Method](#37-geodesic-distance-heat-method)
  - [38. Spectral Mesh Processing](#38-spectral-mesh-processing)
- **[V. Animation, Skinning, and Deformation](#v-animation-skinning-and-deformation)**
  - [39. Skeletal Animation: Skinning](#39-skeletal-animation-skinning)
  - [40. Inverse Kinematics on the GPU](#40-inverse-kinematics-on-the-gpu)
  - [41. Shape Deformation Beyond Skinning](#41-shape-deformation-beyond-skinning)
  - [42. Hair and Strand Geometry](#42-hair-and-strand-geometry)
  - [43. Secondary Animation Dynamics](#43-secondary-animation-dynamics)
  - [44. Skeletal Mesh Retargeting](#44-skeletal-mesh-retargeting)
  - [45. Soft Body FEM and Projective Dynamics](#45-soft-body-fem-and-projective-dynamics)
  - [46. Cage-Based Deformation: Mean Value and Green Coordinates](#46-cage-based-deformation-mean-value-and-green-coordinates)
  - [47. Laplacian Mesh Editing and Interactive Deformation](#47-laplacian-mesh-editing-and-interactive-deformation)
- **[VI. Physics and Simulation](#vi-physics-and-simulation)**
  - [48. Fluid Simulation on the GPU](#48-fluid-simulation-on-the-gpu)
  - [49. Deformable Bodies and Continuum Mechanics](#49-deformable-bodies-and-continuum-mechanics)
  - [50. Rigid Body Dynamics on GPU](#50-rigid-body-dynamics-on-gpu)
  - [51. Crowd and Multi-Agent Simulation](#51-crowd-and-multi-agent-simulation)
  - [52. Fracture and Destruction](#52-fracture-and-destruction)
  - [53. Rope and Cable Simulation](#53-rope-and-cable-simulation)
  - [54. GPU Cloth Simulation](#54-gpu-cloth-simulation)
  - [55. GPU Fluid-Solid Coupling](#55-gpu-fluid-solid-coupling)
  - [56. Granular Material Simulation (DEM)](#56-granular-material-simulation-dem)
  - [57. Terrain Sculpting and Hydraulic Erosion](#57-terrain-sculpting-and-hydraulic-erosion)
  - [58. GPU Fluid Surface Extraction](#58-gpu-fluid-surface-extraction)
  - [59. Continuous Collision Detection (CCD)](#59-continuous-collision-detection-ccd)
  - [60. Minkowski Sum and GPU Motion Planning](#60-minkowski-sum-and-gpu-motion-planning)
- **[VII. SDF and Volumetric Methods](#vii-sdf-and-volumetric-methods)**
  - [61. Level Set Methods](#61-level-set-methods)
  - [62. Swept Volumes and Minkowski Sums](#62-swept-volumes-and-minkowski-sums)
  - [63. Volumetric 3D Reconstruction (TSDF Fusion)](#63-volumetric-3d-reconstruction-tsdf-fusion)
  - [64. SDF-Based Collision Detection](#64-sdf-based-collision-detection)
  - [65. Isosurface Extraction](#65-isosurface-extraction)
  - [66. GPU Ambient Occlusion](#66-gpu-ambient-occlusion)
  - [67. Volumetric Ray Marching](#67-volumetric-ray-marching)
  - [68. GPU Sparse Narrow-Band Level-Set Evolution](#68-gpu-sparse-narrow-band-level-set-evolution)
- **[VIII. Terrain, Procedural Content, and Environment](#viii-terrain-procedural-content-and-environment)**
  - [69. Procedural Geometry Generation](#69-procedural-geometry-generation)
  - [70. Navigation Meshes and GPU Pathfinding](#70-navigation-meshes-and-gpu-pathfinding)
  - [71. Terrain LOD and Streaming](#71-terrain-lod-and-streaming)
  - [72. GPU Particle Systems and Ribbon Geometry](#72-gpu-particle-systems-and-ribbon-geometry)
  - [73. GPU Ocean and Wave Simulation](#73-gpu-ocean-and-wave-simulation)
  - [74. GPU Geospatial Processing: Large-Scale Terrain](#74-gpu-geospatial-processing-large-scale-terrain)
- **[IX. Ray Tracing and Optical Geometry](#ix-ray-tracing-and-optical-geometry)**
  - [75. Ray Tracing Geometry Pipeline](#75-ray-tracing-geometry-pipeline)
  - [76. Shadow Geometry Algorithms](#76-shadow-geometry-algorithms)
  - [77. IBL and Environment Map Preprocessing](#77-ibl-and-environment-map-preprocessing)
  - [78. Global Illumination Geometry](#78-global-illumination-geometry)
  - [79. Geometric Optics: Lens, Caustics, and Beam Tracing](#79-geometric-optics-lens-caustics-and-beam-tracing)
  - [80. Screen-Space Reflections (SSR/SSSR)](#80-screen-space-reflections-ssrsssr)
  - [81. GPU Radiosity: Form Factor Computation](#81-gpu-radiosity-form-factor-computation)
  - [82. GPU Path Tracing Pipeline](#82-gpu-path-tracing-pipeline)
  - [83. Atmospheric Scattering and Sky Rendering](#83-atmospheric-scattering-and-sky-rendering)
  - [84. Subsurface Scattering Geometry](#84-subsurface-scattering-geometry)
  - [85. Order-Independent Transparency](#85-order-independent-transparency)
  - [86. GPU Polygon and Frustum Clipping](#86-gpu-polygon-and-frustum-clipping)
- **[X. Point Cloud, Reconstruction, and Perception](#x-point-cloud-reconstruction-and-perception)**
  - [87. Point Cloud Processing](#87-point-cloud-processing)
  - [88. GPU Structure from Motion and Multi-View Stereo](#88-gpu-structure-from-motion-and-multi-view-stereo)
  - [89. 3D Feature Descriptors and Pose Estimation](#89-3d-feature-descriptors-and-pose-estimation)
  - [90. Iterative Closest Point (ICP) Registration](#90-iterative-closest-point-icp-registration)
- **[XI. Neural and Learned Geometry](#xi-neural-and-learned-geometry)**
  - [91. 3D Gaussian Splatting and Neural Geometry](#91-3d-gaussian-splatting-and-neural-geometry)
  - [92. Geometric Deep Learning](#92-geometric-deep-learning)
  - [93. Differentiable Rendering and Inverse Geometry](#93-differentiable-rendering-and-inverse-geometry)
  - [94. 3D Gaussian Splatting](#94-3d-gaussian-splatting)
  - [95. Neural Geometry: NeRF and Neural Implicit Surfaces](#95-neural-geometry-nerf-and-neural-implicit-surfaces)
- **[XII. Specialized and Cross-Domain Applications](#xii-specialized-and-cross-domain-applications)**
  - [96. Scientific Visualization Geometry](#96-scientific-visualization-geometry)
  - [97. Decal and Surface Projection Geometry](#97-decal-and-surface-projection-geometry)
  - [98. GPU Texture Synthesis and Procedural Shading Geometry](#98-gpu-texture-synthesis-and-procedural-shading-geometry)
  - [99. Acoustic Ray Tracing and Sound Geometry](#99-acoustic-ray-tracing-and-sound-geometry)
  - [100. VR/XR Geometry: Reprojection and Foveation](#100-vrxr-geometry-reprojection-and-foveation)
  - [101. Silhouette Detection and Feature Lines](#101-silhouette-detection-and-feature-lines)
  - [102. Micro-Polygon Displacement and GPU Tessellation](#102-micro-polygon-displacement-and-gpu-tessellation)
  - [103. GPU SDF Font and Curve Rendering](#103-gpu-sdf-font-and-curve-rendering)
  - [104. Molecular Surface Computation](#104-molecular-surface-computation)
- **[XIII. GPU Algorithm Primitives for Geometry](#xiii-gpu-algorithm-primitives-for-geometry)**
  - [105. GPU Sampling: Poisson Disk and Blue Noise](#105-gpu-sampling-poisson-disk-and-blue-noise)
  - [106. GPU Convex Hull Computation](#106-gpu-convex-hull-computation)
  - [107. GPU Radix Sort as a Geometry Primitive](#107-gpu-radix-sort-as-a-geometry-primitive)


---

## I. Surface Representation and Modeling

Covers the mathematical primitives used to represent smooth, curved, and implicit geometry on the GPU: Catmull-Clark and Loop subdivision surfaces that refine coarse control meshes toward smooth limit surfaces; parametric NURBS and Bézier patches used in CAD and precision engineering visualization; implicit surfaces defined by scalar field functions (metaballs, SDF CSG); Bézier and B-spline curve tessellation; and 2D vector graphics rendered via GPU fragment shaders. These algorithms determine the geometric representation before any simulation, texturing, or shading is applied.

### 1. Subdivision Surfaces on the GPU

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
// 1. Compute skinned_base_positions[] via GPU compute pre-pass (§39.3)
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

### 2. NURBS and B-Spline Surfaces

NURBS (Non-Uniform Rational B-Splines) are the standard surface representation in CAD, engineering, and industrial design — they describe smooth curves and surfaces exactly, without the approximation error introduced by triangle tessellation. GPU evaluation requires reformulating the Cox-de Boor recursion into parallel form: each thread evaluates one basis function independently, then a weighted sum (rational NURBS) or barycentric combination (tensor-product B-spline patches) assembles the surface point. Tessellation shaders or compute dispatches emit the resulting triangle mesh at a level of detail matched to screen-space curvature and, for engineering applications, to a configurable chord-tolerance threshold.

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

### 3. Metaballs and Implicit Surfaces

Metaballs and implicit surfaces represent geometry as the iso-level of a scalar field f(x,y,z) = c, where the visible surface is the set of points where f equals a chosen threshold. Metaballs define f as a sum of radially symmetric falloff kernels centred on control points, producing organic blended shapes that merge and separate as control points move. On the GPU the scalar field is evaluated across a 3D voxel grid in a compute shader, and isosurface extraction (Marching Cubes, Dual Contouring, or Surface Nets) converts it to a triangle mesh each frame. SDF-based CSG — building complex shapes by composing min, max, and complement operations on primitive signed-distance fields — is a generalization of the same idea and is covered alongside metaballs here because it shares the same evaluation and extraction pipeline.

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
5. **Smooth normals** — §10.5 post-pass on the extracted mesh
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

**Applications.** GPU Voronoi/Delaunay is used in: procedural biome generation (each Voronoi cell = one biome with coherent terrain parameters), fluid mesh reconstruction (Poisson surface reconstruction seeds from SPH particles), and GPU-side LOD meshing where scattered high-frequency samples (from adaptive tessellation) need triangulation before mesh simplification (§10.1). [Source: Hoff et al. "Fast Computation of Generalized Voronoi Diagrams Using Graphics Hardware", SIGGRAPH 1999; Rong & Tan "Jump Flooding in GPU with Applications to Voronoi Diagram and Distance Transform", I3D 2006]

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

1. **Octree construction** — build an adaptive octree over the point cloud; deeper cells near high-point-density regions. One approach: Morton-sort points (§24.1 pattern), then split cells that exceed a count threshold.

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

4. **Poisson solve** — solve ∇²χ = div on the grid. On GPU, a Conjugate Gradient solver (same pattern as §10.12 CG step) handles the sparse 7-point stencil Laplacian. For large grids (256³+), a multigrid V-cycle reduces iteration count from O(N²) to O(N log N):

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

### 4. Spline Curves: GPU Tessellation

B-spline and Bézier curve tessellation converts a compact control-point representation into dense polyline or ribbon geometry suitable for rasterization. The central challenge is adaptive tessellation — generating more vertices where curvature is high or the curve is close to the camera, and fewer elsewhere — without a serial dependency between curve segments. Tessellation shaders (hull + domain stage) are the native API path for Bézier patches; compute-shader approaches writing to vertex buffers offer more flexibility for non-standard curve types such as Catmull-Rom splines, cubic Hermite curves used in animation paths, and NURBS curves from CAD data. Both paths are covered here together with the screen-space flatness metric that drives LOD selection.

### 4.1 Catmull-Rom Splines

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

### 4.2 Cubic Hermite and B-Spline Curves

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

### 4.3 Tube Geometry via Compute

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

### 4.4 Arc-Length Reparameterization

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

    // Evaluate spline at tLocal (same as §4.1 TES logic)
    // ...
    framePos[frame] = vec4(evaluatedPos, 1.0);
}
```

The cumulative arc-length table is computed once (or whenever control points change) and reused for all frame queries. For real-time camera paths, M = 1024 samples per segment gives sub-millimeter accuracy at typical scene scales.

---

### 5. 2D Vector Graphics on GPU

*Audience: graphics application developers.*

GPU geometry algorithms are not exclusive to 3D. 2D vector graphics — Bézier curves, fonts, strokes, filled paths — have their own GPU-native rendering techniques that avoid tessellation entirely.

### 5.1 Loop-Blinn Bézier Rendering

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

### 5.2 MSDF Font Atlas Rendering

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

### 5.3 Stroke Expansion in Compute

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

### 5.4 Stencil-Then-Cover for Filled Paths

Fill arbitrary paths without tessellation: (1) stencil pass renders a triangle fan from the path origin, incrementing/decrementing stencil per edge crossing direction; (2) cover pass draws a bounding quad with stencil test — only non-zero stencil pixels get filled. Implemented via `vkCmdSetStencilOp` and two draw calls. [Source: Kilgard & Bolz "GPU-accelerated Path Rendering", SIGGRAPH Asia 2012]

---

### 6. GPU Curve Rendering for CAD and Precision Graphics

*Audience: graphics application developers, systems developers.*

§2 covered NURBS *tessellation* to triangle meshes for rendering. CAD and precision graphics require *exact* curve and trimmed surface rendering — displaying NURBS without tessellation error, handling trimming boundaries pixel-precisely, and rendering parametric curves at any zoom level without LOD artifacts.

### 6.1 Direct NURBS Rasterization via Ray Casting

Ray cast each screen pixel against the NURBS surface by inverting the parametric mapping (Newton iteration). The fragment shader solves for (u,v) such that S(u,v) = p_ray:

```glsl
// nurbs_direct.frag — Newton iteration for pixel-exact NURBS display
uniform mat4  ctrlPts[MAX_DEG+1][MAX_DEG+1];
uniform float knotsU[MAX_KNOTS_U], knotsV[MAX_KNOTS_V];
in vec3 rayDir;
void main() {
    vec2 uv = vec2(0.5, 0.5);  // initial parameter estimate
    for (int iter=0; iter<MAX_ITER; iter++) {
        vec3  S, Su, Sv;
        evalNURBS(ctrlPts, knotsU, knotsV, uv, S, Su, Sv);
        vec3 err = S - rayOrigin - gl_FragCoord.z * rayDir;
        mat2 J   = mat2(dot(Su, Su), dot(Su, Sv),
                        dot(Su, Sv), dot(Sv, Sv));
        vec2 rhs = vec2(dot(err, Su), dot(err, Sv));
        uv      -= inverse(J) * rhs;
        uv       = clamp(uv, vec2(0), vec2(1));
        if (length(err) < 1e-6) break;
    }
    // If converged and uv in domain: shade; else discard
    if (!inTrimRegion(uv)) discard;
    gl_FragDepth = projectToDepth(evalNURBSPoint(ctrlPts, knotsU, knotsV, uv));
}
```

[Source: Guthe et al. "GPU-Based Trimmed NURBS Rendering", IEEE VIS 2005; Loop & Blinn "Real-Time GPU Rendering of Piecewise Algebraic Surfaces", SIGGRAPH 2006]

### 6.2 Trimming Curves via Stencil

NURBS trimming boundaries are 2D curves in (u,v) parameter space. Rasterize them into a trim mask using the stencil approach from §5.4 (stencil-then-cover), applied in UV parameter space:

```glsl
// trim_stencil.vert — project trim curve segment to parameter space
layout(location=0) in vec2 paramPos;  // (u,v) control points
void main() {
    // Map (u,v) → NDC: trim mask rendered into a 1024×1024 texture
    gl_Position = vec4(paramPos*2.0 - 1.0, 0.5, 1.0);
}
// Render with stencil increment/decrement; then test trim mask in §6.1 fragment shader
```

[Source: Moreton "Watertight Tessellation Using Forward Differencing", HWWS 2001; Steinberg et al. "GPU-Accelerated Trimmed NURBS", IEEE CAD 2023]

### 6.3 Offset Curves and Parallel Curves for Machining

Offset curves (for CNC tool radius compensation) are not NURBS — they require approximation. GPU-accelerated piecewise Bézier offset: sample the original curve at N points, compute offset points by moving along the normal, then fit a Bézier to the offset samples:

```glsl
// offset_curve.comp — sample NURBS curve, offset by tool radius
layout(set=0,binding=0) readonly buffer NURBSKnots { float knots[]; };
layout(set=0,binding=1) readonly buffer CtrlPts    { vec2  cp[];    };
layout(set=0,binding=2) writeonly buffer OffsetPts { vec2  opt[];   };
layout(push_constant) uniform PC { float r; uint N; } pc;
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    float t = float(i) / float(pc.N-1);
    vec2  C, Ct;
    evalNURBS2D(cp, knots, t, C, Ct);
    vec2  n   = normalize(vec2(-Ct.y, Ct.x));  // normal to tangent
    opt[i]    = C + pc.r * n;
}
// Follow with GPU Bézier least-squares fit (§105.2 area sampling + linear solve)
```

[Source: Tiller & Hanson "Offsets of Two-Dimensional Profiles", IEEE CGA 1984; Piegl & Tiller "The NURBS Book", 2nd ed., §8.4]

---

### 7. GPU Boolean Mesh Operations (CSG)

*Audience: graphics application developers, systems developers.*

Constructive Solid Geometry (CSG) computes the union, intersection, or difference of two closed meshes. GPU algorithms use either depth-peeling for screen-space CSG or BSP-tree decomposition for exact mesh-to-mesh boolean operations.

### 7.1 Depth-Peeling CSG

Depth peeling (Everitt 2001) renders the scene in multiple passes, each time extracting the next depth layer. CSG boolean operations are implemented by combining pairs of depth layers from two meshes:

```glsl
// csg_depthpeel.frag — fragment shader: CSG union via depth-layer compositing
// For A ∪ B: at each pixel, output the minimum first-hit depth from A or B
// Implemented via N depth-peeling passes accumulating RGBA + depth per layer
uniform sampler2D prevDepthA;  // previous layer depth for object A
uniform sampler2D prevDepthB;  // previous layer depth for object B
in float fragDepth;
uniform int pass;  // which peel pass (0 = first layer)
void main() {
    float depA = texture(prevDepthA, gl_FragCoord.xy/SCREEN_SIZE).r;
    float depB = texture(prevDepthB, gl_FragCoord.xy/SCREEN_SIZE).r;
    // Discard if this fragment is not the next layer beyond the previous peel
    if (pass > 0 && fragDepth <= depA + 1e-5 && objectID == A_ID) discard;
    if (pass > 0 && fragDepth <= depB + 1e-5 && objectID == B_ID) discard;
    // Union: emit fragment if inside A XOR inside B XOR boolean rule
    gl_FragDepth = fragDepth;
}
```

[Source: Everitt "Interactive Order-Independent Transparency", NVIDIA Whitepaper 2001; Kirsch & Döllner "Rendering Techniques for Hardware-Accelerated Image-Space CSG", WSCG 2004]

### 7.2 Mesh BSP Boolean Operations

For exact (non-screen-space) CSG, decompose each mesh into a BSP tree, then evaluate the boolean predicate recursively. GPU parallelises the per-triangle BSP-insert test:

```glsl
// bsp_classify.comp — classify triangles of mesh B against BSP planes of mesh A
layout(set=0,binding=0) readonly buffer BSPPlanes { vec4 planes[]; };  // of mesh A
layout(set=0,binding=1) readonly buffer TriVerts  { vec3 verts[];  };  // of mesh B
layout(set=0,binding=2) writeonly buffer Classification { uint cls[]; };  // INSIDE/OUTSIDE/SPLIT per tri
layout(local_size_x=64) in;
void main() {
    uint t   = gl_GlobalInvocationID.x;
    vec3 v0  = verts[t*3], v1 = verts[t*3+1], v2 = verts[t*3+2];
    uint cls_out = OUTSIDE;
    // Walk BSP tree: test all three vertices against splitting plane
    for (uint p=0; p<N_PLANES; p++) {
        vec4 pl  = planes[p];
        float d0 = dot(pl.xyz, v0)+pl.w, d1=dot(pl.xyz,v1)+pl.w, d2=dot(pl.xyz,v2)+pl.w;
        if (d0<0&&d1<0&&d2<0) cls_out=INSIDE;
        else if (d0>0||d1>0||d2>0) cls_out=OUTSIDE;
        else cls_out=SPLIT;
    }
    cls[t] = cls_out;
}
```

[Source: Naylor et al. "Merging BSP Trees Yields Polyhedral Set Operations", SIGGRAPH 1990; Bernstein & Fussell "Fast, Exact, Linear Booleans", SGP 2009]

### 7.3 Cork-Style Winding Number CSG

The winding-number approach (Jacobson et al. 2013) avoids BSP trees: for each triangle of mesh B, determine inside/outside relative to mesh A by evaluating the generalised winding number W(p) at the centroid. GPU evaluation of W(p) for all triangles simultaneously:

```glsl
// winding_number.comp — generalized winding number for triangle centroid
layout(set=0,binding=0) readonly buffer MeshA_Tris { vec3 aVerts[]; };
layout(set=0,binding=1) readonly buffer QueryPts   { vec3 qpts[];   };
layout(set=0,binding=2) writeonly buffer Winding   { float W[];     };
layout(local_size_x=64) in;
void main() {
    uint q  = gl_GlobalInvocationID.x;
    vec3 p  = qpts[q];
    float w = 0.0;
    for (uint t=0; t<N_TRIS_A; t++) {
        vec3 a = aVerts[t*3]-p, b = aVerts[t*3+1]-p, c = aVerts[t*3+2]-p;
        float la=length(a), lb=length(b), lc=length(c);
        // Solid angle subtended by triangle (Van Oosterom & Strackee 1983)
        float num = dot(a, cross(b, c));
        float den = la*lb*lc + dot(a,b)*lc + dot(b,c)*la + dot(a,c)*lb;
        w += 2.0 * atan(num, den);
    }
    W[q] = w / (4.0*3.14159);
}
```

[Source: Jacobson et al. "Robust Inside-Outside Segmentation Using Generalized Winding Numbers", SIGGRAPH 2013; Cork CSG: https://github.com/gilbo/cork]

### 7.4 Real-Time Destruction via Voronoi Fracture

CSG applied to destruction: pre-fracture a mesh into Voronoi cells (§52 adjacency graph), then use depth-peeling CSG at runtime to reveal internal surfaces when a cell breaks:

```glsl
// fracture_reveal.comp — mark Voronoi cells as broken, emit internal face geometry
layout(set=0,binding=0) readonly buffer VoronoiCells { VoronoiCell cells[]; };
layout(set=0,binding=1) readonly buffer ImpactPt     { vec3 impact; float force; };
layout(set=0,binding=2) buffer BrokenMask { uint broken[]; };
layout(set=0,binding=3) buffer DrawCmd    { DrawIndexedIndirectCommand cmds[]; };
layout(local_size_x=64) in;
void main() {
    uint c = gl_GlobalInvocationID.x;
    float d = length(cells[c].centroid - impact);
    float fractureDist = 0.5 * force;  // impact-force-derived radius
    if (d < fractureDist) {
        if (atomicOr(broken[c/32], 1u<<(c%32)) == 0u) {
            // Enable draw command for internal faces of this cell
            cmds[c].instanceCount = 1u;
        }
    }
}
```

[Source: Parker & O'Brien "Real-Time Deformation and Fracture in a Game Context", SCA 2009; Müller et al. "Unified Particle Physics for Real-Time Applications", SIGGRAPH 2014]

---

### 8. Implicit Surface Blending: MetaBalls and SDF Primitives

*Audience: graphics application developers.*

SDF primitives — spheres, capsules, tori, rounded boxes — are analytically defined functions φ(p) returning the signed distance from any point p to the surface. Composing them via smooth-minimum (smin) blends shapes without discrete topology. This procedural SDF modelling is the basis of character shapes in games (Clayxels, ZBrush DynaMesh), real-time destruction, and fluid surface reconstruction (§48 FLIP → metaball surface).

### 8.1 Analytic SDF Primitives

```glsl
// sdf_primitives.glsl — common analytic SDF functions
float sdSphere(vec3 p, float r)         { return length(p) - r; }
float sdCapsule(vec3 p, vec3 a, vec3 b, float r) {
    vec3 pa=p-a, ba=b-a;
    float h=clamp(dot(pa,ba)/dot(ba,ba),0.0,1.0);
    return length(pa-ba*h)-r;
}
float sdTorus(vec3 p, vec2 t) {
    vec2 q = vec2(length(p.xz)-t.x, p.y);
    return length(q)-t.y;
}
float sdRoundBox(vec3 p, vec3 b, float r) {
    vec3 q = abs(p)-b;
    return length(max(q,0.0))+min(max(q.x,max(q.y,q.z)),0.0)-r;
}
// Smooth minimum — blends two SDFs with radius k
float smin(float a, float b, float k) {
    float h = clamp(0.5+0.5*(b-a)/k, 0.0, 1.0);
    return mix(b, a, h) - k*h*(1.0-h);
}
```

[Source: Quilez "Inigo Quilez — SDF Primitives" https://iquilezles.org/articles/distfunctions/]

### 8.2 Metaball Field Composition

A metaball field sums falloff contributions from N sphere centres. The field value at p is the sum of Gaussian-like falloffs; the isosurface (§65 marching cubes at field=1) gives the blobby shape:

```glsl
// metaball_field.comp — evaluate metaball field over a voxel grid
layout(set=0,binding=0) readonly buffer Balls { vec4 balls[]; };  // xyz=centre, w=strength
layout(set=0,binding=1, r32f) uniform writeonly image3D fieldOut;
layout(push_constant) uniform PC { vec3 gridMin; float dx; ivec3 dim; uint N; } pc;
layout(local_size_x=4,local_size_y=4,local_size_z=4) in;
void main() {
    ivec3 id  = ivec3(gl_GlobalInvocationID);
    vec3  p   = pc.gridMin + (vec3(id)+0.5)*pc.dx;
    float f   = 0.0;
    for (uint b=0; b<pc.N; b++) {
        float d2 = dot(p-balls[b].xyz, p-balls[b].xyz);
        float r2 = balls[b].w*balls[b].w;
        if (d2 < r2) f += balls[b].w * (1.0 - d2/r2) * (1.0 - d2/r2);  // Blinn kernel
    }
    imageStore(fieldOut, id, vec4(f));
}
// Then run §65.1 marching cubes at isovalue=1.0 to extract the metaball surface
```

[Source: Blinn "A Generalization of Algebraic Surface Drawing", ACM TOG 1982; Wyvill et al. "Data Structure for Soft Objects", The Visual Computer 1986]

### 8.3 SDF-Based Character Body Assembly

Assemble a character body from per-bone SDF capsules (§8.1), blending adjacent bones with smin. Each bone capsule is defined by the skinned joint endpoints (§39 output):

```glsl
// character_sdf.comp — assemble character body SDF from bone capsules
layout(set=0,binding=0) readonly buffer BoneEndpoints { vec4 boneA[]; vec4 boneB[]; };
layout(set=0,binding=1, r32f) uniform writeonly image3D characterSDF;
layout(push_constant) uniform PC { vec3 gridMin; float dx; uint nBones; float blendK; } pc;
layout(local_size_x=4,local_size_y=4,local_size_z=4) in;
void main() {
    ivec3 id = ivec3(gl_GlobalInvocationID);
    vec3  p  = pc.gridMin + (vec3(id)+0.5)*pc.dx;
    float d  = 1e30;
    for (uint b=0; b<pc.nBones; b++) {
        float dc = sdCapsule(p, boneA[b].xyz, boneB[b].xyz, boneA[b].w);
        d = smin(d, dc, pc.blendK);
    }
    imageStore(characterSDF, id, vec4(d));
}
// Character body SDF supports: §64 collision, §7 CSG destruction, §65 mesh extraction
```

[Source: Turk & O'Brien "Shape Transformation Using Variational Implicit Functions", SIGGRAPH 1999; SDF character: Perlin "An Image Synthesizer", SIGGRAPH 1985; modern use: Clayxels Unity plugin]

### 8.4 Real-Time SDF Ray Marching

Render the assembled SDF scene (§8.3 character + §8.1 primitives + §62 Minkowski sum obstacles) via sphere tracing (Lipschitz ray marching), stepping by φ(p) at each sample:

```glsl
// sphere_trace.frag — sphere tracing through composite SDF scene
uniform vec3 camPos; uniform vec3 lightDir;
in vec3 rayDir;
float sceneSDF(vec3 p) {
    float d = 1e30;
    d = smin(d, sdSphere(p-vec3(0,1,0), 1.0), 0.3);
    d = smin(d, sdCapsule(p, vec3(-1,0,0), vec3(1,0,0), 0.3), 0.2);
    d = min(d, p.y);  // ground plane
    return d;
}
void main() {
    vec3  p = camPos; float t = 0.0;
    for (int i=0; i<128; i++) {
        float d = sceneSDF(p);
        if (d < 1e-4) break;
        if (t > 100.0) discard;
        p += rayDir * d; t += d;
    }
    vec3 n = normalize(vec3(
        sceneSDF(p+vec3(1e-3,0,0))-sceneSDF(p-vec3(1e-3,0,0)),
        sceneSDF(p+vec3(0,1e-3,0))-sceneSDF(p-vec3(0,1e-3,0)),
        sceneSDF(p+vec3(0,0,1e-3))-sceneSDF(p-vec3(0,0,1e-3))));
    fragColor = vec4(max(0.0,dot(n,lightDir))*vec3(1), 1);
}
```

[Source: Hart "Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces", The Visual Computer 1996; Quilez "Raymarching Distance Fields" https://iquilezles.org/articles/raymarchingdf/]

---

### 9. Poisson Surface Reconstruction

*Audience: systems developers, graphics application developers.*

Poisson surface reconstruction (Kazhdan et al. 2006, screened variant 2013) converts an oriented point cloud (positions + normals) into a watertight triangle mesh. It solves a Poisson equation on an adaptively refined octree, then extracts the isosurface (§65.1 marching cubes). GPU acceleration targets the octree assembly and the sparse linear solve.

### 9.1 Oriented Point Splatting into Octree

Insert each point's normal contribution into the octree's vector field grid. The indicator function gradient ∇χ is approximated by the point normals smoothed by a B-spline basis:

```glsl
// poisson_splat.comp — splat point normals into adaptive octree cells
layout(set=0,binding=0) readonly buffer Points { vec3 pts[]; vec3 nrm[]; };
layout(set=0,binding=1) buffer OctreeVec { vec3 vecField[]; };   // accumulated normal field
layout(set=0,binding=2) readonly buffer OctreeGrid { uint cellAtDepth[]; };
layout(push_constant) uniform PC { float cellSz; vec3 gridMin; int maxDepth; uint N; } pc;
layout(local_size_x=64) in;
void main() {
    uint i   = gl_GlobalInvocationID.x;
    if (i >= pc.N) return;
    // Find octree leaf cell containing pts[i], splat B-spline weight × normal
    ivec3 c  = ivec3((pts[i]-pc.gridMin)/pc.cellSz);
    uint  ci = octreeCellIdx(c, pc.maxDepth);
    float w  = bsplineWeight(pts[i], pc.gridMin + vec3(c)*pc.cellSz, pc.cellSz);
    atomicAdd_vec3(vecField[ci], w * nrm[i]);
}
```

[Source: Kazhdan et al. "Poisson Surface Reconstruction", SGP 2006; Kazhdan & Hoppe "Screened Poisson Surface Reconstruction", ACM TOG 2013]

### 9.2 Octree Laplacian Assembly

Assemble the sparse Laplacian for the octree by summing stencil contributions from each cell's 6-face neighbours. GPU parallelises over octree cells:

```glsl
// poisson_laplacian.comp — assemble Laplacian stencil for Poisson solve
layout(set=0,binding=0) readonly buffer OctreeCells { OctCell cells[]; };
layout(set=0,binding=1) buffer LaplacianRows { SparseRow rows[]; };
layout(local_size_x=64) in;
void main() {
    uint c  = gl_GlobalInvocationID.x;
    float diag = 0.0; uint nNbr = 0;
    for (int f=0; f<6; f++) {
        uint nb = cells[c].neighbours[f];
        if (nb == INVALID) continue;
        float w  = faceWeight(cells[c], cells[nb]);  // B-spline overlap integral
        rows[c].off[nNbr] = nb; rows[c].w[nNbr] = -w; nNbr++;
        diag += w;
    }
    rows[c].diag = diag; rows[c].n = nNbr;
}
// Solve: L·χ = div(vecField) via Gauss-Seidel or conjugate gradient (§37.1 pattern)
```

### 9.3 Adaptive Octree Refinement

Refine the octree where point density is high (more points → smaller cells → more detail). GPU atomic counting of points per cell at each level:

```glsl
// poisson_refine.comp — count points per octree cell, flag for subdivision
layout(set=0,binding=0) readonly buffer Points { vec3 pts[]; };
layout(set=0,binding=1) buffer CellCount { uint cnt[]; };
layout(set=0,binding=2) buffer SubdivFlag { uint flag[]; };
layout(push_constant) uniform PC { float cellSz; vec3 gridMin; int level; uint N; uint thresh; } pc;
layout(local_size_x=64) in;
void main() {
    uint i   = gl_GlobalInvocationID.x;
    if (i >= pc.N) return;
    ivec3 c  = ivec3((pts[i]-pc.gridMin)/(pc.cellSz * float(1<<(MAX_DEPTH-pc.level))));
    uint  ci = mortonEncode(c);
    uint  prev = atomicAdd(cnt[ci], 1u);
    if (prev == pc.thresh) atomicOr(flag[ci/32], 1u<<(ci%32));
}
// CPU: compact flagged cells, allocate children, re-insert points into next level
```

[Source: Kazhdan et al. 2006; GPU Poisson: Kazhdan & Hoppe "Screened Poisson Surface Reconstruction", 2013 §5; implementation: https://github.com/mkazhdan/PoissonRecon]

### 9.4 Isovalue Selection and Mesh Extraction

After solving for χ, choose the isovalue as the mean χ value at the input point positions, then run §65.1 marching cubes. The screened variant (2013) adds point-interpolation constraints that automatically set the isovalue:

```glsl
// poisson_isovalue.comp — compute average χ at input point positions
layout(set=0,binding=0) readonly buffer Points { vec3 pts[]; };
layout(set=0,binding=1) readonly buffer Chi { float chi[]; };
layout(set=0,binding=2) readonly buffer OctGrid { uint cellIdx[]; };
layout(set=0,binding=3) buffer IsoSum { float sum; uint cnt; };
layout(local_size_x=64) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    float v = trilinearSampleChi(chi, OctGrid, pts[i]);
    atomicAdd(IsoSum.sum, v);  // Note: use atomicAdd on float via floatBitsToUint trick
    atomicAdd(IsoSum.cnt, 1u);
}
// isovalue = IsoSum.sum / IsoSum.cnt
// Then: §65.1 marching cubes on the chi volume at this isovalue → watertight mesh
```

[Source: Kazhdan & Hoppe 2013; Fuhrmann & Goesele "Floating Scale Surface Reconstruction", ACM TOG 2014]

---


---

## II. Mesh Processing and Topology

Covers algorithms that operate on triangle or polygon meshes to change their connectivity, complexity, or parameterization: simplification and remeshing for runtime LOD, UV parameterization and atlas packing for texture baking, Delaunay triangulation for quality-guaranteed mesh generation, voxelization for collision and SDF computation, mesh fairing and smoothing for noise removal, segmentation for part decomposition, repair for watertightness, and streaming-friendly compression codecs (Draco, meshopt). These algorithms typically run as offline preprocess steps or as compute dispatches that feed data into the real-time rendering pipeline.

### 10. GPU Mesh Simplification and LOD

GPU subdivision (§1) adds geometric detail; the inverse — removing detail for distant objects — is equally critical for real-time budgets. Two complementary approaches exist: quadric-error-metric (QEM) edge collapse for quality-preserving offline LOD generation, and vertex clustering for GPU-parallel online simplification.

### 10.1 Quadric Error Metric (Garland-Heckbert)

The QEM (Garland & Heckbert, 1997) assigns each vertex v a 4×4 error matrix Q = ΣᵢKᵢ where Kᵢ = ppᵀ for each adjacent face plane p = [nₓ, nᵧ, nᵤ, d]ᵀ (plane equation as a 4-vector). The cost of collapsing edge (v₁, v₂) to optimal position v̄:

```
Q_combined = Q₁ + Q₂
v̄ = argmin v̄ᵀ Q_combined v̄   (solve the 3×3 linear subsystem)
cost = v̄ᵀ Q_combined v̄
```

Collapse edges in order of cost using a min-heap; after each collapse, update the affected neighbor quadrics. The algorithm is inherently sequential due to topological dependencies between collapses.

### 10.2 meshoptimizer

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

For meshlet-based rendering (§10.4), meshoptimizer's `meshopt_buildMeshlets` already incorporates vertex cache awareness at the meshlet granularity: each meshlet's local index buffer is a tight 64-vertex / 124-triangle unit that fits entirely within the mesh shader's per-workgroup registers, making PTVC optimization less critical at draw time but more important within the meshlet build step.

**GPU porting status — simplification.** `meshopt_simplify` (QEM edge collapse) is CPU-only. GPU simplification is an active area: Unreal Engine 5's Nanite performs cluster-level hierarchical simplification on the GPU as part of its virtual geometry system, but this is not a direct port of QEM — it uses a custom cluster-DAG structure built offline. There is no production GPU port of the meshoptimizer simplification API. For real-time LOD generation (e.g., runtime terrain or streaming), GPU vertex clustering (§10.3) is the practical GPU alternative at the cost of lower quality than QEM.

**GPU porting status — encode/decode.** `meshopt_decodeIndexBuffer` and `meshopt_decodeVertexBuffer` are explicitly designed to be portable to GPU compute. The decode algorithms are tight loops with no irregular branching, and Kapoulkine has noted that a GLSL/HLSL compute shader decode pass is a stated roadmap goal. The encode step (delta coding, bit packing) remains CPU-only. A GPU decode pass would enable streaming compressed geometry directly from a staging buffer into a vertex/index buffer in a single compute dispatch, eliminating CPU decompression latency.

### 10.3 Vertex Clustering on GPU

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

### 10.4 LOD Selection in Mesh Shaders

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

### 10.5 Smooth Normals Post-Pass

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

### 10.6 Procedural GPU Instancing: Grass, Hair, Fur

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

**LOD.** The task shader can switch to a billboard card (2 triangles per blade) beyond ~50m and suppress blades entirely past ~150m. For hair, replace the 5-vertex blade with a 10–16 vertex strand following a Catmull-Rom spline through simulation control points output by a PBD pass (§39.6). Fur uses a shorter, denser strand with a random length distribution driven by a density-map alpha channel.

[Source: Acton (2021) "Procedural Grass in 'Ghost of Tsushima'" GDC; Epic Games "UE5 Nanite Foliage" tech blog]

### 10.7 Billboard and Impostor LOD Atlases

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

### 10.8 Bent Normals Precomputation

A **bent normal** at a surface point is the average unoccluded hemisphere direction — pointing toward the open sky rather than toward nearby occluders. When used to look up an irradiance probe or environment map, bent normals dramatically reduce light leaking under overhangs, inside concave surfaces, and near contact shadows, at zero fragment-shader cost beyond the texture lookup.

**Baking pass (`bent_normal_bake.comp`):** For each surface vertex, cast N rays in a cosine-weighted hemisphere and accumulate the directions of unoccluded rays. The BVH built in §24 provides the ray-cast primitive:

```glsl
// bent_normal_bake.comp — one thread per surface vertex
layout(local_size_x=64) in;

layout(set=0, binding=0) readonly  buffer Positions { vec4 positions[]; };
layout(set=0, binding=1) readonly  buffer Normals   { vec4 normals[];   };
layout(set=0, binding=2) readonly  buffer BVH       { /* LBVH nodes from §24 */ };
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

### 10.9 Geometry Compression and Quantization

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

### 10.10 Mesh Stitching and Seam Welding

DCC tools and glTF exporters produce meshes with split vertices at UV seams, hard-normal boundaries, and part joints — positions that are geometrically coincident but stored as separate vertices with different attributes. This produces visual cracks in subdivision (§1), incorrect smooth normals (§10.5), and doubles the vertex count along every seam. GPU-side stitching merges these before any geometry processing.

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

### 10.11 Mesh Boolean Operations: Polygon Clipping on GPU

SDF Boolean operations (§3.7) approximate the result via a distance field. **Polygon clipping** produces exact Boolean operations on triangle meshes by clipping each triangle of mesh A against each triangle of mesh B at their intersection contour, then classifying and keeping the desired inside/outside fragments.

**Sutherland-Hodgman polygon clipping on GPU.** For each candidate pair of triangles (A_i, B_j) from the BVH broad-phase (§24.1–8.3), clip triangle A_i against the plane of B_j:

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

1. **BVH broad-phase** — find all (A_i, B_j) AABB pairs that overlap (§24.1–8.3)
2. **Triangle-triangle intersection** (compute) — for each pair, compute the intersection line segment; discard non-intersecting pairs
3. **Sutherland-Hodgman clipping** (compute) — clip each intersecting triangle against the other's plane; emit sub-triangles into output buffer
4. **Inside/outside classification** — ray-cast from each sub-triangle centroid to classify interior/exterior using the opposite mesh's BVH
5. **Triangle soup assembly** — merge kept sub-triangles from both meshes into output index buffer

Full mesh Boolean on GPU is an active research area. Production tools (Blender's `bmesh`, OpenCASCADE `BRepAlgoAPI`) run on CPU, but GPU broad-phase acceleration can reduce step 1–2 from minutes to seconds for complex meshes. The GPU pipeline above handles the intersection kernel; CPU coordinates the global classification. [Source: Greiner & Hormann "Efficient Clipping of Arbitrary Polygons", TOG 1998; Bernstein & Fussell "Fast, Exact, Linear Booleans", SGP 2009]

### 10.12 Harmonic and LSCM UV Parameterization

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

**Practical output.** For a 100k-triangle mesh, GPU Laplacian assembly takes ~5 ms; CG convergence (50–200 iterations) takes ~10–50 ms. The result feeds directly into the baking pipeline (§10.13). Production tools: xatlas (MIT, CPU-only) for automatic atlas layout with seam optimization; libigl `igl::lscm` + `igl::harmonic` (CPU); uvatlas (DirectX SDK, CPU). [Source: Lévy et al. "Least Squares Conformal Maps for Automatic Texture Atlas Generation", SIGGRAPH 2002; Desbrun et al. "Intrinsic Parameterizations of Surface Meshes", Eurographics 2002]

### 10.13 GPU Parametric Texture Baking

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
layout(set=0,binding=4) readonly buffer HiPolyBVH  { /* LBVH §24 */ };
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
- **AO / bent normals** — §10.8 bent normal bake using BVH ray casting
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

### 10.14 GPU Mesh Repair

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

### 10.15 Sphere Packing and Farthest-Point Sampling

**Farthest-point sampling (FPS)** selects a subset of N points from a larger set M such that each selected point is as far as possible from all previously selected points. It produces a well-distributed, coverage-maximizing sample set — ideal for: light probe placement (maximize coverage of the scene), LOD seed selection (§10.1 QEM clustering), impostor view-direction sampling (§10.7), and point-cloud downsampling before Poisson reconstruction (§3.12).

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

### 11. GPU Remeshing

*Audience: graphics application developers, systems developers.*

Remeshing reshapes mesh geometry — changing edge lengths, valences, and alignment — without changing the underlying surface. It sits between simplification (§10, which reduces vertex count) and repair (§10.14, which fixes pathological topology).

### 11.1 Isotropic Remeshing

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

### 11.2 Anisotropic Curvature-Aligned Remeshing

Aligns quad/triangle edges to principal curvature directions (§34.4). Pipeline:

1. Compute principal curvature directions per vertex (§34.4 GPU kernel).
2. Smooth a cross-field (4-RoSy) by solving ∇²θ=0 on the mesh (sparse CG, same Laplacian as §34.1).
3. Trace streamlines along cross-field directions via RK4 on-surface integration.
4. Extract quad mesh from streamline network intersections (CPU topology extraction).

Steps 1–3 run on GPU; step 4 is CPU-side. [Source: Bommes et al. "Mixed-Integer Quadrangulation", SIGGRAPH 2009]

### 11.3 Incremental Delaunay Refinement for FEM Meshes

Generating volumetric tet meshes with quality guarantees (minimum dihedral angle > θ) uses Delaunay refinement (Shewchuk 1998): identify poor-quality tets (circumradius/shortest-edge > threshold), insert circumcenter, re-triangulate the star cavity. GPU parallelism: independently bad tets whose circumcenters do not spatially conflict process simultaneously; the §27.2 grid detects conflicts. [Source: Shewchuk "Tetrahedral Mesh Generation by Delaunay Refinement", SoCG 1998]

---

### 12. UV Seam Optimization and Atlas Packing

*Audience: graphics application developers.*

§10.12 covered LSCM and harmonic UV parameterization algorithms. This section covers the upstream problem (where to cut seams to minimise distortion) and the downstream problem (how to pack resulting UV charts into an atlas with minimal wasted space).

### 12.1 Seam Placement via Graph Cuts

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

### 12.2 SLIM: Scalable Locally Injective Mappings on GPU

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

The global step is a sparse linear system (same Laplacian structure as §34.1) solved by conjugate gradient on CPU or GPU sparse CG (§48.2). [Source: Rabinovich et al. "SLIM: Scalable Locally Injective Mappings", SIGGRAPH Asia 2017]

### 12.3 GPU Atlas Bin-Packing: Skyline Algorithm

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

### 13. Mesh Voxelisation

*Audience: systems developers, graphics application developers.*

Mesh voxelisation converts a triangle soup into a 3D voxel grid, producing either a *surface shell* (conservative voxelisation) or a filled *solid* (solid voxelisation). The output is consumed by §61 LSM initialisation, §52 fracture setup, §28 AO baking, physics collision, and GPU fabrication slicing.

### 13.1 Conservative Voxelisation

Conservative voxelisation marks every voxel whose cell overlaps the triangle surface. Schwarz & Seidel (2010) test each triangle against candidate voxels in its AABB:

```glsl
// conserv_vox.comp — one triangle per workgroup, scatter to 3D bitfield
layout(set=0,binding=0) readonly buffer Tris { vec3 triVerts[]; };
layout(set=0,binding=1) buffer Voxels { uint voxels[]; };  // 1 bit per voxel
layout(push_constant) uniform PC { vec3 gridMin; float dx; ivec3 gridDim; } pc;
layout(local_size_x=1) in;
void main() {
    uint t  = gl_GlobalInvocationID.x;
    vec3 v0 = triVerts[t*3], v1 = triVerts[t*3+1], v2 = triVerts[t*3+2];
    vec3 bMin = min(v0, min(v1,v2));
    vec3 bMax = max(v0, max(v1,v2));
    ivec3 cMin = max(ivec3(0), ivec3(floor((bMin - pc.gridMin)/pc.dx)));
    ivec3 cMax = min(pc.gridDim-1, ivec3(ceil((bMax - pc.gridMin)/pc.dx)));
    for (int z=cMin.z; z<=cMax.z; z++)
    for (int y=cMin.y; y<=cMax.y; y++)
    for (int x=cMin.x; x<=cMax.x; x++) {
        vec3 cen = pc.gridMin + (vec3(x,y,z)+0.5)*pc.dx;
        if (triBoxOverlap(cen, pc.dx*0.5, v0,v1,v2)) {
            uint flat = z*pc.gridDim.y*pc.gridDim.x + y*pc.gridDim.x + x;
            atomicOr(voxels[flat/32], 1u << (flat%32));
        }
    }
}
```

`triBoxOverlap` uses the separating axis theorem (SAT) with 13 axes: 3 face normals, 3 box normals, 9 edge cross products. [Source: Schwarz & Seidel "Fast Parallel Surface and Solid Voxelization", SIGGRAPH Asia 2010]

### 13.2 Solid Voxelisation via Parity Ray Casting

Fill the interior by counting triangle crossings along one axis per column. Sort triangles by Z-slab; for each (x,y) column cast a ray in Z and toggle fill at each crossing:

```glsl
// solid_vox.comp — parity fill per XY column
layout(set=0,binding=0) readonly buffer SortedZ { TriZ triBySlab[]; };  // triangles sorted by z min
layout(set=0,binding=1) buffer Voxels { uint voxels[]; };
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 xy  = ivec2(gl_GlobalInvocationID.xy);
    vec2  pos = gridMin.xy + (vec2(xy)+0.5)*dx;
    bool  inside = false;
    for (uint t=0; t<N_TRIS; t++) {
        float zt;
        if (rayTriZ(pos, triBySlab[t], zt)) {
            int z = clamp(int((zt - gridMin.z)/dx), 0, gridDim.z-1);
            uint flat = z*gridDim.y*gridDim.x + xy.y*gridDim.x + xy.x;
            atomicXor(voxels[flat/32], 1u << (flat%32));
        }
    }
}
// Post-pass: prefix OR along Z to fill interior between crossings
```

[Source: Schwarz & Seidel 2010; Pantaleoni "VoxelPipe: A Programmable Pipeline for 3D Voxelization", HPG 2011]

### 13.3 Output Formats

| Format | Vulkan type | Use |
|---|---|---|
| Bitfield SSBO (1 bit/voxel) | `VkBuffer` | Minimal memory; used for Boolean ops, parity fill |
| R8 3D image (0/255) | `VkImage` (3D) | Accessible as `sampler3D`; used for ray marching, §61 LSM |
| R32F SDF 3D image | `VkImage` (3D) | Distance field; used for §3 CSG, §61 redistancing init |
| SSBO of voxel centroids | `VkBuffer` | Sparse point cloud; used for §87 ICP, §28 AO |

Convert bitfield to SDF by running GPU JFA (§3.4) or FMM (§61.3) on the surface shell voxels. [Source: Jones et al. "3D Distance Fields: A Survey", TVCG 2006]

---

### 14. Morphological Operations and Medial Axis

*Audience: systems developers, graphics application developers.*

Morphological operations (erosion, dilation, opening, closing) on 3D binary voxel grids are foundational in medical image processing, GPU fabrication, and shape analysis. §62.3 showed SDF offset (equivalent to dilation of the zero level set); this section covers full morphological operators on voxel bitfields and GPU medial axis extraction.

### 14.1 Binary Dilation and Erosion

Dilation: a voxel is set if any voxel in its structuring element neighbourhood is set. Erosion: a voxel is set only if *all* neighbours are set. For a 3×3×3 ball structuring element:

```glsl
// morph_dilate.comp
layout(set=0,binding=0) readonly buffer VoxIn  { uint vin[];  };
layout(set=0,binding=1) writeonly buffer VoxOut { uint vout[]; };
layout(push_constant) uniform PC { ivec3 gridDim; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 v  = ivec3(gl_GlobalInvocationID);
    uint  out = 0u;
    for (int dz=-1;dz<=1;dz++) for(int dy=-1;dy<=1;dy++) for(int dx=-1;dx<=1;dx++) {
        ivec3 nb = clamp(v+ivec3(dx,dy,dz), ivec3(0), pc.gridDim-1);
        uint  flat = nb.z*pc.gridDim.y*pc.gridDim.x + nb.y*pc.gridDim.x + nb.x;
        out |= (vin[flat/32] >> (flat%32)) & 1u;
    }
    uint flat = v.z*pc.gridDim.y*pc.gridDim.x + v.y*pc.gridDim.x + v.x;
    // atomic OR into output (one bit per voxel)
    atomicOr(vout[flat/32], out << (flat%32));
}
// Erosion: change |= to &= (all neighbours set → set output)
```

Opening (erosion then dilation) removes small protrusions; closing (dilation then erosion) fills small holes. [Source: Serra "Image Analysis and Mathematical Morphology", 1982; GPU morphology: Ritter et al. "GPU-based Morphological Reconstruction", TVCG 2008]

### 14.2 Parallel 3D Thinning for Medial Axis

The medial axis (topological skeleton) is extracted by iteratively removing simple surface voxels — voxels on the surface whose removal does not change the topology. GPU parallel thinning alternates six directional sub-iterations (±X, ±Y, ±Z):

```glsl
// thin_pass.comp — one directional sub-iteration
layout(set=0,binding=0) buffer Voxels { uint vox[]; };
layout(set=0,binding=1) buffer Changed{ uint changed; };
layout(push_constant) uniform PC { ivec3 gridDim; int direction; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;

bool isSurfaceVoxel(ivec3 v, int dir);    // has a background neighbour in 'dir' face
bool isSimpleVoxel(ivec3 v);              // Euler number preserving test (26-connectivity LUT)
bool isEndVoxel(ivec3 v);                 // degree-1 skeleton endpoint

void main() {
    ivec3 v    = ivec3(gl_GlobalInvocationID);
    uint  flat = v.z*pc.gridDim.y*pc.gridDim.x + v.y*pc.gridDim.x + v.x;
    bool  set  = (vox[flat/32] >> (flat%32)) & 1u;
    if (!set) return;
    if (isSurfaceVoxel(v, pc.direction) && isSimpleVoxel(v) && !isEndVoxel(v)) {
        atomicAnd(vox[flat/32], ~(1u << (flat%32)));
        atomicOr(changed, 1u);
    }
}
```

Iterate until `changed==0`. The result is the curve skeleton of the solid object — used for rigging assistance, vessel centreline extraction, and shape-matching. [Source: Palágyi & Kuba "A Parallel 3D 12-Subiteration Thinning Algorithm", Graphical Models 1999; Lee et al. "Building Skeleton Models via 3-D Medial Surface Axis Thinning", CVGIP 1994]

### 14.3 Distance-Field Medial Axis

A faster (but approximate) medial axis: extract ridges of the distance transform where ∇²D is locally maximally negative (D is the Euclidean distance field). Compute via GPU FMM (§61.3) then threshold on Laplacian:

```glsl
// medial_laplacian.comp
layout(set=0,binding=0) readonly buffer SDF { float sdf[]; };
layout(set=0,binding=1) buffer Medial { uint med[]; };
layout(push_constant) uniform PC { ivec3 dim; float threshold; } pc;
layout(local_size_x=64) in;
void main() {
    ivec3 v = ivec3(gl_GlobalInvocationID);
    float lap = -6.0*sdf[flat(v)]
               + sdf[flat(v+ivec3(1,0,0))] + sdf[flat(v-ivec3(1,0,0))]
               + sdf[flat(v+ivec3(0,1,0))] + sdf[flat(v-ivec3(0,1,0))]
               + sdf[flat(v+ivec3(0,0,1))] + sdf[flat(v-ivec3(0,0,1))];
    if (sdf[flat(v)] > 0.0 && lap < pc.threshold)
        atomicOr(med[flat(v)/32], 1u << (flat(v)%32));
}
```

[Source: Siddiqi & Pizer "Medial Representations", Springer 2008]

---

### 15. Progressive Meshes and Geometry Streaming

*Audience: graphics application developers, systems developers.*

Static LOD (§10) simplifies meshes offline and selects one resolution at runtime. Progressive meshes (Hoppe 1996) stream detail *incrementally* — the mesh grows from coarse to fine as data arrives or the camera approaches, without discrete LOD pops. GPU-side vertex split playback and geomorphic interpolation are compute shader tasks.

### 15.1 Vertex Split Records

A progressive mesh encodes a sequence of vertex splits (the inverse of edge collapses). Each record stores:

```c
struct VertexSplit {
    uint  vs;           // vertex to split
    uint  vl, vr;       // left and right flap vertices (may be INVALID)
    uint  fn[2];        // two new face indices
    vec3  pos_s;        // position of vs after split
    vec3  pos_new;      // position of the new vertex
};
```

Offline: run QEM (§10.1) edge collapse sequence, record the inverse. Upload the full split sequence as an SSBO. At runtime, replay splits in reverse order to refine.

### 15.2 GPU Vertex Split Playback

Each frame, identify how many splits to apply (based on view distance or network arrival). Apply splits in a compute dispatch:

```glsl
// vsplit_apply.comp — apply one batch of vertex splits
layout(set=0,binding=0) readonly buffer Splits  { VertexSplit splits[]; };
layout(set=0,binding=1) buffer Positions { vec3 pos[]; };
layout(set=0,binding=2) buffer Indices   { uint idx[]; };
layout(set=0,binding=3) buffer ActiveFaces{ uint activeMask[]; };
layout(push_constant) uniform PC { uint splitBase; uint splitCount; } pc;
layout(local_size_x=64) in;
void main() {
    uint s  = pc.splitBase + gl_GlobalInvocationID.x;
    if (s >= pc.splitBase + pc.splitCount) return;
    VertexSplit vs = splits[s];
    // Allocate new vertex at vs.vsnew (pre-assigned offline)
    pos[vs.vsnew]  = vs.pos_new;
    pos[vs.vs]     = vs.pos_s;
    // Activate new faces
    atomicOr(activeMask[vs.fn[0]/32], 1u << (vs.fn[0]%32));
    atomicOr(activeMask[vs.fn[1]/32], 1u << (vs.fn[1]%32));
    // Update face index buffer (pre-written offline, just mark active)
}
```

[Source: Hoppe "Progressive Meshes", SIGGRAPH 1996; Hoppe "Efficient Implementation of Progressive Meshes", CGF 1998]

### 15.3 Geomorphic Position Interpolation

When a split is applied, the new vertex position interpolates between the collapsed position (vs.pos_s) and the split position (vs.pos_new) to prevent popping:

```glsl
// geomorph.vert — blend vertex positions during split transition
layout(set=0,binding=0) readonly buffer GeoMorphT { float t[]; };  // [0,1] per vertex, fades in
layout(set=0,binding=1) readonly buffer SplitPos  { vec3 splitP[]; };
layout(set=0,binding=2) readonly buffer BasePos   { vec3 baseP[];  };
layout(location=0) in uint vertexID;
void main() {
    vec3 p      = mix(baseP[vertexID], splitP[vertexID], t[vertexID]);
    gl_Position = vp * vec4(p, 1.0);
}
```

t[v] increases from 0 to 1 over N frames after the split is applied, smoothing the transition. [Source: Hoppe 1996; Losasso et al. "Smooth Geometry Images", SGP 2003]

### 15.4 View-Dependent Refinement

Refine faces visible to the camera and simplify back-facing or distant faces each frame. Compute per-face refinement priority in a GPU pass:

```glsl
// vsplit_priority.comp — score each possible split by visual importance
layout(local_size_x=64) in;
void main() {
    uint s    = gl_GlobalInvocationID.x;
    VertexSplit vs = splits[s];
    vec3  mid = (pos[vs.vs] + pos[vs.vl]) * 0.5;
    float dist = length(mid - cameraPos);
    float err  = length(vs.pos_new - vs.pos_s);   // geometric error of collapse
    float screenErr = err * projScale / max(dist, 0.01);
    priority[s] = screenErr;
}
// Sort by priority (GPU radix sort), apply top-K splits this frame
```

[Source: Hoppe "View-Dependent Refinement of Progressive Meshes", SIGGRAPH 1997]

---

### 16. Mesh Compression and Streaming Codecs

*Audience: systems developers, graphics application developers.*

GPU geometry reaches the device as raw vertex and index buffers — but the upstream pipeline that produces those buffers involves lossy and lossless compression codecs, streaming decode, and quantization. This section covers the GPU-accelerable parts of that pipeline.

### 16.1 Geometry Quantization

Vertex positions, normals, and UV coordinates are quantized before transmission. GPU decode (dequantisation) runs in a compute pre-pass before skinning:

```glsl
// dequant.comp — unpack quantized vertex stream to float SSBO
layout(set=0,binding=0) readonly buffer QuantVerts { uint packed[]; };  // 16-bit snorm x,y,z,w
layout(set=0,binding=1) writeonly buffer FloatVerts { vec4 verts[]; };
layout(push_constant) uniform PC { vec3 posMin; vec3 posScale; } pc;
layout(local_size_x=64) in;
void main() {
    uint i   = gl_GlobalInvocationID.x;
    uint p0  = packed[i*2+0];  // x,y packed in one uint
    uint p1  = packed[i*2+1];  // z,w packed in one uint
    float x  = unpackSnorm2x16(p0).x * pc.posScale.x + pc.posMin.x;
    float y  = unpackSnorm2x16(p0).y * pc.posScale.y + pc.posMin.y;
    float z  = unpackSnorm2x16(p1).x * pc.posScale.z + pc.posMin.z;
    verts[i] = vec4(x, y, z, 1.0);
}
```

meshoptimizer (§42 Table A) provides `meshopt_quantizePosition()` and `meshopt_decodeVertexBuffer()` — decode on CPU then upload, or port the bit-unpacking to a compute shader for GPU-side decode. [Source: Zeux "meshoptimizer" https://github.com/zeux/meshoptimizer]

### 16.2 Draco: Connectivity and Position Compression

Draco (Google 2017) uses edgebreaker connectivity encoding (Rossignac 1999) and parallelogram prediction for positions. The full decoder runs on CPU; however, the final attribute prediction step is GPU-parallelisable because it is a per-vertex linear combination of neighbours:

```glsl
// draco_predict.comp — parallelogram prediction final pass (GPU)
layout(set=0,binding=0) readonly buffer Decoded    { int quantPos[];   };  // partially decoded
layout(set=0,binding=1) readonly buffer Prediction { ivec3 predCoords[]; };
layout(set=0,binding=2) writeonly buffer FinalPos  { vec3 pos[];        };
layout(push_constant) uniform PC { vec3 minQ; float quantStep; } pc;
layout(local_size_x=64) in;
void main() {
    uint v   = gl_GlobalInvocationID.x;
    ivec3 q  = ivec3(quantPos[v*3], quantPos[v*3+1], quantPos[v*3+2]);
    q       += predCoords[v];  // add parallelogram prediction
    pos[v]   = vec3(q) * pc.quantStep + pc.minQ;
}
```

[Source: Rossignac "Edgebreaker: Connectivity Compression for Triangle Meshes", TVCG 1999; Galligan et al. "Draco: A Method for Compressing and Decompressing 3D Geometric Meshes and Point Clouds", Google Research 2018; https://github.com/google/draco]

### 16.3 Basis Universal / ETC2 for Geometry Textures

Normal maps and displacement maps used in §54 (cloth), §98 (procedural textures), and §71 (terrain) are compressed with Basis Universal (transcoded to ETC2/BC7 at load time). GPU transcoding converts Basis's intermediate format to the target compressed format in compute:

```glsl
// basis_transcode.comp — ETC2 block encoding (simplified RGB mode)
layout(set=0,binding=0) readonly buffer BasisBlocks { BasisBlock src[]; };
layout(set=0,binding=1) writeonly buffer ETC2Blocks  { ETC2Block  dst[]; };
layout(local_size_x=64) in;
void main() {
    uint b = gl_GlobalInvocationID.x;
    ETC2Block out = encodeETC2(decodeBasisBlock(src[b]));
    dst[b] = out;
}
```

[Source: Binstock "Basis Universal GPU Texture Codec" https://github.com/BinomialLLC/basis_universal; Ström & Akenine-Möller "iPACKMAN: High-Quality, Low-Complexity Texture Compression for Mobile Phones", HWWS 2005]

### 16.4 Index Buffer Compression: Meshlet Delta Encoding

For GPU-driven rendering (§26), index buffers within meshlets (max 64 vertices) use delta encoding + VByte: store the first index as full 32-bit, then encode subsequent indices as signed 8-bit deltas. Decode in the task shader:

```glsl
// meshlet_decode.task.glsl
layout(local_size_x=32) in;
void main() {
    uint m       = gl_WorkGroupID.x;
    uint baseIdx = meshletOffsets[m];
    uint n       = meshletVertCount[m];
    // Decode delta-encoded vertex indices
    uint prev = meshletPackedIdx[baseIdx];
    localVerts[0] = prev;
    for (uint i=1; i<n; i++) {
        int delta = int(int8_t(meshletPackedIdx[baseIdx+i]));
        prev += uint(delta);
        localVerts[i] = prev;
    }
    EmitMeshTasksEXT(n/32+1, 1, 1);
}
```

[Source: Zeux "meshoptimizer"; Wihlidal "Optimizing the Graphics Pipeline with Compute", GDC 2016]

---

### 17. Mesh Parameterization and UV Unwrapping

*Audience: graphics application developers, systems developers.*

Mesh parameterization maps a 3D surface to 2D UV coordinates while minimising distortion. GPU-accelerated methods solve large sparse linear systems or run iterative local/global optimisation (ARAP, LSCM) per vertex in parallel, enabling parameterization of million-triangle meshes in seconds.

### 17.1 LSCM: Least-Squares Conformal Maps

LSCM (Lévy et al. 2002) minimises angle distortion (conformal energy) by solving a least-squares system with two pinned vertices. The normal equation (AᵀA)x = Aᵀb has the same cotangent Laplacian structure as the heat method (§37):

```glsl
// lscm_residual.comp — compute residual of LSCM normal equation for CG iteration
// LSCM energy: E = Σ_t |∂u/∂x - ∂v/∂y|² + |∂u/∂y + ∂v/∂x|²
// Gradient w.r.t. UV coords: sparse matrix-vector product
layout(set=0,binding=0) readonly buffer ConfLap { float L[]; uint col[]; uint row[]; };
layout(set=0,binding=1) readonly buffer UV { vec2 uv[]; };
layout(set=0,binding=2) writeonly buffer Residual { vec2 r[]; };
layout(local_size_x=64) in;
void main() {
    uint v  = gl_GlobalInvocationID.x;
    vec2 Lu = vec2(0.0);
    for (uint e=row[v]; e<row[v+1]; e++)
        Lu += L[e] * uv[col[e]];
    r[v] = Lu;  // CG: r = b - Ax, where b=0 for interior (system is Lx=0 + pin constraints)
}
```

[Source: Lévy et al. "Least Squares Conformal Maps for Automatic Texture Atlas Generation", SIGGRAPH 2002]

### 17.2 ARAP: As-Rigid-As-Possible Parameterization

ARAP (Liu et al. 2008) alternates between (1) a local step fitting per-triangle rotations to the current UV map, and (2) a global step solving a sparse linear system to update UV coordinates:

```glsl
// arap_local.comp — fit per-triangle rotation R_t to minimize ARAP energy
layout(set=0,binding=0) readonly buffer UV  { vec2 uv[]; };
layout(set=0,binding=1) readonly buffer Pos { vec3 pos[]; };
layout(set=0,binding=2) readonly buffer F   { uvec3 faces[]; };
layout(set=0,binding=3) writeonly buffer Rot { mat2 R[]; };
layout(local_size_x=64) in;
void main() {
    uint t    = gl_GlobalInvocationID.x;
    uvec3 vi  = faces[t];
    // S = sum of cotangent-weighted outer products of 3D edges and UV edges
    mat2  S   = mat2(0.0);
    for (int e=0; e<3; e++) {
        uint a = vi[e], b = vi[(e+1)%3];
        vec2 duv = uv[b]-uv[a];
        float cot = cotWeight(pos, vi, e);
        S += cot * outerProduct(duv, vec2(pos[b].xy-pos[a].xy));
    }
    // Closest rotation R = U*Vᵀ from SVD(S)
    mat2 U, V; vec2 sv;
    svd2x2(S, U, sv, V);
    R[t] = U * transpose(V);
}
// arap_global.comp: update UV via sparse Cholesky solve (pre-factored Laplacian)
```

[Source: Liu et al. "A Local/Global Approach to Mesh Parameterization", SGP 2008; Sorkine & Alexa "As-Rigid-As-Possible Surface Modeling", SGP 2007]

### 17.3 Seam Selection via Minimum Spanning Tree

Before parameterization, the mesh must be cut into a topological disk. GPU BFS finds the minimum spanning tree of the dual graph (face adjacency) weighted by edge distortion potential:

```glsl
// seam_select.comp — BFS on dual graph to find spanning tree (seam = non-tree edges)
layout(set=0,binding=0) readonly buffer DualAdj { uint adjFaces[]; uint adjStart[]; };
layout(set=0,binding=1) readonly buffer EdgeWeight { float w[]; };
layout(set=0,binding=2) buffer Visited { uint visited[]; };
layout(set=0,binding=3) buffer Queue   { uint queue[]; uint head; uint tail; };
layout(set=0,binding=4) buffer InTree  { uint inTree[]; };
layout(local_size_x=1) in;  // BFS serial front, parallel edge evaluation
void main() {
    while (head < tail) {
        uint f = queue[atomicAdd(head, 1u)];
        for (uint e=adjStart[f]; e<adjStart[f+1]; e++) {
            uint nb = adjFaces[e];
            if (atomicOr(visited[nb/32], 1u<<(nb%32)) == 0u) {
                inTree[e]    = 1u;
                queue[atomicAdd(tail, 1u)] = nb;
            }
        }
    }
    // Seam edges: those not in spanning tree → used as mesh cuts for parameterization
}
```

[Source: Gu et al. "Geometry Images", SIGGRAPH 2002; Sheffer et al. "ABF++: Fast and Robust Angle Based Flattening", ACM TOG 2005]

### 17.4 Atlas Packing

After parameterization, pack UV charts into a rectangular atlas minimising wasted space. GPU bin-packing using sorted-area decreasing + shelf algorithm:

```glsl
// atlas_pack.comp — place UV charts into atlas (parallel shelf-fit attempt)
layout(set=0,binding=0) readonly buffer Charts { UVChart charts[]; };  // pre-sorted by area
layout(set=0,binding=1) writeonly buffer Placements { uvec4 placements[]; };  // x,y,w,h
layout(set=0,binding=2) buffer ShelfY { uint shelfY[]; };  // current Y of each shelf
layout(local_size_x=1) in;
void main() {
    for (uint c=0; c<N_CHARTS; c++) {
        uint bestShelf = findBestShelf(charts[c].w, charts[c].h, shelfY);
        placements[c] = uvec4(shelfX(bestShelf), shelfY[bestShelf],
                              charts[c].w, charts[c].h);
        shelfY[bestShelf] += charts[c].h;
    }
}
```

[Source: Sander et al. "Texture Mapping Progressive Meshes", SIGGRAPH 2001; xatlas: https://github.com/jpcy/xatlas]

---

### 18. GPU Delaunay Triangulation

*Audience: systems developers, graphics application developers.*

Delaunay triangulation of a 2D or 3D point set produces a triangulation where no point lies inside the circumcircle (2D) or circumsphere (3D) of any triangle, maximising minimum angles. GPU algorithms use massive parallel point insertion with conflict graph management, or jump flooding for the dual Voronoi diagram.

### 18.1 Jump Flooding for Voronoi Diagrams

Jump flooding (Rong & Tan 2006) computes the Voronoi diagram of seed points in a 2D texture in O(log N) passes — each pass propagates closest-seed information at exponentially decreasing step sizes:

```glsl
// jfa.comp — one jump flooding pass
layout(set=0,binding=0) uniform sampler2D voroIn;   // R=seed_x, G=seed_y, B=seed_id (or -1)
layout(set=0,binding=1, rgba32f) uniform writeonly image2D voroOut;
layout(push_constant) uniform PC { int stepSize; int W; int H; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 p   = ivec2(gl_GlobalInvocationID.xy);
    vec4  best = texelFetch(voroIn, p, 0);
    float bestD = (best.z >= 0.0) ?
        length(vec2(p) - best.xy) : 1e30;
    for (int dy=-1; dy<=1; dy++) for (int dx=-1; dx<=1; dx++) {
        if (dx==0 && dy==0) continue;
        ivec2 nb  = p + ivec2(dx,dy)*pc.stepSize;
        if (any(lessThan(nb,ivec2(0)))||any(greaterThanEqual(nb,ivec2(pc.W,pc.H)))) continue;
        vec4  nbv = texelFetch(voroIn, nb, 0);
        if (nbv.z < 0.0) continue;
        float d   = length(vec2(p) - nbv.xy);
        if (d < bestD) { bestD=d; best=nbv; }
    }
    imageStore(voroOut, p, best);
}
// N passes with stepSize = W/2, W/4, ..., 1 → complete Voronoi diagram
```

[Source: Rong & Tan "Jump Flooding in GPU with Applications to Voronoi Diagram and Distance Transform", I3D 2006]

### 18.2 Incremental Delaunay Insertion on GPU

Massive parallel Delaunay: insert all points in random rounds, each round testing point-triangle conflicts and flipping non-Delaunay edges via cavity re-triangulation:

```glsl
// delaunay_conflict.comp — find all triangles whose circumcircle contains a new point
layout(set=0,binding=0) readonly buffer Points    { vec2 pts[]; };
layout(set=0,binding=1) readonly buffer Triangles { uvec3 tris[]; vec3 circ[]; };  // circumcircle centre + radius
layout(set=0,binding=2) buffer Conflict { uint conf[]; };  // bit mask per (point, triangle)
layout(local_size_x=64) in;
void main() {
    uint p  = gl_GlobalInvocationID.x / N_TRIS;
    uint t  = gl_GlobalInvocationID.x % N_TRIS;
    float d = length(pts[p] - circ[t].xy) - circ[t].z;
    if (d < 0.0) atomicOr(conf[p*N_TRIS_WORDS + t/32], 1u<<(t%32));
}
// Each point then claims its conflict polygon cavity and re-triangulates it (serial per point)
```

[Source: Rong et al. "GPU-Assisted Computation of Centroidal Voronoi Tessellation", IEEE TVCG 2011; Navarro et al. "A Survey on Parallel Computing and its Applications in Data-Parallel Problems Using GPU Architecture", SIAM 2014]

### 18.3 Bowyer-Watson on GPU via Atomic Cavity Locking

Bowyer-Watson inserts each point by finding the cavity (set of conflicting triangles), re-triangulating, and updating the mesh. GPU parallelism: multiple non-conflicting points are inserted simultaneously via atomic ownership of cavity triangles:

```glsl
// bowyer_lock.comp — claim cavity triangles for point p (compare-exchange)
layout(set=0,binding=0) buffer TriOwner { uint owner[]; };  // which point owns each triangle
layout(set=0,binding=1) readonly buffer Conflict { uint conf[]; };
layout(local_size_x=64) in;
void main() {
    uint p = gl_GlobalInvocationID.x;
    for (uint t=0; t<N_TRIS; t++) {
        if ((conf[p*N_TRIS_WORDS+t/32] >> (t%32)) & 1u) {
            // Try to claim this triangle for point p
            uint prev = atomicCompSwap(owner[t], UNCLAIMED, p);
            if (prev != UNCLAIMED && prev != p) {
                // Conflict: another point claims the same triangle — retry next round
                releaseOwnedTriangles(p);  return;
            }
        }
    }
    // All cavity triangles claimed: re-triangulate cavity for point p
}
```

[Source: Blandford et al. "Compact Representations of Simplicial Meshes in Two and Three Dimensions", IJCGA 2005; GPU Bowyer-Watson: Cao et al. "Parallel Delaunay Triangulation on GPU", CGA 2014]

### 18.4 3D Delaunay and Weighted Voronoi

3D Delaunay triangulation of point clouds (for surface reconstruction, simulation mesh generation) uses the same jump flooding approach extended to a 3D texture, or a GPU-accelerated CGAL Delaunay via CUDA:

```glsl
// jfa3d.comp — 3D jump flooding for weighted Voronoi (power diagram)
layout(set=0,binding=0) uniform sampler3D voroIn3D;  // RGBA32F: seed xyz + weight
layout(set=0,binding=1, rgba32f) uniform writeonly image3D voroOut3D;
layout(push_constant) uniform PC { int stepSize; } pc;
layout(local_size_x=4,local_size_y=4,local_size_z=4) in;
void main() {
    ivec3 p   = ivec3(gl_GlobalInvocationID);
    vec4  best = texelFetch(voroIn3D, p, 0);
    float bestPow = (best.w>=0) ? pow2dist(vec3(p),best.xyz) - best.w : 1e30;
    for (int dz=-1;dz<=1;dz++) for(int dy=-1;dy<=1;dy++) for(int dx=-1;dx<=1;dx++) {
        if (dx==0&&dy==0&&dz==0) continue;
        ivec3 nb = p+ivec3(dx,dy,dz)*pc.stepSize;
        if (any(lessThan(nb,ivec3(0)))||any(greaterThanEqual(nb,GRID_DIM))) continue;
        vec4  nbv = texelFetch(voroIn3D,nb,0);
        if (nbv.w<0) continue;
        float d   = pow2dist(vec3(p),nbv.xyz) - nbv.w;
        if (d<bestPow) { bestPow=d; best=nbv; }
    }
    imageStore(voroOut3D, p, best);
}
```

[Source: Rong et al. 2011; CGAL 3D Delaunay: https://doc.cgal.org/latest/Triangulation_3/]

---

### 19. Mesh Fairing and Geometric Smoothing

*Audience: systems developers, graphics application developers.*

Mesh fairing removes high-frequency noise from geometry while preserving the shape's global structure. It is a preprocessing step before parameterization (§17), isosurface output (§65), and physics mesh preparation. GPU implementations exploit the same cotangent Laplacian structure as the heat method (§37) but with different energy functionals.

### 19.1 Laplacian Smoothing

Explicit Laplacian smoothing moves each vertex toward its umbrella-weighted centroid. One iteration:

```glsl
// laplacian_smooth.comp — one step of explicit Laplacian smoothing
layout(set=0,binding=0) readonly buffer PosIn  { vec3 posIn[]; };
layout(set=0,binding=1) writeonly buffer PosOut { vec3 posOut[]; };
layout(set=0,binding=2) readonly buffer AdjV  { uint adjVerts[];  };  // 1-ring neighbours
layout(set=0,binding=3) readonly buffer AdjW  { float adjW[];     };  // cotangent weights
layout(set=0,binding=4) readonly buffer AdjStart { uint adjStart[]; };
layout(push_constant) uniform PC { float lambda; } pc;
layout(local_size_x=64) in;
void main() {
    uint v    = gl_GlobalInvocationID.x;
    vec3  L   = vec3(0.0); float wSum = 0.0;
    for (uint e=adjStart[v]; e<adjStart[v+1]; e++) {
        float w = adjW[e];
        L    += w * posIn[adjVerts[e]];
        wSum += w;
    }
    L = (wSum > 1e-8) ? L/wSum : posIn[v];
    posOut[v] = posIn[v] + pc.lambda * (L - posIn[v]);
}
```

Pure Laplacian smoothing shrinks meshes. Taubin smoothing (1995) alternates λ > 0 (smooth) and μ < 0 (anti-shrink, |μ| > |λ|) passes to preserve volume. [Source: Taubin "A Signal Processing Approach To Fair Surface Design", SIGGRAPH 1995]

### 19.2 Implicit Fairing via Sparse Linear Solve

Implicit fairing (Desbrun et al. 1999) solves (I − λLc)p' = p in one step, where Lc is the normalised cotangent Laplacian. This is unconditionally stable and equivalent to many explicit steps:

```glsl
// implicit_fair.comp — one CG iteration for (I - λLc)p' = p
layout(set=0,binding=0) readonly buffer Lc_val { float val[]; uint col[]; uint row[]; };
layout(set=0,binding=1) readonly buffer R { vec3 r[]; };  // CG residual
layout(set=0,binding=2) readonly buffer P { vec3 p[]; };  // CG search direction
layout(set=0,binding=3) writeonly buffer Ap { vec3 Ap_out[]; };
layout(push_constant) uniform PC { float lambda; } pc;
layout(local_size_x=64) in;
void main() {
    uint v   = gl_GlobalInvocationID.x;
    vec3 s   = p[v];                               // (I - λLc)·p = p - λLc·p
    for (uint e=row[v]; e<row[v+1]; e++)
        s -= pc.lambda * val[e] * P[col[e]];
    Ap_out[v] = s;
}
// Full CG loop in C++/Vulkan: run 30-100 iterations, upload new pos via vkCmdUpdateBuffer
```

[Source: Desbrun et al. "Implicit Fairing of Irregular Meshes Using Diffusion and Curvature Flow", SIGGRAPH 1999]

### 19.3 Mean Curvature Flow

Mean curvature flow (MCF) moves each vertex along the mean curvature normal — a geometry-driven smoothing that minimises surface area. The mean curvature vector at vertex v is H(v) = (1/2A)·Lc·pos:

```glsl
// mcf.comp — compute mean curvature normal vector at each vertex
layout(set=0,binding=0) readonly buffer Pos { vec3 pos[]; };
layout(set=0,binding=1) readonly buffer Lc  { float val[]; uint col[]; uint row[]; };
layout(set=0,binding=2) readonly buffer Area { float A[]; };  // mixed Voronoi area
layout(set=0,binding=3) writeonly buffer HN { vec3 hn[]; };
layout(local_size_x=64) in;
void main() {
    uint v   = gl_GlobalInvocationID.x;
    vec3 Lp  = vec3(0.0);
    for (uint e=row[v]; e<row[v+1]; e++)
        Lp += val[e] * pos[col[e]];
    hn[v] = Lp / (2.0 * max(A[v], 1e-8));  // mean curvature normal
}
// Time integrate: pos[v] += dt * hn[v]  (explicit) or solve (I - dt*Lc)·pos' = pos (implicit)
```

[Source: Meyer et al. "Discrete Differential-Geometry Operators for Triangulated 2-Manifolds", VisMath 2003; Desbrun et al. 1999]

### 19.4 Feature-Preserving Smoothing: Bilateral Mesh Filter

Bilateral filtering (Fleishman et al. 2003) preserves edges and corners by weighting neighbour positions by both spatial proximity and normal similarity — the mesh analogue of the §76 bilateral depth filter and §78.4 SVGF:

```glsl
// bilateral_mesh.comp — bilateral mesh smoothing
layout(set=0,binding=0) readonly buffer Pos  { vec3 pos[]; };
layout(set=0,binding=1) readonly buffer Norm { vec3 nor[]; };
layout(set=0,binding=2) writeonly buffer PosOut { vec3 posOut[]; };
layout(set=0,binding=3) readonly buffer AdjV { uint adjV[]; uint adjStart[]; };
layout(push_constant) uniform PC { float sigmaS; float sigmaN; } pc;
layout(local_size_x=64) in;
void main() {
    uint v  = gl_GlobalInvocationID.x;
    vec3 p  = pos[v], n = nor[v];
    vec3 sum = vec3(0.0); float wSum = 0.0;
    for (uint e=adjStart[v]; e<adjStart[v+1]; e++) {
        uint nb   = adjV[e];
        float wS  = exp(-dot(pos[nb]-p,pos[nb]-p)/(2*pc.sigmaS*pc.sigmaS));
        float wN  = exp(-(1.0-dot(n,nor[nb]))/(2*pc.sigmaN*pc.sigmaN));
        float w   = wS * wN;
        sum += w * pos[nb]; wSum += w;
    }
    posOut[v] = (wSum > 1e-8) ? sum/wSum : p;
}
```

[Source: Fleishman et al. "Bilateral Mesh Denoising", SIGGRAPH 2003; Jones et al. "Non-Iterative, Feature-Preserving Mesh Smoothing", SIGGRAPH 2003]

---

### 20. GPU-Accelerated Remeshing

*Audience: systems developers, graphics application developers.*

Remeshing adapts a mesh's triangle density and quality to a target: uniform edge length (isotropic remeshing), curvature-adaptive sizing (anisotropic), or structured quads (quad remeshing). GPU parallelism targets the per-edge operations (split, collapse, flip) executed in rounds of non-conflicting edges.

### 20.1 Edge Flip for Delaunay Quality

Given the cotangent Laplacian weights, flip edges that violate the local Delaunay condition (circumcircle test). Parallelise over independent edge sets (graph colouring):

```glsl
// remesh_flip.comp — Delaunay edge flip (one colour class per dispatch)
layout(set=0,binding=0) buffer HalfEdge { HalfEdgeDS he[]; };
layout(set=0,binding=1) readonly buffer Color { uint color[]; };
layout(push_constant) uniform PC { uint currentColor; } pc;
layout(local_size_x=64) in;
void main() {
    uint e = gl_GlobalInvocationID.x;
    if (color[e] != pc.currentColor) return;
    // Check Delaunay condition: sum of opposite angles < π
    float a0 = oppositeAngle(he, e, 0);
    float a1 = oppositeAngle(he, e, 1);
    if (a0 + a1 > 3.14159 + 1e-5) {
        flipEdge(he, e);  // atomic flip: swap diagonal of adjacent quad
    }
}
```

[Source: Botsch & Kobbelt "A Remeshing Approach to Multiresolution Modeling", SGP 2004]

### 20.2 Edge Split and Collapse

Split edges longer than 4/3 × target length; collapse edges shorter than 4/5 × target length. Each operation is applied to non-conflicting edge sets in parallel:

```glsl
// remesh_split.comp — split edges exceeding target length
layout(set=0,binding=0) buffer Pos      { vec3 pos[];  };
layout(set=0,binding=1) buffer HalfEdge { HalfEdgeDS he[]; };
layout(set=0,binding=2) buffer NewVerts { vec3 nv[];   uint nvCount; };
layout(set=0,binding=3) readonly buffer Color { uint color[]; };
layout(push_constant) uniform PC { float targetLen; uint currentColor; } pc;
layout(local_size_x=64) in;
void main() {
    uint e   = gl_GlobalInvocationID.x;
    if (color[e] != pc.currentColor) return;
    uint  vi = he[e].vert, vj = he[he[e].twin].vert;
    float len= length(pos[vi]-pos[vj]);
    if (len > pc.targetLen * 4.0/3.0) {
        uint slot = atomicAdd(nvCount, 1u);
        nv[slot]  = (pos[vi]+pos[vj])*0.5;
        splitEdge(he, e, slot);  // insert midpoint vertex, update topology
    }
}
// Similarly: collapse_edge for edges shorter than 4/5 * targetLen
```

[Source: Botsch & Kobbelt 2004; Dunyach et al. "Adaptive Remeshing for Real-Time Mesh Deformation", EG 2013]

### 20.3 Tangential Smoothing

After flips and splits, move each vertex to the centroid of its 1-ring *projected onto the local tangent plane* — preserving volume while improving triangle quality:

```glsl
// remesh_tangential.comp — tangential relaxation of vertex positions
layout(set=0,binding=0) readonly buffer Pos  { vec3 pos[];  };
layout(set=0,binding=1) readonly buffer Norm { vec3 nor[];  };
layout(set=0,binding=2) writeonly buffer PosOut { vec3 posOut[]; };
layout(set=0,binding=3) readonly buffer AdjV { uint adjV[]; uint adjStart[]; };
layout(local_size_x=64) in;
void main() {
    uint v   = gl_GlobalInvocationID.x;
    vec3 p   = pos[v], n = nor[v];
    vec3 c   = vec3(0.0);
    uint deg = adjStart[v+1]-adjStart[v];
    for (uint e=adjStart[v];e<adjStart[v+1];e++) c += pos[adjV[e]];
    c /= float(deg);
    // Project onto tangent plane: subtract normal component of (c - p)
    vec3 q   = c - dot(c-p,n)*n;
    posOut[v] = q;
}
```

[Source: Botsch & Kobbelt 2004; Lévy & Bonneel "Variational Anisotropic Surface Meshing with Voronoi Parallel Linear Enumeration", IMSH 2012]

### 20.4 Centroidal Voronoi Tessellation (CVT) Remeshing

CVT remeshing places N sample points on the surface such that each sample is the centroid of its Voronoi region (Lloyd's algorithm). GPU parallelism: for each sample, find the nearest surface point; for each surface triangle, find the nearest sample:

```glsl
// cvt_assign.comp — assign each surface vertex to nearest CVT sample
layout(set=0,binding=0) readonly buffer Pos     { vec3 pos[];  };
layout(set=0,binding=1) readonly buffer Samples { vec3 samp[]; };
layout(set=0,binding=2) writeonly buffer ClusterID { uint cid[]; };
layout(local_size_x=64) in;
void main() {
    uint v   = gl_GlobalInvocationID.x;
    float bestD=1e30; uint bestS=0u;
    for (uint s=0; s<N_SAMPLES; s++) {
        float d = length(pos[v]-samp[s]);
        if (d<bestD) { bestD=d; bestS=s; }
    }
    cid[v] = bestS;
}
// Second pass: for each sample, compute weighted centroid over its cluster → new sample position
// Repeat until convergence (10-50 iterations typical)
```

[Source: Du et al. "Centroidal Voronoi Tessellations: Applications and Algorithms", SIAM Review 1999; GPU CVT: Rong et al. 2011 §67.2]

---

### 21. GPU Mesh Segmentation and Part Decomposition

*Audience: graphics application developers, systems developers.*

Mesh segmentation partitions a mesh into semantically meaningful regions — limbs, torso, head — for applications including automated rigging, per-part LOD generation, texture atlasing (§17), and physics proxy generation (§50). GPU algorithms include geodesic k-means clustering, random walk segmentation, and the shape diameter function (SDF-based thickness measure).

### 21.1 Shape Diameter Function (SDF Thickness)

The shape diameter function (Shapira et al. 2008) at each vertex is the mean length of inward rays that hit the opposite surface — a measure of local thickness used to distinguish thin (arm) from thick (torso) regions:

```glsl
// sdf_thickness.comp — compute SDF thickness via inward ray bundle
layout(set=0,binding=0) readonly buffer Pos  { vec3 pos[];  };
layout(set=0,binding=1) readonly buffer Norm { vec3 nor[];  };
layout(set=0,binding=2) uniform accelerationStructureEXT tlas;
layout(set=0,binding=3) writeonly buffer Thickness { float sdf[]; };
layout(push_constant) uniform PC { int nRays; float maxLen; } pc;
layout(local_size_x=64) in;
void main() {
    uint v   = gl_GlobalInvocationID.x;
    vec3 p   = pos[v] - nor[v]*1e-3;  // offset inward
    float sum = 0.0; int hits = 0;
    for (int r=0; r<pc.nRays; r++) {
        vec3 d = -nor[v] + sampleCone(nor[v], 0.3, halton2D(r));
        d = normalize(d);
        rayQueryEXT rq;
        rayQueryInitializeEXT(rq, tlas, gl_RayFlagsNoneEXT, 0xFF, p, 0.0, d, pc.maxLen);
        rayQueryProceedEXT(rq);
        if (rayQueryGetIntersectionTypeEXT(rq,true) != gl_RayQueryCommittedIntersectionNoneEXT) {
            sum  += rayQueryGetIntersectionTEXT(rq,true);
            hits++;
        }
    }
    sdf[v] = (hits>0) ? sum/float(hits) : pc.maxLen;
}
```

[Source: Shapira et al. "Consistent Mesh Partitioning and Skeletonisation Using the Shape Diameter Function", The Visual Computer 2008]

### 21.2 Geodesic K-Means Clustering

Assign each vertex to the nearest cluster centre in geodesic distance; update centres to the Fréchet mean of their cluster. GPU heat method (§37) computes geodesic distances from each centre:

```glsl
// geodesic_kmeans_assign.comp — assign vertices to nearest geodesic cluster centre
layout(set=0,binding=0) readonly buffer GeoDist { float gd[K_CLUSTERS][N_VERTS]; };  // from §37
layout(set=0,binding=1) writeonly buffer ClusterID { uint cid[]; };
layout(set=0,binding=2) writeonly buffer ClusterDist { float cdist[]; };
layout(local_size_x=64) in;
void main() {
    uint v = gl_GlobalInvocationID.x;
    float bestD=1e30; uint bestC=0u;
    for (uint c=0; c<K_CLUSTERS; c++)
        if (gd[c][v] < bestD) { bestD=gd[c][v]; bestC=c; }
    cid[v]=bestC; cdist[v]=bestD;
}
// Update pass: for each cluster, find vertex minimising sum of geodesic distances (Fréchet mean)
// Repeat assignment+update until convergence (5-20 iterations)
```

[Source: Shlafman et al. "Metamorphosis of Polyhedral Surfaces Using Decomposition", EG 2002; Katz & Tal "Hierarchical Mesh Decomposition Using Fuzzy Clustering and Cuts", SIGGRAPH 2003]

### 21.3 Random Walk Segmentation

Random walk segmentation (Grady 2006) marks a subset of vertices as seeds (one per segment) and solves a linear system to find the probability that a random walk from each vertex first reaches each seed. The label with highest probability wins:

```glsl
// rw_segment.comp — one Jacobi iteration for random walk segmentation
// Solve L·x = b where L is graph Laplacian, x is probability of reaching seed s
layout(set=0,binding=0) readonly buffer L_val { float Lv[]; uint Lcol[]; uint Lrow[]; };
layout(set=0,binding=1) buffer X { float x[]; };  // probability field
layout(set=0,binding=2) readonly buffer Seeds { uint isSeed[]; float seedVal[]; };
layout(local_size_x=64) in;
void main() {
    uint v   = gl_GlobalInvocationID.x;
    if (isSeed[v]) { x[v] = seedVal[v]; return; }  // boundary condition
    float num = 0.0, denom = 0.0;
    for (uint e=Lrow[v]; e<Lrow[v+1]; e++) {
        float w = -Lv[e];  // off-diagonal weights are negative
        num   += w * x[Lcol[e]];
        denom += w;
    }
    x[v] = (denom>1e-8) ? num/denom : 0.0;
}
// Run for each seed label; assign vertex to label with highest probability
```

[Source: Grady "Random Walks for Image Segmentation", IEEE TPAMI 2006; applied to meshes: Lai et al. "Mesh Decomposition with Cross-Boundary Brushes", EG 2009]

### 21.4 Part Adjacency Graph and Skeleton Extraction

From the segmentation (§21.2–87.3), build a part adjacency graph and extract a curve skeleton via medial axis of the SDF thickness field (§21.1):

```glsl
// part_adjacency.comp — build part adjacency graph from segmentation
layout(set=0,binding=0) readonly buffer ClusterID { uint cid[]; };
layout(set=0,binding=1) readonly buffer HalfEdge  { HalfEdgeDS he[]; };
layout(set=0,binding=2) buffer AdjMatrix { uint adj[K_CLUSTERS][K_CLUSTERS]; };
layout(local_size_x=64) in;
void main() {
    uint e  = gl_GlobalInvocationID.x;
    uint ci = cid[he[e].face];
    uint cj = cid[he[he[e].twin].face];
    if (ci != cj) atomicOr(adj[ci][cj], 1u);  // mark adjacency
}
// From adjacency graph: build skeleton by contracting each part to its geodesic centroid
// Connect centroids following adjacency → curve skeleton usable for §44 rigging
```

[Source: Au et al. "Skeleton Extraction by Mesh Contraction", SIGGRAPH 2008; Tagliasacchi et al. "3D Skeletons: A State-of-the-Art Report", EG 2016]

---

### 22. GPU Mesh Repair and Waterproofing

*Audience: systems developers.*

Meshes from 3D scanning, CAD export, isosurface extraction (§65), or CSG operations (§7) frequently contain gaps, T-junctions, non-manifold edges, and inconsistent winding. GPU mesh repair identifies and fixes these defects, producing a watertight manifold mesh suitable for simulation (§50, §55), 3D printing slicing, and CSG (§7).

### 22.1 Non-Manifold Edge Detection

A non-manifold edge is shared by more than two faces, or a boundary edge connecting to a non-manifold vertex. Detect in parallel across all edges:

```glsl
// nonmanifold_detect.comp — count face valence per edge
layout(set=0,binding=0) readonly buffer EdgeFaces { uint ef[]; uint efStart[]; };
layout(set=0,binding=1) writeonly buffer ManifoldFlag { uint flag[]; };
layout(local_size_x=64) in;
void main() {
    uint e = gl_GlobalInvocationID.x;
    uint n = efStart[e+1] - efStart[e];  // face count per edge
    // Manifold: n==2 (interior) or n==1 (boundary). Non-manifold: n==0 or n>2.
    flag[e] = (n == 1u || n == 2u) ? 0u : 1u;
}
```

[Source: Botsch et al. "Polygon Mesh Processing", A K Peters 2010, §1.3]

### 22.2 Hole Detection via Boundary Loop Tracing

Boundary edges (valence 1) form closed loops. GPU BFS traces each loop to find its length and classify it as a small gap (2–4 edges), medium hole, or large hole:

```glsl
// hole_detect.comp — trace boundary loops, compute loop lengths
layout(set=0,binding=0) readonly buffer HalfEdge { HalfEdgeDS he[]; };
layout(set=0,binding=1) buffer BoundaryFlag { uint isBoundary[]; };
layout(set=0,binding=2) buffer HoleLoops { uint loopStart[]; uint loopLen[]; uint nLoops; };
layout(local_size_x=1) in;  // serial BFS; parallelism across separate loops
void main() {
    for (uint e=0; e<N_EDGES; e++) {
        if (!isBoundary[e] || visited[e]) continue;
        uint loop = atomicAdd(nLoops, 1u);
        loopStart[loop] = e;
        uint cur = e; uint len = 0u;
        do { visited[cur]=1u; cur=he[cur].next; len++; } while (cur!=e && len<MAX_LOOP);
        loopLen[loop] = len;
    }
}
```

### 22.3 Hole Filling via Minimum-Area Triangulation

Fill a boundary loop by triangulating the polygon using the minimum-area fan triangulation. For small loops (≤8 edges), exhaustive search is feasible on GPU; for large holes, use the advancing front method:

```glsl
// hole_fill.comp — fill boundary loop with minimum-area fan triangulation
layout(set=0,binding=0) readonly buffer LoopVerts { uint loop[MAX_LOOP_LEN]; };
layout(set=0,binding=1) readonly buffer Pos       { vec3 pos[]; };
layout(set=0,binding=2) buffer NewTris { uvec3 tris[]; uint count; };
layout(push_constant) uniform PC { uint loopLen; uint loopOffset; } pc;
layout(local_size_x=1) in;
void main() {
    // Fan triangulation from first vertex (pivot): N-2 triangles for N-gon
    uint v0 = loop[pc.loopOffset];
    for (uint i=1; i+1<pc.loopLen; i++) {
        uint slot = atomicAdd(count, 1u);
        tris[slot] = uvec3(v0, loop[pc.loopOffset+i], loop[pc.loopOffset+i+1]);
    }
    // For higher quality: Liepa "Filling Holes in Meshes" minimum-area DP algorithm
}
```

[Source: Liepa "Filling Holes in Meshes", SGP 2003; Zhao et al. "A Robust Hole-Filling Algorithm for Triangular Mesh", The Visual Computer 2007]

### 22.4 T-Junction Resolution and Winding Consistency

T-junctions (a vertex of one triangle lying on the edge of another without being shared) cause cracks in rendering. Detect and fix by inserting the T-vertex into the adjacent edge:

```glsl
// tjunction_fix.comp — detect T-junctions: vertex near (but not on) adjacent edge
layout(set=0,binding=0) readonly buffer Pos  { vec3 pos[]; };
layout(set=0,binding=1) readonly buffer Tris { uvec3 tris[]; };
layout(set=0,binding=2) buffer TJunctions { uint tjEdge[]; uint tjVert[]; uint count; };
layout(push_constant) uniform PC { float thresh; uint N_TRIS; } pc;
layout(local_size_x=64) in;
void main() {
    uint t  = gl_GlobalInvocationID.x;
    for (int e=0; e<3; e++) {
        uint va=tris[t][e], vb=tris[t][(e+1)%3];
        // Check all vertices of neighbouring triangles for proximity to edge (va,vb)
        for (uint nt=0; nt<pc.N_TRIS; nt++) {
            if (nt==t) continue;
            for (int k=0; k<3; k++) {
                uint vc = tris[nt][k];
                if (vc==va||vc==vb) continue;
                float d = pointEdgeDist(pos[vc], pos[va], pos[vb]);
                if (d < pc.thresh) {
                    uint slot = atomicAdd(count, 1u);
                    tjEdge[slot] = t*3+e; tjVert[slot]=vc;
                }
            }
        }
    }
}
// CPU post-pass: for each T-junction, split edge (va,vb) at projected point, retopologize
```

[Source: Attene et al. "Polygon Mesh Repairing: An Application Perspective", ACM Computing Surveys 2013; Murali & Funkhouser "Consistent Solid and Boundary Representations from Arbitrary Polygonal Data", I3D 1997]

---

### 23. Constrained Delaunay Refinement

*Audience: systems developers.*

Quality mesh generation for FEM (§45) and simulation requires triangles with bounded aspect ratio (no slivers). Ruppert's algorithm (1995) and Chew's second algorithm iteratively insert circumcentre points to eliminate poor-quality triangles. GPU parallelism targets the parallel circumcentre cavity identification and point insertion, with CPU fallback for conflict resolution.

### 23.1 Triangle Quality Classification

Classify all triangles by their minimum angle (or aspect ratio). Triangles with minimum angle < θ_min (typically 20°–25°) are queued for refinement:

```glsl
// delref_quality.comp — classify triangles by minimum angle
layout(set=0,binding=0) readonly buffer Pos  { vec3 pos[];  };
layout(set=0,binding=1) readonly buffer Tris { uvec3 tri[]; };
layout(set=0,binding=2) buffer BadTris { uint bad[]; uint count; };
layout(push_constant) uniform PC { float minAngle; } pc;
layout(local_size_x=64) in;
void main() {
    uint t  = gl_GlobalInvocationID.x;
    vec3 a  = pos[tri[t].x], b = pos[tri[t].y], c = pos[tri[t].z];
    float A = acos(clamp(dot(normalize(b-a),normalize(c-a)),-1.0,1.0));
    float B = acos(clamp(dot(normalize(a-b),normalize(c-b)),-1.0,1.0));
    float C = 3.14159 - A - B;
    if (min(A,min(B,C)) < pc.minAngle) {
        uint slot = atomicAdd(count, 1u);
        bad[slot]  = t;
    }
}
```

[Source: Ruppert "A Delaunay Refinement Algorithm for Quality 2-Dimensional Mesh Generation", JACM 1995; Shewchuk "Delaunay Refinement Algorithms for Triangular Mesh Generation", Computational Geometry 2002]

### 23.2 Circumcentre Computation and Encroachment Test

For each bad triangle, compute its circumcentre. If the circumcentre is inside the domain and does not encroach on any constrained edge (i.e., does not fall in any constrained edge's diametral circle), it can be inserted:

```glsl
// delref_circumcentre.comp — compute circumcentre and test encroachment
layout(set=0,binding=0) readonly buffer Pos      { vec3 pos[];  };
layout(set=0,binding=1) readonly buffer Tris     { uvec3 tri[]; };
layout(set=0,binding=2) readonly buffer BadTris  { uint bad[];  uint count; };
layout(set=0,binding=3) readonly buffer ConstrEdges { uvec2 cedges[]; uint nCE; };
layout(set=0,binding=4) writeonly buffer InsertPts { vec3 ipts[]; uint valid[]; };
layout(local_size_x=64) in;
void main() {
    uint li = gl_GlobalInvocationID.x;
    if (li >= count) return;
    uint t  = bad[li];
    vec3 a  = pos[tri[t].x], b = pos[tri[t].y], c = pos[tri[t].z];
    // Circumcentre in 2D (project to XZ plane for terrain meshes)
    vec2 cc = circumcentre2D(a.xz, b.xz, c.xz);
    bool enc = false;
    for (uint e=0; e<nCE && !enc; e++) {
        vec2 p = pos[cedges[e].x].xz, q = pos[cedges[e].y].xz;
        enc = dot(cc-(p+q)*0.5, cc-(p+q)*0.5) < dot((q-p)*0.5,(q-p)*0.5);
    }
    ipts[li] = vec3(cc.x, 0.0, cc.y);
    valid[li] = enc ? 0u : 1u;
}
```

### 23.3 Parallel Point Insertion with Conflict Detection

Insert non-conflicting circumcentres in parallel. Two insertions conflict if their cavities (circumscribed empty circles) overlap. Detect and defer conflicts:

```glsl
// delref_insert.comp — parallel insertion with cavity conflict detection
layout(set=0,binding=0) readonly buffer InsertPts { vec3 ipts[];  };
layout(set=0,binding=1) readonly buffer Valid     { uint valid[]; uint count; };
layout(set=0,binding=2) buffer Pos   { vec3 pos[];   };
layout(set=0,binding=3) buffer Tris  { uvec3 tri[];  };
layout(set=0,binding=4) buffer ConflictMask { uint conflict[]; };
layout(local_size_x=64) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    if (i>=count || !valid[i]) return;
    // Find all triangles whose circumcircle contains ipts[i] (the cavity)
    // If another insertion point j is also in this cavity, mark as conflict
    for (uint j=0; j<count; j++) {
        if (j==i || !valid[j]) continue;
        if (pointInCircumcircle(ipts[j], pos, tri, cavityTris[i])) {
            atomicOr(conflict[i], 1u);
            atomicOr(conflict[j], 1u);
        }
    }
}
// Non-conflicting points (conflict[i]==0) are inserted; conflicting points deferred to next round
```

[Source: Shewchuk 2002; Chernikov & Chrisochoides "Parallel Guaranteed Quality Planar Delaunay Mesh Generation by Concurrent Point Insertion", SIAM 2006]

### 23.4 Bowyer-Watson Cavity Retriangulation

After inserting a point, retriangulate its cavity (the set of triangles whose circumcircle contains the new point) using the Bowyer-Watson algorithm:

```glsl
// bowyer_watson.comp — cavity retriangulation after point insertion
layout(set=0,binding=0) buffer Pos  { vec3 pos[];  };
layout(set=0,binding=1) buffer Tris { uvec3 tri[]; };
layout(set=0,binding=2) readonly buffer Cavity { uint cavTris[]; uint nCav; uint newVert; };
layout(set=0,binding=3) buffer BoundaryEdges { uvec2 bEdge[]; uint nBE; };
layout(local_size_x=1) in;  // serial per-point; parallelism across non-conflicting points
void main() {
    // Collect boundary edges of cavity (edges shared by exactly one cavity triangle)
    for (uint i=0;i<nCav;i++) for(int e=0;e<3;e++) {
        uvec2 edge = sortedEdge(tri[cavTris[i]], e);
        bool inner = false;
        for(uint j=0;j<nCav&&!inner;j++) if(j!=i&&hasEdge(tri[cavTris[j]],edge)) inner=true;
        if (!inner) { uint s=atomicAdd(nBE,1u); bEdge[s]=edge; }
    }
    // Delete cavity triangles; connect new vertex to each boundary edge
    for (uint e=0; e<nBE; e++) {
        uint slot = allocTri();
        tri[slot] = uvec3(newVert, bEdge[e].x, bEdge[e].y);
    }
}
```

[Source: Bowyer "Computing Dirichlet Tessellations", Computer Journal 1981; Watson "Computing the n-Dimensional Delaunay Tessellation with Application to Voronoi Polytopes", Computer Journal 1981]

---


---

## III. Spatial Data Structures and Visibility

Spatial data structures are the acceleration infrastructure for ray traversal, occlusion culling, and GPU-driven indirect rendering. This category covers GPU BVH construction (§24 introduces the LBVH radix-sort approach; §31 covers the full parallel SAH-guided builder), Sparse Voxel DAGs for compact voxel scene representation, SDF baking via jump flooding, visibility buffer patterns for decoupled shading, precomputed visibility for portal-based culling, and the GPU-driven indirect draw pipeline that uses these structures to cull meshlets before rasterization.

### 24. GPU BVH Construction

A Bounding Volume Hierarchy (BVH) is the spatial acceleration structure underlying Vulkan ray tracing (Ch135), GPU collision detection, and ray-cast inverse kinematics. For dynamic geometry (skinned characters, deforming cloth), rebuilding the BVH every frame on the GPU eliminates the CPU bottleneck.

### 24.1 Morton Codes and LBVH

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

### 24.2 LBVH Tree Construction (Karras 2012)

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

### 24.3 AABB Propagation

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

**Applications.** The GPU-built LBVH feeds directly into `VkAccelerationStructureBuildGeometryInfoKHR` for Vulkan ray tracing (Ch135) via `VK_ACCELERATION_STRUCTURE_BUILD_MODE_UPDATE_KHR` — the LBVH topology is provided as a host-side hint and the driver refines it. The same structure serves as a broad-phase collision structure for compute-side ragdoll and IK ray-casting (§40) without leaving the GPU.

[Source: Lauterbach et al. (2009), "Fast BVH Construction on GPUs", Eurographics; Karras (2012), "Maximizing Parallelism in the Construction of BVHs, Octrees, and k-d Trees", HPG]

### 24.4 GPU Narrow-Phase Collision: SAT and GJK

The BVH (§24.1–8.3) is a broad-phase structure: it quickly eliminates non-overlapping bounding box pairs. Candidate pairs that pass the broad-phase need a **narrow-phase** test to determine actual geometric contact — position, normal, and penetration depth — for physics constraint generation. Two algorithms dominate GPU narrow-phase: SAT (Separating Axis Theorem) for OBB pairs, and GJK (Gilbert-Johnson-Keerthi) for general convex hulls.

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

**GPU dispatch pattern.** Assign one workgroup per candidate pair from the BVH broad-phase output. For 1000 candidate pairs, a single dispatch of 1000 workgroups (one per pair) completes narrow-phase in ~0.05 ms on a midrange GPU. Contact results feed into a physics constraint solver (PBD §39.6 or a rigid-body impulse solver) in the next compute pass.

[Source: Gilbert, Johnson, Keerthi "A Fast Procedure for Computing the Distance Between Complex Objects in Three-Dimensional Space", IEEE RA 1988; van den Bergen "Collision Detection in Interactive 3D Environments", 2003]

### 24.5 Convex Hull Generation on GPU

A convex hull is the tightest convex polytope enclosing a point set. It is required as: (a) a preprocessing step before GJK (§24.4) — GJK requires convex inputs; (b) the physics collider for rigid bodies before VHACD convex decomposition; (c) the initial seed polytope for EPA (Expanding Polytope Algorithm) penetration depth computation.

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
    // Upload ch.m_points as the convex hull vertex set for GJK §24.4
}
```

V-HACD runs on CPU in 0.1–2 s depending on resolution; the resulting convex pieces are uploaded once as static SSBOs for GJK dispatch at runtime. [Source: Mamou & Ghazali "A Simple and Efficient Approach for 3D Mesh Approximate Convex Decomposition", ICIP 2009; V-HACD v4.0 GitHub https://github.com/kmammou/v-hacd]

**GPU porting status.** No GPU port of VHACD exists or is in active development. The VHACD algorithm's inner loop — iterative voxelization, PCA convex hull approximation, and concavity-driven splitting — involves adaptive tree decomposition with data-dependent termination that does not parallelize efficiently. The more significant shift is that NVIDIA PhysX 5 (Ch69) and GPU-native physics engines increasingly use SDF-based collision geometry rather than convex decomposition: the SDF is computed once on GPU and ray-cast for collision queries, entirely bypassing the need for convex decomposition. For engines that still require explicit convex hulls (e.g., for GJK §24.4 or network-replicated physics shapes), VHACD remains a CPU-only offline preprocessing step.

---

### 25. Screen-Space Geometry Techniques

Screen-space methods approximate geometry operations using the depth buffer, G-buffer, or reprojected history — achieving results close to full geometric solutions at a fraction of the cost by operating in the 2D screen domain rather than the 3D object domain.

### 25.1 Screen-Space Tessellation and Dynamic LOD

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

### 25.2 Parallax Occlusion Mapping

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

### 25.3 Temporal Geometry Reprojection

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

**Geometry reprojection for bent normals / AO.** Progressive bakes (§10.8) use temporal reprojection to accumulate AO samples across frames: each frame adds N new ray samples per vertex; the EMA blend `result = mix(cached, newSample, alpha)` converges to the full-sample result over 1/alpha frames. Motion vectors from the motion-vector pass reproject the cached AO from last frame's vertex positions:

```glsl
// reproject_ao.comp — one thread per vertex
vec2  prevUV    = uv + texture(motionVectors, uv).rg;   // reprojected UV
float cachedAO  = texture(prevAOMap, prevUV).r;
float newAO     = computeAOSample(pos, normal, bvh);    // N rays this frame
outAO[i]        = mix(cachedAO, newAO, 1.0/16.0);      // 16-frame EMA
```

**Temporal geometry caching.** For static environment geometry, cached G-buffer attributes (normals, albedo, depth) from the previous frame can fill pixels that were occluded this frame but visible last frame (disocclusion holes from camera movement). The reprojection confidence is weighted by motion-vector magnitude and depth difference — discard cache hits where the depth delta exceeds a threshold (indicating a different surface is now visible). This is the basis of Unreal Engine's Lumen global illumination cache and DXR "Restir" temporal reuse for ray-traced shadows. [Source: Karis "High Quality Temporal Supersampling", SIGGRAPH 2014; Schied et al. "Spatiotemporal Variance-Guided Filtering", HPG 2017]

---

### 26. GPU-Driven Rendering and Virtual Geometry

*Audience: graphics application developers, systems developers.*

Modern GPU-driven pipelines invert the CPU↔GPU command relationship: the GPU selects what to draw, computes per-object transforms, culls occluded geometry, and emits indirect draw commands — the CPU only submits one or a handful of dispatch calls. This section covers the core GPU data structures and algorithms that make this possible, from meshlet decomposition through cluster-DAG LOD selection.

### 26.1 Meshlets and Cluster Hierarchies

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

### 26.2 Amplification/Task Shader Culling Pipeline

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

### 26.3 Hierarchical Z-Buffer Occlusion Culling

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

### 26.4 Two-Phase Occlusion and Indirect Draw Compaction

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

### 26.5 Virtual Geometry: Cluster DAG and Error-Metric LOD

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

### 27. Spatial Data Structures Beyond BVH

*Audience: systems developers, graphics application developers.*

§24 covers GPU LBVH construction for ray-surface intersection. Other spatial data structures serve different access patterns: k-d trees for nearest-neighbour queries, uniform grids for short-range interactions (SPH, collision), and sparse voxel octrees for multi-resolution volume traversal.

### 27.1 GPU k-d Trees

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

### 27.2 Uniform Grids and Cell-List Hashing

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

### 27.3 Sparse Voxel Octrees (SVO)

An SVO stores only non-empty octree nodes, indexed by Morton (Z-curve) codes. GPU construction (Laine & Karras 2010):

1. **Compute Morton codes** for all triangles (centroid → 30-bit Morton code). Radix sort.
2. **Bottom-up merge:** Threads at level L examine adjacent codes; if they share a common ancestor at level L, mark as one node.
3. **Compact** non-empty nodes into a flat SSBO with parent/child pointers stored as integer offsets.

GPU ray traversal follows the standard octree DDA: descend to the deepest non-empty child overlapping the ray, march forward, ascend when the ray exits a node. A **contour** field at each SDF leaf enables sub-voxel geometry detail. NanoVDB (§3.6) implements a production-quality version of this layout optimised for GPU cache coherence. [Source: Laine & Karras "Efficient Sparse Voxel Octrees", I3D 2010; Crassin et al. "GigaVoxels: Ray-Guided Streaming for Efficient and Detailed Voxel Rendering", I3D 2009]

---

### 28. Precomputed Visibility and Lightmap Baking

*Audience: graphics application developers.*

Lightmap baking pre-integrates indirect illumination into per-texel texture samples, allowing high-quality global illumination at zero runtime cost. The GPU-accelerated bake pipeline dispatches one ray (or hemisphere of rays) per atlas texel using the VK_KHR_ray_tracing_pipeline from §75.

### 28.1 Texel-World Position Atlas

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

### 28.2 Hemisphere Ray Batching for AO

Each texel dispatches N ray queries against the scene TLAS (§75.3) in a hemisphere centred on the surface normal. Uses `VK_KHR_ray_query` (inline ray queries in compute) to avoid full RT pipeline overhead for AO:

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

### 28.3 Irradiance Baking with Spherical Harmonics

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

### 28.4 Lightmap Dilation and Padding

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

### 29. Visibility Queries and GPU Occlusion

*Audience: graphics application developers, systems developers.*

§26.3 covered Hierarchical Z-Buffer (HZB) occlusion culling within the GPU-Driven Rendering pipeline. This section covers the broader family of GPU visibility query mechanisms: hardware occlusion queries, software occlusion rasterisers in compute, and portal/sector GPU culling for indoor scenes.

### 29.1 Hardware Occlusion Queries

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

### 29.2 Software Occlusion Rasteriser in Compute

A software occlusion rasteriser (SOR) in compute shaders allows the same-frame query results needed for GPU-driven pipelines (§26.4 two-phase occlusion). The SOR builds a low-resolution depth buffer (e.g., 256×128) from occluder proxies in compute, then tests occludee AABBs against it:

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

### 29.3 Portal and Sector Visibility Culling

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

Iterate BFS until no new sectors are added. Objects in non-visible sectors are excluded from the indirect draw buffer (§26.4). [Source: Luebke & Georges "Portals and Mirrors: Simple, Fast Evaluation of Potentially Visible Sets", I3D 1995]

### 29.4 Conditional Rendering via Predication

`VK_EXT_conditional_rendering` allows draw calls to be skipped based on a buffer value without CPU readback, closing the loop between hardware OQ results (§29.1) and draw culling in a single frame:

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

This avoids the 1–2 frame readback latency of §29.1 at the cost of conservative culling (last frame's results drive this frame's draws). [Source: Vulkan Spec VK_EXT_conditional_rendering]

---

### 30. Sparse Voxel DAG (SVDAG)

*Audience: systems developers.*

The Sparse Voxel DAG (Kampe et al. 2013) is a losslessly compressed SVO: identical subtrees are merged into a directed acyclic graph, achieving 5–10× compression over SVO while supporting the same ray traversal. The DAG represents binary (solid/empty) voxel geometry and is the standard data structure for GPU ambient occlusion precomputation at extreme resolution (up to 2²⁴ voxels).

### 30.1 SVDAG Construction: Subtree Hashing

Build the SVO bottom-up (§27.3), then hash each internal node by its children's hashes. Nodes with identical hash are merged:

```glsl
// svdag_hash.comp — hash each SVO node at level L (leaves first)
layout(set=0,binding=0) readonly buffer SVONodes { SVONode nodes[]; };
layout(set=0,binding=1) writeonly buffer Hashes  { uint64_t hashes[]; };
layout(local_size_x=64) in;
void main() {
    uint n = gl_GlobalInvocationID.x;
    if (!isLevel(nodes[n], CURRENT_LEVEL)) return;
    // FNV-1a hash of 8 child hashes
    uint64_t h = FNV_OFFSET_BASIS;
    for (int c=0; c<8; c++) {
        uint64_t ch = nodes[n].childMask & (1u<<c) ?
                      hashes[nodes[n].children[c]] : EMPTY_HASH;
        h = (h ^ ch) * FNV_PRIME;
    }
    hashes[n] = h;
}
// GPU sort by hash → find duplicates → remap children to canonical representative
```

[Source: Kampe et al. "High Resolution Sparse Voxel DAGs", SIGGRAPH 2013; https://github.com/Phyronnaz/DAGCompression]

### 30.2 SVDAG Ray Traversal

SVDAG traversal is identical to SVO DDA (§27.3) except that multiple SVO nodes map to the same DAG node:

```glsl
// svdag_traverse.comp — ray traverse SVDAG for AO visibility
layout(set=0,binding=0) readonly buffer SVDAGNodes { uint children[8]; uint childMask; } dag[];
layout(set=0,binding=1) readonly buffer Remap { uint svoToDAG[]; };
layout(set=0,binding=2, r8) uniform writeonly image2D occOut;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    vec3  ro  = gPos[pix], rd = cosineSampleHemisphere(gNorm[pix], halton2D(pix));
    // Stack-based DDA traversal of SVDAG
    uint stack[MAX_DEPTH];  int sp = 0;
    uint node = svoToDAG[0];  // root
    float t   = 0.0;
    while (sp >= 0 && t < AO_RADIUS) {
        if (dag[node].childMask == 0) {  // leaf: solid voxel hit
            imageStore(occOut, pix, vec4(0.0)); return;
        }
        uint childIdx = childIndexAlongRay(ro+rd*t, depth(sp));
        if (dag[node].childMask & (1u<<childIdx)) {
            stack[sp++] = node;
            node = svoToDAG[dag[node].children[childIdx]];
        } else {
            t = nextEmptyChild(ro, rd, depth(sp));
        }
    }
    imageStore(occOut, pix, vec4(1.0));  // no hit: unoccluded
}
```

[Source: Kampe et al. 2013; Villanueva et al. "SSVDAG: Symmetry-Aware Sparse Voxel DAGs", I3D 2016]

### 30.3 Clustered SVDAG for Dynamic Scenes

For dynamic objects, maintain a Clustered SVDAG (CDAG): the scene is split into static (full SVDAG) and dynamic (per-object SVO, merged at runtime). Each frame, re-insert dynamic SVOs into the DAG:

```glsl
// cdag_merge.comp — re-insert dynamic SVO nodes into static SVDAG root
layout(set=0,binding=0) buffer SVDAGRoot { uint dag[]; };
layout(set=0,binding=1) readonly buffer DynSVO  { SVONode dynNodes[]; };
layout(local_size_x=64) in;
void main() {
    uint n = gl_GlobalInvocationID.x;
    if (!dynNodes[n].dirty) return;
    // Walk SVDAG from root to this node's position, OR in child mask bit
    ivec3 voxCoord = dynNodes[n].voxelCoord;
    uint dagNode   = 0u;  // root
    for (int d=MAX_DEPTH-1; d>=0; d--) {
        uint childIdx = (voxCoord>>d) & 7u;
        atomicOr(dag[dagNode*9+8], 1u<<childIdx);  // set child-present bit
        dagNode = dag[dagNode*9+childIdx];
    }
}
```

[Source: Laine & Karras "Efficient Sparse Voxel Octrees", I3D 2010; Crassin et al. 2011 (§51.3)]

---

### 31. GPU BVH Construction

*Audience: systems developers, graphics application developers.*

Every ray tracing pipeline (§75), rigid body collision system (§50), and acoustic ray tracer (§99) consumes a BVH — but §75 only covers traversal. This section covers GPU construction: Linear BVH (LBVH) via Morton codes and radix sort, the SAH-HLBVH hybrid for better tree quality, and PLOC for parallel locally-ordered clustering that approaches offline SAH quality at GPU speeds.

### 31.1 Morton Code Generation

Assign each primitive a 30-bit Morton code by interleaving the x, y, z bits of its normalised centroid coordinate. All codes are computed in parallel:

```glsl
// morton.comp — compute Morton codes for AABB centroids
layout(set=0,binding=0) readonly buffer Centroids { vec3 cen[]; };
layout(set=0,binding=1) writeonly buffer Codes { uint64_t codes[]; };
layout(push_constant) uniform PC { vec3 sceneMin; vec3 sceneExtent; uint N; } pc;
layout(local_size_x=64) in;

uint expandBits(uint v) {           // 10-bit → 30-bit interleave
    v = (v * 0x00010001u) & 0xFF0000FFu;
    v = (v * 0x00000101u) & 0x0F00F00Fu;
    v = (v * 0x00000011u) & 0xC30C30C3u;
    v = (v * 0x00000005u) & 0x49249249u;
    return v;
}
void main() {
    uint i   = gl_GlobalInvocationID.x;
    if (i >= pc.N) return;
    vec3  n  = (cen[i] - pc.sceneMin) / pc.sceneExtent;  // [0,1]³
    uint  xi = clamp(uint(n.x*1024), 0u, 1023u);
    uint  yi = clamp(uint(n.y*1024), 0u, 1023u);
    uint  zi = clamp(uint(n.z*1024), 0u, 1023u);
    uint  mc = (expandBits(xi)<<2) | (expandBits(yi)<<1) | expandBits(zi);
    codes[i] = (uint64_t(mc) << 32) | uint64_t(i);  // high = code, low = prim index
}
```

[Source: Lauterbach et al. "Fast BVH Construction on GPUs", EG 2009; Pantaleoni & Luebke "HLBVH: Hierarchical LBVH Construction for Real-Time Ray Tracing of Dynamic Geometry", HPG 2010]

### 31.2 LBVH Internal Node Construction

After GPU radix sort of Morton codes (§25.4), construct the binary radix tree in O(N) by finding the split position of each internal node (the highest differing bit between the leftmost and rightmost Morton code in its range):

```glsl
// lbvh_build.comp — construct LBVH internal nodes from sorted Morton codes
layout(set=0,binding=0) readonly buffer SortedCodes { uint64_t codes[]; };
layout(set=0,binding=1) writeonly buffer Nodes { BVHNode nodes[]; };  // 2N-1 nodes total
layout(push_constant) uniform PC { uint N; } pc;
layout(local_size_x=64) in;

int commonUpperBits(uint a, uint b) { return 31 - findMSB(a ^ b); }

void main() {
    uint i = gl_GlobalInvocationID.x;
    if (i >= pc.N - 1) return;  // N-1 internal nodes
    uint ci = uint(codes[i] >> 32), ci1 = uint(codes[i+1] >> 32);
    // Determine which direction the range extends
    int d = sign(commonUpperBits(ci, uint(codes[min(i+1,pc.N-1)]>>32))
                -commonUpperBits(ci, uint(codes[max(i-1,0u)  >>32])));
    // Binary search for the other end of the range, then the split
    uint jMin = i, jMax = i + uint(d) * (pc.N-1);
    uint split = (jMin + jMax) / 2u;
    // Set left/right children (leaf if single element, internal node otherwise)
    nodes[i].left  = (min(split, jMax) == split) ? split + pc.N-1 : split;
    nodes[i].right = (max(split+1,jMin)==split+1) ? split+1+pc.N-1: split+1;
    nodes[nodes[i].left ].parent = i;
    nodes[nodes[i].right].parent = i;
}
```

[Source: Karras "Maximizing Parallelism in the Construction of BVHs, Octrees, and k-d Trees", HPG 2012]

### 31.3 Bottom-Up AABB Refitting

After building the tree topology in §31.2, refit each internal node's AABB bottom-up using an atomic counter to ensure each parent is processed only after both children are done:

```glsl
// lbvh_refit.comp — bottom-up AABB refit via atomic parent counter
layout(set=0,binding=0) buffer Nodes { BVHNode nodes[]; };
layout(set=0,binding=1) buffer Counter { uint cnt[]; };  // per internal node
layout(push_constant) uniform PC { uint N; } pc;
layout(local_size_x=64) in;
void main() {
    uint leaf = gl_GlobalInvocationID.x;
    if (leaf >= pc.N) return;
    uint node = nodes[leaf + pc.N - 1].parent;  // start from leaf's parent
    while (node != INVALID) {
        uint prev = atomicAdd(cnt[node], 1u);
        if (prev == 0u) return;  // first thread to reach this node — wait for sibling
        // Both children done: merge their AABBs
        nodes[node].aabbMin = min(nodes[nodes[node].left].aabbMin,
                                  nodes[nodes[node].right].aabbMin);
        nodes[node].aabbMax = max(nodes[nodes[node].left].aabbMax,
                                  nodes[nodes[node].right].aabbMax);
        node = nodes[node].parent;
    }
}
```

[Source: Karras 2012; Aila & Laine "Understanding the Efficiency of Ray Traversal on GPUs", HPG 2009]

### 31.4 PLOC: Parallel Locally-Ordered Clustering

PLOC (Meister & Bittner 2018) achieves near-SAH quality by iteratively merging nearest neighbours in Morton-code-sorted order. Each pass considers a fixed neighbourhood radius r, finds the best merge partner for each cluster, and forms parent nodes:

```glsl
// ploc_merge.comp — one PLOC merge pass
layout(set=0,binding=0) buffer Clusters { uint clusterIdx[]; uint N; };
layout(set=0,binding=1) readonly buffer NodeAABB { vec3 aabbMin[]; vec3 aabbMax[]; };
layout(set=0,binding=2) buffer Parent { uint parent[]; };
layout(push_constant) uniform PC { int radius; } pc;
layout(local_size_x=64) in;
float surfaceArea(vec3 mn, vec3 mx) { vec3 e=mx-mn; return 2*(e.x*e.y+e.y*e.z+e.z*e.x); }
void main() {
    uint i = gl_GlobalInvocationID.x;
    if (i >= Clusters.N) return;
    uint ci = clusterIdx[i];
    uint best = INVALID; float bestSA = 1e30;
    for (int d = -pc.radius; d <= pc.radius; d++) {
        if (d == 0 || int(i)+d < 0 || i+d >= Clusters.N) continue;
        uint cj = clusterIdx[i+d];
        float sa = surfaceArea(min(aabbMin[ci],aabbMin[cj]),
                               max(aabbMax[ci],aabbMax[cj]));
        if (sa < bestSA) { bestSA=sa; best=cj; }
    }
    parent[ci] = best;  // mutual best pairs → merge into new internal node
}
```

[Source: Meister & Bittner "Parallel Locally-Ordered Clustering for Bounding Volume Hierarchy Construction", IEEE TVCG 2018]

---

### 32. SDF Baking and 3D Jump Flooding

*Audience: systems developers, graphics application developers.*

§64 uses SDFs for collision detection and §7 for CSG, but both assume the SDF already exists. This section covers GPU algorithms for *generating* an SDF from a triangle mesh: brute-force per-voxel closest-triangle search, 3D jump flooding, and the exact two-pass method used in game engines for runtime SDF baking.

### 32.1 Brute-Force SDF: Closest Triangle per Voxel

For small meshes or low-resolution grids, compute each voxel's signed distance by checking all triangles:

```glsl
// sdf_brute.comp — brute-force signed distance for each voxel
layout(set=0,binding=0) readonly buffer Verts { vec3 verts[]; };
layout(set=0,binding=1) readonly buffer Tris  { uvec3 tris[]; };
layout(set=0,binding=2, r32f) uniform writeonly image3D sdfOut;
layout(push_constant) uniform PC { vec3 gridMin; float dx; ivec3 dim; uint nTris; } pc;
layout(local_size_x=4,local_size_y=4,local_size_z=4) in;
void main() {
    ivec3 id  = ivec3(gl_GlobalInvocationID);
    vec3  p   = pc.gridMin + (vec3(id)+0.5)*pc.dx;
    float minD = 1e30; vec3 closestN = vec3(0,1,0);
    for (uint t=0; t<pc.nTris; t++) {
        vec3 a=verts[tris[t].x], b=verts[tris[t].y], c=verts[tris[t].z];
        float d = pointTriangleDist(p, a, b, c);
        if (d < minD) { minD=d; closestN=normalize(cross(b-a,c-a)); }
    }
    float sign = dot(p-verts[tris[0].x], closestN) < 0.0 ? -1.0 : 1.0;
    imageStore(sdfOut, id, vec4(sign * minD));
}
```

[Source: Bridson "Fluid Simulation for Computer Graphics", 2nd ed., §3; Jones et al. "3D Distance Fields: A Survey of Techniques and Applications", IEEE TVCG 2006]

### 32.2 3D Jump Flooding for Unsigned Distance

Extend the 2D JFA (§18.1) to 3D: each voxel stores the nearest seed (surface sample) position. Run log₂(max(W,H,D)) passes with halving step sizes:

```glsl
// jfa3d_sdf.comp — 3D jump flooding pass for SDF baking
layout(set=0,binding=0) uniform sampler3D jfaIn;   // RGBA32F: nearest seed xyz + unused
layout(set=0,binding=1, rgba32f) uniform writeonly image3D jfaOut;
layout(push_constant) uniform PC { int step; } pc;
layout(local_size_x=4,local_size_y=4,local_size_z=4) in;
void main() {
    ivec3 p   = ivec3(gl_GlobalInvocationID);
    vec4  best = texelFetch(jfaIn, p, 0);
    float bestD = (best.w > 0.5) ? length(vec3(p)-best.xyz) : 1e30;
    for (int dz=-1;dz<=1;dz++) for(int dy=-1;dy<=1;dy++) for(int dx=-1;dx<=1;dx++) {
        if (dx==0&&dy==0&&dz==0) continue;
        ivec3 nb  = p + ivec3(dx,dy,dz)*pc.step;
        if (any(lessThan(nb,ivec3(0)))||any(greaterThanEqual(nb,GRID_DIM))) continue;
        vec4  nbv = texelFetch(jfaIn, nb, 0);
        if (nbv.w < 0.5) continue;
        float d   = length(vec3(p)-nbv.xyz);
        if (d < bestD) { bestD=d; best=nbv; }
    }
    imageStore(jfaOut, p, best);
}
// After JFA: sign pass — ray cast from each voxel to determine inside/outside
```

[Source: Rong & Tan 2006 (§67.1); Cao et al. "Parallel Banding Algorithm to Compute Exact Distance Transform with the GPU", I3D 2010]

### 32.3 Sign Determination via Parity Ray Cast

After JFA gives unsigned distance, determine the sign with a parity test: cast a ray from each voxel along +X, count triangle intersections; odd count → inside:

```glsl
// sdf_sign.comp — determine inside/outside via ray parity test
layout(set=0,binding=0) readonly buffer Verts { vec3 verts[]; };
layout(set=0,binding=1) readonly buffer Tris  { uvec3 tris[]; };
layout(set=0,binding=2) buffer SDFVol { float sdf[]; };
layout(push_constant) uniform PC { vec3 gridMin; float dx; ivec3 dim; uint nTris; } pc;
layout(local_size_x=4,local_size_y=4,local_size_z=4) in;
void main() {
    ivec3 id  = ivec3(gl_GlobalInvocationID);
    vec3  p   = pc.gridMin + (vec3(id)+0.5)*pc.dx;
    int   cnt = 0;
    for (uint t=0; t<pc.nTris; t++) {
        vec3 a=verts[tris[t].x], b=verts[tris[t].y], c=verts[tris[t].z];
        // Möller-Trumbore ray-triangle intersect along +X ray from p
        if (rayTriIntersectX(p, a, b, c)) cnt++;
    }
    uint flat = id.z*pc.dim.y*pc.dim.x + id.y*pc.dim.x + id.x;
    if ((cnt & 1) == 1) sdf[flat] = -abs(sdf[flat]);  // inside: negate
}
```

[Source: Sethian "Level Set Methods and Fast Marching Methods", 1999; Osman "GPU-Accelerated SDF Generation for Real-Time Physics", GDC 2019]

### 32.4 Narrow-Band SDF via BVH Lookup

For production use (game engine physics baking), limit the expensive closest-triangle search to a narrow band near the surface (|φ| < threshold), using the BVH (§31) to accelerate the nearest-triangle query:

```glsl
// sdf_bvh.comp — narrow-band SDF via BVH nearest-triangle traversal
layout(set=0,binding=0) readonly buffer BVH { BVHNode nodes[]; };
layout(set=0,binding=1) readonly buffer Verts { vec3 verts[]; };
layout(set=0,binding=2) readonly buffer Tris  { uvec3 tris[]; };
layout(set=0,binding=3) buffer SDFVol { float sdf[]; };
layout(push_constant) uniform PC { vec3 gridMin; float dx; float band; } pc;
layout(local_size_x=4,local_size_y=4,local_size_z=4) in;
void main() {
    ivec3 id  = ivec3(gl_GlobalInvocationID);
    vec3  p   = pc.gridMin + (vec3(id)+0.5)*pc.dx;
    // BVH nearest-primitive traversal (priority queue by node AABB distance)
    float minD = bvhNearestTriangle(p, nodes, verts, tris);
    if (minD > pc.band) { sdf[flatIdx(id)] = pc.band; return; }  // clamp outside band
    sdf[flatIdx(id)] = minD * signFromParity(p, tris, verts);
}
```

[Source: Mauch "Efficient Algorithms for Solving Static Hamilton-Jacobi Equations", Caltech PhD 2003; Unreal Engine "Distance Field Soft Shadows" implementation, UE5 source]

---

### 33. Virtual Geometry: GPU-Driven Cluster LOD

*Audience: graphics application developers, systems developers.*

Virtual geometry (Epic's Nanite, 2021) eliminates traditional LOD selection in favour of a GPU-driven pipeline that culls and renders micro-triangle clusters at a granularity finer than a draw call. The key insight: precompute a DAG of cluster groups from coarse to fine, then at runtime traverse the DAG on the GPU to select the finest clusters whose projected screen error stays below one pixel. §26 covers GPU-driven indirect draw; this section covers the cluster DAG construction, GPU traversal, software rasterisation, and visibility buffer shading.

### 33.1 Cluster DAG Construction (Offline)

The mesh is partitioned into clusters of 128 triangles each using METIS-style graph partitioning. Adjacent clusters are then grouped (8 clusters per group) and simplified with QEM (§10) to produce a coarser level. The DAG encodes, for each cluster group, a bounding sphere and a maximum screen-space error:

```glsl
// cluster_dag_build.comp — compute max edge error for a simplified cluster group
layout(set=0,binding=0) readonly buffer ClusterVerts { vec3 verts[]; };
layout(set=0,binding=1) readonly buffer OrigVerts    { vec3 orig[];  };
layout(set=0,binding=2) writeonly buffer ClusterError { float err[];  };
layout(push_constant) uniform PC { uint N_CLUSTERS; float epsilon; } pc;
layout(local_size_x=64) in;
void main() {
    uint c   = gl_GlobalInvocationID.x;
    // Error = max Hausdorff distance from simplified cluster to original
    float maxD = 0.0;
    for (uint v=clusterStart[c]; v<clusterEnd[c]; v++) {
        float d = closestPointDist(verts[v], orig, origStart[c], origEnd[c]);
        maxD = max(maxD, d);
    }
    err[c] = maxD;
}
```

[Source: Karis et al. "A Deep Dive into Nanite Virtualized Geometry", SIGGRAPH 2021; Garland & Heckbert §7]

### 33.2 GPU Cluster DAG Traversal

At runtime, traverse the LOD DAG top-down using a persistent thread group. For each cluster group, project its bounding sphere to screen space and compare the error to one pixel. If the error is acceptable, emit all clusters in this group; otherwise, recurse into finer children:

```glsl
// cluster_traverse.comp — GPU DAG traversal, emit visible clusters
layout(set=0,binding=0) readonly buffer DAGNodes { ClusterGroup dag[]; };
layout(set=0,binding=1) buffer QueueIn  { uint queue[]; uint qHead; uint qTail; };
layout(set=0,binding=2) buffer QueueOut { uint visible[]; uint count; };
layout(push_constant) uniform PC { mat4 viewProj; vec3 camPos; float pixelError; uint N_ROOTS; } pc;
layout(local_size_x=64) in;
void main() {
    uint tid = gl_GlobalInvocationID.x;
    // Persistent: each thread pulls one group from the queue
    while (true) {
        uint slot = atomicAdd(qHead, 1u);
        if (slot >= qTail) break;
        uint g   = queue[slot];
        ClusterGroup cg = dag[g];
        // Project bounding sphere: screen-space error = worldError * focalLen / dist
        float dist  = length(cg.boundSphere.xyz - pc.camPos);
        float ssErr = cg.maxError * FOCAL_LEN / max(dist - cg.boundSphere.w, 1e-3);
        if (ssErr < pc.pixelError || cg.childCount == 0u) {
            // Accept this LOD level: emit all clusters in group
            for (uint c=0; c<cg.clusterCount; c++) {
                uint vs = atomicAdd(count, 1u);
                visible[vs] = cg.firstCluster + c;
            }
        } else {
            // Recurse into children
            for (uint ci=0; ci<cg.childCount; ci++) {
                uint cs = atomicAdd(qTail, 1u);
                queue[cs] = cg.firstChild + ci;
            }
        }
    }
}
```

[Source: Karis et al. 2021; Wihlidal "Optimizing the Graphics Pipeline with Compute", GDC 2016]

### 33.3 Software Rasterisation for Micro-Triangles

Clusters selected at the finest LOD often contain triangles smaller than a pixel. Nanite uses software rasterisation via atomicMax on a 64-bit depth+cluster ID buffer, bypassing the hardware rasteriser for these tiny triangles:

```glsl
// software_rast.comp — rasterise micro-triangles via 64-bit atomic depth buffer
layout(set=0,binding=0) readonly buffer Clusters  { ClusterData clusters[]; };
layout(set=0,binding=1) readonly buffer Verts     { vec3 verts[];  };
layout(set=0,binding=2) readonly buffer Indices   { uint  idx[];   };
layout(set=0,binding=3) buffer DepthCluster { uint64_t depthCluster[]; };  // hi=depth, lo=cluster+tri
layout(push_constant) uniform PC { mat4 mvp; uvec2 res; } pc;
layout(local_size_x=64) in;
void main() {
    uint tri  = gl_GlobalInvocationID.x;
    // Project triangle vertices
    vec2 p0 = project(verts[idx[tri*3+0]], pc.mvp, pc.res);
    vec2 p1 = project(verts[idx[tri*3+1]], pc.mvp, pc.res);
    vec2 p2 = project(verts[idx[tri*3+2]], pc.mvp, pc.res);
    float z0=projZ(verts[idx[tri*3+0]],pc.mvp), z1=projZ(verts[idx[tri*3+1]],pc.mvp), z2=projZ(verts[idx[tri*3+2]],pc.mvp);
    // Scan-line rasterise AABB of triangle
    ivec2 minP=ivec2(floor(min(p0,min(p1,p2)))), maxP=ivec2(ceil(max(p0,max(p1,p2))));
    for (int y=minP.y;y<=maxP.y;y++) for(int x=minP.x;x<=maxP.x;x++) {
        vec3 bary = barycentrics(vec2(x,y),p0,p1,p2);
        if (any(lessThan(bary,vec3(0)))) continue;
        float z    = bary.x*z0+bary.y*z1+bary.z*z2;
        uint64_t depthID = (uint64_t(floatBitsToUint(z))<<32)|uint64_t(tri);
        atomicMax(depthCluster[y*pc.res.x+x], depthID);
    }
}
```

[Source: Karis et al. 2021 §8; Wihlidal 2016; Olsson "Cluster-Based Rendering", SIGGRAPH 2020]

### 33.4 Visibility Buffer Shading

The visibility buffer (Burns & Hunt 2013) stores cluster ID + triangle ID per pixel instead of shading attributes. A deferred shading pass reconstructs material attributes by re-deriving barycentric coordinates:

```glsl
// vbuffer_shade.comp — shade from visibility buffer (cluster+tri ID per pixel)
layout(set=0,binding=0) readonly buffer DepthCluster { uint64_t depthCluster[]; };
layout(set=0,binding=1) readonly buffer Verts        { vec3 verts[];   };
layout(set=0,binding=2) readonly buffer UV           { vec2 uvs[];     };
layout(set=0,binding=3) uniform sampler2DArray atlas; // virtual texture atlas (§28)
layout(set=0,binding=4, rgba16f) writeonly image2D shaded;
layout(push_constant) uniform PC { mat4 mvp; uvec2 res; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    uint64_t dc = depthCluster[pix.y*pc.res.x+pix.x];
    uint triID = uint(dc & 0xFFFFFFFF);
    // Re-derive barycentrics from pixel position + triangle vertices
    vec2 p     = vec2(pix)+0.5;
    vec2 p0=project(verts[idx[triID*3+0]],pc.mvp,pc.res);
    vec2 p1=project(verts[idx[triID*3+1]],pc.mvp,pc.res);
    vec2 p2=project(verts[idx[triID*3+2]],pc.mvp,pc.res);
    vec3 bary  = barycentrics(p,p0,p1,p2);
    vec2 uv    = bary.x*uvs[idx[triID*3+0]]+bary.y*uvs[idx[triID*3+1]]+bary.z*uvs[idx[triID*3+2]];
    vec4 color = texture(atlas, vec3(uv, float(materialID(triID))));
    imageStore(shaded, pix, color);
}
```

[Source: Burns & Hunt "The Visibility Buffer: A Cache-Friendly Approach to Deferred Shading", JCGT 2013; Engel "Deferred+", GDC 2019]

---


---

## IV. Differential Geometry and Analysis

Differential geometry provides the mathematical language for curvature, geodesics, and spectral analysis on discrete triangle meshes. This category covers the cotangent-weighted Laplace-Beltrami operator and its use in curvature estimation and smoothing, geodesic distance computation via the heat method (which reduces to two linear solves), spectral decomposition of the mesh Laplacian for shape analysis and compression, and functional maps for correspondence and shape transfer between meshes with different connectivity. These techniques are primarily used in geometry processing tools and offline analysis pipelines rather than in real-time rendering.

### 34. Geodesics and Discrete Differential Geometry

*Audience: graphics application developers, systems developers.*

Discrete differential geometry provides a rigorous framework for computing smooth differential quantities (curvature, Laplacian, geodesic distance) on triangle meshes. All core operations reduce to sparse linear algebra on the mesh adjacency graph — well-suited for GPU conjugate gradient solvers.

### 34.1 Cotangent Laplacian

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

### 34.2 Heat Method for Geodesic Distance

The **heat method** (Crane, Weischedel, Wardetzky 2013) computes exact geodesic distances on triangle meshes in three GPU steps:

**Step 1 — Heat flow:** Solve `(M − δt L) u = δ` where M is the mass matrix, L is the cotangent Laplacian, δ is a heat source at the query vertex, and δt = h² (h = mean edge length). One sparse CG solve (∼20 iterations):

```glsl
// Preconditioned CG loop (one kernel per iteration, barrier between)
// A = M - dt*L,  b = delta_source,  solve A*u = b
```

**Step 2 — Normalize gradient:** Per-triangle compute ∇u, then normalize: X = −∇u / ‖∇u‖. This is fully parallel (one thread per triangle).

**Step 3 — Poisson solve:** Solve `L φ = ∇·X` (divergence of the normalized gradient field). Another sparse CG solve. The result φ is the geodesic distance field, shifted to zero at the source vertex.

All three steps use the same cotangent Laplacian built once per mesh. Total GPU time for a 100k-vertex mesh: ∼5 ms on modern hardware. [Source: Crane et al. "Geodesics in Heat", ACM TOG 2013; code at https://github.com/nmwsharp/geometry-central]

### 34.3 Mean Curvature Flow and Mesh Smoothing

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

### 34.4 Principal Curvature Estimation on GPU

Principal curvatures κ₁, κ₂ and their directions e₁, e₂ are the eigenvalues/vectors of the shape operator at each vertex. GPU estimation uses the per-triangle curvature tensor method (Rusinkiewicz 2004):

1. Per-triangle: fit a 2×2 curvature tensor from position and normal differences at the triangle's vertices (parallel).
2. Per-vertex: scatter-accumulate (atomicAdd) weighted triangle tensors onto vertex SSBO.
3. Per-vertex: 2×2 symmetric eigendecomposition (closed-form, one thread per vertex) → κ₁, κ₂, e₁, e₂.

Principal curvatures feed downstream algorithms: anisotropic remeshing (align quads to curvature directions), feature line extraction (ridges where κ₁ is locally maximal), and shading. [Source: Rusinkiewicz "Estimating Curvatures and Their Derivatives on Triangle Meshes", 3DPVT 2004]

---

### 35. Spectral Geometry Processing

*Audience: systems developers, graphics application developers.*

Spectral geometry processing applies signal-processing concepts (filtering, frequency decomposition) to triangle meshes by treating the cotangent Laplacian as the mesh's "frequency operator". Low-frequency eigenvectors correspond to smooth, large-scale shape variations; high-frequency eigenvectors capture fine surface detail.

### 35.1 Graph Laplacian Eigenbasis

The graph Laplacian L = D − A (degree matrix minus adjacency) has V eigenvectors Φ = [φ₁, …, φ_V] and eigenvalues Λ = diag(λ₁, …, λ_V). Any per-vertex scalar function f decomposes as f̂ = Φᵀ f. Filtered reconstruction: f' = Φ h(Λ) f̂ where h is the frequency filter (e.g., low-pass: h(λ) = exp(−t λ)).

Full eigenbasis computation (O(V³)) is impractical for large meshes. **Chebyshev polynomial approximation** avoids explicit eigenvectors: any spectral filter h(L) f ≈ Σₖ cₖ Tₖ(L) f, where Tₖ are Chebyshev polynomials evaluated via the recurrence Tₖ(L) f = 2L·Tₖ₋₁(L)f − Tₖ₋₂(L)f. Each step is one sparse matrix-vector product — directly GPU-parallelisable.

[Source: Shuman et al. "The Emerging Field of Signal Processing on Graphs", IEEE Signal Processing Magazine 2013; Defferrard et al. "Convolutional Neural Networks on Graphs with Fast Localized Spectral Filtering", NIPS 2016]

### 35.2 Bilateral Mesh Filtering

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

### 35.3 Anisotropic Diffusion and Curvature Flow

**Anisotropic diffusion** (Clarenz et al. 2000) generalises the Laplacian smoother by introducing a per-vertex diffusion tensor D(∇n) that suppresses diffusion across edges (where the normal gradient is large) while amplifying diffusion along edges:

```
dx/dt = div(D(∇n) · ∇x)
```

On a triangle mesh, the per-edge weight replaces the cotangent weight in the Laplacian with w_ij · g(‖nᵢ − nⱼ‖) where g is a decreasing edge-stopping function (e.g., g(s) = 1/(1 + s²/K²)). One CG solve per time step.

**Willmore flow** minimises the total squared mean curvature ∫ H² dA, producing globally fair surfaces. The gradient flow involves fourth-order differential operators (biharmonic) — requires solving L Δx = f where L is the Laplacian squared, achievable as two cascaded CG solves. [Source: Clarenz et al. "Anisotropic Geometric Diffusion in Surface Processing", VIS 2000; Crane et al. "Robust Fairing via Conformal Curvature Flow", SIGGRAPH 2013]

---

### 36. Geometric Shape Morphing and Functional Maps

*Audience: graphics application developers.*

Shape morphing creates a smooth geometric transition between two shapes with potentially different topology. Unlike blend shapes (§39.5, which require identical topology), functional maps establish point-to-point correspondence between different meshes via their spectral representations, enabling morphing between a human hand and a cat paw, or completing a partial scan.

### 36.1 Functional Maps: Spectral Correspondence

A functional map C ∈ ℝ^{k×k} represents a map between two shapes M and N in the basis of their first k Laplacian eigenfunctions (§35.1). Given descriptor functions on M (e.g., WKS, HKS), solve for C minimising:

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

### 36.2 Point-to-Point Map from Functional Map

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

### 36.3 Vertex-Interpolated Morphing

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

### 37. Geodesic Distance: Heat Method

*Audience: graphics application developers, systems developers.*

The heat method (Crane et al. 2013, 2017) computes geodesic distances on a surface mesh by: (1) solving the heat equation for a short time from source vertices; (2) computing the gradient field of the heat solution; (3) solving a Poisson equation on the gradient field to recover the distance function. All three steps are linear solves — GPU-parallelizable via iterative methods or direct sparse factorization.

### 37.1 Heat Diffusion Step

Solve (I + t·L)u = u₀ once, where L is the cotangent Laplacian, t is the timestep, and u₀ is 1 at sources and 0 elsewhere. Use conjugate gradient on the GPU (Jacobi preconditioned, §87.2 pattern):

```glsl
// heat_diffuse.comp — one Gauss-Seidel / Jacobi iteration for (I + tL)u = u0
layout(set=0,binding=0) readonly buffer Laplacian { float L_val[]; uint L_col[]; uint L_row[]; };
layout(set=0,binding=1) readonly buffer U_in { float u_in[]; };
layout(set=0,binding=2) writeonly buffer U_out { float u_out[]; };
layout(push_constant) uniform PC { float t; float dt_inv; } pc;  // t = heat timestep
layout(local_size_x=64) in;
void main() {
    uint v   = gl_GlobalInvocationID.x;
    float diag = 1.0;
    float rhs  = u_in[v];  // u₀ at source, 0 elsewhere (set before first iter)
    float Lv   = 0.0;
    for (uint e=L_row[v]; e<L_row[v+1]; e++) {
        uint nb = L_col[e];
        float w = L_val[e];
        Lv   += w * u_in[nb];
        diag += pc.t * (-w);  // diagonal: 1 + t * sum(weights)
    }
    u_out[v] = (rhs - pc.t * Lv) / diag;
}
```

[Source: Crane et al. "Geodesics in Heat: A New Approach to Computing Distance Based on Heat Flow", ACM TOG 2013]

### 37.2 Gradient Field Computation

After convergence of the heat solve, compute the normalised gradient field X = -∇u / |∇u| per face:

```glsl
// heat_gradient.comp — gradient of heat solution per triangle face
layout(set=0,binding=0) readonly buffer U { float u[]; };
layout(set=0,binding=1) readonly buffer Verts { vec3 verts[]; };
layout(set=0,binding=2) readonly buffer Faces { uvec3 faces[]; };
layout(set=0,binding=3) writeonly buffer Grad { vec3 X[]; };
layout(local_size_x=64) in;
void main() {
    uint f   = gl_GlobalInvocationID.x;
    uvec3 vi = faces[f];
    vec3  v0 = verts[vi.x], v1 = verts[vi.y], v2 = verts[vi.z];
    float u0 = u[vi.x],     u1 = u[vi.y],     u2 = u[vi.z];
    vec3  n  = cross(v1-v0, v2-v0);
    float A  = length(n) * 0.5;  // triangle area
    vec3  gu = (u0 * cross(n, v2-v1) + u1 * cross(n, v0-v2) + u2 * cross(n, v1-v0)) / (2.0*A*A);
    X[f]     = -normalize(gu);
}
```

### 37.3 Poisson Solve for Distance

Solve ∆φ = ∇·X where X is the gradient field from §37.2. The divergence of X becomes the right-hand side of a Poisson equation with the same cotangent Laplacian:

```glsl
// heat_divergence.comp — RHS: divergence of gradient field X at vertex v
layout(set=0,binding=0) readonly buffer Grad  { vec3 X[]; };
layout(set=0,binding=1) readonly buffer Verts { vec3 verts[]; };
layout(set=0,binding=2) readonly buffer FaceList { uint vFaces[]; uint vFaceStart[]; };
layout(set=0,binding=3) readonly buffer Cot   { float cotWeights[]; };
layout(set=0,binding=4) writeonly buffer Div  { float div[]; };
layout(local_size_x=64) in;
void main() {
    uint v  = gl_GlobalInvocationID.x;
    float d = 0.0;
    for (uint i=vFaceStart[v]; i<vFaceStart[v+1]; i++) {
        uint  f  = vFaces[i];
        // Contribution: (cot(α_ij)(X_f·e_ij) + cot(β_ij)(X_f·e_ij))/2 per edge
        d += cotWeights[i] * dot(X[f], vertexEdge(v, f));
    }
    div[v] = d;
}
// Then: solve L·phi = div using same Jacobi CG as §37.1
// Subtract phi[source] to get distance (phi is unique up to a constant)
```

[Source: Crane et al. 2013; implementation: https://github.com/nmwsharp/geometry-central]

### 37.4 Polyhedral Distance via Vector Heat Method

The vector heat method (Sharp et al. 2019) extends the heat method to parallel transport of vectors on surfaces — computing logarithmic maps, Voronoi diagrams on meshes, and mean curvature flow, all via GPU linear solves:

```glsl
// vector_heat.comp — transport a tangent vector along the surface (parallel transport)
// Replaces scalar u with a complex number c ∈ ℂ representing tangent vector direction
// Solve (I + t·L_c)c = c₀ where L_c is the complex connection Laplacian
// L_c[i,j] = -w_ij * exp(iθ_ij)  (θ = rotational offset at shared edge)
layout(set=0,binding=0) readonly buffer ConnLap { vec2 L_val[]; uint L_col[]; uint L_row[]; };
layout(set=0,binding=1) buffer C { vec2 c[]; };  // complex tangent vector field
layout(local_size_x=64) in;
void main() {
    uint v = gl_GlobalInvocationID.x;
    vec2 rhs = c[v], Lv = vec2(0.0); float diag = 1.0;
    for (uint e=L_row[v]; e<L_row[v+1]; e++) {
        vec2 w   = L_val[e];  // complex weight (rotated)
        vec2 cn  = c[L_col[e]];
        Lv   += vec2(w.x*cn.x-w.y*cn.y, w.x*cn.y+w.y*cn.x);
        diag += length(w);
    }
    c[v] = (rhs - Lv) / diag;
}
```

[Source: Sharp et al. "The Vector Heat Method", ACM TOG 2019]

---

### 38. Spectral Mesh Processing

*Audience: systems developers, graphics application developers.*

Spectral geometry processing analyses meshes via the eigenvectors and eigenvalues of the cotangent Laplacian — the manifold analogue of Fourier analysis. Low-frequency eigenvectors capture global shape; high-frequency ones capture fine detail. GPU applications include shape-preserving deformation, mesh segmentation, and shape descriptor computation for registration (§87 ICP complement).

### 38.1 Laplacian Eigenvector Computation via Power Iteration

For the k smallest eigenvalues of the cotangent Laplacian L, use the GPU-parallel power iteration (inverse iteration) with deflation:

```glsl
// power_iter.comp — one step of inverse power iteration for Laplacian eigenvector
layout(set=0,binding=0) readonly buffer Lc { float val[]; uint col[]; uint row[]; };
layout(set=0,binding=1) buffer Vec { vec3 v[]; };   // current eigenvector estimate
layout(set=0,binding=2) readonly buffer MassInv { float mInv[]; };  // 1/area per vertex
layout(local_size_x=64) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    float Lv= 0.0;
    for (uint e=row[i]; e<row[i+1]; e++)
        Lv += val[e] * v[col[e]].x;
    // Inverse iteration: solve (L - σM)v = Mv  via Jacobi step
    v[i].y = mInv[i] * (Lv - SIGMA * v[i].x);  // y = new estimate
}
// After convergence: Gram-Schmidt orthogonalisation against already-found eigenvectors
```

[Source: Vallet & Lévy "Spectral Geometry Processing with Manifold Harmonics", EG 2008; Taubin "A Signal Processing Approach" 1995]

### 38.2 Heat Kernel Signature (HKS)

HKS (Sun et al. 2009) is a shape descriptor invariant to isometric deformations, computed from the diagonal of the heat kernel K(x,x,t) = Σₖ exp(−λₖ t)φₖ(x)² for multiple time scales t:

```glsl
// hks.comp — compute HKS descriptor at each vertex from precomputed eigenvectors
layout(set=0,binding=0) readonly buffer Eigenvecs { float phi[K_EIGS][N_VERTS]; };
layout(set=0,binding=1) readonly buffer Eigenvals  { float lambda[K_EIGS]; };
layout(set=0,binding=2) writeonly buffer HKS { float hks[N_VERTS][N_TIMES]; };
layout(push_constant) uniform PC { float tMin; float tMax; int nTimes; } pc;
layout(local_size_x=64) in;
void main() {
    uint v  = gl_GlobalInvocationID.x;
    for (int ti=0; ti<pc.nTimes; ti++) {
        float t  = pc.tMin * pow(pc.tMax/pc.tMin, float(ti)/float(pc.nTimes-1));
        float hk = 0.0;
        for (int k=0; k<K_EIGS; k++)
            hk += exp(-lambda[k]*t) * phi[k][v] * phi[k][v];
        hks[v][ti] = hk;
    }
}
```

[Source: Sun et al. "A Concise and Provably Informative Multi-Scale Signature Based on Heat Diffusion", SGP 2009]

### 38.3 Spectral Shape Correspondence

Given HKS descriptors for two meshes, find correspondences by nearest-neighbour matching in descriptor space. GPU k-NN on the HKS feature matrix (N_VERTS × N_TIMES):

```glsl
// hks_match.comp — find nearest neighbour in HKS descriptor space
layout(set=0,binding=0) readonly buffer HKS_A { float hksA[N_A][N_TIMES]; };
layout(set=0,binding=1) readonly buffer HKS_B { float hksB[N_B][N_TIMES]; };
layout(set=0,binding=2) writeonly buffer Matches { uint match[N_A]; };
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    float bestD = 1e30; uint bestJ = 0u;
    for (uint j=0; j<N_B; j++) {
        float d = 0.0;
        for (int t=0; t<N_TIMES; t++) {
            float diff = hksA[i][t] - hksB[j][t];
            d += diff*diff;
        }
        if (d < bestD) { bestD=d; bestJ=j; }
    }
    match[i] = bestJ;
}
```

[Source: Ovsjanikov et al. "Functional Maps: A Flexible Representation of Maps Between Shapes", SIGGRAPH 2012]

### 38.4 Low-Pass Mesh Filtering via Spectral Truncation

A low-pass filter retains only the first K eigenvectors, suppressing high-frequency noise. This is the spectral equivalent of §19.1 Laplacian smoothing but with controllable frequency cutoff:

```glsl
// spectral_filter.comp — reconstruct mesh from truncated eigenvectors
layout(set=0,binding=0) readonly buffer Eigenvecs { float phi[K_EIGS][N_VERTS]; };
layout(set=0,binding=1) readonly buffer Coeffs    { vec3  c[K_EIGS]; };  // projection coeff per eigenvec
layout(set=0,binding=2) writeonly buffer PosOut   { vec3 posOut[]; };
layout(push_constant) uniform PC { int K; } pc;  // truncation (K < K_EIGS)
layout(local_size_x=64) in;
void main() {
    uint v  = gl_GlobalInvocationID.x;
    vec3 p  = vec3(0.0);
    for (int k=0; k<pc.K; k++)
        p += phi[k][v] * c[k];
    posOut[v] = p;
}
// Coeffs[k] = <pos, phi_k>_M (mass-weighted inner product) — computed in a prior pass
```

[Source: Vallet & Lévy 2008; Karni & Gotsman "Spectral Compression of Mesh Geometry", SIGGRAPH 2000]

---


---

## V. Animation, Skinning, and Deformation

Character animation on the GPU covers the full pipeline from skeletal joint transforms through vertex deformation to secondary motion dynamics. Linear blend skinning (LBS) and its quality improvements (dual-quaternion skinning, corrective blendshapes) drive the base mesh; IK solvers (CCD, FABRIK, Jacobian) compute joint angles from end-effector constraints in compute shaders; cage-based methods (Mean Value Coordinates, Green Coordinates) and Laplacian mesh editing provide smooth artist-controlled shape deformation; and projective dynamics and FEM handle soft-body secondary motion. Hair strand simulation and secondary dynamics add organic motion on top of the primary skeleton.

### 39. Skeletal Animation: Skinning

Skeletal animation is the standard character deformation pipeline: a hierarchy of bones drives the skin mesh through linear blend skinning (LBS) or dual-quaternion skinning (DQS) evaluated in a vertex or compute shader. Each vertex stores up to eight (bone index, weight) pairs; the shader fetches each bone's current world-space transform from a joint-matrix buffer, blends the weighted transforms, and applies the result to the rest-pose vertex position and normal. The skinned output is consumed unchanged by subsequent rendering passes — shadow maps, depth pre-pass, and collision queries — making the skinning dispatch a natural compute pre-pass that writes GPU-resident vertex buffers rather than a fixed-function vertex-shader stage.

### 39.1 Forward Kinematics on GPU

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

### 39.2 Linear Blend Skinning (LBS)

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

### 39.3 Dual Quaternion Skinning (DQS)

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

### 39.4 GPU Compute Skinning Pre-Pass

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

### 39.5 Blend Shapes (Morph Targets)

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

### 39.6 PBD Cloth and Soft-Body Simulation

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

### 40. Inverse Kinematics on the GPU

IK solvers within a single chain are inherently sequential, but multiple independent chains (different characters, tentacles, ragdoll limbs) parallelize trivially: one workgroup per chain.

### 40.1 CCD: Cyclic Coordinate Descent

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

### 40.2 FABRIK: Forward and Backward Reaching IK

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

### 40.3 Jacobian-Based IK and Damped Least Squares

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

### 41. Shape Deformation Beyond Skinning

*Audience: graphics application developers.*

Skeletal skinning (§39) deforms a mesh by blending bone transforms. More general deformation methods impose different structural constraints: ARAP preserves local rigidity cell-by-cell, cage deformation provides artistic control via a coarse proxy, and RBF deformation scatters influence from a sparse set of handles.

### 41.1 As-Rigid-As-Possible (ARAP) Deformation

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

### 41.2 Cage-Based Deformation: Mean Value Coordinates

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

### 41.3 Radial Basis Function Deformation

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

### 42. Hair and Strand Geometry

*Audience: graphics application developers.*

Hair simulation and rendering sit at the intersection of geometry generation, physics simulation, and LOD management. Unlike mesh-based geometry, strand geometry is one-dimensional (a piecewise curve) but must interact with a 3D collision surface, self-collide at high density, and transition between strand, shell (hair card), and volume representations across LOD levels.

### 42.1 Strand Representation and Cosserat Rods

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

### 42.2 Bending and Torsion Constraints

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

### 42.3 Strand Self-Collision via Spatial Hashing

At hair densities of 50–200k strands, brute-force self-collision is infeasible. Uniform grid hashing (§27.2 pattern) detects strand segment pairs within a collision radius:

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

### 42.4 LOD: Strand-to-Shell-to-Volume Transitions

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

### 43. Secondary Animation Dynamics

*Audience: graphics application developers.*

Primary animation (§39) drives bones via forward kinematics or motion capture. Secondary dynamics add physical plausibility to appendages — hair, clothing tails, antennas, cheeks — without full simulation. The three algorithms here all run as a per-bone post-pass on the GPU after the main FK/IK solve.

### 43.1 Jiggle Bones: Per-Bone Damped Spring

A jiggle bone tracks a target position (animated joint position) with a damped spring. The spring mass follows a delayed, oscillating trajectory:

```glsl
// jiggle.comp
struct JiggleBone { vec3 pos; vec3 vel; float stiffness; float damping; float mass; };
layout(set=0,binding=0) buffer Jiggles  { JiggleBone jb[]; };
layout(set=0,binding=1) readonly buffer Targets { vec3 target[]; };
layout(push_constant) uniform PC { float dt; } pc;
layout(local_size_x=64) in;
void main() {
    uint i   = gl_GlobalInvocationID.x;
    vec3 err = target[i] - jb[i].pos;
    vec3 acc = (err * jb[i].stiffness - jb[i].vel * jb[i].damping) / jb[i].mass;
    jb[i].vel += acc * pc.dt;
    jb[i].pos += jb[i].vel * pc.dt;
}
```

The resulting world position replaces the animated bone position before skinning (§39.4), propagating jiggle through the bone hierarchy. [Source: Lander "Oh My God, I Inverted Kine!", Game Developer Magazine 1998]

### 43.2 Volume-Preserving Squash and Stretch

Scaling a bone along its primary axis by s requires compensating the perpendicular axes by 1/√s to conserve volume. Computed per-bone after FK:

```glsl
// squash_stretch.comp
layout(set=0,binding=0) buffer BoneMats { mat4 boneMat[]; };
layout(push_constant) uniform PC { uint boneIdx; float stretchAxis; } pc;
layout(local_size_x=1) in;
void main() {
    mat4 m    = boneMat[pc.boneIdx];
    float s   = length(m[int(pc.stretchAxis)].xyz);  // current length along stretch axis
    float inv = 1.0 / sqrt(max(s, 1e-4));
    // scale perpendicular axes
    uint a1 = (uint(pc.stretchAxis)+1)%3;
    uint a2 = (uint(pc.stretchAxis)+2)%3;
    m[a1].xyz *= inv;
    m[a2].xyz *= inv;
    boneMat[pc.boneIdx] = m;
}
```

### 43.3 Pose-Space Corrective Blend Shapes

Corrective shapes fix skinning artefacts (candy-wrapping at elbows, volume collapse at shoulders) by activating blend shapes driven by joint angle. Each corrective is a blend shape (§39.5) with weight interpolated by RBF over pose space:

```glsl
// corrective_rbf.comp — evaluate corrective weights at current pose
layout(set=0,binding=0) readonly buffer RBFCenters { vec3 centers[]; };  // training poses
layout(set=0,binding=1) readonly buffer RBFWeights { float weights[];  };  // RBF coefficients
layout(set=0,binding=2) readonly buffer CurrPose   { vec3 pose;         };  // current joint angles
layout(set=0,binding=3) writeonly buffer BlendW    { float blendWeight[]; };
layout(local_size_x=64) in;
void main() {
    uint c   = gl_GlobalInvocationID.x;
    float r  = length(pose - centers[c]);
    float phi = r * r * log(max(r, 1e-6));  // thin-plate spline RBF
    blendWeight[c] = weights[c] * phi;
}
// Sum blendWeight[] → corrective blend shape activation scalar
// Then apply blend shape displacement (§39.5 kernel)
```

[Source: Lewis et al. "Pose Space Deformation", SIGGRAPH 2000; Sloan et al. "Shape by Example", I3D 2001]

---

### 44. Skeletal Mesh Retargeting

*Audience: graphics application developers.*

Retargeting transfers skeletal animation from a source skeleton (e.g., a motion-capture actor) to a target skeleton (a game character) with different bone lengths, proportions, and topology. GPU-accelerated retargeting enables real-time animation remapping for procedural character diversity.

### 44.1 Skeleton Normalization

Normalize both source and target skeletons to a height-1 T-pose by dividing each bone's rest-pose length by the total skeleton height. Store per-bone scale factors:

```glsl
// skel_normalize.comp — compute per-bone length ratios src/tgt
layout(set=0,binding=0) readonly buffer SrcRest { mat4 srcBones[]; };
layout(set=0,binding=1) readonly buffer TgtRest { mat4 tgtBones[]; };
layout(set=0,binding=2) writeonly buffer Ratio   { float ratio[];   };
layout(push_constant) uniform PC { uint N; float srcHeight; float tgtHeight; } pc;
layout(local_size_x=64) in;
void main() {
    uint b    = gl_GlobalInvocationID.x;
    float sLen = length(srcBones[b][3].xyz - srcBones[parentOf(b)][3].xyz) / pc.srcHeight;
    float tLen = length(tgtBones[b][3].xyz - tgtBones[parentOf(b)][3].xyz) / pc.tgtHeight;
    ratio[b]   = (tLen > 1e-5) ? sLen / tLen : 1.0;
}
```

### 44.2 Rotation Retargeting via Quaternion Mapping

Copy source joint rotations to target joints (by name match or user-defined mapping), scale translation components by the bone length ratio:

```glsl
// retarget.comp — apply source animation to target skeleton
layout(set=0,binding=0) readonly buffer SrcPose { Quaternion srcQ[]; vec3 srcT[]; };
layout(set=0,binding=1) readonly buffer Ratio   { float ratio[]; };
layout(set=0,binding=2) readonly buffer SrcToTgt{ int mapping[]; };  // src bone → tgt bone (-1=none)
layout(set=0,binding=3) buffer TgtPose { Quaternion tgtQ[]; vec3 tgtT[]; };
layout(local_size_x=64) in;
void main() {
    uint s  = gl_GlobalInvocationID.x;
    int  t  = mapping[s];
    if (t < 0) return;
    tgtQ[t] = srcQ[s];                // rotation transfers directly
    tgtT[t] = srcT[s] * ratio[t];    // translation scaled by bone ratio
}
```

[Source: Gleicher "Retargetting Motion to New Characters", SIGGRAPH 1998; Monzani et al. "Using an Intermediate Skeleton and Inverse Kinematics for Motion Retargeting", EG 2000]

### 44.3 Foot Planting Constraint

After rotation retargeting, the character's feet may float above or sink below the target's terrain. Project foot effectors to the terrain surface via IK (§40 FABRIK) with a constraint that the foot contact point remains on the terrain SDF (§64.3):

```glsl
// foot_plant.comp — pull foot to terrain surface using SDF query + FABRIK
layout(local_size_x=2) in;  // left and right foot in parallel
void main() {
    uint foot    = gl_LocalInvocationID.x;
    vec3 ankle   = worldPos[ankleJoint[foot]];
    float phi    = terrainSDF(ankle);
    if (phi < 0.0) {
        vec3 target = ankle - terrainNormal(ankle) * phi;
        // FABRIK solve from ankle to target (§40.2)
        fabrik(ankleJoint[foot], target, MAX_IK_ITER);
    }
}
```

[Source: Lander "Making Kine More Flexible", Game Developer Magazine 1998; Kovar et al. "Motion Graphs", SIGGRAPH 2002]

### 44.4 Procedural Diversity via Pose Blending

Generate a crowd of diverse characters from one animation by blending retargeted poses with procedurally varied joint angles (noise-driven secondary motion §43.1, body proportions via ratio[], gesture libraries):

```glsl
// pose_blend.comp — blend retargeted base pose with per-character variation
layout(set=0,binding=0) readonly buffer BasePose { Quaternion base[N_BONES]; };
layout(set=0,binding=1) readonly buffer VarPose  { Quaternion var[N_BONES * N_CHARS]; };
layout(set=0,binding=2) buffer OutPose  { Quaternion out[N_BONES * N_CHARS]; };
layout(push_constant) uniform PC { float varWeight; } pc;
layout(local_size_x=64) in;
void main() {
    uint bi = gl_GlobalInvocationID.x % N_BONES;
    uint ci = gl_GlobalInvocationID.x / N_BONES;
    out[ci*N_BONES+bi] = slerpQ(base[bi], var[ci*N_BONES+bi], pc.varWeight);
}
```

[Source: Hecker et al. "Real-Time Motion Retargeting to Highly Varied User-Created Morphologies", SIGGRAPH 2008]

---

### 45. Soft Body FEM and Projective Dynamics

*Audience: systems developers, graphics application developers.*

Finite element method (FEM) soft body simulation deforms a tetrahedral mesh under elastic forces derived from a continuum mechanics energy functional. Projective dynamics (Bouaziz et al. 2014) reformulates FEM as a sequence of local projections plus a global linear solve, enabling GPU parallelism across both steps. This is distinct from XPBD cloth (§54), which applies position constraints on surface triangles; FEM/PD operates on volumetric tetrahedra with material parameters (Young's modulus, Poisson ratio).

### 45.1 Tetrahedral Mesh Construction from Surface

Convert a watertight surface mesh (§22) to a tetrahedral volume mesh using constrained Delaunay tetrahedralization. The GPU accelerates the initial point insertion; CPU TetGen handles the constrained phase:

```glsl
// tet_classify.comp — classify surface samples as inside/outside via ray query
layout(set=0,binding=0) readonly buffer SurfPts { vec3 pts[]; };
layout(set=0,binding=1) uniform accelerationStructureEXT surfaceTLAS;
layout(set=0,binding=2) writeonly buffer Inside { uint inside[]; };
layout(local_size_x=64) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    vec3 p  = pts[i];
    // Count intersections along +X ray; odd = inside
    rayQueryEXT rq;
    uint hits = 0u;
    rayQueryInitializeEXT(rq, surfaceTLAS, gl_RayFlagsNoneEXT, 0xFF, p, 1e-4, vec3(1,0,0), 1e9);
    while (rayQueryProceedEXT(rq))
        if (rayQueryGetIntersectionTypeEXT(rq,false)==gl_RayQueryCandidateIntersectionTriangleEXT)
            { hits++; rayQueryConfirmIntersectionEXT(rq); }
    inside[i] = hits % 2u;
}
```

[Source: Si "TetGen: A Delaunay-Based Quality Tetrahedral Mesh Generator", ACM TOMS 2015]

### 45.2 Deformation Gradient and Elastic Force

For each tetrahedron, compute the deformation gradient F = DS·Dm⁻¹ (D = deformed shape matrix, Dm = rest-shape matrix), then derive the elastic force from the corotational or neo-Hookean energy:

```glsl
// fem_elastic_force.comp — corotational FEM: compute nodal forces per tet
layout(set=0,binding=0) readonly buffer RestPos  { vec3 rest[];  };
layout(set=0,binding=1) readonly buffer CurPos   { vec3 cur[];   };
layout(set=0,binding=2) readonly buffer Tets     { uvec4 tets[]; };
layout(set=0,binding=3) readonly buffer DmInv    { mat3 DmInv[]; };  // precomputed rest inverse
layout(set=0,binding=4) readonly buffer Material { float mu[]; float lambda[]; };
layout(set=0,binding=5) buffer Forces { vec3 f[]; };  // accumulated per vertex
layout(local_size_x=64) in;
void main() {
    uint t   = gl_GlobalInvocationID.x;
    uvec4 vi = tets[t];
    // Deformed shape matrix
    mat3 Ds  = mat3(cur[vi.y]-cur[vi.x], cur[vi.z]-cur[vi.x], cur[vi.w]-cur[vi.x]);
    mat3 F   = Ds * DmInv[t];
    // Corotational: polar decompose F = R*S, compute stress from S
    mat3 R, S; polarDecompose(F, R, S);
    mat3 E   = S - mat3(1.0);  // Green strain (linearised)
    float trE= E[0][0]+E[1][1]+E[2][2];
    mat3 P   = R * (2.0*mu[t]*E + lambda[t]*trE*mat3(1.0));  // 1st Piola-Kirchhoff
    // Nodal force: f_i = -P * DmInv^T * volume (scattered to vertices)
    mat3 H   = -P * transpose(DmInv[t]) * tetVolume(rest, vi);
    atomicAdd_vec3(f[vi.x], -(H[0]+H[1]+H[2]));
    atomicAdd_vec3(f[vi.y],   H[0]);
    atomicAdd_vec3(f[vi.z],   H[1]);
    atomicAdd_vec3(f[vi.w],   H[2]);
}
```

[Source: Sifakis & Barbic "FEM Simulation of 3D Deformable Solids", SIGGRAPH Course 2012; Irving et al. "Invertible Finite Elements for Robust Simulation of Large Deformation", SCA 2004]

### 45.3 Projective Dynamics: Local Projection Step

Projective dynamics (Bouaziz et al. 2014) replaces the nonlinear force evaluation with a local projection: for each constraint (tetrahedral volume, edge length, etc.), project the current positions onto the constraint manifold. Massively parallel across constraints:

```glsl
// pd_local_project.comp — PD local step: project tetrahedra onto volume constraint
layout(set=0,binding=0) readonly buffer Pos      { vec3 pos[];  };
layout(set=0,binding=1) readonly buffer Tets     { uvec4 tets[]; };
layout(set=0,binding=2) readonly buffer RestVol  { float rv[];   };
layout(set=0,binding=3) writeonly buffer Aux     { vec3 aux[];   };  // auxiliary positions p_i
layout(local_size_x=64) in;
void main() {
    uint t   = gl_GlobalInvocationID.x;
    uvec4 vi = tets[t];
    vec3  p[4]; for(int k=0;k<4;k++) p[k] = pos[vi[k]];
    // Current volume: V = dot(p1-p0, cross(p2-p0, p3-p0)) / 6
    float V  = dot(p[1]-p[0], cross(p[2]-p[0], p[3]-p[0])) / 6.0;
    // Project: scale positions uniformly to restore rest volume
    float scale = cbrt(rv[t] / max(abs(V), 1e-8));
    vec3 c = (p[0]+p[1]+p[2]+p[3])*0.25;
    for (int k=0;k<4;k++) aux[t*4+k] = c + scale*(p[k]-c);
}
```

[Source: Bouaziz et al. "Projective Dynamics: Fusing Constraint Projections for Fast Simulation", SIGGRAPH 2014]

### 45.4 Projective Dynamics: Global Linear Solve

The global step solves a fixed sparse linear system (the system matrix factors once at setup) for each iteration. Use a pre-factored Cholesky solve — compute the right-hand side on GPU, solve on CPU or GPU with cuSolver:

```glsl
// pd_global_rhs.comp — assemble RHS for PD global step
layout(set=0,binding=0) readonly buffer Aux      { vec3 aux[];  };   // from local step
layout(set=0,binding=1) readonly buffer Mass     { float m[];   };
layout(set=0,binding=2) readonly buffer PrevPos  { vec3 s[];    };   // inertial target s = q + h²M⁻¹f_ext
layout(set=0,binding=3) buffer RHS { vec3 rhs[]; };
layout(push_constant) uniform PC { float h2; uint N_VERTS; } pc;  // h²
layout(local_size_x=64) in;
void main() {
    uint v   = gl_GlobalInvocationID.x;
    if (v >= pc.N_VERTS) return;
    // RHS_i = M_i/h² * s_i + sum_{constraints c containing i} w_c * p_c
    // Constraint auxiliary sums accumulated separately via atomicAdd
    rhs[v]  = m[v]/pc.h2 * s[v];  // inertial term
    // Constraint terms added by a second pass over constraints
}
// Then solve: (M/h² + sum w_c A_c^T A_c) * q_{n+1} = rhs
// System matrix is constant → prefactor Cholesky once at startup
```

[Source: Bouaziz et al. 2014; Liu et al. "Fast Simulation of Mass-Spring Systems", SIGGRAPH Asia 2013]

---

### 46. Cage-Based Deformation: Mean Value and Green Coordinates

*Audience: graphics application developers.*

Cage-based deformation encloses a detailed mesh inside a coarse control cage. Moving cage vertices deforms the interior mesh via barycentric-like weights computed from the cage. Mean value coordinates (Floater 2003; Ju et al. 2005) and Green coordinates (Lipman et al. 2008) provide smooth, volume-preserving deformation. GPU precomputes per-vertex weights at bind time; runtime deformation is a weighted sum — embarrassingly parallel.

### 46.1 Mean Value Coordinate Weight Computation (Bind)

For each interior vertex v, integrate the mean value weights w_j(v) over each cage triangle by summing spherical wedge angles:

```glsl
// mvc_bind.comp — compute mean value coordinates for each mesh vertex wrt cage
layout(set=0,binding=0) readonly buffer MeshPos  { vec3 mesh[];  };
layout(set=0,binding=1) readonly buffer CagePos  { vec3 cage[];  };
layout(set=0,binding=2) readonly buffer CageTris { uvec3 ctri[]; };
layout(set=0,binding=3) writeonly buffer Weights { float w[];    };  // [N_MESH][N_CAGE]
layout(push_constant) uniform PC { uint N_MESH; uint N_CAGE; uint N_TRI; } pc;
layout(local_size_x=64) in;
void main() {
    uint v   = gl_GlobalInvocationID.x;
    if (v >= pc.N_MESH) return;
    vec3 p   = mesh[v];
    float wSum = 0.0;
    // Accumulate mean value weight for each cage vertex
    for (uint t=0; t<pc.N_TRI; t++) {
        uvec3 ci = ctri[t];
        vec3 u[3]; float l[3];
        for(int k=0;k<3;k++){ u[k]=normalize(cage[ci[k]]-p); l[k]=length(cage[ci[k]]-p); }
        // Spherical triangle contribution (Ju et al. 2005 §3)
        float theta[3];
        theta[0] = 2.0*asin(length(u[1]-u[2])*0.5);
        theta[1] = 2.0*asin(length(u[0]-u[2])*0.5);
        theta[2] = 2.0*asin(length(u[0]-u[1])*0.5);
        float h  = (theta[0]+theta[1]+theta[2])*0.5;
        if (abs(PI-h)<1e-5) continue;  // on triangle plane
        float c[3]; for(int k=0;k<3;k++) c[k]=2.0*sin(h)*sin(h-theta[k])/sin(theta[(k+1)%3])/sin(theta[(k+2)%3])-1.0;
        float sign_ = (determinant(mat3(u[0],u[1],u[2]))<0)?-1.0:1.0;
        float d[3]; for(int k=0;k<3;k++) d[k]=sign_*sqrt(1.0-c[k]*c[k]);
        for(int k=0;k<3;k++){ float wi=(theta[k]-c[(k+2)%3]*theta[(k+1)%3]-c[(k+1)%3]*theta[(k+2)%3])/(l[k]*sin(theta[(k+1)%3])*d[(k+2)%3]); w[v*pc.N_CAGE+ci[k]]+=wi; wSum+=wi; }
    }
    // Normalise
    for(uint c=0;c<pc.N_CAGE;c++) w[v*pc.N_CAGE+c] /= wSum;
}
```

[Source: Ju et al. "Mean Value Coordinates for Closed Triangular Meshes", SIGGRAPH 2005; Floater "Mean Value Coordinates", CAGD 2003]

### 46.2 Runtime MVC Deformation

With precomputed weights, deform each mesh vertex as a weighted sum of displaced cage vertices — trivially parallel:

```glsl
// mvc_deform.comp — apply cage deformation to mesh vertices
layout(set=0,binding=0) readonly buffer Weights    { float w[];     };  // [N_MESH][N_CAGE]
layout(set=0,binding=1) readonly buffer CagePos    { vec3 cage[];   };  // deformed cage
layout(set=0,binding=2) writeonly buffer MeshPosOut { vec3 mout[];  };
layout(push_constant) uniform PC { uint N_MESH; uint N_CAGE; } pc;
layout(local_size_x=64) in;
void main() {
    uint v  = gl_GlobalInvocationID.x;
    if (v >= pc.N_MESH) return;
    vec3 p  = vec3(0.0);
    for (uint c=0; c<pc.N_CAGE; c++) p += w[v*pc.N_CAGE+c] * cage[c];
    mout[v] = p;
}
```

### 46.3 Green Coordinates (Conformal Cage)

Green coordinates (Lipman et al. 2008) add normal-scaled terms to MVC so that the deformation is approximately conformal — preserving local shape under cage rotation:

```glsl
// green_deform.comp — Green coordinate deformation with normal scaling terms
layout(set=0,binding=0) readonly buffer PhiWeights { float phi[]; };   // vertex weights [N_MESH][N_CAGE]
layout(set=0,binding=1) readonly buffer PsiWeights { float psi[]; };   // normal weights [N_MESH][N_TRI]
layout(set=0,binding=2) readonly buffer CagePos    { vec3 cage[];  };
layout(set=0,binding=3) readonly buffer CageNorm   { vec3 cnor[];  };  // deformed cage face normals
layout(set=0,binding=4) writeonly buffer MeshOut   { vec3 mout[];  };
layout(push_constant) uniform PC { uint N_MESH; uint N_CAGE; uint N_TRI; } pc;
layout(local_size_x=64) in;
void main() {
    uint v  = gl_GlobalInvocationID.x;
    if (v >= pc.N_MESH) return;
    vec3 p  = vec3(0.0);
    for (uint c=0; c<pc.N_CAGE; c++) p += phi[v*pc.N_CAGE+c] * cage[c];
    for (uint t=0; t<pc.N_TRI;  t++) p += psi[v*pc.N_TRI+t]  * cnor[t];
    mout[v] = p;
}
```

[Source: Lipman et al. "Green Coordinates", SIGGRAPH 2008; Ben-Chen et al. "Variational Harmonic Maps for Space Deformation", SIGGRAPH 2009]

### 46.4 Cage Editing and Undo via Differential Coordinates

Store the mesh deformation in differential (Laplacian) coordinates (§47) relative to the cage transform so that cage edits propagate smoothly and undo is cheap:

```glsl
// cage_differential.comp — convert mesh positions to cage-relative Laplacian coords
layout(set=0,binding=0) readonly buffer MeshPos  { vec3 mesh[];  };
layout(set=0,binding=1) readonly buffer MeshAdj  { uint adj[]; uint adjStart[]; };
layout(set=0,binding=2) readonly buffer CageXform { mat4 xform; };  // cage rigid transform
layout(set=0,binding=3) writeonly buffer Delta    { vec3 delta[]; };
layout(local_size_x=64) in;
void main() {
    uint v   = gl_GlobalInvocationID.x;
    vec3 p   = (inverse(xform)*vec4(mesh[v],1.0)).xyz;  // cage-local position
    vec3 lap = vec3(0.0); uint deg = adjStart[v+1]-adjStart[v];
    for (uint e=adjStart[v];e<adjStart[v+1];e++)
        lap += (inverse(xform)*vec4(mesh[adjStart[e]],1.0)).xyz;
    delta[v] = p - lap/float(deg);  // Laplacian coordinate in cage-local frame
}
```

[Source: Sorkine et al. "Laplacian Surface Editing", SGP 2004 §102; Botsch & Sorkine "On Linear Variational Surface Deformation Methods", IEEE TVCG 2008]

---

### 47. Laplacian Mesh Editing and Interactive Deformation

*Audience: graphics application developers.*

Laplacian mesh editing (Sorkine et al. 2004) deforms a mesh by specifying a sparse set of handle constraints and solving a bilaplacian linear system to minimise changes in the Laplacian coordinates (differential coordinates that encode local shape). GPU compute accelerates the constraint-assembly and iterative CG solve; the system matrix factors once for a given handle topology.

### 47.1 Cotangent Laplacian Assembly

Assemble the cotangent-weighted Laplacian matrix L (§37.1 pattern) from the mesh connectivity. Store as a sparse CSR matrix:

```glsl
// lap_cotangent.comp — assemble cotangent weights for each half-edge
layout(set=0,binding=0) readonly buffer Pos       { vec3 pos[];  };
layout(set=0,binding=1) readonly buffer HalfEdge  { HalfEdgeDS he[]; };
layout(set=0,binding=2) writeonly buffer CotWeight { float cot[]; };  // per half-edge weight
layout(local_size_x=64) in;
void main() {
    uint e  = gl_GlobalInvocationID.x;
    // Cotangent of the angle opposite to edge e in its face
    uint vi = he[e].vert, vj = he[he[e].next].vert, vk = he[he[he[e].next].next].vert;
    vec3 a  = pos[vi]-pos[vk], b = pos[vj]-pos[vk];
    float cotA = dot(a,b) / max(length(cross(a,b)), 1e-8);
    cot[e]  = cotA * 0.5;
    // Symmetric: cot[twin(e)] added by the twin edge's pass
}
```

[Source: Sorkine et al. "Laplacian Surface Editing", SGP 2004; Desbrun et al. "Implicit Fairing of Irregular Meshes Using Diffusion and Curvature Flow", SIGGRAPH 1999]

### 47.2 Differential Coordinate Encoding

Encode each vertex as its Laplacian coordinate δᵢ = Lᵢ·p (the difference between a vertex and its weighted neighbours). Handle vertices are constrained; the solve minimises ‖Lp − δ‖² subject to handle positions:

```glsl
// lap_delta.comp — compute differential (Laplacian) coordinates per vertex
layout(set=0,binding=0) readonly buffer Pos       { vec3 pos[];     };
layout(set=0,binding=1) readonly buffer AdjV      { uint adjV[];    };
layout(set=0,binding=2) readonly buffer AdjW      { float adjW[];   };  // cotangent weights
layout(set=0,binding=3) readonly buffer AdjStart  { uint adjStart[]; };
layout(set=0,binding=4) writeonly buffer Delta    { vec3 delta[];   };
layout(local_size_x=64) in;
void main() {
    uint v   = gl_GlobalInvocationID.x;
    float wSum = 0.0; vec3 lap = vec3(0.0);
    for (uint e=adjStart[v]; e<adjStart[v+1]; e++) {
        lap  += adjW[e] * pos[adjV[e]];
        wSum += adjW[e];
    }
    delta[v] = pos[v] - (wSum>0.0 ? lap/wSum : vec3(0.0));
}
```

### 47.3 CG Solve for Handle-Constrained Deformation

With handles fixed, solve the augmented system [L; λI_handles]·p = [δ; λ·c_handles] via conjugate gradient. One CG iteration per dispatch:

```glsl
// lap_cg_iter.comp — one CG iteration for Laplacian deformation solve
layout(set=0,binding=0) readonly buffer A_val  { float Av[]; };   // sparse L^T L + λ I_handles
layout(set=0,binding=1) readonly buffer A_col  { uint  Ac[]; };
layout(set=0,binding=2) readonly buffer A_row  { uint  Ar[]; };
layout(set=0,binding=3) buffer X    { vec3 x[]; };    // current solution
layout(set=0,binding=4) buffer R    { vec3 r[]; };    // residual
layout(set=0,binding=5) buffer P    { vec3 p[]; };    // search direction
layout(set=0,binding=6) buffer Ap   { vec3 ap[]; };   // A·p
layout(set=0,binding=7) buffer Alpha_beta { float alpha; float beta; float rr_old; float rr_new; };
layout(local_size_x=64) in;
void main() {
    uint v   = gl_GlobalInvocationID.x;
    // Ap = A * p (sparse matrix-vector multiply)
    vec3 Apv = vec3(0.0);
    for (uint e=Ar[v]; e<Ar[v+1]; e++) Apv += Av[e] * p[Ac[e]];
    ap[v] = Apv;
    // alpha, beta computed by CPU reduction between dispatches; apply x, r updates here
    x[v] += alpha * p[v];
    r[v] -= alpha * ap[v];
}
// CPU: compute rr_new = sum |r|², beta = rr_new/rr_old, update p: p = r + beta*p; repeat ~50 iters
```

[Source: Sorkine et al. 2004; Botsch & Sorkine 2008; Shewchuk "An Introduction to the Conjugate Gradient Method Without the Agonizing Pain", 1994]

### 47.4 As-Rigid-As-Possible (ARAP) Surface Modelling

ARAP (Sorkine & Alexa 2007) alternates between a local step (fit a rigid rotation to each vertex neighbourhood) and a global Laplacian solve. The local step is parallel; the global step reuses the same prefactored system as §47.3:

```glsl
// arap_local.comp — ARAP local step: fit rotation Rᵢ to each vertex neighbourhood
layout(set=0,binding=0) readonly buffer RestPos { vec3 rest[];  };
layout(set=0,binding=1) readonly buffer CurPos  { vec3 cur[];   };
layout(set=0,binding=2) readonly buffer AdjV    { uint adjV[];  };
layout(set=0,binding=3) readonly buffer AdjW    { float adjW[]; };
layout(set=0,binding=4) readonly buffer AdjStart{ uint adjStart[]; };
layout(set=0,binding=5) writeonly buffer Rotations { mat3 R[];  };
layout(local_size_x=64) in;
void main() {
    uint v   = gl_GlobalInvocationID.x;
    // Covariance: S = sum_j w_ij (rest_i - rest_j)^T (cur_i - cur_j)
    mat3 S   = mat3(0.0);
    for (uint e=adjStart[v]; e<adjStart[v+1]; e++) {
        uint j = adjV[e]; float w = adjW[e];
        vec3 dr= rest[v]-rest[j], dc = cur[v]-cur[j];
        S += w * outerProduct(dr, dc);
    }
    // SVD of S → R = V U^T
    mat3 U,V; vec3 sigma; svd3x3(S, U, sigma, V);
    R[v] = V * transpose(U);
}
// Global step: RHS_i = sum_j w_ij/2 * (Ri+Rj)*(rest_i-rest_j); solve same Laplacian system
```

[Source: Sorkine & Alexa "As-Rigid-As-Possible Surface Modeling", SGP 2007; §66.3 ARAP UV parameterization uses the same local step]

---


---

## VI. Physics and Simulation

GPU simulation covers the full range of physical phenomena relevant to real-time and offline rendering: rigid body dynamics with parallel broad-phase collision detection (BVH, spatial hashing), continuous collision detection for fast-moving objects, fluid simulation via SPH particle methods and Eulerian pressure-projection grids, position-based and projective dynamics for cloth and soft bodies, the Discrete Element Method (DEM) for granular materials, and fracture/destruction via Voronoi decomposition. The Minkowski sum and swept volumes appear here because they are the geometric primitives underlying GJK and EPA collision algorithms. All systems run as compute dispatches that write updated vertex positions consumed by the rendering pipeline each frame.

### 48. Fluid Simulation on the GPU

*Audience: graphics application developers, systems developers.*

Fluid simulation divides into two major families: **Eulerian** (fixed grid, track field values at cell centers/faces) and **Lagrangian** (particles follow the fluid). Hybrid schemes (FLIP, APIC) combine both. GPU implementations exploit the data-parallelism in per-cell and per-particle updates, with the bottleneck usually being the pressure Poisson solve.

### 48.1 MAC Grid and Semi-Lagrangian Advection

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

### 48.2 Pressure Projection via Conjugate Gradient

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

### 48.3 Position-Based Fluids (PBF) on GPU

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

### 48.4 FLIP: Hybrid Particle-Grid

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

### 48.5 Lattice Boltzmann Method (LBM)

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

### 49. Deformable Bodies and Continuum Mechanics

*Audience: graphics application developers, systems developers.*

PBD cloth (§39.6) handles thin surfaces via distance and bending constraints. Volumetric deformable bodies require a continuum mechanics formulation — FEM for stiff materials, MPM for materials that fracture, flow, or undergo large deformation. Both decompose naturally into per-element parallel local steps and a global linear solve.

### 49.1 Mass-Spring Systems on GPU

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

### 49.2 Corotational FEM on GPU

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

### 49.3 Material Point Method (MPM) on GPU

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

### 50. Rigid Body Dynamics on GPU

*Audience: graphics application developers, systems developers.*

PhysX 5 (§42 Library Landscape Table B) provides a production GPU rigid body solver; this section covers the algorithms so implementors can build Vulkan-native physics or understand what PhysX does under the hood.

### 50.1 Broadphase: SAP Pair Generation

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

Pairs feed the island detection and narrowphase (§24.4 SAT/GJK). [Source: Tonge "Iterative Rigid Body Simulation", GDC 2013]

### 50.2 Island Detection via GPU Label Propagation

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

### 50.3 Sequential Impulse Solver with Graph Colouring

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

### 50.4 Featherstone Articulated Body Algorithm

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

### 51. Crowd and Multi-Agent Simulation

*Audience: graphics application developers.*

Navigation meshes (§70) compute flow fields telling each agent which direction to move. This section covers the per-agent algorithms that consume those fields and avoid collisions with other agents at run time.

### 51.1 ORCA: Optimal Reciprocal Collision Avoidance

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

    // iterate nearby agents via spatial hash (§27.2)
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

### 51.2 Reynolds Boid Flocking

Three-rule boid steering (separation, alignment, cohesion) over a spatial-hash neighbourhood:

```glsl
// boids.comp
layout(local_size_x=64) in;
void main() {
    uint i=gl_GlobalInvocationID.x;
    vec3 pi=pos[i], vi=vel[i];
    vec3 sep=vec3(0), ali=vec3(0), coh=vec3(0); uint n=0;
    // iterate neighbours within BOID_RADIUS via §27.2 grid
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

### 51.3 Agent Steering over Flow Fields

Blend ORCA collision-avoidance velocity with flow-field global direction:

```glsl
// steer_integrate.comp
layout(local_size_x=64) in;
void main() {
    uint i=gl_GlobalInvocationID.x;
    ivec2 ci=ivec2(floor(pos[i].xz/CELL_SIZE));
    vec2 flow=flowField[ci.y*GRID_W+ci.x].dir;   // §70.3
    vec2 orca=newVel[i];                            // §51.1
    vec2 steer=normalize(mix(flow,orca,ORCA_BLEND))*MAX_SPEED;
    vel[i].xz=steer;
    pos[i]+=vel[i]*DT;
}
```

### 51.4 GPU Crowd Rendering via Task Shaders

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

Distant agents use billboard impostors (§10.7); close agents use full animated meshlets (§26.1–10.2). [Source: Reynolds 1987; van den Berg et al. 2011]

---

### 52. Fracture and Destruction

*Audience: graphics application developers.*

Fracture combines geometry processing (§3, §10, §49) with physics (§50) to produce visually plausible breaking and crumbling. GPU implementations pre-fracture offline for performance-critical paths and use runtime SDF crack propagation for interactive destruction.

### 52.1 Voronoi Pre-Fracture

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

After JFA, run GPU marching cubes (§3.2) on boundaries between adjacent seed regions to extract per-fragment surface meshes. Each fragment becomes an independent BLAS (§75.2) and rigid body (§50). [Source: Müller et al. "Real Time Dynamic Fracture with Volumetric Approximate Convex Decompositions", SIGGRAPH 2013]

### 52.2 Runtime Brittle Fracture via SDF Crack Propagation

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

### 52.3 FEM Ductile Fracture

Per-element failure criterion on the corotational FEM from §49.2:

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

### 53. Rope and Cable Simulation

*Audience: graphics application developers.*

Ropes and cables are one-dimensional elastic bodies with stretch stiffness, bending stiffness, torsion, and friction/contact with other objects. Unlike hair (§42, many thin non-extensible strands) ropes are single extensible bodies with full Cosserat rod dynamics, catenary rest configuration, and pulley/constraint coupling.

### 53.1 Extensible Cosserat Rod Dynamics

Model the rope as N segments with position **d**ᵢ and material frame (bishop frame) **eᵢ¹, eᵢ², eᵢ³**. Forces: stretch (resists length change), bending (resists curvature change), torsion (resists frame twist):

```glsl
// rope_forces.comp — compute stretch and bending forces per segment
layout(set=0,binding=0) buffer Pos { vec4 pos[]; };   // w=invMass
layout(set=0,binding=1) buffer Frame { mat3 frame[]; };
layout(set=0,binding=2) writeonly buffer Force { vec3 force[]; };
layout(push_constant) uniform PC {
    float ks;   // stretch stiffness
    float kb;   // bending stiffness
    float restLen;
} pc;
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    if (i == 0u || i >= N_SEGS) return;

    // Stretch force
    vec3  e    = pos[i].xyz - pos[i-1].xyz;
    float len  = length(e);
    vec3  fStr = pc.ks * (len - pc.restLen) * normalize(e);
    force[i]   -= fStr;
    force[i-1] += fStr;

    // Bending force (discrete curvature via Bishop frame parallel transport)
    if (i < N_SEGS - 1) {
        vec3 t0 = normalize(pos[i  ].xyz - pos[i-1].xyz);
        vec3 t1 = normalize(pos[i+1].xyz - pos[i  ].xyz);
        vec3 kb_v = 2.0 * cross(t0, t1) / (1.0 + dot(t0, t1));
        force[i-1] += pc.kb * kb_v / len;
        force[i+1] -= pc.kb * kb_v / len;
    }
}
```

[Source: Bergou et al. "Discrete Elastic Rods", SIGGRAPH 2008; Kaufman et al. "Adaptive Nonlinearity for Collisions in Complex Rod Assemblies", SIGGRAPH 2014]

### 53.2 Catenary Initialisation

A rope under gravity with fixed endpoints hangs in a catenary. Initialise segment positions analytically to avoid transient oscillations from a straight-line start:

```glsl
// catenary_init.comp — compute catenary segment positions
layout(set=0,binding=0) writeonly buffer Pos { vec4 pos[]; };
layout(push_constant) uniform PC {
    vec3 p0, p1;    // endpoints
    float ropeLen;  float g;
    int   N;        // number of segments
} pc;
layout(local_size_x=64) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    float t = float(i) / float(pc.N);
    // Solve catenary parameter a: L = a·sinh(d/(2a))·2 (binary search)
    float d  = length(pc.p1.xz - pc.p0.xz);
    float a  = solveCatenaryA(d, pc.ropeLen - abs(pc.p1.y - pc.p0.y));
    float x  = mix(pc.p0.x, pc.p1.x, t);
    float z  = mix(pc.p0.z, pc.p1.z, t);
    float xc = mix(-d*0.5, d*0.5, t);
    float y  = a * cosh(xc/a) + pc.p0.y - a;
    pos[i]   = vec4(x, y, z, i==0u||i==uint(pc.N)?0.0:1.0);  // w=invMass; 0=pinned
}
```

[Source: Irvine "Irvine on Cable Statics", 1981]

### 53.3 Pulley Constraint

A pulley maintains constant total rope length L_total = L_left + L_right while allowing the split point to slide:

```glsl
// pulley.comp — enforce pulley length constraint
layout(set=0,binding=0) buffer Pos { vec4 pos[]; };
layout(push_constant) uniform PC {
    uint leftEnd;     // index of rope endpoint on left side
    uint rightEnd;    // index of rope endpoint on right side
    uint pulleyVert;  // fixed pulley attachment point index
    float totalLen;
} pc;
layout(local_size_x=1) in;
void main() {
    float Ll = computeRopeLength(pc.leftEnd,  pc.pulleyVert);
    float Lr = computeRopeLength(pc.pulleyVert, pc.rightEnd);
    float err = (Ll + Lr) - pc.totalLen;
    // Distribute correction: shorten longer side, extend shorter side
    float frac = Ll / (Ll + Lr + 1e-5);
    applyLengthCorrection(pc.leftEnd,  pc.pulleyVert,  err * frac);
    applyLengthCorrection(pc.pulleyVert, pc.rightEnd, -err * (1.0-frac));
}
```

### 53.4 Rope-Rigid Body Contact

When the rope collides with a rigid body sphere (§50), apply position correction pushing the rope segment outside the sphere and a friction impulse tangent to the surface:

```glsl
// rope_contact.comp
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    for (uint b=0; b<N_BODIES; b++) {
        vec3  d    = pos[i].xyz - bodies[b].pos;
        float dist = length(d);
        float pen  = bodies[b].radius - dist;
        if (pen > 0.0 && pos[i].w > 0.0) {
            vec3 n    = normalize(d);
            pos[i].xyz += n * pen;
            // Friction: damp tangential velocity
            vec3 vt   = vel[i] - dot(vel[i],n)*n;
            vel[i]   -= vt * FRICTION;
        }
    }
}
```

[Source: Bergou et al. 2008; Spillmann & Teschner "An Adaptive Contact Model for the Robust Simulation of Knots", CGF 2008]

---

### 54. GPU Cloth Simulation

*Audience: graphics application developers.*

§39.6 introduced PBD cloth as a subsection of skeletal animation; §49.1 covered mass-spring systems. This section provides the dedicated treatment cloth deserves: woven fabric orthotropy, large-scale self-collision, runtime cutting and tearing, and aerodynamic forces. These distinguish cloth from both hair (§42, one-dimensional strands) and continuum FEM (§49, volumetric solids).

### 54.1 Orthotropic Cloth Constraints

Woven fabric resists stretch differently along warp (thread direction) and weft (cross-thread) axes. Encode per-face stiffness as an orthotropic tensor rather than a single Young's modulus:

```glsl
// cloth_stretch.comp — orthotropic stretch constraint per triangle
layout(set=0,binding=0) buffer Pos  { vec4 pos[];  };  // w = invMass
layout(set=0,binding=1) readonly buffer UV   { vec2 uv[];   };  // cloth-space coords
layout(set=0,binding=2) readonly buffer Tris { uvec3 tris[]; };
layout(push_constant) uniform PC { float ku; float kv; float dt; } pc;
layout(local_size_x=64) in;
void main() {
    uint t  = gl_GlobalInvocationID.x;
    uvec3 vi = tris[t];
    vec3  p0 = pos[vi.x].xyz, p1 = pos[vi.y].xyz, p2 = pos[vi.z].xyz;
    vec2  u0 = uv[vi.x],      u1 = uv[vi.y],       u2 = uv[vi.z];

    // Deformation gradient F = [p1-p0, p2-p0] * inv([u1-u0, u2-u0])
    mat2x3 dX = mat2x3(p1-p0, p2-p0);
    mat2   dU = mat2(u1-u0, u2-u0);
    mat2x3 F  = dX * inverse(dU);

    // Stretch along warp (u) and weft (v) axes
    float Cu = length(F[0]) - 1.0;  // stretch along warp
    float Cv = length(F[1]) - 1.0;  // stretch along weft
    vec3  nU = normalize(F[0]);
    vec3  nV = normalize(F[1]);

    // XPBD correction (simplified; full: account for constraint gradients)
    float corrU = pc.ku * Cu / (pos[vi.x].w + pos[vi.y].w + 1e-6);
    float corrV = pc.kv * Cv / (pos[vi.x].w + pos[vi.y].w + 1e-6);
    pos[vi.x].xyz -= corrU * nU * pos[vi.x].w;
    pos[vi.y].xyz += corrU * nU * pos[vi.y].w;
    pos[vi.x].xyz -= corrV * nV * pos[vi.x].w;
    pos[vi.z].xyz += corrV * nV * pos[vi.z].w;
}
```

[Source: Provot "Deformation Constraints in a Mass-Spring Model to Describe Rigid Cloth Behaviour", Graphics Interface 1995; Baraff & Witkin "Large Steps in Cloth Simulation", SIGGRAPH 1998]

### 54.2 Cloth Self-Collision via Spatial Hash

Self-collision at cloth densities (50k–500k triangles) uses the §27.2 spatial hash: each cloth vertex is hashed into a grid cell; pairs within one cell test for proximity:

```glsl
// cloth_selfcol.comp — vertex-vertex self-collision
layout(set=0,binding=0) buffer Pos { vec4 pos[]; };
layout(set=0,binding=1) readonly buffer GridStart { uint gStart[]; };
layout(set=0,binding=2) readonly buffer GridPts   { uint gPts[]; };
layout(push_constant) uniform PC { float thickness; float repulsion; } pc;
layout(local_size_x=64) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    ivec3 ci = ivec3(floor(pos[i].xyz / pc.thickness));
    for (int dz=-1;dz<=1;dz++) for(int dy=-1;dy<=1;dy++) for(int dx=-1;dx<=1;dx++) {
        uint cell = hashCell3(ci + ivec3(dx,dy,dz));
        for (uint k=gStart[cell]; k<gStart[cell+1]; k++) {
            uint j = gPts[k];
            if (j == i || sameTriangle(i,j)) continue;
            vec3  d = pos[i].xyz - pos[j].xyz;
            float dist = length(d);
            if (dist < pc.thickness && dist > 1e-5) {
                vec3 push = normalize(d) * (pc.thickness - dist) * pc.repulsion;
                if (pos[i].w > 0.0) pos[i].xyz += push * pos[i].w;
                if (pos[j].w > 0.0) pos[j].xyz -= push * pos[j].w;
            }
        }
    }
}
```

[Source: Bridson et al. "Robust Treatment of Collisions, Contact and Friction for Cloth Animation", SIGGRAPH 2002]

### 54.3 Aerodynamic Forces: Lift and Drag

Wind interaction applies lift and drag forces per cloth triangle based on the panel normal and relative wind velocity:

```glsl
// cloth_aero.comp — aerodynamic forces on cloth triangles
layout(set=0,binding=0) readonly buffer Pos  { vec4 pos[];  };
layout(set=0,binding=1) readonly buffer Vel  { vec3 vel[];  };
layout(set=0,binding=2) readonly buffer Tris { uvec3 t[];   };
layout(set=0,binding=3) buffer Force { vec3 force[]; };
layout(push_constant) uniform PC { vec3 wind; float rho; float cd; float cl; } pc;
layout(local_size_x=64) in;
void main() {
    uint ti  = gl_GlobalInvocationID.x;
    uvec3 vi = t[ti];
    vec3  p0 = pos[vi.x].xyz, p1 = pos[vi.y].xyz, p2 = pos[vi.z].xyz;
    vec3  vAvg = (vel[vi.x]+vel[vi.y]+vel[vi.z]) / 3.0;
    vec3  vRel = pc.wind - vAvg;
    vec3  n    = normalize(cross(p1-p0, p2-p0));
    float area = length(cross(p1-p0, p2-p0)) * 0.5;
    float vn   = dot(vRel, n);
    vec3  drag = 0.5 * pc.rho * pc.cd * area * length(vRel) * vRel;
    vec3  lift = 0.5 * pc.rho * pc.cl * area * vn * cross(cross(vRel,n), vRel);
    vec3  f    = (drag + lift) / 3.0;
    for (int k=0;k<3;k++) atomicAdd_vec3(force[vi[k]], f);
}
```

[Source: Leaf et al. "Interactive Design of Periodic Yarn-Level Cloth Patterns", SIGGRAPH 2018; Volino & Magnenat-Thalmann "Aerodynamic Effects on Cloth", Computers & Graphics 2001]

### 54.4 Runtime Cloth Cutting and Tearing

When edge stress exceeds a threshold, duplicate the shared vertex and update face connectivity:

```glsl
// cloth_tear.comp — detect overstressed edges, flag for topology update
layout(set=0,binding=0) readonly buffer Pos  { vec4 pos[]; };
layout(set=0,binding=1) readonly buffer Edges { uvec2 edges[]; };
layout(set=0,binding=2) buffer TearFlags { uint flags[]; };
layout(push_constant) uniform PC { float maxStress; float restLen; } pc;
layout(local_size_x=64) in;
void main() {
    uint e = gl_GlobalInvocationID.x;
    float stretch = length(pos[edges[e].x].xyz - pos[edges[e].y].xyz) / pc.restLen;
    if (stretch > pc.maxStress) atomicOr(flags[e/32], 1u << (e%32));
}
// CPU post-pass: for each flagged edge, duplicate vertex, retopologize adjacent faces,
// upload updated index buffer to GPU via vkCmdUpdateBuffer
```

[Source: Müller et al. "Real-Time Simulation of Deformation and Fracture of Stiff Materials", EG SCA 2001; Su et al. "Interactive Cutting of Deformable Objects in Virtual Environments", PRESENCE 2009]

---

### 55. GPU Fluid-Solid Coupling

*Audience: systems developers, graphics application developers.*

§48 (GPU fluid) and §50 (rigid body dynamics) operate independently. Two-way coupling means: (a) the solid moves fluid at the fluid-solid boundary (*no-slip* velocity boundary condition); (b) the fluid exerts pressure and viscous forces on the solid that accelerate it. This section covers the GPU geometry algorithms that implement this coupling.

### 55.1 Solid Boundary Voxelisation for Fluid Grids

At each frame, rasterise rigid body geometry into the fluid MAC grid cells. Voxels inside the solid are marked as solid cells; the fluid solver's pressure projection (§48.2) excludes them:

```glsl
// solid_vox_fluid.comp — mark MAC grid cells as solid or fluid
layout(set=0,binding=0) readonly buffer SolidTris { vec3 triV[]; };
layout(set=0,binding=1) buffer FluidMask  { uint solidMask[]; };  // 1 bit per cell
layout(push_constant) uniform PC { vec3 gridMin; float dx; ivec3 dim; uint nTris; } pc;
layout(local_size_x=64) in;
void main() {
    uint t   = gl_GlobalInvocationID.x;
    if (t >= pc.nTris) return;
    vec3  v0 = triV[t*3], v1 = triV[t*3+1], v2 = triV[t*3+2];
    ivec3 bMin = ivec3(floor((min(v0,min(v1,v2))-pc.gridMin)/pc.dx));
    ivec3 bMax = ivec3(ceil( (max(v0,max(v1,v2))-pc.gridMin)/pc.dx));
    bMin = max(bMin, ivec3(0));  bMax = min(bMax, pc.dim-1);
    for (int z=bMin.z;z<=bMax.z;z++) for(int y=bMin.y;y<=bMax.y;y++) for(int x=bMin.x;x<=bMax.x;x++) {
        vec3 cen = pc.gridMin + (vec3(x,y,z)+0.5)*pc.dx;
        if (triBoxOverlap(cen, pc.dx*0.5, v0,v1,v2)) {
            uint flat = z*pc.dim.y*pc.dim.x + y*pc.dim.x + x;
            atomicOr(solidMask[flat/32], 1u << (flat%32));
        }
    }
}
```

### 55.2 No-Slip Velocity Boundary Condition

Fluid cells adjacent to a solid boundary inherit the solid's velocity at the boundary face. For face (i+½, j, k) between a fluid and solid cell:

```glsl
// noslip.comp — set boundary velocity in MAC grid from solid body velocity
layout(set=0,binding=0) buffer VelX { float vx[]; };   // MAC grid face velocities
layout(set=0,binding=1) readonly buffer SolidMask { uint smask[]; };
layout(set=0,binding=2) readonly buffer SolidVelGrid { vec3 sv[]; };  // interpolated solid vel
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 v   = ivec3(gl_GlobalInvocationID);
    uint  c   = flatIdx(v), cp1 = flatIdx(v+ivec3(1,0,0));
    bool  solidC  = bool((smask[c  /32]>>(c  %32))&1u);
    bool  solidN  = bool((smask[cp1/32]>>(cp1%32))&1u);
    if (solidC != solidN)  // face on solid-fluid boundary
        vx[flatFaceX(v)] = sv[solidC ? c : cp1].x;  // solid velocity at boundary
}
```

[Source: Bridson "Fluid Simulation for Computer Graphics", 2nd ed., §4; Batty et al. "A Fast Variational Framework for Accurate Solid-Fluid Coupling", SIGGRAPH 2007]

### 55.3 Fluid Pressure Force on Solid

Integrate fluid pressure over the solid surface to compute net force and torque on the rigid body. Sample pressure at each solid surface triangle's centroid from the MAC grid:

```glsl
// fluid_force.comp — accumulate pressure force on solid from fluid
layout(set=0,binding=0) readonly buffer Pressure { float p[]; };
layout(set=0,binding=1) readonly buffer SolidTris { vec3 tv[]; vec3 tn[]; };
layout(set=0,binding=2) buffer Force { vec3 force; vec3 torque; };
layout(push_constant) uniform PC { vec3 gridMin; float dx; vec3 centroid; } pc;
layout(local_size_x=64) in;
void main() {
    uint t = gl_GlobalInvocationID.x;
    vec3 cen  = (tv[t*3]+tv[t*3+1]+tv[t*3+2])/3.0;
    float area = length(cross(tv[t*3+1]-tv[t*3],tv[t*3+2]-tv[t*3]))*0.5;
    vec3  n    = tn[t];
    ivec3 ci   = ivec3(floor((cen-pc.gridMin)/pc.dx));
    float pval = trilinearSamplePressure(p, ci, (cen-pc.gridMin)/pc.dx-vec3(ci));
    vec3  f    = -pval * n * area;
    atomicAdd_vec3(force,  f);
    atomicAdd_vec3(torque, cross(cen-pc.centroid, f));
}
```

### 55.4 Two-Way Coupling Loop

The per-frame GPU pipeline for two-way fluid-solid coupling:

```
1. vkCmdDispatch(solid_vox_fluid)    — §55.1: mark solid cells
2. vkCmdDispatch(noslip)             — §55.2: set boundary velocities
3. vkCmdDispatch(fluid_advect)       — §48.1: semi-Lagrangian advection
4. vkCmdDispatch(fluid_pressure_cg)  — §48.2: pressure projection (respects solid cells)
5. vkCmdDispatch(fluid_force)        — §55.3: compute force on solid
6. vkCmdDispatch(rigid_body_step)    — §50.3: integrate solid velocity with fluid force
7. vkCmdDispatch(blas_refit)         — §75.5: refit solid BLAS for next frame's ray queries
```

[Source: Batty et al. 2007; Robinson-Mosher et al. "Two-way Coupling of Fluids to Rigid and Deformable Solids and Shells", SIGGRAPH 2008]

---

### 56. Granular Material Simulation (DEM)

*Audience: systems developers, graphics application developers.*

The Discrete Element Method (DEM) simulates granular materials — sand, gravel, powder, pebbles — as collections of rigid spheres or polyhedral grains interacting via contact forces. Each particle-particle contact generates a spring-dashpot force; GPU parallelism comes from the same spatial hashing (§27.2) used in SPH (§48.4) and cloth self-collision (§54.2).

### 56.1 DEM Force Computation: Hertz Contact

For sphere-sphere contact, the Hertz contact model computes a non-linear spring force proportional to δ^(3/2) where δ is the overlap depth:

```glsl
// dem_force.comp — sphere-sphere Hertz contact forces
layout(set=0,binding=0) buffer Pos    { vec4 pos[];  };  // w = radius
layout(set=0,binding=1) buffer Vel    { vec3 vel[];  };
layout(set=0,binding=2) buffer AngVel { vec3 omega[]; };
layout(set=0,binding=3) buffer Force  { vec3 force[]; };
layout(set=0,binding=4) buffer Torque { vec3 torque[]; };
layout(set=0,binding=5) readonly buffer Pairs { uvec2 pairs[]; };
layout(push_constant) uniform PC { float E; float nu; float gamma; float mu_f; float dt; } pc;
layout(local_size_x=64) in;
void main() {
    uint p   = gl_GlobalInvocationID.x;
    uint i   = pairs[p].x, j = pairs[p].y;
    vec3  d  = pos[j].xyz - pos[i].xyz;
    float r  = pos[i].w + pos[j].w;
    float dl = length(d) - r;  // negative = overlap
    if (dl >= 0.0) return;
    float delta = -dl;
    vec3  n  = normalize(d);
    // Hertz normal force
    float Estar = pc.E / (2.0*(1.0-pc.nu*pc.nu));
    float Rstar = pos[i].w * pos[j].w / r;
    float kn    = 4.0/3.0 * Estar * sqrt(Rstar * delta);
    // Relative velocity at contact point
    vec3  vc = vel[j]-vel[i] + cross(omega[j],pos[j].w*(-n)) - cross(omega[i],pos[i].w*n);
    float vn = dot(vc, n);
    // Tangential (friction) force via Coulomb limit
    vec3  vt = vc - vn*n;
    vec3  Fn = (kn * delta + pc.gamma * vn) * n;
    vec3  Ft = clampLength(-pc.mu_f*length(Fn)*normalize(vt), length(Fn)*pc.mu_f);
    vec3  F  = Fn + Ft;
    atomicAdd_vec3(force[i], -F);
    atomicAdd_vec3(force[j],  F);
    atomicAdd_vec3(torque[i], cross(-pos[i].w*n, -F));
    atomicAdd_vec3(torque[j], cross( pos[j].w*n,  F));
}
```

[Source: Cundall & Strack "A Discrete Numerical Model for Granular Assemblies", Géotechnique 1979; Šmilauer et al. "Yade Documentation 2nd ed.", 2015]

### 56.2 Spatial Hashing for DEM Neighbour Search

Reuse the §27.2 / §54.2 spatial hash to find all particle pairs within contact range each frame. The hash cell size equals the maximum particle diameter:

```glsl
// dem_hash.comp — insert particles into spatial hash, find contacts
layout(set=0,binding=0) readonly buffer Pos   { vec4 pos[];  };  // w = radius
layout(set=0,binding=1) buffer HashTable { uint cells[HASH_SIZE]; };
layout(set=0,binding=2) buffer ContactList { uvec2 contacts[]; uint count; };
layout(push_constant) uniform PC { float cellSize; uint N; } pc;
layout(local_size_x=64) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    if (i >= pc.N) return;
    ivec3 ci = ivec3(floor(pos[i].xyz / pc.cellSize));
    // Check 3×3×3 neighbourhood for overlapping particles
    for (int dz=-1;dz<=1;dz++) for(int dy=-1;dy<=1;dy++) for(int dx=-1;dx<=1;dx++) {
        uint cell = hashCell3(ci+ivec3(dx,dy,dz));
        for (uint k=hashStart[cell]; k<hashStart[cell+1]; k++) {
            uint j = hashPts[k];
            if (j <= i) continue;
            float d = length(pos[i].xyz-pos[j].xyz) - (pos[i].w+pos[j].w);
            if (d < 0.0) {
                uint slot = atomicAdd(count, 1u);
                contacts[slot] = uvec2(i,j);
            }
        }
    }
}
```

### 56.3 DEM–Fluid Coupling (CFD-DEM)

For wet granular flows (slurry, sediment transport), couple DEM with the fluid solver (§48 / §55): the fluid exerts drag on each particle; each particle displaces fluid from its volume:

```glsl
// dem_fluid_drag.comp — compute drag force on DEM sphere from local fluid velocity
layout(set=0,binding=0) readonly buffer Pos   { vec4 pos[];  };
layout(set=0,binding=1) readonly buffer VelP  { vec3 vp[];   };
layout(set=0,binding=2) uniform sampler3D fluidVel;  // MAC grid interpolated to particle pos
layout(set=0,binding=3) buffer Force { vec3 force[]; };
layout(push_constant) uniform PC { float rho; float nu; vec3 gridMin; float dx; } pc;
layout(local_size_x=64) in;
void main() {
    uint i    = gl_GlobalInvocationID.x;
    vec3  uvw = (pos[i].xyz - pc.gridMin) / (pc.dx * GRID_DIM);
    vec3  vf  = texture(fluidVel, uvw).xyz;
    vec3  vrel = vf - vp[i];
    float Re  = 2.0*pos[i].w*length(vrel)/pc.nu;
    float Cd  = (Re < 1000.0) ? 24.0/Re*(1.0+0.15*pow(Re,0.687)) : 0.44;
    float A   = 3.14159*pos[i].w*pos[i].w;
    vec3  Fd  = 0.5*pc.rho*Cd*A*length(vrel)*vrel;
    atomicAdd_vec3(force[i], Fd);
}
```

[Source: Di Felice "The Voidage Function for Fluid-Particle Interaction Systems", IJMF 1994; Kloss et al. "Models, Algorithms and Validation for Opensource DEM and CFD-DEM", Progress in CFD 2012]

### 56.4 Granular Surface Rendering: Instanced Sphere Splatting

Render millions of DEM spheres efficiently as GPU-instanced impostors: a single quad per particle, oriented toward the camera, with a ray-sphere intersection in the fragment shader:

```glsl
// dem_render.vert — instance sphere impostor billboard
layout(location=0) in vec2 quadUV;  // [-1,1]^2 screen quad
layout(location=0) flat out vec3 sphereCenter;
layout(location=1) flat out float sphereRadius;
layout(push_constant) uniform PC { mat4 viewProj; vec3 camPos; } pc;
void main() {
    uint i  = gl_InstanceIndex;
    sphereCenter = pos[i].xyz;
    sphereRadius = pos[i].w;
    // Billboard: offset quad in view space
    vec3 right = normalize(cross(pc.camPos-sphereCenter, vec3(0,1,0)));
    vec3 up    = cross(normalize(pc.camPos-sphereCenter), right);
    vec3 wp    = sphereCenter + (right*quadUV.x + up*quadUV.y)*sphereRadius*1.01;
    gl_Position = pc.viewProj * vec4(wp,1.0);
}
// Fragment: ray-sphere intersection → depth override for correct occlusion
```

[Source: Gumhold "Splatting Illuminated Ellipsoids with Depth Correction", VMV 2003; Müller "Particle-Based Fluid Simulation for Interactive Applications", SCA 2003]

---

### 57. Terrain Sculpting and Hydraulic Erosion

*Audience: graphics application developers.*

Procedural terrain generation (§71 covers runtime rendering; §73 covers ocean simulation) leaves open the sculpting and erosion simulation that shapes the terrain heightfield before rendering. GPU hydraulic erosion simulates millions of water droplets carrying sediment down gradient-following paths; thermal erosion redistributes material based on angle-of-repose; and interactive sculpting applies brush strokes at render resolution.

### 57.1 Particle-Based Hydraulic Erosion

Each virtual water droplet carries sediment, picks up material where excess capacity exists, and deposits where at capacity. Millions of droplets run in parallel, each as an independent compute thread:

```glsl
// erosion_droplet.comp — one GPU hydraulic erosion droplet simulation
layout(set=0,binding=0) buffer Heightmap { float h[]; };
layout(set=0,binding=1) buffer Sediment  { float sed[]; };
layout(set=0,binding=2) readonly buffer StartPos { vec2 startPos[]; };
layout(push_constant) uniform PC {
    int W; int H; float inertia; float capacity;
    float deposition; float erosion; float evaporation; float gravity;
    int maxSteps;
} pc;
layout(local_size_x=64) in;
void main() {
    uint di = gl_GlobalInvocationID.x;
    vec2 pos = startPos[di];
    vec2 dir = vec2(0.0);
    float water=1.0, speed=0.0, sediment=0.0;
    for (int s=0; s<pc.maxSteps && water>0.01; s++) {
        ivec2 c = ivec2(pos);
        if (c.x<1||c.y<1||c.x>=pc.W-1||c.y>=pc.H-1) break;
        // Bilinear gradient of heightmap
        vec2 grad = vec2(
            h[(c.y  )*pc.W+(c.x+1)] - h[(c.y  )*pc.W+(c.x-1)],
            h[(c.y+1)*pc.W+(c.x  )] - h[(c.y-1)*pc.W+(c.x  )]) * 0.5;
        dir = normalize(mix(-grad, dir, pc.inertia));
        vec2 npos = pos + dir;
        float dh  = bilinearH(h, npos, pc.W) - bilinearH(h, pos, pc.W);
        float cap = max(-dh, 0.01) * speed * water * pc.capacity;
        if (sediment > cap || dh > 0.0) {
            float dep = (dh > 0.0) ? min(dh, sediment) : (sediment-cap)*pc.deposition;
            atomicAdd_float(h[c.y*pc.W+c.x], dep);
            sediment -= dep;
        } else {
            float ero = min((cap-sediment)*pc.erosion, -dh);
            atomicAdd_float(h[c.y*pc.W+c.x], -ero);
            sediment += ero;
        }
        speed  = sqrt(max(speed*speed + dh*pc.gravity, 0.0));
        water *= (1.0 - pc.evaporation);
        pos    = npos;
    }
}
```

[Source: Olsen "Realtime Procedural Terrain Generation", 2004; Beneš & Forsbach "Layered Data Representation for Visual Simulation of Terrain Erosion", SCCG 2001; GPU erosion: Št'ava et al. "Interactive Terrain Modeling Using Hydraulic Erosion", SCA 2008]

### 57.2 Thermal Erosion

Thermal erosion redistributes material from cells whose slope exceeds the angle of repose (typically 30–40° for soil). Each grid cell checks its 4 neighbours:

```glsl
// thermal_erosion.comp — redistribute material above angle of repose
layout(set=0,binding=0) buffer Heightmap { float h[]; };
layout(push_constant) uniform PC { int W; int H; float dx; float tanAngle; float rate; } pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 id  = ivec2(gl_GlobalInvocationID.xy);
    if (id.x<=0||id.y<=0||id.x>=pc.W-1||id.y>=pc.H-1) return;
    uint  c   = id.y*pc.W+id.x;
    float hc  = h[c];
    float dTotal = 0.0; float excess = 0.0;
    const ivec2 DIRS[4] = {ivec2(1,0),ivec2(-1,0),ivec2(0,1),ivec2(0,-1)};
    float dh[4];
    for (int d=0;d<4;d++) {
        uint nb = (id.y+DIRS[d].y)*pc.W+(id.x+DIRS[d].x);
        dh[d]  = hc - h[nb];
        if (dh[d] > pc.tanAngle*pc.dx) { dTotal+=dh[d]; excess+=dh[d]-pc.tanAngle*pc.dx; }
    }
    if (dTotal < 1e-6) return;
    float move = excess * pc.rate * 0.5;
    atomicAdd_float(h[c], -move);
    for (int d=0;d<4;d++) if (dh[d] > pc.tanAngle*pc.dx) {
        uint nb = (id.y+DIRS[d].y)*pc.W+(id.x+DIRS[d].x);
        atomicAdd_float(h[nb], move * dh[d]/dTotal);
    }
}
```

[Source: Musgrave et al. "The Synthesis and Rendering of Eroded Fractal Terrains", SIGGRAPH 1989; Št'ava et al. 2008]

### 57.3 Interactive Terrain Sculpting Brushes

Real-time sculpting applies a Gaussian-weighted height offset inside a brush radius. The brush follows the user's cursor position at interactive rates:

```glsl
// sculpt_brush.comp — apply sculpt brush to heightmap
layout(set=0,binding=0) buffer Heightmap { float h[]; };
layout(push_constant) uniform PC {
    vec2  centre;     // brush centre in heightmap coordinates
    float radius;     // brush radius in pixels
    float strength;   // height change per frame
    float falloff;    // Gaussian sigma / radius ratio
    int   W; int H;
    int   mode;       // 0=raise 1=lower 2=smooth 3=flatten
} pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 id  = ivec2(gl_GlobalInvocationID.xy);
    vec2  d   = vec2(id) - pc.centre;
    float r   = length(d) / pc.radius;
    if (r > 1.0) return;
    float w   = exp(-r*r/(2.0*pc.falloff*pc.falloff));
    uint  flat= id.y*pc.W+id.x;
    if      (pc.mode==0) h[flat] += w * pc.strength;
    else if (pc.mode==1) h[flat] -= w * pc.strength;
    else if (pc.mode==2) {  // smooth: Laplacian step
        float avg = (h[flat-1]+h[flat+1]+h[flat-pc.W]+h[flat+pc.W])*0.25;
        h[flat]   = mix(h[flat], avg, w * pc.strength);
    }
    else if (pc.mode==3) h[flat] = mix(h[flat], pc.strength, w);  // flatten to target height
}
```

[Source: Gain et al. "Terrain Sketching", I3D 2009; Hnaidi et al. "Feature Based Terrain Generation Using Diffusion Equation", CGF 2010]

### 57.4 Heightmap Normal and LOD Update

After sculpting or erosion, recompute normals and terrain LOD tiles (§71 pattern) for the dirty region:

```glsl
// terrain_normals.comp — recompute normals for a dirty tile after sculpting
layout(set=0,binding=0) readonly buffer Heightmap { float h[]; };
layout(set=0,binding=1) writeonly buffer Normals { vec3 nor[]; };
layout(push_constant) uniform PC { int W; float dx; ivec4 dirtyRect; } pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 id  = ivec2(gl_GlobalInvocationID.xy) + pc.dirtyRect.xy;
    if (any(greaterThanEqual(id, pc.dirtyRect.zw))) return;
    uint  c   = id.y*pc.W+id.x;
    float hL  = h[c-1], hR=h[c+1], hD=h[c-pc.W], hU=h[c+pc.W];
    vec3  n   = normalize(vec3(hL-hR, 2.0*pc.dx, hD-hU));
    nor[c]    = n;
}
// vkCmdUpdateBuffer the dirty tile heightmap → GPU; then dispatch normal update
// Optionally: rebuild LOD quadtree for the dirty tile (§71 clipmap update)
```

[Source: Losasso & Hoppe "Geometry Clipmaps: Terrain Rendering Using Nested Regular Grids", SIGGRAPH 2004; Bruneton & Neyret "Real-time Realistic Ocean Lighting using Seamless Transitions from Geometry to BRDF", EG 2010]

---

### 58. GPU Fluid Surface Extraction

*Audience: graphics application developers.*

§48 (SPH/FLIP fluid simulation) produces particle positions and velocities; §65 (isosurface extraction) expects a grid scalar field. Bridging them for liquid rendering requires: (1) splatting anisotropic kernels from particles into a density grid; (2) running marching cubes on that grid; (3) smoothing the resulting mesh. This pipeline differs from generic isosurface extraction because the kernels are oriented by the local velocity gradient to produce smooth water surfaces without blobby artefacts.

### 58.1 Isotropic Density Splatting

The simplest approach: for each particle i, add a radial kernel contribution to nearby grid cells. Parallelise over particles with atomic accumulation:

```glsl
// fluid_splat_iso.comp — isotropic kernel splatting into density grid
layout(set=0,binding=0) readonly buffer Particles { vec4 pos[]; };  // w = mass
layout(set=0,binding=1) buffer Density { float rho[]; };
layout(push_constant) uniform PC {
    vec3 gridMin; float dx; ivec3 dim; float h; uint N;
} pc;
layout(local_size_x=64) in;
void main() {
    uint p   = gl_GlobalInvocationID.x;
    if (p >= pc.N) return;
    vec3  pi  = pos[p].xyz;
    float m   = pos[p].w;
    ivec3 ci  = ivec3(floor((pi - pc.gridMin) / pc.dx));
    int   r   = int(ceil(pc.h / pc.dx));
    for (int dz=-r;dz<=r;dz++) for(int dy=-r;dy<=r;dy++) for(int dx=-r;dx<=r;dx++) {
        ivec3 nb = ci + ivec3(dx,dy,dz);
        if (any(lessThan(nb,ivec3(0)))||any(greaterThanEqual(nb,pc.dim))) continue;
        vec3  xg = pc.gridMin + (vec3(nb)+0.5)*pc.dx;
        float r2 = dot(pi-xg,pi-xg);
        if (r2 > pc.h*pc.h) continue;
        float q  = sqrt(r2)/pc.h;
        float W  = max(0.0, 1.0-q*q) * (315.0/(64.0*3.14159*pow(pc.h,3.0)));
        atomicAdd_float(rho[nb.z*pc.dim.y*pc.dim.x + nb.y*pc.dim.x + nb.x], m*W);
    }
}
```

[Source: Zhu & Bridson "Animating Sand as a Fluid", SIGGRAPH 2005; Müller et al. "Particle-Based Fluid Simulation", SCA 2003]

### 58.2 Anisotropic Kernel Splatting

Yu & Turk (2013) orient each particle's kernel using the local velocity gradient covariance matrix, producing smooth surface sheets rather than blobs:

```glsl
// fluid_splat_aniso.comp — anisotropic kernel splatting (Yu & Turk 2013)
layout(set=0,binding=0) readonly buffer Particles { vec3 pos[]; };
layout(set=0,binding=1) readonly buffer KernelG   { mat3 G[];   };  // per-particle anisotropy
layout(set=0,binding=2) buffer Density { float rho[]; };
layout(push_constant) uniform PC { vec3 gridMin; float dx; ivec3 dim; uint N; } pc;
layout(local_size_x=64) in;
void main() {
    uint p  = gl_GlobalInvocationID.x;
    if (p >= pc.N) return;
    mat3  Gp = G[p];
    // Bounding box of anisotropic kernel in grid space
    vec3  ext = vec3(length(Gp[0]), length(Gp[1]), length(Gp[2]));
    ivec3 ci  = ivec3(floor((pos[p]-pc.gridMin)/pc.dx));
    ivec3 r   = ivec3(ceil(ext/pc.dx)) + 1;
    for (int dz=-r.z;dz<=r.z;dz++) for(int dy=-r.y;dy<=r.y;dy++) for(int dx=-r.x;dx<=r.x;dx++) {
        ivec3 nb  = ci + ivec3(dx,dy,dz);
        if (any(lessThan(nb,ivec3(0)))||any(greaterThanEqual(nb,pc.dim))) continue;
        vec3  xg  = pc.gridMin + (vec3(nb)+0.5)*pc.dx;
        vec3  d   = Gp * (xg - pos[p]);  // transform to kernel space
        float r2  = dot(d,d);
        if (r2 >= 1.0) continue;
        float W   = pow(1.0-r2, 3.0) * (315.0/(64.0*3.14159));
        atomicAdd_float(rho[flatIdx(nb,pc.dim)], W * determinant(Gp));
    }
}
```

[Source: Yu & Turk "Reconstructing Surfaces of Particle-Based Fluids Using Anisotropic Kernels", ACM TOG 2013]

### 58.3 Screen-Space Fluid Rendering (Depth Smoothing)

An alternative to volumetric splatting: render particle spheres to a depth buffer, blur the depth field with a curvature-flow filter, then shade the resulting surface as a thin dielectric:

```glsl
// fluid_depthsmooth.comp — bilateral curvature-flow filter on fluid depth buffer
layout(set=0,binding=0) uniform sampler2D fluidDepth;   // particle sphere depth
layout(set=0,binding=1, r32f) uniform writeonly image2D smoothedDepth;
layout(push_constant) uniform PC { float sigmaD; float sigmaZ; int radius; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    float zc   = texelFetch(fluidDepth, pix, 0).r;
    if (zc <= 0.0) { imageStore(smoothedDepth, pix, vec4(0)); return; }
    float sum  = 0.0, wSum = 0.0;
    for (int dy=-pc.radius;dy<=pc.radius;dy++) for(int dx=-pc.radius;dx<=pc.radius;dx++) {
        float zn  = texelFetch(fluidDepth, pix+ivec2(dx,dy), 0).r;
        if (zn <= 0.0) continue;
        float wD  = exp(-(dx*dx+dy*dy)/(2.0*pc.sigmaD*pc.sigmaD));
        float wZ  = exp(-(zc-zn)*(zc-zn)/(2.0*pc.sigmaZ*pc.sigmaZ));
        sum  += wD*wZ*zn; wSum += wD*wZ;
    }
    imageStore(smoothedDepth, pix, vec4(wSum>0.0 ? sum/wSum : zc));
}
```

[Source: van der Laan et al. "Screen Space Fluid Rendering with Curvature Flow", I3D 2009; Green "Screen Space Fluid Rendering", GDC 2010]

### 58.4 Surface Normal and Shading

Reconstruct the surface normal from the smoothed depth buffer, then shade as a thin glass/water surface using Fresnel reflectance and refraction with a cubemap:

```glsl
// fluid_shade.frag — shade fluid surface from smoothed depth
uniform sampler2D smoothDepth;
uniform samplerCube envMap;
uniform mat4 invProj;
in vec2 texUV;
void main() {
    float z  = texture(smoothDepth, texUV).r;
    if (z <= 0.0) discard;
    // Reconstruct view-space position
    vec4  vp = invProj * vec4(texUV*2-1, z*2-1, 1); vp /= vp.w;
    // Finite-difference normal from depth buffer
    vec3  n  = normalize(vec3(
        dFdx(vp.z), dFdy(vp.z), -length(vec2(dFdx(vp.z),dFdy(vp.z)))));
    // Fresnel reflection + refraction
    vec3  v  = normalize(-vp.xyz);
    float F  = 0.02 + 0.98*pow(1.0-max(dot(n,v),0.0), 5.0);
    vec3  r  = reflect(-v, n);
    vec3  t  = refract(-v, n, 1.0/1.33);  // water IOR
    fragColor = vec4(mix(texture(envMap,t).rgb, texture(envMap,r).rgb, F), 1.0);
}
```

[Source: Green 2010; James & Fatahalian "Precomputing Interactive Dynamic Deformable Scenes", SIGGRAPH 2003]

---

### 59. Continuous Collision Detection (CCD)

*Audience: systems developers, graphics application developers.*

Discrete collision detection (§64) checks for overlap at a single instant; fast-moving or thin objects may tunnel through each other between frames. Continuous collision detection finds the time of first contact (TOC) τ ∈ [0,1] within a timestep by solving the polynomial distance equation between swept primitives. GPU parallelism runs independent TOC solves for all candidate pairs simultaneously.

### 59.1 Vertex–Triangle CCD via Conservative Advancement

Conservative advancement (Zhang et al. 2007) iteratively advances the time along the minimum separation distance until contact or the interval is exhausted:

```glsl
// ccd_vertex_tri.comp — vertex-triangle CCD via conservative advancement
layout(set=0,binding=0) readonly buffer PosT0 { vec3 p0[]; };   // positions at t=0
layout(set=0,binding=1) readonly buffer PosT1 { vec3 p1[]; };   // positions at t=1
layout(set=0,binding=2) readonly buffer Pairs  { uvec4 vt[];    };  // vertex idx + 3 tri verts
layout(set=0,binding=3) writeonly buffer TOC   { float toc[];   };  // time of contact (-1=none)
layout(local_size_x=64) in;
void main() {
    uint pi = gl_GlobalInvocationID.x;
    uint vi = vt[pi].x;
    uint ai = vt[pi].y, bi = vt[pi].z, ci = vt[pi].w;
    float t  = 0.0;
    for (int iter=0; iter<64; iter++) {
        vec3 v  = mix(p0[vi],p1[vi],t);
        vec3 a  = mix(p0[ai],p1[ai],t);
        vec3 b  = mix(p0[bi],p1[bi],t);
        vec3 c  = mix(p0[ci],p1[ci],t);
        float d = pointTriDist(v, a, b, c);
        if (d < 1e-6) { toc[pi]=t; return; }
        // Motion bound: max speed × remaining time
        float vMax = max(length(p1[vi]-p0[vi]),
                     max(length(p1[ai]-p0[ai]),
                     max(length(p1[bi]-p0[bi]), length(p1[ci]-p0[ci]))));
        float dt   = d / (vMax + 1e-8);
        t += dt;
        if (t >= 1.0) break;
    }
    toc[pi] = -1.0;  // no contact
}
```

[Source: Zhang et al. "Continuous Collision Detection for Rigid and Articulated Bodies", SIGGRAPH 2007; Tang et al. "ICCD: Interactive Continuous Collision Detection between Deformable Models Using Connectivity-Based Culling", IEEE TVCG 2009]

### 59.2 Edge–Edge CCD via Cubic Root Finding

For edge-edge contact between two moving line segments, the coplanarity condition produces a cubic polynomial in τ whose roots are the candidate contact times:

```glsl
// ccd_edge_edge.comp — edge-edge CCD via cubic root isolation
layout(set=0,binding=0) readonly buffer PosT0 { vec3 p0[]; };
layout(set=0,binding=1) readonly buffer PosT1 { vec3 p1[]; };
layout(set=0,binding=2) readonly buffer EEPairs { uvec4 ee[]; };
layout(set=0,binding=3) writeonly buffer TOC { float toc[]; };
layout(local_size_x=64) in;

// Cubic coefficients of (e1(τ) × e2(τ)) · (v1(τ) - v3(τ)) = 0
void cubicCoeffs(vec3 a0,vec3 a1,vec3 b0,vec3 b1,vec3 c0,vec3 c1,vec3 d0,vec3 d1,
                 out float A, out float B, out float C, out float D) {
    vec3 da=a1-a0, db=b1-b0, dc=c1-c0, dd=d1-d0;
    // A*t³ + B*t² + C*t + D = scalar triple product of swept edge vectors
    vec3 e1=b0-a0, e2=d0-c0, r=c0-a0;
    A = dot(cross(da-db, dc-dd), da-db) + dot(cross(e1,dc-dd), da-db)
      + dot(cross(da-db, e2), da-db);
    // (simplified — full derivation in Bridson et al. 2002)
    B = dot(cross(e1,e2), da-db) + dot(cross(da-db,e2),r)
      + dot(cross(e1,dc-dd),r);
    C = dot(cross(e1,e2),r) + dot(cross(da-db,e2),r);
    D = dot(cross(e1,e2),r);
}
void main() {
    uint pi = gl_GlobalInvocationID.x;
    float A,B,C,D;
    cubicCoeffs(p0[ee[pi].x],p1[ee[pi].x], p0[ee[pi].y],p1[ee[pi].y],
                p0[ee[pi].z],p1[ee[pi].z], p0[ee[pi].w],p1[ee[pi].w], A,B,C,D);
    float roots[3]; int nr = solveCubic(A,B,C,D,roots);
    float earliest = 1.0;
    for (int r=0; r<nr; r++) if (roots[r]>0&&roots[r]<earliest&&verifyContact(roots[r],pi)) earliest=roots[r];
    toc[pi] = (earliest < 1.0) ? earliest : -1.0;
}
```

[Source: Bridson et al. "Robust Treatment of Collisions, Contact and Friction for Cloth Animation", SIGGRAPH 2002; Provot "Collision and Self-Collision Handling in Cloth Model Dedicated to Design Garments", CAS 1997]

### 59.3 Broadphase CCD via Swept AABB

Before running exact CCD, cull pairs whose swept bounding boxes (union of AABB at t=0 and t=1) do not overlap:

```glsl
// ccd_broadphase.comp — swept AABB overlap test for CCD culling
layout(set=0,binding=0) readonly buffer AABB_T0 { vec6 aabb0[]; };
layout(set=0,binding=1) readonly buffer AABB_T1 { vec6 aabb1[]; };
layout(set=0,binding=2) readonly buffer Candidates { uvec2 pairs[]; };
layout(set=0,binding=3) buffer Survivors { uvec2 surv[]; uint count; };
layout(local_size_x=64) in;
void main() {
    uint pi  = gl_GlobalInvocationID.x;
    uint a   = pairs[pi].x, b = pairs[pi].y;
    // Swept AABB: union of t=0 and t=1 box
    vec3 aminS = min(aabb0[a].xyz, aabb1[a].xyz);
    vec3 amaxS = max(aabb0[a].xyz+aabb0[a].xyz, aabb1[a].xyz+aabb1[a].xyz);
    vec3 bminS = min(aabb0[b].xyz, aabb1[b].xyz);
    vec3 bmaxS = max(aabb0[b].xyz+aabb0[b].xyz, aabb1[b].xyz+aabb1[b].xyz);
    if (all(lessThan(aminS,bmaxS)) && all(lessThan(bminS,amaxS))) {
        uint slot = atomicAdd(count, 1u);
        surv[slot] = pairs[pi];
    }
}
```

[Source: Ericson "Real-Time Collision Detection", Morgan Kaufmann 2005, §5.3]

### 59.4 CCD Response: Impact Zone Projection

After finding τ, project the impacting vertices out of collision along the contact normal and apply impulses. The impact zone (Bridson et al. 2002) groups all vertices involved in a contact cluster:

```glsl
// ccd_response.comp — apply collision response at TOC τ
layout(set=0,binding=0) buffer Pos { vec3 pos[]; };
layout(set=0,binding=1) buffer Vel { vec3 vel[]; };
layout(set=0,binding=2) readonly buffer Contacts { CCDContact contacts[]; };
layout(push_constant) uniform PC { float restitution; float friction; } pc;
layout(local_size_x=64) in;
void main() {
    uint c  = gl_GlobalInvocationID.x;
    CCDContact ct = contacts[c];
    vec3  n = ct.normal;
    float vRel = dot(vel[ct.v] - vel[ct.a]*ct.w0 - vel[ct.b]*ct.w1 - vel[ct.c]*ct.w2, n);
    if (vRel >= 0.0) return;  // separating: no impulse needed
    float j = -(1.0+pc.restitution)*vRel /
              (ct.invMassV + ct.w0*ct.w0*ct.invMassA + ct.w1*ct.w1*ct.invMassB + ct.w2*ct.w2*ct.invMassC);
    atomicAdd_vec3(vel[ct.v],  j*n*ct.invMassV);
    atomicAdd_vec3(vel[ct.a], -j*n*ct.invMassA*ct.w0);
    atomicAdd_vec3(vel[ct.b], -j*n*ct.invMassB*ct.w1);
    atomicAdd_vec3(vel[ct.c], -j*n*ct.invMassC*ct.w2);
}
```

[Source: Bridson et al. 2002; Harmon et al. "Robust Treatment of Simultaneous Collisions", SIGGRAPH 2008]

---

### 60. Minkowski Sum and GPU Motion Planning

*Audience: systems developers.*

The Minkowski sum A⊕B of two convex shapes A and B is the shape swept by placing B's centre on every point of A; it is the configuration-space obstacle (C-obstacle) for robot motion planning. GPU computation of Minkowski sums on convex polytopes and GPU-parallel probabilistic roadmap (PRM) and RRT planning enable real-time collision-free path queries.

### 60.1 Minkowski Sum of Convex Polytopes (GJK-Based)

For two convex polytopes A and B with n_A and n_B vertices respectively, the Minkowski sum has at most O(n_A · n_B) vertices. GPU enumerates candidate extreme points and culls non-hull points:

```glsl
// mink_sum_vertices.comp — generate Minkowski sum vertex candidates
layout(set=0,binding=0) readonly buffer VertsA { vec3 vA[]; };
layout(set=0,binding=1) readonly buffer VertsB { vec3 vB[]; };
layout(set=0,binding=2) writeonly buffer SumVerts { vec3 sv[]; };
layout(local_size_x=16,local_size_y=16) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    uint j  = gl_GlobalInvocationID.y;
    if (i >= N_A || j >= N_B) return;
    sv[i*N_B+j] = vA[i] + vB[j];  // Minkowski sum vertex candidate
}
// Then run §106 GPU convex hull on sv[] to get the actual Minkowski sum hull
```

[Source: Edelsbrunner "Algorithms in Combinatorial Geometry", Springer 1987; van den Bergen "Collision Detection in Interactive 3D Environments", Morgan Kaufmann 2004]

### 60.2 Minkowski Difference for GJK Collision

The GJK distance algorithm (§24) operates implicitly on the Minkowski difference A⊖B = A⊕(−B) via support functions. GPU parallelises GJK for a large batch of shape pairs simultaneously:

```glsl
// gjk_batch.comp — GJK distance for a batch of (convex A, convex B) pairs
layout(set=0,binding=0) readonly buffer ShapeA { ConvexShape shapesA[]; };
layout(set=0,binding=1) readonly buffer ShapeB { ConvexShape shapesB[]; };
layout(set=0,binding=2) writeonly buffer Dist  { float dist[]; };  // negative = penetrating
layout(local_size_x=64) in;
void main() {
    uint p  = gl_GlobalInvocationID.x;
    ConvexShape A = shapesA[p], B = shapesB[p];
    vec3 v  = A.centre - B.centre;
    vec3 simplex[4]; int nSimplex = 0;
    for (int iter=0; iter<64; iter++) {
        // Support on Minkowski difference: sup_{A⊖B}(v) = sup_A(v) - sup_B(-v)
        vec3 w  = support(A, v) - support(B, -v);
        if (dot(v,v) - dot(w,v) < 1e-8) { dist[p]=length(v); return; }
        addToSimplex(simplex, nSimplex, w);
        v = nearestSimplexPoint(simplex, nSimplex);
    }
    dist[p] = -1.0;  // penetrating
}
```

[Source: Gilbert et al. "A Fast Procedure for Computing the Distance Between Complex Objects in Three-Dimensional Space", IEEE T-RA 1988; §8]

### 60.3 Probabilistic Roadmap (PRM) on GPU

GPU-parallel PRM builds a configuration-space roadmap by sampling random configurations, testing each for collision (§60.2 GJK batch), and connecting collision-free neighbours:

```glsl
// prm_sample.comp — sample random configurations and test collision-free
layout(set=0,binding=0) readonly buffer BlueNoise { vec2 bn[]; };
layout(set=0,binding=1) readonly buffer Obstacles { ConvexShape obs[]; };
layout(set=0,binding=2) buffer Roadmap { vec3 configs[]; uint count; };
layout(push_constant) uniform PC { vec3 cMin; vec3 cRange; uint N_SAMPLE; uint N_OBS; } pc;
layout(local_size_x=64) in;
void main() {
    uint s   = gl_GlobalInvocationID.x;
    if (s >= pc.N_SAMPLE) return;
    vec3 q   = pc.cMin + vec3(bn[s*3%BN_SIZE],bn[(s*3+1)%BN_SIZE].x,bn[(s*3+2)%BN_SIZE].x)*pc.cRange;
    // Test q against all obstacles via GJK
    bool free = true;
    for (uint o=0; o<pc.N_OBS && free; o++) {
        ConvexShape robot = robotAt(q);  // robot shape at config q
        if (gjkPenetrating(robot, obs[o])) free=false;
    }
    if (free) { uint slot=atomicAdd(count,1u); configs[slot]=q; }
}
```

[Source: Kavraki et al. "Probabilistic Roadmaps for Path Planning in High-Dimensional Configuration Spaces", IEEE T-RA 1996; Lauterbach et al. "gProximity: Hierarchical GPU-Based Operations for Collision and Distance Queries", EG 2010]

### 60.4 RRT Nearest-Neighbour on GPU

Rapidly-exploring random tree (RRT) nearest-neighbour search — finding the closest existing tree node to a random sample — is the bottleneck. GPU parallelises the linear search across all tree nodes:

```glsl
// rrt_nearest.comp — find nearest tree node to random sample q_rand
layout(set=0,binding=0) readonly buffer Tree   { vec3 nodes[]; uint treeSize; };
layout(set=0,binding=1) readonly buffer QRand  { vec3 qRand;  };
layout(set=0,binding=2) buffer NearestIdx { uint idx; float dist; };
layout(local_size_x=256) in;
shared uint sIdx[256]; shared float sDist[256];
void main() {
    uint i  = gl_GlobalInvocationID.x;
    uint li = gl_LocalInvocationID.x;
    float d = (i < treeSize) ? length(nodes[i]-qRand) : 1e30;
    sIdx[li]=i; sDist[li]=d; barrier();
    for (uint s=128u;s>0u;s>>=1) {
        if (li<s && sDist[li+s]<sDist[li]) { sDist[li]=sDist[li+s]; sIdx[li]=sIdx[li+s]; }
        barrier();
    }
    if (li==0) { atomicMin_float_idx(dist, idx, sDist[0], sIdx[0]); }
}
```

[Source: LaValle "Rapidly-Exploring Random Trees: A New Tool for Path Planning", TR 1998; GPU RRT: Murray et al. "GPU Acceleration of the Fast Marching Method", Parallel Computing 2009]

---


---

## VII. SDF and Volumetric Methods

Signed Distance Functions provide a unified representation for implicit geometry, collision proximity queries, and volumetric rendering effects. This category covers level-set evolution equations for simulating moving interfaces (flame fronts, water surfaces), TSDF volumetric reconstruction from depth sensor streams, isosurface extraction algorithms (Marching Cubes, Dual Contouring) that recover triangle meshes from scalar fields, SDF-based proximity collision detection, volumetric ray marching for clouds and participating media, and GPU ambient occlusion estimated by ray marching against an SDF volume. The GPU sparse narrow-band level-set solver is the real-time counterpart to OpenVDB-style offline VFX pipelines.

### 61. Level Set Methods

*Audience: systems developers, graphics application developers.*

Level set methods (Osher & Sethian 1988) represent surfaces implicitly as the zero crossing of a scalar field φ(x,t) stored on a grid. Unlike JFA (§3.4), which constructs SDFs from point sources, LSM *evolves* the SDF forward in time using a PDE. Key applications: fluid free surfaces (§48 uses MAC grid + LSM hybrid), phase-field fracture (§52), topology-changing surface animation.

### 61.1 Level Set Advection

The fundamental LSM PDE for a velocity field **v**:

∂φ/∂t + **v** · ∇φ = 0

Discretise using semi-Lagrangian advection (same stencil as §48.1 MAC advection, but applied to the scalar SDF field):

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

### 61.2 SDF Reinitialization (Redistancing)

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

### 61.3 GPU Fast Marching Method (FMM)

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

### 61.4 Narrow-Band Level Sets

Only voxels within a band of width ±W·dx of φ=0 need updating. Compact active voxels with prefix scan (§52 dirty-bitmask pattern) before each sweep to avoid dispatching over the full grid. For a 256³ grid with W=3, this limits active voxels to ~5% of the grid at typical surface areas.

---

### 62. Swept Volumes and Minkowski Sums

*Audience: systems developers, graphics application developers.*

The swept volume of an object O moving along a trajectory T is the union of all instantaneous placements O(t). GPU computation is used in robot workspace analysis, CNC toolpath collision avoidance, and dynamic broad-phase proximity queries (§50.1 SAP needs swept AABBs for fast-moving objects).

### 62.1 Swept AABB for Broadphase

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

Feed swept AABBs into the SAP broadphase (§50.1) for CCD (Continuous Collision Detection).

### 62.2 Minkowski Sum of Convex Polyhedra

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

For non-convex objects: decompose via V-HACD (§42 Table A) then sum each convex part pair and union the resulting meshes. [Source: Dobkin et al. "Computing the Minkowski Sum of Convex Polytopes", Discrete & Computational Geometry 1993]

### 62.3 CNC Toolpath Swept Volume via SDF Dilation

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

### 63. Volumetric 3D Reconstruction (TSDF Fusion)

*Audience: systems developers, graphics application developers.*

Truncated Signed Distance Function (TSDF) fusion (Newcombe et al. "KinectFusion" 2011) integrates a stream of depth images into a persistent 3D SDF volume on the GPU in real time. Each new depth frame updates the TSDF; GPU marching cubes extracts the current surface mesh on demand.

### 63.1 Depth Map Ray Casting and TSDF Update

For each depth pixel (u,v) with depth z, back-project into world space, then update all voxels along the ray within truncation distance μ:

```glsl
// tsdf_integrate.comp — integrate one depth frame into TSDF volume
layout(set=0,binding=0) buffer TSDF  { float tsdf[];   };  // current SDF value
layout(set=0,binding=1) buffer Weight{ float weight[]; };  // running sum of weights
layout(set=0,binding=2) uniform sampler2D depthImage;
layout(push_constant) uniform PC {
    mat4 T_world_cam;    // camera pose in world
    mat4 K_inv;          // inverse camera intrinsics
    vec3 gridMin; float dx;
    ivec3 gridDim; float mu;   // truncation distance
} pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 vox = ivec3(gl_GlobalInvocationID);
    vec3  wp  = pc.gridMin + (vec3(vox)+0.5)*pc.dx;
    // Transform world point into camera space
    vec4  cp  = inverse(pc.T_world_cam) * vec4(wp, 1.0);
    if (cp.z <= 0.0) return;
    // Project to pixel
    vec3  pp  = (inverse(mat3(pc.K_inv)) * cp.xyz) / cp.z;
    vec2  uv  = pp.xy / vec2(IMAGE_W, IMAGE_H);
    if (any(lessThan(uv,vec2(0))) || any(greaterThan(uv,vec2(1)))) return;
    float Dmeas = texture(depthImage, uv).r;
    if (Dmeas <= 0.0) return;
    float sdf_val = Dmeas - cp.z;   // signed distance along ray
    if (sdf_val < -pc.mu) return;   // behind surface, too far
    float tsdf_val = clamp(sdf_val / pc.mu, -1.0, 1.0);
    uint  flat     = vox.z*pc.gridDim.y*pc.gridDim.x + vox.y*pc.gridDim.x + vox.x;
    float w        = 1.0;
    tsdf[flat]   = (tsdf[flat] * weight[flat] + tsdf_val * w) / (weight[flat] + w);
    weight[flat] += w;
}
```

[Source: Newcombe et al. "KinectFusion: Real-Time Dense Surface Mapping and Tracking", ISMAR 2011]

### 63.2 Camera Pose Tracking via ICP on TSDF Normal Field

Each incoming depth frame is aligned to the current TSDF by ICP (§87.3) against the TSDF surface normal field. Project current TSDF normals to a normal map, then run point-to-plane ICP between the new depth frame's point cloud and the TSDF normal map:

```glsl
// tsdf_normal.comp — extract surface normal from TSDF gradient
layout(set=0,binding=0) readonly buffer TSDF { float tsdf[]; };
layout(set=0,binding=1, rgba16f) uniform writeonly image3D normalVol;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 v = ivec3(gl_GlobalInvocationID);
    // Central difference gradient of TSDF → surface normal
    float dx = tsdf[flat(v+ivec3(1,0,0))] - tsdf[flat(v-ivec3(1,0,0))];
    float dy = tsdf[flat(v+ivec3(0,1,0))] - tsdf[flat(v-ivec3(0,1,0))];
    float dz = tsdf[flat(v+ivec3(0,0,1))] - tsdf[flat(v-ivec3(0,0,1))];
    imageStore(normalVol, v, vec4(normalize(vec3(dx,dy,dz)), 0.0));
}
```

### 63.3 Mesh Extraction from TSDF

Run GPU marching cubes (§3.2) on the TSDF volume at the zero crossing. For real-time KinectFusion the MC runs every frame on a 256³ grid in ~2 ms (RTX 3070); for offline processing a 512³ grid runs every 10th frame.

```glsl
// tsdf_mc.comp — marching cubes on TSDF (reuses §3.2 pattern)
// Sample tsdf[] at 8 cube corners; look up MC edge table; emit triangles.
// Vertices interpolated at zero crossing: v = lerp(a,b, tsdf[a]/(tsdf[a]-tsdf[b]))
```

[Source: Newcombe et al. 2011; Steinbrücker et al. "Real-Time Visual Odometry from Dense RGB-D Images", ICCVW 2011; open-source reference: Open3D KinectFusion https://www.open3d.org]

### 63.4 Voxel Hashing for Unbounded Reconstruction

The 256³ bounded grid of basic KinectFusion limits scene size. Voxel hashing (Nießner et al. 2013) stores only occupied voxel blocks in a GPU hash table:

```glsl
// vhashing_lookup.comp — voxel hash table lookup/insert
layout(set=0,binding=0) buffer HashTable { HashEntry table[HASH_SIZE]; };
uint hashKey(ivec3 blockCoord) {
    return (blockCoord.x*73856093u ^ blockCoord.y*19349663u ^ blockCoord.z*83492791u)
           % HASH_SIZE;
}
// On insert: linear probing with atomicCompSwap on entry.key
// On lookup: follow probe chain until key match or empty slot
```

[Source: Nießner et al. "Real-time 3D Reconstruction at Scale Using Voxel Hashing", SIGGRAPH Asia 2013]

---

### 64. SDF-Based Collision Detection

*Audience: graphics application developers, systems developers.*

§24 covered BVH/GJK collision for convex polyhedra. SDF-based collision detection uses the signed distance field representation directly: query the SDF at the penetrating point for depth and gradient direction, or run GJK over SDF-defined primitives. This is the standard approach for deformable body–rigid body coupling, character–environment collision, and SDF CSG contact.

### 64.1 Point-SDF Penetration Query

For a rigid body with vertices {vᵢ}, query each vertex against the scene SDF. If φ(vᵢ) < 0, the point is inside; depth = |φ(vᵢ)|, normal = ∇φ at vᵢ:

```glsl
// sdf_contact.comp — generate contacts for rigid body vertices vs scene SDF
layout(set=0,binding=0) readonly buffer Vertices { vec3 verts[]; };
layout(set=0,binding=1) uniform sampler3D sceneSDF;
layout(set=0,binding=2) buffer Contacts { ContactPoint contacts[]; };
layout(set=0,binding=3) buffer ContactCount { uint count; };
layout(push_constant) uniform PC { vec3 sdfMin; float sdfDx; mat4 bodyTransform; } pc;
layout(local_size_x=64) in;
void main() {
    uint i    = gl_GlobalInvocationID.x;
    vec3 wp   = (pc.bodyTransform * vec4(verts[i],1.0)).xyz;
    vec3 uvw  = (wp - pc.sdfMin) / (pc.sdfDx * SDF_DIM);
    if (any(lessThan(uvw,vec3(0)))||any(greaterThan(uvw,vec3(1)))) return;
    float phi = texture(sceneSDF, uvw).r;
    if (phi < 0.0) {
        // Normal from central-difference gradient of SDF
        vec3 n = normalize(vec3(
            textureOffset(sceneSDF,uvw,ivec3(1,0,0)).r - textureOffset(sceneSDF,uvw,ivec3(-1,0,0)).r,
            textureOffset(sceneSDF,uvw,ivec3(0,1,0)).r - textureOffset(sceneSDF,uvw,ivec3(0,-1,0)).r,
            textureOffset(sceneSDF,uvw,ivec3(0,0,1)).r - textureOffset(sceneSDF,uvw,ivec3(0,0,-1)).r
        ));
        uint slot = atomicAdd(count, 1u);
        contacts[slot] = ContactPoint(wp, n, -phi, i);
    }
}
```

[Source: Müller et al. "Unified Particle Physics for Real-Time Applications", SIGGRAPH 2014; Macklin et al. "XPBD: Position-Based Simulation of Compliant Constrained Dynamics", MIG 2016]

### 64.2 SDF-SDF Collision via Gradient Descent

When both objects are represented as SDFs, find the closest pair of surface points by gradient-following one SDF's zero-crossing in the other's field:

```glsl
// sdf_sdf_closest.comp — Newton iterate for SDF-SDF closest point
layout(local_size_x=64) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    vec3 p  = candidatePoint[i];  // initial seed near boundary
    for (int iter=0; iter<MAX_ITER; iter++) {
        float phiA = sampleSDF(sdfA, p);
        float phiB = sampleSDF(sdfB, p);
        vec3  nA   = sdfGradient(sdfA, p);
        vec3  nB   = sdfGradient(sdfB, p);
        if (abs(phiA) < TOL && abs(phiB) < TOL) break;
        // Step toward zero crossing of both SDFs simultaneously
        p -= (phiA * nA + phiB * nB) * 0.5;
    }
    closestPoint[i] = p;
    penetrationDepth[i] = -min(sampleSDF(sdfA,p), sampleSDF(sdfB,p));
}
```

### 64.3 SDF-Based Character–Terrain Collision

For character capsule vs heightfield terrain (§71.4), the SDF approach is exact and fast. Evaluate the terrain SDF at the capsule centre and hemispheres; push the capsule out by the penetration depth along the SDF gradient:

```glsl
// capsule_terrain.comp
layout(local_size_x=64) in;
void main() {
    uint c   = gl_GlobalInvocationID.x;
    vec3 base= capsule[c].base, tip = capsule[c].tip;
    float r  = capsule[c].radius;
    // Sample terrain SDF at 3 points along capsule axis
    for (int k=0; k<=2; k++) {
        vec3 p    = mix(base, tip, float(k)*0.5);
        float phi = terrainSDF(p);
        if (phi < r) {
            vec3 n = terrainNormal(p);
            float pen = r - phi;
            capsule[c].base += n * pen;
            capsule[c].tip  += n * pen;
        }
    }
}
```

[Source: Erleben et al. "Physics-Based Animation", Charles River Media 2005; Macklin et al. 2014]

### 64.4 EPA: Expanding Polytope Algorithm for SDF Primitives

When GJK (§24.4) determines that two SDF-defined convex primitives intersect, EPA finds the minimum penetration depth by expanding a polytope outward until it touches the Minkowski difference boundary. Each EPA iteration runs on GPU for independent object pairs:

```glsl
// epa_step.comp — one EPA polytope expansion step
layout(set=0,binding=0) buffer Polytope { vec3 verts[]; uint nVerts; };
layout(set=0,binding=1) buffer EPAFaces  { uvec3 faces[]; float dist[]; uint nFaces; };
layout(local_size_x=1) in;
void main() {
    // Find closest face (minimum distance to origin)
    uint best = 0;
    for (uint f=1; f<EPAFaces.nFaces; f++)
        if (dist[f] < dist[best]) best=f;
    // Support point of Minkowski diff in face normal direction
    vec3 n    = faceNormal(faces[best]);
    vec3 supp = supportSDF_A(n) - supportSDF_B(-n);
    if (dot(supp,n) - dist[best] < EPA_TOL) { penetrationNormal=n; penetrationDepth=dist[best]; return; }
    // Expand polytope: add supp, remove faces visible from supp, add horizon faces
    expandPolytope(supp);
}
```

[Source: Bergen "A Fast and Robust GJK Implementation for Collision Detection of Convex Objects", JGTOOLS 1999]

---

### 65. Isosurface Extraction

*Audience: graphics application developers, systems developers.*

Isosurface extraction converts a scalar field φ(x,y,z) into a triangle or quad mesh at a given isovalue τ. The field may be a density grid (CT/MRI), a level set (§61), a TSDF (§63), or an SDF (§64). Three GPU-native algorithms exist: Marching Cubes, Dual Contouring, and Surface Nets.

### 65.1 Marching Cubes on GPU

Marching Cubes (Lorensen & Cline 1987) classifies each grid cell by the sign pattern of its 8 corner values, looks up the triangle configuration (256-entry table), and emits triangles. GPU implementation uses a two-pass approach:

```glsl
// mc_classify.comp — pass 1: count triangles per cell
layout(set=0,binding=0) uniform sampler3D phi;  // scalar field
layout(set=0,binding=1) writeonly buffer TriCount { uint nTris[]; };
layout(set=0,binding=2) readonly buffer MCTable  { uint config[256]; };  // tri count table
layout(push_constant) uniform PC { float isoVal; ivec3 dim; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 c  = ivec3(gl_GlobalInvocationID);
    if (any(greaterThanEqual(c, pc.dim-1))) return;
    uint  ci = 0u;
    for (int k=0;k<8;k++) {
        ivec3 ofs = ivec3(k&1,(k>>1)&1,(k>>2)&1);
        float v   = texelFetch(phi, c+ofs, 0).r;
        if (v > pc.isoVal) ci |= (1u<<k);
    }
    nTris[flatIdx(c)] = bitCount(config[ci]);  // precomputed triangle count
}
// Pass 2: prefix-sum nTris → offset per cell, then emit vertices via edge interpolation
```

For pass 2, each cell looks up the edge intersection table, interpolates vertex positions on the 12 edges, and writes into a vertex buffer at the prefix-summed offset. [Source: Lorensen & Cline "Marching Cubes: A High Resolution 3D Surface Construction Algorithm", SIGGRAPH 1987; GPU MC: Nguyen "GPU Gems 3 — Chapter 1: Generating Complex Procedural Terrains", 2007]

### 65.2 Dual Contouring

Dual Contouring (Ju et al. 2002) places a vertex inside each active cell (one that straddles the isosurface) rather than on edges — producing sharp features by solving a quadratic error function (QEF) from the edge intersection normals:

```glsl
// dc_qef.comp — solve QEF per active cell to find dual vertex
layout(set=0,binding=0) readonly buffer Edges { EdgeIntersection ei[]; };
layout(set=0,binding=1) writeonly buffer DualVerts { vec3 dv[]; };
layout(local_size_x=64) in;
void main() {
    uint c  = gl_GlobalInvocationID.x;
    // Collect edge intersections for this cell (up to 12)
    vec3 A[12]; vec3 n[12]; int m=0;
    for (int e=0; e<12; e++) {
        EdgeIntersection x = getCellEdge(ei, c, e);
        if (x.active) { A[m]=x.pos; n[m]=x.normal; m++; }
    }
    if (m==0) { dv[c]=vec3(0); return; }
    // QEF: minimize sum_i (nᵢᵀ(x-Aᵢ))² — solve 3×3 normal equation
    mat3 ATA = mat3(0); vec3 ATb = vec3(0);
    for (int i=0;i<m;i++) {
        ATA += outerProduct(n[i],n[i]);
        ATb += dot(n[i],A[i]) * n[i];
    }
    dv[c] = clampToCell(inverse(ATA+1e-4*mat3(1))*ATb, c);
}
```

[Source: Ju et al. "Dual Contouring of Hermite Data", SIGGRAPH 2002; Schaefer & Warren "Dual Contouring: The Secret Sauce", 2002]

### 65.3 Surface Nets

Surface Nets (Gibson 1998) places a vertex at the centroid of all active edge intersection points in a cell — a simpler, smoother alternative to Dual Contouring, GPU-parallelizable per cell:

```glsl
// surface_nets.comp — place vertex at centroid of edge intersections
layout(set=0,binding=0) uniform sampler3D phi;
layout(set=0,binding=1) writeonly buffer SNVerts { vec3 snv[]; uint active[]; };
layout(push_constant) uniform PC { float isoVal; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 c  = ivec3(gl_GlobalInvocationID);
    vec3  sum = vec3(0.0); int cnt = 0;
    // 12 edges: check sign change, interpolate crossing position
    for (int e=0; e<12; e++) {
        ivec3 a = c + EDGE_A[e], b = c + EDGE_B[e];
        float va = texelFetch(phi,a,0).r, vb = texelFetch(phi,b,0).r;
        if ((va < pc.isoVal) != (vb < pc.isoVal)) {
            float t = (pc.isoVal - va) / (vb - va);
            sum += mix(vec3(a), vec3(b), t);
            cnt++;
        }
    }
    if (cnt > 0) { snv[flatIdx(c)] = sum/float(cnt); active[flatIdx(c)] = 1u; }
    else           active[flatIdx(c)] = 0u;
}
// Quad faces generated by connecting dual vertices of adjacent active cells
```

[Source: Gibson "Constrained Elastic Surface Nets: Generating Smooth Surfaces from Binary Segmented Data", MICCAI 1998; Naive Surface Nets implementation: https://0fps.net/2012/07/12/smooth-voxel-terrain/]

### 65.4 Adaptive Isosurface via SVO

For large sparse volumes, run marching cubes only on leaf nodes of the sparse voxel octree (§27.3) that straddle the isosurface. The SVO traversal is a BFS over active nodes:

```glsl
// mc_svo.comp — dispatch MC only on active SVO leaves near isosurface
layout(set=0,binding=0) readonly buffer SVONodes  { SVONode nodes[]; };
layout(set=0,binding=1) writeonly buffer ActiveLeaves { uint leaves[]; };
layout(set=0,binding=2) buffer LeafCount { uint count; };
layout(local_size_x=64) in;
void main() {
    uint n = gl_GlobalInvocationID.x;
    if (!nodes[n].isLeaf) return;
    // Node straddles isosurface if min and max corner values differ in sign
    if (nodes[n].phiMin < 0.0 && nodes[n].phiMax > 0.0) {
        uint slot = atomicAdd(count, 1u);
        leaves[slot] = n;
    }
}
// vkCmdDispatchIndirect with count → MC pass 1 (§65.1) over leaves only
```

[Source: Kazhdan et al. "Screened Poisson Surface Reconstruction", TOG 2013; SVO MC: Crassin & Green "Octree-Based Sparse Voxelization", GPU Pro 4, 2013]

---

### 66. GPU Ambient Occlusion

*Audience: graphics application developers.*

Ambient occlusion quantifies how much of the hemisphere above a surface point is occluded by nearby geometry — a geometric signal, not a lighting model. Four GPU algorithm families span the quality/performance spectrum: SSAO (screen-space), HBAO+ (horizon-based), GTAO (ground-truth AO), and precomputed baked AO (ray traced).

### 66.1 SSAO: Screen-Space Ambient Occlusion

SSAO (Crytek 2007) samples random positions in a hemisphere around each G-buffer point, tests each against the depth buffer, and counts occlusions:

```glsl
// ssao.comp — screen-space ambient occlusion
layout(set=0,binding=0) uniform sampler2D gDepth;
layout(set=0,binding=1) uniform sampler2D gNormal;
layout(set=0,binding=2) uniform sampler2D noiseTexture;  // 4×4 random rotation vectors
layout(set=0,binding=3) readonly buffer SSAOKernel { vec3 kernel[N_SAMPLES]; };
layout(set=0,binding=4, r8) uniform writeonly image2D aoOut;
layout(push_constant) uniform PC { mat4 proj; mat4 invProj; float radius; float bias; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    vec3  n    = texelFetch(gNormal, pix, 0).xyz;
    float d    = texelFetch(gDepth, pix, 0).r;
    vec3  pos  = viewSpacePos(pix, d, pc.invProj);
    // TBN from noise texture rotation
    vec3  randVec = texelFetch(noiseTexture, pix%4, 0).xyz;
    vec3  t    = normalize(randVec - n*dot(randVec,n));
    vec3  b    = cross(n, t);
    mat3  TBN  = mat3(t, b, n);
    float occ  = 0.0;
    for (uint s=0; s<N_SAMPLES; s++) {
        vec3  spos = pos + TBN * kernel[s] * pc.radius;
        vec4  clip = pc.proj * vec4(spos,1.0);
        vec2  suv  = clip.xy/clip.w*0.5+0.5;
        float sd   = texture(gDepth, suv).r;
        float sz   = viewSpaceZ(sd, pc.invProj);
        float rangeCheck = smoothstep(0.0,1.0,pc.radius/abs(pos.z-sz));
        occ += (sz >= spos.z + pc.bias ? 1.0 : 0.0) * rangeCheck;
    }
    imageStore(aoOut, pix, vec4(1.0 - occ/float(N_SAMPLES)));
}
```

[Source: Mittring "Finding Next Gen — CryEngine 2", SIGGRAPH 2007; Shanmugam & Arikan "Hardware Accelerated Ambient Occlusion Techniques on GPUs", I3D 2007]

### 66.2 HBAO+: Horizon-Based Ambient Occlusion

HBAO (Bavoil & Sainz 2008) marches along multiple screen-space directions from each pixel, finds the maximum elevation angle (horizon), and integrates the sine of the horizon to estimate AO. HBAO+ (NVIDIA 2012) adds interleaved sampling to reduce noise:

```glsl
// hbao.comp — horizon-based AO, one direction step
layout(set=0,binding=0) uniform sampler2D linearDepth;
layout(set=0,binding=1) uniform sampler2D gNormal;
layout(set=0,binding=2, r8) uniform writeonly image2D aoDir;
layout(push_constant) uniform PC { vec2 focalLen; float radius; float stepSize; int nDir; int nSteps; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv  = (vec2(pix)+0.5)/vec2(imageSize(aoDir));
    vec3  P   = reconstructView(uv, texture(linearDepth,uv).r);
    vec3  N   = texture(gNormal, uv).xyz;
    float ao  = 0.0;
    for (int d=0; d<pc.nDir; d++) {
        float angle = float(d)/float(pc.nDir) * 3.14159;
        vec2  dir   = vec2(cos(angle), sin(angle)) / pc.focalLen * pc.stepSize;
        float h1 = -1e9, h2 = -1e9;  // max horizon angles
        for (int s=1; s<=pc.nSteps; s++) {
            vec3 Q1 = reconstructView(uv+dir*float(s), texture(linearDepth,uv+dir*float(s)).r);
            vec3 Q2 = reconstructView(uv-dir*float(s), texture(linearDepth,uv-dir*float(s)).r);
            vec3 v1 = Q1-P, v2 = Q2-P;
            float d1 = length(v1), d2 = length(v2);
            if (d1 < pc.radius) h1 = max(h1, dot(normalize(v1),N)/d1);
            if (d2 < pc.radius) h2 = max(h2, dot(normalize(v2),N)/d2);
        }
        ao += 1.0 - 0.5*(sin(asin(h1))+sin(asin(h2)));
    }
    imageStore(aoDir, pix, vec4(ao/float(pc.nDir)));
}
```

[Source: Bavoil & Sainz "Image-Space Horizon-Based Ambient Occlusion", SIGGRAPH 2008 Sketches; NVIDIA HBAO+ SDK]

### 66.3 GTAO: Ground-Truth Ambient Occlusion

GTAO (Jimenez et al. 2016) corrects the hemisphere integration formula so that the result converges to ray-traced AO as sample count increases — unlike SSAO and HBAO which use ad-hoc approximations. The bent normal (the direction of least occlusion) is also computed:

```glsl
// gtao.comp — ground-truth AO integration (visibility function integral)
// Each slice integrates visibility along a screen-space direction using
// the exact formula: AO = 1 - (1/π) ∫₀^π ∫_h1^h2 cos(θ-γ) dθ dφ
// where γ = projected normal angle, h1/h2 = horizon angles per direction
float integrateArc(float h1, float h2, float n) {
    // Analytical integral of the visibility function
    float sinN = sin(n);
    return (-cos(2*h1-n)+cos(n)+2*h1*sinN - (-cos(2*h2-n)+cos(n)+2*h2*sinN)) * 0.25;
}
// main: for each direction slice, compute h1, h2, call integrateArc, accumulate
```

[Source: Jimenez et al. "Practical Real-Time Strategies for Accurate Indirect Occlusion", SIGGRAPH 2016; XeGTAO implementation: https://github.com/GameTechDev/XeGTAO]

### 66.4 Ray-Traced AO and Bent Normals

With `VK_KHR_ray_query`, AO becomes a direct ray cast from each G-buffer point into the hemisphere, returning a binary hit/miss:

```glsl
// rtao.comp — ray-traced ambient occlusion via ray query
#extension GL_EXT_ray_query : require
layout(set=0,binding=0) uniform accelerationStructureEXT tlas;
layout(set=0,binding=1) uniform sampler2D gWorldPos;
layout(set=0,binding=2) uniform sampler2D gNormal;
layout(set=0,binding=3, r16f) uniform writeonly image2D aoOut;
layout(set=0,binding=4) readonly buffer Halton { vec2 halton[]; };
layout(push_constant) uniform PC { uint frameIdx; float radius; int nRays; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    vec3  P   = texelFetch(gWorldPos, pix, 0).xyz;
    vec3  N   = texelFetch(gNormal,   pix, 0).xyz;
    float ao  = 0.0;
    uint seed = pix.y*WIDTH+pix.x + pc.frameIdx*WIDTH*HEIGHT;
    for (int r=0; r<pc.nRays; r++) {
        vec3 dir = cosineSampleHemisphere(N, halton[seed%1024+r]);
        rayQueryEXT rq;
        rayQueryInitializeEXT(rq, tlas, gl_RayFlagsTerminateOnFirstHitEXT,
            0xFF, P+N*1e-3, 0.0, dir, pc.radius);
        rayQueryProceedEXT(rq);
        ao += (rayQueryGetIntersectionTypeEXT(rq,true)
               == gl_RayQueryCommittedIntersectionNoneEXT) ? 1.0 : 0.0;
    }
    imageStore(aoOut, pix, vec4(ao/float(pc.nRays)));
}
```

[Source: Stachowiak "Stochastic All The Things: Raytracing in Hybrid Real-Time Rendering", Digital Dragons 2018]

---

### 67. Volumetric Ray Marching

*Audience: graphics application developers.*

Volumetric rendering integrates light scattering and absorption through a participating medium — clouds, fog, smoke, fire, subsurface scattering in skin. The GPU algorithm is ray marching through a 3D volume from the near plane to the far plane or until the accumulated opacity reaches 1. This is distinct from TSDF (§63, geometry reconstruction) and level sets (§61, surface evolution).

### 67.1 Basic Ray Marching

Ray march from the eye through a 3D density volume, accumulating scattered radiance and transmitted light:

```glsl
// raymarch.frag — single-pass volumetric ray march
uniform sampler3D density;  // 3D density field
uniform sampler3D temperature;  // for fire/emission
uniform vec3 lightDir; uniform vec3 lightColor;
uniform float stepSize; uniform int maxSteps;
uniform float sigmaA; uniform float sigmaS;  // absorption and scattering coefficients
in vec3 rayOrigin, rayDir;
void main() {
    vec3  P   = rayOrigin;
    float T   = 1.0;  // transmittance
    vec3  L   = vec3(0.0);
    for (int i=0; i<maxSteps && T > 0.01; i++) {
        P += rayDir * stepSize;
        if (!inBounds(P)) break;
        vec3  uvw = worldToUVW(P);
        float rho = texture(density, uvw).r;
        if (rho < 1e-5) continue;
        float sigT = (sigmaA + sigmaS) * rho;
        float dT   = exp(-sigT * stepSize);
        // Shadow ray (simplified: single shadow sample)
        float shadow = shadowMarch(P, lightDir, density, sigT);
        // Phase function (Henyey-Greenstein)
        float cosTheta = dot(rayDir, -lightDir);
        float phase = henyeyGreenstein(cosTheta, 0.6);
        vec3  emit  = texture(temperature, uvw).r > 0.5 ?
                      blackbody(texture(temperature,uvw).r * 2000.0) : vec3(0.0);
        L  += T * (1.0-dT) * (lightColor * phase * shadow * sigmaS + emit);
        T  *= dT;
    }
    fragColor = vec4(L, 1.0-T);
}
```

[Source: Wrenninge & Bin Zafar "Production Volume Rendering", SIGGRAPH 2011; Fong et al. "Production Volume Rendering: Design and Implementation", SIGGRAPH 2017 Course]

### 67.2 Froxel Volumetrics

Froxels (frustum voxels, Hillaire 2016) discretize the camera frustum into a 3D grid in clip space — typically 160×90×64. A compute shader fills density/scattering into the froxel grid, then ray marches through it in-place:

```glsl
// froxel_fill.comp — populate froxel density from world-space volumes
layout(set=0,binding=0, rgba16f) uniform image3D froxelGrid;  // R=density G=emission B=scatter
layout(push_constant) uniform PC { mat4 invViewProj; vec3 camPos; float nearZ; float farZ; int D; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=4) in;
void main() {
    ivec3 fr  = ivec3(gl_GlobalInvocationID);
    vec3  ndc = vec3(vec2(fr.xy)/vec2(160,90)*2.0-1.0,
                     froxelDepth(fr.z, pc.nearZ, pc.farZ, pc.D));
    vec4  wp  = pc.invViewProj * vec4(ndc,1.0);
    wp /= wp.w;
    // Evaluate all registered volume primitives at world position wp.xyz
    float density = evalVolumes(wp.xyz);  // sum of Gaussian/exponential fog volumes
    imageStore(froxelGrid, fr, vec4(density, emissionAt(wp.xyz), scatterAt(wp.xyz), 0.0));
}
// Second pass: ray march through froxel grid (128 cells per screen-space ray)
```

[Source: Hillaire "Physically Based Sky, Atmosphere and Cloud Rendering in Frostbite", SIGGRAPH 2016; Wronski "Volumetric Fog: Unified, Compute Shader Based Solution to Atmospheric Scattering", ACM Digital Library 2014]

### 67.3 Atmospheric Scattering

Physically based sky rendering (Bruneton & Neyret 2008) precomputes multiple-scattering LUTs (transmittance, scattering, irradiance) and samples them at runtime for sky colour and aerial perspective. The precomputation runs as a series of 2D/3D compute passes:

```glsl
// atmos_transmittance.comp — precompute transmittance LUT T(h, μ)
// h = altitude, μ = cos(view-zenith angle)
layout(set=0,binding=0, rg16f) uniform writeonly image2D transLUT;
layout(push_constant) uniform PC { float Rearth; float Ratmos; float Hr; float Hm; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 id = ivec2(gl_GlobalInvocationID.xy);
    float h  = pc.Rearth + float(id.y)/LUT_H * (pc.Ratmos-pc.Rearth);
    float mu = float(id.x)/LUT_W * 2.0 - 1.0;
    // Integrate optical depth along view ray from (h,μ) to atmosphere boundary
    float optR = 0.0, optM = 0.0;
    float ds   = rayLengthAtmos(h, mu, pc.Rearth, pc.Ratmos) / STEPS;
    for (int s=0; s<STEPS; s++) {
        float r = length(vec2(h,0)+vec2(mu,sqrt(1-mu*mu))*(s+0.5)*ds);
        optR += exp(-(r-pc.Rearth)/pc.Hr) * ds;
        optM += exp(-(r-pc.Rearth)/pc.Hm) * ds;
    }
    imageStore(transLUT, id, vec4(exp(-betaR*optR-betaM*optM*1.1)));
}
```

[Source: Bruneton & Neyret "Precomputed Atmospheric Scattering", CGF 2008; Hillaire "A Scalable and Production Ready Sky and Atmosphere Rendering Technique", EG 2020]

### 67.4 VDB Volume Traversal for Rendering

OpenVDB / NanoVDB sparse volumes (§27 SVO ancestry) are traversed for rendering via DDA (Digital Differential Analyzer) through the active voxel tree:

```glsl
// nanovdb_march.comp — ray march through NanoVDB sparse volume
// NanoVDB buffer is uploaded as an SSBO; tree traversal uses the NanoVDB C++ API
// compiled via glslang extension or CUDA interop
layout(set=0,binding=0) readonly buffer NVDBBuffer { uint nvdb[]; };
layout(set=0,binding=1, rgba16f) uniform writeonly image2D renderOut;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    vec3  ro   = cameraPos;
    vec3  rd   = normalize(cameraRay(pix));
    float T    = 1.0;  vec3 L = vec3(0.0);
    // DDA traverse: skip empty nodes in O(log N) per step
    NVDBAccessor acc = createAccessor(nvdb);
    float t = 0.0;
    while (t < MAX_DIST && T > 0.01) {
        vec3  p  = ro + rd*t;
        float rho= nvdbSampleFloat(acc, p);
        float dt = rho > 0.0 ? DENSE_STEP : nextEmptyBoundary(acc, p, rd);
        if (rho > 1e-5) {
            float dT = exp(-rho * dt * SIGMA_T);
            L += T*(1-dT)*scatterLight(p, rd);
            T *= dT;
        }
        t += dt;
    }
    imageStore(renderOut, pix, vec4(L, 1.0-T));
}
```

[Source: Museth "NanoVDB: A GPU-Friendly and Portable VDB Data Structure for Real-Time Rendering and Simulation", SIGGRAPH 2021; https://github.com/AcademySoftwareFoundation/openvdb]

---

### 68. GPU Sparse Narrow-Band Level-Set Evolution

*Audience: systems developers, graphics application developers.*

Dense level sets (§61) compute on every voxel; the interface occupies only the narrow band |φ| ≤ 3Δx. Sparse narrow-band representations (VDB/OpenVDB §96.1) store only active voxels, enabling much finer grids for the same memory budget. GPU sparse level sets maintain an active list and update only band voxels per timestep.

### 68.1 Active Voxel List Construction

At each timestep, collect all voxels within the narrow band |φ| ≤ k·Δx into a compact active list:

```glsl
// narrowband_collect.comp — collect active voxels into compact list
layout(set=0,binding=0) readonly buffer Phi    { float phi[];  };
layout(set=0,binding=1) buffer ActiveList { uint active[]; uint count; };
layout(push_constant) uniform PC { ivec3 dim; float dx; float bandWidth; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 c = ivec3(gl_GlobalInvocationID);
    if (any(greaterThanEqual(c,pc.dim))) return;
    uint idx = c.z*pc.dim.y*pc.dim.x + c.y*pc.dim.x + c.x;
    if (abs(phi[idx]) <= pc.bandWidth * pc.dx) {
        uint slot = atomicAdd(count, 1u);
        active[slot] = idx;
    }
}
```

[Source: Museth et al. "OpenVDB: An Open Source Data Structure and Toolkit for High-Resolution Volumes", SIGGRAPH 2013; Nielsen & Museth "Dynamic Tubular Grid: An Efficient Data Structure and Algorithms for High Resolution Level Sets", JCGT 2006]

### 68.2 Narrow-Band Reinitialization via Fast Sweeping

Reinitialise the SDF to exact distance values within the narrow band using a fast sweeping / Godunov scheme applied only to active voxels:

```glsl
// narrowband_reinit.comp — Godunov upwind reinitialization (one sweep pass)
layout(set=0,binding=0) readonly buffer ActiveList { uint active[]; uint count; };
layout(set=0,binding=1) buffer Phi { float phi[]; };
layout(push_constant) uniform PC { ivec3 dim; float dx; int sweep; } pc;
layout(local_size_x=64) in;
void main() {
    uint li  = gl_GlobalInvocationID.x;
    if (li >= count) return;
    uint idx = active[li];
    ivec3 c  = unflatten(idx, pc.dim);
    // Godunov upwind: pick upwind neighbour in each axis
    float px = minNeighbour(phi, c, ivec3(1,0,0), pc.dim, pc.sweep);
    float py = minNeighbour(phi, c, ivec3(0,1,0), pc.dim, pc.sweep);
    float pz = minNeighbour(phi, c, ivec3(0,0,1), pc.dim, pc.sweep);
    float d  = solveGodunovEikonal(px, py, pz, pc.dx);
    phi[idx] = sign(phi[idx]) * min(abs(phi[idx]), d);
}
```

[Source: Zhao "A Fast Sweeping Method for Eikonal Equations", Mathematics of Computation 2004; Bridson "Fluid Simulation for Computer Graphics", 2008 §3]

### 68.3 Level-Set Advection on Active Voxels

Advect the level set under a velocity field using a 5th-order WENO scheme for the spatial derivative and TVD-RK3 for time integration, applied only to active voxels:

```glsl
// narrowband_advect.comp — WENO5 spatial derivative, Euler time step on active voxels
layout(set=0,binding=0) readonly buffer Vel      { vec3 vel[];  };
layout(set=0,binding=1) readonly buffer PhiIn    { float phiIn[]; };
layout(set=0,binding=2) writeonly buffer PhiOut  { float phiOut[]; };
layout(set=0,binding=3) readonly buffer ActiveList { uint active[]; uint count; };
layout(push_constant) uniform PC { ivec3 dim; float dx; float dt; } pc;
layout(local_size_x=64) in;
void main() {
    uint li  = gl_GlobalInvocationID.x;
    if (li >= count) return;
    uint idx = active[li];
    ivec3 c  = unflatten(idx, pc.dim);
    vec3  v  = vel[idx];
    // WENO5 upwind: select direction per-axis based on sign of velocity
    float dpx = (v.x>0) ? weno5Minus(phiIn, c, ivec3(1,0,0), pc.dim, pc.dx)
                        : weno5Plus (phiIn, c, ivec3(1,0,0), pc.dim, pc.dx);
    float dpy = (v.y>0) ? weno5Minus(phiIn, c, ivec3(0,1,0), pc.dim, pc.dx)
                        : weno5Plus (phiIn, c, ivec3(0,1,0), pc.dim, pc.dx);
    float dpz = (v.z>0) ? weno5Minus(phiIn, c, ivec3(0,0,1), pc.dim, pc.dx)
                        : weno5Plus (phiIn, c, ivec3(0,0,1), pc.dim, pc.dx);
    phiOut[idx] = phiIn[idx] - pc.dt * (v.x*dpx + v.y*dpy + v.z*dpz);
}
```

[Source: Osher & Fedkiw "Level Set Methods and Dynamic Implicit Surfaces", Springer 2003; Jiang & Peng "Weighted ENO Schemes on Triangular Meshes", JCP 2000]

### 68.4 Topology Change and Band Rebuild

After each advection step, check whether previously inactive voxels have entered the band (zero crossing propagation) and add them to the active list:

```glsl
// narrowband_expand.comp — add newly active voxels (|phi|<band) to active list
layout(set=0,binding=0) readonly buffer Phi { float phi[]; };
layout(set=0,binding=1) buffer ActiveList { uint active[]; uint count; };
layout(set=0,binding=2) buffer ActiveMask { uint mask[];  };  // 1 if currently active
layout(push_constant) uniform PC { ivec3 dim; float dx; float band; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 c = ivec3(gl_GlobalInvocationID);
    if (any(greaterThanEqual(c,pc.dim))) return;
    uint idx = flatIdx(c,pc.dim);
    if (mask[idx]!=0u) return;  // already active
    if (abs(phi[idx]) <= pc.band*pc.dx) {
        mask[idx]=1u;
        uint slot=atomicAdd(count,1u);
        active[slot]=idx;
    }
}
```

[Source: Museth et al. 2013; Houston et al. "A Unified Particle/Grid Fluid Simulation Algorithm", Proceedings of Graphics Interface 2006]

---


---

## VIII. Terrain, Procedural Content, and Environment

Large-scale environment geometry requires specialized LOD, streaming, and simulation approaches. This category covers GPU terrain rendering with quadtree or clipmap LOD and streaming from disk, hydraulic erosion simulation that produces naturalistic height fields, ocean wave synthesis via inverse FFT of a Philips/JONSWAP spectrum, GPU geospatial processing for satellite-scale terrain meshes, procedural geometry generation via compute shaders (L-systems, instanced geometry), particle and ribbon geometry systems for VFX, and navigation mesh construction for AI pathfinding on dynamically generated terrain. All of these share a common challenge: representing continuous, kilometre-scale natural phenomena efficiently on GPU.

### 69. Procedural Geometry Generation

*Audience: graphics application developers.*

Procedural generation creates geometry analytically rather than from stored mesh data, enabling effectively infinite variation at GPU-native speed. The common pattern: a compute shader writes vertex/index data into a pre-allocated SSBO that is then consumed directly by the rasterizer — no CPU roundtrip.

### 69.1 GPU Terrain: Fractional Brownian Motion and Erosion

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

### 69.2 L-Systems in Compute Shaders

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

### 69.3 Wave Function Collapse for Tile Geometry

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

### 69.4 Task Shader Procedural Instancing

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

### 70. Navigation Meshes and GPU Pathfinding

*Audience: graphics application developers.*

Navigation mesh (navmesh) generation and pathfinding are geometry algorithms in disguise: voxelisation is a rasterization problem, region labelling is a BFS problem on a graph, and flow field computation is a parallel shortest-path problem on a grid — all map to Vulkan compute naturally.

### 70.1 Voxelisation for NavMesh Generation (Recast-style)

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

### 70.2 Parallel Region Labelling via GPU BFS

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

### 70.3 GPU Flow Fields for Navigation

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

### 71. Terrain LOD and Streaming

*Audience: graphics application developers.*

§69 covered procedural terrain *generation*. This section covers runtime terrain *rendering*: quadtree LOD selection, geomorphing to prevent popping, virtual texture streaming, and GPU heightfield collision.

### 71.1 CDLOD: Continuous Distance-Dependent Level of Detail

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

### 71.2 Geomorphing to Prevent LOD Popping

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

### 71.3 Virtual Terrain Texturing: Sparse Clipmaps

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

### 71.4 GPU Heightfield Ray Intersection

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

### 72. GPU Particle Systems and Ribbon Geometry

*Audience: graphics application developers.*

Particle systems are the standard GPU geometry for effects (fire, smoke, sparks, rain, magic). Unlike §48 physics-driven SPH, this section covers the *rendering geometry pipeline*: how particles are born, evolved, compacted, and converted into drawable primitives each frame.

### 72.1 Particle Lifecycle: Emit, Age, Kill

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

### 72.2 Billboard Expansion in Mesh Shaders

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

### 72.3 Ribbon / Trail Geometry via Rolling Ring Buffer

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

### 72.4 Soft Particles via Depth Comparison

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

### 73. GPU Ocean and Wave Simulation

*Audience: graphics application developers.*

Ocean rendering combines two complementary techniques: an FFT-based statistical wave model (Tessendorf 2001) for large-scale height-field waves, and Gerstner (trochoidal) waves for deterministic art-directed swells. Both produce vertex-displacement maps consumed by a tessellated grid each frame.

### 73.1 Phillips Spectrum and Initial Conditions

The Phillips spectrum P(k) models the energy distribution of ocean waves as a function of wave vector **k**:

```
P(k) = A · exp(-1/(k·L)²) / k⁴ · |k̂ · ŵ|²
```

where L = V²/g (Pierson-Moskowitz limit speed), V = wind speed, ŵ = wind direction. GPU initialisation generates the complex amplitude h₀(**k**) by sampling Gaussian random numbers and weighting by √P(k):

```glsl
// ocean_init.comp — initialise complex spectrum h₀(k)
layout(set=0,binding=0, rg32f) uniform writeonly image2D h0;   // complex amplitudes
layout(set=0,binding=1) uniform sampler2D gaussianNoise;       // pre-generated Gaussian pairs
layout(push_constant) uniform PC {
    float A; vec2 windDir; float windSpeed; float g;
    int N; float L;  // grid size N×N, patch size L metres
} pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 id  = ivec2(gl_GlobalInvocationID.xy);
    vec2  k   = (vec2(id) - float(pc.N)*0.5) * (2.0*3.14159265/pc.L);
    float km  = max(length(k), 1e-6);
    float kL  = pc.windSpeed * pc.windSpeed / pc.g;
    float ph  = pc.A * exp(-1.0/(km*kL)*(km*kL)) / (km*km*km*km);
    float kw  = dot(normalize(k), pc.windDir);
    ph *= kw * kw;
    if (kw < 0.0) ph *= 0.07;   // damp waves against wind
    vec2 xi = texture(gaussianNoise, vec2(id)/float(pc.N)).rg;
    imageStore(h0, id, vec4(xi * sqrt(ph * 0.5), 0, 0));
}
```

[Source: Tessendorf "Simulating Ocean Water", SIGGRAPH Course Notes 2001]

### 73.2 Time Evolution and IFFT

Each frame, propagate h₀ to h(**k**,t) using the dispersion relation ω(k) = √(g·|**k**|):

```glsl
// ocean_update.comp — time-evolve spectrum, prepare IFFT inputs
layout(set=0,binding=0) readonly buffer H0    { vec2 h0[];    };
layout(set=0,binding=1) writeonly buffer Hkt  { vec2 hkt[];   };  // height
layout(set=0,binding=2) writeonly buffer Dxt  { vec2 dxt[];   };  // choppy X
layout(set=0,binding=3) writeonly buffer Dzt  { vec2 dzt[];   };  // choppy Z
layout(push_constant) uniform PC { float t; float g; float L; int N; } pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 id  = ivec2(gl_GlobalInvocationID.xy);
    int   idx = id.y * pc.N + id.x;
    vec2  k   = (vec2(id) - float(pc.N)*0.5) * (2.0*3.14159265/pc.L);
    float km  = max(length(k), 1e-6);
    float w   = sqrt(pc.g * km);
    // Euler formula: e^{iwt} = cos(wt) + i·sin(wt)
    float c   = cos(w * pc.t), s = sin(w * pc.t);
    vec2  h   = h0[idx];
    vec2  hc  = vec2(h.x, -h.y);   // conjugate of h0(-k)
    vec2  ht  = vec2(h.x*c - h.y*s, h.x*s + h.y*c)
              + vec2(hc.x*c + hc.y*s, -hc.x*s + hc.y*c);
    hkt[idx]  = ht;
    vec2 kn   = k / km;
    dxt[idx]  = vec2(-ht.y * kn.x,  ht.x * kn.x);  // ik_x · h(k,t)
    dzt[idx]  = vec2(-ht.y * kn.y,  ht.x * kn.y);  // ik_z · h(k,t)
}
```

Apply a 2D IFFT to `hkt`, `dxt`, `dzt` using a GPU FFT library (VkFFT, §73.5) to obtain height field H(x,z,t) and choppy displacement (Dx, Dz).

### 73.3 Choppy Waves via Jacobian

Horizontal displacement (choppiness) moves water particles toward wave crests, sharpening peaks. The Jacobian J of the displacement field detects wave breaking (J < 0 → foam):

```glsl
// ocean_jacobian.comp — compute Jacobian for foam mask
layout(set=0,binding=0) readonly buffer Dx { float dx[]; };
layout(set=0,binding=1) readonly buffer Dz { float dz[]; };
layout(set=0,binding=2, r8) uniform writeonly image2D foamMask;
layout(push_constant) uniform PC { int N; float L; float choppiness; } pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 id = ivec2(gl_GlobalInvocationID.xy);
    int   w  = pc.N;
    ivec2 nx = ivec2((id.x+1)%w, id.y), px = ivec2((id.x-1+w)%w, id.y);
    ivec2 nz = ivec2(id.x, (id.y+1)%w), pz = ivec2(id.x, (id.y-1+w)%w);
    float dDx_dx = (dx[nz.y*w+nx.x] - dx[pz.y*w+px.x]) / (2.0 * pc.L / float(w));
    float dDz_dz = (dz[nz.y*w+nz.x] - dz[pz.y*w+pz.x]) / (2.0 * pc.L / float(w));
    float J = (1.0 + pc.choppiness*dDx_dx) * (1.0 + pc.choppiness*dDz_dz);
    float foam = clamp(1.0 - J, 0.0, 1.0);
    imageStore(foamMask, id, vec4(foam));
}
```

[Source: Tessendorf 2001; Bruneton & Neyret "Real-time Realistic Ocean Lighting", EuroGraphics 2010]

### 73.4 Gerstner Wave Summation

For art-directed waves, sum N Gerstner (trochoidal) waves analytically — no FFT required, suitable for 4–8 waves with precise control:

```glsl
// ocean.vert — Gerstner wave displacement in vertex shader
struct GerstnerWave { vec2 dir; float amp, freq, phase, steep; };
layout(set=0,binding=0) readonly buffer Waves { GerstnerWave waves[MAX_WAVES]; };
layout(push_constant) uniform PC { int N; float time; } pc;
void main() {
    vec3 pos = inPos;
    vec3 nor = vec3(0,1,0);
    for (int i=0; i<pc.N; i++) {
        GerstnerWave w = waves[i];
        float theta = dot(w.dir, pos.xz)*w.freq + pc.time*w.phase;
        float s = sin(theta), c = cos(theta);
        pos.xz += w.steep * w.amp * w.dir * c;
        pos.y  += w.amp * s;
        nor.xz -= w.dir * w.freq * w.amp * c;
        nor.y  -= w.steep * w.freq * w.amp * s;
    }
    gl_Position = vp * vec4(pos, 1.0);
    vNormal     = normalize(nor);
}
```

[Source: Finch "Effective Water Simulation from Physical Models", GPU Gems 1, Ch. 1]

### 73.5 VkFFT Integration

VkFFT ([source](https://github.com/DTolm/VkFFT), MIT) provides Vulkan-native FFT on the GPU — the compute backbone for §73.2. It generates SPIR-V at runtime for the target device:

```cpp
VkFFTApplication app{};
VkFFTConfiguration cfg{};
cfg.FFTdim        = 2;
cfg.size[0]       = N; cfg.size[1] = N;
cfg.device        = &device;
cfg.queue         = &computeQueue;
cfg.isInputFormatted = 1;
cfg.bufferSize    = &bufferSize;
cfg.buffer        = &hktBuffer;
VkFFTResult res   = initializeVkFFT(&app, cfg);
// Each frame:
VkFFTLaunchParams lp{}; lp.commandBuffer = cmd; lp.inputBuffer = &hktBuffer;
VkFFTAppend(&app, -1, &lp);  // inverse FFT
```

[Source: Tolmachev "VkFFT — A Performant, Cross-Platform and Open-Source GPU FFT Library", 2021]

---

### 74. GPU Geospatial Processing: Large-Scale Terrain

*Audience: systems developers, graphics application developers.*

Real-world terrain from LIDAR point clouds or satellite DEMs requires GPU processing pipelines distinct from procedural terrain (§71): reprojection from geographic coordinates (WGS84/UTM), sparse tiling, massive point cloud decimation, and multi-level terrain mesh streaming. Applications include mapping, simulation, autonomous vehicle testing, and digital twins.

### 74.1 WGS84 to ECEF and Local Frame Projection

Reproject geographic coordinates (longitude, latitude, altitude) to Earth-Centred Earth-Fixed (ECEF) Cartesian coordinates, then to a local ENU (East-North-Up) frame for rendering:

```glsl
// geo_project.comp — WGS84 → ECEF → local ENU frame
layout(set=0,binding=0) readonly buffer LLA  { vec3 lla[];  };  // lon,lat,alt in degrees/metres
layout(set=0,binding=1) writeonly buffer ECEF { vec3 ecef[]; };
layout(push_constant) uniform PC { vec3 origin_LLA; uint N; } pc;  // local origin for ENU
layout(local_size_x=64) in;
const float a = 6378137.0;           // WGS84 semi-major
const float e2 = 0.00669437999014;   // eccentricity²
void main() {
    uint i   = gl_GlobalInvocationID.x;
    if (i >= pc.N) return;
    float lat = radians(lla[i].y), lon = radians(lla[i].x), alt = lla[i].z;
    float N_  = a / sqrt(1.0 - e2*sin(lat)*sin(lat));
    ecef[i]   = vec3((N_+alt)*cos(lat)*cos(lon),
                     (N_+alt)*cos(lat)*sin(lon),
                     (N_*(1.0-e2)+alt)*sin(lat));
}
// Second pass: rotate ECEF to local ENU using rotation matrix from origin LLA
```

[Source: Bowring "Transformation from Spatial to Geographic Coordinates", Survey Review 1976; EPSG:4978 WGS84 specification]

### 74.2 GPU LIDAR Point Cloud Decimation

Reduce massive LIDAR datasets (billions of points) to a renderable density using a GPU voxel-grid filter: keep one point per grid cell, selected by highest return intensity:

```glsl
// lidar_decimate.comp — voxel-grid decimation of LIDAR point cloud
layout(set=0,binding=0) readonly buffer Points { vec4 pts[]; };  // xyz + intensity
layout(set=0,binding=1) buffer VoxelBest { vec4 best[]; };       // best point per voxel
layout(set=0,binding=2) buffer VoxelInt  { float bestInt[]; };   // best intensity per voxel
layout(push_constant) uniform PC { vec3 gridMin; float cellSize; ivec3 dim; uint N; } pc;
layout(local_size_x=64) in;
void main() {
    uint p  = gl_GlobalInvocationID.x;
    if (p >= pc.N) return;
    ivec3 c = ivec3((pts[p].xyz - pc.gridMin) / pc.cellSize);
    if (any(lessThan(c,ivec3(0)))||any(greaterThanEqual(c,pc.dim))) return;
    uint idx = c.z*pc.dim.y*pc.dim.x + c.y*pc.dim.x + c.x;
    // Atomic max on intensity to select best return per cell
    atomicMax_float_idx(bestInt[idx], /* index field in best[] */ p, pts[p].w, p);
    // After: compact best[] to output cloud (see §72.2 prefix-scan pattern)
}
```

[Source: Rusu & Cousins "3D is Here: Point Cloud Library", ICRA 2011; Hornung et al. "OctoMap: An Efficient Probabilistic 3D Mapping Framework", Autonomous Robots 2013]

### 74.3 GPU Terrain Mesh Streaming with CDLOD

Continuous Distance-Dependent Level of Detail (CDLOD, Strugar 2010) generates terrain meshes from a height map hierarchy on-the-fly with no precomputed meshes. The GPU produces one quad-grid per visible tile at the appropriate LOD:

```glsl
// cdlod_terrain.vert — CDLOD terrain vertex: sample heightmap with LOD blend
layout(push_constant) uniform PC { vec2 tileOffset; float tileScale; int lod;
    vec3 cameraPos; mat4 mvp; } pc;
uniform sampler2D heightmapMip;   // mip chain of heightmap
in vec2 vertexUV;                 // grid vertex [0,1]²
void main() {
    vec2 worldXZ  = pc.tileOffset + vertexUV * pc.tileScale;
    // Sample two LOD levels and blend based on camera distance
    float h0 = textureLod(heightmapMip, worldXZ/TERRAIN_SIZE, float(pc.lod)).r;
    float h1 = textureLod(heightmapMip, worldXZ/TERRAIN_SIZE, float(pc.lod+1)).r;
    float dist   = length(vec2(worldXZ.x,worldXZ.y) - pc.cameraPos.xz);
    float blend  = clamp((dist - pc.tileScale*MORPH_START) / (pc.tileScale*MORPH_RANGE), 0.0, 1.0);
    float height = mix(h0, h1, blend) * MAX_HEIGHT;
    gl_Position  = pc.mvp * vec4(worldXZ.x, height, worldXZ.y, 1.0);
}
```

[Source: Strugar "Continuous Distance-Dependent Level of Detail for Rendering Heightmaps", JCGT 2010; Ulrich "Rendering Massive Terrains Using Chunked LOD", SIGGRAPH 2002]

### 74.4 GPU Terrain Normal and Slope Computation

Compute normals and slope maps from height map data for lighting, road gradient analysis, and erosion simulation (§57):

```glsl
// terrain_normals.comp — compute normals from heightmap via finite differences
layout(set=0,binding=0) uniform sampler2D heightmap;
layout(set=0,binding=1, rgba16_snorm) writeonly image2D normalMap;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    float hL  = texelFetchOffset(heightmap, pix, 0, ivec2(-1,0)).r;
    float hR  = texelFetchOffset(heightmap, pix, 0, ivec2( 1,0)).r;
    float hD  = texelFetchOffset(heightmap, pix, 0, ivec2(0,-1)).r;
    float hU  = texelFetchOffset(heightmap, pix, 0, ivec2(0, 1)).r;
    vec3  n   = normalize(vec3(hL-hR, 2.0*CELL_SIZE/MAX_HEIGHT, hD-hU));
    imageStore(normalMap, pix, vec4(n*0.5+0.5, 0.0));
}
```

[Source: Strugar 2010; Turkowski "Filters for Common Resampling Tasks", Graphics Gems 1990]

---


---

## IX. Ray Tracing and Optical Geometry

Ray tracing unifies shadowing, reflection, ambient occlusion, global illumination, and caustics under a single BVH traversal abstraction. This category covers the Vulkan ray tracing pipeline (ray generation, closest-hit, any-hit, miss shaders), full path-tracer construction, shadow geometry algorithms viewed from a geometry perspective (shadow volumes, shadow map ray casting), IBL preprocessing (importance sampling, SH projection from environment maps), radiosity form-factor computation via hemicube or GPU ray casting, atmospheric scattering geometry (Rayleigh/Mie phase functions in world space), screen-space reflection ray generation and reprojection, subsurface scattering geometry (dipole and BSSRDF volumetric model), and order-independent transparency as an optical compositing problem. GPU polygon clipping is included here as a core primitive for software rasterizers and ray-geometry intersection pipelines.

### 75. Ray Tracing Geometry Pipeline

*Audience: graphics application developers, systems developers.*

Vulkan ray tracing (VK_KHR_ray_tracing_pipeline, core in Vulkan 1.2+ with KHR extensions) organises geometry into a two-level hierarchy. §24 covered LBVH *construction* as a compute algorithm; this section covers the Vulkan RT *pipeline* as a geometry-consumption interface — acceleration structure management, procedural geometry via intersection shaders, and animated geometry update strategies.

### 75.1 Acceleration Structure Fundamentals: BLAS and TLAS

- **BLAS (Bottom-Level Acceleration Structure):** Contains actual geometry — triangle meshes or AABBs for procedural geometry. Built once per unique mesh shape.
- **TLAS (Top-Level Acceleration Structure):** Contains instances of BLASes with per-instance 3×4 transforms, visibility masks, and shader binding table offsets. Rebuilt each frame for animated scenes.

Queries traverse TLAS → BLAS → geometry test. RT cores (NVIDIA) and BVH hardware (AMD RDNA3+) handle traversal; application provides shader callbacks at each intersection event.

### 75.2 Building BLAS from Vertex/Index Buffers

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

### 75.3 TLAS and Instance Transforms

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

For dynamic scenes, rebuild the TLAS each frame (instances are just a flat buffer of 64-byte structs — fast) while BLASes stay fixed unless mesh topology changes. For skinned meshes use BLAS refit (§75.5) then TLAS rebuild.

### 75.4 Procedural Geometry via Intersection Shaders

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

### 75.5 BLAS Refit for Animated Geometry

When vertex positions change (skinning, cloth, fluid surface) but triangle topology stays constant, **refit** updates the BLAS tree in O(N) rather than rebuilding in O(N log N):

```cpp
// First frame: build with ALLOW_UPDATE flag
buildInfo.flags = VK_BUILD_ACCELERATION_STRUCTURE_PREFER_FAST_BUILD_BIT_KHR |
                  VK_BUILD_ACCELERATION_STRUCTURE_ALLOW_UPDATE_BIT_KHR;
buildInfo.mode  = VK_BUILD_ACCELERATION_STRUCTURE_MODE_BUILD_KHR;
// ...build...

// Each subsequent frame after skinning pre-pass (§39.4):
buildInfo.mode    = VK_BUILD_ACCELERATION_STRUCTURE_MODE_UPDATE_KHR;
buildInfo.srcAccelerationStructure = blas;   // refit in place
buildInfo.dstAccelerationStructure = blas;
vkCmdBuildAccelerationStructuresKHR(cmd, 1, &buildInfo, &pRange);
```

Refit traverses the existing BLAS bottom-up recomputing AABBs from updated vertex positions without re-sorting primitives. Quality degrades over many frames as the tree drifts from optimal. A common strategy: refit for K frames, then full rebuild every K+1th frame. [Source: Wyman et al. "Introduction to DirectX Raytracing" §3; NVIDIA Vulkan RT best practices]

---

### 76. Shadow Geometry Algorithms

*Audience: graphics application developers.*

Shadows are the most geometry-intensive secondary effect in real-time rendering. Each technique represents a different geometry computation on the GPU: CSM partitions the view frustum, shadow volumes extrude silhouette edges, VSM filters depth maps, and RT shadows cast rays against the scene TLAS.

### 76.1 Cascaded Shadow Maps: Frustum Partitioning

CSM splits the view frustum into N subfrusta, each rendered to its own shadow map at a resolution appropriate for that depth range. GPU frustum split computation:

```glsl
// csm_splits.comp — compute N cascade split depths
layout(set=0,binding=0) writeonly buffer Splits { float splitDepths[MAX_CASCADES+1]; };
layout(push_constant) uniform PC {
    float zNear; float zFar; int N; float lambda;  // lambda blends log/uniform
} pc;
layout(local_size_x=1) in;
void main() {
    splitDepths[0] = pc.zNear;
    splitDepths[pc.N] = pc.zFar;
    for (int i=1; i<pc.N; i++) {
        float fi  = float(i) / float(pc.N);
        float log = pc.zNear * pow(pc.zFar/pc.zNear, fi);
        float uni = pc.zNear + (pc.zFar - pc.zNear) * fi;
        splitDepths[i] = mix(uni, log, pc.lambda);
    }
}
```

Each cascade renders the scene with a tight ortho projection enclosing the cascade subfustum intersected with the scene AABB. [Source: Engel "Cascaded Shadow Maps", ShaderX 5, 2006]

### 76.2 Shadow Volume Stencil: Silhouette Detection

GPU shadow volumes (Everitt & Kilgard "Robust Shadow Volumes" 2002) require detecting silhouette edges — edges where one adjacent face is front-lit and the other is back-lit. Compute silhouette in a pre-pass:

```glsl
// silhouette.comp — detect silhouette edges
layout(set=0,binding=0) readonly buffer Edges { EdgeAdj edges[]; };  // each edge: v0,v1,f0,f1
layout(set=0,binding=1) readonly buffer FaceNormals { vec3 fNorm[]; };
layout(set=0,binding=2) writeonly buffer Silhouette { SilEdge sil[]; };
layout(set=0,binding=3) buffer SilCount { uint count; };
layout(push_constant) uniform PC { vec3 lightPos; } pc;
layout(local_size_x=64) in;
void main() {
    uint e = gl_GlobalInvocationID.x;
    bool f0lit = dot(fNorm[edges[e].f0], pc.lightPos - edgeMidpoint(edges[e])) > 0.0;
    bool f1lit = dot(fNorm[edges[e].f1], pc.lightPos - edgeMidpoint(edges[e])) > 0.0;
    if (f0lit != f1lit) {
        uint slot = atomicAdd(count, 1u);
        sil[slot] = SilEdge(edges[e].v0, edges[e].v1, f0lit ? edges[e].f0 : edges[e].f1);
    }
}
```

Then extrude silhouette edges to infinity in a mesh shader or vertex shader, render front/back faces into stencil with zfail. [Source: Everitt & Kilgard "Practical and Robust Stenciled Shadow Volumes", 2002]

### 76.3 VSM / EVSM: Moment Shadow Maps

Variance Shadow Maps (Donnelly & Lauritzen 2006) store E[d] and E[d²] in a two-channel shadow map, enabling hardware PCF filtering and Gaussian blur for soft shadows:

```glsl
// shadow_depth.frag — write moments to VSM
layout(location=0) out vec2 moments;
void main() {
    float d  = gl_FragCoord.z;
    moments  = vec2(d, d*d);
    // Bias: moments.y += 0.25*(dFdx(d)*dFdx(d) + dFdy(d)*dFdy(d))
}
// shadow_sample.glsl — Chebyshev upper bound
float chebyshev(vec2 moments, float d) {
    if (d <= moments.x) return 1.0;
    float var = moments.y - moments.x*moments.x;
    var       = max(var, 1e-5);
    float delta = d - moments.x;
    return var / (var + delta*delta);
}
```

EVSM (Annen et al. 2008) extends to 4 moments (positive/negative exponential warp) to eliminate light bleeding. [Source: Donnelly & Lauritzen "Variance Shadow Maps", I3D 2006]

### 76.4 Ray-Traced Shadows via VK_KHR_ray_query

Inline ray queries against the scene TLAS (§75.3) produce ground-truth shadow masks in a deferred lighting pass:

```glsl
// rt_shadow.comp
#extension GL_EXT_ray_query : require
layout(set=0,binding=0) uniform accelerationStructureEXT tlas;
layout(set=0,binding=1) readonly buffer GBufPos { vec4 gPos[]; };
layout(set=0,binding=2, r8) uniform writeonly image2D shadowMask;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    vec3  p    = gPos[pix.y*WIDTH+pix.x].xyz;
    vec3  ldir = normalize(lightPos - p);
    float llen = length(lightPos - p);
    rayQueryEXT rq;
    rayQueryInitializeEXT(rq, tlas,
        gl_RayFlagsTerminateOnFirstHitEXT | gl_RayFlagsSkipClosestHitShaderEXT,
        0xFF, p + ldir*0.002, 0.001, ldir, llen - 0.01);
    rayQueryProceedEXT(rq);
    float shadow = (rayQueryGetIntersectionTypeEXT(rq,true)
                    == gl_RayQueryCommittedIntersectionNoneEXT) ? 1.0 : 0.0;
    imageStore(shadowMask, pix, vec4(shadow));
}
```

[Source: Vulkan Spec VK_KHR_ray_query; Wyman "A Gentle Introduction to DirectX Raytracing", SIGGRAPH 2018]

---

### 77. IBL and Environment Map Preprocessing

*Audience: graphics application developers.*

Image-Based Lighting (IBL) precomputes the integral of incoming radiance over a hemisphere, storing the result in two GPU textures: a prefiltered radiance map (PMREM) and a BRDF integration LUT. A third preprocessing step projects the environment into L2 spherical harmonics for diffuse irradiance.

### 77.1 Spherical Harmonics Projection

Project an HDR cubemap into 9 L2 SH coefficients (one per colour channel × 9 basis functions). Each pixel contributes to all 9 bands:

```glsl
// sh_project.comp — SH projection from equirectangular HDR
layout(set=0,binding=0) uniform sampler2D hdrEnv;
layout(set=0,binding=1) buffer SHCoeffs { vec3 sh[9]; };  // initialise to 0 before dispatch
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 id  = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv  = (vec2(id)+0.5) / vec2(ENV_W, ENV_H);
    float phi = uv.x * 2.0*3.14159265;
    float theta = uv.y * 3.14159265;
    vec3  dir = vec3(sin(theta)*cos(phi), cos(theta), sin(theta)*sin(phi));
    float sinT = sin(theta);
    vec3  col = texture(hdrEnv, uv).rgb * sinT;   // solid angle weight
    // L2 SH basis functions
    float Y[9];
    Y[0] = 0.282095;
    Y[1] = 0.488603*dir.y; Y[2] = 0.488603*dir.z; Y[3] = 0.488603*dir.x;
    Y[4] = 1.092548*dir.x*dir.y; Y[5] = 1.092548*dir.y*dir.z;
    Y[6] = 0.315392*(3.0*dir.z*dir.z-1.0);
    Y[7] = 1.092548*dir.x*dir.z; Y[8] = 0.546274*(dir.x*dir.x-dir.y*dir.y);
    for (int k=0; k<9; k++) atomicAdd_vec3(sh[k], col*Y[k]);
}
// Normalise by (4π / N_pixels) after dispatch
```

[Source: Ramamoorthi & Hanrahan "An Efficient Representation for Irradiance Environment Maps", SIGGRAPH 2001]

### 77.2 PMREM: Prefiltered Radiance Mip Chain

Each mip level stores the GGX-filtered radiance for a different roughness value. Importance-sample the GGX NDF to generate sample directions, then accumulate:

```glsl
// pmrem.comp — one mip level, roughness = u_roughness
layout(set=0,binding=0) uniform samplerCube envCube;
layout(set=0,binding=1, rgba16f) uniform writeonly imageCube pmrem;
layout(push_constant) uniform PC { float roughness; uint mipSize; uint numSamples; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec3 id  = ivec3(gl_GlobalInvocationID);
    vec3  N   = cubeFaceDir(id.z, (vec2(id.xy)+0.5)/float(pc.mipSize));
    vec3  col = vec3(0); float totalW = 0.0;
    for (uint i=0; i<pc.numSamples; i++) {
        vec2  xi  = hammersley(i, pc.numSamples);
        vec3  H   = importanceSampleGGX(xi, N, pc.roughness);
        vec3  L   = normalize(2.0*dot(N,H)*H - N);
        float NdL = max(dot(N,L), 0.0);
        if (NdL > 0.0) {
            col    += texture(envCube, L).rgb * NdL;
            totalW += NdL;
        }
    }
    imageStore(pmrem, id, vec4(col / max(totalW, 1e-5), 1.0));
}
```

[Source: Karis "Real Shading in Unreal Engine 4", SIGGRAPH 2013]

### 77.3 BRDF Integration LUT

Precompute the split-sum BRDF integral for all (NdotV, roughness) pairs into a 2D R16G16 LUT:

```glsl
// brdf_lut.comp — split-sum BRDF integration
layout(set=0,binding=0, rg16f) uniform writeonly image2D brdfLUT;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    vec2  uv        = (vec2(gl_GlobalInvocationID.xy)+0.5) / vec2(LUT_SIZE);
    float NdotV     = uv.x;
    float roughness = uv.y;
    vec3  V = vec3(sqrt(1.0-NdotV*NdotV), 0.0, NdotV);
    float A = 0.0, B = 0.0;
    for (uint i=0; i<NUM_SAMPLES; i++) {
        vec2  xi  = hammersley(i, NUM_SAMPLES);
        vec3  H   = importanceSampleGGX(xi, vec3(0,0,1), roughness);
        vec3  L   = normalize(2.0*dot(V,H)*H - V);
        float NdL = max(L.z, 0.0);
        float NdH = max(H.z, 0.0);
        float VdH = max(dot(V,H), 0.0);
        if (NdL > 0.0) {
            float G   = G_SmithGGX(NdotV, NdL, roughness);
            float Gv  = G * VdH / max(NdH * NdotV, 1e-5);
            float Fc  = pow(1.0-VdH, 5.0);
            A += (1.0-Fc)*Gv; B += Fc*Gv;
        }
    }
    imageStore(brdfLUT, ivec2(gl_GlobalInvocationID.xy), vec4(A,B,0,0)/float(NUM_SAMPLES));
}
```

At runtime: `Lr = pmrem.sample(R, roughness) * (F0*A + B)` where A,B are from the LUT. [Source: Karis 2013; Lagarde & de Rousiers "Moving Frostbite to PBR", SIGGRAPH 2014]

---

### 78. Global Illumination Geometry

*Audience: graphics application developers.*

Global illumination algorithms compute indirect lighting — light that bounces one or more times before reaching the eye. Three GPU geometry methods dominate real-time GI: Reflective Shadow Maps (RSM) sample the first indirect bounce from shadow map geometry; Light Propagation Volumes (LPV) propagate SH-encoded radiance through a 3D grid; Voxel Cone Tracing (VXGI) traces cones through a sparse voxel scene representation.

### 78.1 Reflective Shadow Maps

RSM (Dachsbacher & Stamminger 2005) treats each shadow map texel as a virtual point light (VPL) for the first indirect bounce. Each G-buffer pixel gathers from a random subset of RSM VPLs:

```glsl
// rsm_gather.comp — indirect irradiance from RSM VPLs
layout(set=0,binding=0) uniform sampler2D rsmPos;    // world-space position in RSM
layout(set=0,binding=1) uniform sampler2D rsmNorm;   // surface normal in RSM
layout(set=0,binding=2) uniform sampler2D rsmFlux;   // reflected flux in RSM
layout(set=0,binding=3) readonly buffer GBufPos  { vec4 gPos[]; };
layout(set=0,binding=4) readonly buffer GBufNorm { vec4 gNorm[]; };
layout(set=0,binding=5, rgba16f) uniform writeonly image2D indirectOut;
layout(set=0,binding=6) readonly buffer RSMSamples { vec2 samples[N_RSM_SAMPLES]; };
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    vec3  x   = gPos[pix.y*WIDTH+pix.x].xyz;
    vec3  n   = gNorm[pix.y*WIDTH+pix.x].xyz;
    vec3  E   = vec3(0.0);
    for (uint s=0; s<N_RSM_SAMPLES; s++) {
        vec2  uv   = samples[s];
        vec3  xp   = texture(rsmPos,  uv).xyz;
        vec3  np   = texture(rsmNorm, uv).xyz;
        vec3  flux = texture(rsmFlux, uv).rgb;
        vec3  d    = x - xp;
        float dist2= max(dot(d,d), 1e-4);
        float form = max(0.0, dot(np, d)) * max(0.0, dot(n, -d));
        E += flux * form / (dist2 * dist2);
    }
    imageStore(indirectOut, pix, vec4(E / float(N_RSM_SAMPLES), 1.0));
}
```

[Source: Dachsbacher & Stamminger "Reflective Shadow Maps", I3D 2005]

### 78.2 Light Propagation Volumes

LPV (Kaplanyan & Dachsbacher 2010) propagates SH-encoded radiance through a 3D grid of 32×32×32 cells in 8 GPU passes (one per direction of the 6-faced propagation kernel plus diagonals):

```glsl
// lpv_propagate.comp — one LPV propagation step
layout(set=0,binding=0) readonly buffer LPVIn  { vec4 shR_in[];  vec4 shG_in[];  vec4 shB_in[];  };
layout(set=0,binding=1) writeonly buffer LPVOut { vec4 shR_out[]; vec4 shG_out[]; vec4 shB_out[]; };
layout(push_constant) uniform PC { ivec3 gridDim; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
const ivec3 FACES[6] = {ivec3(1,0,0),ivec3(-1,0,0),ivec3(0,1,0),
                         ivec3(0,-1,0),ivec3(0,0,1),ivec3(0,0,-1)};
void main() {
    ivec3 v   = ivec3(gl_GlobalInvocationID);
    uint  out = flatIdx(v);
    vec4  accR = vec4(0), accG = vec4(0), accB = vec4(0);
    for (int f=0; f<6; f++) {
        ivec3 nb = v - FACES[f];
        if (any(lessThan(nb,ivec3(0))) || any(greaterThanEqual(nb,pc.gridDim))) continue;
        uint src = flatIdx(nb);
        // Rotate SH coefficients to align FACES[f] direction, scale by solid angle
        mat4 R = shRotation(vec3(FACES[f]));
        accR += R * shR_in[src] * SH_SOLID_ANGLE;
        accG += R * shG_in[src] * SH_SOLID_ANGLE;
        accB += R * shB_in[src] * SH_SOLID_ANGLE;
    }
    shR_out[out] = accR; shG_out[out] = accG; shB_out[out] = accB;
}
```

[Source: Kaplanyan & Dachsbacher "Cascaded Light Propagation Volumes for Real-Time Indirect Illumination", I3D 2010]

### 78.3 Voxel Cone Tracing (VXGI)

VXGI (Crassin et al. 2011) stores a sparse voxel octree (§27.3) of scene radiance, then traces cones through it using anisotropic voxel mip-maps. The cone traces N levels of the SVO hierarchy to compute indirect diffuse and specular:

```glsl
// vxgi_cone.glsl — diffuse cone trace for one fragment
vec3 traceConeDiffuse(vec3 origin, vec3 normal, float aperture) {
    vec3 accum    = vec3(0.0);
    float occlusion = 0.0;
    float dist    = VOXEL_SIZE;
    while (dist < MAX_DIST && occlusion < 1.0) {
        float diameter = max(VOXEL_SIZE, 2.0 * aperture * dist);
        float mip      = log2(diameter / VOXEL_SIZE);
        vec3  pos      = origin + normal * dist;
        vec4  voxel    = textureLod(voxelSVO, worldToSVO(pos), mip);
        accum      += (1.0 - occlusion) * voxel.a * voxel.rgb;
        occlusion  += (1.0 - occlusion) * voxel.a;
        dist       += diameter * CONE_STEP;
    }
    return accum;
}
void main() {
    vec3 indirect = vec3(0.0);
    // Hemispherical diffuse: 5 cones at aperture 60°
    for (int c=0; c<5; c++)
        indirect += traceConeDiffuse(worldPos+normal*VOXEL_SIZE, coneDir[c], radians(60.0));
    indirect /= 5.0;
}
```

[Source: Crassin et al. "Interactive Indirect Illumination Using Voxel Cone Tracing", CGF 2011; NVIDIA VXGI SDK]

### 78.4 GI Denoising Geometry Buffer

All three GI methods produce noisy output. Spatiotemporal denoising (SVGF, Schied et al. 2017) uses a geometry-guided bilateral filter with 3×3 À-trous wavelet passes, rejecting samples across depth and normal discontinuities:

```glsl
// svgf_atrous.comp — one À-trous wavelet pass
layout(set=0,binding=0) uniform sampler2D colorIn;
layout(set=0,binding=1) uniform sampler2D gDepth;
layout(set=0,binding=2) uniform sampler2D gNormal;
layout(set=0,binding=3, rgba16f) uniform writeonly image2D colorOut;
layout(push_constant) uniform PC { int stepWidth; float phiColor; float phiNormal; float phiDepth; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    vec4  cen = texelFetch(colorIn, pix, 0);
    vec3  nc  = texelFetch(gNormal, pix, 0).xyz;
    float dc  = texelFetch(gDepth,  pix, 0).r;
    vec4  sum = vec4(0.0); float wSum = 0.0;
    const int r = 2;
    for (int dy=-r; dy<=r; dy++) for (int dx=-r; dx<=r; dx++) {
        ivec2 nb  = pix + ivec2(dx,dy)*pc.stepWidth;
        vec4  cn  = texelFetch(colorIn, nb, 0);
        float wC  = exp(-dot(cen.rgb-cn.rgb,cen.rgb-cn.rgb)/pc.phiColor);
        float wN  = pow(max(0.0,dot(nc,texelFetch(gNormal,nb,0).xyz)),pc.phiNormal);
        float wD  = exp(-abs(dc-texelFetch(gDepth,nb,0).r)/pc.phiDepth);
        float w   = wC*wN*wD*kernel[dy+r][dx+r];
        sum += cn*w; wSum += w;
    }
    imageStore(colorOut, pix, sum/max(wSum,1e-5));
}
```

[Source: Schied et al. "Spatiotemporal Variance-Guided Filtering", HPG 2017]

---

### 79. Geometric Optics: Lens, Caustics, and Beam Tracing

*Audience: graphics application developers.*

Geometric optics models light as rays that refract at dielectric surfaces and focus into caustics. GPU implementations use the scene TLAS for refraction ray tracing, photon mapping for caustic density estimation, and beam tracing for specular-specular transport.

### 79.1 Lens Flare Geometry

GPU lens flares place starburst and halo sprites along the screen-space line from light source to screen centre. A depth-aware occlusion test determines flare intensity:

```glsl
// lensflare.comp — compute flare element positions and intensities
layout(set=0,binding=0) readonly buffer LightSrc { vec4 lightScreenPos; };
layout(set=0,binding=1) uniform sampler2D depthBuf;
layout(set=0,binding=2) writeonly buffer Flares { FlareElement flares[MAX_ELEMENTS]; };
layout(local_size_x=1) in;
void main() {
    vec2  lsp   = lightScreenPos.xy;           // NDC light position
    float vis   = 0.0;
    const int   S = 16;
    for (int i=0; i<S; i++) {
        vec2 jit = disk16[i] * FLARE_OCCLUSION_RADIUS;
        vec2 uv  = lsp*0.5+0.5 + jit;
        float sd = texture(depthBuf, uv).r;
        float ld = lightScreenPos.w;           // light depth in NDC
        if (sd >= ld - 1e-3) vis += 1.0/float(S);
    }
    vec2  axis = vec2(0.5) - lsp*0.5+0.5;
    for (uint e=0; e<N_FLARE_ELEMENTS; e++) {
        flares[e].pos    = (lsp*0.5+0.5) + axis * flareOffset[e];
        flares[e].scale  = flareScale[e] * vis;
        flares[e].tileID = flareTile[e];
    }
}
```

[Source: Mittring "Finding Next Gen — CryEngine 2", SIGGRAPH 2007; Jimenez et al. "Practical Real-Time Lens-Flare Rendering", CGF 2012]

### 79.2 Caustic Photon Mapping

Caustics (focused light through glass/water) are computed via bidirectional photon tracing. Photons are emitted from the light source, refracted through dielectrics using Snell's law, and collected on diffuse surfaces:

```glsl
// caustic_photon.raygen.glsl
layout(location=0) rayPayloadEXT struct { vec3 power; vec3 dir; uint depth; } pload;
layout(set=0,binding=0) uniform accelerationStructureEXT tlas;
layout(set=0,binding=1) buffer Photons { Photon photons[]; };
layout(set=0,binding=2) buffer PhotonCount { uint count; };
void main() {
    uint idx = gl_LaunchIDEXT.x;
    pload.power = lightColor * PHOTON_POWER;
    pload.dir   = uniformSampleCone(lightDir, lightAngle, halton2D(idx));
    pload.depth = 0u;
    traceRayEXT(tlas, gl_RayFlagsNoneEXT, 0xFF,
                0, 1, 0, lightPos, 0.001, pload.dir, MAX_DIST, 0);
}
// caustic_chit.glsl — refract at dielectric, deposit photon on diffuse surface
// At dielectric: compute Fresnel ratio, refract via Snell's law, recurse
// At diffuse surface: store photon position + power in buffer
```

[Source: Jensen "Realistic Image Synthesis Using Photon Mapping", 2001; GPU photon mapping: Hachisuka et al. "Progressive Photon Mapping", SIGGRAPH Asia 2008]

### 79.3 Caustic Density Estimation

After tracing N photons, estimate caustic irradiance at each surface point via GPU k-NN in the photon buffer (§87.1 pattern) and kernel density estimation:

```glsl
// caustic_kde.comp — k-NN density estimation at G-buffer pixels
layout(set=0,binding=0) readonly buffer Photons { Photon photons[]; };
layout(set=0,binding=1) readonly buffer GBufPos { vec4 gPos[]; };
layout(set=0,binding=2, rgba16f) uniform writeonly image2D causticOut;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    vec3  p   = gPos[pix.y*WIDTH+pix.x].xyz;
    // Find K nearest photons (brute force for small N, §87.1 k-d tree for large)
    float r2  = KDE_RADIUS*KDE_RADIUS;
    vec3  E   = vec3(0.0);
    for (uint i=0; i<N_PHOTONS; i++) {
        float d2 = dot(photons[i].pos-p, photons[i].pos-p);
        if (d2 < r2) E += photons[i].power * (1.0 - sqrt(d2/r2));  // cone kernel
    }
    imageStore(causticOut, pix, vec4(E / (3.14159*r2), 1.0));
}
```

[Source: Jensen 2001; Zhu et al. "Real-time Caustics Using Photon Mapping on GPU", 2007]

---

### 80. Screen-Space Reflections (SSR/SSSR)

*Audience: graphics application developers.*

Screen-space reflections (SSR) trace reflection rays against the depth buffer using hierarchical Z-buffer ray marching, providing real-time specular GI for surfaces visible on screen. Stochastic SSR (SSSR) adds per-pixel random ray selection and temporal accumulation to handle rough surfaces with variable BRDF lobe widths.

### 80.1 Hierarchical Z-Buffer Ray Marching

Build a mip chain of the depth buffer where each level stores the maximum depth in each 2×2 tile. March the reflection ray through increasing mip levels, stepping coarsely over empty space:

```glsl
// ssr_hiz_march.comp — hierarchical Z ray march for reflection
layout(set=0,binding=0) uniform sampler2D hiZPyramid;  // max-depth mip chain
layout(set=0,binding=1) uniform sampler2D gNormal;
layout(set=0,binding=2) uniform sampler2D gDepth;
layout(set=0,binding=3, rgba16f) uniform writeonly image2D ssrOut;
layout(push_constant) uniform PC { mat4 proj; mat4 invProj; mat4 invView; float maxDist; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    vec3  N    = texelFetch(gNormal, pix, 0).xyz;
    float d    = texelFetch(gDepth,  pix, 0).r;
    vec3  P    = viewSpacePos(pix, d, pc.invProj);
    vec3  V    = normalize(-P);
    vec3  R    = reflect(-V, N);
    // March R through HiZ pyramid
    vec3  Q    = P; int mip = 0;
    vec4  hit  = vec4(0);
    for (int i=0; i<64; i++) {
        Q += R * (0.01 * float(1<<mip));
        vec4  clip = pc.proj * vec4(Q,1); clip /= clip.w;
        vec2  uv   = clip.xy*0.5+0.5;
        float zHiZ = textureLod(hiZPyramid, uv, float(mip)).r;
        if (Q.z < zHiZ - 0.01) { if (mip==0) { hit=vec4(uv,Q.z,1); break; } mip=max(0,mip-1); }
        else mip = min(mip+1, MAX_MIP);
        if (uv.x<0||uv.x>1||uv.y<0||uv.y>1||Q.z>0) break;
    }
    imageStore(ssrOut, pix, hit);
}
```

[Source: McGuire & Mara "Efficient GPU Screen-Space Ray Tracing", JCGT 2014; Stachowiak "Stochastic All The Things" 2018]

### 80.2 Stochastic SSR with GGX Importance Sampling

SSSR (Stachowiak 2018) samples one ray per pixel per frame from the GGX BRDF lobe, giving rough-surface reflections at the cost of one sample per pixel plus temporal accumulation:

```glsl
// sssr_sample.comp — sample one GGX reflection ray per pixel
layout(set=0,binding=0) uniform sampler2D gRoughness;
layout(set=0,binding=1) readonly buffer BlueNoise { vec2 bn[]; };  // §105 blue noise
layout(set=0,binding=2, rgba16f) uniform writeonly image2D rayDirOut;
layout(push_constant) uniform PC { uint frameIdx; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    float rough= texelFetch(gRoughness, pix, 0).r;
    uint  seed = (pix.y*WIDTH+pix.x + pc.frameIdx*WIDTH*HEIGHT) % BN_SIZE;
    vec2  xi   = bn[seed];
    // GGX importance sample: half-vector H from roughness
    float a    = rough*rough;
    float phi  = 2.0*3.14159*xi.x;
    float cosT = sqrt((1.0-xi.y)/(1.0+(a*a-1.0)*xi.y));
    float sinT = sqrt(1.0-cosT*cosT);
    vec3  H    = tangentToWorld(vec3(sinT*cos(phi), sinT*sin(phi), cosT), gNormal(pix));
    vec3  R    = reflect(-viewDir(pix), H);
    imageStore(rayDirOut, pix, vec4(R,rough));
}
```

[Source: Stachowiak 2018; Walter et al. "Microfacet Models for Refraction through Rough Surfaces", EGSR 2007]

### 80.3 Temporal Accumulation and Reprojection

Accumulate SSR samples over multiple frames by reprojecting the previous frame's result:

```glsl
// ssr_temporal.comp — temporal accumulation for SSR
layout(set=0,binding=0) uniform sampler2D ssrCurrent;   // this frame's SSR sample
layout(set=0,binding=1) uniform sampler2D ssrHistory;   // previous accumulated result
layout(set=0,binding=2) uniform sampler2D motionVec;    // §100.2 motion vectors
layout(set=0,binding=3, rgba16f) uniform writeonly image2D ssrAccum;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv   = (vec2(pix)+0.5)/vec2(imageSize(ssrAccum));
    vec2  mv   = texture(motionVec, uv).rg;
    vec4  cur  = texelFetch(ssrCurrent, pix, 0);
    vec4  hist = texture(ssrHistory, uv-mv);
    // Neighbourhood clamp to prevent ghosting
    vec4  mn   = vec4(1e30), mx = vec4(-1e30);
    for (int dy=-1;dy<=1;dy++) for(int dx=-1;dx<=1;dx++) {
        vec4 s = texelFetch(ssrCurrent,pix+ivec2(dx,dy),0);
        mn=min(mn,s); mx=max(mx,s);
    }
    hist       = clamp(hist, mn, mx);
    imageStore(ssrAccum, pix, mix(hist, cur, 0.1));
}
```

[Source: Karis "High Quality Temporal Supersampling", SIGGRAPH 2014; Stachowiak 2018]

### 80.4 SSR Fade and Composite

Fade reflections at screen edges (where the ray exits the screen) and blend with the specular IBL (§77) fallback:

```glsl
// ssr_composite.frag — blend SSR with IBL fallback
uniform sampler2D ssrAccum;    // §80.3 accumulated SSR
uniform sampler2D iblSpecular; // §77 IBL specular
uniform sampler2D gRoughness;
uniform sampler2D gMetallic;
in vec2 texUV;
void main() {
    vec4  ssr  = texture(ssrAccum, texUV);
    float conf = ssr.a;  // hit confidence (0=miss, 1=hit)
    // Fade at screen edges
    float edgeFade = 1.0;
    edgeFade *= smoothstep(0.0, 0.05, texUV.x) * smoothstep(1.0, 0.95, texUV.x);
    edgeFade *= smoothstep(0.0, 0.05, texUV.y) * smoothstep(1.0, 0.95, texUV.y);
    conf *= edgeFade;
    // Fade with roughness (SSR valid only for smooth surfaces)
    conf *= 1.0 - smoothstep(0.3, 0.6, texture(gRoughness,texUV).r);
    vec3  ibl  = texture(iblSpecular, texUV).rgb;
    fragColor  = vec4(mix(ibl, ssr.rgb, conf), 1.0);
}
```

[Source: McGuire & Mara 2014; Yasin "Screen Space Reflections in The Surge", GDC 2018]

---

### 81. GPU Radiosity: Form Factor Computation

*Audience: graphics application developers.*

Radiosity (Goral et al. 1984) computes diffuse inter-reflections by solving a linear system where the coefficient matrix contains *form factors* F_{ij} — the fraction of energy leaving patch i that arrives at patch j. GPU hemicube rendering computes form factors for all patches in a scene in parallel.

### 81.1 Hemicube Rendering for Form Factors

For each patch i, render the scene from i's centroid onto a hemicube (five 90° faces). The delta form factor ΔF_{ij} for each hemicube texel is a precomputed weight; the form factor F_{ij} is the sum over all texels covered by patch j:

```glsl
// hemicube_project.comp — project patches onto hemicube face, accumulate form factors
layout(set=0,binding=0) readonly buffer PatchCentroid { vec3 cen[];  };
layout(set=0,binding=1) readonly buffer PatchNormal   { vec3 nor[];  };
layout(set=0,binding=2) readonly buffer PatchArea     { float area[]; };
layout(set=0,binding=3) buffer FormFactor { float F[MAX_PATCHES][MAX_PATCHES]; };
layout(push_constant) uniform PC { uint srcPatch; uint hemiFace; uvec2 hemiRes; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    // Ray from patch centroid through hemicube texel
    vec3  rayDir = hemicubeRayDir(pix, pc.hemiFace, pc.hemiRes);
    float dff    = deltaFormFactor(pix, pc.hemiFace, pc.hemiRes);  // precomputed weight
    // Find which patch this ray hits (via BVH traverse §31)
    uint  hitPatch = bvhRayQuery(cen[pc.srcPatch] + nor[pc.srcPatch]*1e-3, rayDir);
    if (hitPatch == INVALID) return;
    atomicAdd_float(F[pc.srcPatch][hitPatch], dff);
}
```

[Source: Cohen & Greenberg "The Hemi-Cube: A Radiosity Solution for Complex Environments", SIGGRAPH 1985; GPU radiosity: Stamminger et al. "Hierarchical Solution for Radiosity on GPUs", EG 2002]

### 81.2 Progressive Radiosity Solve

The radiosity system B = E + ρ·F·B (B=radiosity, E=emission, ρ=reflectance, F=form factor matrix) is solved by iterative shooting: in each step, the patch with the most unshot energy broadcasts to all others:

```glsl
// radiosity_shoot.comp — shoot energy from max-energy patch to all others
layout(set=0,binding=0) buffer Radiosity { vec3 B[];       };  // per-patch radiosity
layout(set=0,binding=1) buffer Unshot    { vec3 unshot[];  };  // energy waiting to be shot
layout(set=0,binding=2) readonly buffer FormFactor { float F[MAX_PATCHES][MAX_PATCHES]; };
layout(set=0,binding=3) readonly buffer Reflectance { vec3 rho[]; };
layout(push_constant) uniform PC { uint srcPatch; float srcArea; } pc;
layout(local_size_x=64) in;
void main() {
    uint j  = gl_GlobalInvocationID.x;
    vec3 dB = rho[j] * F[pc.srcPatch][j] * unshot[pc.srcPatch] * pc.srcArea / area[j];
    B[j]       += dB;
    unshot[j]  += dB;
}
// CPU: after each pass, zero unshot[srcPatch], find next max-energy patch, repeat
```

[Source: Cohen et al. "A Progressive Refinement Approach for Fast Radiosity Image Generation", SIGGRAPH 1988; Neumann "Monte Carlo Radiosity", Computing 1995]

### 81.3 Hierarchical Radiosity

For large scenes, hierarchical radiosity (Hanrahan et al. 1991) subdivides patches and only computes form factors at the appropriate refinement level. GPU kernel: for each patch pair, test whether their form factor contribution exceeds a threshold; if so, subdivide:

```glsl
// hrad_refine.comp — decide whether patch pair (i,j) needs subdivision
layout(set=0,binding=0) readonly buffer PatchArea { float area[]; };
layout(set=0,binding=1) readonly buffer FormFactor { float F[][]; };
layout(set=0,binding=2) buffer RefineFlags { uint subdivide[]; };
layout(push_constant) uniform PC { float eps; } pc;
layout(local_size_x=64) in;
void main() {
    uint pair = gl_GlobalInvocationID.x;
    uint i = patchPairs[pair].x, j = patchPairs[pair].y;
    // BF oracle: if F[i][j] * B[i] * area[i] > eps → need finer form factor
    float BF = length(unshot[i]) * F[i][j] * area[i];
    if (BF > pc.eps) {
        atomicOr(subdivide[i], 1u);
        atomicOr(subdivide[j], 1u);
    }
}
```

[Source: Hanrahan et al. "A Rapid Hierarchical Radiosity Algorithm", SIGGRAPH 1991]

### 81.4 Irradiance Caching for Radiosity Rendering

Store per-vertex irradiance from the radiosity solve; interpolate at shade points using the irradiance cache (Ward & Heckbert 1992):

```glsl
// irradiance_cache.frag — lookup interpolated irradiance from radiosity patches
uniform sampler2D gWorldPos;
uniform sampler2D gNormal;
layout(set=0,binding=0) readonly buffer IrradCache { IrradSample samples[]; };
layout(set=0,binding=1) readonly buffer KDTree { uint kd[]; };
in vec2 texUV;
void main() {
    vec3  P  = texture(gWorldPos, texUV).xyz;
    vec3  N  = texture(gNormal,   texUV).xyz;
    vec3  E  = vec3(0.0); float wSum = 0.0;
    // k-NN lookup in irradiance cache (§37.1 pattern → BVH §31)
    for (int k=0; k<CACHE_K; k++) {
        IrradSample s = findKthNearest(kd, samples, P, k);
        float w = 1.0 / (length(P-s.pos) + s.Ri * (1.0-dot(N,s.nor)));
        E    += w * s.irrad;
        wSum += w;
    }
    fragColor = vec4(E/wSum * albedo, 1.0);
}
```

[Source: Ward & Heckbert "Irradiance Gradients", EGWR 1992; Krivánek et al. "Radiance Caching for Efficient Global Illumination Computation", IEEE TVCG 2005]

---

### 82. GPU Path Tracing Pipeline

*Audience: graphics application developers, systems developers.*

Unbiased Monte Carlo path tracing produces reference-quality global illumination by simulating light transport directly. GPU path tracing uses the TLAS/BLAS hierarchy of `VK_KHR_ray_tracing_pipeline` or ray queries, with multiple importance sampling (MIS) weighting direct-light and BRDF samples. The challenge is per-ray divergence: shader sorting (wavefront path tracing) groups coherent rays together before dispatch.

### 82.1 Primary Ray Generation and TLAS Traversal

```glsl
// pt_raygen.rgen — generate primary rays from a thin-lens camera model
#version 460
#extension GL_EXT_ray_tracing : require
layout(set=0,binding=0) uniform accelerationStructureEXT tlas;
layout(set=0,binding=1, rgba32f) uniform image2D accumBuffer;
layout(set=0,binding=2) readonly buffer BlueNoise { vec2 bn[]; };
layout(push_constant) uniform PC { mat4 invViewProj; vec3 camPos; float lensR; uint frame; } pc;
layout(location=0) rayPayloadEXT PathPayload payload;
void main() {
    ivec2 pix  = ivec2(gl_LaunchIDEXT.xy);
    uint  seed = (pix.y*gl_LaunchSizeEXT.x + pix.x + pc.frame*gl_LaunchSizeEXT.x*gl_LaunchSizeEXT.y) % BN_SIZE;
    vec2  jitter = bn[seed];
    vec2  uv   = (vec2(pix)+jitter) / vec2(gl_LaunchSizeEXT.xy) * 2.0 - 1.0;
    vec4  target = pc.invViewProj * vec4(uv, 1.0, 1.0); target /= target.w;
    // Thin-lens DoF
    vec2  lens  = concentricDiskSample(bn[(seed+1)%BN_SIZE]) * pc.lensR;
    vec3  fPoint = target.xyz;
    vec3  origin = pc.camPos + vec3(lens,0);
    vec3  dir    = normalize(fPoint - origin);
    payload.radiance = vec3(0); payload.throughput = vec3(1); payload.depth = 0;
    traceRayEXT(tlas, gl_RayFlagsNoneEXT, 0xFF, 0, 0, 0, origin, 1e-3, dir, 1e4, 0);
    vec4 prev  = imageLoad(accumBuffer, pix);
    imageStore(accumBuffer, pix, prev + vec4(payload.radiance, 1.0));
}
```

[Source: Pharr et al. "Physically Based Rendering", 4th ed. 2023, §16; Shirley et al. "Ray Tracing in One Weekend" series]

### 82.2 Closest-Hit Shader: MIS Direct Lighting

Multiple importance sampling (MIS) combines a direct-light sample and a BRDF sample using power heuristic weights to reduce variance:

```glsl
// pt_closesthit.rchit — shade surface point with MIS direct lighting + BRDF sample
#extension GL_EXT_ray_tracing : require
#extension GL_EXT_ray_query   : require
layout(location=0) rayPayloadInEXT PathPayload payload;
layout(location=1) rayPayloadEXT   ShadowPayload shadow;
hitAttributeEXT vec2 bary;
void main() {
    SurfacePoint sp = reconstructSurface(gl_PrimitiveID, bary);
    if (sp.emission != vec3(0)) { payload.radiance += payload.throughput * sp.emission; return; }
    // --- Direct light sample (NEE) ---
    LightSample ls = sampleLight(sp.pos, sp.nor, payload.seed);
    float pdfBRDF  = evalBRDFpdf(sp, ls.dir);
    float wLight   = misPowerHeuristic(ls.pdf, pdfBRDF);
    bool visible   = shadowRay(sp.pos, ls.dir, ls.dist);
    if (visible)
        payload.radiance += payload.throughput * evalBRDF(sp, ls.dir) * ls.Le * wLight / ls.pdf;
    // --- BRDF sample (continue path) ---
    vec3 wi; float pdfW;
    vec3 f   = sampleBRDF(sp, payload.seed, wi, pdfW);
    float pdfLight = lightPdf(sp.pos, wi);
    float wBRDF    = misPowerHeuristic(pdfW, pdfLight);
    payload.throughput *= f * abs(dot(wi,sp.nor)) * wBRDF / max(pdfW, 1e-8);
    // Russian roulette
    float q = max(payload.throughput.r, max(payload.throughput.g, payload.throughput.b));
    if (rand(payload.seed) > q) { payload.throughput=vec3(0); return; }
    payload.throughput /= q;
    // Spawn next ray
    payload.origin = sp.pos + sp.nor*1e-3; payload.dir = wi; payload.depth++;
}
```

[Source: Pharr et al. 2023 §12.2 MIS; Veach "Robust Monte Carlo Methods for Light Transport Simulation", PhD 1997]

### 82.3 Wavefront Path Tracing: Shader Sorting

GPU warp divergence kills throughput when adjacent paths hit different materials. Wavefront path tracing (Laine et al. 2013) uses separate queues per shader type. After intersection, paths are sorted by material ID and dispatched in homogeneous waves:

```glsl
// pt_sort_queues.comp — classify intersection records by material type
layout(set=0,binding=0) readonly buffer HitRecords { HitRecord hits[]; };
layout(set=0,binding=1) buffer Queue_Lambert  { uint q[]; uint cnt; };
layout(set=0,binding=2) buffer Queue_GGX      { uint q[]; uint cnt; };
layout(set=0,binding=3) buffer Queue_Glass    { uint q[]; uint cnt; };
layout(set=0,binding=4) buffer Queue_Miss     { uint q[]; uint cnt; };
layout(local_size_x=64) in;
void main() {
    uint p = gl_GlobalInvocationID.x;
    if (hits[p].miss) {
        uint s = atomicAdd(Queue_Miss.cnt, 1u); Queue_Miss.q[s]=p; return;
    }
    switch (hits[p].materialType) {
        case MAT_LAMBERT: { uint s=atomicAdd(Queue_Lambert.cnt,1u); Queue_Lambert.q[s]=p; break; }
        case MAT_GGX:     { uint s=atomicAdd(Queue_GGX.cnt,1u);     Queue_GGX.q[s]=p;     break; }
        case MAT_GLASS:   { uint s=atomicAdd(Queue_Glass.cnt,1u);    Queue_Glass.q[s]=p;   break; }
    }
}
// Each queue is then dispatched as a separate compute shader (material kernel)
```

[Source: Laine et al. "Megakernels Considered Harmful: Wavefront Path Tracing on GPUs", HPG 2013]

### 82.4 Denoising: SVGF À-trous Wavelet Filter

After accumulating N samples, SVGF (§78.4 pattern applied to path tracing) denoises via spatially-varying À-trous wavelets guided by geometry buffers:

```glsl
// svgf_atrous.comp — SVGF À-trous wavelet denoise pass k
layout(set=0,binding=0) uniform sampler2D colorIn;    // noisy accumulated radiance
layout(set=0,binding=1) uniform sampler2D gNormal;
layout(set=0,binding=2) uniform sampler2D gDepth;
layout(set=0,binding=3) uniform sampler2D gAlbedo;
layout(set=0,binding=4, rgba32f) writeonly image2D colorOut;
layout(push_constant) uniform PC { int stepWidth; float phiColor; float phiNorm; float phiDepth; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    vec3  cC   = texelFetch(colorIn,pix,0).rgb / texelFetch(gAlbedo,pix,0).rgb;  // modulate albedo
    vec3  nC   = texelFetch(gNormal,pix,0).xyz;
    float dC   = texelFetch(gDepth, pix,0).r;
    vec3  sum  = vec3(0); float wSum=0.0;
    const float kernel[3] = {3.0/8.0, 1.0/4.0, 1.0/16.0};
    for (int dy=-2;dy<=2;dy++) for(int dx=-2;dx<=2;dx++) {
        ivec2 nb   = pix + ivec2(dx,dy)*pc.stepWidth;
        vec3  cN   = texelFetch(colorIn,nb,0).rgb / texelFetch(gAlbedo,nb,0).rgb;
        float wC   = exp(-dot(cN-cC,cN-cC)/pc.phiColor);
        float wN   = pow(max(dot(nC,texelFetch(gNormal,nb,0).xyz),0.0),pc.phiNorm);
        float wD   = exp(-abs(dC-texelFetch(gDepth,nb,0).r)/pc.phiDepth);
        float h    = kernel[abs(dx)]*kernel[abs(dy)];
        float w    = h*wC*wN*wD; sum+=w*cN; wSum+=w;
    }
    imageStore(colorOut,pix,vec4(sum/wSum * texelFetch(gAlbedo,pix,0).rgb,1.0));
}
```

[Source: Schied et al. "Spatiotemporal Variance-Guided Filtering", HPG 2017; §51.4]

---

### 83. Atmospheric Scattering and Sky Rendering

*Audience: graphics application developers.*

Atmospheric scattering (Mie + Rayleigh) produces realistic sky colours, horizon glow, and aerial perspective. GPU implementations precompute transmittance and in-scatter into 2D/4D lookup tables (LUTs), then evaluate them at runtime with two texture samples — following Bruneton & Neyret (2008) and the Unreal/Unity production extensions.

### 83.1 Transmittance LUT Precomputation

For each (view zenith, altitude) pair, integrate extinction along the ray to space using the Chapman function approximation:

```glsl
// atmo_transmittance.comp — precompute transmittance LUT T(h, cosθ)
layout(set=0,binding=0, rg32f) writeonly image2D transmittanceLUT;  // (Rₑ, Rₘ) transmittance
layout(push_constant) uniform PC { float Rg; float Rt; float HR; float HM;
    float betaR; float betaM; int numSteps; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    vec2  uv  = (vec2(gl_GlobalInvocationID.xy)+0.5)/vec2(imageSize(transmittanceLUT));
    float h   = pc.Rg + uv.x*(pc.Rt-pc.Rg);          // altitude
    float cosT= uv.y * 2.0 - 1.0;                     // cos of view-zenith
    float dt  = atmosphericRayLength(h, cosT, pc.Rt) / float(pc.numSteps);
    vec2  tau = vec2(0.0);  // optical depth (Rayleigh, Mie)
    for (int i=0; i<pc.numSteps; i++) {
        float ri = h + (float(i)+0.5)*dt*cosT;
        tau.x   += exp(-(ri-pc.Rg)/pc.HR) * dt;
        tau.y   += exp(-(ri-pc.Rg)/pc.HM) * dt;
    }
    vec2 T = exp(-vec2(pc.betaR,pc.betaM) * tau);
    imageStore(transmittanceLUT, ivec2(gl_GlobalInvocationID.xy), vec4(T,0,0));
}
```

[Source: Bruneton & Neyret "Precomputed Atmospheric Scattering", EGSR 2008; Hillaire "A Scalable and Production Ready Sky and Atmosphere Rendering Technique", EGSR 2020]

### 83.2 Single-Scatter In-Scatter LUT

For each (altitude, view zenith, sun zenith) triple, integrate the single-scatter contribution (light reaching a point, scattering toward the viewer) using the transmittance LUT:

```glsl
// atmo_inscatter.comp — single in-scatter LUT (3D texture: altitude × viewZen × sunZen)
layout(set=0,binding=0) uniform sampler2D transmittanceLUT;
layout(set=0,binding=1, rgba32f) writeonly image3D inScatterLUT;
layout(local_size_x=4,local_size_y=4,local_size_z=4) in;
void main() {
    vec3 uvw  = (vec3(gl_GlobalInvocationID)+0.5)/vec3(imageSize(inScatterLUT));
    float h   = Rg + uvw.x*(Rt-Rg);
    float cosV= uvw.y*2.0-1.0;
    float cosS= uvw.z*2.0-1.0;
    float dt  = atmosphericRayLength(h,cosV,Rt)/float(STEPS);
    vec3  inR=vec3(0), inM=vec3(0);
    for (int i=0; i<STEPS; i++) {
        float t  = (float(i)+0.5)*dt;
        float ri = sqrt(h*h + t*t + 2.0*h*t*cosV);
        vec2  Ti = textureLod(transmittanceLUT, txUV(ri,cosS), 0.0).rg;  // sun→point
        vec2  Tv = textureLod(transmittanceLUT, txUV(h, cosV), 0.0).rg;  // point→cam
        float rhoR = exp(-(ri-Rg)/HR), rhoM = exp(-(ri-Rg)/HM);
        inR += rhoR * Ti * Tv * dt;
        inM += rhoM * Ti * Tv * dt;
    }
    imageStore(inScatterLUT, ivec3(gl_GlobalInvocationID),
               vec4(betaR*inR + betaM/(4.0*PI)*inM, 1.0));
}
```

[Source: Bruneton & Neyret 2008; Elek "Rendering Parametrizable Planetary Atmospheres with Multiple Scattering in Real-Time", CESCG 2009]

### 83.3 Runtime Sky Evaluation

At runtime, the sky colour in direction d from camera altitude h is the in-scatter LUT lookup, attenuated by the sun's phase function:

```glsl
// sky.frag — evaluate sky colour from precomputed LUTs
uniform sampler2D transmittanceLUT;
uniform sampler3D inScatterLUT;
uniform vec3 sunDir;
in vec3 viewDir;
void main() {
    vec3  V    = normalize(viewDir);
    float cosV = V.y;                       // view zenith angle
    float cosS = sunDir.y;                  // sun zenith
    float h    = cameraAltitude + Rg;       // metres from planet center
    vec3  inS  = texture(inScatterLUT, vec3(altToUV(h), cosToUV(cosV), cosToUV(cosS))).rgb;
    // Phase functions
    float mu   = dot(V, sunDir);
    float phR  = 3.0/(16.0*PI) * (1.0+mu*mu);
    float phM  = 3.0/(8.0*PI) * (1.0-g*g)*(1.0+mu*mu) / ((2.0+g*g)*pow(1.0+g*g-2.0*g*mu, 1.5));
    fragColor  = vec4(inS * (phR + phM) * SOLAR_IRRADIANCE, 1.0);
}
```

[Source: Bruneton & Neyret 2008; Hillaire 2020]

### 83.4 Aerial Perspective (Depth-Based Fog)

Apply aerial perspective to scene geometry by integrating the in-scatter and transmittance along the view ray to each depth sample:

```glsl
// aerial_perspective.comp — compute aerial perspective for each depth sample
layout(set=0,binding=0) uniform sampler2D depthBuffer;
layout(set=0,binding=1) uniform sampler3D inScatterLUT;
layout(set=0,binding=2) uniform sampler2D transmittanceLUT;
layout(set=0,binding=3, rgba16f) writeonly image2D aerialFog;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    float depth= texelFetch(depthBuffer, pix, 0).r;
    vec3  worldPos = worldSpacePos(pix, depth);
    float dist     = length(worldPos - cameraPos);
    // Transmittance and inscatter from camera to world position
    vec2  T    = texture(transmittanceLUT, txUV(cameraAlt, dot(normalize(worldPos-cameraPos), UP))).rg;
    vec3  inS  = texture(inScatterLUT, scatterUV(cameraAlt, viewAngle, sunAngle)).rgb;
    vec3  fog  = (1.0-T.x) * inS * SOLAR_IRRADIANCE;
    imageStore(aerialFog, pix, vec4(fog, T.x));  // apply in composite pass
}
```

[Source: Bruneton & Neyret 2008; Lagarde & de Rousiers "Moving Frostbite to Physically Based Rendering", SIGGRAPH 2014]

---

### 84. Subsurface Scattering Geometry

*Audience: graphics application developers.*

Subsurface scattering (SSS) models light penetrating translucent materials (skin, marble, wax) and exiting at a different surface point. GPU implementations use the separable screen-space SSS blur (Jiménez et al. 2009), the pre-integrated skin BRDF (d'Eon & Luebke 2007), and ray-traced diffusion profiles via `VK_KHR_ray_query` for high-quality offline SSS.

### 84.1 Screen-Space SSS Blur (Jiménez et al. 2009)

Blur the irradiance buffer in screen space along each of three directions using a Gaussian kernel shaped by the diffusion profile of the material:

```glsl
// sss_blur.comp — separable screen-space SSS blur along one axis
layout(set=0,binding=0) uniform sampler2D irradianceTex;
layout(set=0,binding=1) uniform sampler2D depthTex;
layout(set=0,binding=2, rgba16f) writeonly image2D ssssOut;
layout(push_constant) uniform PC { vec2 dir; float sssStrength; float correction; } pc;
// Skin diffusion kernel: 6 Gaussians (d'Eon & Luebke 2007), precomputed
const vec4 KERNEL[6] = {
    vec4(0.233, 0.455, 0.649, 0.0),   // variance 0.0064
    vec4(0.1,   0.336, 0.344, 0.0064),
    vec4(0.118, 0.198, 0.0,  0.0484),
    vec4(0.113, 0.007, 0.007, 0.187),
    vec4(0.358, 0.004, 0.0,  0.567),
    vec4(0.078, 0.0,   0.0,  1.99)
};
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix   = ivec2(gl_GlobalInvocationID.xy);
    float depth = texelFetch(depthTex, pix, 0).r;
    vec4  color = vec4(0.0);
    for (int i=-KERNEL_SIZE; i<=KERNEL_SIZE; i++) {
        float s   = KERNEL[abs(i)%6].w;
        ivec2 nb  = pix + ivec2(round(pc.dir * float(i) / sqrt(s)));
        float dnb = texelFetch(depthTex, nb, 0).r;
        float wD  = exp(-abs(depth-dnb)*pc.correction);
        color    += texelFetch(irradianceTex, nb, 0) * KERNEL[abs(i)%6].xyzw * wD;
    }
    imageStore(ssssOut, pix, color);
}
```

[Source: Jiménez et al. "Screen-Space Perceptual Rendering of Human Skin", ACM TAP 2009; d'Eon & Luebke "Advanced Techniques for Realistic Real-Time Skin Rendering", GPU Gems 3 2007]

### 84.2 Pre-Integrated Skin BRDF

The pre-integrated skin shading model (d'Eon & Luebke 2007) precomputes the integral of the diffusion profile over a sphere of radius 1/κ (curvature) and stores it as a 2D LUT indexed by (N·L, curvature):

```glsl
// skin_brdf.frag — evaluate pre-integrated skin BRDF from LUT
uniform sampler2D skinBRDF_LUT;   // (NdotL, curvature) → pre-integrated diffusion
uniform sampler2D curvatureTex;   // per-pixel mean curvature (1/radius)
uniform vec3 lightDir;
in vec3 worldNor;
in vec2 texUV;
void main() {
    float NdotL  = dot(worldNor, lightDir) * 0.5 + 0.5;  // remap [−1,1]→[0,1]
    float kappa  = texture(curvatureTex, texUV).r;         // curvature in 1/m
    // LUT stores RGBE irradiance for each (NdotL, curvature) pair
    vec3  irrad  = texture(skinBRDF_LUT, vec2(NdotL, kappa * 0.1)).rgb;
    vec3  albedo = texture(skinAlbedo, texUV).rgb;
    fragColor    = vec4(irrad * albedo, 1.0);
}
```

[Source: d'Eon & Luebke 2007; Penner "Pre-Integrated Skin Shading", ShaderX 9 2011]

### 84.3 Ray-Traced SSS via Diffusion Dipole Ray Queries

For high-quality SSS, shoot diffusion probe rays below the surface and accumulate the dipole contribution from each hit:

```glsl
// sss_raytrace.comp — ray-traced SSS via dipole model and ray query
layout(set=0,binding=0) readonly buffer SurfacePoints { vec3 pos[]; vec3 nor[]; };
layout(set=0,binding=1) uniform accelerationStructureEXT tlas;
layout(set=0,binding=2) writeonly buffer SSSIrrad { vec3 irrad[]; };
layout(push_constant) uniform PC { int nRays; float sigma_a; float sigma_s; float eta; } pc;
layout(local_size_x=64) in;
void main() {
    uint v    = gl_GlobalInvocationID.x;
    vec3 p    = pos[v], n = nor[v];
    float mfp = 1.0 / (pc.sigma_a + pc.sigma_s);  // mean free path
    vec3  E   = vec3(0.0);
    for (int r=0; r<pc.nRays; r++) {
        // Sample a point on the surface within ~3 MFP using disc sampling
        vec3 probe = p + sampleDisk(n, mfp*3.0, halton2D(r));
        float dist = length(probe - p);
        // Dipole BSSRDF: R_d(dist)
        float Rd   = dipoleRd(dist, pc.sigma_a, pc.sigma_s, pc.eta);
        // Test visibility from probe to light
        rayQueryEXT rq;
        rayQueryInitializeEXT(rq, tlas, gl_RayFlagsTerminateOnFirstHitEXT, 0xFF,
                              probe+n*1e-3, 0.0, lightDir, 1e4);
        rayQueryProceedEXT(rq);
        if (rayQueryGetIntersectionTypeEXT(rq,true)==gl_RayQueryCommittedIntersectionNoneEXT)
            E += Rd * lightColor * max(dot(n,lightDir),0.0);
    }
    irrad[v] = E * 2.0 * PI * mfp / float(pc.nRays);
}
```

[Source: Jensen et al. "A Practical Model for Subsurface Light Transport", SIGGRAPH 2001; d'Eon & Luebke 2007]

### 84.4 Translucency: Thin-Slab Transmission

For thin translucent objects (ears, leaves), approximate SSS by transmitting direct light through the slab using Beer's law with a transmitted thickness map:

```glsl
// translucency.frag — thin-slab SSS via transmitted thickness (Beer's law)
uniform sampler2D thicknessMap;   // pre-baked or real-time ray-traced thickness
uniform vec3 lightDir;
uniform vec3 lightColor;
uniform vec3 subsurfaceColor;     // material absorption spectrum
in vec3 worldNor;
in vec2 texUV;
void main() {
    float thickness = texture(thicknessMap, texUV).r;  // metres
    vec3  V         = normalize(viewDir);
    // Wrap lighting for back-face transmittance
    float NdotL_wrap = dot(-worldNor, lightDir) * 0.5 + 0.5;
    // Beer–Lambert attenuation: T = exp(-sigma_a * thickness)
    vec3  T         = exp(-subsurfaceColor * thickness);
    vec3  backLight = lightColor * T * NdotL_wrap;
    // Add to front-face diffuse
    vec3  diffuse   = max(dot(worldNor, lightDir), 0.0) * lightColor;
    fragColor       = vec4((diffuse + backLight) * texture(albedo, texUV).rgb, 1.0);
}
```

[Source: Green "Real-Time Approximations to Subsurface Scattering", GPU Gems 1 2004; Jiménez et al. 2010 "Separable Subsurface Scattering"]

---

### 85. Order-Independent Transparency

*Audience: graphics application developers.*

Transparent geometry must be composited in back-to-front depth order, but sorting draw calls per-frame is expensive and breaks GPU parallelism for complex scenes (particle systems, hair, foliage). Order-independent transparency (OIT) algorithms resolve alpha blending on the GPU without sorting: per-pixel linked lists (PPLL), k-buffer, and weighted blended OIT (WBOIT) cover the quality/performance trade-off.

### 85.1 Per-Pixel Linked List Construction

Append each transparent fragment to a per-pixel linked list stored in a flat buffer. A head-pointer image stores the list head per pixel; atomic exchange chains fragments together:

```glsl
// oit_ppll_build.frag — append fragment to per-pixel linked list
layout(set=0,binding=0, r32ui) uniform uimage2D headImg;   // per-pixel head pointer
layout(set=0,binding=1) buffer FragList { OITFragment frags[]; uint count; };
layout(location=0) in vec4 fragColor;
void main() {
    uint slot = atomicAdd(count, 1u);
    if (slot >= MAX_FRAGS) return;
    frags[slot].color = packUnorm4x8(fragColor);
    frags[slot].depth = gl_FragCoord.z;
    // Atomically prepend to per-pixel list
    uint prev = imageAtomicExchange(headImg, ivec2(gl_FragCoord.xy), slot);
    frags[slot].next = prev;
}
```

[Source: Yang et al. "Real-Time Concurrent Linked List Construction on the GPU", EG 2010; Crassin "OIT and GBuffer Compositing Using the New D3D11 Atomic Counting and UAVs", NVIDIA 2010]

### 85.2 Per-Pixel Linked List Resolve (Sort and Blend)

In a fullscreen pass, traverse the linked list for each pixel, sort fragments by depth, and composite back-to-front:

```glsl
// oit_ppll_resolve.comp — sort and blend per-pixel fragment list
layout(set=0,binding=0, r32ui) uniform readonly uimage2D headImg;
layout(set=0,binding=1) readonly buffer FragList { OITFragment frags[]; };
layout(set=0,binding=2, rgba16f) uniform writeonly image2D outColor;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix   = ivec2(gl_GlobalInvocationID.xy);
    uint  head  = imageLoad(headImg, pix).r;
    // Collect up to MAX_LAYERS fragments into a local array
    uint  idx[MAX_LAYERS]; int n=0;
    for (uint cur=head; cur!=0xFFFFFFFF && n<MAX_LAYERS; cur=frags[cur].next) idx[n++]=cur;
    // Insertion sort by depth (small n: 4-16 typical)
    for(int i=1;i<n;i++) { uint k=idx[i]; int j=i-1;
        while(j>=0 && frags[idx[j]].depth < frags[k].depth) { idx[j+1]=idx[j]; j--; } idx[j+1]=k; }
    // Back-to-front alpha blend
    vec4 color = vec4(0.0);
    for(int i=0;i<n;i++) {
        vec4 fc = unpackUnorm4x8(frags[idx[i]].color);
        color   = fc.a*fc + (1.0-fc.a)*color;
    }
    imageStore(outColor, pix, color);
}
```

### 85.3 Weighted Blended OIT (McGuire & Bavoil 2013)

WBOIT avoids sorting entirely by accumulating a weighted sum and weight sum in two render targets. The final composite divides out the weights. Approximate but hardware-friendly and suitable for particles:

```glsl
// wboit_accum.frag — accumulate weighted OIT into two render targets
layout(location=0) out vec4 accumRT;   // sum: w(z) * premultAlpha(color)
layout(location=1) out float revealRT; // product: (1 - alpha) per pixel
in vec4 fragColor;
void main() {
    float z = gl_FragCoord.z;
    // McGuire & Bavoil weight function: balances near/far coverage
    float w = clamp(pow(min(1.0, fragColor.a*10.0)+0.01, 3.0)*1e8
              * pow(1.0-z*0.9, 3.0), 1e-2, 3e3);
    accumRT  = vec4(fragColor.rgb*fragColor.a, fragColor.a) * w;
    revealRT = fragColor.a;  // blended with glBlendFunc(ZERO, ONE_MINUS_SRC_ALPHA)
}
```

```glsl
// wboit_composite.frag — reconstruct final colour from WBOIT accum buffers
uniform sampler2D accumTex;
uniform sampler2D revealTex;
in vec2 texUV;
void main() {
    vec4  accum  = texture(accumTex,  texUV);
    float reveal = texture(revealTex, texUV).r;
    if (reveal >= 1.0 - 1e-4) discard;  // fully opaque: skip
    fragColor = vec4(accum.rgb / max(accum.a, 1e-5), 1.0-reveal);
}
```

[Source: McGuire & Bavoil "Weighted Blended Order-Independent Transparency", JCGT 2013]

### 85.4 K-Buffer Stochastic Transparency

K-buffer (Sintorn & Assarsson 2009) keeps only the k nearest fragments per pixel. For k=4, use a 4-entry insertion sort in the fragment shader with atomic compare-and-swap on the sorted buffer:

```glsl
// kbuffer_build.frag — maintain sorted k-nearest fragments per pixel
layout(set=0,binding=0) buffer KBuffer { uint64_t kbuf[][K]; };  // packed depth+color, K per pixel
in vec4 fragColor;
void main() {
    ivec2 pix  = ivec2(gl_FragCoord.xy);
    uint  flat = pix.y*WIDTH+pix.x;
    uint64_t entry = (uint64_t(floatBitsToUint(gl_FragCoord.z))<<32)|uint64_t(packUnorm4x8(fragColor));
    // Insert entry into sorted k-buffer at this pixel (max depth evicted)
    for (int k=0; k<K; k++) {
        uint64_t old = atomicMax(kbuf[flat][k], entry);
        if (old == 0UL || entry < old) break;  // inserted in sorted position
        entry = max(entry, old);  // propagate evicted entry to next slot
    }
}
```

[Source: Sintorn & Assarsson "A Real-Time Shadow Algorithm for Unstructured Meshes", SIGGRAPH 2009; Myers & Bavoil "Stochastic Transparency", I3D 2007]

---

### 86. GPU Polygon and Frustum Clipping

*Audience: systems developers, graphics application developers.*

Clipping geometry against a frustum or arbitrary half-spaces is required for: shadow map rendering (clip to light frustum), portal rendering, decal projection (§97), constructive solid geometry (§7), and conservative rasterisation. GPU Sutherland-Hodgman clips polygons against each plane independently; hardware clipping handles triangle-level frustum culling automatically, but compute-shader clipping is needed for non-standard clip volumes.

### 86.1 Sutherland-Hodgman on GPU (Per-Polygon Thread)

Run the Sutherland-Hodgman algorithm independently for each polygon. Each thread processes one input polygon against all clip planes, outputting a clipped polygon:

```glsl
// sh_clip.comp — Sutherland-Hodgman polygon clipping against N half-spaces
layout(set=0,binding=0) readonly buffer InPolys   { Polygon inP[];  };   // variable-length vertex lists
layout(set=0,binding=1) readonly buffer ClipPlanes { vec4 planes[]; };   // ax+by+cz+d≥0 = inside
layout(set=0,binding=2) writeonly buffer OutPolys  { Polygon outP[]; };
layout(push_constant) uniform PC { uint N_POLYS; uint N_PLANES; } pc;
layout(local_size_x=64) in;
void main() {
    uint poly = gl_GlobalInvocationID.x;
    if (poly >= pc.N_POLYS) return;
    // Copy input polygon to local registers (≤16 vertices typical)
    vec3  buf[32]; int nv = inP[poly].n;
    for(int i=0;i<nv;i++) buf[i]=inP[poly].v[i];
    for (uint pl=0; pl<pc.N_PLANES; pl++) {
        vec4  P = planes[pl];
        vec3  tmp[32]; int nt=0;
        for (int i=0; i<nv; i++) {
            vec3 a=buf[i], b=buf[(i+1)%nv];
            bool aIn=(dot(P.xyz,a)+P.w)>=0.0, bIn=(dot(P.xyz,b)+P.w)>=0.0;
            if (aIn) tmp[nt++]=a;
            if (aIn!=bIn) {
                float t=(dot(P.xyz,a)+P.w)/(dot(P.xyz,a)-dot(P.xyz,b));
                tmp[nt++]=mix(a,b,t);
            }
        }
        nv=nt; for(int i=0;i<nv;i++) buf[i]=tmp[i];
        if (nv<3) break;
    }
    outP[poly].n=nv; for(int i=0;i<nv;i++) outP[poly].v[i]=buf[i];
}
```

[Source: Sutherland & Hodgman "Reentrant Polygon Clipping", CACM 1974; Blinn & Newell "Clipping Using Homogeneous Coordinates", SIGGRAPH 1978]

### 86.2 Triangle Frustum Culling via Mesh Shader Amplification

The task/amplification shader phase (§101.2 mesh shader pattern) naturally implements per-cluster frustum culling before spawning mesh shader workgroups:

```glsl
// frustum_cull.task.glsl — amplification shader: cull clusters against view frustum
layout(local_size_x=32) in;
taskPayloadSharedEXT uint visibleClusters[32];
void main() {
    uint c   = gl_GlobalInvocationID.x;
    bool vis = sphereInFrustum(clusterBounds[c].sphere, frustumPlanes);
    // Stream compaction via ballot: emit only visible clusters
    uvec4 ballot = subgroupBallot(vis);
    uint  nVis   = subgroupBallotBitCount(ballot);
    if (gl_LocalInvocationID.x==0) EmitMeshTasksEXT(nVis,1,1);
    if (vis) {
        uint slot = subgroupBallotExclusiveBitCount(ballot);
        visibleClusters[slot] = c;
    }
}
```

[Source: Wihlidal 2016 §90; Kubisch 2018; Harris "GPU-Driven Rendering Pipelines", SIGGRAPH 2015]

### 86.3 Conservative Rasterisation for Voxelisation

Conservative rasterisation expands each triangle outward so every voxel column touched by the triangle is rasterised — required for §13 solid voxelisation. Implement via geometry shader triangle expansion or the `VK_EXT_conservative_rasterization` extension:

```glsl
// conservative_rast.geom — expand triangle for conservative rasterisation
layout(triangles) in;
layout(triangle_strip, max_vertices=3) out;
uniform vec2 pixelSize;  // 1/viewport
void main() {
    vec2 p[3]; for(int i=0;i<3;i++) p[i]=gl_in[i].gl_Position.xy/gl_in[i].gl_Position.w;
    // Edge half-planes and their outward normals (scaled by half-pixel diagonal)
    float diag = length(pixelSize)*0.5*sqrt(2.0);
    for (int i=0;i<3;i++) {
        vec2 e   = p[(i+1)%3]-p[i];
        vec2 n   = normalize(vec2(-e.y,e.x));
        p[i]    += n*diag;
    }
    for (int i=0;i<3;i++) {
        gl_Position=vec4(p[i]*gl_in[i].gl_Position.w, gl_in[i].gl_Position.zw);
        EmitVertex();
    }
    EndPrimitive();
}
```

[Source: Hasselgren et al. "Conservative Rasterization", GPU Gems 2 2005; §41 voxelisation]

### 86.4 Clip-Space W-Buffer and Homogeneous Clipping

Clipping in homogeneous coordinates before the perspective divide handles the near-plane clipping case exactly, preventing division-by-zero artefacts for triangles crossing the camera:

```glsl
// homogeneous_clip.comp — clip triangle in homogeneous space against w=ε near plane
layout(set=0,binding=0) readonly buffer InTris  { vec4 hVerts[][3]; };  // homogeneous clip coords
layout(set=0,binding=1) buffer OutTris { vec4 oVerts[][3]; uint count; };
layout(local_size_x=64) in;
void main() {
    uint t  = gl_GlobalInvocationID.x;
    vec4 v[3]; for(int i=0;i<3;i++) v[i]=hVerts[t][i];
    float eps = 1e-4;
    // Sutherland-Hodgman against w≥ε (near plane in homogeneous space)
    vec4 out_[4]; int no=0;
    for(int i=0;i<3;i++) {
        vec4 a=v[i], b=v[(i+1)%3];
        bool aIn=(a.w>=eps), bIn=(b.w>=eps);
        if(aIn) out_[no++]=a;
        if(aIn!=bIn) { float t2=(a.w-eps)/(a.w-b.w); out_[no++]=mix(a,b,t2); }
    }
    // Fan-triangulate out_ into output
    for(int i=1;i<no-1;i++) {
        uint s=atomicAdd(count,1u);
        oVerts[s][0]=out_[0]; oVerts[s][1]=out_[i]; oVerts[s][2]=out_[i+1];
    }
}
```

[Source: Blinn & Newell 1978; Heckbert & Moreton "Interpolation for Polygon Texture Mapping and Shading", State of the Art in Computer Graphics 1991]

---


---

## X. Point Cloud, Reconstruction, and Perception

Point clouds are the native output of depth sensors, LiDAR scanners, and multi-view reconstruction pipelines. This category covers GPU-accelerated point cloud processing (normal estimation via PCA on local neighbourhoods, outlier removal, voxel downsampling), Structure from Motion and multi-view stereo for recovering 3D geometry from unstructured image collections, Iterative Closest Point (ICP) registration for aligning successive point cloud frames, and 3D feature descriptor computation (FPFH, SHOT, learned descriptors) for recognition and pose estimation. These algorithms form the geometry perception pipeline for robotics, autonomous vehicles, and AR/VR applications.

### 87. Point Cloud Processing

*Audience: systems developers, graphics application developers.*

Point clouds are the direct output of depth cameras (structured light, LiDAR, stereo), photogrammetry pipelines, and 3DGS/NeRF density fields. GPU point cloud processing bridges raw sensor data and the mesh/SDF representations used throughout this chapter.

### 87.1 GPU k-NN via Tiled Brute-Force

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

For larger clouds use the GPU k-d tree (§27.1) or uniform grid radius search (§27.2). [Source: Garcia et al. "Fast k Nearest Neighbor Search Using GPU", CVPR Workshop 2008]

### 87.2 Normal Estimation from k-NN

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

### 87.3 GPU ICP Registration

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

### 87.4 GPU RANSAC Primitive Fitting

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

### 87.5 Euclidean Segmentation via Label Propagation

Cluster points within distance δ using the same iterative label-propagation pattern from §70.2 and §50.2, but in 3D over a spatial hash grid:

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

### 88. GPU Structure from Motion and Multi-View Stereo

*Audience: systems developers, graphics application developers.*

Structure from Motion (SfM) and Multi-View Stereo (MVS) reconstruct 3D geometry from multiple camera images. GPU acceleration targets: feature detection and matching (ORB, AKAZE), fundamental matrix estimation via GPU-RANSAC, depth map computation via semi-global matching (SGM), and depth map fusion into a dense point cloud (§9 Poisson surface).

### 88.1 GPU-Accelerated ORB Feature Detection

ORB (Rublee et al. 2011) combines FAST corner detection with binary BRIEF descriptors. The FAST response at each pixel is computed in parallel:

```glsl
// fast_detect.comp — FAST-9 corner response at each pixel
layout(set=0,binding=0) uniform sampler2D grayImg;
layout(set=0,binding=1, r8ui) uniform writeonly uimage2D responseOut;
layout(local_size_x=16,local_size_y=16) in;
const ivec2 CIRCLE9[16] = {ivec2(0,3),ivec2(1,3),ivec2(2,2),ivec2(3,1),
                            ivec2(3,0),ivec2(3,-1),ivec2(2,-2),ivec2(1,-3),
                            ivec2(0,-3),ivec2(-1,-3),ivec2(-2,-2),ivec2(-3,-1),
                            ivec2(-3,0),ivec2(-3,1),ivec2(-2,2),ivec2(-1,3)};
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    float cen = texelFetch(grayImg, pix, 0).r;
    float thresh = 0.1;
    uint bright=0u, dark=0u;
    for (int i=0;i<16;i++) {
        float nb = texelFetch(grayImg, pix+CIRCLE9[i], 0).r;
        if (nb > cen+thresh) bright |= (1u<<i);
        if (nb < cen-thresh) dark   |= (1u<<i);
    }
    // FAST-9: 9 consecutive bits set in 16-bit circle
    bool corner = false;
    for (int i=0;i<16&&!corner;i++) corner = (popcount((bright|(bright<<16))>>(i)&0x1FFu)==9)
                                           ||(popcount((dark  |(dark  <<16))>>(i)&0x1FFu)==9);
    imageStore(responseOut, pix, uvec4(corner?255u:0u));
}
```

[Source: Rublee et al. "ORB: An Efficient Alternative to SIFT or SURF", ICCV 2011; Sinha et al. "GPU-Based Video Feature Tracking And Matching", EDGE 2006]

### 88.2 RANSAC Fundamental Matrix Estimation

GPU RANSAC samples N hypothesis sets in parallel, each computing the 7/8-point fundamental matrix and counting inliers:

```glsl
// ransac_fundamental.comp — parallel RANSAC for fundamental matrix F
layout(set=0,binding=0) readonly buffer Matches { vec4 matches[]; };  // x,y in img A; x,y in img B
layout(set=0,binding=1) readonly buffer Seeds   { uint seeds[][8]; };  // random 8-point sets
layout(set=0,binding=2) buffer InlierCount { uint cnt[]; };
layout(set=0,binding=3) buffer BestF { mat3 F[]; };
layout(push_constant) uniform PC { uint N_HYPOTHESES; uint N_MATCHES; float thresh; } pc;
layout(local_size_x=64) in;
void main() {
    uint h   = gl_GlobalInvocationID.x;
    if (h >= pc.N_HYPOTHESES) return;
    // Compute F from 8 sampled correspondences via DLT (8-point algorithm)
    mat3 F   = compute8PointF(matches, seeds[h]);
    uint inl = 0u;
    for (uint m=0; m<pc.N_MATCHES; m++) {
        vec3 x1 = vec3(matches[m].xy,1), x2 = vec3(matches[m].zw,1);
        float err = abs(dot(x2, F*x1));  // Sampson distance
        if (err < pc.thresh) inl++;
    }
    cnt[h]   = inl;
    F[h]     = F;
}
// CPU: find argmax(cnt), polish with all inliers → final F
```

[Source: Hartley & Zisserman "Multiple View Geometry in Computer Vision", 2004; GPU RANSAC: Ni et al. "Out-of-core Bundle Adjustment for Large-scale 3D Reconstruction", ICCV 2007]

### 88.3 Semi-Global Matching (SGM) Depth Estimation

SGM (Hirschmüller 2008) computes dense stereo depth maps by aggregating matching costs along multiple scan-line paths. GPU parallelism runs all rows simultaneously:

```glsl
// sgm_cost.comp — compute matching cost volume (SAD-based)
layout(set=0,binding=0) uniform sampler2D imgL, imgR;
layout(set=0,binding=1) writeonly buffer CostVol { float cost[]; };  // [H][W][DISP_RANGE]
layout(push_constant) uniform PC { int dispRange; int winSz; } pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    for (int d=0; d<pc.dispRange; d++) {
        float sad = 0.0;
        for (int dy=-pc.winSz;dy<=pc.winSz;dy++)
            for (int dx=-pc.winSz;dx<=pc.winSz;dx++) {
                float vL = texelFetch(imgL, pix+ivec2(dx,dy), 0).r;
                float vR = texelFetch(imgR, pix+ivec2(dx-d,dy), 0).r;
                sad += abs(vL-vR);
            }
        cost[(pix.y*WIDTH+pix.x)*pc.dispRange+d] = sad;
    }
}
// SGM aggregation: for each direction (8 paths), run dynamic programming along path
```

[Source: Hirschmüller "Stereo Processing by Semiglobal Matching and Mutual Information", IEEE TPAMI 2008; GPU SGM: Hernandez-Juarez et al. "Embedded Real-Time Stereo Estimation via Semi-Global Matching on the GPU", CVPRW 2016]

### 88.4 Depth Map Fusion into Point Cloud

Fuse per-image depth maps into a consistent dense point cloud using confidence-weighted averaging, then optionally feed into §9 Poisson surface reconstruction:

```glsl
// depth_fuse.comp — back-project depth map pixels into world-space point cloud
layout(set=0,binding=0) uniform sampler2D depthMap;
layout(set=0,binding=1) readonly buffer Confidence { float conf[]; };
layout(set=0,binding=2) buffer PointCloud { vec3 pts[]; vec3 nrm[]; float w[]; };
layout(set=0,binding=3) buffer PointCount { uint count; };
layout(push_constant) uniform PC { mat4 invViewProj; mat3 KInv; float confThresh; } pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    float d    = texelFetch(depthMap, pix, 0).r;
    float c    = conf[pix.y*WIDTH+pix.x];
    if (d <= 0.0 || c < pc.confThresh) return;
    vec4  ws   = pc.invViewProj * vec4(vec2(pix)/vec2(WIDTH,HEIGHT)*2-1, d*2-1, 1);
    ws /= ws.w;
    uint slot  = atomicAdd(count, 1u);
    pts[slot]  = ws.xyz;
    nrm[slot]  = computeNormalFromDepth(depthMap, pix, pc.KInv);
    w[slot]    = c;
}
```

[Source: Merrell et al. "Real-Time Visibility-Based Fusion of Depth Maps", ICCV 2007; Schönberger & Frahm "Structure-from-Motion Revisited", CVPR 2016]

---

### 89. 3D Feature Descriptors and Pose Estimation

*Audience: systems developers, graphics application developers.*

3D feature descriptors encode local geometry around a keypoint into a compact vector used for point cloud matching and pose estimation. GPU algorithms compute Fast Point Feature Histograms (FPFH) and SHOT descriptors in parallel across all keypoints, then use GPU-RANSAC (§88.2 pattern) to estimate the 6-DOF rigid transform aligning two point clouds.

### 89.1 Normal Estimation via PCA

Before computing descriptors, estimate the surface normal at each point from its k-NN via PCA. The smallest eigenvector of the 3×3 covariance matrix of the neighbour positions is the normal:

```glsl
// normal_pca.comp — PCA normal estimation from k-NN
layout(set=0,binding=0) readonly buffer Pts { vec3 pts[]; };
layout(set=0,binding=1) readonly buffer KNN { uint knn[K_NN]; };  // k-NN indices per point
layout(set=0,binding=2) writeonly buffer Normals { vec3 nor[]; };
layout(local_size_x=64) in;
void main() {
    uint p  = gl_GlobalInvocationID.x;
    vec3 c  = pts[p];
    for (uint k=0;k<K_NN;k++) c += pts[knn[p*K_NN+k]];
    c /= float(K_NN+1);
    mat3 Cov = mat3(0.0);
    for (uint k=0;k<K_NN;k++) {
        vec3 d = pts[knn[p*K_NN+k]] - c;
        Cov   += outerProduct(d,d);
    }
    // Smallest eigenvalue of Cov → normal (power iteration on inverse)
    vec3 n = smallestEigenvec3x3(Cov);
    // Orient toward view point (flip if dot < 0)
    nor[p] = dot(n, VIEW_ORIGIN - pts[p]) > 0.0 ? n : -n;
}
```

[Source: Rusu et al. "Towards 3D Point Cloud Based Object Maps for Household Environments", Robotics and Autonomous Systems 2008]

### 89.2 FPFH Descriptor Computation

FPFH (Rusu et al. 2009) encodes the pairwise angular relationships between a point and its k-neighbours into a 33-dimensional histogram:

```glsl
// fpfh.comp — compute FPFH descriptor for each keypoint
layout(set=0,binding=0) readonly buffer Pts { vec3 pts[]; };
layout(set=0,binding=1) readonly buffer Nor { vec3 nor[]; };
layout(set=0,binding=2) readonly buffer KNN { uint knn[]; };
layout(set=0,binding=3) writeonly buffer FPFH { float fpfh[N_PTS][33]; };
layout(local_size_x=64) in;
void main() {
    uint p  = gl_GlobalInvocationID.x;
    float hist[33] = {0.0};
    for (uint k=0; k<K_NN; k++) {
        uint q   = knn[p*K_NN+k];
        vec3 d   = normalize(pts[q]-pts[p]);
        // Darboux frame angles: α, φ, θ
        vec3 u   = nor[p];
        vec3 v   = cross(d, u);
        vec3 w   = cross(u, v);
        float alpha = dot(v, nor[q]);
        float phi   = dot(u, d);
        float theta = atan(dot(w,nor[q]), dot(u,nor[q]));
        // Bin into 11-bin histograms for each angle
        hist[int((alpha+1.0)*5.0)]     += 1.0/float(K_NN);
        hist[11+int((phi+1.0)*5.0)]    += 1.0/float(K_NN);
        hist[22+int((theta+3.14159)*11.0/6.28318)] += 1.0/float(K_NN);
    }
    for (int i=0;i<33;i++) fpfh[p][i] = hist[i];
}
```

[Source: Rusu et al. "Fast Point Feature Histograms (FPFH) for 3D Registration", ICRA 2009]

### 89.3 Descriptor Matching via GPU k-NN

Match FPFH descriptors between two point clouds by finding the nearest-neighbour in descriptor space. Brute-force is O(N·M·33); approximate GPU k-NN via product quantization for large clouds:

```glsl
// fpfh_match.comp — brute-force nearest-neighbour matching in FPFH space
layout(set=0,binding=0) readonly buffer FPFH_A { float fA[N_A][33]; };
layout(set=0,binding=1) readonly buffer FPFH_B { float fB[N_B][33]; };
layout(set=0,binding=2) writeonly buffer Matches { uint match[N_A]; float matchDist[N_A]; };
layout(local_size_x=64) in;
void main() {
    uint i   = gl_GlobalInvocationID.x;
    float bestD = 1e30; uint bestJ = 0u;
    for (uint j=0; j<N_B; j++) {
        float d = 0.0;
        for (int k=0; k<33; k++) { float diff=fA[i][k]-fB[j][k]; d+=diff*diff; }
        if (d < bestD) { bestD=d; bestJ=j; }
    }
    match[i]=bestJ; matchDist[i]=bestD;
}
```

[Source: Rusu et al. 2009; GPU ANN: Johnson & Hebert "Using Spin Images for Efficient Object Recognition in Cluttered 3D Scenes", IEEE TPAMI 1999]

### 89.4 6-DOF Pose Estimation via GPU-RANSAC

Use the §88.2 GPU-RANSAC pattern with 3-point samples (minimum for a rigid transform) to estimate the transformation matrix aligning the two point clouds:

```glsl
// pose_ransac.comp — GPU-RANSAC for 6-DOF point cloud alignment
layout(set=0,binding=0) readonly buffer Pts_A { vec3 pA[]; };
layout(set=0,binding=1) readonly buffer Pts_B { vec3 pB[]; };
layout(set=0,binding=2) readonly buffer Matches { uint match[]; };
layout(set=0,binding=3) readonly buffer Seeds   { uint seeds[][3]; };
layout(set=0,binding=4) buffer InlierCount { uint cnt[]; };
layout(set=0,binding=5) buffer BestTransform { mat4 T[]; };
layout(push_constant) uniform PC { uint N_HYPO; uint N_MATCHES; float thresh; } pc;
layout(local_size_x=64) in;
void main() {
    uint h  = gl_GlobalInvocationID.x;
    if (h >= pc.N_HYPO) return;
    // Compute rigid transform from 3 point correspondences (SVD-based)
    mat4 Th = computeRigidTransform3(pA, pB, match, seeds[h]);
    uint inl = 0u;
    for (uint m=0; m<pc.N_MATCHES; m++) {
        vec3 ta = (Th * vec4(pA[m],1.0)).xyz;
        if (length(ta - pB[match[m]]) < pc.thresh) inl++;
    }
    cnt[h]=inl; T[h]=Th;
}
// CPU: pick argmax(cnt), refine with all inliers via ICP (§87)
```

[Source: Rusu et al. "Fast Global Registration", ECCV 2016; Fischler & Bolles "Random Sample Consensus: A Paradigm for Model Fitting", CACM 1981]

---

### 90. Iterative Closest Point (ICP) Registration

*Audience: systems developers, graphics application developers.*

ICP (Besl & McKay 1992) aligns two point clouds by alternating: (1) find nearest-neighbour correspondences; (2) compute the rigid transform minimising their sum-squared distance. GPU parallelism covers both steps. §89 covers FPFH-based initialisation; ICP refines the result.

### 90.1 GPU k-NN Correspondence Search

For each point in the source cloud, find the nearest point in the target cloud. Brute-force on GPU: O(N·M) dot products, but with SIMD parallelism across 10s of millions of pairs:

```glsl
// icp_nn.comp — nearest-neighbour correspondences between source and target
layout(set=0,binding=0) readonly buffer Src { vec3 src[]; };
layout(set=0,binding=1) readonly buffer Tgt { vec3 tgt[]; };
layout(set=0,binding=2) writeonly buffer Corr { uint corrIdx[]; float corrDist[]; };
layout(local_size_x=64) in;
void main() {
    uint i   = gl_GlobalInvocationID.x;
    if (i >= N_SRC) return;
    float bestD=1e30; uint bestJ=0u;
    for (uint j=0; j<N_TGT; j++) {
        float d = dot(src[i]-tgt[j],src[i]-tgt[j]);
        if (d<bestD) { bestD=d; bestJ=j; }
    }
    corrIdx[i]=bestJ; corrDist[i]=sqrt(bestD);
}
```

[Source: Besl & McKay "A Method for Registration of 3-D Shapes", IEEE TPAMI 1992; GPU ICP: Tamaki et al. "Softassign and EM-ICP on GPU", IVCNZ 2010]

### 90.2 Cross-Covariance Matrix via Parallel Reduction

From N correspondence pairs (pᵢ, qᵢ), compute the cross-covariance H = Σ (pᵢ − p̄)(qᵢ − q̄)ᵀ needed for SVD. Use a parallel reduction:

```glsl
// icp_crosscov.comp — parallel reduction of cross-covariance matrix
layout(set=0,binding=0) readonly buffer Src  { vec3 src[];  };
layout(set=0,binding=1) readonly buffer Corr { uint corrIdx[]; };
layout(set=0,binding=2) readonly buffer Tgt  { vec3 tgt[];  };
layout(set=0,binding=3) readonly buffer Means { vec3 srcMean; vec3 tgtMean; };
layout(set=0,binding=4) buffer H { float H[9]; };  // 3x3 cross-covariance
layout(local_size_x=64) in;
shared float sH[64][9];
void main() {
    uint i  = gl_GlobalInvocationID.x;
    uint li = gl_LocalInvocationID.x;
    mat3 local_H = mat3(0.0);
    if (i < N_SRC) {
        vec3 p = src[i] - srcMean;
        vec3 q = tgt[corrIdx[i]] - tgtMean;
        local_H = outerProduct(p,q);
    }
    // Store to shared, reduce
    for (int k=0;k<9;k++) sH[li][k] = local_H[k/3][k%3];
    barrier();
    for (uint s=32u;s>0u;s>>=1) {
        if (li<s) for(int k=0;k<9;k++) sH[li][k]+=sH[li+s][k];
        barrier();
    }
    if (li==0) for(int k=0;k<9;k++) atomicAdd_float(H[k], sH[0][k]);
}
```

[Source: Besl & McKay 1992; Segal & Akeley "The OpenGL Graphics System", 2004 §ICP appendix]

### 90.3 SVD-Based Transform Estimation

The optimal rotation is R = V·Uᵀ where H = U·Σ·Vᵀ. Compute on CPU after the GPU reduction for a 3×3 matrix (negligible cost); translation is t = q̄ − R·p̄:

```glsl
// icp_transform_apply.comp — apply current R,t to source point cloud
layout(set=0,binding=0) readonly buffer SrcIn  { vec3 srcIn[];  };
layout(set=0,binding=1) writeonly buffer SrcOut { vec3 srcOut[]; };
layout(set=0,binding=2) readonly buffer RT { mat3 R; vec3 t; };
layout(local_size_x=64) in;
void main() {
    uint i    = gl_GlobalInvocationID.x;
    if (i >= N_SRC) return;
    srcOut[i] = R * srcIn[i] + t;
}
// Iterate: dispatch icp_nn → icp_crosscov → CPU SVD → icp_transform_apply
// Converges in 20-50 iterations for ≤45° initial misalignment
```

### 90.4 Point-to-Plane ICP for Faster Convergence

Point-to-plane ICP (Chen & Medioni 1992) minimises the sum of squared distances along the target normal, converging roughly 10× faster than point-to-point. The optimal transform is solved by a 6×6 linear system:

```glsl
// icp_point2plane.comp — accumulate 6x6 linear system for point-to-plane ICP
layout(set=0,binding=0) readonly buffer Src     { vec3 src[];  };
layout(set=0,binding=1) readonly buffer TgtPts  { vec3 tgt[];  };
layout(set=0,binding=2) readonly buffer TgtNors { vec3 tnor[]; };
layout(set=0,binding=3) readonly buffer Corr    { uint corrIdx[]; };
layout(set=0,binding=4) buffer AtA { float ata[36]; float atb[6]; };  // 6x6 system
layout(local_size_x=64) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    if (i>=N_SRC) return;
    vec3 p  = src[i], q = tgt[corrIdx[i]], n = tnor[corrIdx[i]];
    // Row of A: [n × p, n]
    vec3 cn = cross(p, n);
    float a[6] = {cn.x,cn.y,cn.z,n.x,n.y,n.z};
    float b    = dot(n, q-p);
    for (int r=0;r<6;r++) {
        atomicAdd_float(atb[r], a[r]*b);
        for(int c=r;c<6;c++) atomicAdd_float(ata[r*6+c], a[r]*a[c]);
    }
}
// Solve 6x6 AtA·x = Atb on CPU → x = [ω₁,ω₂,ω₃,t₁,t₂,t₃] → compose transform
```

[Source: Chen & Medioni "Object Modeling by Registration of Multiple Range Images", IVC 1992; Low "Linear Least-Squares Optimization for Point-to-Plane ICP Surface Registration", UNC TR 2004]

---


---

## XI. Neural and Learned Geometry

Neural geometry methods replace hand-designed algorithms with learned representations trained on image or geometry supervision. 3D Gaussian Splatting (§91 provides the conceptual overview; §94 covers the full implementation including the differentiable rasterizer and densification) renders scenes as oriented Gaussian splats fitted to multi-view images; Neural Radiance Fields (NeRF) and neural SDFs represent geometry as MLPs queried at continuous 3D positions; Geometric Deep Learning applies graph neural networks and SE(3)-equivariant architectures to mesh classification and segmentation; and differentiable rendering enables gradient-based inverse problems — recovering geometry, materials, and lighting from image observations. These workloads run as CUDA or Vulkan compute dispatches and typically require tensor core acceleration.

### 91. 3D Gaussian Splatting and Neural Geometry

*Audience: graphics application developers.*

3DGS (Kerbl et al. 2023) replaces triangle meshes with a set of 3D Gaussian primitives, each parameterised by position, covariance, opacity, and view-dependent colour encoded in spherical harmonics. Rendering is a differentiable rasterization pipeline that has overtaken NeRF for real-time novel-view synthesis. The GPU pipeline is fully Vulkan/compute-compatible.

### 91.1 3DGS Representation and Projection

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

### 91.2 Tile Rasterizer: Sort-by-Depth and Alpha Compositing

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

### 91.3 Mesh Extraction from 3DGS and NeRF

Converting a trained 3DGS scene to a triangle mesh enables traditional rendering pipelines and physics simulation. Two approaches:

**SuGaR** (Guédon & Lepetit 2023) adds a regularisation term during training that aligns Gaussians to surface sheets, then extracts a mesh by Poisson reconstruction (§3.12) from Gaussian centers weighted by opacity.

**NeRF → mesh** (using Instant NGP): marching cubes on the NeRF density field at a threshold σ_surface. The density grid is evaluated at each voxel center in a compute dispatch, then piped into the two-pass GPU MC pipeline from §3.2. Texture maps are baked by computing the NeRF colour at each surface point (§10.13 GPU texture baking pipeline).

**Occupancy networks** predict binary occupancy o(x) ∈ {0,1} for any query point. Run inference on a 256³ voxel grid in parallel (batched MLP evaluation in compute), then extract the isosurface with GPU marching cubes. [Source: Guédon & Lepetit "SuGaR: Surface-Aligned Gaussian Splatting", CVPR 2024; Müller et al. "Instant Neural Graphics Primitives", SIGGRAPH 2022]

---

### 92. Geometric Deep Learning

*Audience: graphics application developers, systems developers.*

Geometric deep learning runs neural network inference directly on point clouds and triangle meshes, enabling semantic segmentation, shape completion, and neural implicit surface extraction on the same GPU that renders the scene.

### 92.1 PointNet on GPU

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

Global max-pool over all points aggregates per-point features into a global descriptor via a parallel reduce (same pattern as §48.3). [Source: Qi et al. "PointNet", CVPR 2017; https://github.com/charlesq34/pointnet]

### 92.2 Graph Convolution on Mesh Edge Graphs

Graph neural network message passing on mesh edges — structurally identical to the cotangent Laplacian evaluation (§34.1) with learned weights:

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

### 92.3 Neural SDF Inference on GPU

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

### 92.4 Feature Distillation from 3DGS to Mesh Atlas

Bake semantic/appearance features from a trained 3DGS scene (§91) into a mesh UV atlas (§10.13) for downstream editing or segmentation:

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

### 93. Differentiable Rendering and Inverse Geometry

*Audience: graphics application developers, systems developers.*

Differentiable rendering computes the gradient ∂L/∂θ of a scalar loss L (e.g., image reconstruction error) with respect to scene geometry parameters θ (vertex positions, SDF values, Gaussian means). This enables gradient-based geometry optimisation: mesh fitting to photographs, geometry parameter tuning, neural inverse rendering.

### 93.1 Differentiable Rasterization: Soft Visibility

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

### 93.2 Differentiable Ray Tracing: Geometry Gradients

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

### 93.3 3DGS Gradient Flow

For 3D Gaussian splatting (§91), the backward pass of the tile rasterizer propagates image-space gradients back to per-Gaussian mean μ, covariance Σ, opacity α, and spherical harmonic coefficients. The forward-pass tile rasterizer stores per-pixel transmittance T for the backward pass:

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

### 93.4 Geometry Optimisation Loop

The full loop: forward render → compute loss → backward pass → gradient descent step on vertex positions/SDF values:

```python
# Host-side loop (pseudo-code; actual dispatch via Vulkan compute)
for iteration in range(MAX_ITERS):
    vkCmdDispatch(forward_render)      # §93.1 soft rasteriser or §75 RT
    vkCmdDispatch(loss_compute)        # MSE / perceptual loss
    vkCmdDispatch(backward_pass)       # §93.2 or §93.3 gradient accumulation
    vkCmdDispatch(adam_update)         # Adam: m = β1*m + (1-β1)*g; ...
    # After convergence: extract mesh from optimised SDF via §3.2 GPU MC
```

Adam per-parameter momentum/variance update also runs in compute — one thread per vertex/voxel/Gaussian. [Source: Kingma & Ba "Adam: A Method for Stochastic Optimization", ICLR 2015]

---

### 94. 3D Gaussian Splatting

*Audience: graphics application developers.*

3D Gaussian Splatting (3DGS, Kerbl et al. 2023) represents a scene as a set of anisotropic 3D Gaussians — each with a position, covariance matrix, opacity, and spherical harmonic colour coefficients — and rasterizes them via a GPU tile-based alpha compositing pipeline. It achieves real-time novel-view synthesis quality matching NeRF at 30–120 fps without ray marching.

### 94.1 Gaussian Representation and SH Coefficients

Each Gaussian stores: centre μ ∈ ℝ³, scale s ∈ ℝ³, rotation quaternion q ∈ ℝ⁴ (which together define the 3D covariance Σ = R·S·Sᵀ·Rᵀ), opacity α ∈ [0,1], and spherical harmonic coefficients for view-dependent colour (up to degree 3, 48 floats per Gaussian):

```glsl
// gaussian_project.comp — project 3D Gaussians to 2D screen-space ellipses
struct Gaussian3D {
    vec3  mu;       // centre
    vec4  q;        // rotation quaternion
    vec3  s;        // scale (log space)
    float alpha;
    float sh[48];   // SH colour coefficients (degree 0–3)
};
layout(set=0,binding=0) readonly buffer Gaussians { Gaussian3D g[]; };
layout(set=0,binding=1) writeonly buffer Projected { Gaussian2D proj[]; };
layout(set=0,binding=2) writeonly buffer Keys { uint64_t tileDepthKey[]; };
layout(push_constant) uniform PC { mat4 view; mat4 proj; vec2 focalLen; uint W; } pc;
layout(local_size_x=64) in;
void main() {
    uint i   = gl_GlobalInvocationID.x;
    Gaussian3D gi = g[i];
    // Build 3D covariance from scale+rotation
    mat3 S   = mat3(exp(gi.s.x),0,0, 0,exp(gi.s.y),0, 0,0,exp(gi.s.z));
    mat3 R   = quatToMat3(gi.q);
    mat3 Sig = R * S * transpose(S) * transpose(R);
    // Project to 2D: Σ' = J·W·Σ·Wᵀ·Jᵀ  (Zwicker et al. 2001 EWA splatting)
    mat3 W   = mat3(pc.view);
    vec3 tc  = (pc.view * vec4(gi.mu,1.0)).xyz;
    mat3x2 J = mat3x2(pc.focalLen.x/tc.z, 0,
                       0, pc.focalLen.y/tc.z,
                       -pc.focalLen.x*tc.x/(tc.z*tc.z), -pc.focalLen.y*tc.y/(tc.z*tc.z));
    mat2 cov2D = mat2(J * W * Sig * transpose(W) * transpose(J));
    // Tile key: sort by tile (high 32 bits) then depth (low 32 bits)
    vec4 clip  = pc.proj * vec4(tc,1.0);
    vec2 ndc   = clip.xy / clip.w;
    ivec2 tile = ivec2((ndc*0.5+0.5) * vec2(pc.W/TILE_SZ));
    tileDepthKey[i] = (uint64_t(tile.y*TILES_X+tile.x) << 32) | floatBitsToUint(clip.w);
    proj[i] = Gaussian2D(ndc, cov2D, evalSH(gi.sh, -normalize(tc)), gi.alpha);
}
```

[Source: Kerbl et al. "3D Gaussian Splatting for Real-Time Radiance Field Rendering", SIGGRAPH 2023; Zwicker et al. "EWA Splatting", IEEE TVCG 2002]

### 94.2 Tile-Based Sorting and Rasterization

After projection, sort Gaussians by tile+depth key using GPU radix sort (§25.4), then rasterize each tile in a compute shader doing back-to-front alpha compositing:

```glsl
// gaussian_raster.comp — tile-based alpha compositing of sorted Gaussians
layout(set=0,binding=0) readonly buffer SortedIdx  { uint idx[]; };
layout(set=0,binding=1) readonly buffer Projected  { Gaussian2D proj[]; };
layout(set=0,binding=2) readonly buffer TileRanges { uvec2 ranges[]; };  // [start,end) per tile
layout(set=0,binding=3, rgba16f) uniform writeonly image2D outImg;
layout(local_size_x=TILE_SZ,local_size_y=TILE_SZ) in;
shared Gaussian2D tileCache[MAX_GAUSSIANS_PER_TILE];
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    uint  tile = gl_WorkGroupID.y * TILES_X + gl_WorkGroupID.x;
    uvec2 rng  = ranges[tile];
    vec3  C    = vec3(0.0);
    float T    = 1.0;  // transmittance
    for (uint b=rng.x; b<rng.y && T>0.001; b++) {
        Gaussian2D gs = proj[idx[b]];
        vec2  d   = vec2(pix) - gs.center * 0.5 * vec2(imageSize(outImg));
        // Evaluate 2D Gaussian: power = exp(-0.5 * dᵀ Σ⁻¹ d)
        mat2  Si  = inverse(gs.cov2D + mat2(0.3));  // add low-pass filter
        float pwr = exp(-0.5 * dot(d, Si * d));
        float a   = min(0.99, gs.alpha * pwr);
        C  += T * a * gs.color;
        T  *= (1.0 - a);
    }
    imageStore(outImg, pix, vec4(C + T*BACKGROUND_COLOR, 1.0));
}
```

[Source: Kerbl et al. 2023; implementation reference: https://github.com/graphdeco-inria/gaussian-splatting]

### 94.3 Gaussian Culling and LOD

For large scenes (millions of Gaussians), cull those outside the view frustum and those below a screen-space size threshold in the projection pass:

```glsl
// gaussian_cull.comp — frustum cull + size cull before projection
layout(set=0,binding=0) readonly buffer Gaussians { Gaussian3D g[]; };
layout(set=0,binding=1) writeonly buffer Visible  { uint visIdx[]; };
layout(set=0,binding=2) buffer VisCount { uint count; };
layout(push_constant) uniform PC { mat4 viewProj; float minScreenArea; } pc;
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    vec4 c = pc.viewProj * vec4(g[i].mu, 1.0);
    vec3 s = exp(g[i].s);
    float maxS = max(s.x, max(s.y, s.z));
    float screenSz = maxS / c.w;  // rough screen-space radius estimate
    // Frustum cull: NDC check with Gaussian radius
    if (c.w < 0.01 || abs(c.x/c.w) > 1.3 || abs(c.y/c.w) > 1.3) return;
    if (screenSz < pc.minScreenArea) return;
    uint slot = atomicAdd(count, 1u);
    visIdx[slot] = i;
}
```

[Source: Ye et al. "AbsGS: Recovering Fine Details for 3D Gaussian Splatting", 2024; Mallick et al. "Taming 3DGS: High-Quality Radiance Fields with Limited Resources", SIGGRAPH Asia 2024]

### 94.4 Gaussian Densification and Pruning

During training (offline, CUDA), over-reconstructed regions require splitting large Gaussians and cloning small ones; under-reconstructed regions require pruning transparent Gaussians. The GPU densification kernel:

```cuda
// gaussian_densify.cu — adaptive density control (CUDA training pass)
__global__ void densifyAndPrune(
    Gaussian3D* g, float* grad2D, float* maxRadii,
    uint* splitMask, uint* cloneMask, uint* pruneMask,
    float gradThresh, float sizeThresh, float alphaThresh, int N)
{
    int i = blockIdx.x*blockDim.x + threadIdx.x;
    if (i >= N) return;
    float g2  = grad2D[i];
    float sz  = max(exp(g[i].s.x), max(exp(g[i].s.y), exp(g[i].s.z)));
    if (g[i].alpha < alphaThresh) { pruneMask[i] = 1; return; }
    if (g2 > gradThresh) {
        if (sz > sizeThresh) splitMask[i] = 1;  // large + high grad → split
        else                 cloneMask[i] = 1;   // small + high grad → clone
    }
}
```

[Source: Kerbl et al. 2023 §5.2; https://github.com/graphdeco-inria/gaussian-splatting/blob/main/scene/gaussian_model.py]

---

### 95. Neural Geometry: NeRF and Neural Implicit Surfaces

*Audience: graphics application developers.*

Neural Radiance Fields (NeRF, Mildenhall et al. 2020) and neural signed distance functions (DeepSDF, Park et al. 2019) represent geometry implicitly as the weights of a neural network. At inference time, GPU shaders evaluate these networks along camera rays, producing novel views or surface geometry without an explicit triangle mesh. Instant-NGP (Müller et al. 2022) replaces the large MLP with a multiresolution hash grid, achieving real-time NeRF rendering.

### 95.1 NeRF: Volume Rendering with MLP Inference

A NeRF MLP maps (x, y, z, θ, φ) → (colour, density). Each camera ray is sampled at N stratified depths and the MLP is evaluated at each sample, then composited via the volume rendering equation (§67.1):

```glsl
// nerf_inference.comp — evaluate NeRF MLP for one camera ray
layout(set=0,binding=0) readonly buffer MLPWeights { float W0[]; float W1[]; float W2[]; };  // layer weights
layout(set=0,binding=1, rgba16f) uniform writeonly image2D renderOut;
layout(push_constant) uniform PC { mat4 invView; vec2 fov; uint W; uint H; uint N_SAMPLES; } pc;
layout(local_size_x=8,local_size_y=8) in;

vec4 evalMLP(vec3 pos, vec3 dir) {
    // Positional encoding: [sin(2^k π x), cos(2^k π x)] for k=0..9
    float feat[63];  // 3 + 2*3*10 = 63 input features (pos encode)
    posEncode(pos, dir, feat);
    // Forward pass: 8-layer ReLU MLP (256 hidden units)
    float h[256];
    for (int l=0; l<8; l++) matmulReLU(feat, W0 + l*256*256, h, 256);
    return vec4(sigmoid(h[0]), sigmoid(h[1]), sigmoid(h[2]),  // RGB
                relu(h[3]));                                    // density σ
}
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    vec3  ro   = (pc.invView * vec4(0,0,0,1)).xyz;
    vec3  rd   = normalize((pc.invView * vec4(pixelDir(pix, pc.fov),1)).xyz);
    vec3  C    = vec3(0); float T = 1.0;
    float tNear=0.1, tFar=10.0, dt=(tFar-tNear)/float(pc.N_SAMPLES);
    for (uint s=0; s<pc.N_SAMPLES && T>0.01; s++) {
        float t = tNear + (float(s)+0.5)*dt;
        vec4 rgbσ = evalMLP(ro+rd*t, rd);
        float a   = 1.0 - exp(-rgbσ.w*dt);
        C += T * a * rgbσ.rgb;
        T *= (1.0-a);
    }
    imageStore(renderOut, pix, vec4(C,1));
}
```

[Source: Mildenhall et al. "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis", ECCV 2020; https://www.matthewtancik.com/nerf]

### 95.2 Instant-NGP: Multiresolution Hash Grid

Instant-NGP (Müller et al. 2022) replaces the large positional encoding + deep MLP with a multi-resolution hash table of learnable feature vectors (16 levels, 2^19 entries each, 2 features per entry), followed by a tiny 2-layer MLP. GPU inference uses CUDA/Vulkan shared memory to batch hash lookups:

```glsl
// ngp_hash_encode.comp — lookup hash grid features for a batch of 3D positions
layout(set=0,binding=0) readonly buffer HashGrid { vec2 entries[N_LEVELS][HASH_SIZE]; };
layout(set=0,binding=1) readonly buffer Positions { vec3 pts[]; };
layout(set=0,binding=2) writeonly buffer Features { float feats[]; };  // N_LEVELS*2 per point
layout(push_constant) uniform PC { float baseRes; float perLevelScale; uint N; } pc;
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    if (i >= pc.N) return;
    vec3 p  = pts[i];
    for (int lv=0; lv<N_LEVELS; lv++) {
        float res = pc.baseRes * pow(pc.perLevelScale, float(lv));
        vec3  psc = p * res;
        ivec3 c0  = ivec3(floor(psc));
        vec3  f   = fract(psc);
        // Trilinear interpolation of 8 hash entries
        vec2  feat = vec2(0.0);
        for (int dz=0;dz<=1;dz++) for(int dy=0;dy<=1;dy++) for(int dx=0;dx<=1;dx++) {
            ivec3 c  = c0 + ivec3(dx,dy,dz);
            uint  h  = (c.x*2654435761u ^ c.y*805459861u ^ c.z*3674653429u) % HASH_SIZE;
            float w  = mix(dx?f.x:1-f.x, 0, 0) * mix(dy?f.y:1-f.y,0,0)
                     * (dz ? f.z : 1.0-f.z);
            feat    += w * entries[lv][h];
        }
        feats[i*N_LEVELS*2 + lv*2 + 0] = feat.x;
        feats[i*N_LEVELS*2 + lv*2 + 1] = feat.y;
    }
}
```

[Source: Müller et al. "Instant Neural Graphics Primitives with a Multiresolution Hash Encoding", SIGGRAPH 2022; https://github.com/NVlabs/instant-ngp]

### 95.3 Neural SDF: DeepSDF Inference

DeepSDF (Park et al. 2019) represents a shape as φ(x) = MLP(x, z) where z is a learned latent code. At inference, evaluate the MLP at each query point. Combined with §65 marching cubes on the neural SDF grid, this produces a mesh:

```glsl
// deepsdf_infer.comp — evaluate DeepSDF MLP on a voxel grid for isosurface extraction
layout(set=0,binding=0) readonly buffer SDFWeights { float layers[N_LAYERS][512][512+3]; };
layout(set=0,binding=1) readonly buffer LatentCode { float z[256]; };
layout(set=0,binding=2, r32f) uniform writeonly image3D sdfGrid;
layout(push_constant) uniform PC { vec3 gridMin; float dx; } pc;
layout(local_size_x=4,local_size_y=4,local_size_z=4) in;
void main() {
    ivec3 id = ivec3(gl_GlobalInvocationID);
    vec3  p  = pc.gridMin + (vec3(id)+0.5)*pc.dx;
    // Concatenate [p, z] as input to 8-layer 512-unit MLP
    float h[512]; buildInput(p, z, h);
    for (int l=0; l<8; l++) {
        if (l == 4) addSkip(p, z, h);  // skip connection at layer 4
        matmulReLU(h, layers[l], h, 512);
    }
    imageStore(sdfGrid, id, vec4(tanh(h[0])));  // output: signed distance
}
// Then run §65.1 marching cubes on the sdfGrid to extract the surface
```

[Source: Park et al. "DeepSDF: Learning Continuous Signed Distance Functions for Shape Representation", CVPR 2019]

### 95.4 NeuS: Neural Implicit Surface for Reconstruction

NeuS (Wang et al. 2021) learns an SDF-parameterised volume density function s(φ) = sigmoid(−φ/ε) that correctly renders the zero-level-set surface from multi-view images. The rendering integral uses the SDF value to bias the density toward the surface:

```glsl
// neus_render.comp — NeuS volume rendering with SDF-derived density
// After training, inference evaluates the same §95.3 MLP + colour network
// The key difference from NeRF (§95.1) is the density function:
// ρ(t) = max(-dφ/dt, 0) * sigmoid(-φ/ε) / (sigmoid(-φ/ε) + sigmoid(φ/ε))
// This makes the surface lie exactly on the SDF zero crossing.
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    vec3  ro  = cameraPos, rd = rayDir(pix);
    vec3  C   = vec3(0); float T = 1.0;
    float prevPhi = evalSDF_MLP(ro + rd*T_NEAR);
    for (uint s=0; s<N_SAMPLES; s++) {
        float t    = T_NEAR + float(s)*DT;
        float phi  = evalSDF_MLP(ro + rd*t);
        float dphi = (phi - prevPhi) / DT;
        float rho  = max(-dphi,0.0) * sigmoid(-phi/EPSILON) /
                     (sigmoid(-phi/EPSILON) + sigmoid(phi/EPSILON) + 1e-5);
        float a    = 1.0 - exp(-rho*DT);
        C += T * a * evalColor_MLP(ro+rd*t, rd);
        T *= (1.0-a); prevPhi=phi;
    }
    imageStore(renderOut, pix, vec4(C,1));
}
```

[Source: Wang et al. "NeuS: Learning Neural Implicit Surfaces by Volume Rendering for Multi-view Reconstruction", NeurIPS 2021]

---


---

## XII. Specialized and Cross-Domain Applications

This category collects geometry algorithms that serve specific application domains rather than the general rendering pipeline. Scientific visualization renders isosurfaces and streamlines from simulation scalar and vector fields; acoustic ray tracing models sound propagation geometry; molecular surface computation (Connolly, SAS) supports biochemistry visualization; VR/XR reprojection geometry warps previously rendered frames to reduce latency; GPU SDF font rendering produces crisp text at all scales without rasterization artefacts; micro-polygon displacement adds film-quality surface detail via tessellation; silhouette detection drives both NPR rendering and shadow volume construction; and GPU texture synthesis generates tileable detail patterns. Each draws on core GPU geometry primitives but targets a narrow, well-defined application context.

### 96. Scientific Visualization Geometry

*Audience: systems developers, graphics application developers.*

Simulation output — velocity fields, scalar fields, tensor fields — requires geometry algorithms to extract, integrate, and render meaningful visual structures. All compute patterns here reuse infrastructure from earlier sections: RK4 integration, marching cubes, mesh shaders.

### 96.1 Streamline and Pathline Integration

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

Tube geometry (§4.3) converts streamlines to renderable ribbons. [Source: Jobard & Lefer "Creating Evenly-Spaced Streamlines", EuroVis 1997]

### 96.2 Empty-Space Skipping for Volume Ray Casting

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

### 96.3 Tensor Field Glyphs in Mesh Shaders

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

### 96.4 Time-Varying Isosurface Extraction

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

### 97. Decal and Surface Projection Geometry

*Audience: graphics application developers.*

Decals project surface detail (bullet holes, dirt, emblems) onto arbitrary geometry without modifying the underlying mesh. GPU decal systems fall into two families: *deferred decals* (project in screen space using the deferred G-buffer) and *mesh decals* (clip decal polygon against scene triangles on GPU to generate a coplanar mesh).

### 97.1 Deferred Screen-Space Decals

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

### 97.2 Mesh Decal: GPU Polygon Clipping

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

### 97.3 Stencil-Volume Projection

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

### 98. GPU Texture Synthesis and Procedural Shading Geometry

*Audience: graphics application developers.*

Procedural texturing generates surface detail without pre-authored texture assets — essential for infinite terrain, aged materials, and biological patterns. The geometric outputs (normal maps, displacement maps, colour masks) feed directly into the rendering pipeline as GPU images.

### 98.1 Reaction-Diffusion on GPU

Turing's reaction-diffusion system (1952) generates biological patterns — spots, stripes, labyrinth textures — by iterating two coupled PDEs on a 2D grid. The Gray-Scott model is GPU-friendly:

```glsl
// rdiff.comp — Gray-Scott reaction-diffusion step
layout(set=0,binding=0) uniform sampler2D uvIn;    // R=U chemical, G=V chemical
layout(set=0,binding=1, rg32f) uniform writeonly image2D uvOut;
layout(push_constant) uniform PC {
    float Du; float Dv; float F; float k; float dt;
} pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 p   = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv  = texelFetch(uvIn, p, 0).rg;
    float u   = uv.r, v = uv.g;
    // Discrete Laplacian (5-point stencil)
    float lapU = texelFetch(uvIn,p+ivec2(1,0),0).r + texelFetch(uvIn,p+ivec2(-1,0),0).r
               + texelFetch(uvIn,p+ivec2(0,1),0).r + texelFetch(uvIn,p+ivec2(0,-1),0).r
               - 4.0*u;
    float lapV = texelFetch(uvIn,p+ivec2(1,0),0).g + texelFetch(uvIn,p+ivec2(-1,0),0).g
               + texelFetch(uvIn,p+ivec2(0,1),0).g + texelFetch(uvIn,p+ivec2(0,-1),0).g
               - 4.0*v;
    float uvv  = u * v * v;
    float nu   = u + pc.dt*(pc.Du*lapU - uvv + pc.F*(1.0-u));
    float nv   = v + pc.dt*(pc.Dv*lapV + uvv - (pc.F+pc.k)*v);
    imageStore(uvOut, p, vec4(clamp(nu,0,1), clamp(nv,0,1), 0, 0));
}
```

Iterate 500–5000 steps offline or stream frames in real time for animated biological textures. [Source: Pearson "Complex Patterns in a Simple System", Science 1993; GPU RD: Rumpf & Strzodka "Numerical Methods for the Gray-Scott Model", 2000]

### 98.2 Wang Tiles for Non-Repeating Textures

Wang tiles (Wang 1961, Cohen et al. 2003) tile the plane aperiodically by ensuring adjacent tile edges match colour constraints — eliminating the visible repetition of tiled textures:

```glsl
// wang_sample.glsl — fragment shader texture lookup via Wang tile set
uniform sampler2DArray tileAtlas;  // N tiles × atlas
uniform sampler2D      tileIndex;  // pre-computed Wang tiling index map
in vec2 worldUV;
void main() {
    // Look up which tile occupies this texel's cell
    ivec2 cell  = ivec2(floor(worldUV));
    float tileID= texture(tileIndex, vec2(cell) / float(INDEX_SIZE)).r * NUM_TILES;
    // Within-tile UV
    vec2  localUV = fract(worldUV);
    fragColor = texture(tileAtlas, vec3(localUV, tileID));
}
```

GPU generation of valid Wang tilings: assign edge colours to a grid via a compute shader respecting the Wang constraint, then look up the corresponding tile from the set of 2^(2E) tiles for E edge colours. [Source: Cohen et al. "Wang Tiles for Image and Texture Generation", SIGGRAPH 2003]

### 98.3 Noise-Based Displacement and Normal Map Generation

Generate displacement maps from procedural noise for real-time terrain detail, without pre-authored assets. Combine fBm (§69.1) with domain warping:

```glsl
// displace_gen.comp — generate displacement + normal map from fBm + domain warp
layout(set=0,binding=0, r16f)  uniform writeonly image2D dispMap;
layout(set=0,binding=1, rgba8) uniform writeonly image2D normMap;
layout(push_constant) uniform PC { float scale; float warpStr; int octaves; vec2 offset; } pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 id  = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv  = (vec2(id)+0.5)/vec2(MAP_SIZE) + pc.offset;
    // Domain warp: offset UV by another fBm
    vec2  warp = vec2(fbm(uv + vec2(1.7,9.2), pc.octaves),
                      fbm(uv + vec2(8.3,2.8), pc.octaves)) * pc.warpStr;
    float h    = fbm(uv + warp, pc.octaves) * pc.scale;
    imageStore(dispMap, id, vec4(h));
    // Central-difference normal from displacement
    float hR = fbm((uv+warp)+vec2(1.0/MAP_SIZE,0), pc.octaves)*pc.scale;
    float hU = fbm((uv+warp)+vec2(0,1.0/MAP_SIZE), pc.octaves)*pc.scale;
    vec3  n  = normalize(vec3(h-hR, 1.0/MAP_SIZE, h-hU));
    imageStore(normMap, id, vec4(n*0.5+0.5, 1.0));
}
```

[Source: Quilez "Inigo Quilez — Articles: Domain Warping" https://iquilezles.org/articles/warp/]

---

### 99. Acoustic Ray Tracing and Sound Geometry

*Audience: systems developers, graphics application developers.*

Geometric acoustics treats sound as rays that reflect off surfaces according to the angle of incidence, attenuate with distance, and are absorbed by surface materials. GPU acceleration uses the same TLAS/BLAS infrastructure as visual ray tracing (§75) but with a much larger number of rays per source and a different termination criterion.

### 99.1 Acoustic Ray Emission

Each sound source emits N rays uniformly distributed over the unit sphere using Halton sampling (§105.1). Rays carry energy and terminate when energy falls below a threshold or when the maximum reflection count is reached:

```glsl
// acoustic_emit.raygen.glsl — emit acoustic rays from point source
#version 460
#extension GL_EXT_ray_tracing : require
struct AcousticRayPayload {
    float energy;        // current energy (starts at 1.0)
    float pathLength;    // accumulated path length
    uint  bounces;       // bounce count
    vec3  direction;
};
layout(location=0) rayPayloadEXT AcousticRayPayload payload;
layout(set=0,binding=0) uniform accelerationStructureEXT tlas;
layout(set=0,binding=1) writeonly buffer Impulse { vec2 impulse[]; };  // (energy, time)
layout(push_constant) uniform PC { vec3 sourcePos; float speedOfSound; uint nRays; } pc;
void main() {
    uint rayIdx  = gl_LaunchIDEXT.x;
    vec3 dir     = fibonacciSphere(rayIdx, pc.nRays);
    payload      = AcousticRayPayload(1.0, 0.0, 0u, dir);
    traceRayEXT(tlas, gl_RayFlagsNoneEXT, 0xFF,
                0, 1, 0, pc.sourcePos, 0.01, dir, MAX_RAY_DIST, 0);
}
```

### 99.2 Acoustic Closest-Hit: Reflection and Absorption

Each surface has a frequency-dependent absorption coefficient α. At each hit, attenuate the ray energy by (1-α) and reflect according to the surface normal:

```glsl
// acoustic_chit.glsl — acoustic closest hit shader
#extension GL_EXT_ray_tracing : require
layout(location=0) rayPayloadInEXT AcousticRayPayload payload;
hitAttributeEXT vec2 bary;
layout(set=0,binding=2) readonly buffer Materials { AcousticMaterial mats[]; };
void main() {
    AcousticMaterial m = mats[gl_InstanceCustomIndexEXT];
    payload.energy    *= (1.0 - m.absorption);
    payload.pathLength += gl_HitTEXT;
    payload.bounces++;
    // Record energy arrival at listener (if ray passes near listener position)
    float distToListener = length(listenerPos - gl_WorldRayOriginEXT + gl_HitTEXT*gl_WorldRayDirectionEXT);
    if (distToListener < LISTENER_RADIUS) {
        uint slot = atomicAdd(impulseCount, 1u);
        float arrivalTime = payload.pathLength / SPEED_OF_SOUND;
        impulse[slot] = vec2(payload.energy / (distToListener*distToListener), arrivalTime);
    }
    if (payload.energy < MIN_ENERGY || payload.bounces >= MAX_BOUNCES) return;
    // Specular reflection
    vec3 n = faceNormal();
    vec3 r = reflect(gl_WorldRayDirectionEXT, n);
    traceRayEXT(tlas, gl_RayFlagsNoneEXT, 0xFF,
                0, 1, 0, gl_WorldRayOriginEXT + gl_HitTEXT*gl_WorldRayDirectionEXT + n*0.001,
                0.001, r, MAX_RAY_DIST, 0);
}
```

[Source: Vorländer "Auralization: Fundamentals of Acoustics", 2008; GPU acoustic ray tracing: Schissler et al. "High-Order Diffraction and Diffuse Reflections for Interactive Sound Propagation in Large Environments", SIGGRAPH 2014]

### 99.3 Sound Occlusion Query via Ray Query

A cheaper occlusion-only query (no reflection) determines whether a sound source is audible from the listener position — equivalent to a shadow ray (§76.4) but in the audio domain:

```glsl
// sound_occlusion.comp
#extension GL_EXT_ray_query : require
layout(set=0,binding=0) uniform accelerationStructureEXT tlas;
layout(set=0,binding=1) readonly buffer Sources { SourceData sources[]; };
layout(set=0,binding=2) writeonly buffer Occlusion { float occl[]; };
layout(local_size_x=64) in;
void main() {
    uint s    = gl_GlobalInvocationID.x;
    vec3  dir = sources[s].pos - listenerPos;
    float len = length(dir);
    rayQueryEXT rq;
    rayQueryInitializeEXT(rq, tlas, gl_RayFlagsTerminateOnFirstHitEXT,
        ACOUSTIC_MASK, listenerPos, 0.01, normalize(dir), len);
    rayQueryProceedEXT(rq);
    occl[s] = (rayQueryGetIntersectionTypeEXT(rq,true)
               == gl_RayQueryCommittedIntersectionNoneEXT) ? 1.0 : 0.0;
}
```

[Source: Moeck et al. "Progressive Perceptual Audio Rendering of Complex Scenes", I3D 2007]

---

### 100. VR/XR Geometry: Reprojection and Foveation

*Audience: graphics application developers, systems developers.*

VR headsets have stringent per-eye frame rate requirements (90–120 Hz) and eye-tracking hardware that identifies the foveal region with sub-1° accuracy. GPU geometry algorithms support two XR-specific techniques: (1) reprojection (ATW/ASW) synthesises missing frames from depth and motion; (2) foveated rendering reduces triangle and shading work outside the fixation point.

### 100.1 Asynchronous TimeWarp (ATW)

ATW (Oculus 2014) re-projects the previous frame's colour+depth into the current head pose. For each output pixel, un-project using the previous frame's depth, transform to the new pose, and re-project:

```glsl
// atw.comp — Asynchronous TimeWarp reprojection
layout(set=0,binding=0) uniform sampler2D prevColor;
layout(set=0,binding=1) uniform sampler2D prevDepth;
layout(set=0,binding=2, rgba8) uniform writeonly image2D outColor;
layout(push_constant) uniform PC {
    mat4 prevViewProj;  // previous frame view-projection
    mat4 currViewProj;  // current head pose view-projection
    mat4 invPrevProj;   // inverse previous projection
} pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv   = (vec2(pix)+0.5)/vec2(imageSize(outColor));
    float d    = texture(prevDepth, uv).r;
    // Unproject from previous clip space to world space
    vec4  clip = vec4(uv*2.0-1.0, d*2.0-1.0, 1.0);
    vec4  world= pc.invPrevProj * clip;
    world /= world.w;
    // Reproject into current head pose
    vec4  curr = pc.currViewProj * world;
    vec2  nuv  = curr.xy/curr.w*0.5+0.5;
    vec4  col  = texture(prevColor, nuv);
    imageStore(outColor, pix, col);
}
```

[Source: Oculus "Asynchronous TimeWarp", Meta Developer Blog 2014; Valve "Low Latency VR Rendering" GDC 2015]

### 100.2 Application SpaceWarp (ASW)

ASW (Meta 2021) synthesises interleaved frames at half the application frame rate using optical flow. A motion vector field (from depth+pose) warps the previous frame to produce the synthesised frame:

```glsl
// asw_motionvec.comp — compute motion vectors from depth + pose change
layout(set=0,binding=0) uniform sampler2D depth;
layout(set=0,binding=1, rg16f) uniform writeonly image2D motionOut;
layout(push_constant) uniform PC { mat4 prevVP; mat4 currVP; mat4 invCurrProj; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv  = (vec2(pix)+0.5)/vec2(imageSize(motionOut));
    float d   = texture(depth, uv).r;
    vec4  clip= vec4(uv*2.0-1.0, d*2.0-1.0, 1.0);
    vec4  ws  = pc.invCurrProj * clip; ws /= ws.w;
    vec4  prev= pc.prevVP * ws;
    vec2  prevUV = prev.xy/prev.w*0.5+0.5;
    imageStore(motionOut, pix, vec4(uv - prevUV, 0, 0));
}
// Warp pass: for each output pixel, backward-sample prevColor along motion vector
```

[Source: Meta "Asynchronous SpaceWarp" Developer Blog 2021; Liang et al. "NSFF: Neural Scene Flow Fields for Space-Time View Synthesis", CVPR 2021]

### 100.3 Fixed-Foveated Rendering: Variable Rate Shading

Fixed foveation divides the render target into centre (1×1 shading rate), mid-ring (1×2), and periphery (2×4) regions and uses `VK_KHR_fragment_shading_rate` to reduce shading work. The shading rate image is written by a compute pass using the eye-tracking gaze position:

```glsl
// foveated_sri.comp — write shading rate image from gaze centre
layout(set=0,binding=0, r8ui) uniform writeonly uimage2D shadingRateImg;
layout(push_constant) uniform PC { vec2 gazeUV; float innerR; float outerR; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 tile = ivec2(gl_GlobalInvocationID.xy);  // one thread per shading rate tile (8×8 px)
    vec2  uv   = (vec2(tile)+0.5)/vec2(imageSize(shadingRateImg));
    float d    = length(uv - pc.gazeUV);
    uint  rate;
    if      (d < pc.innerR) rate = VK_FRAGMENT_SHADING_RATE_1_INVOCATION_PER_PIXEL;
    else if (d < pc.outerR) rate = VK_FRAGMENT_SHADING_RATE_1_INVOCATION_PER_2X1_PIXELS;
    else                    rate = VK_FRAGMENT_SHADING_RATE_1_INVOCATION_PER_2X2_PIXELS;
    imageStore(shadingRateImg, tile, uvec4(rate));
}
```

[Source: Guenter et al. "Foveated 3D Graphics", SIGGRAPH Asia 2012; Vulkan `VK_KHR_fragment_shading_rate` specification]

### 100.4 Foveated Mesh LOD

For geometry workload, reduce triangle density in the periphery. A task shader computes the screen-space distance from the foveal centre and selects a meshlet LOD tier accordingly:

```glsl
// foveated_task.task.glsl — select meshlet LOD based on foveal distance
layout(local_size_x=32) in;
void main() {
    uint m = gl_WorkGroupID.x;
    vec4 clip = viewProj * vec4(meshletCentre[m],1.0);
    vec2 ndc  = clip.xy/clip.w;
    float d   = length(ndc - gazeNDC);      // distance from gaze in NDC
    uint lod  = (d < 0.15) ? 0 :            // full detail
                (d < 0.40) ? 1 :            // medium
                              2;             // coarse
    uint meshletBase = lodMeshletBase[m*3+lod];
    uint meshletCount= lodMeshletCount[m*3+lod];
    EmitMeshTasksEXT(meshletCount, 1, 1);
}
```

[Source: Stengel et al. "Adaptive Image-Space Sampling for Gaze-Contingent Real-time Rendering", EG 2016; Meta Quest Pro eye-tracking developer guide 2023]

---

### 101. Silhouette Detection and Feature Lines

*Audience: graphics application developers.*

Silhouette edges are mesh edges where one adjacent face is front-facing and the other is back-facing relative to the view direction. GPU detection operates on the half-edge data structure, parallelising over all edges. Feature lines (ridges, valleys, suggestive contours) require curvature information from §38 spectral processing.

### 101.1 Silhouette Edge Detection on Half-Edge Mesh

For each edge, test the dot product of the face normals with the view vector. Emit geometry for silhouette edges:

```glsl
// silhouette_detect.comp — detect silhouette edges from half-edge mesh
layout(set=0,binding=0) readonly buffer Pos       { vec3 pos[];   };
layout(set=0,binding=1) readonly buffer FaceNorm  { vec3 fnorm[]; };
layout(set=0,binding=2) readonly buffer HalfEdge  { HalfEdgeDS he[]; };
layout(set=0,binding=3) buffer SilEdges { uvec2 silEdges[]; uint count; };
layout(push_constant) uniform PC { vec3 viewPos; } pc;
layout(local_size_x=64) in;
void main() {
    uint e   = gl_GlobalInvocationID.x;
    if (he[e].twin == INVALID) return;  // boundary edge: always emit
    uint f0  = he[e].face, f1 = he[he[e].twin].face;
    vec3 eP  = pos[he[e].vert];
    float d0 = dot(fnorm[f0], pc.viewPos - eP);
    float d1 = dot(fnorm[f1], pc.viewPos - eP);
    if (d0 * d1 < 0.0) {  // sign change → silhouette
        uint slot = atomicAdd(count, 1u);
        silEdges[slot] = uvec2(he[e].vert, he[he[e].twin].vert);
    }
}
```

[Source: Gooch et al. "A Non-Photorealistic Lighting Model For Automatic Technical Illustration", SIGGRAPH 1998; Card & Mitchell "Non-Photorealistic Rendering with Pixel and Vertex Shaders", ShaderX 2002]

### 101.2 Silhouette Fattening via Mesh Shader

Expand silhouette edges into thin quads (fin geometry) in the mesh shader to produce anti-aliased outlines without geometry shader overhead:

```glsl
// silhouette_mesh.mesh.glsl — emit fin quad for each silhouette edge
layout(local_size_x=32) in;
layout(triangles, max_vertices=4, max_primitives=2) out;
void main() {
    uint e    = gl_WorkGroupID.x;
    vec3 v0   = worldPos[silEdges[e].x];
    vec3 v1   = worldPos[silEdges[e].y];
    vec3 view = normalize(cameraPos - (v0+v1)*0.5);
    vec3 edge = normalize(v1-v0);
    vec3 outDir = normalize(cross(edge, view));  // extrude direction
    // Emit 4 vertices: inner edge v0,v1; outer edge v0+w*out, v1+w*out
    SetMeshOutputsEXT(4, 2);
    gl_MeshVerticesEXT[0].gl_Position = viewProj*vec4(v0,1);
    gl_MeshVerticesEXT[1].gl_Position = viewProj*vec4(v1,1);
    gl_MeshVerticesEXT[2].gl_Position = viewProj*vec4(v0+outDir*OUTLINE_WIDTH,1);
    gl_MeshVerticesEXT[3].gl_Position = viewProj*vec4(v1+outDir*OUTLINE_WIDTH,1);
    gl_PrimitiveTriangleIndicesEXT[0] = uvec3(0,1,2);
    gl_PrimitiveTriangleIndicesEXT[1] = uvec3(1,3,2);
}
```

[Source: Raskar & Cohen "Image Precision Silhouette Edges on a GPU", I3D 1999; Isenberg et al. "A Developer's Guide to Silhouette Algorithms for Polygonal Models", IEEE CGA 2003]

### 101.3 Ridge and Valley Lines from Principal Curvature

Ridges are loci where the maximum principal curvature κ₁ is locally maximal along its curvature direction e₁. Detect them per-edge by sign change of the directional derivative of κ₁:

```glsl
// ridge_valley.comp — detect ridge/valley lines from principal curvature
layout(set=0,binding=0) readonly buffer Kappa  { vec2 kappa[]; };  // k1,k2 per vertex
layout(set=0,binding=1) readonly buffer E1dir  { vec3 e1[];    };  // max curvature direction
layout(set=0,binding=2) readonly buffer HalfEdge { HalfEdgeDS he[]; };
layout(set=0,binding=3) buffer RidgeEdges { uvec2 ridges[]; uint count; };
layout(local_size_x=64) in;
void main() {
    uint e  = gl_GlobalInvocationID.x;
    uint vi = he[e].vert, vj = he[he[e].twin].vert;
    // Directional derivative of k1 along edge direction
    vec3  edgeDir = normalize(pos[vj]-pos[vi]);
    float dk1i    = dot(e1[vi], edgeDir) * kappa[vi].x;
    float dk1j    = dot(e1[vj], edgeDir) * kappa[vj].x;
    if (dk1i * dk1j < 0.0 && max(kappa[vi].x, kappa[vj].x) > RIDGE_THRESHOLD) {
        uint slot = atomicAdd(count, 1u);
        ridges[slot] = uvec2(vi, vj);
    }
}
```

[Source: Ohtake et al. "Ridge-Valley Lines on Meshes via Implicit Surface Fitting", SIGGRAPH 2004; DeCarlo et al. "Suggestive Contours for Conveying Shape", SIGGRAPH 2003]

### 101.4 Suggestive Contours

Suggestive contours are where the radial curvature κᵣ (curvature in the view direction projected onto the surface) crosses zero with negative derivative — predicting silhouettes that would appear under small view perturbations:

```glsl
// suggestive_contour.comp — find suggestive contour edges
layout(set=0,binding=0) readonly buffer Kappa  { vec2 kappa[]; };   // k1, k2
layout(set=0,binding=1) readonly buffer PrinDir { vec3 e1[]; vec3 e2[]; };
layout(set=0,binding=2) readonly buffer Pos    { vec3 pos[]; };
layout(set=0,binding=3) readonly buffer HalfEdge { HalfEdgeDS he[]; };
layout(set=0,binding=4) buffer SCEdges { uvec2 sc[]; uint count; };
layout(push_constant) uniform PC { vec3 viewPos; } pc;
layout(local_size_x=64) in;

float radialCurvature(uint v) {
    vec3  w    = normalize(pc.viewPos - pos[v]);
    vec3  wt   = normalize(w - dot(w, nor[v])*nor[v]);  // project to tangent
    float cosA = dot(wt, e1[v]);
    float sinA = dot(wt, e2[v]);
    return kappa[v].x*cosA*cosA + kappa[v].y*sinA*sinA;
}
void main() {
    uint e  = gl_GlobalInvocationID.x;
    uint vi = he[e].vert, vj = he[he[e].twin].vert;
    float ki = radialCurvature(vi), kj = radialCurvature(vj);
    if (ki * kj < 0.0)  {  // zero crossing of radial curvature
        uint slot = atomicAdd(count, 1u);
        sc[slot] = uvec2(vi, vj);
    }
}
```

[Source: DeCarlo et al. 2003; Judd et al. "Apparent Ridges for Line Drawing", SIGGRAPH 2007]

---

### 102. Micro-Polygon Displacement and GPU Tessellation

*Audience: graphics application developers.*

Displacement mapping (Cook 1984) adds fine surface detail by offsetting vertices along the normal by a scalar value sampled from a height texture. GPU tessellation via the hardware tessellator (Vulkan tessellation shaders) or mesh shaders generates the dense vertex grid at runtime; §1 subdivision surfaces and §33 virtual geometry represent related approaches. True micro-polygon displacement requires crack-free patch stitching and adaptive tessellation factor computation.

### 102.1 Adaptive Tessellation Factor Computation

Tessellation factors are set per-edge based on projected edge length or curvature, ensuring one tessellation sample per pixel:

```glsl
// tess_control.tesc — adaptive edge tessellation factors from projected length
#version 460
layout(vertices=3) out;
layout(push_constant) uniform PC { mat4 mvp; vec2 viewportRes; float targetEdgePx; } pc;
void main() {
    gl_out[gl_InvocationID].gl_Position = gl_in[gl_InvocationID].gl_Position;
    if (gl_InvocationID == 0) {
        // Project each edge midpoint, compute screen-space length
        for (int e=0;e<3;e++) {
            vec4 p0 = pc.mvp * gl_in[e].gl_Position;
            vec4 p1 = pc.mvp * gl_in[(e+1)%3].gl_Position;
            vec2 s0 = (p0.xy/p0.w*0.5+0.5)*pc.viewportRes;
            vec2 s1 = (p1.xy/p1.w*0.5+0.5)*pc.viewportRes;
            gl_TessLevelOuter[e] = clamp(length(s1-s0)/pc.targetEdgePx, 1.0, 64.0);
        }
        gl_TessLevelInner[0] = (gl_TessLevelOuter[0]+gl_TessLevelOuter[1]+gl_TessLevelOuter[2])/3.0;
    }
}
```

[Source: Schwarz & Stamminger "Bitmask Soft Shadows", EG 2007; Nießner et al. "Feature-Adaptive GPU Rendering of Catmull-Clark Subdivision Surfaces", SIGGRAPH 2012]

### 102.2 Displacement Evaluation in Tessellation Evaluation Shader

```glsl
// tess_eval.tese — displacement along interpolated normal
#version 460
layout(triangles, fractional_odd_spacing, ccw) in;
layout(push_constant) uniform PC { mat4 mvp; float dispScale; } pc;
uniform sampler2D heightTex;
in vec3 tcNormal[];
in vec2 tcUV[];
void main() {
    // Barycentric interpolation of position, normal, UV
    vec3 pos = gl_TessCoord.x*gl_in[0].gl_Position.xyz
             + gl_TessCoord.y*gl_in[1].gl_Position.xyz
             + gl_TessCoord.z*gl_in[2].gl_Position.xyz;
    vec3 nor = normalize(gl_TessCoord.x*tcNormal[0]
                        +gl_TessCoord.y*tcNormal[1]
                        +gl_TessCoord.z*tcNormal[2]);
    vec2 uv  = gl_TessCoord.x*tcUV[0]+gl_TessCoord.y*tcUV[1]+gl_TessCoord.z*tcUV[2];
    float h  = texture(heightTex, uv).r;
    pos     += nor * h * pc.dispScale;
    gl_Position = pc.mvp * vec4(pos, 1.0);
}
```

[Source: Cook "Shade Trees", SIGGRAPH 1984; Tatarchuk "Practical Parallax Occlusion Mapping for Highly Detailed Surface Rendering", ShaderX 5, 2006]

### 102.3 Crack-Free Patch Stitching

Adjacent patches with different tessellation factors produce T-junctions and cracks. The hardware tessellator handles this within a patch, but between patches the outer tessellation levels must match. Compute the outer tessellation factor for each shared edge identically from both patches:

```glsl
// tess_control_crackfree.tesc — crack-free outer tessellation via shared edge hash
layout(push_constant) uniform PC { mat4 mvp; vec2 res; float targetPx; } pc;
float edgeTessLevel(vec3 p0, vec3 p1) {
    vec4 c0=pc.mvp*vec4(p0,1), c1=pc.mvp*vec4(p1,1);
    vec2 s0=(c0.xy/c0.w*0.5+0.5)*pc.res, s1=(c1.xy/c1.w*0.5+0.5)*pc.res;
    return clamp(length(s1-s0)/pc.targetPx, 1.0, 64.0);
}
void main() {
    // Both patches sharing edge (v0,v1) must call edgeTessLevel(v0,v1)
    // in the same vertex order → sorted by vertex index
    uvec2 e = uvec2(min(vIdx[0],vIdx[1]), max(vIdx[0],vIdx[1]));
    gl_TessLevelOuter[0] = edgeTessLevel(pos[e.x],pos[e.y]);
    // ... similarly for other edges
}
```

[Source: Nießner et al. 2012; Walton "Tessellation on Any Budget", GDC 2011]

### 102.4 Mesh Shader Displacement for Nanite-Compatible Micro-Polygons

When using §33 virtual geometry, displacement can be applied in the mesh shader itself, offsetting cluster vertices along normals sampled from a virtual texture:

```glsl
// disp_mesh.mesh.glsl — apply displacement in mesh shader for cluster rendering
layout(local_size_x=128) in;
layout(triangles, max_vertices=128, max_primitives=126) out;
void main() {
    uint v = gl_LocalInvocationID.x;
    SetMeshOutputsEXT(N_VERTS, N_TRIS);
    if (v < N_VERTS) {
        vec3 pos = clusterVerts[v];
        vec3 nor = clusterNormals[v];
        vec2 uv  = clusterUVs[v];
        float h  = texture(heightMap, uv).r;
        pos     += nor * h * dispScale;
        gl_MeshVerticesEXT[v].gl_Position = mvp * vec4(pos,1.0);
        outUV[v] = uv; outNor[v] = nor;
    }
    if (v < N_TRIS) gl_PrimitiveTriangleIndicesEXT[v] = clusterTris[v];
}
```

[Source: Karis et al. 2021 §90; Kubisch "Turing Mesh Shaders", NVIDIA 2018]

---

### 103. GPU SDF Font and Curve Rendering

*Audience: graphics application developers.*

Signed distance field (SDF) fonts (Green 2007) pre-render each glyph to a low-resolution SDF texture; at runtime, the fragment shader reconstructs sharp edges regardless of scale by thresholding the interpolated SDF value. Multi-channel SDF (MSDF, Chlumský 2016) encodes the distance in three colour channels to preserve sharp corners. Loop-Blinn (2005) renders cubic Bézier curves exactly on the GPU without pre-baking.

### 103.1 Offline SDF Glyph Baking

Bake glyph outlines (from FreeType) into a low-resolution SDF texture by computing the signed distance from each texel to the nearest contour edge:

```glsl
// sdf_bake.comp — bake SDF for a single glyph from filled scanline bitmap
layout(set=0,binding=0) readonly buffer GlyphBitmap { uint bitmap[]; };  // 1-bit per pixel
layout(set=0,binding=1, r8_snorm) writeonly image2D sdfOut;
layout(push_constant) uniform PC { ivec2 bitmapDim; ivec2 sdfDim; int searchRadius; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 sdfPix = ivec2(gl_GlobalInvocationID.xy);
    if (any(greaterThanEqual(sdfPix,pc.sdfDim))) return;
    // Map SDF pixel to bitmap space (upsample factor)
    ivec2 bPix  = sdfPix * pc.bitmapDim / pc.sdfDim;
    bool inside = bitfieldExtract(bitmap[bPix.y*(pc.bitmapDim.x/32)+bPix.x/32], bPix.x%32, 1) != 0;
    float minD  = float(pc.searchRadius);
    for (int dy=-pc.searchRadius;dy<=pc.searchRadius;dy++)
    for (int dx=-pc.searchRadius;dx<=pc.searchRadius;dx++) {
        ivec2 nb  = bPix+ivec2(dx,dy);
        if (any(lessThan(nb,ivec2(0)))||any(greaterThanEqual(nb,pc.bitmapDim))) continue;
        bool nbIn = bitfieldExtract(bitmap[nb.y*(pc.bitmapDim.x/32)+nb.x/32], nb.x%32, 1) != 0;
        if (nbIn != inside) minD = min(minD, length(vec2(dx,dy)));
    }
    float sdf = (inside ? 1.0 : -1.0) * minD / float(pc.searchRadius) * 0.5;
    imageStore(sdfOut, sdfPix, vec4(sdf));
}
```

[Source: Green "Improved Alpha-Tested Magnification for Vector Textures and Special Effects", SIGGRAPH 2007]

### 103.2 Runtime SDF Font Rendering

In the fragment shader, threshold the SDF and apply anti-aliasing using the screen-space derivative of the SDF:

```glsl
// sdf_font.frag — render SDF glyph with smooth anti-aliased edges
uniform sampler2D sdfAtlas;
uniform vec4 textColor;
in vec2 texUV;
void main() {
    float d     = texture(sdfAtlas, texUV).r;
    // Width of the AA region in SDF units: ~0.5 / (screen pixels per SDF texel)
    float width = fwidth(d) * 0.7;
    float alpha = smoothstep(0.5 - width, 0.5 + width, d);
    fragColor   = vec4(textColor.rgb, textColor.a * alpha);
}
```

[Source: Green 2007; Behdad "State of Text Rendering 2", 2024]

### 103.3 Multi-Channel SDF (MSDF) for Sharp Corners

MSDF (Chlumský 2016) stores the minimum distance to the nearest contour edge in three separate colour channels (one per pseudo-distance direction), then takes the median at render time to reconstruct sharp corners that single-channel SDF blurs:

```glsl
// msdf_font.frag — render MSDF glyph: median of three channels for sharp corners
uniform sampler2D msdfAtlas;
uniform vec4 textColor;
in vec2 texUV;
float median(float r, float g, float b) { return max(min(r,g),min(max(r,g),b)); }
void main() {
    vec3  msd   = texture(msdfAtlas, texUV).rgb;
    float d     = median(msd.r, msd.g, msd.b);
    float width = fwidth(d) * 0.7;
    float alpha = smoothstep(0.5-width, 0.5+width, d);
    fragColor   = vec4(textColor.rgb, textColor.a * alpha);
}
```

[Source: Chlumský "Shape Decomposition for Multi-Channel Distance Fields", Master's thesis, Czech Technical University 2015]

### 103.4 Loop-Blinn Cubic Bézier Rendering

Loop & Blinn (2005) render cubic Bézier curves exactly on the GPU by encoding each bezier patch as a quad, computing a cubic discriminant texture, and discarding fragments outside the curve in the fragment shader:

```glsl
// loop_blinn.frag — inside/outside test for cubic Bézier via Loop-Blinn klmn coords
in vec3 klmn;  // per-vertex cubic texture coordinates from vertex shader
void main() {
    // Fragment is inside the loop/serpentine/cusp if klmn.x³ - klmn.y*klmn.z < 0
    float f = klmn.x*klmn.x*klmn.x - klmn.y*klmn.z;
    if (f > 0.0) discard;  // outside curve fill region
    fragColor = vec4(vec3(0), 1.0);  // inside: fill with text color
}
```

[Source: Loop & Blinn "Resolution Independent Curve Rendering Using Programmable Graphics Hardware", SIGGRAPH 2005; Nehab & Hoppe "Random-Access Rendering of General Vector Graphics", SIGGRAPH Asia 2008]

---

### 104. Molecular Surface Computation

*Audience: systems developers.*

Molecular surfaces (solvent-accessible surface, SAS; solvent-excluded surface, SES; Connolly surface) define the geometric boundary between a protein and its solvent. GPU algorithms compute these surfaces for real-time docking visualization, drug discovery pipelines, and volumetric electrostatics. Inputs are atom positions and van der Waals radii; outputs are triangle meshes or SDF grids.

### 104.1 Solvent-Accessible Surface via SDF Grid

The SAS is the union of spheres of radius rᵢ + rₛ centred on each atom (rᵢ = van der Waals, rₛ = solvent probe). Compute as a SDF grid and extract with marching cubes (§65):

```glsl
// molsurf_sas.comp — compute SAS SDF: min distance to expanded atom spheres
layout(set=0,binding=0) readonly buffer Atoms { vec4 atoms[]; };  // xyz + vdW radius
layout(set=0,binding=1) buffer SDF { float sdf[]; };
layout(push_constant) uniform PC { vec3 gridMin; float dx; ivec3 dim; float probeR; uint N; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 c  = ivec3(gl_GlobalInvocationID);
    if (any(greaterThanEqual(c,pc.dim))) return;
    vec3  xg = pc.gridMin + (vec3(c)+0.5)*pc.dx;
    float d  = 1e30;
    for (uint a=0; a<pc.N; a++) {
        float r = atoms[a].w + pc.probeR;  // expanded radius
        d = min(d, length(xg - atoms[a].xyz) - r);
    }
    sdf[c.z*pc.dim.y*pc.dim.x + c.y*pc.dim.x + c.x] = d;
}
// Then run §65 marching cubes on sdf[] at isovalue 0.0
```

[Source: Connolly "Analytical Molecular Surface Calculation", Journal of Applied Crystallography 1983; Krone et al. "GPU-Based Interactive Visualization Techniques for Molecular Dynamics Simulations", Computer Graphics Forum 2012]

### 104.2 Solvent-Excluded Surface via Rolling Probe

The SES (Connolly surface) accounts for where the probe sphere cannot fit between atoms — the re-entrant surface. GPU computation identifies contact circles and toroidal re-entrant patches:

```glsl
// molsurf_ses.comp — classify SES patch type per grid voxel
layout(set=0,binding=0) readonly buffer Atoms { vec4 atoms[]; };
layout(set=0,binding=1) buffer SDF { float sdf[]; };
layout(push_constant) uniform PC { vec3 gridMin; float dx; ivec3 dim; float probeR; uint N; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 c  = ivec3(gl_GlobalInvocationID);
    vec3  xg = pc.gridMin + (vec3(c)+0.5)*pc.dx;
    // Probe-center-accessible surface: min distance from probe centre to SAS
    float dSAS = 1e30;
    for (uint a=0; a<pc.N; a++) dSAS = min(dSAS, length(xg-atoms[a].xyz) - (atoms[a].w+pc.probeR));
    // SES: subtract probe radius from probe-accessible SDF
    sdf[flatIdx(c,pc.dim)] = dSAS - pc.probeR;
    // Result: negative inside SES, positive outside; isovalue 0 gives the surface
}
```

[Source: Connolly 1983; Parulek & Viola "Implicit Representation of Molecular Surfaces", IEEE TVCG 2012]

### 104.3 Ambient Occlusion for Molecular Visualization

Molecular AO integrates over the hemisphere above each surface point whether other atoms occlude it — a proxy for solvent accessibility and electrostatic exposure:

```glsl
// mol_ao.comp — ambient occlusion for molecular surface via ray queries
layout(set=0,binding=0) readonly buffer SurfPts { vec3 spts[];  };
layout(set=0,binding=1) readonly buffer SurfNor  { vec3 snor[];  };
layout(set=0,binding=2) uniform accelerationStructureEXT tlas;  // BVH of atom spheres §31
layout(set=0,binding=3) writeonly buffer AO { float ao[]; };
layout(push_constant) uniform PC { int nRays; float maxDist; } pc;
layout(local_size_x=64) in;
void main() {
    uint v = gl_GlobalInvocationID.x;
    vec3 p = spts[v] + snor[v]*1e-3;
    float occ = 0.0;
    for (int r=0; r<pc.nRays; r++) {
        vec3 d = cosineWeightedDir(snor[v], halton2D(r));
        rayQueryEXT rq;
        rayQueryInitializeEXT(rq, tlas, gl_RayFlagsTerminateOnFirstHitEXT, 0xFF, p, 0.0, d, pc.maxDist);
        rayQueryProceedEXT(rq);
        if (rayQueryGetIntersectionTypeEXT(rq,true)!=gl_RayQueryCommittedIntersectionNoneEXT)
            occ += 1.0;
    }
    ao[v] = 1.0 - occ / float(pc.nRays);
}
```

[Source: Tarini et al. "Ambient Occlusion and Edge Cueing to Enhance Real Time Molecular Visualization", IEEE TVCG 2006; §62 RTAO pattern]

### 104.4 Electrostatic Potential on Surface (Poisson-Boltzmann)

Map the Poisson-Boltzmann electrostatic potential onto the molecular surface by solving the linearised PB equation on the SDF grid with a conjugate gradient solver (§37.2 pattern):

```glsl
// poisson_boltzmann.comp — one Jacobi iteration of linearised PB equation
// ∇²φ = κ²φ - ρ/ε₀ (inside cavity: κ=0; outside: κ=Debye screening)
layout(set=0,binding=0) readonly buffer SDF     { float sdf[];   };  // §104.2 surface
layout(set=0,binding=1) readonly buffer Charges { vec4  charges[]; }; // xyz + charge
layout(set=0,binding=2) buffer Phi { float phi[]; };
layout(push_constant) uniform PC { ivec3 dim; float dx; float kappa2_out; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 c   = ivec3(gl_GlobalInvocationID);
    if (any(greaterThanEqual(c,pc.dim))) return;
    uint idx  = flatIdx(c,pc.dim);
    float inside = step(sdf[idx], 0.0);  // 1=inside, 0=outside
    float kappa2  = pc.kappa2_out * (1.0-inside);  // screening only outside
    // 6-point Laplacian
    float lap = (phi[flatIdx(c+ivec3(1,0,0),pc.dim)] + phi[flatIdx(c-ivec3(1,0,0),pc.dim)]
               + phi[flatIdx(c+ivec3(0,1,0),pc.dim)] + phi[flatIdx(c-ivec3(0,1,0),pc.dim)]
               + phi[flatIdx(c+ivec3(0,0,1),pc.dim)] + phi[flatIdx(c-ivec3(0,0,1),pc.dim)]) / (pc.dx*pc.dx);
    float rho = atomChargeDensity(c, charges);
    phi[idx]  = (lap - rho) / (6.0/(pc.dx*pc.dx) + kappa2);
}
```

[Source: Baker et al. "Electrostatics of Nanosystems: Application to Microtubules and the Ribosome", PNAS 2001; APBS software; Sharp & Honig "Electrostatic Interactions in Macromolecules", Annual Review 1990]

---


---

## XIII. GPU Algorithm Primitives for Geometry

These are the low-level compute algorithms that serve as building blocks for the higher-level geometry techniques throughout this chapter. GPU Poisson disk and blue noise sampling underpin mesh point distribution, lightmap texel placement, and stochastic rendering; parallel convex hull computation (the Quickhull algorithm parallelized over GPU threads) appears inside GJK collision detection and computational geometry tools; and GPU radix sort is the enabling primitive for LBVH BVH construction, depth-sorting for OIT, and particle simulation step reordering. These three algorithms rarely appear in isolation — they are invoked implicitly inside the vast majority of the algorithms in the other twelve categories.

### 105. GPU Sampling: Poisson Disk and Blue Noise

*Audience: graphics application developers, systems developers.*

Low-discrepancy sampling, Poisson disk distributions, and blue noise are foundational primitives used throughout the chapter: AO hemisphere samples (§28), particle seeding (§72), LOD representative points (§10), and Monte Carlo integration in §77 IBL preprocessing. This section covers GPU-native generation of these distributions.

### 105.1 Low-Discrepancy Sequences: Halton and Sobol

Halton sequences in base b are computed per-sample with no state — fully parallel:

```glsl
// halton.glsl — single-sample Halton in base b
float halton(uint index, uint base) {
    float f = 1.0, r = 0.0;
    uint i = index;
    while (i > 0u) { f /= float(base); r += f * float(i % base); i /= base; }
    return r;
}
vec2 halton2D(uint i) { return vec2(halton(i,2), halton(i,3)); }
```

Sobol sequences require a precomputed direction-number table (max 21201 dimensions, 32 bits each) stored in a GPU buffer. [Source: Joe & Kuo "Constructing Sobol Sequences with Better Two-Dimensional Projections", SIAM J. Sci. Comput. 2010; GPU Sobol: Pharr et al. "Physically Based Rendering", 4th ed.]

### 105.2 Area-Weighted Triangle Sampling on Mesh Surfaces

Sample a uniform point on a mesh surface: select a triangle with probability ∝ area (CDF inversion on prefix-sum area array), then sample within the triangle using Osada's square-root method:

```glsl
// surface_sample.comp — sample N uniform points on mesh surface
layout(set=0,binding=0) readonly buffer AreaCDF  { float cdf[];   };  // prefix-sum areas
layout(set=0,binding=1) readonly buffer Tris     { vec3  tv[];    };  // 3 verts per tri
layout(set=0,binding=2) writeonly buffer Samples { vec3  samples[]; };
layout(local_size_x=64) in;
void main() {
    uint  i  = gl_GlobalInvocationID.x;
    vec2  xi = halton2D(i);
    // Binary search in CDF for triangle index
    uint lo=0u, hi=N_TRIS;
    while (lo<hi) { uint mid=(lo+hi)/2u; if(cdf[mid]<xi.x*cdf[N_TRIS]) lo=mid+1u; else hi=mid; }
    uint  t  = lo;
    float r1 = sqrt(xi.y), r2 = halton(i, 5);
    samples[i] = (1.0-r1)*tv[t*3] + r1*(1.0-r2)*tv[t*3+1] + r1*r2*tv[t*3+2];
}
```

[Source: Osada et al. "Shape Distributions", SIGGRAPH 2002]

### 105.3 GPU Poisson Disk Dart Throwing

Generate a Poisson disk distribution with minimum distance r via GPU dart throwing with spatial hash rejection:

```glsl
// poisson_dart.comp — one round of dart throwing
layout(set=0,binding=0) buffer Accepted { vec3 pts[]; };
layout(set=0,binding=1) buffer AccCount { uint count; };
layout(set=0,binding=2) buffer Grid     { uint gridPts[]; uint gridCnt[]; };  // §27.2 hash
layout(push_constant) uniform PC { float minDist; uint round; uint nDarts; } pc;
layout(local_size_x=64) in;
void main() {
    uint  i    = gl_GlobalInvocationID.x;
    vec3  dart = randomPointInDomain(i + pc.round * pc.nDarts);
    ivec3 ci   = ivec3(floor(dart / pc.minDist));
    bool  ok   = true;
    for (int dz=-2;dz<=2&&ok;dz++) for(int dy=-2;dy<=2&&ok;dy++) for(int dx=-2;dx<=2&&ok;dx++) {
        uint cell = hashCell3(ci+ivec3(dx,dy,dz));
        for (uint k=0; k<gridCnt[cell]&&ok; k++)
            if (length(dart - pts[gridPts[cell*MAX_PER_CELL+k]]) < pc.minDist) ok=false;
    }
    if (ok) {
        uint slot = atomicAdd(count, 1u);
        pts[slot] = dart;
        insertGrid(dart, slot);
    }
}
```

[Source: Bridson "Fast Poisson Disk Sampling in Arbitrary Dimensions", SIGGRAPH 2007 Sketches]

### 105.4 Blue Noise via Lloyd Relaxation on Mesh

Centroidal Voronoi tessellation (CVT) gives blue-noise-quality distributions with perfectly uniform coverage. Iterate: (1) partition surface into Voronoi cells (nearest-sample per surface point); (2) move each sample to the centroid of its Voronoi cell:

```glsl
// lloyd_relax.comp — one Lloyd iteration: find centroid of each Voronoi cell
layout(set=0,binding=0) buffer Samples { vec3 pts[]; };
layout(set=0,binding=1) readonly buffer Surface{ vec3 surf[]; };  // dense surface sample set
layout(set=0,binding=2) buffer CentAcc { vec3 acc[]; uint cnt[]; };
layout(local_size_x=64) in;
void main() {
    uint p = gl_GlobalInvocationID.x;   // surface point index
    // Find nearest Poisson disk sample (GPU k-NN §87.1 or brute force for small N)
    uint nearest = findNearest(surf[p], pts, N_SAMPLES);
    atomicAdd_vec3(acc[nearest], surf[p]);
    atomicAdd(cnt[nearest], 1u);
}
// Post-pass: pts[i] = acc[i] / cnt[i] (project back to surface)
```

[Source: Du et al. "Centroidal Voronoi Tessellations: Applications and Algorithms", SIAM Review 1999]

---

### 106. GPU Convex Hull Computation

*Audience: systems developers, graphics application developers.*

Convex hulls are needed to bootstrap rigid body shapes (§50), compute GJK support functions (§24), and fit BVH leaf bounds (§31). GPU QuickHull operates on point clouds of up to tens of millions of points; GPU incremental insertion processes batches of points in parallel.

### 106.1 GPU QuickHull: Initial Simplex

QuickHull begins by finding the 6 extreme points (±X, ±Y, ±Z), forming an initial tetrahedron. Find extremes in parallel via a reduction:

```glsl
// qhull_extremes.comp — find 6 axis-aligned extremes via parallel reduction
layout(set=0,binding=0) readonly buffer Pts { vec3 pts[]; };
layout(set=0,binding=1) buffer Extremes { vec3 extr[6]; };  // ±x, ±y, ±z extremes
layout(local_size_x=256) in;
shared vec3 sMin[256], sMax[256];
void main() {
    uint i  = gl_GlobalInvocationID.x;
    uint li = gl_LocalInvocationID.x;
    sMin[li] = (i < N_PTS) ? pts[i] : vec3(1e30);
    sMax[li] = (i < N_PTS) ? pts[i] : vec3(-1e30);
    barrier();
    for (uint s=128u; s>0u; s>>=1) {
        if (li<s) { sMin[li]=min(sMin[li],sMin[li+s]); sMax[li]=max(sMax[li],sMax[li+s]); }
        barrier();
    }
    if (li==0) {
        atomicMin_vec3(extr[0], sMin[0]);  // -x extreme
        atomicMax_vec3(extr[1], sMax[0]);  // +x, etc.
    }
}
```

### 106.2 GPU QuickHull: Furthest-Point Assignment

For each face of the current hull, find the point furthest above it (positive distance). Points inside all faces are discarded:

```glsl
// qhull_assign.comp — assign outside points to faces, find furthest per face
layout(set=0,binding=0) readonly buffer Pts   { vec3 pts[];   };
layout(set=0,binding=1) readonly buffer Faces { vec4 planes[]; };  // normal + d
layout(set=0,binding=2) buffer Assignment { int assign[];  };  // face index or -1 (inside)
layout(set=0,binding=3) buffer FurthestD  { float maxDist[]; };  // per face
layout(set=0,binding=4) buffer FurthestI  { uint  maxIdx[];  };  // per face
layout(local_size_x=64) in;
void main() {
    uint p = gl_GlobalInvocationID.x;
    if (p >= N_PTS) return;
    float bestD = 0.0; int bestF = -1;
    for (uint f=0; f<N_FACES; f++) {
        float d = dot(planes[f].xyz, pts[p]) + planes[f].w;
        if (d > bestD) { bestD=d; bestF=int(f); }
    }
    assign[p] = bestF;
    if (bestF >= 0) atomicMax_float_idx(maxDist[bestF], maxIdx[bestF], bestD, p);
}
```

[Source: Barber et al. "The Quickhull Algorithm for Convex Hulls", ACM TOMS 1996; GPU QuickHull: Cao et al. "Parallel Bounding Volume Hierarchies", SIGGRAPH 2010]

### 106.3 Horizon Edge and Face Expansion

Once the furthest point p for a face is found, compute the horizon ridge (edges visible from p), create new faces from p to each horizon edge, and remove the visible faces:

```glsl
// qhull_expand.comp — find horizon edges visible from furthest point
layout(set=0,binding=0) readonly buffer Faces { HullFace faces[]; };  // v0,v1,v2 + neighbours
layout(set=0,binding=1) readonly buffer Pts   { vec3 pts[]; };
layout(set=0,binding=2) buffer HorizonEdges { uvec2 hedges[]; uint hedgeCount; };
layout(push_constant) uniform PC { uint apexIdx; uint seedFace; } pc;
layout(local_size_x=1) in;  // serial BFS; parallelism across separate hull operations
void main() {
    // BFS from seedFace: mark faces visible from pts[apexIdx], collect horizon edges
    uint queue[MAX_FACES]; uint head=0, tail=0;
    queue[tail++] = pc.seedFace; visibleMask[pc.seedFace]=1u;
    while (head<tail) {
        uint f = queue[head++];
        for (int e=0;e<3;e++) {
            uint nb = faces[f].neighbours[e];
            float d  = dot(faces[nb].plane.xyz, pts[pc.apexIdx]) + faces[nb].plane.w;
            if (d > 0.0) { if (!visibleMask[nb]) { visibleMask[nb]=1u; queue[tail++]=nb; }}
            else {  // horizon edge: boundary between visible and non-visible
                uint slot = atomicAdd(hedgeCount, 1u);
                hedges[slot] = uvec2(faces[f].verts[e], faces[f].verts[(e+1)%3]);
            }
        }
    }
}
```

[Source: Barber et al. 1996; O'Brien "Fast and Simple Physics Using Sequential Impulses", GDC 2006]

### 106.4 Incremental Parallel Hull for Point Batches

For massive point clouds, process points in batches: run §106.1–82.3 serially on a random initial subset (1000 points), then for the remaining points, classify inside/outside in parallel (§106.2) and only add points outside the current hull:

```glsl
// qhull_batch_classify.comp — classify new batch against current hull
layout(set=0,binding=0) readonly buffer NewPts  { vec3 newPts[];  };
layout(set=0,binding=1) readonly buffer HullPlanes { vec4 planes[]; };
layout(set=0,binding=2) buffer Outside { vec3 outside[]; uint count; };
layout(local_size_x=64) in;
void main() {
    uint p = gl_GlobalInvocationID.x;
    for (uint f=0; f<N_HULL_FACES; f++)
        if (dot(planes[f].xyz, newPts[p]) + planes[f].w > 1e-5) {
            uint slot = atomicAdd(count, 1u);
            outside[slot] = newPts[p];
            return;
        }
    // Inside all planes → discard (already enclosed by hull)
}
```

[Source: Preparata & Shamos "Computational Geometry: An Introduction", 1985; GPU hull: Lauterbach et al. 2009 §70.1]

---

### 107. GPU Radix Sort as a Geometry Primitive

*Audience: systems developers.*

Radix sort underlies a surprising fraction of GPU geometry algorithms: Morton code sorting for BVH construction (§31), particle bucket assignment (§48), mesh edge/face key sorting for adjacency construction (§18, §106), histogram prefix sums for stream compaction (§26, §72), and order-independent transparency (§85). This section presents the canonical 4-pass GPU radix sort (Merrill & Grimshaw 2010 / CUB/Thrust pattern) in Vulkan compute, applicable as a geometry primitive throughout the chapter.

### 107.1 Histogram Pass: Count Digit Frequencies

For each 2-bit digit position, count the frequency of each of the 4 digit values within each thread block:

```glsl
// radix_histogram.comp — count 2-bit digit frequencies per workgroup
layout(set=0,binding=0) readonly buffer Keys   { uint keys[];  };
layout(set=0,binding=1) buffer Histograms { uint hist[N_WORKGROUPS][4]; };
layout(push_constant) uniform PC { uint N; uint bitOffset; } pc;
layout(local_size_x=256) in;
shared uint sharedHist[4];
void main() {
    if (gl_LocalInvocationID.x < 4u) sharedHist[gl_LocalInvocationID.x]=0u;
    barrier();
    uint i  = gl_GlobalInvocationID.x;
    if (i < pc.N) {
        uint digit = (keys[i] >> pc.bitOffset) & 3u;
        atomicAdd(sharedHist[digit], 1u);
    }
    barrier();
    if (gl_LocalInvocationID.x < 4u)
        hist[gl_WorkGroupID.x][gl_LocalInvocationID.x] = sharedHist[gl_LocalInvocationID.x];
}
```

[Source: Merrill & Grimshaw "Revisiting Sorting for GPGPU Stream Architectures", PACT 2010; CUB DeviceRadixSort]

### 107.2 Prefix Sum (Scan) Across Histograms

Exclusive prefix-sum the per-workgroup histograms to produce the global scatter offsets for each digit value:

```glsl
// radix_scan.comp — exclusive prefix sum of histograms across workgroups
layout(set=0,binding=0) readonly buffer Histograms { uint hist[N_WORKGROUPS][4]; };
layout(set=0,binding=1) writeonly buffer Offsets   { uint offsets[N_WORKGROUPS][4]; };
layout(local_size_x=N_WORKGROUPS) in;
shared uint sharedScan[N_WORKGROUPS];
void main() {
    uint digit = gl_WorkGroupID.x;  // one workgroup per digit value
    uint wg    = gl_LocalInvocationID.x;
    sharedScan[wg] = (wg < N_WORKGROUPS) ? hist[wg][digit] : 0u;
    barrier();
    // Hillis-Steele inclusive scan
    for (uint s=1u; s<N_WORKGROUPS; s<<=1) {
        uint v = (wg>=s) ? sharedScan[wg-s] : 0u;
        barrier(); sharedScan[wg] += v; barrier();
    }
    // Convert to exclusive by shifting right
    offsets[wg][digit] = (wg>0) ? sharedScan[wg-1] : 0u;
}
```

[Source: Harris et al. "Parallel Prefix Sum (Scan) with CUDA", GPU Gems 3 2007; Hillis & Steele "Data Parallel Algorithms", CACM 1986]

### 107.3 Scatter Pass: Place Keys at Computed Offsets

Each thread computes its output position from the workgroup-local prefix sum and the global offset, then scatters the key-value pair:

```glsl
// radix_scatter.comp — scatter keys and values to sorted positions
layout(set=0,binding=0) readonly buffer KeysIn   { uint keysIn[];   };
layout(set=0,binding=1) readonly buffer ValsIn   { uint valsIn[];   };
layout(set=0,binding=2) readonly buffer Offsets  { uint offsets[][4]; };
layout(set=0,binding=3) writeonly buffer KeysOut { uint keysOut[];  };
layout(set=0,binding=4) writeonly buffer ValsOut { uint valsOut[];  };
layout(push_constant) uniform PC { uint N; uint bitOffset; } pc;
layout(local_size_x=256) in;
shared uint sharedRanks[256];
void main() {
    uint i     = gl_GlobalInvocationID.x;
    uint li    = gl_LocalInvocationID.x;
    uint wg    = gl_WorkGroupID.x;
    uint digit = (i < pc.N) ? (keysIn[i] >> pc.bitOffset) & 3u : 0u;
    // Compute local rank within workgroup for this digit using shared ballot
    uint rank  = localRank(digit, li);  // count of same digit before li
    sharedRanks[li] = offsets[wg][digit] + rank;
    barrier();
    if (i < pc.N) { keysOut[sharedRanks[li]]=keysIn[i]; valsOut[sharedRanks[li]]=valsIn[i]; }
}
```

[Source: Merrill & Grimshaw 2010; Satish et al. "Designing Efficient Sorting Algorithms for Manycore GPUs", IPDPS 2009]

### 107.4 Full 32-Bit Sort and Application to Geometry

Compose 16 passes of 2-bit radix sort (or 8 passes of 4-bit) to sort 32-bit keys. Apply to geometry: sort triangle indices by Morton code (§31.1) for BVH construction, or sort vertex indices by material ID (§82.3 wavefront path tracing), or sort particle cell IDs for spatial hash (§27):

```glsl
// morton_sort_dispatch.comp — sort primitives by 30-bit Morton code for LBVH
layout(set=0,binding=0) readonly buffer Centers { vec3 cen[]; };
layout(set=0,binding=1) writeonly buffer MortonKeys { uint mortonKey[]; };
layout(set=0,binding=2) writeonly buffer SortVals   { uint triIdx[];    };
layout(push_constant) uniform PC { vec3 sceneMin; float invSceneSize; uint N; } pc;
layout(local_size_x=64) in;
void main() {
    uint t   = gl_GlobalInvocationID.x;
    if (t >= pc.N) return;
    vec3  n  = (cen[t]-pc.sceneMin)*pc.invSceneSize;  // normalise to [0,1]³
    uvec3 q  = uvec3(n*1023.0);  // 10 bits per axis
    // Interleave bits: Morton code
    uint m = 0u;
    for(int b=0;b<10;b++) m|=((q.x>>b)&1u)<<(3*b)|((q.y>>b)&1u)<<(3*b+1)|((q.z>>b)&1u)<<(3*b+2);
    mortonKey[t]=m; triIdx[t]=t;
}
// Then run §107.1–107.3 for 15 passes (30-bit sort); result is LBVH input order (§31.2)
```

[Source: Lauterbach et al. "Fast BVH Construction on GPUs", EG 2009 §70; Karras "Maximizing Parallelism in the Construction of BVHs, Octrees, and k-d Trees", HPG 2012]

---


---

## 108. Library Landscape

**Coverage gap.** No single open-source library covers more than a narrow slice of the algorithms in this chapter. The table below maps the available libraries against the chapter's topic areas; empty cells represent functionality that must be implemented directly in shader/compute code using the patterns shown in the preceding sections. This fragmentation is the norm in GPU geometry programming: the field is young enough that GPU-native algorithmic libraries have not yet consolidated around a common abstraction layer comparable to what BLAS/LAPACK provide for linear algebra.

The closest approximation to a broad GPU geometry toolkit is **NVIDIA's** ecosystem when CUDA is available: cuSPARSE (sparse solvers for Poisson/LSCM), Thrust (prefix scans, radix sort, stream compaction), and NVCC-compiled versions of CPU libraries. On non-NVIDIA hardware the pattern in this chapter — implementing each algorithm as a self-contained Vulkan compute shader from first principles — remains the only portable path.

Two tables follow — one per group of nine sections — because a single 22-column table is unreadable. — means no coverage in that section.

**Table A — §1–§25 coverage**

| Library | Ver | GPU Backend | §1 Subdiv | §2 NURBS | §3 Implicit | §39 Skeletal | §40 IK | §4 Splines | §10 Mesh | §24 BVH/Coll | §25 SS | Best Use |
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

**Table B — §26–§70 coverage**

| Library | Ver | GPU Backend | §26 GPU-Driven | §48 Fluid | §49 Deform | §69 Proc | §34 Geo | §41 Shape | §91 3DGS | §27 Spatial | §35 Spectral | §70 NavMesh | Best Use |
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

**libigl** ([source](https://github.com/libigl/libigl)) provides `igl::lbs_matrix()` and `igl::dqs()` for skinning, `igl::bbw()` for bounded biharmonic weights, `igl::lscm()` and `igl::harmonic()` for UV parameterization, `igl::marching_tets()` for isosurface extraction, `igl::heat_geodesics()` and `igl::cotmatrix()` for geodesics and the cotangent Laplacian, `igl::arap()` for ARAP deformation, `igl::mvc()` for mean value coordinates, and `igl::eigs()` for spectral geometry. All CPU-side using Eigen; upload results to GPU as vertex buffers. libigl has the widest algorithmic breadth of any portable library in this chapter — it spans §1, §3, §39, §10, §34, §41, and §35 — but has no GPU execution path.

**GeometricTools** ([source](https://github.com/davideberly/GeometricTools), Boost license) provides `GTL::NURBSSurface<float, 3>` and `GTL::CatmullClarkSurface` as header-only CPU implementations. Use for offline asset preprocessing.

**meshoptimizer** ([source](https://github.com/zeux/meshoptimizer), MIT) provides `meshopt_simplify()`, `meshopt_optimizeVertexCache()`, and `meshopt_buildMeshlets()` with `meshopt_computeMeshletBounds()` — the last two are §26 GPU-Driven Rendering directly. Generate the full LOD chain and meshlet data offline, upload to a single `VkBuffer`, and dispatch via the task/mesh shader pipeline.

**NanoVDB** ([source](https://github.com/AcademySoftwareFoundation/openvdb), Apache 2.0, OpenVDB 9.0+) converts OpenVDB trees to flat GPU buffers via `nanovdb::openToNanoVDB()`. The companion GLSL header provides tree-descent accessors matching §27.3 (SVO traversal). Requires `VK_EXT_scalar_block_layout`.

**xatlas** ([source](https://github.com/jpcy/xatlas), MIT) generates UV atlases as a preprocessing step for texture baking (§10.13). CPU-only; outputs UV coordinates and a remapped index buffer ready for GPU upload.

**PoissonRecon** ([source](https://github.com/mkazhdan/PoissonRecon), MIT) is the reference screened Poisson surface reconstruction (§3.12). CPU multi-threaded (TBB); 10–60 s for 1M-point clouds.

**V-HACD 4.0** ([source](https://github.com/kmammou/v-hacd), BSD-3) decomposes non-convex meshes into convex hull approximations for physics (§24.5). CPU-only.

**Triangle** ([source](https://www.cs.cmu.edu/~quake/triangle.html), Shewchuk, public domain) is the reference 2D CDT and Ruppert quality-mesh generator (§3.11). Single-file C library.

**SPlisHSPlasH** ([source](https://github.com/InteractiveComputerGraphics/SPlisHSPlasH), MIT) is the most complete open-source GPU fluid simulation library. It implements SPH, PBF (Macklin 2013), DFSPH (divergence-free SPH), IISPH, PBD, and FLIP variants, all with CUDA backends. Spatial hashing for neighbour search runs on GPU; pressure and density solvers run fully on-device. The library outputs particle positions each step — couple to the SPH surface extraction pipeline from §3.8 to produce renderable meshes in real time.

**PhysX 5** ([source](https://github.com/NVIDIA-Omniverse/PhysX), BSD-3) adds GPU-native FEM soft bodies (corotational linear elasticity on tet meshes, §49.2) and GPU cloth (Position-Based Dynamics, matching §39.6 and §49.1) alongside GPU rigid body simulation and PBD fluid. The soft body solver runs entirely on CUDA; results are available as CUDA device pointers that map directly to Vulkan external memory via `VK_KHR_external_memory` + `VK_EXT_external_memory_host`. No Vulkan-native API; requires interop.

**geometry-central** ([source](https://github.com/nmwsharp/geometry-central), MIT) is the research reference implementation for discrete differential geometry: `HeatMethodDistance` (§34.2), cotangent Laplacian assembly, `StripePatterns` for seamless parameterization, vector heat method, and spectral basis computation (§35.1). CPU-only; results upload to GPU. Essential for verifying custom Vulkan compute DDG implementations against a known-correct baseline.

**gsplat** ([source](https://github.com/nerfstudio-project/gsplat), Apache 2.0) is a production-quality GPU 3DGS rasterizer supporting CUDA and ROCm backends. It implements the full pipeline from §91 — Gaussian projection, tile assignment, radix sort, and per-tile alpha compositing — with forward and backward passes for training. For Vulkan integration, render to a CUDA surface then copy via external memory, or port the tile rasterizer kernels to Vulkan compute using the same algorithm (§91.2).

**FastNoise2** ([source](https://github.com/Auburn/FastNoise2), MIT) provides a SIMD node-graph noise system supporting Perlin, OpenSimplex2, Cellular, Domain Warp, and fractal combinations (§69.1 terrain fBm). A CUDA backend is available for GPU-side evaluation. On CPU, AVX2/AVX-512 SIMD makes it fast enough for real-time heightmap generation.

**Recast/Detour** ([source](https://github.com/recastnavigation/recastnavigation), Zlib) is the canonical navmesh library used by most game engines. Recast's voxelisation and region-labelling steps (§70.1–19.2) are CPU-only; no GPU port exists. The flow-field construction in §70.3 is implemented directly in compute shaders without a library equivalent. Recast is the correct preprocessing tool for static navmesh geometry; Detour handles the runtime pathfinding query API.

**Table C — §75–§96 coverage**

| Library | Ver | GPU Backend | §75 RT Geom | §87 Point Cloud | §50 Rigid Body | §51 Crowd | §11 Remesh | §52 Fracture | §71 Terrain | §92 Geo DL | §5 2D Vector | §96 Sci Vis | Best Use |
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

**Vulkan RT Extensions** (KHR, Vulkan 1.2+) are the only "library" for §75 — AS build/update APIs are in the driver. VulkanMemoryAllocator ([source](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator)) simplifies the scratch/result buffer lifecycle for BLAS/TLAS operations.

**PCL** ([source](https://pointclouds.org), BSD-3) provides `pcl::KdTreeFLANN<pcl::PointXYZ>` for CPU k-NN, `pcl::IterativeClosestPoint<>` for ICP, `pcl::SACSegmentation<>` for RANSAC plane/sphere fitting, and `pcl::visualization::PCLVisualizer` for scientific vis. No GPU backend; use for preprocessing before uploading point clouds to Vulkan SSBOs.

**Open3D** ([source](https://www.open3d.org), MIT) adds a CUDA k-NN search (`open3d.core.nns.NearestNeighborSearch`), GPU ICP variants (point-to-plane, colored ICP), RANSAC registration, and a PointNet++ training pipeline via PyTorch. The `open3d.visualization.Visualizer` covers §96 scientific vis. Open3D is the most capable GPU point cloud toolkit outside CUDA-only paths.

**Bullet 3** ([source](https://github.com/bulletphysics/bullet3), Zlib) implements SAP broadphase, sequential impulse (Catto solver), and Featherstone ABA for articulated bodies (§50). A partial OpenCL backend (`btGpuBroadphaseProxy`) parallelises the broadphase; the narrow-phase and solver remain on CPU. The `HACD` component provides Voronoi-style approximate convex decomposition for pre-fracture (§52.1).

**RVO2** ([source](https://gamma.cs.unc.edu/RVO2/), Apache 2.0) is the reference CPU implementation of ORCA (van den Berg 2011) for crowd simulation (§51.1). For GPU port use the compute shader pattern from §51.1 with the §27.2 spatial hash for neighbourhood queries.

**OpenMesh** ([source](https://www.graphics.rwth-aachen.de/OpenMesh/), LGPL) provides a halfedge-based mesh data structure with `OpenMesh::Subdivider::Uniform::CatmullClarkT<>` and edge split/collapse/flip operations for isotropic remeshing (§11.1). CPU-only; use for offline remesh passes before GPU upload.

**msdfgen** ([source](https://github.com/Chlumsky/msdfgen), MIT) generates multi-channel SDF atlases from font outlines or SVG paths — the offline preprocessing step for §5.2 MSDF font rendering. Produces a `VkImage` data asset; no runtime GPU component.

**PyTorch Geometric** ([source](https://pyg.org), MIT) provides `torch_geometric.nn.PointNetConv`, graph convolution layers, and batched k-NN search (`torch_cluster.knn_graph`) all running on CUDA or ROCm. The trained model weights (§92) can be extracted and run as a Vulkan compute shader MLP inference pass.

**VTK** ([source](https://vtk.org), BSD-3) covers §96 scientific visualization: `vtkStreamTracer` for streamlines, `vtkVolume` with GPU ray casting, `vtkTensorGlyph` for superquadrics, and `vtkMarchingCubes` for isosurface extraction. The OpenGL compute backend (`vtkOpenGLGPUVolumeRayCastMapper`) uses GL compute shaders; Vulkan integration requires the VTK-m ([source](https://m.vtk.org)) parallel framework with a Kokkos backend.

**Table D — §42–§29 coverage**

| Library | Ver | GPU Backend | §42 Hair | §61 LSM | §93 DiffRender | §72 Particles | §28 Lightmap | §12 UV Pack | §62 Swept Vol | §36 Morphing | §97 Decals | §29 Occlusion | Best Use |
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

**TressFX 4.1** ([source](https://github.com/GPUOpen-Effects/TressFX), MIT) is the most complete open-source GPU hair library, implementing the full §42 pipeline: position-based strand simulation (stretch, bending, torsion constraints), strand self-collision via spatial hashing, collision with scene geometry, and LOD-controlled strand-to-shell transitions. Vulkan and DX12 backends. Integrates with the engine's TLAS via `VK_KHR_ray_tracing_pipeline` for strand shadow rays.

**NvDiffRast** ([source](https://github.com/NVIDIAGameWorks/nvdiffrast), Apache 2.0) provides CUDA-accelerated differentiable rasterisation (§93.1): forward rasteriser with anti-aliased edge gradients, texture sample backward pass, and interpolation gradients. Includes a `torch.autograd.Function` wrapper. For Vulkan integration, run the forward pass in nvdiffrast (CUDA) and extract gradients into a Vulkan-mapped buffer via `VK_KHR_external_memory`.

**PyTorch3D** ([source](https://github.com/facebookresearch/pytorch3d), BSD) extends §92 (geometric deep learning) with differentiable mesh rendering (§93), functional map utilities (§36), and point cloud operations. The `pytorch3d.renderer.MeshRasterizer` implements the soft rasterisation from §93.1.

**Embree 4** ([source](https://www.embree.org), Apache 2.0) provides CPU ray tracing via AVX-512/AVX2 SIMD — the standard reference backend for offline lightmap baking (§28). `rtcOccluded1M()` batches occlusion queries for AO hemisphere sampling. Pair with a GPU denoiser (OIDN) for high-quality baked lighting.

**Radeon Rays 4** ([source](https://github.com/GPUOpen-LibrariesAndSDKs/RadeonRays_SDK), MIT) is the GPU counterpart: Vulkan and HIP ray query acceleration for AO baking (§28) and differentiable rendering (§93) on AMD hardware. Complements the `VK_KHR_ray_query` inline queries in §28.2 with a higher-level batch API.

**Table E — §73–§53 coverage**

| Library | Ver | GPU Backend | §73 Ocean | §13 Voxelise | §63 TSDF | §76 Shadows | §77 IBL | §43 SecAnim | §14 Morph | §105 Sampling | §15 ProgMesh | §53 Rope | Best Use |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| [VkFFT](https://github.com/DTolm/VkFFT) | 1.3 | Vulkan / CUDA / HIP | ✓ IFFT core | — | — | — | — | — | — | — | — | — | GPU FFT for ocean |
| [Crest Ocean](https://github.com/wave-harmonic/crest) | 4.x | Unity (Vulkan) | ✓ full pipeline | — | — | — | — | — | — | — | — | — | Production ocean system |
| [Open3D](https://www.open3d.org) | 0.18 | CUDA | — | — | ✓ KinFu | — | — | — | — | — | — | — | TSDF fusion (KinectFusion) |
| [OpenVDB](https://www.openvdb.org) | 11.0 | None (CPU + TBB) | — | ✓ mesh→VDB | — | — | — | — | ✓ morphology | — | — | — | Mesh voxelisation + morphology |
| [CGAL](https://www.cgal.org) | 6.1 | None | — | ✓ conservative | — | — | — | — | ✓ morph (CPU) | ✓ Poisson (CPU) | — | — | Geometry preprocessing |
| [Falcor](https://github.com/NVIDIAGameWorks/Falcor) | 7.0 | Vulkan / DX12 | — | — | — | ✓ CSM/RT shadow | ✓ PMREM/LUT | — | — | ✓ Halton/Sobol | — | — | Research renderer, IBL + shadows |
| [Filament](https://github.com/google/filament) | 1.x | Vulkan / Metal | — | — | — | ✓ CSM/DPCF | ✓ IBL bake | — | — | — | — | — | Production PBR + IBL |
| [igl](https://github.com/libigl/libigl) | 2.5 | None (CPU) | — | — | — | — | — | ✓ corrective shapes | ✓ medial axis | ✓ Poisson (CPU) | ✓ progressive | — | Shape analysis, progressive mesh |
| [Discrete Elastic Rods](https://github.com/bastibl/der) | research | None (CPU) | — | — | — | — | — | — | — | — | — | ✓ Cosserat sim | Cosserat rod reference impl |
| [OIIO](https://github.com/AcademySoftwareFoundation/OpenImageIO) | 2.5 | None (CPU) | — | — | — | — | ✓ env filter | — | — | — | — | — | HDR env map processing |

**VkFFT** ([source](https://github.com/DTolm/VkFFT), MIT) is the only portable GPU FFT with a Vulkan backend. It generates optimal SPIR-V at runtime for the target device's subgroup size and supports batch 2D FFTs — the exact operation §73.2 requires for the Tessendorf ocean IFFT.

**Crest Ocean** ([source](https://github.com/wave-harmonic/crest), MIT) is a production GPU ocean system implementing the full §73 pipeline: Phillips spectrum initialisation, GPU FFT via Unity's compute, choppy wave Jacobian foam, Gerstner wave blending, and LOD-aware shoreline interaction. The Unity Vulkan backend means its shader source maps directly to the GLSL compute patterns in §73.

**Open3D** covers §63 (TSDF fusion) with its `open3d.pipelines.integration.ScalableTSDFVolume` (voxel hashing, §63.4) and `open3d.pipelines.odometry` (ICP tracking, §63.2). The CUDA backend fuses 512³ volumes at 30 fps on modern hardware.

**OpenVDB** ([source](https://www.openvdb.org), MPL-2.0) provides `openvdb::tools::meshToVolume()` for conservative mesh voxelisation (§13.1) and `openvdb::tools::morphology::dilateVoxels()` / `erodeVoxels()` for morphological operations (§14.1). No GPU execution path; NanoVDB (Table A) handles the GPU-side sparse volume traversal.

**Falcor** ([source](https://github.com/NVIDIAGameWorks/Falcor), BSD-3) is NVIDIA's research renderer with Vulkan/DX12 backends. It implements cascaded shadow maps (§76.1), ray-traced shadows via `VK_KHR_ray_query` (§76.4), PMREM prefiltering (§77.2), BRDF LUT baking (§77.3), and low-discrepancy samplers (Halton, Sobol — §105.1). Its shader source is the best available reference for §76–§77 on Vulkan.

**Filament** ([source](https://github.com/google/filament), Apache 2.0) is Google's production PBR renderer with a Vulkan backend. It implements DPCF (Distance-field Percentage Closer Filtering) soft shadows, a custom cascaded shadow system (§76.1), IBL prefiltering (§77.2–44.3), and the split-sum approximation. The Filament material compiler generates GLSL/SPIR-V from a material definition language — a useful reference for §77 IBL integration in Vulkan.

**libigl** (Table A/B) also covers §15: `igl::decimate()` generates the edge-collapse sequence, and `igl::qslim()` records vertex split data in Hoppe-compatible format. The progressive mesh implementation is CPU-side; upload the pre-computed split stream as an SSBO for GPU playback (§15.2).

**Table F — §54–§55 coverage**

| Library | Ver | GPU Backend | §54 Cloth | §78 GI | §64 SDF-Col | §98 ProcTex | §99 Acoustic | §16 Compress | §79 Caustics | §6 NURBS | §44 Retarget | §55 Fluid-Solid | Best Use |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| [XPBD (Matthias Müller)](https://matthias.game-tech.ch/research/small-steps-in-physics-simulation/) | research | CPU/CUDA | ✓ cloth XPBD | — | — | — | — | — | — | — | — | — | XPBD cloth reference |
| [Nvidia Flex / PhysX 5](https://developer.nvidia.com/flex) | 5.x | CUDA | ✓ cloth | — | ✓ SDF contacts | — | — | — | — | — | ✓ coupling | CUDA cloth + fluid coupling |
| [meshoptimizer](https://github.com/zeux/meshoptimizer) | 0.21 | None (CPU) | — | — | — | — | — | ✓ quantize/delta | — | — | — | — | Mesh compression pre-pass |
| [Draco](https://github.com/google/draco) | 1.5 | None (CPU) | — | — | — | — | — | ✓ edgebreaker | — | — | — | — | Connectivity + pos compression |
| [Basis Universal](https://github.com/BinomialLLC/basis_universal) | 1.16 | GPU transcode | — | — | — | ✓ geom textures | — | ✓ BC7/ETC2 | — | — | — | — | GPU texture codec |
| [Steam Audio](https://valvesoftware.github.io/steam-audio/) | 4.5 | CPU/AVX2 | — | — | — | — | ✓ acoustic RT | — | — | — | — | — | GPU occlusion + acoustic RT |
| [OpenCASCADE](https://www.opencascade.com) | 7.8 | None (CPU) | — | — | — | — | — | — | — | ✓ NURBS exact | — | — | CAD kernel, NURBS trim |
| [SVGF (Schied ref)](https://github.com/tiechui94/SVGF) | research | Vulkan | — | ✓ denoising | — | — | — | — | ✓ photon denoise | — | — | — | GI spatiotemporal denoising |
| [Cem Yuksel Yarn](https://www.cemyuksel.com/research/yarnrendering/) | research | CUDA | ✓ orthotropy | — | — | — | — | — | — | — | — | — | Woven fabric orthotropy |
| [Mixamo / Rokoko](https://www.rokoko.com) | SaaS | Cloud | — | — | — | — | — | — | — | — | ✓ retarget | — | Animation retargeting service |

**XPBD cloth** (Müller et al. 2020 "Small Steps in Physics Simulation") is the reference implementation for §54. Orthotropic constraints (§54.1), self-collision (§54.2), and tearing (§54.4) follow the patterns in the paper. No Vulkan port exists; the compute shader implementations in §54 are the primary Vulkan-native path.

**NVIDIA Flex / PhysX 5** (CUDA backend) covers §54 (cloth via particle position constraints and shape matching), §64.1 (SDF contact generation), and §55.4 (two-way fluid-solid coupling via SPH). It is CUDA-only; the Vulkan-native equivalents are the custom compute pipelines in each section.

**meshoptimizer** (§16.1, §16.4) provides `meshopt_quantizePosition()` for geometry quantization and the meshlet delta encoding scheme. The CPU decode runs before GPU upload; §16.1 shows the GPU dequantisation compute pass for streaming scenarios.

**Draco** (§16.2) implements edgebreaker connectivity compression and parallelogram position prediction. The GPU-side prediction pass (§16.2 listing) accelerates the final attribute decode for large meshes.

**Basis Universal** (§16.3) transcodes to ETC2/BC7 on CPU or GPU. The GPU transcoding compute shader pattern in §16.3 maps to `basisu_gpu.comp` in the upstream codebase.

**Steam Audio** (§99) provides GPU-accelerated acoustic ray tracing via CPU path tracing with AVX2 vectorisation and optional GPU occlusion queries. The Vulkan ray query approach in §99.3 is the Vulkan-native complement; Steam Audio is the production-quality reference.

**OpenCASCADE** (§6) is the reference CAD kernel implementing exact NURBS evaluation, trimming via boundary representation, and offset curve approximation. The CPU-side evaluation in §6.1-57.3 uses OCC geometry; the GPU Newton iteration in §6.1 replaces the OCC evaluator for display.

**Table G — §94–§56 coverage**

| Library | Ver | GPU Backend | §94 3DGS | §65 Isosurface | §66 AO | §7 CSG | §67 VolRender | §37 Geodesic | §17 UV Param | §18 Delaunay | §30 SVDAG | §56 DEM | Best Use |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| [gaussian-splatting](https://github.com/graphdeco-inria/gaussian-splatting) | research | CUDA | ✓ full pipeline | — | — | — | — | — | — | — | — | — | Reference 3DGS training+render |
| [gsplat](https://github.com/nerfstudio-project/gsplat) | 1.x | CUDA / Vulkan | ✓ optimized | — | — | — | — | — | — | — | — | — | Production 3DGS rasterizer |
| [OpenVDB/NanoVDB](https://www.openvdb.org) | 11.0 | CUDA/Vulkan | — | ✓ MC on VDB | — | — | ✓ DDA traversal | — | — | — | — | — | Volumetric rendering + MC |
| [XeGTAO](https://github.com/GameTechDev/XeGTAO) | main | DX12/Vulkan | — | — | ✓ GTAO | — | — | — | — | — | — | — | Ground-truth AO reference |
| [geometry-central](https://github.com/nmwsharp/geometry-central) | main | None (CPU) | — | — | — | — | — | ✓ heat method | ✓ LSCM/ARAP | — | — | — | Geodesic distance + UV |
| [xatlas](https://github.com/jpcy/xatlas) | main | None (CPU) | — | — | — | — | — | — | ✓ atlas pack | — | — | — | UV atlas generation |
| [Cork CSG](https://github.com/gilbo/cork) | main | None (CPU) | — | — | — | ✓ winding # | — | — | — | — | — | — | Boolean mesh operations |
| [Frostbite Atmosphere](https://github.com/sebh/UnrealEngineSkyAtmosphere) | research | Vulkan/DX12 | — | — | — | — | ✓ atmospheric LUT | — | — | — | — | — | Physically based sky |
| [DAGCompression](https://github.com/Phyronnaz/DAGCompression) | research | CUDA | — | — | — | — | — | — | — | — | ✓ SVDAG build | — | SVO→DAG compression |
| [LIGGGHTS/YADE](https://yade-dem.org) | 3.x | None (CPU+MPI) | — | — | — | — | — | — | — | — | — | ✓ DEM sim | Reference DEM solver |

**gaussian-splatting** (Kerbl et al. 2023, MIT) is the original reference implementation of §94 in CUDA PyTorch. The **gsplat** library ([nerfstudio-project](https://github.com/nerfstudio-project/gsplat)) is a faster, more maintainable CUDA rasterizer with an emerging Vulkan compute backend, implementing the tile-based radix-sort + compositing pipeline of §94.2 at production performance.

**NanoVDB** covers §65.4 (adaptive isosurface extraction from sparse volumes) and §67.4 (DDA traversal for volumetric ray marching). Its GLSL bindings allow direct use in Vulkan compute shaders without CUDA. OpenVDB's `tools::VolumeToMesh` implements Dual Contouring (§65.2) on the CPU; GPU marching cubes on OpenVDB nodes uses the §65.1 pattern applied to NanoVDB leaf data.

**XeGTAO** ([Intel GameTechDev](https://github.com/GameTechDev/XeGTAO), MIT) is the reference implementation of §66.3 GTAO with spatial denoising, a HLSL/DX12 implementation straightforwardly portable to GLSL/Vulkan. It includes bent-normal computation and temporal accumulation.

**geometry-central** ([nmwsharp](https://github.com/nmwsharp/geometry-central), MIT) implements the heat method (§37) and the vector heat method (§37.4) with cotangent Laplacian assembly and Cholesky factorization (via Eigen). The linear system structure is GPU-amenable via the Jacobi-CG patterns in §37.1–65.3; geometry-central provides the reference for the operator assembly step. It also implements LSCM and ARAP parameterization (§17.1–66.2).

**xatlas** (§17.4, MIT) is the most widely deployed GPU-rendering UV atlas library (used in Unity, Blender, many game studios). It generates seam cuts (§17.3 spanning tree approach) and packs charts into a rectangle atlas. CPU-only; the GPU upload of the resulting UV buffer is the standard integration path.

**Table H — §31–§57 coverage**

| Library | Ver | GPU Backend | §31 BVH Build | §32 SDF Bake | §19 Fairing | §95 Neural Geom | §8 MetaBalls | §9 Poisson | §100 VR/XR | §38 Spectral | §88 SfM/MVS | §57 Erosion | Best Use |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| [RadeonRays 4](https://github.com/GPUOpen-LibrariesAndSDKs/RadeonRays_SDK) | 4.x | Vulkan/HIP | ✓ LBVH+PLOC | — | — | — | — | — | — | — | — | — | GPU BVH build on AMD/Vulkan |
| [Embree 4](https://github.com/embree/embree) | 4.x | ISPC/SYCL | ✓ SAH-HLBVH | — | — | — | — | — | — | — | — | — | High-quality CPU/SYCL BVH |
| [instant-ngp](https://github.com/NVlabs/instant-ngp) | main | CUDA | — | — | — | ✓ NGP hash-grid | — | — | — | — | — | — | Real-time NeRF training+inference |
| [nerfstudio](https://github.com/nerfstudio-project/nerfstudio) | 1.x | CUDA | — | — | — | ✓ NeRF/Gaussian | — | — | — | — | — | — | NeRF/3DGS training framework |
| [PoissonRecon](https://github.com/mkazhdan/PoissonRecon) | 15.x | None (CPU+TBB) | — | — | — | — | — | ✓ screened Poisson | — | — | — | — | Watertight surface from points |
| [geometry-central](https://github.com/nmwsharp/geometry-central) | main | None (CPU) | — | — | ✓ Taubin/implicit | — | — | — | — | ✓ HKS/WKS | — | — | Mesh fairing + spectral descriptors |
| [OpenVR/OpenXR SDK](https://github.com/KhronosGroup/OpenXR-SDK) | 1.1 | Vulkan | — | — | — | — | — | — | ✓ ATW/ASW | — | — | — | VR reprojection API |
| [COLMAP](https://github.com/colmap/colmap) | 3.x | CUDA | — | — | — | — | — | — | — | — | ✓ SfM+MVS | — | Reference SfM pipeline |
| [OpenMVS](https://github.com/cdcseacave/openMVS) | 2.x | CUDA | — | — | — | — | — | — | — | — | ✓ MVS dense | — | Dense multi-view stereo |
| [Terrain erosion (GPU)](https://github.com/bshishov/UnityTerrainErosionGPU) | research | Vulkan/Unity | — | — | — | — | — | — | — | — | — | ✓ hydraulic+thermal | GPU erosion reference impl |

**RadeonRays 4** ([GPUOpen](https://github.com/GPUOpen-LibrariesAndSDKs/RadeonRays_SDK), MIT) implements LBVH and PLOC BVH construction (§31.1–70.4) with a Vulkan backend — the only portable Vulkan-native BVH builder available as a library. Its construction pipeline matches the Morton-code + radix-sort + refit pattern of §31 exactly; it exports `VkAccelerationStructure`-compatible BVH data for use with `VK_KHR_ray_tracing_pipeline`.

**Embree 4** (Intel, Apache 2.0) implements SAH-HLBVH and PLOC with an ISPC CPU backend and an emerging SYCL (oneAPI) GPU backend. On x86 CPUs it is the quality reference; the SYCL path targets Intel Arc GPUs and provides a template for §31.4 PLOC on non-AMD Vulkan hardware.

**instant-ngp** (§95.2, NVIDIA, custom license) implements the multiresolution hash grid encoding and tiny-MLP inference in CUDA. Its Vulkan interop path (for display) uses CUDA–Vulkan semaphore synchronisation; the hash grid structure in §95.2 is taken directly from its CUDA kernel. **nerfstudio** provides a training framework around gsplat (§94), instant-ngp, and NeuS (§95.4) with a Python API for inference export.

**PoissonRecon** (Kazhdan, MIT) is the reference implementation of §9 with adaptive octree refinement, screened Poisson solve, and isovalue selection. CPU+TBB; the GPU patterns in §9.1–75.4 show how to accelerate the splatting, Laplacian assembly, and isovalue computation passes.

**COLMAP** (§88, BSD-3) implements the full SfM pipeline (feature detection, matching, geometric verification, bundle adjustment) with CUDA-accelerated feature extraction and matching (§88.1). **OpenMVS** extends it with GPU-accelerated SGM depth estimation (§88.3) and depth map fusion (§88.4) on CUDA.

**What is missing.** Spanning all eight tables, the most significant gaps on Vulkan are: GPU BVH construction (covered by RadeonRays on Vulkan; Embree on SYCL), SDF baking (§32 — no Vulkan library; jump-flooding compute patterns are the only path), mesh fairing (§19 — CPU-only in geometry-central), and terrain erosion (§57 — Unity/CUDA reference only). Neural geometry (§95) is CUDA-exclusive. Spectral processing (§38) has no GPU library. The CUDA ecosystem now covers approximately thirty-five of the seventy-nine sections; the Vulkan compute shader patterns across §26–§57 remain the primary portable path for the remainder.

**Table I — §58–§22 coverage**

| Library | Ver | GPU Backend | §58 Fluid Surf | §59 CCD | §106 ConvHull | §20 Remesh | §80 SSR | §101 Silhouette | §89 Descriptors | §21 Segment | §81 Radiosity | §22 Repair | Best Use |
|---------|-----|-------------|:--------------:|:-------:|:------------:|:----------:|:-------:|:--------------:|:---------------:|:-----------:|:-------------:|:----------:|----------|
| SplishSplash | 2.13 | CUDA | ✓ (anisotropic §58.2) | — | — | — | — | — | — | — | — | — | SPH/FLIP fluid simulation |
| bullet3 | 3.25 | CUDA/OpenCL | — | ✓ (§59.1 conservative) | ✓ (GJK-based) | — | — | — | — | — | — | — | Rigid body + CCD |
| Open3D | 0.18 | CUDA/Tensor | — | — | ✓ (GPU QuickHull §106) | ✓ (CVT §20.4) | — | — | ✓ (FPFH §89.2) | — | — | ✓ (§22.1) | Point cloud processing |
| PCL | 1.14 | CUDA (partial) | — | — | — | — | — | — | ✓ (FPFH/SHOT §89) | ✓ (k-means §21.2) | — | — | Robotics point cloud |
| meshoptimizer | 0.21 | CPU | — | — | — | ✓ (edge ops §20.2) | — | — | — | — | — | ✓ (§22.4) | Mesh optimisation/repair |
| libigl | 2.5 | CPU+OpenGL | — | — | — | ✓ (ARAP §17) | — | ✓ (§101.3) | — | ✓ (rw §21.3) | — | ✓ (§22.3) | Geometry processing research |
| geometry-central | 1.0 | CPU | — | — | — | ✓ (flip §20.1) | — | ✓ (§101.4) | — | — | — | — | Differential geometry |
| Radeon Rays | 4.0 | Vulkan/HIP | — | — | — | — | — | — | — | — | — | — | Ray query (used by §80.1 HiZ) |
| drjit / Mitsuba 3 | 0.4 | CUDA/LLVM | ✓ (§58.3 depth smooth) | — | — | — | — | — | — | — | ✓ (§81.2 radiosiy) | — | Differentiable rendering |
| NVIDIA VKRay | SDK | Vulkan | — | — | — | — | ✓ (§80.1 HiZ) | — | — | — | — | — | RTX ray tracing SDK |
| Manifold | 2.4 | CPU | — | — | — | — | — | — | — | — | — | ✓ (watertight CSG §22) | Robust mesh operations |

**SplishSplash** (Bender et al., MIT) implements DFSPH and PCISPH with CUDA kernels for neighbour search, pressure solve, and anisotropic kernel computation (§58.2). Its surface reconstruction pipeline calls the GPU marching cubes of §65 on a density grid built with the Yu & Turk anisotropic splatting kernel.

**Open3D** (Intel/NVIDIA, MIT) provides a GPU tensor API covering point cloud k-NN, GPU convex hull (§106), CVT-based resampling (§20.4), FPFH descriptor computation (§89.2), and mesh repair (§22.1 non-manifold detection). Its CUDA backend is the most complete open GPU geometry library for the §58–§22 range.

**Manifold** (Knyszewski, Apache 2.0) implements exact, robust Boolean operations on manifold meshes and produces watertight output guaranteed to be manifold. It underpins the §22 repair discussion and is the recommended library for 3D printing pipeline repair.

**What is missing in §58–§22.** GPU CCD (§59) has no Vulkan library; bullet3's CCD is CPU-based with CUDA broadphase only. GPU radiosity (§81) has no modern GPU library; all production engines compute it offline on the CPU or bake it with ray tracing (§28). Screen-space reflections (§80) are universally engine-internal (Unreal TSR, Unity URP). Silhouette detection (§101) and suggestive contours (§101.4) have no GPU library. Spanning all nine tables (§1–§22), the Vulkan compute shader patterns remain the primary portable path for the majority of sections not served by CUDA-exclusive libraries.

**Table J — §33–§104 coverage**

| Library | Ver | GPU Backend | §33 VirtGeom | §82 PathTrace | §90 ICP | §102 Displace | §68 LevelSet | §103 SDF Font | §83 Atmo | §84 SSS | §74 Geo | §104 MolSurf | Best Use |
|---------|-----|-------------|:------------:|:-------------:|:-------:|:------------:|:------------:|:------------:|:--------:|:-------:|:-------:|:-----------:|----------|
| Nanite (Unreal 5) | UE5.4 | Vulkan/D3D12 | ✓ (§33 native) | — | — | ✓ (§102 mesh shader) | — | — | ✓ (§83 plugin) | — | — | — | Production virtual geometry |
| NVIDIA Falcor | 7.0 | Vulkan/D3D12 | — | ✓ (§82 full PT) | — | — | — | — | — | — | — | — | Research path tracing framework |
| Mitsuba 3 | 3.5 | CUDA/LLVM | — | ✓ (§82 MIS PT) | — | — | — | — | ✓ (§83 spectral) | ✓ (§84.3) | — | — | Differentiable physically-based |
| Open3D | 0.18 | CUDA/Tensor | — | — | ✓ (§90 GPU ICP) | — | — | — | — | — | ✓ (§74.2 LIDAR) | — | Point cloud & terrain processing |
| OpenVDB | 11.0 | CUDA/NanoVDB | — | — | — | — | ✓ (§68 sparse LS) | — | — | — | — | — | Sparse volumetric computation |
| msdfgen | 1.12 | CPU | — | — | — | — | — | ✓ (§103.3 MSDF) | — | — | — | — | Multi-channel SDF glyph baking |
| SkyAtmosphere (UE) | UE5.4 | Vulkan | — | — | — | — | — | — | ✓ (§83 LUT) | — | — | — | Production sky rendering |
| SSSS (Jiménez) | 2.0 | D3D11 (port) | — | — | — | — | — | — | — | ✓ (§84.1) | — | — | Separable screen-space skin SSS |
| PDAL | 2.7 | CPU | — | — | — | — | — | — | — | — | ✓ (§74 pipeline) | — | LIDAR/geospatial pipeline |
| APBS | 3.4 | CUDA | — | — | — | — | — | — | — | — | — | ✓ (§104.4 PB) | Molecular electrostatics solver |
| VMD | 1.9 | CUDA/OpenCL | — | — | — | — | — | — | — | — | — | ✓ (§104 SAS/SES) | Molecular visualization |

**Nanite** (Epic Games, built into Unreal Engine 5, proprietary) implements §33 end-to-end: offline cluster DAG construction (§33.1), GPU persistent-thread traversal (§33.2), software rasterisation via 64-bit atomic depth buffer (§33.3), and visibility buffer shading (§33.4). The Unreal sky atmosphere plugin implements §83 LUT-based scattering. Nanite's displacement (§102) is applied via mesh shader at cluster granularity.

**NVIDIA Falcor** (BSD-3-Clause) is a real-time rendering research framework implementing a full wavefront path tracer (§82): MIS direct lighting, shader sorting (§82.3), and SVGF denoising (§82.4). It uses Vulkan or D3D12 ray tracing pipelines and serves as the reference implementation for §82.

**OpenVDB / NanoVDB** (Academy Software Foundation, MPL-2.0) implements the VDB hierarchical sparse grid — the reference structure for §68 narrow-band level sets. NanoVDB provides a CUDA/Vulkan-compatible read-only view of the VDB tree enabling GPU advection (§68.3) and reinitialization (§68.2) on the GPU.

**What is missing in §33–§104.** GPU path tracing (§82) is dominated by CUDA/OptiX (Falcor, LuxCoreRender); Vulkan ray tracing pipeline equivalents exist (Vulkan-glTF-PBR, Kajiya) but lack the MIS+wavefront sorting of production PT. Atmospheric scattering (§83) is engine-internal in all production renderers; the Bruneton LUT approach (§83.1–96.2) is the only open, portable reference. Molecular surface (§104) GPU libraries are exclusively CUDA (VMD/APBS). Spanning all ten tables (§1–§104), the CUDA ecosystem dominates approximately forty-five sections; the Vulkan compute patterns remain the sole portable path for the remainder.

**Table K — §45–§107 coverage**

| Library | Ver | GPU Backend | §45 Soft FEM | §46 Cage Deform | §47 Lap Edit | §23 Del Refine | §60 Mink/PRM | §85 OIT | §86 Clipping | §107 Radix Sort | Best Use |
|---------|-----|-------------|:-------------:|:----------------:|:-------------:|:---------------:|:-------------:|:--------:|:-------------:|:---------------:|----------|
| projective-dynamics | research | CUDA | ✓ (§45.3–100.4) | — | — | — | — | — | — | — | PD soft body research |
| FEBio | 4.0 | CPU+CUDA | ✓ (§45.2 FEM) | — | — | — | — | — | — | — | Biomedical FEM |
| libigl | 2.5 | CPU | — | ✓ (§46 MVC) | ✓ (§47 ARAP) | — | — | — | — | — | Geometry processing |
| geometry-central | 1.0 | CPU | — | — | ✓ (§47.1 cotLap) | — | — | — | — | — | Differential geometry |
| TetGen | 1.6 | CPU | ✓ (§45.1 tets) | — | — | ✓ (§23 CDT) | — | — | — | — | Quality tet meshing |
| Triangle (Shewchuk) | 1.6 | CPU | — | — | — | ✓ (§23 Ruppert) | — | — | — | — | 2D quality meshing |
| FCL | 0.7 | CPU | — | — | — | — | ✓ (§60.2 GJK) | — | — | — | Robot collision/planning |
| OMPL | 1.6 | CPU | — | — | — | — | ✓ (§60.3 PRM/RRT) | — | — | — | Motion planning |
| OIT (McGuire) | 2013 | OpenGL/Vulkan | — | — | — | — | — | ✓ (§85.3 WBOIT) | — | — | OIT reference impl |
| Vulkan SDK samples | 1.3 | Vulkan | — | — | — | — | — | ✓ (§85.1 PPLL) | ✓ (§86.2 task) | — | Vulkan OIT + culling |
| VkRadixSort | 2023 | Vulkan | — | — | — | — | — | — | — | ✓ (§107 full sort) | Vulkan compute radix sort |
| CUB / Thrust | CUDA 12 | CUDA | — | — | — | — | — | — | — | ✓ (§107 reference) | CUDA sort primitive |

**VkRadixSort** (Wihlidal, MIT) implements the full 4-pass radix sort (§107.1–107.3) in Vulkan compute shaders, directly applicable to §31 LBVH Morton code sorting and §48 particle spatial hashing. It serves as the portable Vulkan equivalent of CUB's `DeviceRadixSort`. **CUB** (NVIDIA, BSD-3) is the authoritative CUDA implementation, used internally by Thrust, cuBLAS, and all CUDA geometry libraries in this chapter.

**TetGen** (Si, AGPL-3.0) implements constrained Delaunay tetrahedralization (§23 in 3D) and Ruppert-quality refinement. The §45.1 inside/outside GPU classify pass feeds TetGen; its output provides the tetrahedral mesh for §45.2 FEM and §45.3 projective dynamics.

**OMPL** (Rice/CMU, BSD-3) implements PRM (§60.3) and RRT (§60.4) with pluggable collision checkers. Connecting it to the GPU GJK batch checker (§60.2) via FCL's CUDA backend closes the gap between GPU broadphase and CPU planning.

**What is missing in §45–§107.** Soft body FEM (§45) has no Vulkan GPU library; the projective dynamics research code is CUDA-only. Cage deformation (§46) and Laplacian editing (§47) have CPU implementations in libigl/geometry-central but no GPU libraries. Constrained Delaunay refinement (§23) is CPU-only universally. Motion planning (§60) GPU acceleration is an active research area with no production library. OIT (§85) is widely engine-internal; the PPLL and WBOIT patterns are the only portable open references. Radix sort (§107) has VkRadixSort as the sole Vulkan open library; all other geometry libraries use CUDA/CUB internally.

**Overall library landscape summary (§1–§107).** Across all eleven tables, approximately fifty sections have production GPU library support (mostly CUDA). The remaining fifty-seven sections — spanning IK (§40), Laplacian processing (§19, §47), spectral methods (§38), constrained meshing (§23), motion planning (§60), atmospheric scattering (§83), molecular surfaces (§104), and others — rely on the Vulkan compute shader patterns presented in this chapter as the primary portable implementation path. This reflects the general state of the GPU geometry ecosystem: the CUDA library layer is mature and deep; the Vulkan/portable compute layer is the frontier, with the patterns in §1–§107 constituting a practical reference implementation guide.

### 108.12 Why No Comprehensive Vulkan/Slang Geometry Library Exists

The preceding eleven tables reveal a striking asymmetry: roughly half of the algorithm classes in this chapter have no portable (non-CUDA) library at all, and almost none of those that do have Vulkan compute as their primary backend. This is not accidental. Several reinforcing forces explain the gap, and understanding them is as useful to a systems engineer as knowing which library to reach for.

**CUDA's fifteen-year head start closed the question before Vulkan was ready.** CUDA launched in 2007; Thrust, CUB, cuBLAS, and cuSPARSE followed within the next three years. By the time Vulkan compute reached production maturity around 2019, the entire academic and production GPU geometry ecosystem had already crystallised around CUDA. Every SIGGRAPH paper with released code, every course note with GPU pseudocode, and every open-source geometry library assumed CUDA. Authors went where the momentum was, and the momentum compounded.

**No organisation has the right incentive.** NVIDIA built its CUDA library ecosystem as a deliberate platform lock-in investment — the libraries are strategic assets, not acts of altruism. No equivalent incentive exists for Vulkan. The Khronos Group standardises APIs; it does not ship algorithm implementations. AMD, Intel, and ARM each benefit from Vulkan's *existence* as a portable standard, but none benefits enough from a shared, cross-vendor compute geometry library to fund one unilaterally. The constituency that would benefit most — developers targeting hardware they do not control — is also the constituency with the least capital to invest.

**The required shader language did not exist until recently.** Writing a reusable portable geometry library in raw GLSL means maintaining one version per target (GLSL for Vulkan, HLSL for D3D12, MSL for Metal, CUDA C++ for NVIDIA). That tripling of maintenance burden alone is enough to deter most contributors. [Slang](https://shader-slang.com/) (Microsoft Research origins, open-sourced by NVIDIA in 2022–2023) is the first language capable of compiling the same source to SPIR-V, DXIL, PTX, and Metal Shading Language simultaneously. It is also the first GPU shading language with generics, interfaces, and module-level encapsulation sufficient to write library-grade code. The language prerequisite for a serious portable geometry library has existed for roughly two years — not long enough for a library ecosystem to form around it. [Source: He et al. "Slang: A System for Portable and High-Performance DSL Implementation", PLDI 2023]

**Vulkan's API surface is hostile to library authorship.** A CUDA geometry function requires approximately ten lines of setup before the first byte of algorithm code. An equivalent Vulkan compute dispatch requires pipeline objects, descriptor set layouts, descriptor pool allocation, render pass configuration (for graphics shaders), synchronisation barriers, and explicit memory allocation and layout transitions — typically 300–500 lines of boilerplate before the algorithm begins. A library must either carry that boilerplate inside every exported function (making the library enormous and hard to compose) or build an abstraction layer first (which is itself a substantial independent project). Both paths raise the barrier to contribution far above what a research group writing a paper-release codebase will accept.

**Driver fragmentation multiplies the validation burden.** A CUDA library ships against a single driver stack across homogeneous hardware. A Vulkan compute library must validate correctness and performance across AMD RDNA 2/3/4, NVIDIA Turing/Ampere/Ada/Blackwell, Intel Arc/Xe, ARM Mali-G series, Qualcomm Adreno, and MoltenVK on Apple Silicon — each with distinct driver bugs, subgroup sizes (4 to 128), memory models, cooperative matrix extension support, and occupancy characteristics. The quality-assurance surface is roughly eight times larger for the same algorithm, and most of those driver combinations generate bug reports only sporadically. A sustained engineering investment in cross-driver testing is required before a first release can be trusted — a cost that individual research groups cannot absorb.

**Dynamic GPU memory allocation is still painful in Vulkan.** Several algorithms in this chapter — BVH cavity expansion (§106), Delaunay refinement (§23.3), QuickHull horizon-edge collection (§106.3), PPLL fragment append (§85.1) — need to grow output buffers based on runtime-computed sizes. In CUDA, device-side `malloc` and `new` are supported directly inside kernels. In Vulkan compute, the idiomatic approach is to pre-allocate worst-case buffers (wasting memory), use atomic counters with a two-pass pattern (adding latency), or bounce back to the host for reallocation (adding synchronisation overhead). None of these alternatives packages cleanly as a library call with a simple size parameter, because the output size depends on data-dependent control flow that only resolves at runtime on the device.

**The users who need portability do not currently need heavy algorithms.** The applications that run sophisticated BVH construction, spectral mesh processing, or projective dynamics soft bodies are, in practice, running on NVIDIA workstation or datacenter GPUs — hardware where CUDA is already available and preferred. The users who genuinely require Vulkan portability — browser-based (WebGPU), mobile (ARM Mali, Adreno), Apple Silicon (via Metal/MoltenVK) — typically need terrain rendering, particle effects, and basic skinning, not Poisson surface reconstruction or acoustic ray tracing. The portability need and the algorithm complexity need exist in largely disjoint user populations.

**The most credible near-term path: Slang + wgpu.** Two developments have changed the calculus since 2022. First, Slang solves the language fragmentation problem: a single `.slang` source compiles portably to all major GPU targets. Second, [wgpu](https://wgpu.rs/) (Rust, used in Firefox's WebGPU implementation and adopted by several game engines) abstracts Vulkan, Metal, and D3D12 at a level that reduces the boilerplate barrier to a manageable size — comparable to writing a Vulkan application with a mature framework rather than raw API calls. A GPU geometry library written in Slang and exposing a wgpu compute interface would be the first genuine portable candidate. As of 2026 such a library does not exist, but the preconditions — language, abstraction layer, and demonstrated cross-vendor hardware base — have only recently all been satisfied simultaneously. The patterns in §1–§107 of this chapter are, in part, a pre-specification of what that library would contain.

---


## 109. Performance Reference

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


## 110. Integrations

- **Ch24 (Vulkan/EGL)** — Vulkan resource allocation patterns for SSBO-based subdivision and skinning buffers; pipeline barrier placement between compute passes.
- **Ch25 (GPU Compute)** — Compute shader dispatch setup, shared memory usage for marching cubes tile optimization, prefix-scan patterns for compaction, and radix sort for Morton-code-based LBVH (§24).
- **Ch127 (Mesh Shaders and VRS)** — Mesh shaders replace the tessellation pipeline for subdivision patches; the amplification shader's LOD selection (§10.4) uses cluster bounding spheres from the same meshlet data structures described in Ch127.
- **Ch133 (Vulkan Compute Queues)** — Skinning pre-pass, IK compute, and BVH rebuild should run on the async compute queue to overlap with graphics rendering; see Ch133 for synchronization between compute and graphics queues.
- **Ch135 (Vulkan Ray Tracing)** — The GPU-built LBVH (§24) feeds into `VkAccelerationStructureBuildGeometryInfoKHR` via `VK_ACCELERATION_STRUCTURE_BUILD_MODE_UPDATE_KHR` for dynamic geometry; Ch135 covers the full AS build and GLSL ray-tracing shader integration. The BLAS/TLAS pipeline in §75 of this chapter provides the geometry layer that Ch135's ray tracing shaders query.
- **Ch141 (Cooperative Matrices)** — QEF matrix accumulation in dual contouring (§3.3) and Jacobian construction in IK (§40.3) benefit from cooperative matrix operations when matrix sizes exceed the warp-level primitive dimensions. The MLP inference in §92 (PointNet, neural SDF) maps directly to cooperative matrix multiply-accumulate patterns.
- **Ch154 (GPU-Driven Rendering)** — GPU-driven pipelines require that geometry (subdivision patches, skinned meshes, LOD index buffers) be finalized in compute before indirect draw commands are issued; see Ch154's indirect dispatch patterns. The task-shader crowd rendering in §51.4 uses the same indirect draw compaction flow as Ch154's two-phase occlusion pass.
- **Ch204 (Shader Algorithm Catalog)** — Reference for quaternion arithmetic primitives (`quat_mul`, `quat_from_axis_angle`) used in CCD and DQS shaders; also covers RK4 integration primitives reused in §71.4 heightfield collision and §96.1 streamline tracing.
- **Ch209 (GPU Simulation Pipelines)** — The fluid (§48), deformable body (§49), rigid body (§50), fracture (§52), and crowd (§51) sections of this chapter provide the algorithm kernels; Ch209 covers how to assemble these kernels into a full simulation pipeline with shared memory layouts, double-buffering, and cross-queue synchronization. Hair strand simulation (§42) and level set evolution (§61) connect to the same pipeline architecture.
- **Ch210 (Terrain and Streaming)** — The CDLOD quadtree (§71.1), sparse clipmap (§71.3), and heightfield ray intersection (§71.4) in this chapter are the GPU geometry algorithms; Ch210 covers the full terrain system including heightmap streaming, virtual texture feedback, and integration with the Vulkan sparse binding API.
- **Ch135 (Vulkan Ray Tracing, lightmap baking)** — §28 uses `VK_KHR_ray_query` inline queries for per-texel AO baking; Ch135 covers the full RT pipeline, acceleration structure lifetime management, and shader binding table layout needed to build the bake pipeline.
- **Ch152 (Rust GPU)** — Differentiable rendering (§93) and geometric deep learning (§92) are active areas for GPU-accelerated training in Rust; the neural SDF inference kernel from §92.3 maps directly to rust-gpu compute shaders for Vulkan-native ML inference.
- **Ch154 (GPU-Driven Rendering, visibility)** — The software occlusion rasteriser (§29.2) and portal BFS (§29.3) feed into the two-phase occlusion pass (§26.4) described in Ch154; §29 provides the culling query algorithms, Ch154 provides the indirect draw compaction that acts on their results.
- **Ch133 (Vulkan Compute Queues, ocean)** — The ocean FFT pipeline (§73) dispatches four compute passes per frame (spectrum evolve, IFFT ×3 for height/Dx/Dz, Jacobian). These passes run on the async compute queue overlapping with shadow map rendering (§76); Ch133's multi-queue synchronisation patterns apply directly.
- **Ch24 (Vulkan/EGL, IBL)** — The PMREM compute dispatch (§77.2) and BRDF LUT bake (§77.3) write to cubemap array images and 2D images respectively. Ch24 covers the image layout transitions (`VK_IMAGE_LAYOUT_GENERAL` during compute write, `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL` during fragment read) required for these preprocessing resources.
- **Ch210 (Terrain and Streaming, progressive mesh)** — Progressive mesh streaming (§15) and terrain clipmap streaming (§71.3) use the same pattern: a ring-buffer SSBO refilled by a background transfer queue while the graphics queue renders from already-uploaded data. Ch210 covers the transfer queue + sparse binding setup.

