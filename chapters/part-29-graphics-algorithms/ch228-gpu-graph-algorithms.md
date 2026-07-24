# Chapter 228: GPU Graph Algorithms

**Audiences:** Graphics application developers implementing nav mesh path queries, scene dependency graphs, or GPU-accelerated analytics; systems developers building GPU-resident graph data structures. Readers should be familiar with Vulkan compute dispatch basics (Ch24–25) and GPU memory access patterns (Ch221).

**Relationship to Part XXIX.** Previous chapters in this part cover spatial structures (Ch209), physics island detection (Ch210), computational geometry (Ch222), shape analysis (Ch224), and computational topology (Ch225). Graph algorithms are the combinatorial layer beneath all of those: a BVH is an implicit binary tree graph, mesh islands are connected components, and shape correspondence is a matching problem on a graph. This chapter collects the named GPU graph algorithms — BFS, SSSP, connected components, PageRank, MST, coloring, triangle counting, community detection, and GNN inference — that those applications reach for repeatedly.

---

## Table of Contents

1. [Graphs on the GPU](#1-graphs-on-the-gpu)
2. [Parallel BFS](#2-parallel-bfs)
3. [Single-Source Shortest Path](#3-single-source-shortest-path)
4. [Connected Components](#4-connected-components)
5. [PageRank and Centrality](#5-pagerank-and-centrality)
6. [Minimum Spanning Tree](#6-minimum-spanning-tree)
7. [Graph Coloring and Independent Sets](#7-graph-coloring-and-independent-sets)
8. [Triangle Counting and Listing](#8-triangle-counting-and-listing)
9. [Community Detection](#9-community-detection)
10. [GPU Graph Neural Network Inference](#10-gpu-graph-neural-network-inference)
11. [Production Libraries](#11-production-libraries)
12. [Integration with the Graphics Pipeline](#12-integration-with-the-graphics-pipeline)
- [Integrations](#integrations)

---

## 1. Graphs on the GPU

### 1.1 The SIMT Mismatch with Irregular Graph Structure

SIMT execution assumes all threads in a warp follow the same instruction path and access memory in a strided, predictable pattern. Graph algorithms violate both assumptions. Two key structural properties cause trouble:

**Degree heterogeneity.** Real-world graphs — road networks, scene graphs, social networks — follow power-law or near-power-law degree distributions. The highest-degree vertex may have millions of neighbours; most vertices have fewer than ten. When one thread per vertex processes its neighbour list, the high-degree vertex's thread runs for millions of iterations while neighbouring warp lanes idle — classic warp divergence at the work granularity level.

**Pointer-chasing access.** The canonical graph traversal pattern is: read a vertex's neighbour list offset, then scatter to those neighbours. Each scatter address depends on the graph topology; without preprocessing, consecutive threads access random memory addresses, collapsing memory bandwidth to a fraction of peak.

Two programming models address these properties:

- **Frontier-based (level-synchronous) BFS:** All active vertices at the current level are processed together; barriers separate levels. This maps onto GPU bulk-synchronous execution but can waste threads when the frontier is small.
- **Work-efficient models:** Load-balance by assigning work not per vertex but per edge (warp-per-edge, CTA-per-vertex for high-degree). The Gunrock operator model (§11) formalises this with a `load_balance_t` template parameter.

### 1.2 GPU Graph Data Structures

**Compressed Sparse Row (CSR).** The standard GPU-resident graph representation uses three flat arrays:

```cpp
// CSR representation for a graph with V vertices and E directed edges
struct CSRGraph {
    int* row_offsets;   // size V+1: row_offsets[v]..row_offsets[v+1] spans neighbors of v
    int* col_indices;   // size E: neighbor vertex IDs
    float* values;      // size E: edge weights (optional)
    int num_vertices;
    int num_edges;
};
```

Accessing all neighbours of vertex `v` is `col_indices[row_offsets[v] .. row_offsets[v+1]-1]`. If thread `t` processes vertex `t`, all threads in a warp access consecutive `row_offsets` entries (coalesced), but the subsequent gather into `col_indices` is irregular.

**Coordinate list (COO / edge list).** Two arrays of length E store `(src, dst)` pairs, plus an optional weight array. COO is convenient for edge-parallel algorithms (Bellman-Ford relaxation) and simple to construct but requires O(E) sort for CSR conversion. `thrust::sort_by_key` on the source array achieves this on GPU in O(E log E) time.

**Adjacency matrix (sparse).** For dense graphs or SpMV-based algorithms (PageRank §5, GraphBLAS §11), the adjacency matrix stored in CSR format and multiplied with a dense vector via cuSPARSE `cusparseSpMV` is the natural choice. The `CUSPARSE_SPMV_CSR_ALG2` algorithm handles power-law degree distributions better than algorithm 1 by sorting rows internally. [Source: cuSPARSE documentation, https://docs.nvidia.com/cuda/cusparse/index.html#cusparse-generic-function-spmv]

**Memory layout for coalesced access.** Reordering vertices using a space-filling curve (Hilbert order, RCM bandwidth reduction) clusters spatially adjacent vertices in memory, improving neighbour gather locality by 3–5× on graphs derived from meshes or grids. For pure social graphs this helps less. [Source: Merrill, Garland, "Merge-based Parallel Sparse Matrix-Vector Multiplication", SC 2016, https://dl.acm.org/doi/10.5555/3014904.3014982]

**Static vs dynamic graphs.** CSR is immutable: adding an edge requires rebuilding the entire structure. Dynamic GPU graphs use chunked adjacency lists or GPU hash tables (e.g., cuDF/cuGraph's `EdgeList` backed by a hash table) to support incremental edge insertion. The 2025 cuGraph release added streaming graph primitives through the `cugraph::GraphCSR` C++ API, but the Python `cugraph` layer targets static snapshots. [Source: cuGraph GitHub repository, https://github.com/rapidsai/cugraph]

---

## 2. Parallel BFS

### 2.1 Level-Synchronous BFS

Level-synchronous BFS processes one frontier (the set of vertices at distance d from the source) per kernel launch. The canonical top-down variant:

```cuda
// Top-down BFS kernel — one thread per frontier vertex
__global__ void bfs_top_down(
    const int* row_offsets, const int* col_indices,
    const int* frontier, int frontier_size,
    int* next_frontier, int* next_size,
    int* visited, int* distance, int level)
{
    int tid = blockIdx.x * blockDim.x + threadIdx.x;
    if (tid >= frontier_size) return;

    int v = frontier[tid];
    for (int e = row_offsets[v]; e < row_offsets[v + 1]; ++e) {
        int u = col_indices[e];
        if (atomicCAS(&visited[u], 0, 1) == 0) {
            distance[u] = level + 1;
            int pos = atomicAdd(next_size, 1);
            next_frontier[pos] = u;
        }
    }
}
```

The `atomicCAS` guards against duplicate insertions when multiple threads discover the same unvisited vertex simultaneously. The output compaction pattern (writing to `next_frontier` via `atomicAdd`) creates contention when many vertices are discovered simultaneously; warp-level prefix scan (`__popc` of a `__ballot_sync` mask) replaces per-thread atomics with a single warp-level atomic:

```cuda
unsigned int mask = __ballot_sync(0xffffffff, should_insert);
int lane    = __ffs(mask) - 1;           // leader lane
int warp_count = __popc(mask);
int warp_base;
if (threadIdx.x % 32 == lane)
    warp_base = atomicAdd(next_size, warp_count);
warp_base = __shfl_sync(mask, warp_base, lane);
int my_offset = __popc(mask & ((1u << (threadIdx.x % 32)) - 1));
if (should_insert) next_frontier[warp_base + my_offset] = u;
```

### 2.2 Direction-Optimising BFS (Beamer et al.)

When the frontier is dense (approaching half of all vertices), the top-down approach wastes work because most outgoing edges lead to already-visited vertices. Beamer, Asanovic, and Patterson (SC 2012) introduced a direction-optimising BFS that switches to a **bottom-up** pass when the frontier is large: each unvisited vertex checks whether *any* of its neighbours is in the current frontier. [Source: Beamer, Asanovic, Patterson, "Direction-Optimizing Breadth-First Search", SC 2012, https://dl.acm.org/doi/10.5555/2388996.2389013]

The switching heuristic uses two empirically tuned parameters:
- Switch to bottom-up when `frontier_edges > α × total_edges_to_unvisited` (α ≈ 1/14)
- Switch back to top-down when `frontier_size < β × num_vertices` (β ≈ 1/24)

On GPU the bottom-up pass launches one thread per *unvisited* vertex and tests its neighbour list for frontier membership using a bitmask indexed by vertex ID:

```cuda
__global__ void bfs_bottom_up(
    const int* row_offsets, const int* col_indices,
    const uint32_t* frontier_bitmask,
    int* next_frontier_bitmask, int* unvisited,
    int* distance, int level, int num_unvisited)
{
    int tid = blockIdx.x * blockDim.x + threadIdx.x;
    if (tid >= num_unvisited) return;
    int v = unvisited[tid];
    for (int e = row_offsets[v]; e < row_offsets[v + 1]; ++e) {
        int u = col_indices[e];
        if (frontier_bitmask[u / 32] & (1u << (u % 32))) {
            distance[v] = level + 1;
            atomicOr(&next_frontier_bitmask[v / 32], 1u << (v % 32));
            break;
        }
    }
}
```

The bitmask representation reduces the memory footprint of the frontier from O(|frontier| × 4 bytes) to O(V / 8 bytes) and allows testing membership with a single 32-bit read.

### 2.3 Multi-Source BFS for Unweighted SSSP

Running BFS simultaneously from multiple sources, each tagged with a unique source ID, computes unweighted multi-source shortest paths in a single traversal. In practice this is implemented by initialising the frontier with all source vertices at level 0, each labelled with its source index. The distance and predecessor arrays then record both the distance and the nearest source — equivalent to a Voronoi diagram on the graph. Applications include GPU nav mesh distance fields (§12) and mesh island-to-island distance computation.

cuGraph exposes this directly:

```python
import cugraph, cudf

G = cugraph.Graph()
G.from_cudf_edgelist(edges_df, source='src', destination='dst')

sources = cudf.Series([0, 17, 42])   # three source vertices
result  = cugraph.multi_source_bfs(G, sources)
# result: cudf.DataFrame with columns ['vertex', 'distance', 'predecessor']
```
[Source: cuGraph traversal module, https://github.com/rapidsai/cugraph/tree/main/python/cugraph/cugraph/traversal]

---

## 3. Single-Source Shortest Path

### 3.1 Bellman-Ford: Parallel Edge Relaxation

Bellman-Ford relaxes all edges in parallel for V−1 iterations. Each iteration updates `dist[dst] = min(dist[dst], dist[src] + weight)`:

```cuda
__global__ void bellman_ford_relax(
    const int* src, const int* dst, const float* weight,
    float* dist, int* changed, int num_edges)
{
    int eid = blockIdx.x * blockDim.x + threadIdx.x;
    if (eid >= num_edges) return;
    float new_dist = dist[src[eid]] + weight[eid];
    if (new_dist < dist[dst[eid]]) {
        atomicMin_float(&dist[dst[eid]], new_dist);   // float atomicMin via int reinterpret
        *changed = 1;
    }
}
```

Early termination via a convergence flag (`changed`) avoids the full V−1 iterations when the graph has small diameter. GPU Bellman-Ford is O(V × E / P) with P GPU threads; its main virtue is simplicity and correctness on graphs with negative-weight edges (but no negative cycles). It scales poorly for large-diameter graphs.

### 3.2 Delta-Stepping (Bucket Queue on GPU)

Delta-stepping partitions vertices into buckets of width Δ. Vertices in the current bucket are relaxed; any newly relaxed vertex is placed in bucket `⌊dist/Δ⌋`. The algorithm processes buckets in order; within a bucket, it alternates between "light" relaxations (edge weight ≤ Δ) and "heavy" relaxations (weight > Δ) to achieve work-efficiency. [Source: Meyer, Sanders, "Δ-stepping: a parallelizable shortest path algorithm", Journal of Algorithms 2003, https://doi.org/10.1016/S0196-6774(03)00076-2]

GPU delta-stepping uses an array-of-lists bucket structure in device memory. Each bucket is a compact list of vertex IDs, maintained with atomic push. Choosing Δ ≈ 1/d̄ where d̄ is the average edge weight balances bucket count against per-bucket work; Δ too small creates many nearly-empty buckets, Δ too large reverts toward Bellman-Ford.

cuGraph's `sssp` dispatches to a delta-stepping implementation internally:

```python
result = cugraph.sssp(G, source=0, cutoff=1e9)
# result: cudf.DataFrame with columns ['vertex', 'distance', 'predecessor']
```
[Source: cuGraph sssp module, https://github.com/rapidsai/cugraph/blob/main/python/cugraph/cugraph/traversal/sssp.py]

### 3.3 GPU Dijkstra with Priority Queue

Classical Dijkstra is sequential (it processes one vertex at a time), but GPU variants exploit the fact that multiple vertices can be at the same distance. A warp-level tournament tree implements a parallel min-heap on shared memory: 16 lanes hold the current minimum candidates; a tree-reduction finds the global minimum in log₂(16) = 4 steps. Alternatively, a global min-heap maintained with `atomicMin` achieves parallelism at the cost of contention.

For small graphs resident in shared memory (< 48 KB), blocked Dijkstra processes a subgraph within a single CTA, storing the distance array in shared memory with warp-shuffle-based priority queue updates. This approach reaches near-memory-bandwidth performance for dense, diameter-bounded graphs.

### 3.4 Algorithm Comparison

| Algorithm | Complexity (sequential) | GPU parallelism | Negative weights | Best for |
|---|---|---|---|---|
| Bellman-Ford | O(V·E) | Edge-parallel, 1 kernel/iter | Yes (no negative cycles) | Small graphs, negative weights |
| Delta-stepping | O(V + E + V·d̄/Δ) expected | Bucket-parallel | No | Moderate graphs, uniform weights |
| GPU Dijkstra | O((V+E) log V) | Warp tournament-tree | No | Small dense graphs in smem |

Davidson, Baxter, Garland, and Owens (IPDPS 2014) benchmarked these on the NVIDIA K40, finding delta-stepping 3–5× faster than Bellman-Ford on road networks and social graphs, with Dijkstra competitive only for graphs fitting in L2. [Source: Davidson, Baxter, Garland, Owens, "Work-Efficient Parallel GPU Methods for Single-Source Shortest Paths", IPDPS 2014, https://doi.org/10.1109/IPDPS.2014.87]

---

## 4. Connected Components

### 4.1 Shiloach-Vishkin Hooking and Shortcutting

The Shiloach-Vishkin algorithm maintains a parent array D where `D[v]` is the component representative of vertex `v`. Two phases repeat until convergence:

- **Hooking:** For each edge (u, v), if `D[u] ≠ D[v]`, set `D[D[u]] = D[v]` (connect components).
- **Shortcutting:** For each vertex `v`, set `D[v] = D[D[v]]` (flatten the tree to reduce iterations).

Both operations are embarrassingly parallel and map directly to compute kernels. Convergence is detected by a reduction over a per-iteration change flag. The algorithm requires O(log V) iterations in expectation.

```cuda
__global__ void sv_hook(const int* src, const int* dst, int* parent, int num_edges) {
    int eid = blockIdx.x * blockDim.x + threadIdx.x;
    if (eid >= num_edges) return;
    int u = src[eid], v = dst[eid];
    while (parent[u] != parent[v]) {
        int pu = parent[u], pv = parent[v];
        if (pu < pv) atomicMin(&parent[pu], pv);
        else         atomicMin(&parent[pv], pu);
        u = parent[u]; v = parent[v];
    }
}
__global__ void sv_shortcut(int* parent, int n) {
    int v = blockIdx.x * blockDim.x + threadIdx.x;
    if (v >= n) return;
    while (parent[parent[v]] != parent[v])
        parent[v] = parent[parent[v]];
}
```

### 4.2 ECL-CC

ECL-CC (developed by Burtscher et al. at Texas State University) is empirically the fastest GPU connected components implementation on a range of graph types. It combines label propagation seeded with each vertex's smallest neighbour ID and a GPU-optimised union-find with three key design choices:

1. **Adaptive thread granularity.** A vertex with fewer than 8 neighbours is processed by a single thread; 8–32 by a half-warp; >32 by a full warp or CTA, eliminating load imbalance.
2. **Lock-free path compression.** The union-find uses atomic operations without locks, tolerating temporary inconsistency that self-corrects via path compression.
3. **Asynchronous propagation.** Updates are applied immediately (without a full barrier between iterations), reducing the number of synchronisation rounds needed. [Source: Jaiganesh and Burtscher, "A High-Performance Connected Components Implementation for GPUs", ICS 2018, https://dl.acm.org/doi/10.1145/3205289.3205291]

```bash
# Build and run ECL-CC on a CSR binary graph
git clone https://github.com/burtscher/ECL-CC
make
./ECL-CC graph.egr   # graph in Burtscher's binary CSR format
```

### 4.3 GPU Union-Find with Path Compression

Parallel union-find on GPU uses two arrays: `parent[]` and `rank[]`. Union by rank and path compression guarantee near-O(1) amortised find. The critical GPU adaptation is the atomic CAS loop for `find`:

```cuda
__device__ int find(int* parent, int x) {
    int root = x;
    while (parent[root] != root) root = parent[root];
    // Path compression (halving)
    while (parent[x] != root) {
        int next = parent[x];
        parent[x] = root;
        x = next;
    }
    return root;
}
__device__ void unite(int* parent, int* rank, int a, int b) {
    a = find(parent, a); b = find(parent, b);
    if (a == b) return;
    if (rank[a] < rank[b]) { int tmp = a; a = b; b = tmp; }
    atomicCAS(&parent[b], b, a);
    if (rank[a] == rank[b]) atomicAdd(&rank[a], 1);
}
```

### 4.4 Weakly and Strongly Connected Components

**Weakly connected components (WCC)** treat directed edges as undirected and are solved by the union-find or Shiloach-Vishkin algorithms above. cuGraph provides:

```python
wcc_result = cugraph.weakly_connected_components(G)
# cudf.DataFrame with columns ['vertex', 'labels']
```

**Strongly connected components (SCC)** require two BFS traversals (Kosaraju's algorithm): one on the original graph, one on the transposed graph, with vertices processed in reverse finish order. The GPU implementation runs two BFS sweeps plus a radix sort on finish times. cuGraph provides:

```python
scc_result = cugraph.strongly_connected_components(G)
```
[Source: cuGraph components module, https://github.com/rapidsai/cugraph/blob/main/python/cugraph/cugraph/components/connectivity.py]

**Applications in graphics.** WCC detects disconnected mesh islands for separate UV unwrapping, separate physics simulation islands (Ch210), and scene cluster culling. SCC detects cycles in render pass dependency graphs for GPU frame scheduling (§12).

---

## 5. PageRank and Centrality

### 5.1 Power Iteration via SpMV

PageRank computes the stationary distribution of a random walk on the directed graph. The iterative update is:

```
r_new = α · A^T · r_old + (1 − α) · 1/V
```

where A is the column-stochastic adjacency matrix and α ≈ 0.85 is the damping factor. Implemented as sparse matrix-vector multiplication (SpMV) on the CSR adjacency matrix using cuSPARSE `cusparseSpMV`, each iteration multiplies the transposed adjacency matrix by the current rank vector. Convergence is tested via L1 residual norm ‖r_new − r_old‖₁ < ε.

cuGraph wraps this in a single Python call:

```python
pr_result = cugraph.pagerank(
    G,
    alpha=0.85,
    max_iter=100,
    tol=1.0e-5,
    fail_on_nonconvergence=False,   # returns (df, converged_bool)
)
# pr_result[0]: cudf.DataFrame with columns ['vertex', 'pagerank']
```
[Source: cuGraph pagerank module, https://github.com/rapidsai/cugraph/blob/main/python/cugraph/cugraph/link_analysis/pagerank.py]

**Personalised PageRank** biases the random walk toward a seed set by replacing the uniform restart distribution with a non-uniform `personalization` vector. cuGraph accepts a cudf.DataFrame as the `personalization` parameter with columns `vertex` and `values`.

### 5.2 Betweenness Centrality

Betweenness centrality BC(v) counts the fraction of shortest paths between all vertex pairs that pass through v. Brandes (2001) computes exact BC in O(V·E) time via V BFS traversals plus backward dependency accumulation. On GPU, each of the V BFS traversals is launched independently, with warp-level parallelism within each level of the BFS tree.

For large graphs, approximate BC via k random source samples (k ≪ V) reduces cost to O(k·E):

```python
bc_result = cugraph.betweenness_centrality(
    G,
    k=500,              # number of random source samples
    normalized=True,
    random_state=42,
)
# bc_result: cudf.DataFrame with columns ['vertex', 'betweenness_centrality']
```
[Source: cuGraph betweenness_centrality module, https://github.com/rapidsai/cugraph/blob/main/python/cugraph/cugraph/centrality/betweenness_centrality.py]

---

## 6. Minimum Spanning Tree

### 6.1 Borůvka's Algorithm on GPU

Borůvka's algorithm repeatedly merges components by finding the minimum-weight outgoing edge from each component, making it inherently parallel. One GPU iteration:

1. **Find min outgoing edge per component:** One kernel processes all edges; for each edge (u, v) with different component labels, atomically update the minimum outgoing edge for `component[u]` and `component[v]` using a CAS loop on a packed `(weight, endpoint)` value.
2. **Hook components:** For each component, add the minimum outgoing edge to the spanning tree and union the two components.
3. **Relabel:** Re-run path compression on the parent array to normalise component labels.

Sorting edges by weight with `thrust::sort_by_key` once before the algorithm begins allows the find-min step to terminate early at the first valid edge per component. The algorithm completes in O(log V) iterations. [Source: Vineet and Narayanan, "CUDA solutions for the SSSP problem", HiPC 2009, https://doi.org/10.1007/978-3-642-11322-2_12]

cuGraph's `minimum_spanning_tree` uses the Borůvka implementation:

```python
mst = cugraph.minimum_spanning_tree(G, algorithm="boruvka")
# mst: cugraph.Graph containing only the MST edges
print(f"MST edges: {mst.number_of_edges()}, MST weight: {mst.view_edge_list()['weights'].sum()}")
```
[Source: cuGraph MST module, https://github.com/rapidsai/cugraph/blob/main/python/cugraph/cugraph/tree/minimum_spanning_tree.py]

### 6.2 GPU Kruskal

Kruskal's algorithm sorts all edges by weight, then adds edges greedily using union-find to avoid cycles. On GPU, the radix sort (`thrust::sort_by_key` on edge weights) runs in O(E log E) time using count-sort passes; the sequential union-find pass is then bottlenecked by data dependency. Batched parallel union-find (processing edges in parallel with CAS and retrying on conflict) achieves GPU utilisation at the cost of some extra work.

For graphs where E ≫ V (dense graphs or graphs with many parallel edges), Kruskal is preferred over Borůvka because the sort dominates and is highly GPU-efficient. For sparse graphs, Borůvka's O(log V) iteration count is typically faster.

### 6.3 Applications

MST applications in graphics include:

- **Mesh connectivity repair:** After Delaunay remeshing (Ch222), the MST of a point cloud with geodesic edge weights defines the minimum cost skeleton connecting all vertices — useful for detecting missing geometry or non-manifold boundaries.
- **Level-of-detail clustering:** The MST of a scene graph weighted by spatial distance or visual similarity gives a natural hierarchy for LOD grouping (§9).
- **Texture atlas seam selection:** Mesh parameterisation tools (e.g., xatlas) use spanning tree structures to select seams that minimise texture distortion.

---

## 7. Graph Coloring and Independent Sets

### 7.1 Jones-Plassmann Parallel Coloring

Graph coloring assigns integer labels (colours) to vertices such that no two adjacent vertices share a colour, minimising the number of colours used. The greedy Jones-Plassmann (JP) algorithm assigns a random priority to each vertex and processes vertices in parallel: a vertex is colored at the current round if its priority is higher than all its neighbours' priorities.

```cuda
__global__ void jp_color(
    const int* row_offsets, const int* col_indices,
    const float* priority, int* color, int num_vertices)
{
    int v = blockIdx.x * blockDim.x + threadIdx.x;
    if (v >= num_vertices || color[v] != -1) return;

    // Check if v has higher priority than all uncolored neighbours
    bool is_local_max = true;
    for (int e = row_offsets[v]; e < row_offsets[v+1] && is_local_max; ++e) {
        int u = col_indices[e];
        if (color[u] == -1 && priority[u] > priority[v])
            is_local_max = false;
    }
    if (!is_local_max) return;

    // Assign smallest valid colour
    // (Uses a bitmask of neighbour colours to find first unused)
    uint64_t used = 0;
    for (int e = row_offsets[v]; e < row_offsets[v+1]; ++e) {
        int c = color[col_indices[e]];
        if (c >= 0 && c < 64) used |= (1ULL << c);
    }
    color[v] = __builtin_ctzll(~used);   // lowest unset bit
}
```

This kernel is launched iteratively; each round colors a constant fraction of uncolored vertices. The expected number of rounds is O(log V) for random priority assignments. [Source: Jones, Plassmann, "A Parallel Graph Coloring Heuristic", SIAM Journal on Scientific Computing 1993, https://doi.org/10.1137/0914073]

### 7.2 Distance-1 and Distance-2 Coloring

Distance-1 coloring (standard graph coloring) is sufficient for Gauss-Seidel smoothers in algebraic multigrid (AMG, Ch226): vertices of the same colour can be updated independently. Distance-2 coloring ensures no two vertices within graph distance 2 share a colour — required for Jacobi-type ILU ordering and for symmetric Gauss-Seidel. Distance-2 coloring is equivalent to coloring the square of the graph G² and requires O(d̄²) expected colours (d̄ = average degree), which grows quickly for high-degree graphs.

### 7.3 Maximal Independent Set via Luby's Algorithm

A maximal independent set (MIS) is a vertex subset S such that no two vertices in S are adjacent, and every vertex not in S has a neighbour in S. Luby's randomised parallel MIS selects candidate vertices (those whose random priority exceeds all their neighbours') each round and adds them to S:

```
Repeat until all vertices are processed:
  1. Each unprocessed vertex v picks random priority p(v)
  2. v joins MIS if p(v) > p(u) for all neighbours u
  3. Remove v and all its neighbours from the active set
```

Each round removes a constant fraction of vertices in expectation; the algorithm terminates in O(log V) rounds. [Source: Luby, "A Simple Parallel Algorithm for the Maximal Independent Set Problem", SIAM Journal on Computing 1986, https://doi.org/10.1137/0215074]

MIS is used in mesh coarsening for AMG (every other vertex is coarsened), in sparse direct solvers for elimination ordering, and in point cloud thinning (Ch211).

---

## 8. Triangle Counting and Listing

### 8.1 Set Intersection for Triangle Counting

A triangle is a triple (u, v, w) where all three edges (u,v), (v,w), (u,w) are present. For each directed edge (u, v) (u < v), the number of triangles containing that edge equals |N(u) ∩ N(v)| — the size of the intersection of their neighbour sets. Computing all pairwise intersections has O(E · d̄) work; with sorted adjacency lists, each intersection is a linear-time merge or binary search.

### 8.2 Warp-Per-Edge GPU Triangle Counting

Green et al. assign one GPU warp to each directed edge (u, v). The warp cooperatively intersects the sorted neighbour lists of u and v using a two-pointer merge, with threads handling different list positions:

```cuda
__global__ void triangle_count_warp(
    const int* row_offsets, const int* col_indices,
    unsigned long long* total_triangles, int num_edges,
    const int* edge_src, const int* edge_dst)
{
    int warp_id = (blockIdx.x * blockDim.x + threadIdx.x) / 32;
    int lane    = threadIdx.x % 32;
    if (warp_id >= num_edges) return;

    int u = edge_src[warp_id], v = edge_dst[warp_id];
    // Neighbourhood bounds for u and v (stored in sorted order)
    int au = row_offsets[u], bu = row_offsets[u+1];
    int av = row_offsets[v], bv = row_offsets[v+1];

    unsigned long long count = 0;
    // Parallel merge: warp lanes handle different chunks
    int p = au + lane, q = av;
    while (p < bu && q < bv) {
        int pu = col_indices[p], pv = col_indices[q];
        if (pu == pv) { ++count; ++p; ++q; }
        else if (pu < pv) p += 32;   // advance u-side by warp width
        else              ++q;
    }
    // Warp-level reduction of count
    count += __shfl_down_sync(0xffffffff, count, 16);
    count += __shfl_down_sync(0xffffffff, count,  8);
    count += __shfl_down_sync(0xffffffff, count,  4);
    count += __shfl_down_sync(0xffffffff, count,  2);
    count += __shfl_down_sync(0xffffffff, count,  1);
    if (lane == 0) atomicAdd(total_triangles, count);
}
```

[Source: Green, Bulu, Gilbert, "GPU merge path: a GPU merging algorithm", ICS 2012; Polok, Ila, "Fast Radix Sort for Sparse Linear Algebra on the GPU", 2013; also: H. Kabir and K. Gropp, "High-performance triangle counting on GPUs", IEEE/ACM HiPC 2017]

cuGraph exposes triangle counting directly:

```python
tc_result = cugraph.triangle_count(G)
# tc_result: cudf.DataFrame with columns ['vertex', 'counts']
# Sum gives total triangle count / 3 (each triangle counted at 3 vertices)
total = tc_result['counts'].sum() // 3
```
[Source: cuGraph triangle_count module, https://github.com/rapidsai/cugraph/blob/main/python/cugraph/cugraph/community/triangle_count.py]

### 8.3 Applications

- **Mesh quality metrics:** Triangle counting on the vertex adjacency graph of a mesh detects irregular star patterns (vertices connected to many others without shared triangles in the mesh adjacency graph) that indicate near-degenerate mesh structure.
- **Clustering coefficient:** The local clustering coefficient of a vertex v is `triangles(v) / (deg(v) × (deg(v)-1) / 2)` — a measure of neighbourhood density that drives LOD decisions.

---

## 9. Community Detection

### 9.1 Louvain Modularity Optimisation on GPU

Louvain is a two-phase greedy algorithm for modularity maximisation. Phase 1 assigns each vertex to the community of the neighbour that maximises the modularity gain ΔQ. Phase 2 contracts the graph, replacing each community with a super-vertex. The two phases alternate until no improvement exceeds a threshold.

On GPU, Phase 1 maps naturally to a vertex-parallel kernel: each vertex evaluates its modularity gain relative to each neighbouring community and issues an atomic update to the community assignment array. Race conditions between vertices evaluating each other cause occasional non-determinism, but convergence to a local optimum is guaranteed. [Source: Ghosh, Halappanavar, Tumeo, Kalyanaraman, Chavarria-Miranda, "Distributed Louvain Algorithm for Graph Community Detection", IPDPS 2018, https://doi.org/10.1109/IPDPS.2018.00098]

```python
parts, modularity = cugraph.louvain(
    G,
    max_level=100,
    resolution=1.0,
    threshold=1e-7,
)
# parts: cudf.DataFrame ['vertex', 'partition']
# modularity: float — global modularity of the partition
print(f"Communities: {parts['partition'].nunique()}, Q = {modularity:.4f}")
```
[Source: cuGraph louvain module, https://github.com/rapidsai/cugraph/blob/main/python/cugraph/cugraph/community/louvain.py]

Ensemble Clustering for Graphs (ECG) improves robustness by running truncated Louvain on multiple random permutations of the graph to estimate edge community membership weights, then running full Louvain on the reweighted graph:

```python
ecg_parts, ecg_q = cugraph.ecg(G, ensemble_size=100, resolution=1.0)
```
[Source: cuGraph ecg module, https://github.com/rapidsai/cugraph/blob/main/python/cugraph/cugraph/community/ecg.py]

### 9.2 Label Propagation Algorithm

The Label Propagation Algorithm (LPA) initialises each vertex with a unique label, then repeatedly assigns each vertex the label held by the majority of its neighbours (breaking ties randomly). LPA is one GPU kernel per iteration with no phase-2 coarsening:

```cuda
__global__ void lpa_update(
    const int* row_offsets, const int* col_indices,
    const int* old_label, int* new_label,
    int* changed, int num_vertices)
{
    int v = blockIdx.x * blockDim.x + threadIdx.x;
    if (v >= num_vertices) return;
    // Count votes per label (using shared memory histogram for small alphabets)
    int best_label = old_label[v], best_count = 0;
    for (int e = row_offsets[v]; e < row_offsets[v+1]; ++e) {
        int u_label = old_label[col_indices[e]];
        // Simplified: track local majority via two-candidate tracking
        if (u_label == best_label) ++best_count;
        else if (best_count == 0) { best_label = u_label; best_count = 1; }
        else --best_count;
    }
    if (new_label[v] != best_label) { new_label[v] = best_label; *changed = 1; }
}
```

LPA is faster per iteration than Louvain (no modularity computation) but produces lower-quality communities. It is preferred for real-time scene clustering applications where quality is secondary to speed.

### 9.3 Applications in Graphics

- **Level-of-detail clustering:** Community detection on a scene graph weighted by spatial proximity and visual similarity groups objects for automatic LOD generation.
- **Scene partitioning for GPU culling:** Communities map to GPU frustum culling cells; objects in the same community are likely in the same cell.
- **Mesh segmentation:** Communities on the mesh dual graph (faces as vertices, shared edges as edges weighted by dihedral angle) identify meaningful shape parts.

---

## 10. GPU Graph Neural Network Inference

### 10.1 Message Passing Framework on GPU

Graph Neural Networks compute vertex embeddings by iterating message passing: each vertex aggregates features from its neighbours, then updates its own embedding. The generic message-passing update is:

```
h_v^(l+1) = UPDATE(h_v^(l), AGGREGATE({MSG(h_v^(l), h_u^(l), e_{uv}) : u ∈ N(v)}))
```

On GPU over a CSR graph, this decomposes into a **scatter** (compute per-edge messages from source vertex features) followed by a **gather** (aggregate messages into destination vertices):

```cuda
// Scatter: for each edge (u→v), compute message and write to edge buffer
__global__ void scatter_messages(
    const int* edge_src, const int* edge_dst,
    const float* h, float* edge_msg,
    int num_edges, int feat_dim)
{
    int eid = blockIdx.x * blockDim.x + threadIdx.x;
    if (eid >= num_edges) return;
    int u = edge_src[eid];
    // Identity message: copy source features
    for (int f = 0; f < feat_dim; ++f)
        edge_msg[eid * feat_dim + f] = h[u * feat_dim + f];
}
// Gather: for each vertex v, sum messages from incoming edges
__global__ void gather_sum(
    const int* row_offsets_in, const int* edge_ids,
    const float* edge_msg, float* h_new,
    int num_vertices, int feat_dim)
{
    int v = blockIdx.x * blockDim.x + threadIdx.x;
    if (v >= num_vertices) return;
    for (int f = 0; f < feat_dim; ++f) {
        float acc = 0.0f;
        for (int e = row_offsets_in[v]; e < row_offsets_in[v+1]; ++e)
            acc += edge_msg[edge_ids[e] * feat_dim + f];
        h_new[v * feat_dim + f] = acc;
    }
}
```

The alternative formulation — expressing the aggregation as a sparse matrix multiply `H_new = A · H_old` (where A is the adjacency matrix) — maps directly onto cuSPARSE `cusparseSpMM` and is used by GraphSAGE and GCN.

### 10.2 DGL GPU Kernels

The Deep Graph Library (DGL) wraps the scatter-gather pattern in a high-level Python API. The `update_all` function dispatches a message function and a reduce function over the entire graph in one GPU pass:

```python
import dgl, dgl.function as fn, torch

# Build DGL graph from edge lists (GPU tensors)
g = dgl.graph((src_tensor, dst_tensor)).to('cuda')
g.ndata['h'] = node_features_cuda   # shape [N, F]
g.edata['w'] = edge_weights_cuda    # shape [E, 1]

# One message-passing step: copy source features, then sum at destination
g.update_all(
    message_func=fn.copy_u('h', 'm'),   # m_e = h_src[e]
    reduce_func=fn.sum('m', 'h_new'),   # h_new[v] = Σ m_e for e→v
)
new_features = g.ndata['h_new']         # shape [N, F]
```

DGL internally chooses between SpMM (for dense features) and a custom CUDA scatter-gather kernel (for sparse or attention-weighted aggregation) based on the message function type. [Source: DGL documentation — Message Passing, https://docs.dgl.ai/guide/message.html]

### 10.3 PyTorch Geometric

PyG structures GNN layers via the `MessagePassing` base class, which users subclass to implement `message()`, `aggregate()`, and `update()`:

```python
from torch_geometric.nn import MessagePassing
import torch

class GraphSAGEConv(MessagePassing):
    def __init__(self, in_dim, out_dim):
        super().__init__(aggr='mean')   # mean aggregation
        self.lin = torch.nn.Linear(in_dim * 2, out_dim)

    def forward(self, x, edge_index):
        # edge_index: [2, E] tensor on CUDA
        return self.propagate(edge_index, x=x)

    def message(self, x_j):        # x_j: neighbour features [E, F]
        return x_j

    def update(self, aggr_out, x): # concatenate self + aggregated
        return self.lin(torch.cat([x, aggr_out], dim=-1))
```

PyG dispatches to `torch_sparse.SparseTensor` for the aggregation step, using ROCm/CUDA kernels from `torch-sparse`. For large graphs, `NeighborLoader` samples a fixed-size neighbourhood to bound memory use per mini-batch. [Source: PyTorch Geometric documentation — MessagePassing, https://pytorch-geometric.readthedocs.io/en/latest/tutorial/create_gnn.html]

### 10.4 Graph Attention Network Sparse Attention

Graph Attention Networks (GAT) compute per-edge attention weights α_{uv} before aggregation. The attention mechanism over E edges is a sparse softmax: for each vertex v, normalise the attention weights over its incoming edges. On GPU this is a segmented softmax (one segment per vertex) implemented via CUB `DeviceSegmentedReduce`:

```cuda
// Compute per-vertex max for numerically stable softmax
// (uses row_offsets to define segments)
cub::DeviceSegmentedReduce::Max(
    d_temp_storage, temp_storage_bytes,
    d_attn_scores, d_max_per_vertex,
    num_vertices, d_row_offsets, d_row_offsets + 1);
```

DGL and PyG both implement GAT with fused CUDA kernels that combine the attention computation and aggregation to avoid materialising the full E × F attention-weighted message buffer.

### 10.5 Applications

- **Mesh segmentation (Ch224):** GNNs on the mesh dual graph classify faces into semantic parts using local geometric features.
- **Scene graph reasoning:** GNNs propagate semantic relationships (object-on, object-supports) over a scene graph to infer affordances for robotics or AR applications.
- **Nav mesh query acceleration:** GNNs trained on partial graph topology predict near-optimal path heuristics for A* search on large nav meshes.

---

## 11. Production Libraries

### 11.1 cuGraph (RAPIDS)

cuGraph is the primary GPU graph analytics library on NVIDIA hardware. It is part of the RAPIDS ecosystem and operates on cuDF DataFrames, enabling zero-copy integration with GPU-resident tabular data.

**Architecture.** The Python layer (`cugraph` package) wraps `pylibcugraph` (a lower-level Cython API that mirrors the C++ `libcugraph` library). Algorithms are implemented in CUDA C++ and exposed through a NetworkX-compatible interface. Graph construction accepts cuDF `DataFrame` edge lists, CuPy/SciPy sparse matrices, or NetworkX graphs.

**Algorithm coverage.** Verified algorithm categories in the cuGraph 2025 Python API:

| Category | Algorithms |
|---|---|
| Traversal | `bfs`, `sssp`, `multi_source_bfs`, `concurrent_bfs` |
| Link Analysis | `pagerank`, `hits` |
| Centrality | `betweenness_centrality`, `edge_betweenness_centrality`, `katz_centrality`, `eigenvector_centrality` |
| Community | `louvain`, `leiden`, `ecg`, `triangle_count`, `k_truss` |
| Components | `weakly_connected_components`, `strongly_connected_components` |
| Trees | `minimum_spanning_tree`, `maximum_spanning_tree` |
| Sampling | `uniform_random_walks`, `node2vec_random_walks`, `homogeneous_neighbor_sample` |

[Source: cuGraph Python `__init__.py`, https://github.com/rapidsai/cugraph/blob/main/python/cugraph/cugraph/__init__.py]

**Installation:**

```bash
conda install -c rapidsai -c nvidia cugraph=25.02 python=3.11 cuda-version=12.0
```

### 11.2 Gunrock

Gunrock is a CUDA graph processing library from UC Davis. It uses a **data-centric, bulk-synchronous** programming model built around vertex and edge *frontiers*. An algorithm is expressed as an `advance` operator (expand the frontier to neighbours satisfying a predicate) plus a `filter` operator (discard frontier elements failing a condition). Different load-balance strategies (block-mapped, merge-path, thread-mapped) are selected per-operator via a template parameter.

The library was rewritten as "Gunrock Essentials" and subsequently re-merged; the current `main` branch (v2.x+) represents the essentials architecture:

```cpp
#include <gunrock/algorithms/bfs.hxx>
using vertex_t = int; using edge_t = int; using weight_t = float;

// Load graph from Matrix Market format
io::matrix_market_t<vertex_t, edge_t, weight_t> mm;
auto [properties, coo] = mm.load(argv[1]);
format::csr_t<memory_space_t::device, vertex_t, edge_t, weight_t> csr;
csr.from_coo(coo);
auto G = graph::build<memory_space_t::device>(properties, csr);

// Run BFS from source vertex 0
thrust::device_vector<vertex_t> distances(G.get_number_of_vertices(), INT_MAX);
thrust::device_vector<vertex_t> predecessors(G.get_number_of_vertices(), -1);
gunrock::bfs::param_t<vertex_t> param(/*source=*/0);
gunrock::bfs::result_t<vertex_t> result(distances.data().get(),
                                         predecessors.data().get());
float elapsed = gunrock::bfs::run(G, param, result, context);
```

[Source: Gunrock GitHub repository, https://github.com/gunrock/gunrock]

**Available algorithms in Gunrock (2025):** BFS, SSSP, PageRank, Betweenness Centrality, Graph Coloring.

### 11.3 SuiteSparse:GraphBLAS and LAGraph

GraphBLAS defines graph algorithms as linear algebra operations over semirings — algebraic structures that replace the standard `(+, ×)` arithmetic with user-defined operations. BFS over a `(OR, AND)` semiring is a matrix-vector multiply: `frontier_new = A ⊗ frontier` where `⊗` denotes the semiring multiply. This formulation makes BFS, SSSP, connected components, and PageRank expressible as masked SpMV or SpGEMM (sparse matrix-matrix multiply). [Source: Kepner, Gilbert et al., "Mathematical Foundations of the GraphBLAS", IPDPS Workshops 2016, https://doi.org/10.1109/IPDPSW.2016.185]

SuiteSparse:GraphBLAS (v10.3.2, July 2026) is the reference C implementation:

```c
#include "GraphBLAS.h"

GrB_Matrix A;          // adjacency matrix
GrB_Vector frontier;   // current BFS frontier

// BFS: frontier_new = frontier * A over (OR, AND) semiring
GrB_vxm(frontier_new, mask, GrB_NULL,
         GrB_LOR_LAND_BOOL,    // predefined (LOR,LAND) boolean semiring
         frontier, A, GrB_NULL);
```

[Source: SuiteSparse:GraphBLAS GitHub, https://github.com/DrTimothyAldenDavis/GraphBLAS]

**GPU status (as of July 2026).** The CUDA backend in SuiteSparse:GraphBLAS is explicitly described as "a draft that isn't ready to use yet." Production GraphBLAS workloads run on CPU only. The CPU library is mature, powers RedisGraph, and is the built-in sparse engine in MATLAB R2021a.

**LAGraph** (https://github.com/GraphBLAS/LAGraph) is a collection of graph algorithms implemented on top of GraphBLAS, including BFS (with push-pull switching via `G->AT` and `G->out_degree`), connected components, and centrality algorithms. LAGraph requires SuiteSparse:GraphBLAS v9.0+ and provides the algorithm layer; GraphBLAS provides the engine.

### 11.4 AMD ROCm Graph Analytics

AMD ROCm does not ship a standalone graph analytics library equivalent to cuGraph as of mid-2026. The term "hipGraph" in AMD nomenclature refers to **HIP execution graphs** (the HIP analog of CUDA Graphs for kernel launch dependency tracking), not a graph data analytics library — a naming collision to be aware of.

GPU graph analytics on AMD hardware is primarily accessed through:

- **RAPIDS on ROCm:** The RAPIDS project provides experimental ROCm/HIP builds of cuDF and cuGraph, enabling the cuGraph Python API to run on AMD GPUs via HIP backend. Status varies by RAPIDS release; consult `https://rapids.ai/rocm` for the current compatibility matrix.
- **Gunrock on ROCm:** Gunrock's HIP port compiles via `hipcc`; algorithm coverage mirrors the CUDA version for graphs that fit within single-GPU memory.
- **PyTorch Geometric on ROCm:** `torch-scatter` and `torch-sparse` provide ROCm builds, enabling PyG GNN inference on AMD hardware.

---

## 12. Integration with the Graphics Pipeline

### 12.1 Nav Mesh as a GPU-Resident Graph

Recast/Detour is the standard CPU nav mesh library (used by Unity, Godot, and custom game engines). Its output is a polygon mesh with adjacency links that maps directly to a CSR graph. Uploading this to the GPU enables batched GPU pathfinding:

```cpp
// Upload Detour nav mesh adjacency to GPU CSR
struct NavMeshCSR {
    uint32_t* row_offsets;    // VkBuffer, size (num_polys+1) * 4
    uint32_t* neighbors;      // VkBuffer, size num_links * 4
    float*    link_costs;     // VkBuffer, size num_links * 4 (travel cost)
};
// GPU BFS/Dijkstra from multiple agent positions simultaneously (multi-source BFS)
// returns a distance field over the nav mesh for all agents in one kernel launch
```

GPU multi-source BFS (§2.3) computes the unweighted distance field from all agent positions in a single pass — equivalent to Voronoi labelling on the nav mesh graph. Weighted SSSP (delta-stepping, §3.2) computes accurate travel times accounting for terrain cost modifiers.

### 12.2 GPU Scene Graph Dependency Resolution

A render pass dependency graph is a directed acyclic graph (DAG) where each node is a render pass and edges encode read-after-write resource dependencies. GPU SCC (§4.4) detects cycles that indicate invalid dependencies. Topological sort (a BFS variant on the DAG) determines the valid execution order:

```glsl
// Compute shader: one thread per render pass node
// BFS-based topological sort for dependency resolution
// in_degree[v] = count of incoming edges
// When in_degree[v] drops to 0, enqueue v into the next frontier
```

The GPU Vulkan frame graph libraries (such as those in the Godot 4 rendering device, Ch41) perform this dependency analysis on the CPU today; GPU acceleration becomes relevant when the number of render passes exceeds several thousand (e.g., in voxel GI probe updates or deferred decal passes).

### 12.3 GPU BFS for Portal/PVS Precomputation

Potentially visible set (PVS) precomputation determines which leaf cells of a BSP tree are visible from each other via the portal graph. The portal graph is a small graph (hundreds to thousands of portals in typical indoor scenes) where nodes are portals and edges connect portals that share a cell. GPU BFS on the portal graph from every leaf cell computes the full PVS in parallel, with one BFS instance per cell launched as independent grid dimensions.

For outdoor scenes using GPU-driven occlusion queries (Ch209), the BFS over the scene's spatial graph (cells connected if within view distance) prunes the query set before the hardware occlusion pass.

### 12.4 Mesh Island Detection before GPU Delaunay

GPU Delaunay triangulation (Ch222) assumes the input point set is a single connected domain. When processing a mesh with multiple disconnected islands, running Delaunay on the full point set produces unwanted inter-island triangles. WCC (§4.4) identifies the islands; each island is then triangulated independently:

```python
# Detect mesh islands using cuGraph WCC on vertex adjacency graph
import cugraph, cudf

# Build vertex adjacency graph from mesh edges
G = cugraph.Graph()
G.from_cudf_edgelist(mesh_edges_df, source='v0', destination='v1')
components = cugraph.weakly_connected_components(G)
# components['labels'] assigns each vertex an island ID
# Process each island independently for Delaunay triangulation
for island_id in components['labels'].unique().to_pandas():
    island_verts = components[components['labels'] == island_id]['vertex']
    triangulate_island(island_verts)
```

This pattern also applies before UV unwrapping (seam boundaries define disconnected chart domains) and before skeleton extraction where mesh self-intersections create spurious connections.

---

## Integrations

**Ch209 — GPU Spatial Structures, Differential Geometry, and Animation.** BVHs are implicit binary tree graphs; BVH traversal is a graph search. Precomputed visibility (PVS) structures and sparse voxel DAGs are graph structures to which BFS connectivity analysis (§2) applies directly.

**Ch210 — GPU Physics and Volumetric Methods.** Physics simulation islands are connected components of the contact graph. The union-find structure (§4.3) and GPU WCC (§4.4) identify islands for independent constraint solving. The Shiloach-Vishkin algorithm (§4.1) is directly applicable to contact graph analysis.

**Ch222 — Computational Geometry Algorithms on GPU.** GPU Delaunay triangulation produces a triangulation graph; triangle counting (§8) evaluates the quality of that triangulation. Mesh island detection via WCC (§4.4, §12.4) preconditions the Delaunay input.

**Ch224 — 3D Shape Analysis Algorithms.** Graph-based shape segmentation (normalised cuts on the mesh dual graph) uses the spectral methods described in that chapter. GNN inference for mesh segmentation (§10.5) consumes the shape analysis pipeline's output as initial vertex features.

**Ch226 — Sparse Linear Algebra on GPU.** PageRank (§5.1) is one iteration of SpMV; the chapter covers the cuSPARSE and rocSPARSE APIs used in this formulation. Graph Laplacian construction and solve for Laplacian mesh editing and AMG smoothers (§7.2) are SpMV operations treated in depth in Ch226. Graph coloring (§7) determines the independent-set ordering for Gauss-Seidel smoothers in AMG solvers from Ch226.

**Ch229 — GPU GNN Inference.** Section §10 of this chapter covers message-passing GNN inference at the operator level (scatter-gather over CSR, DGL `update_all`, PyG `MessagePassing`). Ch229 covers end-to-end GNN model deployment — model serialisation (ONNX, TorchScript), runtime optimisation (TensorRT-GNN, DGL inference server), and batching strategies for production inference workloads.
