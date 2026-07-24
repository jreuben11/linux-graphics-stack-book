# Chapter 209: GPU Geometry Algorithms — Spatial Structures, Differential Geometry, and Animation (Part XXIX)

*Part XXIX — Graphics Algorithms*

**Audiences:** Graphics application developers implementing acceleration structures, character animation, or geometry analysis pipelines on Linux using Vulkan or OpenGL compute.

This chapter is the second volume of the GPU Geometry Algorithms series. It covers three domains that bridge static geometry and dynamic simulation: Category III (Spatial Data Structures and Visibility) covers BVH construction and traversal, octrees, k-d trees, sparse voxel DAGs, screen-space geometry techniques, and GPU-driven virtual geometry (Nanite-style cluster LOD); Category IV (Differential Geometry and Analysis) covers discrete curvature, geodesics via the heat method, spectral mesh processing, and shape morphing with functional maps; Category V (Animation, Skinning, and Deformation) covers skeletal LBS/DQS skinning, IK solvers, blend shapes, hair strand geometry, secondary dynamics, mesh retargeting, soft-body FEM, cage-based deformation, and Laplacian mesh editing.

## Table of Contents

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


## Integrations

**Ch208** (this series) covers the surface representations (subdivision, NURBS, metaballs) that the spatial structures here index and the mesh processing output that animation deforms. **Ch210** (this series) covers the physics simulation that drives secondary animation dynamics catalogued in Cat V. **Ch135** (Vulkan Ray Tracing) covers the TLAS/BLAS acceleration structure API that wraps the BVH representations in Cat III. **Ch133** (Vulkan Compute Queues) covers async compute scheduling for BVH refitting and skinning dispatches. **Ch154** (GPU-Driven Rendering) covers the cluster LOD and GPU-driven indirect draw pipeline that Cat III virtual geometry feeds.
