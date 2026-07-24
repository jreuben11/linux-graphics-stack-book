# Chapter 225: Computational Topology Algorithms on GPU

**Audiences:** Graphics application developers implementing topological mesh analysis, handle and tunnel detection, or shape simplification; scientific visualisation engineers using TDA (topological data analysis); researchers applying persistent homology to graphics data; systems developers building robust geometry kernels.

**Relationship to the series.** The GPU Geometry Algorithms series (Ch208–212) covers mesh *construction*, *deformation*, and *spatial indexing*. Chapter 222 covers computational geometry algorithms. Chapter 224 covers shape *analysis* (descriptors, matching). This chapter covers the *topological structure* of shapes — algorithms from algebraic topology and topological data analysis applied to geometry, adapted for GPU parallelism. The emphasis is on practical graphics and visualisation applications, not pure mathematics.

**CPU vs. GPU split.** Topology algorithms present a different parallelism profile from geometry algorithms. The most widely deployed production TDA tools (TTK, GUDHI, Ripser) run on CPU, with MPI-distributed memory for large datasets. GPU acceleration is available for a narrow but important set of operations: the Ripser++ library (up to 30× speedup over Ripser), all-pairs distance computation for complex construction, parallel union-find via atomic compare-and-swap, parallel reduction for Euler characteristic counting, and parallel sort for filtration ordering. The sections below identify which components are genuinely GPU-accelerated, which are illustrative patterns, and which are currently CPU-only research problems.

---

## Table of Contents

