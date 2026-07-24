# Chapter 222: Computational Geometry Algorithms on GPU

**Audiences:** Graphics application developers implementing UI hit-testing, vector graphics, CAD kernels, or game collision systems; systems developers building spatial query infrastructure; computational geometry researchers porting classical algorithms to GPU.

**Relationship to Ch208–212.** The GPU Geometry Algorithms series (Ch208–Ch212) covers 3D geometry: subdivision surfaces, mesh processing, BVH for ray tracing, physics simulation, and Voronoi briefly via jump flooding. This chapter covers the complementary domain of *exact and approximate 2D and 3D computational geometry algorithms* that those chapters do not treat in depth: polygon Boolean operations, convex hull construction, comprehensive geometric intersection suites, approximate nearest-neighbour search in high dimensions, and combinatorial geometry (arrangements, point location, visibility polygons, robust predicates).

**CPU vs. GPU split.** Many classical computational geometry algorithms are inherently sequential: they trace linked-list topology, perform pointer-chasing on doubly-connected edge lists, or exhibit data-dependent branching that collapses SIMT lanes. The GPU excels at the embarrassingly parallel *query layer* — testing millions of point-in-polygon pairs, running N×M intersection tests, or evaluating support functions across a fleet of convex bodies in parallel — while exact combinatorial topology (arrangement construction, constrained triangulation, Boolean graph traversal) typically runs on the CPU and uploads its result as Vulkan buffers. Sections below note which components run where and show the GPU upload pattern alongside any CPU library API.

---

## Table of Contents

