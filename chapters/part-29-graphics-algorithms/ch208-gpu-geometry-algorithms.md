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


---

## Algorithm Selection Guide

This chapter spans 107 algorithms across 13 categories. Use this guide to identify the right
starting point for your problem without reading all 15,000 lines.

### Quick Reference: By Problem Type

| Your starting point / goal | Go to |
|---|---|
| Smooth curved surface from control points (CAD, film VFX) | §I — Surface Representation |
| Triangle mesh needs cleaning, LOD, or UV layout | §II — Mesh Processing |
| Accelerate ray traversal or cull draw calls before the GPU sees them | §III — Spatial Data Structures |
| Measure curvature, geodesic distance, or shape similarity | §IV — Differential Geometry |
| Deform a character mesh — skeleton, IK, blendshapes | §V — Animation & Deformation |
| Simulate fluids, cloth, rigid bodies, or fracture | §VI — Physics & Simulation |
| Represent geometry as a scalar field, reconstruct a surface from sensors | §VII — SDF & Volumetric |
| Generate or render terrain, ocean, particles, or procedural environments | §VIII — Terrain & Procedural |
| Ray-traced shadows, reflections, GI, or atmospheric effects | §IX — Ray Tracing & Optical |
| Input is a point cloud (LiDAR, depth sensor, SfM) | §X — Point Cloud |
| Novel-view synthesis, learned shape representation, differentiable rendering | §XI — Neural Geometry |
| Domain-specific: acoustics, molecular surfaces, VR reprojection, SDF fonts | §XII — Specialized |
| Need a low-level parallel primitive: sort, sample, convex hull | §XIII — GPU Primitives |

### Quick Reference: By Runtime Constraint

| Constraint | Preferred approaches | Avoid |
|---|---|---|
| Real-time (< 16 ms frame) | §III BVH/GPU-driven, §V LBS/DQS, §VII SDF collision, §IX SSR/shadow maps | §IV spectral, §VI FEM, §XI NeRF inference without baking |
| Interactive offline (seconds) | §VI SPH/PBD, §VII TSDF fusion, §X ICP registration | §XI full NeRF training |
| Offline preprocess (minutes–hours) | §II UV parameterization, §IV geodesics/spectral, §VIII erosion, §IX path tracing bake | Real-time use of offline results without caching |
| GPU memory < 1 GB | §III SVDAG (voxels), §VII narrow-band level sets, §VIII clipmap (stream tiles) | §XI full 3DGS scene (can be 500 MB–4 GB) |
| Must run on integrated GPU / no discrete GPU | §V PBD (CPU-fallback friendly), §II meshopt compression | §VI Eulerian fluids, §XI 3DGS rasterizer |

---

### Choosing Within Category I — Surface Representation

**Use subdivision surfaces (§1)** when: the input is a low-polygon control cage (from a DCC
tool or CAD package) and you need a smooth, watertight surface with guaranteed tangent
continuity. Catmull-Clark for quad-dominant meshes (film/VFX), Loop for triangle meshes
(games/real-time). Use OpenSubdiv; do not roll your own stencil evaluator.

**Use NURBS / B-splines (§2)** when: geometry originates from a CAD system (STEP/IGES) and
exact representation matters — machining tolerances, engineering analysis. Tessellate to a mesh
once and cache; do not evaluate NURBS per frame in a rendering loop.

**Use metaballs / implicit SDF (§3, §8)** when: shapes must blend, merge, or split dynamically
(fluid surface, clay sculpting, creature morphing). If the shape is static, bake to a mesh
instead — implicit evaluation per frame is expensive. Prefer §8 (SDF blending via `smin`) over
classic metaball field functions for GPU efficiency.

**Use spline curve tessellation (§4)** when: geometry is inherently 1D (roads, cables, hair
guide curves, animation paths). Use adaptive tessellation matched to screen-space chord length;
fixed subdivision wastes triangles at distance.

**Use 2D vector graphics on GPU (§5, §6)** when: UI, text (prefer §103 SDF fonts for small
sizes), SVG icons, or CAD line drawings need resolution-independent rendering. Do not use for
filled polygon rendering at scale — a rasterized texture atlas is faster above ~10,000 paths.