1. [Topology for Graphics Programmers](#1-topology-for-graphics-programmers)
2. [Euler Characteristic and Genus Computation](#2-euler-characteristic-and-genus-computation)
3. [Simplicial Complexes on GPU](#3-simplicial-complexes-on-gpu)
4. [Persistent Homology](#4-persistent-homology)
5. [Discrete Morse Theory](#5-discrete-morse-theory)
6. [Reeb Graphs and Contour Trees](#6-reeb-graphs-and-contour-trees)
7. [Persistent Homology for Point Clouds](#7-persistent-homology-for-point-clouds)
8. [Handle and Tunnel Detection](#8-handle-and-tunnel-detection)
9. [Topological Persistence for Scalar Fields on Meshes](#9-topological-persistence-for-scalar-fields-on-meshes)
10. [Mapper Algorithm and TDA Visualisation](#10-mapper-algorithm-and-tda-visualisation)
11. [Topological Mesh Repair](#11-topological-mesh-repair)
12. [Topology-Aware Rendering and Visualisation](#12-topology-aware-rendering-and-visualisation)
- [Integrations](#integrations)

---

## 1. Topology for Graphics Programmers

Geometry asks questions about measurement: lengths, angles, areas, curvature. Topology asks questions about connectivity and shape that persist under continuous deformation — bending, stretching, and squashing, but not cutting or gluing. A cube, a sphere, and a tetrahedron are geometrically distinct but topologically equivalent: any one can be continuously deformed into any other. A torus cannot be deformed into a sphere, because it has a fundamentally different hole structure. This distinction — what survives continuous deformation — is what computational topology formalises and what this chapter's algorithms compute.

### 1.1 Key Topological Invariants

**Euler characteristic χ.** For a triangulated surface without boundary, χ = V − E + F where V = vertices, E = edges, F = triangular faces. This integer is a topological invariant: two surfaces that can be continuously deformed into each other share the same Euler characteristic. For a sphere, χ = 2. For a torus, χ = 0.

**Genus g.** For a closed, connected, orientable surface, the genus counts the number of handles (holes through the surface, like the hole of a torus). It relates to the Euler characteristic by:

```
g = (2 − χ) / 2
```

A sphere has g = 0; a torus has g = 1; a double torus (figure-8 surface) has g = 2.

**Betti numbers.** The Betti numbers β₀, β₁, β₂ give a richer description:
- β₀ = number of connected components
- β₁ = number of independent loops (1-dimensional holes, or "tunnels" through the surface)
- β₂ = number of enclosed voids (2-dimensional holes)

For a closed orientable surface: β₁ = 2g. For a closed surface with g handles and k connected components: χ = β₀ − β₁ + β₂ = k − 2g + k = 2k − 2g (using Poincaré duality β₀ = β₂ = k for a closed orientable surface). The relation χ = β₀ − β₁ + β₂ is the Euler–Poincaré formula, which holds for any finite simplicial complex.

**Orientability.** A surface is orientable if you can assign a consistent outward-facing normal everywhere. A Möbius strip is non-orientable. For graphics purposes, orientable meshes are the norm: non-orientable meshes cannot be consistently rendered with backface culling. Orientation consistency can be verified and repaired (§11).

**Manifold classification.** A 2-manifold (surface) is locally homeomorphic to a disk or a half-disk at its boundary. The classification theorem for compact surfaces states that every closed orientable surface is topologically a sphere with g handles. Non-manifold features — edges incident to more than two faces, or vertices whose star is not a disk — break this classification and are the primary targets of mesh repair algorithms.

### 1.2 Why Topology Matters in Graphics

| Application | Topological property needed |
|---|---|
| Mesh repair | Non-manifold vertex/edge detection, hole filling |
| UV unwrapping | Handle surgery to reduce genus before cutting seams |
| Skeleton extraction | Loop generators of H₁ give skeleton branches |
| Level-set tracking | Contour tree tracks topology changes as scalar threshold moves |
| Shape analysis | Betti numbers as rotation/scale-invariant descriptors |
| 3D scan cleaning | Persistent homology distinguishes true handles from scanning noise |
| Terrain analysis | Reeb graph encodes mountain ridges, valleys, passes |

---

## 2. Euler Characteristic and Genus Computation

### 2.1 Counting V, E, F in Parallel

Given a triangle mesh with a flat index buffer, computing χ requires counting vertices, edges, and faces in parallel.

**Face count** is trivially the number of triangles: F = index_count / 3, computed on the CPU before dispatch.

**Vertex count** V is the number of unique vertex indices — typically the size of the vertex buffer, assuming no unreferenced vertices.

**Edge count** is not directly encoded. Each triangle contributes three half-edges. Edges shared by exactly two triangles appear twice; boundary edges appear once. The standard GPU approach is to build a sorted edge list and deduplicate:

```glsl
// GLSL compute — emit half-edges
// Binding 0: index buffer (uint32, triangle list)
// Binding 1: output half-edge pairs buffer (uvec2, unordered)
layout(local_size_x = 256) in;

layout(std430, binding = 0) readonly buffer Indices { uint idx[]; };
layout(std430, binding = 1) writeonly buffer HalfEdges { uvec2 he[]; };

void main() {
    uint tri = gl_GlobalInvocationID.x;
    if (tri >= pc.num_triangles) return;
    uint i0 = idx[3 * tri + 0];
    uint i1 = idx[3 * tri + 1];
    uint i2 = idx[3 * tri + 2];
    // Canonicalise: store min in .x, max in .y so edge (a,b) == (b,a)
    he[3 * tri + 0] = uvec2(min(i0, i1), max(i0, i1));
    he[3 * tri + 1] = uvec2(min(i1, i2), max(i1, i2));
    he[3 * tri + 2] = uvec2(min(i2, i0), max(i2, i0));
}
```

After this dispatch, sort the uvec2 pairs with a GPU radix sort (e.g., `vk_radix_sort` for Vulkan or `thrust::sort` for CUDA), then count unique pairs with a parallel compaction pass using subgroup ballots or `subgroupElect()`. The unique count is E.

[Source: Vulkan Specification §9 — Compute Pipelines, https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html]

### 2.2 Genus from Euler Characteristic

```c
// CPU post-processing after GPU reduce returns V, E, F
int chi = (int)V - (int)E + (int)F;           // Euler characteristic
int boundary_edges = 0;                         // count edges with 1 incident face
// ... (read back from GPU or compute separately)
// For closed orientable surface:
int genus = (2 - chi) / 2;                     // requires chi even for valid closed surface
// For surface with b boundary components:
// chi = 2 - 2g - b  =>  g = (2 - chi - b) / 2
```

**Batched genus computation.** When processing a collection of meshes (e.g., instanced objects in a scene, or a mesh database), emit one sub-dispatch per mesh and use the mesh index as a key in the output buffer. A subgroup reduce accumulates per-mesh V, E, F:

```glsl
layout(local_size_x = 32) in;  // one subgroup per workgroup

// Each lane processes one element (vertex, edge, or face) of this mesh
// Use subgroupAdd to sum within subgroup, then globallyCoherent atomicAdd
subgroupBarrier();
uint laneV = /* 1 if this lane is a vertex element, else 0 */;
uint meshV = subgroupAdd(laneV);
if (subgroupElect()) atomicAdd(mesh_stats[mesh_id].V, meshV);
```

### 2.3 Non-Manifold Detection via Topology

The edge valence is a byproduct of the half-edge sort: edges with count > 2 in the sorted list are non-manifold edges. A single parallel scan suffices:

```c
// After GPU sort+compact: edge_counts[e] = number of incident triangles
// A valid manifold has edge_counts[e] in {1, 2} for all e
// Count non-manifold edges with GPU reduce:
uint nonmanifold_edges = thrust::count_if(
    edge_counts.begin(), edge_counts.end(),
    [] __device__ (uint c) { return c > 2; }
);
```

Non-manifold vertices (vertex star is not a disk) require checking that each vertex's adjacent triangles form a single fan with no gaps or multiple components. This is harder to parallelise and is typically done on CPU using a BFS over the local neighbourhood — or with GPU per-vertex fan sort as described in §11.

---

## 3. Simplicial Complexes on GPU

### 3.1 Representation

A simplicial complex K is a collection of simplices (vertices, edges, triangles, tetrahedra, …) that is closed under taking faces: if σ ∈ K and τ ⊆ σ, then τ ∈ K. For GPU processing, the simplices of each dimension are stored in separate flat arrays:

```c
// Flat GPU-friendly simplicial complex layout
struct SimplicialComplex {
    uint32_t *vertices;      // 0-simplices: indices into point buffer
    uint32_t *edges;         // 1-simplices: pairs (v0, v1), sorted
    uint32_t *triangles;     // 2-simplices: triples (v0, v1, v2)
    uint32_t *tetrahedra;    // 3-simplices: quadruples (v0, v1, v2, v3)
    float    *filtration;    // one filtration value per simplex (all dims interleaved)
    uint32_t  nv, ne, nt, ntet;
};
```

The filtration value attached to each simplex records when it enters the complex as the scale parameter ε grows — the core quantity for persistent homology.

### 3.2 Vietoris–Rips Complex Construction

The Vietoris–Rips complex of a point cloud P at scale ε includes all simplices whose vertices are pairwise within distance ε. Construction:

1. **All-pairs distance matrix** (GPU, O(n²)): Each thread (i,j) computes dist(p_i, p_j).
2. **Edge enumeration** (GPU): Emit edge (i,j) when dist < ε; use prefix scan for compaction.
3. **Triangle enumeration** (GPU): For each edge (i,j) and edge (j,k) with dist(i,k) < ε, emit triangle (i,j,k). This is O(n·d̄) where d̄ is average degree.

```glsl
// GLSL compute — all-pairs distances for Rips edge enumeration
layout(local_size_x = 16, local_size_y = 16) in;

layout(std430, binding = 0) readonly  buffer Points   { vec3 pts[];    };
layout(std430, binding = 1) readonly  buffer Params   { float eps; uint n; };
layout(std430, binding = 2) writeonly buffer EdgeFlag { uint flags[];  }; // 1 = edge exists
layout(std430, binding = 3) writeonly buffer Dists    { float dists[]; }; // filtration value

void main() {
    uint i = gl_GlobalInvocationID.x;
    uint j = gl_GlobalInvocationID.y;
    if (i >= n || j >= n || j <= i) return; // upper triangle only
    float d = distance(pts[i], pts[j]);
    uint linear = i * n + j;
    dists[linear]  = d;
    flags[linear]  = (d <= eps) ? 1u : 0u;
}
```

After the flag buffer is filled, `thrust::exclusive_scan` or a Vulkan compute prefix scan compacts flags into a dense edge list. The GUDHI library provides a Sparse Rips variant that achieves linear complex size with a (1+O(ε))-interleaved approximation, avoiding the O(n²) edge count of the dense Rips. [Source: GUDHI Rips Complex documentation, https://gudhi.inria.fr/doc/latest/group__rips__complex.html]

### 3.3 Alpha Complex

The Alpha complex is a Delaunay-filtered Rips complex: it includes simplex σ only when σ has an empty circumball of radius ≤ α. The GUDHI Alpha complex uses CGAL's Delaunay triangulation internally. For 3D data, GUDHI provides `Alpha_complex_3d` with FAST, SAFE, and EXACT kernel options. Filtration values are squared circumradii, with smaller values propagated to faces via the Gabriel condition:

```python
import gudhi
alpha_cx = gudhi.AlphaComplex(points=point_cloud)  # calls CGAL Delaunay
st = alpha_cx.create_complex(gudhi.SimplexTree())
st.compute_persistence()
```

[Source: GUDHI Alpha Complex documentation, https://gudhi.inria.fr/doc/latest/group__alpha__complex.html]

The GPU contribution to Alpha complex construction is computing the Delaunay triangulation (via a GPU Delaunay algorithm such as GpuDT or the method of Edelsbrunner–Shah implemented in CGAL). However, the filtration propagation step — iterating faces in order of decreasing dimension — is sequential in dimension and is typically done on CPU. Note: GPU Delaunay implementations are active research; CGAL's Alpha complex does not expose a GPU backend as of TTK 1.3.0.

### 3.4 Čech Complex

The Čech complex includes simplex σ when the smallest enclosing ball of σ's vertices has radius ≤ r. The smallest enclosing ball is the miniball — which equals the circumball only when the simplex is acute; for obtuse simplices the miniball is smaller than the circumball. GUDHI provides `Cech_complex` which uses the miniball algorithm for all simplex shapes. The Čech complex is topologically tighter than Rips (by the nerve lemma) but computationally more expensive. For GPU implementation, the per-triangle circumradius test is parallelisable:

```glsl
// GPU circumradius test for triangles (Čech 2-simplex inclusion)
void main() {
    uint tri = gl_GlobalInvocationID.x;
    vec3 a = pts[tri_i[3*tri+0]];
    vec3 b = pts[tri_i[3*tri+1]];
    vec3 c = pts[tri_i[3*tri+2]];
    // Circumradius formula for triangle in 3D
    vec3 ab = b - a, ac = c - a;
    float num = length(ab) * length(ac) * length(b - c);
    float denom = 2.0 * length(cross(ab, ac));
    float R = num / max(denom, 1e-10);
    filtration[tri] = R;
    flag[tri] = (R <= eps) ? 1u : 0u;
}
```

---

## 4. Persistent Homology

### 4.1 The Filtration and Persistence Diagram

Given a simplicial complex with a filtration (a scalar value on each simplex that is at least as large as the filtration value of each face), persistent homology tracks topological features as we sweep the filtration threshold from 0 to ∞. A connected component is *born* when its youngest vertex enters; it *dies* when it merges with an older component. A loop is born when an edge creates a cycle; it dies when a triangle fills the cycle.

The output is a **persistence diagram**: a multiset of points (birth, death) in the plane, one per topological feature. A feature that never dies is paired with ∞. Features far from the diagonal (long lifetime = death − birth) are topologically significant; features near the diagonal are interpreted as noise.

### 4.2 Boundary Matrix Reduction

The algebraic core of persistent homology is the reduction of the boundary matrix ∂ over a field (typically F₂ = {0,1}). The boundary matrix is an m×m matrix (m = total number of simplices) where column j represents the boundary of the j-th simplex (the sum of its codimension-1 faces). The standard reduction algorithm processes columns left to right, using Gaussian elimination to make each column either zero or have a unique low entry (the index of the lowest non-zero row). The pairs (low(j), j) give the persistence pairs.

**Standard algorithm complexity:** O(m³) worst case, O(m^ω) in practice. For Rips complexes on even modest point clouds, m can reach millions.

### 4.3 Ripser: Efficient CPU Persistent Homology

Ripser is the state-of-the-art CPU library for computing Vietoris–Rips persistence barcodes. Its key innovations:

- **Persistent cohomology** instead of homology: dual formulation reduces memory
- **Clearing optimisation**: if a simplex's coboundary reduces to zero, it need not be stored
- **Apparent pairs**: pairs of simplices (σ, τ) where σ is the cofacet of τ with the same cofacet order — these reduce immediately without pivoting and account for the vast majority of columns
- **Enclosing radius threshold**: once all features with lifetime > 0 are captured, computation terminates

Ripser's latest release (1.2.1, March 2021) outperforms prior tools by up to 40× in speed and 15× in memory. [Source: Ripser GitHub repository, https://github.com/Ripser/ripser]

```bash
# Ripser CLI: compute H₀ and H₁ of a distance matrix up to threshold 2.0
./ripser --format distance --dim 1 --threshold 2.0 distance_matrix.txt
# Output: persistence pairs (birth, death) for each dimension
```

### 4.4 Ripser++: GPU-Accelerated Rips Persistence

Ripser++ (simonzhang00/ripser-plusplus) extends Ripser with CUDA GPU acceleration. The key observation is that up to 99.9% of the columns in a cleared coboundary matrix correspond to apparent pairs, and apparent pair detection is embarrassingly parallel — each column can be tested independently. Ripser++ splits the computation:

- **GPU phase**: Filtration construction with clearing, and apparent pair extraction — massively parallelised with CUDA kernels
- **CPU phase**: Submatrix reduction for the small fraction of non-apparent columns

Performance results:
- Up to **30× total speedup** over Ripser
- Up to **2.0× CPU memory efficiency**
- Requires CUDA ≥ 10.1; recommended ≥ 20 GB GPU memory for large datasets
- On a 6 GB GPU: 15× speedup on sphere_3_192 dataset through dimension 3

[Source: Ripser++ GitHub repository, https://github.com/simonzhang00/ripser-plusplus]

```bash
# Build Ripser++ (requires CUDA 10.1+)
git clone https://github.com/simonzhang00/ripser-plusplus
cd ripser-plusplus && mkdir build && cd build
cmake -DCUDA_TOOLKIT_ROOT_DIR=/usr/local/cuda ..
make -j$(nproc)

# Run on a point cloud distance matrix
./ripser++ --format distance --dim 2 --threshold 1.5 distances.txt
```

### 4.5 Cubical Persistent Homology: CubicalRipser

For volumetric data (CT scans, voxel grids, density fields), the appropriate complex is a *cubical complex* — each voxel is a cube (3-cell), faces are squares (2-cells), edges are line segments (1-cells), and vertices are corners (0-cells). Cubical persistence avoids explicitly constructing and filtering a simplicial complex.

CubicalRipser (CubicalRipser/CubicalRipser_3dim) computes cubical persistence for 3D voxel data:
- Input: DIPHA format or Perseus format voxel data
- Output: persistence pairs as CSV or DIPHA
- Algorithm: adapts Ripser's approach to the cubical filtration; significantly faster than the general DIPHA tool for 2D and 3D data
- Two computation modes: `link_find` (default for H₀) and `compute_pairs`

[Source: CubicalRipser GitHub repository, https://github.com/CubicalRipser/CubicalRipser_3dim]

```bash
# CubicalRipser: 3D persistence for a voxel file
./CubicalRipser_3dim --output diagram.csv volume.csv
```

### 4.6 Parallel Boundary Matrix Reduction

The boundary matrix reduction algorithm has an inherent dependency: reducing column j may depend on having already reduced earlier columns that share the same low entry. However, sets of columns with distinct low entries are independent and can be reduced in parallel. Research implementations have exploited this via two strategies:

1. **Right-to-left sweeps with synchronisation:** Identify groups of independent columns at each step, process each group in parallel, then re-examine dependencies.
2. **Chunk-based column distribution:** Partition columns into chunks; within each chunk, apply reduction independently; merge boundaries between chunks.

Note: as of 2024–2025, these parallel boundary matrix reduction approaches are primarily research implementations (e.g., Oineus, Dory). They do not yet provide mature GPU backends; Ripser++ remains the production GPU choice. Note: needs verification of current GPU boundary matrix reduction production status.

### 4.7 GUDHI Python API for Persistence

GUDHI provides a unified Python API that chains complex construction with persistence computation:

```python
import gudhi
import numpy as np

# Build Rips complex from point cloud
points = np.random.rand(100, 3)
rips = gudhi.RipsComplex(points=points, max_edge_length=1.0)
st = rips.create_simplex_tree(max_dimension=2)

# Compute persistence
st.compute_persistence()
# Retrieve H0 and H1 pairs
pairs = st.persistence()
for dim, (birth, death) in pairs:
    print(f"H{dim}: [{birth:.3f}, {death:.3f})")

# Export persistence diagram
gudhi.plot_persistence_diagram(st.persistence())
```

[Source: GUDHI documentation, https://gudhi.inria.fr/doc/latest/]

---

## 5. Discrete Morse Theory

### 5.1 Forman's Discrete Morse Theory

Forman's discrete Morse theory is a combinatorial analogue of classical smooth Morse theory, adapted to simplicial and CW complexes. A **discrete Morse function** f assigns a real value to each simplex of dimension k such that for almost all k-simplices σ:
- at most one codimension-1 face τ of σ satisfies f(τ) ≥ f(σ)
- at most one coface ρ of σ of dimension k+1 satisfies f(ρ) ≤ f(σ)

A simplex that violates neither condition is **critical**. Non-critical simplices are paired: each critical-free k-simplex σ is paired with either a face or a coface to form a V-pair (σ, τ). These pairs form a **gradient vector field** on the complex.

The critical simplices play the role of classical critical points: a critical 0-simplex is a local minimum (connected component birth), a critical 1-simplex is a saddle (loop or tunnel), and a critical 2-simplex is a local maximum (void closure). The count of critical simplices of dimension k bounds the k-th Betti number from above: this is the weak Morse inequality.

**Significance for graphics:** Discrete Morse theory gives a way to reduce a mesh to its topological skeleton by collapsing all V-pairs, leaving only critical simplices. The result is a smaller, topologically equivalent complex — useful for mesh simplification that preserves topology.

### 5.2 Computing a Gradient Vector Field

A standard algorithm (PairCells, or the iterative approach based on Banchoff's method) processes simplices in increasing order of their assigned function value and greedily assigns V-pairs:

```
for each unpaired simplex σ in increasing f-order:
    find cofacet ρ with f(ρ) ≤ f(σ) and ρ unpaired  // violates standard condition
    if found: create V-pair (σ, ρ), mark both paired
    else: mark σ critical
```

The canonical discrete Morse function for a triangle mesh is the function that assigns the maximum vertex value of each simplex — this is the *lower star filtration* and leads to a gradient vector field compatible with the scalar field.

### 5.3 GPU Gradient Pairing

GPU parallelisation of discrete Morse gradient pairing is an active research area. The sequential dependency of the greedy algorithm (each decision affects future availability) limits naive GPU parallelism. Practical approaches:

**Independent set processing:** In each round, identify the set of simplices that can be paired without conflicting with any other pairing decision in the same round (a matching in the Hasse diagram). Process all pairs in this independent set simultaneously. Iterate until convergence.

```glsl
// GPU pass: mark tentative V-pairs for this round
// Each simplex thread checks if its preferred cofacet is also preferring it
layout(local_size_x = 256) in;

layout(std430, binding = 0) readonly  buffer FValues  { float fval[];    };
layout(std430, binding = 1) readwrite buffer Paired   { uint  paired[];  }; // 0=free, else index
layout(std430, binding = 2) readonly  buffer Cofacets { uint  cofacet[]; }; // precomputed

void main() {
    uint sigma = gl_GlobalInvocationID.x;
    if (sigma >= pc.num_simplices) return;
    if (paired[sigma] != 0u) return; // already paired or critical

    uint rho = cofacet[sigma]; // best cofacet (lowest f-value > f[sigma])
    if (rho == ~0u) return;    // no valid cofacet -> critical
    if (paired[rho] != 0u) return;

    // Propose pairing only if rho's preferred face is sigma (symmetric preference)
    // This implements a stable matching to avoid conflicts
    uint sigma_pref = best_face_of[rho];
    if (sigma_pref == sigma) {
        // Atomic: only one thread can pair rho (use CAS)
        // atomicCompSwap returns the *old* value — compare that to confirm the swap landed
        uint old = atomicCompSwap(paired[rho], 0u, sigma + 1u); // +1 so 0 means unpaired
        if (old == 0u) paired[sigma] = rho + 1u;               // confirmed: we won the race
    }
}
```

Multiple rounds are needed; convergence is typically fast in practice. Note: production GPU discrete Morse implementations (e.g., in TTK or as standalone tools) are not yet widely deployed; this pattern is representative of the approach taken in research prototypes.

### 5.4 Applications: Skeleton Extraction and Segmentation

Once a gradient vector field is computed, the **Morse complex** is obtained by tracing gradient paths between critical simplices:
- An *unstable manifold* of a critical 0-simplex (minimum) is a connected component of the final mesh
- A *stable manifold* of a critical 2-simplex (maximum) is a patch of the surface bounded by saddles and maxima
- The Morse–Smale complex partitions the surface into cells, each being the intersection of an unstable and a stable manifold

TTK implements the Morse–Smale complex computation as part of its `MorseSmaleComplex` module, running on CPU with multi-core parallelism. [Source: TTK Topology ToolKit, https://topology-tool-kit.github.io/]

**Topological simplification** cancels pairs of critical simplices: if a minimum m and a saddle s are paired in the persistence diagram (m survives until s merges it into another component), and the persistence death − birth is below a threshold, cancellation removes both from the critical set by modifying the gradient vector field. This is the main tool for smoothing noisy scalar fields while preserving topologically significant features.

---

## 6. Reeb Graphs and Contour Trees

### 6.1 Reeb Graph Definition

Given a continuous scalar function f: M → ℝ on a manifold M, the **Reeb graph** is the quotient space obtained by collapsing each connected component of each level set f⁻¹(c) to a single point. On a simply-connected domain (no loops), the Reeb graph is a tree, called the **contour tree**.

For a triangle mesh with a scalar field on vertices (height, curvature, signed distance function, etc.), the contour tree encodes how the topology of the level sets changes as the threshold sweeps from min to max:
- A leaf corresponds to a local extremum (a new component appears or disappears)
- An internal node corresponds to a merge event (two components become one) or split event (one splits into two)

The contour tree is used for interactive visualisation, shape decomposition, terrain analysis, and molecular surface analysis.

### 6.2 Contour Tree Algorithm

The standard contour tree algorithm — "Computing contour trees in all dimensions," ACM SODA 2000, implemented in TTK's `ContourTree` class — combines two simpler structures:

1. **Join tree (merge tree):** Process vertices in increasing f-value order; track connected components using union-find as edges are added. Each union event creates an internal node.
2. **Split tree:** Same, but in decreasing f-value order (or equivalently, the join tree of −f).
3. **Combine:** Merge the join and split trees to obtain the full contour tree.

[Source: TTK ContourTree class documentation, https://topology-tool-kit.github.io/doc/html/classttk_1_1ContourTree.html]

### 6.3 GPU Parallel Contour Tree

The join tree computation reduces to a connected-component labelling problem, which maps well to GPU parallelism. The key GPU primitive is **path-compressed union-find** with atomic compare-and-swap:

```glsl
// GPU path-compressed union-find for join tree construction
// Process vertices in batches sorted by f-value
// For each new vertex v being added, link its edges to already-active vertices

layout(std430, binding = 0) readwrite buffer Parent  { uint parent[]; };
layout(std430, binding = 1) readonly  buffer FValues { float fval[];  };

// Find root with path compression (iterative, GPU-safe)
uint find(uint x) {
    while (parent[x] != x) {
        uint grandparent = parent[parent[x]];
        parent[x] = grandparent;   // path halving
        x = grandparent;
    }
    return x;
}

// Union by linking the root of the younger component to the older root
void union_sets(uint u, uint v) {
    uint ru = find(u), rv = find(v);
    if (ru == rv) return;
    // Link younger (higher f-value) root to older root
    if (fval[ru] > fval[rv]) atomicCompSwap(parent[ru], ru, rv);
    else                      atomicCompSwap(parent[rv], rv, ru);
}
```

Processing all edges whose lower endpoint is the current vertex batch, then calling union_sets in parallel, yields the join tree. A synchronisation barrier between batches is needed to enforce the f-ordering.

TTK's MPI extension (TTK-MPI, arxiv 2310.08339) distributes the contour tree algorithm across cluster nodes, achieving parallel efficiencies of 20–80% depending on the algorithm, and has been tested on datasets up to 120 billion vertices using 64 nodes (1,536 cores). [Source: TTK-MPI technical report, https://arxiv.org/abs/2310.08339]

### 6.4 TTK API for Contour Trees and Reeb Graphs

```cpp
// TTK C++ API for contour tree computation
#include <ContourTree.h>

ttk::ContourTree ct;
ct.setVertexNeighbors(/*mesh adjacency*/);
ct.setInputDataPointer(0, scalarValues.data());
ct.build();

// Retrieve persistence pairs (used for simplification)
std::vector<std::tuple<int,int,double>> pairs;
ct.getPersistencePairs(pairs);

// Simplify: remove features below 10% of the scalar range
ct.simplify(0.1, /*metric*/ ttk::ContourTree::SaddleConnectors);
ct.computeSkeleton();
```

[Source: TTK 1.3.0, https://topology-tool-kit.github.io/]

TTK also provides `ReebGraph` for domains with loops (g > 0), and the `FTMTree` module for both merge trees and Reeb graphs. All TTK modules integrate as ParaView plugins, enabling visual inspection alongside GPU-rendered scalar fields.

### 6.5 Applications

**Terrain analysis:** The contour tree of a height field over a terrain encodes ridgelines (split nodes), valley junctions (merge nodes), and passes (saddle-like splits that immediately re-merge). Interactive threshold sliders in ParaView/TTK let users explore which terrain regions are connected at each elevation.

**Molecular surfaces:** Level sets of the Gaussian density function around a protein molecule change topology as the isovalue varies — new pockets open and cavities close. The contour tree guides visual exploration without exhaustive per-isovalue rendering.

**Shape decomposition:** Partitioning a mesh by contour tree arcs (each arc = one topological region between consecutive critical values) gives a segmentation that is intrinsically meaningful for shape matching and part-based retrieval.

---

## 7. Persistent Homology for Point Clouds

### 7.1 H₀: Connected Components via Union-Find

The zeroth persistent homology group H₀ tracks connected components as the filtration scale ε grows. This is equivalent to running a minimum spanning tree (MST) computation on the complete graph with edge weights equal to pairwise distances, and reading off the merge events in order.

On GPU, this maps to a parallel union-find with CAS:

```glsl
// GPU H0 persistence via sorted edge processing
// Sort edges by distance; process in order, recording merge events
layout(local_size_x = 256) in;

layout(std430, binding = 0) readonly  buffer SortedEdges { uvec2 edges[]; float dists[]; };
layout(std430, binding = 1) readwrite buffer Parent      { uint  parent[]; };
layout(std430, binding = 2) writeonly buffer Events      { vec2  pairs[];  }; // (birth=0, death=d)
layout(std430, binding = 3) readwrite buffer BirthTime   { float birth[];  }; // birth[root]=min vertex filtration

void main() {
    // Sequential in edge order — process in sorted batches
    // Each thread handles one edge at its assigned sort position
    uint e = gl_GlobalInvocationID.x;
    if (e >= pc.num_edges) return;
    uint u = edges[e].x, v = edges[e].y;
    float d = dists[e];
    uint ru = find(u), rv = find(v);
    if (ru != rv) {
        // Merge younger into older component
        float bu = birth[ru], bv = birth[rv];
        // Younger component dies at distance d; record pair
        uint younger = (bu > bv) ? ru : rv;
        uint older   = (bu > bv) ? rv : ru;
        if (atomicCompSwap(parent[younger], younger, older) == younger) {
            pairs[e] = vec2(bu > bv ? bu : bv, d); // (max_birth, death=d)
        }
    }
}
```

Note: strict sequential order within a sorted edge sequence requires batched processing with synchronisation barriers between batches of equal-distance edges.

### 7.2 H₁: Loops via Boundary Matrix Reduction

First homology H₁ counts independent loops. Persistence of H₁ tracks when loops form (birth: a cycle-creating edge enters) and when they fill (death: a triangle eliminates the loop). Computing H₁ persistence requires the boundary matrix reduction of §4.2, which is the computationally expensive step.

In practice for point cloud TDA, Ripser++ is used for H₁ and H₂ computation (up to user-specified dimension). The GPU handles apparent pair extraction; non-apparent columns go to CPU.

### 7.3 Persistence Representations for Machine Learning

A persistence diagram is a set of unordered points in ℝ², making it incompatible with standard vector-space machine learning. Several representations vectorise it:

**Persistence image:** Map each point (b,d) to a Gaussian kernel centred at (b, d−b) in the (birth, persistence) half-plane; sum all Gaussians on a grid. The resulting image is differentiable with respect to the Gaussian bandwidth and weight function. [Source: GUDHI persistence representations, https://gudhi.inria.fr/doc/latest/]

**Persistence landscape:** The k-th landscape function λₖ(t) = max of tent functions peaked at each (b,d) midpoint, k-th largest. Stable and vectorisable.

**Betti curve:** For each scale t, the value β₁(t) is just the count of alive H₁ intervals. A simple 1D curve.

### 7.4 Wasserstein and Bottleneck Distances

Comparing two persistence diagrams D₁, D₂ uses optimal transport:

- **Bottleneck distance** W∞: the minimum over all bijections σ: D₁ → D₂ of the maximum point distance max_{x∈D₁} ||x − σ(x)||∞ (with diagonal points as free matches).
- **Wasserstein distance** Wₚ: replace max by (Σ ||x − σ(x)||^p)^(1/p).

GUDHI provides bottleneck_distance() and wasserstein_distance() between two persistence diagrams. For GPU acceleration of optimal transport computation, approximate solvers (Sinkhorn algorithm) can compute Wₚ distances between many diagram pairs in batch — this maps to matrix operations on GPU and is useful for comparing a large dataset of shape diagrams simultaneously. Note: as of GUDHI 3.x, the C++ bottleneck and Wasserstein implementations are CPU-only; GPU Sinkhorn for diagram distance is a research integration point.

[Source: GUDHI documentation, https://gudhi.inria.fr/doc/latest/]

### 7.5 Application: Denoising 3D Scan Data

A 3D scanner of a physical object with genus 0 (sphere topology) should produce a mesh with β₁ = 0. Any detected H₁ features in the persistence diagram are topological noise from scanning artefacts. Features close to the diagonal (small persistence) are classified as noise and can be eliminated by topological simplification (§9.4). Features far from the diagonal indicate real topology changes that the scanner erroneously introduced (handles that should be filled). This analysis guides mesh repair (§11).

---

## 8. Handle and Tunnel Detection

### 8.1 Handles and Their Homological Meaning

A *handle* on a surface is a topological feature that contributes to the genus. A genus-g surface has g handles, each contributing 2 to β₁: one loop running around the handle (the *longitude*) and one running through it (the *meridian*). Detecting handles is prerequisite for UV-seam cutting and mesh repair.

A *tunnel* in the context of 3D meshes typically refers to a handle loop that passes through the enclosed volume — relevant for medical mesh analysis (blood vessel networks, trabecular bone architecture).

### 8.2 Generators of H₁: Shortest Non-Contractible Cycles

Given that H₁ is non-trivial (β₁ > 0), the generators of H₁ can be computed as the shortest non-contractible cycles — geodesic loops on the surface that cannot be contracted to a point.

The algorithm is:

1. For each basis element of H₁, find the shortest cycle in its homology class.
2. Use Dijkstra's algorithm on the primal mesh graph, but track the homology class of each path using an assignment of signs to edges (a Z₂ labelling compatible with the dual spanning tree).

The dual spanning tree approach:
1. Build a spanning tree T of the primal mesh graph.
2. The co-tree edges (edges not in T) correspond to independent generators of H₁.
3. For each co-tree edge e = (u,v), the unique cycle in T ∪ {e} is a generator of H₁.
4. Run Dijkstra from u to v on T to find the tree path; the cycle is that path plus the edge e.

**GPU shortest path.** Dijkstra's algorithm on a mesh graph (|V| vertices, |E| edges) can be parallelised via Bellman-Ford relaxation. Each iteration relaxes all edges in parallel:

```glsl
// GPU Bellman-Ford for shortest non-contractible cycle
// One iteration: relax all edges in parallel
layout(local_size_x = 256) in;

layout(std430, binding = 0) readonly  buffer Edges   { uvec2 edges[]; float wt[]; };
layout(std430, binding = 1) readwrite buffer Dist    { float dist[];   };
layout(std430, binding = 2) readwrite buffer Updated { uint  updated;  }; // flag

void main() {
    uint e = gl_GlobalInvocationID.x;
    if (e >= pc.num_edges) return;
    uint u = edges[e].x, v = edges[e].y;
    float d = dist[u] + wt[e];
    if (d < dist[v]) {
        // Atomic min via uint bit reinterpretation (works for positive floats)
        uint dBits = floatBitsToUint(d);
        uint old = atomicMin(/* dist[v] as uint */ reinterpret_cast_uint(dist[v]), dBits);
        if (dBits < old) atomicOr(updated, 1u);
    }
}
```

Bellman-Ford converges in |V| − 1 iterations in the worst case but typically much sooner for meshes. For large meshes, a GPU priority-queue Dijkstra (using atomic operations on a key array) is faster; implementations are available in cuGraph and graph processing toolkits.

### 8.3 Harmonic 1-Forms and Handle Detection

A more algebraic approach to handle detection uses **harmonic 1-forms** — differential 1-forms on the mesh that are closed (d ω = 0) and co-closed (d⋆ω = 0). On a genus-g surface, there are 2g linearly independent harmonic 1-forms, one for each generator of H₁. Computing them requires solving a sparse linear system:

```
L x = b   (discrete Laplacian, with appropriate boundary conditions)
```

where L is the cotan-weight Laplacian matrix. A non-trivial solution indicates a handle. GPU sparse linear system solvers (cuSPARSE, AMGX) can accelerate this, especially for large meshes.

Note: the full pipeline — building the cotan-Laplacian on GPU, solving it with a GPU Krylov solver, and extracting the harmonic 1-form basis — is feasible using cuSPARSE + CUSOLVER but requires careful setup of boundary constraints. For practical mesh repair, counting genus via the Euler characteristic (§2) is sufficient to determine the number of handles without computing their generators.

### 8.4 Application: UV Seam Loop Detection

UV unwrapping a genus-g surface requires cutting 2g seam loops to flatten the surface to a disk. Handle detection algorithms identify candidate seam loops:
1. Compute β₁ via Euler characteristic to count required seam loops.
2. Find the 2g shortest non-contractible cycles (§8.2).
3. Mark these cycles as seam edges, then cut and unfold.

Tools like libigl's `cut_mesh` and CGAL's `Surface_mesh_parameterization` implement this pipeline with CPU algorithms; the GPU contribution is accelerating the shortest path computation for large meshes.

---

## 9. Topological Persistence for Scalar Fields on Meshes

### 9.1 Lower-Star Filtration

Given a scalar field f: V → ℝ on mesh vertices, the **lower-star filtration** builds the complex by adding each vertex v and all simplices (edges, triangles) whose maximum vertex value equals v, in increasing order of f(v). This defines a filtration of the full mesh and enables computing the persistence of f.

The persistence diagram of f identifies critical points (local minima, saddles, local maxima) paired by their topological significance. This is equivalent to computing the contour tree and reading off its persistence pairs.

### 9.2 GPU Sort and Scan-Based Union-Find

The H₀ persistence of the lower-star filtration is a sorted union-find computation. On GPU:

1. **Parallel sort** of vertices by f-value (radix sort: `thrust::sort_by_key`)
2. **Sorted edge processing** in f-order batches (§7.1)
3. **Per-vertex union-find** with atomic CAS

The H₁ persistence (loop tracking) requires also tracking when triangles fill cycles — this needs the boundary matrix approach and is handled by Ripser or by TTK's `PersistenceDiagram` module.

### 9.3 Topological Simplification of Scalar Fields

Given a persistence diagram of a scalar field, **topological simplification** removes features below a threshold ε by modifying the scalar field values at critical points. For a pair (minimum m at value b, saddle s at value d) with persistence d − b < ε, set f(m) = d and f(s) = d (raising the minimum to the saddle height eliminates the feature).

TTK implements topological simplification in its `TopologicalSimplification` module:

```cpp
// TTK C++ API for topological simplification
#include <TopologicalSimplification.h>

ttk::TopologicalSimplification ts;
ts.setInputDataPointer(0, scalarValues.data());
ts.setConstraintIdentifiers(criticalPoints.data(), criticalPoints.size());
ts.setInputOffsets(offsets.data());
ts.execute();
// scalarValues is now modified to remove features below threshold
```

[Source: TTK 1.3.0, https://topology-tool-kit.github.io/]

### 9.4 Application: Smoothing Noisy Curvature Fields

Mean curvature on a triangle mesh is noisy, with many small local extrema. Topological simplification of the curvature field removes extrema with persistence below a user-chosen threshold, yielding a smoother curvature for segmentation and shape analysis — without the blurring introduced by Gaussian smoothing, since topologically significant curvature features are preserved exactly.

Workflow:
1. Compute mean curvature per vertex (GPU: cotan-Laplacian evaluation, one thread per vertex)
2. Run TTK `PersistenceDiagram` on CPU to get curvature persistence pairs
3. Apply TTK `TopologicalSimplification` to remove pairs below threshold
4. Upload simplified curvature to GPU for segmentation rendering

---

## 10. Mapper Algorithm and TDA Visualisation

### 10.1 The Mapper Construction

The Mapper algorithm (introduced in "Topological Methods for the Analysis of High Dimensional Data Sets and 3D Object Recognition," SPBG 2007) produces a graph summary of a high-dimensional dataset by:

1. Choose a **lens function** f: X → ℝ (or ℝ²) that maps each data point to a low-dimensional value (could be a coordinate, eccentricity, density, or a neural network embedding).
2. Cover the range of f with overlapping intervals I₁, I₂, …, Iₙ.
3. For each interval Iᵢ, extract the preimage f⁻¹(Iᵢ) (the subset of points mapping into that interval).
4. Cluster each preimage using any clustering algorithm (k-means, DBSCAN, single-linkage, …).
5. Build the **nerve graph**: one node per cluster, one edge between two clusters from adjacent intervals if they share a data point.

The resulting graph (the "Mapper graph") summarises the topological shape of the dataset. Loops in the graph correspond to H₁ features; connected components to H₀.

### 10.2 Kepler Mapper: Python Implementation

Kepler Mapper is the reference Python implementation of the Mapper algorithm, compatible with scikit-learn clustering algorithms:

```python
import kmapper as km
import sklearn

mapper = km.KeplerMapper(verbose=1)

# Project high-dimensional data to 1D lens
data = ... # shape (N, D)
lens = mapper.fit_transform(data, projection=sklearn.manifold.TSNE())

# Build the Mapper graph
graph = mapper.map(
    lens,
    data,
    cover=km.Cover(n_cubes=10, perc_overlap=0.5),
    clusterer=sklearn.cluster.DBSCAN(eps=0.5, min_samples=3)
)

# Visualise as interactive HTML
mapper.visualize(graph, path_html="mapper_output.html",
                 title="Dataset Topology")
```

[Source: Kepler Mapper documentation, https://kepler-mapper.scikit-tda.org/en/latest/]

### 10.3 GPU Mapper Acceleration Prospects

The two computationally intensive steps in Mapper are clustering (within each interval's preimage) and distance computation between points. Both are GPU-parallelisable:

**GPU clustering:** Each interval's preimage is an independent DBSCAN or k-means problem. Multiple small clustering jobs can be dispatched simultaneously, with one workgroup per interval. GPU DBSCAN implementations (cuML DBSCAN from RAPIDS) apply here.

**GPU all-pairs distance for lens:** If the lens requires pairwise distances (e.g., for TSNE or diffusion maps), the O(n²) distance matrix computation is a GPU standard.

The **nerve construction** (finding which clusters share points) requires set intersection and is irregular in both structure and load. For small Mapper graphs this is fast on CPU; for very large datasets (n > 10⁵, many intervals) a GPU bitmap intersection approach is feasible.

Note: as of 2025, no production GPU Mapper implementation exists. The acceleration prospects above are research-stage proposals; GPU clustering libraries (cuML) are the best current path to accelerating Mapper's most expensive step.

### 10.4 GUDHI Mapper/Cover Complex

GUDHI provides a `MapperComplex` (also called CoverComplex or GraphInducedComplex) that implements Mapper-like constructions. The following illustrates the conceptual API — verify exact method names against the GUDHI cover complex documentation before use:

```python
# Illustrative pseudocode — verify exact API against GUDHI docs
import gudhi.cover_complex as cc

nerve = cc.MapperComplex()
nerve.set_input_points(points)     # Note: verify method name
nerve.set_colors(lens_values)      # Note: verify method name
nerve.set_cover(...)               # Cover with overlapping intervals
nerve.compute_complex()
nerve.save_to_file("mapper.txt")
```

[Source: GUDHI documentation, https://gudhi.inria.fr/doc/latest/]

---

## 11. Topological Mesh Repair

### 11.1 Non-Manifold Detection

A mesh is **2-manifold** if every edge is incident to exactly 1 or 2 triangles, and every vertex's star (union of incident triangles) is a disk (or half-disk at the boundary). Violations are:

- **Non-manifold edges:** incident to 3 or more triangles (T-intersections, self-intersections)
- **Non-manifold vertices:** star is not a disk (e.g., two cones meeting at a tip)

GPU non-manifold edge detection uses the edge valence count from §2.2. For non-manifold vertex detection, the approach is:

1. Sort all (vertex, triangle) incidence pairs by vertex ID (GPU radix sort).
2. For each vertex, collect its incident triangles in a subarray.
3. Build an adjacency graph of the triangles (two triangles adjacent if sharing an edge in the vertex's star).
4. Check that this adjacency graph is connected and forms a single cycle (for interior vertices) or a single path (for boundary vertices).

Step 3–4 is the fan connectivity check. On GPU, this can be parallelised per vertex with subgroup operations, but for large vertex valences it falls back to per-vertex CPU processing.

```glsl
// GPU: count edge valences using atomicAdd into per-edge counter
void main() {
    uint tri = gl_GlobalInvocationID.x;
    if (tri >= pc.num_triangles) return;
    uint i0 = idx[3*tri+0], i1 = idx[3*tri+1], i2 = idx[3*tri+2];
    // Use a hash of the canonical edge pair as the atomic counter key
    // (In practice: sort half-edges, then count runs)
    atomicAdd(edge_count[edge_hash(i0, i1)], 1u);
    atomicAdd(edge_count[edge_hash(i1, i2)], 1u);
    atomicAdd(edge_count[edge_hash(i2, i0)], 1u);
}
```

### 11.2 Orientation Consistency

A manifold mesh should have consistent triangle winding (all outward normals computed by the same convention). Inconsistent orientation is detected by finding edges where the two incident triangles traverse the edge in the same direction (they should traverse in opposite directions for a consistently oriented mesh).

**GPU BFS repair:**

1. Initialise a queue with one "correctly oriented" seed triangle.
2. In each BFS wave, for each triangle in the current wavefront, check each adjacent triangle through each shared edge.
3. If the adjacent triangle traverses the shared edge in the same direction as the current triangle, flip it (reverse its winding order).
4. Add unvisited adjacent triangles to the next wave.

BFS wavefronts can be processed in parallel (all triangles in the same BFS level are independent):

```glsl
// GPU BFS wave: propagate orientation from current_wave to next_wave
void main() {
    uint t = gl_GlobalInvocationID.x;
    if (t >= pc.wave_size) return;
    uint tri = current_wave[t];
    for (uint e = 0; e < 3; ++e) {
        uint adj = adjacency[3*tri + e];
        if (adj == ~0u || visited[adj]) continue;
        if (same_direction(tri, adj, e)) flip(adj); // flip winding
        if (atomicExchange(visited[adj], 1u) == 0u)
            atomicAdd(next_wave_count, 1u); // enqueue adj
    }
}
```

Multiple connected components each need their own BFS seed; β₀ (connected components) determines how many seeds are needed.

### 11.3 Boundary Loop Detection and Hole Filling

Boundary edges (valence 1) form loops. Finding the boundary loops is equivalent to finding connected components of the boundary edge graph. This is another union-find computation:

```c
// CPU: find boundary loops after GPU returns boundary edge list
// boundary_edges: sorted list of (v0, v1) pairs with valence == 1
std::vector<int> parent(num_vertices);
std::iota(parent.begin(), parent.end(), 0);
for (auto [u, v] : boundary_edges) union_sets(parent, u, v);
// Each connected component of parent is one boundary loop
```

Once boundary loops are found, hole filling proceeds by applying Liepa's minimum-area triangulation algorithm (iterative local triangle insertion that minimises area or dihedral angle) for each loop. This is a CPU operation; the filled triangles are then uploaded as additional mesh data.

Production tools:
- `MeshFix`: fully automatic manifold repair tool
- `libigl::is_edge_manifold()`, `libigl::boundary_loop()`: detection utilities
- `CGAL::Polygon_mesh_processing::stitch_borders()`, `is_valid_polygon_mesh()`: CGAL repair pipeline

[Source: libigl tutorial, https://libigl.github.io/tutorial/]

### 11.4 Handle Removal via Topological Surgery

If the goal is to reduce genus (e.g., reduce a genus-2 object to genus-0 for simpler UV parameterisation), handle surgery proceeds:
1. Detect handle loop generators (§8.2).
2. For each handle loop, cut along the loop (duplicate vertices along the seam).
3. Fill the two resulting boundary loops with new triangles (Liepa fill or fan fill).

This reduces genus by 1 per handle removed, at the cost of introducing new triangles and seam edges. The result is topologically equivalent to a lower-genus surface.

---

## 12. Topology-Aware Rendering and Visualisation

### 12.1 Filtering Marching Cubes Output by Topology

GPU marching cubes (Ch210 §2) extracts isosurfaces from volumetric data. For noisy data, the isosurface may have many small disconnected components (small β₀ > 1) and artificial handles. Topology-aware post-processing:

1. Run GPU marching cubes → triangle soup
2. **GPU connected component labelling** (union-find on the output mesh adjacency) → component sizes
3. **Filter by component size:** keep only components whose β₀-contribution is "real" (e.g., above a minimum triangle count threshold)
4. Alternatively, compute the persistence diagram of the volumetric scalar field using CubicalRipser, and discard isosurface patches corresponding to low-persistence features

This avoids the visual clutter of topological noise without modifying the underlying scalar field.

### 12.2 Topological LOD

Level-of-detail schemes that preserve topology — no change in Betti numbers — as the mesh is coarsened maintain the "correct" visual shape. Standard mesh simplification (quadric error metrics, edge collapse) does not guarantee topology preservation: collapsing an edge adjacent to a handle can change β₁.

Topology-preserving simplification uses:
- The **link condition** for edge collapse: an edge (u,v) can be collapsed only if the link of u intersected with the link of v equals the link of the edge (u,v). This is a necessary and sufficient condition for the collapse to preserve topology.
- Implementations: CGAL `Surface_mesh_simplification` with the link condition check; libigl `decimate` with manifold preservation.

For GPU-accelerated topology-preserving simplification, a parallel edge collapse scheme tests the link condition for independent (non-adjacent) edges simultaneously and collapses independent safe edges in each GPU pass. Note: production GPU implementations of topology-preserving simplification are not yet common; this is an active research direction.

### 12.3 Visualising Persistence Diagrams in Vulkan

A persistence diagram is a set of 2D points — birth on one axis, death on the other — rendered as a scatter plot with a diagonal reference line.

```glsl
// Vertex shader for persistence diagram points
// SSBO contains vec2 (birth, death) for each persistence pair
layout(location = 0) out vec4 pointColour;

layout(std430, set = 0, binding = 0) readonly buffer PairsBuffer {
    vec2 pairs[]; // (birth, death)
};

void main() {
    vec2 p = pairs[gl_VertexIndex];
    float persistence = p.y - p.x; // death - birth
    // Map (birth, death) to NDC [-1, 1]
    gl_Position = vec4(p.x * 2.0 - 1.0, p.y * 2.0 - 1.0, 0.0, 1.0);
    gl_PointSize = clamp(persistence * 20.0, 2.0, 12.0);
    // Colour by dimension (H0=blue, H1=red, H2=green)
    pointColour = vec4(0.8, 0.2, 0.2, 1.0); // H1 example
}
```

The diagonal is drawn as a single line from (−1, −1) to (1, 1) using `VK_PRIMITIVE_TOPOLOGY_LINE_LIST`. Points above the diagonal are legitimate pairs; points on the diagonal are zero-persistence (noise).

### 12.4 TTK ParaView Pipeline for Topological Features

TTK 1.3.0 provides ParaView plugins for all its modules, enabling an interactive pipeline:

```
vtkDataSet → TTK PersistenceDiagram → vtkUnstructuredGrid (pairs as edges)
           → TTK TopologicalSimplification → simplified scalar field
           → TTK ContourTree → vtkPolyData (tree arcs as tubes)
           → ParaView vtkProperty → Vulkan render (via VTK's Vulkan backend)
```

[Source: TTK Topology ToolKit, https://topology-tool-kit.github.io/]

ParaView renders the TTK output geometry on the GPU after the CPU-side topological analysis, using its VTK rendering backend (OpenGL by default; Note: a Vulkan backend for VTK/ParaView is under active development — verify current status against the VTK release notes before assuming Vulkan availability). This hybrid pipeline — CPU topology, GPU render — is the current production approach.

### 12.5 Visualising Reeb Graphs and Contour Trees

Reeb graphs and contour trees are rendered as geometric primitives:
- **Nodes** as sphere impostors or point sprites at the geometric centroid of their corresponding level-set component
- **Arcs** as tube geometry (parametric cylinder along the arc path)
- **Arc width or colour** encoding persistence, arc length, or a topological attribute

In Vulkan, the dynamic nature of Reeb graph geometry (arcs appear/disappear as the isovalue slider moves) is handled by keeping the graph in an SSBO and rebuilding the draw indirect buffer each frame based on the current threshold:

```c
// Vulkan: indirect draw for visible Reeb graph arcs
VkDrawIndirectCommand cmd = {
    .vertexCount   = visible_arc_count * 2,  // 2 endpoints per segment
    .instanceCount = 1,
    .firstVertex   = 0,
    .firstInstance = 0,
};
vkCmdDrawIndirect(cmdbuf, indirect_buf, 0, 1, sizeof(cmd));
```

---

## Integrations

**Ch208–212 (GPU Geometry Algorithms series).** The topology algorithms in this chapter operate on the same triangle mesh and volumetric data representations used throughout the Geometry Algorithms series. Mesh V/E/F counting (§2) reuses the index buffer layout described in Ch208. Marching cubes output (Ch210 §2) feeds directly into the topology-aware filtering of §12.1.

**Ch222 (Computational Geometry Algorithms).** Convex hull, Delaunay triangulation, and Alpha complex construction (§3.3) share the GPU parallel prefix scan and compaction patterns developed in Ch222. The robust edge intersection and circumradius predicates used in Čech complex construction (§3.4) are the same as Ch222 §12.

**Ch224 (Shape Analysis Algorithms).** Shape descriptors computed in Ch224 (curvature, geodesic distance, shape diameter function) serve as scalar fields for the topological scalar field analysis of §9 and §6.5. Persistent homology of the curvature field (§9.4) is a natural follow-on to curvature computation in Ch224.

**Ch220 (GPU Image Processing Algorithms).** CubicalRipser (§4.5) operates on volumetric data in the same format as 3D image processing kernels. The GPU Gaussian smoothing, noise reduction, and gradient computation of Ch220 are preprocessing steps before cubical persistent homology.

**Ch210 (GPU Physics and Volumetric Algorithms).** Marching cubes isosurface extraction developed in Ch210 provides the triangle mesh input for Euler characteristic computation (§2) and topological mesh repair (§11). Level-set tracking with topology change detection is a direct application of the contour tree (§6).

**Ch221 (GPU Algorithm Performance).** The performance trade-offs in this chapter — GPU all-pairs distance O(n²) vs. sparse Rips, GPU union-find scalability, CUDA vs. Vulkan compute for boundary matrix reduction — apply the performance measurement methodology of Ch221, particularly subgroup occupancy, memory bandwidth limits, and kernel fusion strategies.

**Ch212 (GPU Neural and Specialised Primitives).** TDA persistence images (§7.3) serve as neural network input features for shape classification. The Mapper graph (§10) is used as a topology-aware dimensionality reduction tool for high-dimensional neural network feature spaces.

**Ch154 (Bindless Resources).** The SSBO-backed dynamic geometry for Reeb graph and persistence diagram visualisation (§12.3, §12.5) uses the bindless resource patterns of Ch154 to pass variable-length topology data to shaders without rebuilding descriptor sets.