1. [GPU Computational Geometry Overview](#1-gpu-computational-geometry-overview)
2. [Convex Hull on GPU](#2-convex-hull-on-gpu)
3. [2D Polygon Boolean Operations](#3-2d-polygon-boolean-operations)
4. [2D and 3D Triangulation](#4-2d-and-3d-triangulation)
5. [Point Location and Geometric Queries](#5-point-location-and-geometric-queries)
6. [Geometric Intersection Testing](#6-geometric-intersection-testing)
7. [Voronoi Diagrams and Delaunay](#7-voronoi-diagrams-and-delaunay)
8. [Approximate Nearest Neighbour Search](#8-approximate-nearest-neighbour-search)
9. [Minkowski Sums and Configuration Space](#9-minkowski-sums-and-configuration-space)
10. [Geometric Arrangements and Sweep Line](#10-geometric-arrangements-and-sweep-line)
11. [Visibility and Line-of-Sight](#11-visibility-and-line-of-sight)
12. [Robust Geometric Predicates](#12-robust-geometric-predicates)
- [Integrations](#integrations)

---

## 1. GPU Computational Geometry Overview

Computational geometry algorithms decompose into two broad classes that respond very differently to GPU parallelism.

**Embarrassingly parallel operations** — distance computations, primitive intersection tests, point-classification queries, and support-function evaluations — map directly onto SIMT: each thread handles one (query, primitive) pair with no communication. An N×M ray-against-triangle batch, a winding-number test for a million query points, or a k-NN distance computation over a fixed dataset all achieve near-linear GPU scaling.

**Topology-sensitive operations** — incremental triangulation, arrangement sweep, half-edge traversal, union-find across merged polygon boundaries — require dynamic graph construction with pointer chasing and irregular memory access. These collapse SIMT lanes and typically run faster on a few CPU cores than on thousands of GPU threads. The canonical GPU response is to decompose the problem: run the topology-sensitive phase on the CPU, then dispatch the resulting data-parallel subproblems to the GPU.

**Randomisation and approximation** restore GPU efficiency for problems that would otherwise be sequential. Randomised incremental algorithms (Bowyer-Watson Delaunay, randomised quickhull, randomised half-plane intersection) restructure computation into parallel batches of independent subproblems. Approximate methods (jump flooding for Voronoi, product quantisation for nearest neighbour, GJK with fixed iteration budgets) trade a bounded error for GPU-friendly data parallelism.

**Output-sensitive complexity** is a recurring challenge. Algorithms like convex hull and arrangement construction have output sizes ranging from O(1) to O(n²). GPU compaction via prefix scan (`thrust::exclusive_scan` / `vkCmdFillBuffer` + compute shader) handles variable-length output: each thread writes a 0/1 flag into a ballot buffer, a scan converts flags to compact indices, then a gather pass collects results. This pattern appears throughout this chapter.

**Key GPU primitives used:**

| Primitive | Vulkan/CUDA mechanism | Use in this chapter |
|---|---|---|
| Parallel reduction | Subgroup `subgroupAdd`, `atomicAdd` | Extreme-point finding (§2), centroid (§7) |
| Prefix scan | Compute shader cascade or `thrust::exclusive_scan` | Compaction after partition (§2, §5, §6) |
| Radix sort | `thrust::sort`, `vk_radix_sort` | Angular sort for visibility (§11), ANN bucket sort (§8) |
| Gather/scatter | `imageLoad`/`imageStore` | JFA seed propagation (§7) |
| Subgroup ballot | `subgroupBallot` | Fast AABB overlap masking (§6) |

[Source: Vulkan Specification §9 — Compute Pipelines, https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html]

---

## 2. Convex Hull on GPU

### 2.1 CGAL Reference Implementations (CPU)

CGAL provides the definitive CPU reference for both 2D and 3D convex hull. The 2D package exposes `convex_hull_2()`, which dispatches to the Graham-Andrew scan (O(n log n)) or the output-sensitive Bykat quickhull variant (O(nh)) depending on iterator category. The 3D package's `convex_hull_3()` implements the full quickhull algorithm and accepts any `MutableFaceGraph` as the output polyhedron. [Source: CGAL 2D Convex Hull Documentation, https://doc.cgal.org/latest/Convex_hull_2/index.html] [Source: CGAL 3D Convex Hull Documentation, https://doc.cgal.org/latest/Convex_hull_3/index.html]

```cpp
// CGAL 2D convex hull — CPU reference
#include <CGAL/Exact_predicates_inexact_constructions_kernel.h>
#include <CGAL/convex_hull_2.h>
#include <vector>

using K = CGAL::Exact_predicates_inexact_constructions_kernel;
using Point_2 = K::Point_2;

std::vector<Point_2> points = { /* ... */ };
std::vector<Point_2> hull;
CGAL::convex_hull_2(points.begin(), points.end(),
                    std::back_inserter(hull));
// hull is in counter-clockwise order, contains h extreme points
```

```cpp
// CGAL 3D convex hull — CPU reference
#include <CGAL/Exact_predicates_inexact_constructions_kernel.h>
#include <CGAL/convex_hull_3.h>
#include <CGAL/Surface_mesh.h>

using K   = CGAL::Exact_predicates_inexact_constructions_kernel;
using Mesh = CGAL::Surface_mesh<K::Point_3>;

std::vector<K::Point_3> pts = { /* ... */ };
Mesh hull_mesh;
CGAL::convex_hull_3(pts.begin(), pts.end(), hull_mesh);
// hull_mesh: triangulated polyhedron of extreme points
```

### 2.2 Parallel 2D Quickhull on GPU

The GPU quickhull algorithm parallelises the *partition* step — classifying all remaining points as inside or outside the current edge set — while the recursive structure stays on the CPU as a work queue.

**Algorithm sketch:**

1. *Initial extreme points* (O(n) parallel): reduce to find min-x, max-x, max-y, min-y extreme points via `subgroupMin`/`subgroupMax` across all threads.
2. *Initial partition* (O(n) parallel): for each point, test which side of the line (left extreme, right extreme) it lies on, writing a 1/0 flag. Compact surviving points with prefix scan.
3. *Recursive fan*: for each edge in the frontier, dispatch a compute shader that (a) finds the farthest point (parallel max of signed area), (b) partitions points into two sub-clouds, (c) appends two new edges to the frontier queue.

The prefix scan compaction uses GLSL subgroup operations or falls back to a two-pass approach:

```glsl
// GLSL — prefix scan compaction for GPU quickhull partition step
// Invoked with one thread per point; shared_flags[] is per-workgroup
#version 460
#extension GL_KHR_shader_subgroup_arithmetic : require

layout(local_size_x = 256) in;
layout(std430, binding = 0) readonly  buffer Points   { vec2 pts[];  };
layout(std430, binding = 1) readonly  buffer Edges    { vec4 edges[]; }; // each: (ax,ay,bx,by)
layout(std430, binding = 2) writeonly buffer Flags    { uint flags[]; };
layout(push_constant) uniform PC { uint n_pts; uint edge_idx; };

void main() {
    uint gid = gl_GlobalInvocationID.x;
    if (gid >= n_pts) return;
    vec2 p = pts[gid];
    vec2 a = edges[edge_idx].xy;
    vec2 b = edges[edge_idx].zw;
    // Signed area (cross product) — positive means point is left of AB
    float cross = (b.x - a.x) * (p.y - a.y) - (b.y - a.y) * (p.x - a.x);
    flags[gid] = (cross > 0.0) ? 1u : 0u;
}
```

After the flag buffer is written, a second compute pass uses `subgroupExclusiveAdd` to compute prefix sums within each workgroup, then a CPU-driven cascade reduces workgroup totals to global prefix offsets. The result is a compact index buffer of points outside the current edge.

**Complexity.** Expected O(n log n) over uniform inputs; worst-case O(n²) for adversarial inputs. The GPU constant factor is small for the partition step but each recursive level incurs a dispatch call, making CPU-GPU round-trips the dominant cost for small n. In practice, GPU quickhull is worthwhile for n > 100,000 points.

### 2.3 3D Quickhull on GPU

The 3D case adds a *point-above-plane* test: for each candidate plane (a triangle face of the current hull), each point is tested against the plane equation (ax + by + cz + d > 0). The *horizon ridge* — the set of boundary edges visible from the farthest point — requires a topology traversal that is most naturally done on the CPU after the GPU identifies the farthest point.

The hybrid pattern: GPU dispatches a reduction kernel to find the farthest point from each face (one subgroup reduce per face in the frontier); CPU collects results, updates the face topology, enqueues new frontier faces; repeat until no point lies outside any face.

---

## 3. 2D Polygon Boolean Operations

### 3.1 Sutherland-Hodgman for Convex Clipping

The Sutherland-Hodgman algorithm clips a *subject* polygon against each edge of a *convex clip* polygon in sequence. Each clipping step is independent: given a subject edge and a clip edge, compute (a) whether each subject vertex is inside or outside the clip half-plane, and (b) any intersection of the subject edge with the clip edge.

On the GPU, Sutherland-Hodgman parallelises over the *subject polygon's edges* when the clip polygon is fixed and the same clip polygon applies to many independent subject polygons (the batch clipping case common in tile-based rendering). Each thread processes one subject polygon against the full clip boundary:

```glsl
// GLSL pseudocode — Sutherland-Hodgman single polygon clip
// One thread per subject polygon; clip polygon is in a UBO
bool inside(vec2 p, vec2 a, vec2 b) {
    return (b.x - a.x) * (p.y - a.y) - (b.y - a.y) * (p.x - a.x) >= 0.0;
}

vec2 intersect_seg(vec2 a, vec2 b, vec2 c, vec2 d) {
    // Line-line intersection parameterised on AB
    vec2 r = b - a, s = d - c;
    float t = cross2(c - a, s) / cross2(r, s);
    return a + t * r;
}
```

Sutherland-Hodgman only handles convex clip polygons. For general (concave, non-convex) polygons, a different approach is required.

### 3.2 Clipper2 — CPU Library for General Polygon Boolean Ops

Clipper2 is a C++17 library implementing polygon intersection, union, difference, and XOR for arbitrary (including self-intersecting) polygons. It processes `Paths64` (integer coordinates) or `PathsD` (floating-point) and returns clipped polygons as `Paths64`/`PathsD` collections. [Source: Clipper2 GitHub, https://github.com/AngusJohnson/Clipper2]

```cpp
// Clipper2 — polygon intersection (CPU)
#include "clipper2/clipper.h"
using namespace Clipper2Lib;

Paths64 subject, clip;
// Populate with integer-coordinate polygon vertices (scale floats by 1e6)
subject.push_back(MakePath({0,0, 100,0, 100,100, 0,100}));
clip.push_back  (MakePath({50,50, 150,50, 150,150, 50,150}));

Paths64 solution = Intersect(subject, clip, FillRule::NonZero);
```

Clipper2 runs entirely on the CPU. It uses an event-driven scanline sweep to enumerate all edge intersections and an even-odd/non-zero fill rule to classify face regions. There is no GPU port of Clipper2 as of mid-2026; the algorithm's data-dependent linked-list traversal does not parallelise well.

**GPU integration pattern.** For interactive applications that need polygon booleans inside a rendering loop: run Clipper2 on the CPU whenever the polygon topology changes (shape editing, animation keyframes), upload the resulting vertex and index arrays as Vulkan vertex buffers, and rasterise the result directly. The GPU sees only the triangle mesh of the result, not the clipping computation.

### 3.3 Greiner-Hormann Algorithm

The Greiner-Hormann algorithm computes polygon intersection and union for non-convex polygons by traversing two polygon vertex lists and toggling entry/exit status at intersection points. It handles the general case but fails at degenerate configurations (coincident edges, vertices on edges) without additional preprocessing. The Sutherland-Hodgman variant handles convex clips; Greiner-Hormann handles the general case. Both are CPU algorithms. [Source: Greiner and Hormann, "Efficient Clipping of Arbitrary Polygons," ACM Transactions on Graphics 17(2), 1998, https://doi.org/10.1145/274363.274364]

**GPU broadphase.** Before invoking either CPU polygon-op algorithm, a GPU compute shader can prune the O(n²) candidate edge pairs using AABB overlap tests, delivering only intersecting edge pairs to the CPU. This is the same pattern used in collision detection broadphase (§6).

### 3.4 Use Cases

- **2D UI hit testing and compositing:** Stencil regions for widget clipping. Clipper2 computes the clipped region once at layout time; GPU rasterises.
- **Vector graphics SVG rendering:** Path fill rules (non-zero, even-odd) implemented via Clipper2's `FillRule` enum before GPU rasterisation.
- **CAD section views:** Cross-section plane intersected with polygon faces, producing 2D contour polygons uploaded as line primitives.

---

## 4. 2D and 3D Triangulation

### 4.1 Ear-Clipping Triangulation

Ear-clipping is the simplest O(n²) triangulation for simple polygons. An *ear* is a vertex whose triangle (formed with its two neighbours) lies entirely inside the polygon. On the CPU, ear-clipping proceeds by iterating the vertex list, clipping one ear per iteration.

**GPU parallelisation of ear identification.** All ear tests can run in parallel: for each vertex v with neighbours u and w, test (a) whether the triangle UVW is counter-clockwise (a valid ear orientation) and (b) whether any other polygon vertex lies inside UVW. Test (b) is an O(n) scan per vertex, making the total O(n²) per round.

```glsl
// GLSL — parallel ear test (one thread per candidate vertex)
layout(std430, binding = 0) readonly buffer Verts { vec2 v[]; };
layout(std430, binding = 1) writeonly buffer IsEar { uint is_ear[]; };
layout(push_constant) uniform PC { uint n; };

bool point_in_triangle(vec2 p, vec2 a, vec2 b, vec2 c) {
    float d1 = sign((b.x-a.x)*(p.y-a.y) - (b.y-a.y)*(p.x-a.x));
    float d2 = sign((c.x-b.x)*(p.y-b.y) - (c.y-b.y)*(p.x-b.x));
    float d3 = sign((a.x-c.x)*(p.y-c.y) - (a.y-c.y)*(p.x-c.x));
    return !((d1 < 0 || d2 < 0 || d3 < 0) && (d1 > 0 || d2 > 0 || d3 > 0));
}

void main() {
    uint i = gl_GlobalInvocationID.x;
    if (i >= n) return;
    uint prev = (i + n - 1) % n;
    uint next = (i + 1) % n;
    vec2 a = v[prev], b = v[i], c = v[next];

    // Must be a left turn (convex vertex)
    float cross = (c.x-a.x)*(b.y-a.y) - (c.y-a.y)*(b.x-a.x);
    if (cross >= 0.0) { is_ear[i] = 0u; return; }

    // No other vertex inside triangle ABC
    for (uint j = 0; j < n; ++j) {
        if (j == prev || j == i || j == next) continue;
        if (point_in_triangle(v[j], a, b, c)) { is_ear[i] = 0u; return; }
    }
    is_ear[i] = 1u;
}
```

The bottleneck is *sequential ear removal*: removing an ear changes the polygon, invalidating adjacent ear flags. Each removal is followed by re-testing only the two adjacent vertices, so at most O(n) GPU dispatches are required, each testing two vertices. This hybrid CPU-topology / GPU-query pattern reduces total work but incurs O(n) dispatch round-trips.

**Practical alternative.** For GPU font glyph triangulation, `libtess2` (a CPU tessellator derived from GLU) is the standard choice — it handles self-intersecting paths, winding rules, and produces well-formed triangle fans suitable for GPU upload. [Source: libtess2 GitHub, https://github.com/memononen/libtess2]

### 4.2 Constrained Delaunay Triangulation (CDT)

Constrained Delaunay triangulation inserts required edges (polygon boundaries, crack lines, CAD features) into a Delaunay triangulation while maintaining the empty-circumcircle property where constraints allow. CGAL implements `Constrained_Delaunay_triangulation_2<Traits,Tds,Itag>`, which handles arbitrary overlapping polyline constraints. [Source: CGAL Constrained Delaunay Triangulation, https://doc.cgal.org/latest/Triangulation_2/index.html]

CDT construction is CPU-only; the Bowyer-Watson incremental insert requires sequential topology updates. The GPU role is a *refinement pass* after CDT: Ruppert's algorithm (Delaunay refinement for quality meshing) inserts circumcenters of bad triangles iteratively. The *quality test* (is a triangle's circumradius-to-shortest-edge ratio above threshold?) can be evaluated in parallel on the GPU for all triangles; the CPU then selects a conflict-free subset of bad triangles and inserts their circumcenters, avoiding dependency conflicts.

### 4.3 Lifting to 3D Convex Hull

A classical reduction: the Delaunay triangulation of n 2D points equals the projection of the lower convex hull of the paraboloid lifting (x, y) → (x, y, x²+y²). This allows using the 3D convex hull algorithm (§2.3) to compute Delaunay triangulations by lifting points to 3D, computing the hull, and projecting down-facing faces back to 2D. GPU 3D quickhull then implies a GPU Delaunay path, though the projection step requires care with degenerate (cocircular) points.

---

## 5. Point Location and Geometric Queries

### 5.1 Point-in-Polygon: Ray Casting and Winding Number

The two standard point-in-polygon algorithms — ray casting and winding number — both admit straightforward GPU parallelism when queries are independent.

**Ray casting** fires a horizontal ray from query point p rightward and counts edge crossings. An odd count implies inside. On the GPU, when the polygon is fixed and Q query points are tested:

```glsl
// GLSL — ray casting point-in-polygon, one thread per query point
layout(std430, binding = 0) readonly  buffer Polygon { vec2 poly[]; };
layout(std430, binding = 1) readonly  buffer Queries { vec2 query[]; };
layout(std430, binding = 2) writeonly buffer Results { uint inside[]; };
layout(push_constant) uniform PC { uint n_poly; uint n_query; };

void main() {
    uint qi = gl_GlobalInvocationID.x;
    if (qi >= n_query) return;
    vec2 p = query[qi];
    int crossings = 0;
    for (uint i = 0; i < n_poly; ++i) {
        vec2 a = poly[i], b = poly[(i + 1) % n_poly];
        if ((a.y > p.y) != (b.y > p.y)) {
            float x_intersect = a.x + (p.y - a.y) / (b.y - a.y) * (b.x - a.x);
            if (p.x < x_intersect) crossings++;
        }
    }
    inside[qi] = uint(crossings & 1);
}
```

**Winding number** counts how many times the polygon winds around p. It correctly handles self-intersecting polygons, where ray casting can give wrong answers. The winding number for a single query runs in O(n_poly) per thread; for Q queries against a fixed polygon, the total GPU work is O(Q × n_poly).

### 5.2 Batch Point Location via BVH

When queries must be located within a *set of polygons* (e.g., finding which UI widget a touch point falls into), build an AABB tree over polygon bounding boxes. Each query point descends the BVH, pruning subtrees that cannot contain the point's position, then tests the candidate leaf polygons. The BVH is built once on the CPU and uploaded as a flattened array of `(aabb_min, aabb_max, left_child, right_child, polygon_idx)` nodes. GPU traversal uses an explicit stack in local memory — typically 32 entries deep — and is the same pattern as BVH ray traversal in §6.

### 5.3 Picking and Selection in 3D

Interactive picking (clicking on a 3D object) proceeds as:
1. Unproject the screen-space click position to a view-space ray (CPU, one matrix multiply).
2. Traverse the scene BVH against this ray on the GPU (a single-thread dispatch suffices for interactive framerates, though batched picking dispatches multiple rays in one pass).
3. Report the closest intersection.

For GPU-driven rendering pipelines (Ch154) where draw calls are GPU-generated, the picking pass can reuse the same BVH used for frustum and occlusion culling, avoiding duplicate data structures.

---

## 6. Geometric Intersection Testing

### 6.1 Primitive Intersection Test Suite

The following tests form the core of any collision system. All are O(1) per pair and are embarrassingly parallel on the GPU.

**Ray-AABB (slab test).** Intersect the ray with each pair of axis-aligned slab planes; the ray hits the AABB if the interval [t_enter, t_exit] is non-empty.

```glsl
bool ray_aabb(vec3 ray_o, vec3 ray_d_inv, vec3 aabb_min, vec3 aabb_max,
              out float t_min, out float t_max) {
    vec3 t0 = (aabb_min - ray_o) * ray_d_inv;
    vec3 t1 = (aabb_max - ray_o) * ray_d_inv;
    vec3 tlo = min(t0, t1), thi = max(t0, t1);
    t_min = max(max(tlo.x, tlo.y), tlo.z);
    t_max = min(min(thi.x, thi.y), thi.z);
    return t_max >= max(t_min, 0.0);
}
```

**Ray-Triangle (Möller-Trumbore).** The Möller-Trumbore algorithm computes the intersection of a ray with a triangle without explicitly constructing the triangle's plane equation. [Source: Möller and Trumbore, "Fast, Minimum Storage Ray/Triangle Intersection," Journal of Graphics Tools, 1997, https://www.tandfonline.com/doi/abs/10.1080/10867651.1997.10487468]

```glsl
bool ray_triangle(vec3 o, vec3 d, vec3 v0, vec3 v1, vec3 v2,
                  out float t, out float u, out float v) {
    const float EPS = 1e-7;
    vec3 e1 = v1 - v0, e2 = v2 - v0;
    vec3 h = cross(d, e2);
    float a = dot(e1, h);
    if (abs(a) < EPS) return false;          // parallel
    float f = 1.0 / a;
    vec3  s = o - v0;
    u = f * dot(s, h);
    if (u < 0.0 || u > 1.0) return false;
    vec3  q = cross(s, e1);
    v = f * dot(d, q);
    if (v < 0.0 || u + v > 1.0) return false;
    t = f * dot(e2, q);
    return t > EPS;
}
```

**Ray-Sphere.** Assumes `d` is normalised; if not, divide the discriminant differently.
```glsl
bool ray_sphere(vec3 o, vec3 d, vec3 centre, float radius, out float t) {
    // d must be unit length; dot(d,d) == 1 is assumed
    vec3 oc = o - centre;
    float b = dot(oc, d);
    float c = dot(oc, oc) - radius * radius;
    float disc = b * b - c;
    if (disc < 0.0) return false;
    t = -b - sqrt(disc);
    return t > 0.0;
}
```

**Ray-Capsule.** Test the ray against the cylinder swept by the segment, then cap with two hemisphere tests. Omitted for space; the decomposition is standard. [Source: Ericson, *Real-Time Collision Detection*, CRC Press 2004, §5.3]

**Segment-Segment (2D).** Two segments AB and CD intersect if the sign of the cross products orient2d(A,B,C) and orient2d(A,B,D) differ, and vice versa for CD's test against AB.

**Triangle-Triangle (Devillers-Guigue).** Uses six orientation tests (orient3d) to determine whether two triangles in 3D intersect. [Source: Devillers and Guigue, "Faster Triangle-Triangle Intersection Tests," INRIA Research Report 4488, 2002, https://inria.hal.science/inria-00072100]

**AABB-AABB.** Two AABBs overlap iff they overlap on all three axes. One conditional per axis:
```glsl
bool aabb_aabb(vec3 a_min, vec3 a_max, vec3 b_min, vec3 b_max) {
    return all(lessThanEqual(a_min, b_max)) && all(lessThanEqual(b_min, a_max));
}
```

**OBB-OBB (Separating Axis Theorem).** Fifteen potential separating axes: three face normals from each OBB, plus nine cross products of edge direction pairs. If no axis separates the projections, the OBBs overlap. [Source: Gottschalk, Lin, and Manocha, "OBBTree: A Hierarchical Structure for Rapid Interference Detection," SIGGRAPH 1996 Proceedings]

**Sphere-AABB.**
```glsl
bool sphere_aabb(vec3 centre, float r, vec3 aabb_min, vec3 aabb_max) {
    vec3 closest = clamp(centre, aabb_min, aabb_max);
    return dot(closest - centre, closest - centre) <= r * r;
}
```

### 6.2 GPU Batch Intersection (N×M Pairs)

When all N objects in set A must be tested against all M objects in set B (N×M broadphase), dispatch `ceil(N*M / 256)` workgroups with one thread per pair:

```glsl
layout(local_size_x = 256) in;
layout(std430, binding = 0) readonly  buffer SetA    { AABB a_boxes[];  };
layout(std430, binding = 1) readonly  buffer SetB    { AABB b_boxes[];  };
layout(std430, binding = 2) writeonly buffer Pairs   { uvec2 pairs[];   };
layout(std430, binding = 3)           buffer Counter { uint n_pairs;     };
layout(push_constant) uniform PC { uint n_a; uint n_b; };

void main() {
    uint tid = gl_GlobalInvocationID.x;
    uint ai  = tid / n_b;
    uint bi  = tid % n_b;
    if (ai >= n_a) return;
    if (aabb_aabb(a_boxes[ai].lo, a_boxes[ai].hi,
                  b_boxes[bi].lo, b_boxes[bi].hi)) {
        uint slot = atomicAdd(n_pairs, 1u);
        pairs[slot] = uvec2(ai, bi);
    }
}
```

The output `pairs` buffer feeds the narrowphase (exact intersection tests) as a second dispatch. This two-phase broadphase/narrowphase mirrors the structure of bullet3's GPU broadphase. [Source: Bullet Physics GPU broadphase, https://github.com/bulletphysics/bullet3]

---

## 7. Voronoi Diagrams and Delaunay

### 7.1 Jump Flooding Algorithm (JFA) for 2D Voronoi

The Jump Flooding Algorithm computes a 2D discrete Voronoi diagram on a GPU render target in O(log n) passes, where n is the larger texture dimension. Each pass propagates seed information in jumps of decreasing step size: n/2, n/4, ..., 1. [Source: Rong and Tan, "Jump Flooding in GPU with Applications to Voronoi Diagram and Distance Transform," I3D 2006, https://doi.org/10.1145/1111411.1111431]

**Algorithm.** Initialise a `RG32UI` texture: seed pixels store their own (x, y) coordinates; all other pixels store (UINT_MAX, UINT_MAX). Then for step k = n/2, n/4, ..., 1:

```glsl
// JFA single pass — fragment shader over full-screen quad
layout(binding = 0) uniform usampler2D seed_tex;
layout(location = 0) out uvec2 out_seed;
uniform int step_k;

void main() {
    ivec2 p = ivec2(gl_FragCoord.xy);
    uvec2 best = texelFetch(seed_tex, p, 0).rg;
    float best_dist = (best == uvec2(~0u)) ? 1e30 : distance(vec2(p), vec2(best));

    for (int dy = -1; dy <= 1; ++dy) {
        for (int dx = -1; dx <= 1; ++dx) {
            ivec2 q = p + ivec2(dx, dy) * step_k;
            if (any(lessThan(q, ivec2(0))) ||
                any(greaterThanEqual(q, textureSize(seed_tex, 0)))) continue;
            uvec2 cand = texelFetch(seed_tex, q, 0).rg;
            if (cand == uvec2(~0u)) continue;
            float d = distance(vec2(p), vec2(cand));
            if (d < best_dist) { best_dist = d; best = cand; }
        }
    }
    out_seed = best;
}
```

After all passes, each pixel holds the coordinates of its nearest seed. A second pass computes Euclidean distance from each pixel to its recorded seed, producing a distance transform.

**Correctness caveats.** Standard JFA (9 neighbours per pass, step halving) is approximate: it can misidentify the nearest seed for pixels in thin Voronoi regions or for seed configurations with very small gaps. The JFA+1 variant adds a final pass at step 1 after all halving steps to correct most errors. JFA+2 adds two extra passes. For exact discrete Voronoi, a dedicated scanline algorithm or the GPU rasterisation trick (render cones/pyramids per seed, depth test selects nearest) is used. [Source: Rong et al., "Variants of Jump Flooding Algorithm for Computing Discrete Voronoi Diagrams," IEEE ISVD 2007]

### 7.2 Centroidal Voronoi Tessellation (Lloyd's Algorithm)

Centroidal Voronoi tessellation (CVT) minimises a clustering energy by iterating:
1. Assign each sample point to its nearest seed (nearest-site assignment).
2. Move each seed to the centroid of its Voronoi cell.

Both steps are GPU-parallel. Step 1 is a JFA query (§7.1) or a brute-force k-NN pass (§8). Step 2 uses parallel reduction: each sample votes for its assigned seed with its position, and a segmented reduction accumulates per-seed position sums and counts. FAISS GPU implements Lloyd's k-means directly (§8) and is the practical tool for CVT in high dimensions.

### 7.3 Bowyer-Watson Incremental Delaunay on GPU

Bowyer-Watson inserts points one at a time, finds all triangles whose circumcircles contain the new point (the *conflict list*), deletes them to form a star-shaped cavity, and re-triangulates from the new point. The conflict list traversal is inherently sequential across insertions.

**GPU acceleration strategies:**
- Run the Bowyer-Watson core on the CPU. GPU accelerates the *circumcircle test* for each (point, triangle) pair in the conflict query, which is an `incircle` predicate (§12) evaluated in parallel.
- Parallelise across *independent* insertions: points that are far enough apart (in separate Voronoi cells of the current triangulation) can be inserted simultaneously without conflicts. This is the approach used in parallel Delaunay implementations based on conflict-free insertion sets — a technique described in the GPU triangulation research literature.

---

## 8. Approximate Nearest Neighbour Search

### 8.1 FAISS GPU

FAISS (Facebook AI Similarity Search) is the standard GPU library for high-dimensional approximate nearest-neighbour (ANN) search on Linux. It exposes GPU index types that serve as drop-in replacements for their CPU counterparts. [Source: FAISS GPU documentation, https://github.com/facebookresearch/faiss/wiki/Faiss-on-the-GPU]

**Index types available on GPU:**

| Index | Description | Use case |
|---|---|---|
| `GpuIndexFlatL2` | Brute-force exact L2 search | Small databases, exact results |
| `GpuIndexFlatIP` | Brute-force inner product | Maximum inner product search |
| `GpuIndexIVFFlat` | Inverted file, flat quantiser | Medium databases, approximate |
| `GpuIndexIVFScalarQuantizer` | IVF + scalar quantisation | Reduced memory footprint |
| `GpuIndexIVFPQ` | IVF + product quantisation | Large databases, aggressive compression |

```cpp
// FAISS GPU — exact k-NN search on a flat index
#include <faiss/gpu/StandardGpuResources.h>
#include <faiss/gpu/GpuIndexFlat.h>

faiss::gpu::StandardGpuResources res;   // allocates scratch memory, cuBLAS handle

faiss::gpu::GpuIndexFlatConfig cfg;
cfg.device = 0;  // GPU device index
faiss::gpu::GpuIndexFlatL2 index(&res, /*dimension=*/128, cfg);

// Add n_base vectors of dimension 128
index.add(n_base, base_vectors_d);      // base_vectors_d: device pointer (float*)

// Search: find k nearest neighbours for each of n_query query vectors
std::vector<faiss::idx_t> I(n_query * k);
std::vector<float>         D(n_query * k);
index.search(n_query, query_vectors_d, k, D.data(), I.data());
// I[i*k .. i*k+k-1]: indices of k nearest neighbours for query i
// D[i*k .. i*k+k-1]: squared L2 distances
```

GPU flat search achieves 5–10× speedup over CPU for batch queries (n_query ≥ 1000). Memory bandwidth — not FLOPs — is the bottleneck; the inner-product matrix between queries and database vectors is computed as a SGEMM on cuBLAS. [Source: FAISS on the GPU wiki, https://github.com/facebookresearch/faiss/wiki/Faiss-on-the-GPU]

**Transferring a CPU index to GPU:**

```cpp
#include <faiss/gpu/GpuCloner.h>
faiss::Index* cpu_index = /* load from disk */;
faiss::Index* gpu_index = faiss::gpu::index_cpu_to_gpu(&res, 0, cpu_index);
```

### 8.2 IVF and Product Quantisation

For databases with millions of vectors, brute-force flat search becomes impractical. Inverted-file (IVF) indexing partitions the database into `nlist` Voronoi cells (via k-means); a query searches only the `nprobe` nearest cells, giving approximate results with controllable accuracy-speed tradeoff.

Product quantisation (PQ) further compresses each database vector by splitting its D dimensions into M sub-vectors, quantising each to a small codebook of K* centroids. The compressed index fits in GPU memory where the full float32 database would not. FAISS limits `k` (number of neighbours returned) and `nprobe` to ≤ 2048.

### 8.3 Brute-Force GPU k-NN for Small n

When the database has fewer than ~100,000 vectors and the dimension is moderate (≤ 512), a custom compute shader brute-force is often simpler to integrate than FAISS:

```glsl
// GLSL — brute-force k-NN (k=1, nearest neighbour)
layout(std430, binding = 0) readonly  buffer DB    { vec4 db[];    }; // float32 vectors
layout(std430, binding = 1) readonly  buffer Query { vec4 q[];     };
layout(std430, binding = 2) writeonly buffer NNIdx { uint nn_idx[]; };
layout(push_constant) uniform PC { uint n_db; uint n_q; uint dim4; };

void main() {
    uint qi = gl_GlobalInvocationID.x;
    if (qi >= n_q) return;
    float best = 1e30; uint best_i = 0u;
    for (uint di = 0; di < n_db; ++di) {
        float dist2 = 0.0;
        for (uint c = 0; c < dim4; ++c) {
            vec4 diff = q[qi * dim4 + c] - db[di * dim4 + c];
            dist2 += dot(diff, diff);
        }
        if (dist2 < best) { best = dist2; best_i = di; }
    }
    nn_idx[qi] = best_i;
}
```

**Applications.** ANN search underpins point cloud registration (Ch211 §ICP), feature matching in visual odometry, and GPU-side texture synthesis (finding best-matching patches).

### 8.4 FLANN — KD-Tree and LSH on CPU

FLANN (Fast Library for Approximate Nearest Neighbours) is a widely used CPU library for ANN search, providing KD-tree and LSH (Locality Sensitive Hashing) index structures. [Source: FLANN GitHub, https://github.com/flann-lib/flann] [Source: Muja and Lowe, "Fast Approximate Nearest Neighbors with Automatic Algorithm Configuration," VISAPP 2009] FLANN is used extensively as the ANN backend in PCL (Point Cloud Library) and OpenCV's feature-matching pipelines.

FLANN operates entirely on the CPU. Its KD-tree search is efficient for moderate dimensions (≤ 30–50), while LSH handles higher dimensions. There is no actively maintained GPU backend in the current FLANN codebase; GPU-accelerated ANN on Linux uses FAISS (§8.1) or, for CUDA-integrated pipelines, cuVS (NVIDIA's cuVS library, part of the RAPIDS ecosystem). [Note: historical GPU KD-tree modules for FLANN have appeared in research forks but are not part of the upstream release — needs verification for current status.]

**CPU usage pattern for GPU integration:**

```cpp
// FLANN — KD-tree ANN on CPU, result uploaded to GPU
#include <flann/flann.hpp>

flann::Matrix<float> dataset(data_ptr, n_pts, dim);
flann::Matrix<float> query  (qry_ptr,  n_qry, dim);
flann::Matrix<int>   indices(idx_ptr,  n_qry, k);
flann::Matrix<float> dists  (dst_ptr,  n_qry, k);

flann::Index<flann::L2<float>> index(dataset,
    flann::KDTreeIndexParams(/*trees=*/4));
index.build();
index.knnSearch(query, indices, dists, k,
    flann::SearchParams(/*checks=*/128));
// Upload indices[] to GPU as an ssbo for subsequent processing
```

---

## 9. Minkowski Sums and Configuration Space

### 9.1 2D Minkowski Sum of Convex Polygons

The Minkowski sum of two convex polygons P (m vertices) and Q (n vertices) is itself a convex polygon with at most m+n vertices. It is computed by the rotating calipers method: merge-sort the edge directions of P and Q by angle, then interleave the edges, accumulating vertex positions. Complexity O(m+n). [Source: O'Rourke, *Computational Geometry in C*, Cambridge 1998, §8.2]

```cpp
// Pseudocode — 2D Minkowski sum by edge direction merge
std::vector<vec2> minkowski_sum_2d(const std::vector<vec2>& P,
                                   const std::vector<vec2>& Q) {
    // Rotate both polygons to start at the bottom-most vertex
    // Sort P's edge vectors by angle; Q's edge vectors by angle
    // Merge the two sorted edge sequences by angle (like merge sort)
    // Accumulate the merged edge sequence from P[0] + Q[0]
    // Return the resulting polygon (at most |P|+|Q| vertices)
}
```

The rotating calipers merge is inherently sequential. **GPU role:** when many Minkowski sum computations are needed in parallel (one per rigid body pair in a physics simulation), dispatch one thread per pair, each executing the O(m+n) merge loop independently. For small polygon vertex counts (m, n ≤ 32), this fits in registers and is practical.

### 9.2 GJK — Gilbert-Johnson-Keerthi Algorithm

GJK computes the minimum distance (or detects overlap) between two convex sets A and B by iteratively finding the simplex in A⊕(-B) = {a - b | a∈A, b∈B} closest to the origin. [Source: Gilbert, Johnson, Keerthi, "A fast procedure for computing the distance between complex objects in three-dimensional space," IEEE Transactions on Robotics and Automation 4(2), 1988]

The key operation is the *support function*: given a direction d, find the point in A furthest in direction d (and similarly for B). For a convex polyhedron with n vertices, this is a linear scan O(n) or O(log n) on a sorted vertex set.

```glsl
// GLSL — GJK support function for a convex polyhedron (one thread per pair)
vec3 support(vec3 d, uint obj_vtx_offset, uint n_vtx) {
    vec3 best = vec3(0.0); float best_dot = -1e30;
    for (uint i = 0; i < n_vtx; ++i) {
        vec3 v = vertices[obj_vtx_offset + i];
        float dd = dot(v, d);
        if (dd > best_dot) { best_dot = dd; best = v; }
    }
    return best;
}
```

GJK iterates: (1) pick support point, (2) update simplex (point/segment/triangle/tetrahedron), (3) find closest point on simplex to origin, (4) check termination. The iteration count is bounded in practice by a small constant (typically 10–30 iterations).

**GPU batch GJK.** For N object pairs in a collision narrowphase, dispatch N threads, each running the full GJK loop for its pair. This is used in the GPU narrowphase of engines such as Bullet's OpenCL backend. The loop terminates independently per pair (no cross-thread synchronisation needed).

### 9.3 EPA — Expanding Polytope Algorithm

When GJK terminates with the origin inside the Minkowski difference (objects overlap), EPA recovers the penetration depth and contact normal. Starting from GJK's final simplex (a tetrahedron enclosing the origin), EPA iteratively expands the polytope toward the origin boundary. [Source: van den Bergen, "A Fast and Robust GJK Implementation," Journal of Graphics Tools 4(2), 1999]

EPA is harder to parallelise than GJK: the polytope expansion requires dynamically inserting new triangles based on which face is closest to the origin. In practice, EPA runs on the CPU for the small number of overlapping pairs identified by GJK.

---

## 10. Geometric Arrangements and Sweep Line

### 10.1 Line Arrangement

The arrangement of n lines in the plane has O(n²) vertices, edges, and faces. Computing it exactly requires a sweep-line algorithm (Bentley-Ottmann) or an incremental approach, both O(n² log n) worst-case. Neither maps well to GPU due to dynamic event-queue management.

**GPU-parallel segment intersection enumeration.** The all-pairs test (does segment i intersect segment j?) is O(n²) and embarrassingly parallel: dispatch n² threads, each performing a 2D segment-segment test (§6.1). This enumerates all intersecting pairs without computing the arrangement topology, which suffices for many applications (e.g., finding all self-intersections in a polyline set):

```glsl
layout(local_size_x = 16, local_size_y = 16) in;
layout(std430, binding = 0) readonly  buffer Segs  { vec4 segs[];   }; // (ax,ay,bx,by)
layout(std430, binding = 1) writeonly buffer Pairs { uvec2 pairs[]; };
layout(std430, binding = 2)           buffer Cnt   { uint count;     };
layout(push_constant) uniform PC { uint n; };

bool segments_intersect(vec2 a, vec2 b, vec2 c, vec2 d) {
    // orient2d sign-change tests
    float d1 = (b.x-a.x)*(c.y-a.y) - (b.y-a.y)*(c.x-a.x);
    float d2 = (b.x-a.x)*(d.y-a.y) - (b.y-a.y)*(d.x-a.x);
    float d3 = (d.x-c.x)*(a.y-c.y) - (d.y-c.y)*(a.x-c.x);
    float d4 = (d.x-c.x)*(b.y-c.y) - (d.y-c.y)*(b.x-c.x);
    return (sign(d1) != sign(d2)) && (sign(d3) != sign(d4));
}

void main() {
    uint i = gl_GlobalInvocationID.x;
    uint j = gl_GlobalInvocationID.y;
    if (i >= n || j >= n || i >= j) return;
    if (segments_intersect(segs[i].xy, segs[i].zw,
                           segs[j].xy, segs[j].zw)) {
        uint slot = atomicAdd(count, 1u);
        pairs[slot] = uvec2(i, j);
    }
}
```

For n > a few thousand segments, the O(n²) GPU approach becomes memory-bound. Pair this with a spatial grid BVH (§5.2) to prune to only nearby segment pairs before the exact test.

### 10.2 Half-Plane Intersection and Linear Programming in 2D

Given n half-planes, their intersection is a (possibly empty or unbounded) convex polygon. This is equivalent to 2D linear programming. Seidel's randomised incremental LP algorithm solves 2D LP in expected O(n) time.

**GPU all-pairs feasibility test.** A weaker query — "is this point feasible (inside all half-planes)?" — is trivially parallel: each thread tests one half-plane in one dot-product comparison. Checking M candidate points against n half-planes is O(M×n) GPU work with no communication.

### 10.3 Applications

- **Shadow computation:** The shadow volume of a convex occluder against a point light is a Minkowski sum (§9.1). The lit/shadowed classification of a surface point is a half-plane intersection query.
- **BSP plane intersection:** Building a BSP tree requires classifying polygon vertices against splitting planes — an O(n) per-plane batch test, GPU-parallel.

---

## 11. Visibility and Line-of-Sight

### 11.1 2D Visibility Polygon

The *visibility polygon* from a query point p is the region of the plane visible from p within a set of obstacles (line segments). The angular sweep algorithm computes it in O(n log n):

1. Collect all obstacle endpoints and sort by polar angle around p.
2. Maintain an active-segment set ordered by distance from p; update it as each angular event is processed.

Steps 1 (endpoint collection) and the distance computation are GPU-parallel; the sorted angular order and active-set maintenance are sequential. **Hybrid approach:** GPU computes `(angle, distance)` for all n endpoints in parallel, CPU sorts and sweeps.

### 11.2 GPU Visibility Graph

The *visibility graph* connects all pairs of obstacle vertices that can see each other. It has O(n²) edges in the worst case. Testing whether two vertices u and v have line-of-sight requires testing their connecting segment against all obstacle edges — O(n) per pair, O(n³) total. On the GPU, all `n²/2` pairs can be tested in parallel, with each thread doing an O(n) inner loop:

```glsl
// GPU visibility graph — one thread per (i,j) vertex pair
void main() {
    uint i = gl_GlobalInvocationID.x;
    uint j = gl_GlobalInvocationID.y;
    if (i >= j || i >= n_verts || j >= n_verts) return;
    vec2 u = verts[i], v = verts[j];
    bool visible = true;
    for (uint k = 0; k < n_segs && visible; ++k) {
        vec2 a = segs[k].xy, b = segs[k].zw;
        if (segments_intersect(u, v, a, b)) visible = false;
    }
    visibility[i * n_verts + j] = uint(visible);
}
```

Total GPU work: O(n² × n_segs). For n = 1,000 vertices and 500 segments, this is 500 million comparisons — achievable in under a second on a modern discrete GPU.

### 11.3 Portal-Based Visibility for 3D Scenes

A *portal* is a convex polygon hole in a wall connecting two convex cells. Visibility determination proceeds by: starting from the camera cell, propagating a *view frustum* through portals into adjacent cells, accumulating clipped frustum windows. This is a CPU-side graph traversal (the portal graph is small, O(100) cells in practice). The GPU contributes *occlusion queries* (`vkCmdBeginQuery` with `VK_QUERY_TYPE_OCCLUSION`) to test whether a portal's bounding geometry is visible before traversing it.

### 11.4 Terrain Line-of-Sight

The *horizon-line algorithm* for terrain visibility proceeds along a ray in the height field, tracking the maximum elevation angle seen so far. Each query ray (one source point, one target point) is independent, enabling GPU parallelism over a batch of (source, target) queries:

```glsl
// GPU terrain line-of-sight — one thread per (source, target) pair
bool terrain_los(vec2 src, vec2 tgt, float src_h, float tgt_h) {
    float horizon_slope = -1e30;
    float dist = distance(src, tgt);
    uint steps = uint(dist);
    for (uint s = 1; s < steps; ++s) {
        vec2 pos = mix(src, tgt, float(s) / float(steps));
        float terrain_h = texture(heightmap, pos / map_size).r * max_height;
        float elev_angle = (terrain_h - src_h) / float(s);
        if (elev_angle > horizon_slope) horizon_slope = elev_angle;
    }
    float tgt_angle = (tgt_h - src_h) / float(steps);
    return tgt_angle >= horizon_slope;
}
```

---

## 12. Robust Geometric Predicates

### 12.1 The Robustness Problem

Floating-point geometry computations are unreliable near degenerate configurations. A sign-of-determinant predicate computed in floating-point may return the wrong sign when the true determinant is near zero but not exactly zero — causing topological inconsistencies (crossing edges that test as non-crossing, points classified as inside when they are outside). These errors corrupt triangulations, Boolean operations, and arrangement construction in non-recoverable ways.

### 12.2 Shewchuk's Adaptive Predicates

Shewchuk's public-domain C implementation provides four predicates that are *adaptive*: they use floating-point arithmetic for the common case (large determinant, answer clearly correct) and switch to exact arithmetic only when the floating-point result is uncertain. [Source: Shewchuk, "Robust Adaptive Floating-Point Geometric Predicates," SoCG 1996, https://www.cs.cmu.edu/~quake/robust.html]

The four predicates:

| Predicate | Meaning | Determinant size |
|---|---|---|
| `orient2d(pa, pb, pc)` | Sign of the orientation of triangle ABC | 3×3 |
| `orient3d(pa, pb, pc, pd)` | Is D above/below plane ABC? | 4×4 |
| `incircle(pa, pb, pc, pd)` | Is D inside the circumcircle of ABC? | 4×4 |
| `insphere(pa, pb, pc, pd, pe)` | Is E inside the circumsphere of ABCD? | 5×5 |

Each predicate is computed in a sequence of increasingly expensive stages. The fast path uses a standard floating-point determinant with an error bound. If the magnitude of the result exceeds the error bound, the sign is reliable and the computation terminates. Otherwise, it falls back to exact expansion arithmetic.

```c
/* Shewchuk orient2d — call from C with 2D point coordinate arrays */
/* Returns positive if (pa, pb, pc) are in counter-clockwise order,
   negative if clockwise, zero if collinear.                         */
REAL orient2d(REAL *pa, REAL *pb, REAL *pc);
REAL orient3d(REAL *pa, REAL *pb, REAL *pc, REAL *pd);
REAL incircle(REAL *pa, REAL *pb, REAL *pc, REAL *pd);
REAL insphere(REAL *pa, REAL *pb, REAL *pc, REAL *pd, REAL *pe);
```

The predicates require `exactinit()` to be called once at program startup to initialise floating-point splitters. On x86 processors with 80-bit extended precision registers, the FPU precision must be set to 64-bit double (FLDCW instruction) before use; on x86-64 with SSE2, this is automatic.

### 12.3 GPU-Friendly Fixed-Point Predicates

Shewchuk's adaptive arithmetic is CPU-only: it relies on `double` precision and branching patterns that diverge badly in SIMT. For GPU geometric predicates, two approaches are used:

**64-bit integer fixed-point.** If all input coordinates are integers (or scaled rational numbers with a fixed denominator), `orient2d` evaluates exactly using 64-bit integer arithmetic: `(pb.x - pa.x) * (int64_t)(pc.y - pa.y) - (pb.y - pa.y) * (int64_t)(pc.x - pa.x)`. The sign of a 64-bit integer product difference is exact with no floating-point error. **Overflow caveat:** this holds only if coordinate magnitudes fit in 32 bits (|x|, |y| ≤ 2³¹ − 1), so that each difference fits in 32 bits and the product fits in 63 bits. Coordinates larger than ~2³¹ require 128-bit arithmetic (not directly available in GLSL; must be decomposed into two 64-bit words on the CPU). GLSL supports `int64_t` arithmetic via `GL_EXT_shader_explicit_arithmetic_types_int64` on any hardware with 64-bit integer support.

```glsl
#extension GL_EXT_shader_explicit_arithmetic_types_int64 : require
// orient2d in 64-bit integer fixed-point (coordinates scaled to integers)
int64_t orient2d_i64(i64vec2 a, i64vec2 b, i64vec2 c) {
    return (b.x - a.x) * (c.y - a.y) - (b.y - a.y) * (c.x - a.x);
}
```

**Filtered floating-point predicates.** Evaluate the predicate in `float32`, compute an error bound from the coordinate magnitudes, and classify as positive/negative/uncertain. Uncertain results are passed back to the CPU for exact evaluation. This is the approach used in triangle mesh processing libraries that dispatch to GPU shaders for bulk classification.

### 12.4 Simulation of Simplicity (SoS)

When exact predicates return exactly zero (degenerate case: three collinear points, four cocircular points), an algorithm that only consumes the sign of the predicate has no valid answer. Simulation of Simplicity (SoS) is a symbolic perturbation that defines a consistent tie-breaking rule: each degenerate configuration is treated as the limit of a generic one, assigning a deterministic non-zero sign. SoS is implemented as a final fallback after exact arithmetic still returns zero. [Source: Edelsbrunner and Mücke, "Simulation of Simplicity," ACM Trans. on Graphics 9(1), 1990, https://doi.org/10.1145/77635.77639]

SoS is CPU-only. On the GPU, degenerate inputs are better handled by pre-processing on the CPU (jitter collinear inputs by an epsilon, then upload the clean dataset) rather than per-thread SoS evaluation.

---

## Integrations

**Ch208 — GPU Geometry Algorithms (Subdivision, Implicit Surfaces, Skinning).** The GPU convex hull of §2 is used as a primitive in Ch208's convex decomposition (VHACD) and BVH construction passes. The Delaunay lifting reduction (§4.3) connects to Ch208's mesh repair and Delaunay refinement sections.

**Ch209 — GPU Spatial-Differential-Animation.** ANN search (§8) feeds into the point-cloud ICP registration described in Ch209/Ch211. The centroidal Voronoi tessellation (§7.2) appears in Ch209's mesh sampling and geodesic remeshing contexts.

**Ch210 — GPU Physics and Volumetric.** The GJK/EPA narrowphase (§9.2–9.3) and N×M broadphase AABB test (§6.2) are the collision detection core described in detail in Ch210's rigid-body simulation section. The Minkowski sum (§9.1) constructs C-space obstacles used in Ch210's motion planning content.

**Ch211 — GPU Terrain, Ray Tracing, Point Cloud.** Terrain line-of-sight (§11.4) is a companion to Ch211's height-field ray tracing. Point cloud k-NN (§8) is used in Ch211's ICP and surface normal estimation sections.

**Ch135 — Vulkan Ray Tracing.** The ray-primitive intersection tests of §6.1 (ray-AABB, Möller-Trumbore, ray-sphere) correspond directly to the intersection shaders and built-in AABB/triangle tests in `VK_KHR_ray_tracing_pipeline`. Understanding the scalar versions in §6.1 clarifies what the hardware accelerates.

**Ch154 — GPU-Driven Rendering.** The GPU broadphase (§6.2) and BVH point location (§5.2) are used in GPU-driven culling pipelines as described in Ch154. The prefix-scan compaction pattern (§1, §2) is the same technique used in Ch154's draw-call compaction via `VK_EXT_multi_draw`.

**Ch105 — Font Rendering, FreeType2, and the Text Pipeline.** The ear-clipping and CDT triangulation discussion (§4.1–4.2) directly applies to how FreeType glyph outlines are converted to triangle meshes for GPU rendering, as described in Ch105's vector-to-raster pipeline.

**Ch37 — Skia and 2D Rendering.** Polygon Boolean operations (§3) underlie Skia's path clipping implementation. The winding-number point-in-polygon query (§5.1) implements Skia's fill rules. Understanding Clipper2 (§3.2) provides the algorithmic substrate for Skia's CPU path operations.

**Ch212 — GPU Neural and Specialised Primitives.** The robust predicate infrastructure (§12) is used in Ch212's mesh repair and watertightness validation steps, where topological consistency requires exact sign computation before uploading to the GPU.