**Use GPU CSG / boolean operations (§7)** sparingly — GPU boolean mesh operations are expensive
and numerically fragile. Prefer SDF-based CSG (§3, §8) for dynamic operations, or precompute
boolean results offline with a robust library (Cork, CGAL) and upload the result mesh.

---

### Choosing Within Category II — Mesh Processing

**Simplification (§10)** vs **remeshing (§11, §20)**: use simplification to generate LOD
levels of an existing mesh (preserving topology, just reducing triangles). Use remeshing when
you need uniform triangle quality for simulation (FEM, cloth) or when the input mesh is too
irregular. Meshoptimizer covers both for game pipelines.

**UV parameterization (§17)** vs **atlas packing (§12)**: these are sequential. Parameterize
first (each chart gets UV coordinates), then pack charts into an atlas. Use xatlas for
automated pipelines; manual seam placement only for hero assets. Always run after final
topology changes — UV seams survive neither remeshing nor simplification.

**Delaunay triangulation (§18, §23)**: use when building quality meshes for FEM or when
triangulating 2D domains (terrain heightmaps, navmesh). Not needed for rendering meshes — the
GPU does not care about Delaunay quality.

**Voxelization (§13)**: use when you need a solid occupancy representation (collision, SDF
initialization, ambient occlusion baking). Surface voxelization is fast; solid voxelization
(conservative) is 5–10× slower and only necessary when interior/exterior matters.

**Compression (§16, §15)**: always compress before network transmission or disk storage. Use
meshopt for game/engine pipelines (best decode speed), Draco for WebGL/glTF targets, and
`KTX2` + `Basis Universal` for textures bundled with geometry. Do not compress real-time
geometry in flight — decompress once, cache the GPU buffer.

**Mesh repair (§22)**: run before any simulation or boolean operation. A non-watertight mesh
will produce incorrect SDF bakes, broken FEM meshes, and wrong voxelizations. Use MeshFix or
manifold as a preprocessing step.

---

### Choosing Within Category III — Spatial Data Structures

**BVH (§24, §31)** is the default acceleration structure for ray traversal, broad-phase
collision detection, and frustum culling. Prefer Vulkan hardware BVH (BLAS/TLAS) when
ray-tracing extensions are available; fall back to LBVH (§24 radix-sort construction) in
compute for software traversal. Use §31 (SAH-guided BVH) only when build time is not a
bottleneck and traversal quality matters more — it builds 3–5× slower than LBVH but produces
30–40% fewer traversal steps.

**Spatial hashing** (not covered as a standalone section but mentioned in §27): prefer over
BVH for particle systems with uniform density. BVH rebuild per frame is expensive when
particles move; a GPU hash grid rebuilds in O(N) with radix sort (§107).

**SVDAG (§30)**: use only for static, very large voxel scenes (city-scale, terrain) where
memory is the primary constraint. Construction is offline; not suitable for dynamic geometry.

**GPU-driven indirect rendering (§26, §33)**: mandatory for scenes with > 5,000 unique draw
calls or > 50,000 meshlets. The CPU cannot submit that many draw calls at 60 Hz. Use
`vkCmdDrawMeshTasksIndirectEXT` (mesh shaders) or `vkCmdDrawIndexedIndirectCount` (traditional
pipeline) depending on hardware support.

**Screen-space techniques (§25)**: use for effects that do not require world-space geometry —
SSAO, SSR, screen-space shadows. They fail at silhouettes and for off-screen occluders; always
provide a fallback (precomputed AO, reflection cubemaps).

**Precomputed visibility (§28)**: worthwhile for indoor/portal-based scenes (rooms, caves,
buildings) where the view graph is known at asset time. Not applicable for open-world terrain
or procedurally generated environments.

---

### Choosing Within Category IV — Differential Geometry

All algorithms in this category are **offline preprocessing tools**, not real-time algorithms.
Run them as asset pipeline steps and cache results.

**Geodesic distance (§37, §34)**: use the heat method (§37) on GPU — it reduces to two sparse
linear solves and is 10–100× faster than exact algorithms for the accuracy levels needed in
graphics. Use exact geodesics (Dijkstra on edges) only when topological accuracy is required
(e.g., seam placement for UV parameterization).

**Laplacian / cotangent weights (§34 — referenced in curvature, smoothing)**: the cotangent
Laplacian is the correct discrete operator for most geometry processing tasks. Do not use
uniform (graph) Laplacian for geometry — it produces area-dependent results.

