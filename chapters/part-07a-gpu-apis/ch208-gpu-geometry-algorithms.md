# Chapter 208: GPU Geometry Algorithms — Subdivision, Implicit Surfaces, Skinning, and IK

**Audiences:** Graphics application developers implementing character animation, procedural geometry, or physics-driven deformation pipelines on Linux using Vulkan or OpenGL compute.

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
3. [Metaballs and Implicit Surfaces](#3-metaballs-and-implicit-surfaces)
   - 3.1 [Field Functions](#31-field-functions)
   - 3.2 [Marching Cubes on the GPU](#32-marching-cubes-on-the-gpu)
   - 3.3 [Dual Contouring with QEF Minimization](#33-dual-contouring-with-qef-minimization)
   - 3.4 [SDF Construction: Jump Flooding on the GPU](#34-sdf-construction-jump-flooding-on-the-gpu)
   - 3.5 [Procedural Noise Density Fields](#35-procedural-noise-density-fields)
   - 3.6 [NanoVDB: Sparse GPU Volumes](#36-nanovdb-sparse-gpu-volumes)
4. [Skeletal Animation: Skinning](#4-skeletal-animation-skinning)
   - 4.1 [Linear Blend Skinning (LBS)](#41-linear-blend-skinning-lbs)
   - 4.2 [Dual Quaternion Skinning (DQS)](#42-dual-quaternion-skinning-dqs)
   - 4.3 [GPU Compute Skinning Pre-Pass](#43-gpu-compute-skinning-pre-pass)
   - 4.4 [Blend Shapes (Morph Targets)](#44-blend-shapes-morph-targets)
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
8. [GPU BVH Construction](#8-gpu-bvh-construction)
   - 8.1 [Morton Codes and LBVH](#81-morton-codes-and-lbvh)
   - 8.2 [LBVH Tree Construction (Karras 2012)](#82-lbvh-tree-construction-karras-2012)
   - 8.3 [AABB Propagation](#83-aabb-propagation)
9. [Library Landscape](#9-library-landscape)
10. [Performance Reference](#10-performance-reference)
11. [Integrations](#integrations)

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

## 4. Skeletal Animation: Skinning

### 4.1 Linear Blend Skinning (LBS)

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

### 4.2 Dual Quaternion Skinning (DQS)

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

### 4.3 GPU Compute Skinning Pre-Pass

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

### 4.4 Blend Shapes (Morph Targets)

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

---

## 9. Library Landscape

| Library | Version | GPU Backend | Subdivision | Skinning | Splines | NURBS | LOD/BVH | Best Use |
|---|---|---|---|---|---|---|---|---|
| **OpenSubdiv** | 3.6.0 | GL compute, CUDA, Metal | Yes (CC, Loop, Bilinear) | No | No | No | No | Production subdivision in VFX/games |
| **CGAL** | 6.1 | None (CPU + TBB) | Yes (CC, Loop, Doo-Sabin) | No | No | No | No | CPU preprocessing, export to GPU |
| **GeometricTools** | 6.x | None | Yes (CC, Doo-Sabin) | No | Yes | Yes | No | CPU reference for offline pipelines |
| **libigl** | 2.5.0 | None (CPU + Eigen) | No | Yes (LBS, DQS) | No | No | No | Automatic skinning weights, ARAP |
| **OpenCASCADE** | 8.0.0p1 | None (OpenGL vis only) | No | No | No | Yes (BRep) | No | CAD NURBS → tessellation → VkBuffer |
| **meshoptimizer** | 0.22 | None (CPU output → GPU) | No | No | No | No | LOD simplify | Production LOD chain and mesh optimization |
| **NanoVDB** | OpenVDB 9.x | CUDA / Vulkan SSBO | No | No | No | No | No | Sparse volume GPU rendering and sphere tracing |

**OpenSubdiv** ([source](https://github.com/PixarAnimationStudios/OpenSubdiv)) is the right choice for any production subdivision pipeline. The lack of a Vulkan evaluator requires the custom SSBO stencil approach described in §1.5.

**CGAL** ([source](https://www.cgal.org), [subdivision](https://doc.cgal.org/latest/Subdivision_method_3/index.html)) provides `CGAL::Subdivision_method_3::CatmullClark_subdivision()` and `Loop_subdivision()` as single-call in-place operations on a `Surface_mesh`:

```cpp
#include <CGAL/subdivision_method_3.h>
namespace Sub = CGAL::Subdivision_method_3;
Sub::CatmullClark_subdivision(mesh,
    CGAL::parameters::number_of_iterations(2));
```

**libigl** ([source](https://github.com/libigl/libigl)) provides `igl::lbs_matrix()` and `igl::dqs()` for computing LBS and DQS skinning, and `igl::bbw()` for computing bounded biharmonic weights (automatic skinning weight painting). All computations are CPU-side; results are uploaded to the GPU as vertex buffers.

**GeometricTools** ([source](https://github.com/davideberly/GeometricTools), Boost license) provides `GTL::NURBSSurface<float, 3>` and `GTL::CatmullClarkSurface` as header-only CPU implementations. Use for offline asset preprocessing: evaluate on the CPU, upload to Vulkan vertex buffers via staging.

**meshoptimizer** ([source](https://github.com/zeux/meshoptimizer), MIT) provides `meshopt_simplify()` (QEM edge collapse), `meshopt_simplifyWithAttributes()` (attribute-preserving QEM), `meshopt_optimizeVertexCache()`, and `meshopt_optimizeOverdraw()`. Generate the full LOD chain offline (one call per LOD level), upload all LOD index buffers to a single `VkBuffer`, and select the active LOD per draw via `firstIndex`/`indexCount` offsets.

**NanoVDB** ([source](https://github.com/AcademySoftwareFoundation/openvdb), Apache 2.0, included in OpenVDB 9.0+) converts OpenVDB trees to flat GPU buffers via `nanovdb::openToNanoVDB()`. The companion GLSL header (`nanovdb/util/NanoVDB.glsl`) provides tree-descent accessors. Requires `VK_EXT_scalar_block_layout` for correct `std430` packing of the NanoVDB node arrays.

---

## 10. Performance Reference

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

## Integrations

- **Ch24 (Vulkan/EGL)** — Vulkan resource allocation patterns for SSBO-based subdivision and skinning buffers; pipeline barrier placement between compute passes.
- **Ch25 (GPU Compute)** — Compute shader dispatch setup, shared memory usage for marching cubes tile optimization, prefix-scan patterns for compaction, and radix sort for Morton-code-based LBVH (§8).
- **Ch127 (Mesh Shaders and VRS)** — Mesh shaders replace the tessellation pipeline for subdivision patches; the amplification shader's LOD selection (§7.4) uses cluster bounding spheres from the same meshlet data structures described in Ch127.
- **Ch133 (Vulkan Compute Queues)** — Skinning pre-pass, IK compute, and BVH rebuild should run on the async compute queue to overlap with graphics rendering; see Ch133 for synchronization between compute and graphics queues.
- **Ch135 (Vulkan Ray Tracing)** — The GPU-built LBVH (§8) feeds into `VkAccelerationStructureBuildGeometryInfoKHR` via `VK_ACCELERATION_STRUCTURE_BUILD_MODE_UPDATE_KHR` for dynamic geometry; Ch135 covers the full AS build and GLSL ray-tracing shader integration.
- **Ch141 (Cooperative Matrices)** — QEF matrix accumulation in dual contouring (§3.3) and Jacobian construction in IK (§5.3) benefit from cooperative matrix operations when matrix sizes exceed the warp-level primitive dimensions.
- **Ch154 (GPU-Driven Rendering)** — GPU-driven pipelines require that geometry (subdivision patches, skinned meshes, LOD index buffers) be finalized in compute before indirect draw commands are issued; see Ch154's indirect dispatch patterns.
- **Ch204 (Shader Algorithm Catalog)** — Reference for quaternion arithmetic primitives (`quat_mul`, `quat_from_axis_angle`) used in CCD and DQS shaders.