**Spectral methods (§35, §38)**: use for shape correspondence, partial matching, and
dimensionality reduction of shape collections. Do not use for per-frame processing — eigensolver
cost is O(kN) with k eigenvectors and is not GPU-friendly for large k.

**Functional maps (§36)**: prefer over point-to-point correspondence when shapes have
significant deformation. Output is a compact matrix; convert to a dense correspondence map only
when needed.

---

### Choosing Within Category V — Animation & Deformation

**Linear blend skinning (§39)** for speed; **dual-quaternion skinning (§39)** for quality.
DQS eliminates the "candy-wrapper" collapse artefact at joints twisted > 45°. The GPU cost
difference is negligible — prefer DQS by default for any character that will be seen up close.
Add corrective blendshapes on top for hero characters.

**IK (§40)**: FABRIK converges faster than CCD for most game chains and requires no matrix
inversion. Use Jacobian (pseudo-inverse or transpose) only when you need precise end-effector
orientation and have < 10 joints in the chain. For full-body IK, use a commercial solver
(PhysX, IK Rig) rather than implementing from scratch.

**Cage-based deformation (§46)**: best when an artist needs to push/pull the cage and see
smooth deformation. More intuitive than blendshapes for large-scale shape changes. Use Green
Coordinates rather than Mean Value Coordinates when the cage is not guaranteed to enclose the
mesh.

**Projective dynamics / PBD (§45)**: the standard for game soft bodies and cloth. Fast,
stable, and easy to control. Prefer over FEM for real-time; use FEM only when physical
accuracy is required (medical simulation, engineering).

**Hair / strand geometry (§42)**: render with line strips or ribbon geometry (camera-facing
quads). Simulate with position-based dynamics (PBD) or XPBD for interactive rates. Do not use
full FEM for hair — it is unnecessarily expensive and not more accurate for the stiffness range
of hair.

---

### Choosing Within Category VI — Physics & Simulation

**SPH (§48)** vs **Eulerian grid (§48)**: SPH is Lagrangian (particles follow the fluid) and
is easier to implement on GPU, handles free surfaces well, and scales to ~10M particles at
interactive rates. Eulerian (grid-based pressure projection) handles turbulence and smoke
better but requires a velocity field grid that can consume large amounts of GPU memory. Use SPH
for water/liquid effects; use Eulerian for smoke and fire.

**PBD / XPBD (§45, §54)** vs **FEM (§45)**: PBD is the right choice for games and VFX —
stable at large timesteps, tunable stiffness, GPU-parallelizable per constraint island.
FEM with implicit time integration is more physically accurate but requires a sparse linear
solver per frame, which is difficult to parallelize efficiently. Use FEM for offline simulation
or when engineering accuracy is required.

**Rigid body broad phase**: spatial hashing for uniform object distributions, BVH for highly
varied object sizes. Rebuild every frame with radix sort (§107) rather than updating
incrementally — incremental BVH updates are rarely faster than full rebuild on GPU.

**CCD (§59)**: only use when objects move more than their own size in a single timestep.
For most game scenarios (objects < 10 m/s, timesteps ~16 ms), discrete collision detection
is sufficient and 5–10× cheaper. Enable CCD selectively for fast-moving thin objects (bullets,
blades, cloth).

**DEM (§56)**: specifically for granular materials — sand, gravel, powder. Do not use for
general rigid bodies; per-particle DEM with 10,000+ grains is expensive. Use instanced
rendering with a pre-baked simulation for decorative granular effects.

**Fracture (§52)**: pre-fracture assets with Voronoi decomposition at asset-creation time
(Blender, Houdini). Runtime fracture requires re-triangulating crack surfaces in a compute
shader and is expensive. Use runtime fracture only for hero destruction moments.

---

### Choosing Within Category VII — SDF & Volumetric

**Level sets (§61)** when the surface topology changes (bubbles merging, flame fronts).
Level sets handle topology change for free; meshes do not. The cost is a 3D voxel grid that
must be advected and re-extracted every frame. Use a sparse narrow-band representation (§68)
rather than a dense grid — typically only ~5% of voxels are near the surface.

**TSDF fusion (§63)**: the standard algorithm for depth-sensor reconstruction (Kinect,
RealSense, structured light). Use KinectFusion-style TSDF for small scenes (room-scale);
use hash-based sparse TSDF for larger scenes. Output is an SDF grid, not a mesh — extract
with marching cubes (§65) when a mesh is needed downstream.

**Marching cubes (§65)** vs **dual contouring**: use marching cubes for speed and simplicity
— it is faster and trivially parallelizable. Use dual contouring when sharp features (edges,
corners) must be preserved (CAD surfaces, architecture). For terrain heightmaps, neither is
needed — generate the mesh directly from the heightfield.

**SDF collision detection (§64)**: ideal for soft body vs rigid environment (character against
terrain, cloth against a character body). Not well-suited for rigid body vs rigid body at high
accuracy — GJK/EPA (§60) is more precise for convex shapes. Combine both: SDF for
broad/medium queries, GJK for narrow-phase.

**Volumetric ray marching (§67)**: for clouds, fire, and participating media. Do not use for
opaque solid geometry — rasterization is 10–100× faster. Always ray-march at half or quarter
resolution and upscale with a depth-aware bilateral filter.

---

### Choosing Within Category VIII — Terrain & Procedural

**Clipmap LOD (§71)** vs **quadtree LOD**: clipmaps give smooth, predictable LOD transitions
at low CPU overhead (just update the ring buffers). Quadtrees give finer LOD control and are
better for non-uniform terrain (dense urban vs flat plains) but require more CPU work per
frame. Use clipmaps for open-world terrain; quadtrees for scenes where you can afford the
CPU budget.

**Hydraulic erosion (§57)**: run once offline. The GPU erosion simulation (particle-based or
grid-based) takes seconds to minutes on a high-res heightmap. Never run at runtime — cache the
eroded heightmap as a texture.

**FFT ocean (§73)**: use for open ocean and large bodies of water. The Philips/JONSWAP
spectrum produces physically plausible wave statistics. Do not use for enclosed bodies of water
(swimming pools, rivers) — a simpler sine-wave sum or gerstner wave is adequate and cheaper.

**Particle systems (§72)**: GPU particle simulation via compute shaders handles millions of
particles in real-time. Separate simulation (position update) from rendering (billboard
generation) — do not store per-particle geometry in the simulation buffer. For VFX effects,
use a combination of particle forces and the FFT advection field from a fluid simulation (§48).

---

### Choosing Within Category IX — Ray Tracing & Optical

**When hardware RT is available (Vulkan RT extensions)**: always use it for primary ray
traversal. The TLAS/BLAS pipeline is 5–20× faster than software BVH traversal in compute.

**Shadow maps** vs **ray-traced shadows**: use shadow maps (cascaded for sun, cube maps for
point lights) as the default. Enable ray-traced shadows only for hero shadow-casters where
softness and contact hardening are critical. PCSS shadow maps approximate soft shadows well
enough for most scenes.

**Screen-space reflections (§80)** vs **ray-traced reflections**: SSR is the primary
reflection method — fast, plausible for low-roughness surfaces, runs in a single compute pass.
Fail gracefully to a reflection cubemap or IBL (§77) for off-screen geometry. Use ray-traced
reflections only for mirror-like surfaces or architectural visualization where accuracy is
required.

**Path tracing (§82)**: real-time path tracing requires ReSTIR (§82) for many-light
rendering and SVGF/NRD denoising to make undersampled output acceptable. Offline baking
(lightmaps, irradiance volumes) is still the right choice for most shipped games — path
tracing for baking is well-established; real-time path tracing requires very recent hardware.

**IBL preprocessing (§77)**: always precompute the split-sum DFG LUT and the pre-filtered
environment map mip chain offline. Cache as DDS/KTX2 textures. Do not evaluate the full
environment integral per frame.

**Subsurface scattering (§84)**: use the screen-space SSS approximation (scatter in a
separable blur along screen-space normals) for real-time skin rendering — it matches the
dipole/BSSRDF result well enough for most viewing distances. Use the full volumetric dipole
only for offline path-traced character renders.

---

### Choosing Within Category X — Point Cloud

**ICP (§90)**: ICP converges only within the basin of attraction of the correct alignment —
typically when the initial misalignment is < 30°. Always provide a coarse initial alignment
first (FPFH feature matching §89, manual landmark alignment, or IMU data). Use point-to-plane
ICP rather than point-to-point — it converges ~3× faster for smooth surfaces.

**SfM / MVS (§88)**: requires dense, overlapping image coverage (> 60% overlap between
adjacent images). Does not work in texture-less or reflective environments. Use a commercial
or mature open-source pipeline (COLMAP, OpenMVG) rather than implementing from scratch — the
robust estimation, bundle adjustment, and outlier rejection are the hard parts.

**Feature descriptors (§89)**: FPFH is the GPU-friendly choice for coarse registration.
For learned descriptors (3DMatch, FCGF), use only if you have a trained model that matches
your point cloud density and domain — generalization across sensor types and scene scales is
poor.

---

### Choosing Within Category XI — Neural Geometry

**3DGS (§91, §94)** vs **NeRF (§95)**: 3DGS renders faster at inference time (real-time at
1080p on RTX 3080-class hardware) and produces sharper results for textured surfaces. NeRF
(and its successors — Instant-NGP, Zip-NeRF) produces better results in fine detail (thin
structures, semi-transparent materials) and is more compact in memory for large scenes. Use
3DGS when render-time performance is critical; use NeRF-family methods when quality and
memory efficiency are critical.

**Geometric deep learning (§92)**: use for classification, segmentation, and correspondence
tasks on shape collections — not for rendering or real-time geometry processing. Requires a
trained model; training time is hours to days.

**Differentiable rendering (§93)**: use for inverse problems — fitting geometry, materials,
or camera parameters from images. Not a rendering algorithm; the differentiable rasterizer is
used to compute gradients, not to produce the final image.

---

### Choosing Within Category XII — Specialized

Each section addresses a narrow domain. Consult the section directly when building:
- **§96** scientific visualization: isosurface rendering of scalar fields (CT, CFD, finite element output)
- **§99** acoustic simulation: sound propagation geometry, diffraction, reverb computation via ray tracing
- **§104** molecular surfaces: Connolly/SAS surfaces for docking, electrostatics visualization
- **§100** VR/XR: reprojection and foveated rendering for latency reduction
- **§103** SDF font rendering: crisp text at all scales using multi-channel SDF atlases
- **§102** micro-polygon displacement: film-quality tessellation displacement for hero assets
- **§101** silhouette detection: NPR rendering outlines and shadow volume edge lists
- **§98** procedural texture synthesis: tileable detail textures generated in compute shaders
- **§97** decal projection: decal geometry via UV-space projection for surface detail

---

### Choosing Within Category XIII — GPU Primitives

**Radix sort (§107)** vs **bitonic sort**: always prefer radix sort for N > 100,000 keys —
it is O(N) vs O(N log² N) and is 3–10× faster in practice on GPU. Bitonic sort is simpler to
implement and competitive for N < 10,000 (fits in shared memory). For BVH construction,
particle simulation step ordering, and OIT depth sorting, radix sort is the right choice.

**Poisson disk sampling (§105)**: use for geometry distribution (foliage, stones, debris) and
Monte Carlo sample generation. Use blue noise specifically for ordered dithering and
screen-space sample patterns — blue noise has better high-frequency spectral properties than
Poisson disk for 2D screen-space use.

**Convex hull (§106)**: needed as a preprocessing step for GJK collision detection (§60), for
computing inertia tensors for rigid bodies (§50), and as a simplification of complex geometry
for broad-phase queries. Compute offline when the mesh is static; only recompute at runtime
for dynamically deforming geometry where the convex hull changes significantly.

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


## Integrations

**Ch209** (this series) covers spatial data structures, differential geometry, and animation that build on the surface and mesh representations established here. **Ch204** (Shader Algorithm Catalog — Rendering, Lighting, Shadows) covers the rendering pipeline that consumes the geometry produced by subdivision, NURBS, and metaball evaluation. **Ch135** (Vulkan Ray Tracing) covers the acceleration structure API — BVH construction and traversal — that wraps the mesh output from Cat II for ray tracing queries. **Ch127** (Mesh Shaders and VRS) covers the meshlet pipeline that consumes the simplified, remeshed, and compressed geometry from Cat II. **Ch154** (GPU-Driven Rendering) covers the indirect draw pipeline that drives LOD selection over the mesh representations catalogued here.
