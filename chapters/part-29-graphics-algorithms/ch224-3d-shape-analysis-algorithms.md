# Chapter 224: 3D Shape Analysis Algorithms

**Audiences:** Graphics application developers implementing shape matching, retrieval, or classification; CAD/CAM engineers building shape libraries; robotics and AR developers performing object recognition from point clouds or meshes; researchers applying geometric deep learning to 3D data.

**Relationship to Ch208–212.** The GPU Geometry Algorithms series (Ch208–Ch212) covers mesh *construction*, *deformation*, and *acceleration structures*: subdivision, BVH building, spatial queries, and physics simulation. This chapter covers the complementary domain of *shape understanding* — algorithms that extract semantic or geometric descriptors from a shape, compare shapes, or detect structure within a shape. The GPU implementation angle (parallel descriptor computation, batch shape comparison) is the connective thread.

**CPU vs GPU split.** Many shape analysis algorithms decompose into a topology-sensitive phase (graph traversal, eigendecomposition, sparse linear solve) that runs on the CPU, and a data-parallel phase (per-vertex feature computation, histogram accumulation, distance matrix evaluation) that runs on the GPU. The sections below call out which phase runs where and show the data-transfer pattern across the boundary.

---

## Table of Contents

1. [Shape Analysis as a GPU Problem](#1-shape-analysis-as-a-gpu-problem)
2. [Local Shape Descriptors](#2-local-shape-descriptors)
3. [Global Shape Descriptors](#3-global-shape-descriptors)
4. [Spectral Shape Analysis](#4-spectral-shape-analysis)
5. [Shape Matching and Correspondence](#5-shape-matching-and-correspondence)
6. [Symmetry Detection](#6-symmetry-detection)
7. [Skeleton Extraction](#7-skeleton-extraction)
8. [Shape Segmentation and Part Decomposition](#8-shape-segmentation-and-part-decomposition)
9. [Shape Completion and Inpainting](#9-shape-completion-and-inpainting)
10. [Shape Retrieval and Classification](#10-shape-retrieval-and-classification)
11. [Geometric Deep Learning on Meshes](#11-geometric-deep-learning-on-meshes)
12. [Production Libraries for Shape Analysis on Linux](#12-production-libraries-for-shape-analysis-on-linux)
- [Integrations](#integrations)

---

## 1. Shape Analysis as a GPU Problem

Shape *construction* algorithms — subdivision, Boolean mesh CSG, BVH building — take raw geometric primitives and produce a well-formed surface representation. Shape *understanding* algorithms take that surface and answer questions: What does this shape look like regardless of its pose? Where are its parts? Does it match a known object in a database?

**The embarrassingly parallel core.** Most shape descriptors reduce to per-vertex or per-patch computations over a local neighbourhood. Given a mesh with N vertices, computing a 33-dimensional FPFH descriptor at every vertex is N independent tasks with identical instruction sequences — a perfect fit for the SIMT programming model. At N = 1 million vertices (a typical high-quality scan), a naive CPU loop at 100 ns per descriptor takes 100 ms; a GPU kernel launching 64-wide warps processes 16 K vertices per wavegroup and finishes in under 5 ms on modern discrete hardware.

**Challenges.** Three properties of shape make analysis harder than raw geometry processing:

- *Scale-invariance.* A descriptor should not change when the shape is uniformly scaled. FPFH achieves this by normalising histogram bins by the number of points; 3D Zernike moments are normalised by the zeroth-order coefficient.
- *Pose-invariance (isometry).* For non-rigid shapes (e.g., a person in different poses), a useful descriptor should be invariant to isometric deformations — bending that preserves geodesic distances. Spectral descriptors (HKS, WKS) derived from the Laplace-Beltrami operator have this property analytically.
- *Partial matching.* Sensor data is often occluded or incomplete. Descriptors must remain discriminative when only a fraction of the surface is visible.

**The shape analysis pipeline.**

```
Point cloud / mesh
       │
       ▼
Preprocessing: voxel downsampling, normal estimation
       │
       ▼
Descriptor extraction (GPU-parallel per vertex/patch)
       │
       ▼
Feature matching / comparison (GPU k-NN or matrix multiply)
       │
       ▼
Classification / retrieval / correspondence
```

The bottleneck shifts across stages depending on problem size. For retrieval over a database of millions of shapes, the GPU k-NN search dominates. For a single-shape segmentation via spectral clustering, the eigendecomposition of the mesh Laplacian dominates, and that is typically CPU-bound unless an iterative GPU eigensolver is used.

**Key GPU primitives.**

| Primitive | API | Shape analysis use |
|---|---|---|
| Parallel reduction | GLSL `subgroupAdd`, CUDA `__reduce_add_sync` | Histogram accumulation, centroid, extreme points |
| Prefix scan | `thrust::exclusive_scan`, compute shader | Compaction of neighbour lists, variable-length output |
| Sparse matrix-vector multiply (SpMV) | cuSPARSE `cusparseSpMV`, rocSPARSE `rocsparse_spmv` | Laplacian solve, graph Laplacian eigenvectors, ARAP step |
| Dense matrix multiply (GEMM) | cuBLAS `cublasSgemm`, rocBLAS `rocblas_sgemm` | Functional map computation, SH projection |
| Radix sort | `thrust::sort_by_key` | k-NN index sorting, histogram bin ordering |

---

## 2. Local Shape Descriptors

Local descriptors characterise the geometry in the neighbourhood of a single point. They are the building blocks of shape matching, feature-based registration, and keypoint detection.

### 2.1 FPFH: Fast Point Feature Histogram

FPFH was introduced to enable real-time feature-based point cloud registration. [Source: Open3D Global Registration Tutorial, https://www.open3d.org/docs/latest/tutorial/pipelines/global_registration.html] The algorithm proceeds in three stages:

**Stage 1 — Normal estimation.** At each query point p, collect the k nearest neighbours (k-NN from a prebuilt KD-tree), form the 3×3 covariance matrix of neighbour positions, and take the eigenvector with the smallest eigenvalue as the surface normal. Normal orientation is made consistent by propagating orientation through a minimum spanning tree of the point cloud.

**Stage 2 — SPFH (Simplified Point Feature Histogram).** For each query point p and each neighbour q_i, define a local Darboux frame (u, v, w) at p:

```
u = n_p
v = (q_i - p) / |q_i - p| × u
w = u × v
```

Compute three angular features:
- α = v · n_q
- φ = (u · (q_i − p)) / |q_i − p|
- θ = arctan(w · n_q, u · n_q)

Bin each feature into 11 bins, producing a 33-dimensional SPFH histogram at p.

**Stage 3 — FPFH.** Weight and sum the SPFH histograms of k neighbours to form the final 33-dimensional FPFH descriptor:

```
FPFH(p) = SPFH(p) + (1/k) Σ_i (1/w_i) SPFH(q_i)
```

where w_i = |q_i − p| is the distance weight.

**Open3D implementation.**

```python
import open3d as o3d

def preprocess_point_cloud(pcd, voxel_size):
    # Downsample
    pcd_down = pcd.voxel_down_sample(voxel_size)

    # Normal estimation: radius = 2× voxel_size, at most 30 neighbours
    radius_normal = voxel_size * 2
    pcd_down.estimate_normals(
        o3d.geometry.KDTreeSearchParamHybrid(radius=radius_normal, max_nn=30))

    # FPFH: radius = 5× voxel_size, at most 100 neighbours
    radius_feature = voxel_size * 5
    pcd_fpfh = o3d.pipelines.registration.compute_fpfh_feature(
        pcd_down,
        o3d.geometry.KDTreeSearchParamHybrid(radius=radius_feature, max_nn=100))
    return pcd_down, pcd_fpfh
```
[Source: Open3D Global Registration Tutorial, https://www.open3d.org/docs/latest/tutorial/pipelines/global_registration.html]

**GPU parallelism.** The SPFH computation for each query point is independent of all others. A GPU kernel launches one thread block per query point. Each thread block:
1. Loads the query point and its k-NN indices (from a prebuilt BVH or KD-tree stored in a GPU buffer).
2. Computes the Darboux frame and three angular features for each neighbour (one thread per neighbour within the block).
3. Accumulates the three 11-bin histograms via shared-memory atomics.
4. Writes the 33-dimensional SPFH vector to global memory.

The FPFH aggregation pass is a second kernel that reads back the SPFH buffer and computes the weighted sum over k-NN SPFH entries — another embarrassingly parallel per-vertex operation.

```glsl
// GLSL compute — SPFH accumulation kernel (one workgroup per query point)
#version 460
#extension GL_KHR_shader_subgroup_arithmetic : require
layout(local_size_x = 32) in;  // one thread per neighbour (k ≤ 32)

layout(std430, binding = 0) readonly buffer Points   { vec3  pts[];   };
layout(std430, binding = 1) readonly buffer Normals  { vec3  nrms[];  };
layout(std430, binding = 2) readonly buffer KNN      { uint  knn[];   }; // [N * K]
layout(std430, binding = 3) writeonly buffer SPFH    { float spfh[];  }; // [N * 33]

layout(push_constant) uniform PC { uint N; uint K; };

shared float hist[33]; // 3 features × 11 bins

void main() {
    uint pidx = gl_WorkGroupID.x;          // query point index
    uint tid  = gl_LocalInvocationID.x;   // neighbour index within workgroup

    if (tid < 33) hist[tid] = 0.0;
    barrier();

    if (tid < K && pidx < N) {
        uint qidx = knn[pidx * K + tid];
        vec3 p  = pts[pidx];
        vec3 np = nrms[pidx];
        vec3 q  = pts[qidx];
        vec3 nq = nrms[qidx];

        vec3 diff = normalize(q - p);
        vec3 u    = np;
        vec3 v    = cross(diff, u);
        vec3 w    = cross(u, v);

        float alpha = dot(v, nq);                          // ∈ [−1,1]
        float phi   = dot(u, diff);                        // ∈ [−1,1]
        float theta = atan(dot(w, nq), dot(u, nq));       // ∈ [−π,π]

        // Bin each feature into 11 bins
        uint a_bin = uint(clamp((alpha + 1.0) * 5.5, 0.0, 10.0));
        uint p_bin = uint(clamp((phi   + 1.0) * 5.5, 0.0, 10.0));
        uint t_bin = uint(clamp((theta / 3.14159 + 1.0) * 5.5, 0.0, 10.0));

        atomicAdd(uint(hist[a_bin]),       1);   // illustrative: use floatBitsToUint trick
        atomicAdd(uint(hist[11 + p_bin]),  1);
        atomicAdd(uint(hist[22 + t_bin]),  1);
    }
    barrier();

    // Write SPFH to global buffer
    if (tid < 33) spfh[pidx * 33 + tid] = hist[tid];
}
// Note: production kernels use floatBitsToUint / atomicAdd with uint reinterpretation,
// then convert to float after accumulation. Illustrative only.
```

### 2.2 SHOT: Signature of Histograms of Orientations

SHOT produces a 352-dimensional descriptor by binning local normal orientations into a spherical grid partitioned into 32 spatial bins (2 radial × 2 elevation × 8 azimuthal) and 11 cosine-of-angle bins, giving 32 × 11 = 352 values. [Source: Open3D Python API — PointCloud, https://www.open3d.org/docs/latest/python_api/open3d.geometry.PointCloud.html]

The key innovation over FPFH is the *local reference frame* (LRF): a full 3D orthonormal frame computed at each point by eigendecomposition of the weighted scatter matrix of neighbours. This makes SHOT more repeatable under noise than FPFH but at higher computational cost (the LRF eigensystem requires 9 floats of state per point vs. three angular features for FPFH).

GPU parallelism mirrors FPFH: one thread block per query point, threads cooperating on the k-NN neighbourhood, with shared-memory histogram accumulation across the 352-element descriptor.

### 2.3 3D SIFT via Laplace-Beltrami Scale Space

The 2D SIFT detector finds keypoints as extrema of a Difference-of-Gaussian (DoG) scale space. The mesh analogue replaces the Gaussian with the *heat diffusion operator* exp(−t L) where L is the discrete Laplace-Beltrami operator. At scale t, a smoothed signal f_t = exp(−t L) f; the DoG approximation is f_t − f_{2t}. Keypoints are vertices where |f_t(v) − f_{2t}(v)| exceeds a threshold and is a local extremum among all neighbours.

On the GPU, each diffusion step is a sparse matrix-vector multiply (SpMV) of the cotangent Laplacian against the current signal buffer, which maps directly to cuSPARSE or rocSPARSE SpMV.

### 2.4 ROPS: Rotational Projection Statistics

ROPS projects the local neighbourhood of a point onto three planes (XY, XZ, YZ) in the local reference frame and computes distribution statistics (mean, variance, skewness, entropy of the 2D histogram) of the projected point counts. [Source: PCL ROPS Feature Documentation, https://pcl.readthedocs.io/projects/tutorials/en/latest/rops_feature.html] The result is a 135-dimensional vector.

Each projection plane is independent: a GPU thread block computes all three projections in parallel using different warps, accumulating per-bin counts in shared memory before computing the five statistics per bin.

---

## 3. Global Shape Descriptors

Global descriptors characterise the entire shape with a single feature vector, enabling coarse shape matching and database retrieval without establishing point correspondences.

### 3.1 D2 Shape Distribution

The D2 shape distribution is the histogram of Euclidean distances between randomly sampled pairs of surface points. [Source: Osada et al., "Shape Distributions," ACM Trans. Graphics 2002, https://dl.acm.org/doi/10.1145/571647.571648] It is compact (typically 64–256 histogram bins), robust to pose variation (distances are invariant to rigid-body transforms), and easy to compute:

**GPU Monte Carlo sampling.** Each GPU thread generates one random pair of surface sample indices using a linear-congruential generator seeded from the thread ID, fetches the two 3D positions from a surface position buffer, computes the Euclidean distance, and atomically increments the corresponding histogram bin. One million samples requires fewer than 2 ms on a modern GPU.

```glsl
// GLSL — D2 shape distribution kernel
#version 460
layout(local_size_x = 256) in;

layout(std430, binding = 0) readonly  buffer SurfPts  { vec3  pts[];        };
layout(std430, binding = 1) coherent  buffer Histogram { uint  histogram[];  }; // [NUM_BINS]

layout(push_constant) uniform PC {
    uint n_pts;
    uint n_samples;
    uint n_bins;
    float max_dist;
    uint rng_seed;
};

uint lcg(uint state) { return state * 1664525u + 1013904223u; }

void main() {
    uint gid = gl_GlobalInvocationID.x;
    if (gid >= n_samples) return;

    // Two independent LCG streams
    uint s0 = lcg(gid ^ rng_seed);
    uint s1 = lcg(s0 ^ 0xDEADBEEFu);

    uint i = s0 % n_pts;
    uint j = s1 % n_pts;
    float d = distance(pts[i], pts[j]);

    uint bin = uint(d / max_dist * float(n_bins));
    bin = min(bin, n_bins - 1u);
    atomicAdd(histogram[bin], 1u);
}
```

Shape comparison uses the L1 or chi-squared distance between normalised histograms. At retrieval time the database descriptors are stored as float arrays and compared in a batched GPU dot-product or distance kernel.

### 3.2 3D Zernike Moments

3D Zernike moments expand a volumetric indicator function f(r, θ, φ) — the signed occupancy of the shape — in the orthonormal 3D Zernike basis {Z_n^l^m} defined on the unit ball. The basis functions factor as:

```
Z_n^l^m(r, θ, φ) = R_n^l(r) · Y_l^m(θ, φ)
```

where R_n^l are radial polynomials and Y_l^m are spherical harmonics. The rotation-invariant descriptor is the set of norms |Ω_n^l| = sqrt(Σ_m |c_n^l^m|²) where c_n^l^m is the projection coefficient. [Source: Novotni and Klein, "3D Zernike Descriptors for Content Based Shape Retrieval," Proc. Solid Modeling 2003, https://dl.acm.org/doi/10.1145/781606.781639]

GPU computation voxelises the shape to a 64³ grid (producing 262,144 voxels) and launches a kernel that evaluates the projection integral c_n^l^m = Σ_{voxels inside shape} f(v) · conj(Z_n^l^m(v)) for each (n, l, m) triple in parallel. For order N=20 there are 121 basis functions; the full projection is a (262144 × 121) matrix multiply, well-suited to a GPU GEMM call.

### 3.3 Spherical Harmonic Descriptor

The spherical harmonic (SH) descriptor projects a scalar function on the unit sphere — typically the distribution of surface normals, or a distance histogram binned by direction — onto the SH basis. The rotation-invariant descriptor is the energy in each frequency band l: E_l = Σ_m |c_l^m|², giving a descriptor of length L_max for order L_max.

**Extended Gaussian Image (EGI).** The EGI is the distribution of outward surface normals mapped to the unit sphere: each surface patch of area A contributes weight A to the point on S² coinciding with its normal direction. The SH descriptor of the EGI is rotation-sensitive; the energy-band norms are rotation-invariant.

**GPU SH projection.** A surface normal buffer of N entries is projected onto (L_max+1)² SH basis values in parallel: each thread evaluates one normal's contribution to all SH coefficients, then a parallel reduction accumulates across threads. For L_max=16, this is 289 SH evaluations per normal — again structurally a (N × 289) matrix multiply.

```python
# PyTorch — batched SH projection (illustrative, using precomputed SH basis matrix)
import torch

def compute_sh_descriptor(normals: torch.Tensor,  # (N, 3) float32 on GPU
                           sh_basis: torch.Tensor, # (289, 3) precomputed for each normal dir
                           L_max: int = 16) -> torch.Tensor:
    """
    Project N surface normals onto spherical harmonic basis, return
    rotation-invariant energy per band. Output shape: (L_max + 1,)
    """
    # Evaluate SH basis at each normal direction: (N, n_coeffs)
    # (In practice, sh_basis is evaluated per normal; shown here as precomputed)
    coeffs = normals @ sh_basis  # (N, n_coeffs) via matmul
    coeffs = coeffs.mean(dim=0)  # (n_coeffs,) mean over surface
    n_coeffs = (L_max + 1) ** 2
    energy = torch.zeros(L_max + 1, device=normals.device)
    idx = 0
    for l in range(L_max + 1):
        band_coeffs = coeffs[idx : idx + 2*l + 1]
        energy[l] = (band_coeffs ** 2).sum().sqrt()
        idx += 2*l + 1
    return energy  # (L_max + 1,) rotation-invariant descriptor
```

---

## 4. Spectral Shape Analysis

Spectral methods analyse a shape through the eigenstructure of the Laplace-Beltrami operator (LBO). Because the LBO is intrinsic — defined entirely by geodesic distances, independent of how the shape is embedded in 3D space — its eigenfunctions yield isometry-invariant descriptors.

### 4.1 The Discrete Laplace-Beltrami Operator

For a triangle mesh with vertex positions V ∈ ℝ^{N×3} and faces F ∈ ℤ^{M×3}, the cotangent discretisation of the LBO is:

```
L_{ij} = (1/2)(cot α_ij + cot β_ij)   for edge (i,j)
L_{ii} = −Σ_{j∈N(i)} L_{ij}
```

where α_ij and β_ij are the two angles opposite to edge (i,j) in the two triangles sharing it. [Source: Botsch et al., "Polygon Mesh Processing," AK Peters 2010; libigl tutorial §304, https://libigl.github.io/tutorial/]

libigl computes the cotangent Laplacian and the lumped mass matrix as:

```cpp
#include <igl/cotmatrix.h>
#include <igl/massmatrix.h>
#include <igl/eigs.h>
#include <Eigen/Sparse>

Eigen::MatrixXd V;          // (N, 3) vertex positions
Eigen::MatrixXi F;          // (M, 3) face indices
Eigen::SparseMatrix<double> L, M;

igl::cotmatrix(V, F, L);    // L: (N×N) negative semi-definite cotangent Laplacian
igl::massmatrix(V, F, igl::MASSMATRIX_TYPE_VORONOI, M); // M: (N×N) diagonal mass matrix

// Solve generalised eigenvalue problem: L φ = λ M φ
// Extract k smallest-magnitude eigenpairs
Eigen::MatrixXd U;    // (N, k) eigenvectors
Eigen::VectorXd S;    // (k,)  eigenvalues
igl::eigs(L, M, k, igl::EIGS_TYPE_SM, U, S);
```
[Source: libigl tutorial §304 — Laplace Equation, https://libigl.github.io/tutorial/]

### 4.2 Heat Kernel Signature (HKS)

The heat kernel K_t(x, y) describes how heat diffuses from point y to point x in time t on the surface. The *heat kernel signature* evaluated at a single point x is the auto-heat-kernel:

```
HKS(x, t) = K_t(x, x) = Σ_i φ_i(x)² · exp(−λ_i · t)
```

where {φ_i, λ_i} are eigenfunctions and eigenvalues of the LBO. [Source: Sun et al., "A Concise and Provably Informative Multi-Scale Signature Based on Heat Diffusion," SGP 2009, https://doi.org/10.1111/j.1467-8659.2009.01515.x]

Evaluated at a discrete set of time scales T = {t_1, …, t_K} (logarithmically spaced), HKS produces a K-dimensional descriptor per vertex. Because the LBO is isometry-invariant, HKS is invariant to isometric deformations (bending without stretching).

```python
import numpy as np

def compute_hks(eigenvectors: np.ndarray,  # (N, k) φ_i(x)
                eigenvalues: np.ndarray,   # (k,) λ_i
                t_min: float = 0.1,
                t_max: float = 1000.0,
                n_times: int = 100) -> np.ndarray:
    """
    Compute Heat Kernel Signature for all vertices.
    Returns (N, n_times) array.
    """
    times = np.geomspace(t_min, t_max, n_times)           # (n_times,)
    # phi_sq[v, i] = φ_i(v)²
    phi_sq = eigenvectors ** 2                              # (N, k)
    # exp_factor[i, t] = exp(−λ_i · t)
    exp_factor = np.exp(-eigenvalues[:, None] * times[None, :])  # (k, n_times)
    # HKS[v, t] = Σ_i phi_sq[v,i] * exp_factor[i, t]
    hks = phi_sq @ exp_factor                               # (N, n_times)
    return hks
```

### 4.3 Wave Kernel Signature (WKS)

The Wave Kernel Signature replaces heat diffusion with the Schrödinger equation. At energy level e, the WKS is:

```
WKS(x, e) = Σ_i φ_i(x)² · exp(−(e − log λ_i)² / (2σ²))
```

evaluated at E discrete energy levels {e_1, …, e_E} logarithmically spaced in the spectrum of the LBO. [Source: Aubry et al., "The Wave Kernel Signature: A Quantum Mechanical Approach to Shape Analysis," ICCV Workshops 2011, https://doi.org/10.1109/ICCVW.2011.6130444]

WKS is more discriminative than HKS near high-frequency features because it uses a Gaussian energy filter rather than an exponential decay, giving each energy level a band-pass character. Both HKS and WKS depend only on the eigenfunctions already computed for the LBO, so computing both costs only an additional matrix multiply once the eigendecomposition is done.

### 4.4 GPU Eigensolver: LOBPCG

Full eigendecomposition of an N×N sparse matrix via ARPACK is CPU-bound and scales as O(k·N·nnz) for k eigenpairs, where nnz is the number of non-zeros in L. For N > 500K vertices, this becomes the performance bottleneck.

LOBPCG (Locally Optimal Block Preconditioned Conjugate Gradient) is an iterative eigensolver that uses SpMV as its inner loop, making it GPU-compatible via cuSPARSE or rocSPARSE. The algorithm maintains a block of candidate eigenvectors X (N × block_size), applies L and M to X in each iteration, and refines the estimates until convergence. [Source: Knyazev, "Toward the Optimal Preconditioned Eigensolver," SIAM J. Sci. Comput. 2001, https://doi.org/10.1137/S1064827500366124]

PyTorch's `torch.lobpcg` implements LOBPCG on GPU tensors:

```python
import torch
import torch.sparse

# L_sparse: (N, N) sparse COO or CSR tensor on GPU (cotangent Laplacian)
# M_diag:   (N,) diagonal of mass matrix as dense tensor

N = L_sparse.shape[0]
k = 50  # number of eigenpairs

# Initial random guess
X0 = torch.randn(N, k, device='cuda', dtype=torch.float64)

eigenvalues, eigenvectors = torch.lobpcg(
    A=L_sparse,
    k=k,
    B=None,            # omit mass weighting here for simplicity
    X=X0,
    n=k,
    tol=1e-6,
    largest=False      # smallest eigenvalues (lowest frequencies)
)
# eigenvectors: (N, k) on GPU; eigenvalues: (k,)
```
[Source: PyTorch torch.lobpcg documentation, https://pytorch.org/docs/stable/generated/torch.lobpcg.html]

For the mass-weighted problem L φ = λ M φ, the preconditioned form M^{-1} L φ = λ φ is used, since M is diagonal and its inverse is trivial to apply.

---

## 5. Shape Matching and Correspondence

Establishing a correspondence between two shapes — finding a map f: X → Y such that f(x) and x are geometrically related — is the fundamental problem underpinning registration, shape interpolation, and texture transfer.

### 5.1 ICP Variants and Correspondence Quality

ICP iterates two steps: find the closest-point correspondence between source and target, then solve for the rigid transform that minimises the alignment error. Ch211 §90 covers the GPU acceleration of ICP in detail; this section focuses on correspondence quality strategies.

**Point-to-point ICP** minimises Σ ||T(s_i) − t_i||², where t_i = NN(T(s_i)) is the target nearest neighbour. This converges slowly near the optimum when the target has a different sampling density.

**Point-to-plane ICP** minimises Σ (n_i · (T(s_i) − t_i))², the squared residual along the target normal. Convergence is faster because the target normal provides a first-order approximation to the surface. [Source: Open3D ICP Registration Tutorial, https://www.open3d.org/docs/latest/tutorial/t_pipelines/t_icp_registration.html]

```python
import open3d as o3d
import open3d.t as o3dt

source = o3dt.geometry.PointCloud(...)
target = o3dt.geometry.PointCloud(...)

# Point-to-plane ICP (requires target normals)
result = o3dt.pipelines.registration.icp(
    source, target,
    max_correspondence_distance=0.05,
    estimation=o3dt.pipelines.registration.TransformationEstimationPointToPlane(),
    criteria=o3dt.pipelines.registration.ICPConvergenceCriteria(
        relative_fitness=1e-6, relative_rmse=1e-6, max_iteration=50))

print(result.transformation)  # (4×4) float64 tensor
```
[Source: Open3D Tensor ICP API, https://www.open3d.org/docs/latest/python_api/open3d.t.pipelines.registration.html]

### 5.2 Functional Maps

Functional maps represent a correspondence between two shapes X and Y not as a point-to-point map but as a linear operator C that converts functions on X to functions on Y in the LBO eigenfunction basis. [Source: Ovsjanikov et al., "Functional Maps: A Flexible Representation of Maps Between Shapes," SIGGRAPH 2012, http://www.lix.polytechnique.fr/~maks/papers/functional_maps.pdf]

If Φ_X = [φ_1^X, …, φ_k^X] is the N_X × k matrix of LBO eigenfunctions on X, and similarly Φ_Y for Y, and if we have a set of corresponding descriptor functions {f_i^X, f_i^Y} (e.g., WKS at the same energy levels), then the functional map C is the k×k matrix satisfying:

```
Φ_Y^T f^Y = C · Φ_X^T f^X
```

This is solved as a least-squares problem for C. On the GPU it is a dense (k×k) linear least-squares with k typically 50–200, so the solve is fast:

```python
import torch

def compute_functional_map(
        phi_x: torch.Tensor,   # (N_X, k) LBO eigenfunctions on shape X
        phi_y: torch.Tensor,   # (N_Y, k) LBO eigenfunctions on shape Y
        f_x:   torch.Tensor,   # (N_X, D) descriptor functions on X (e.g., WKS)
        f_y:   torch.Tensor,   # (N_Y, D) descriptor functions on Y
) -> torch.Tensor:
    """
    Solve for functional map C (k×k) such that Φ_Y^T f_Y ≈ C Φ_X^T f_X.
    All tensors on GPU.
    """
    A = phi_x.T @ f_x   # (k, D) — spectral coefficients of f_x
    B = phi_y.T @ f_y   # (k, D) — spectral coefficients of f_y

    # Solve B = C @ A  =>  C = B @ A^T @ (A @ A^T)^{-1}  (least-squares)
    C, _ = torch.linalg.lstsq(A.T, B.T)  # (k, k)
    return C.T
```

Converting the functional map back to a point-to-point map uses the *nearest-neighbour in spectral embedding*: for each point y in Y, find the point x in X whose spectral embedding C · Φ_X(x) is closest to Φ_Y(y).

### 5.3 Non-Rigid Shape Matching via ARAP

As-Rigid-As-Possible (ARAP) deformation models non-rigid shape matching as an energy minimisation that penalises deviation from local rigidity. For each vertex i, let R_i ∈ SO(3) be a local rotation and let p_i and p'_i be original and deformed positions. The ARAP energy is:

```
E_ARAP = Σ_i Σ_{j∈N(i)} w_ij ||( p'_i − p'_j) − R_i (p_i − p_j)||²
```

where w_ij = (1/2)(cot α_ij + cot β_ij) are the cotangent weights. [Source: Sorkine and Alexa, "As-Rigid-As-Possible Surface Modeling," SGP 2007, https://doi.org/10.2312/SGP/SGP07/109-116]

Minimisation alternates two steps:
1. **Local step**: Given current positions {p'_i}, find the best local rotation R_i for each vertex independently by SVD of a 3×3 matrix. N independent 3×3 SVDs — each on one GPU thread.
2. **Global step**: Given {R_i}, solve a sparse linear system (the Laplacian system with right-hand side depending on R_i) for the new positions {p'_i}. This is a GPU SpMV solve via cuSPARSE or rocSPARSE `csrsv2` / `rocsparse_csrsv_solve`.

The alternation continues until the energy change falls below a threshold (typically 10–20 iterations suffice).

---

## 6. Symmetry Detection

Symmetry detection identifies geometric self-similarity at the global or local level — mirror planes, rotation axes, or repeating motifs. It underpins shape abstraction, repair of corrupted scans, and procedural geometry.

### 6.1 Reflective Symmetry via EGI Voting

A global mirror plane has a normal direction n such that reflecting the shape across the plane maps the shape onto itself. The extended Gaussian image (EGI, §3.3) provides a compact signature: the EGI of a bilaterally symmetric shape is itself symmetric about the plane's normal on S².

The voting algorithm accumulates votes in the space of possible mirror normals (discretised as a 64×128 azimuth/elevation grid on S²):

1. For each pair of surface normals (n_i, n_j), the candidate mirror normal is n_mirror = normalise(n_i + n_j). Increment the vote count at the corresponding grid cell.
2. After processing all pairs, find the grid cell with the most votes.
3. Refine via gradient ascent on the vote accumulator.

Step 1 is O(M²) where M is the number of normal samples, typically 10³ to 10⁴ for a downsampled EGI. A GPU kernel launches one thread per pair, with each thread computing n_mirror and performing an atomic increment at the corresponding grid bin:

```glsl
// GLSL — reflective symmetry EGI voting kernel
#version 460
layout(local_size_x = 256) in;

layout(std430, binding = 0) readonly buffer Normals  { vec3  nrms[];         };
layout(std430, binding = 1) coherent buffer VoteGrid { uint  votes[64*128];  };

layout(push_constant) uniform PC { uint M; }; // number of EGI normals

uint normal_to_bin(vec3 n) {
    float az  = atan(n.y, n.x) / 3.14159 * 0.5 + 0.5;  // [0,1]
    float el  = asin(clamp(n.z, -1.0, 1.0)) / 3.14159 + 0.5; // [0,1]
    uint col  = uint(clamp(az * 128.0, 0.0, 127.0));
    uint row  = uint(clamp(el * 64.0,  0.0, 63.0));
    return row * 128u + col;
}

void main() {
    uint gid = gl_GlobalInvocationID.x;
    uint total_pairs = M * (M - 1u) / 2u;
    if (gid >= total_pairs) return;

    // Decode pair (i, j) from gid using triangular index formula
    uint i = uint((sqrt(8.0 * float(gid) + 1.0) - 1.0) / 2.0);
    uint j = gid - i * (i + 1u) / 2u;

    vec3 mirror = normalize(nrms[i] + nrms[j]);
    atomicAdd(votes[normal_to_bin(mirror)], 1u);
}
```

### 6.2 Rotational Symmetry Detection

A k-fold rotational symmetry about axis a has k evenly spaced rotation angles {2πm/k : m=0,…,k−1}. Detection proceeds by voting in the space of (axis, angle) pairs. The Hough-style vote accumulator is a 3D array indexed by (azimuth of a, elevation of a, candidate k).

For partial symmetry (e.g., a vase with a handle that breaks global symmetry), RANSAC generates symmetry hypotheses by randomly sampling small surface patches, estimating the symmetry transform that maps one patch to another, and scoring the hypothesis by measuring alignment between the entire shape and its transformed copy. Each RANSAC hypothesis verification step is a mini-ICP run on the GPU — a natural parallelism opportunity when scoring thousands of hypotheses simultaneously across different GPU streams.

### 6.3 Intrinsic Symmetry via Diffusion Distance

Two points x, y on a shape are intrinsically symmetric if there exists an isometric automorphism mapping x to y. The *diffusion distance* d_t(x, y) = Σ_i exp(−2λ_i t) (φ_i(x) − φ_i(y))² is isometry-invariant and can reveal intrinsic symmetry: the all-pairs diffusion distance matrix D has an approximate block structure for shapes with intrinsic symmetry. [Source: Coifman and Lafon, "Diffusion Maps," Applied and Computational Harmonic Analysis 2006, https://doi.org/10.1016/j.acha.2006.04.006]

Computing D for N vertices is O(N² · k) where k is the number of eigenfunctions retained, reducible to a GPU GEMM of shape (N × k) by (k × N). For N = 10K and k = 50, this is a (10K × 50) × (50 × 10K) multiply — a 5 billion-operation dense matmul well-suited to a single cuBLAS `cublasSgemm` call.

---

## 7. Skeleton Extraction

Shape skeletons capture the topological and geometric structure of a shape as a lower-dimensional object: a curve network (1D) for elongated shapes or a graph embedded in 3D.

### 7.1 Curve Skeleton via Mean Curvature Flow Contraction

The mean curvature flow (MCF) contracts a mesh iteratively along its mean curvature normal until it degenerates to a 1D structure. The contraction step solves:

```
(I + dt · W^{-1} L) V_{t+1} = V_t
```

where L is the cotangent Laplacian and W is the diagonal area matrix. [Source: Au et al., "Skeleton Extraction by Mesh Contraction," SIGGRAPH 2008, https://doi.org/10.1145/1399504.1360628]

This is a sparse linear system in 3 independent right-hand sides (the x, y, z coordinates of vertex positions). On the GPU, each coordinate is solved independently via a preconditioned conjugate gradient using cuSPARSE SpMV. Connectivity constraints are added as an additional term to prevent zero-length edges from appearing.

After sufficient contraction (typically 20–100 iterations), the contracted mesh vertices lie near the skeletal curve. A connectivity simplification pass collapses short edges and identifies branch points (degree > 2) and endpoints (degree = 1).

### 7.2 L1-Medial Skeleton for Point Clouds

The L1-medial skeleton algorithm computes a 1D medial structure directly from an unorganised point cloud, without requiring a mesh. [Source: Huang et al., "L1-Medial Skeleton of Point Cloud," SIGGRAPH 2013, https://doi.org/10.1145/2461912.2461913] It places a set of sample points {q_j} and iteratively updates them via locally weighted projection onto a local L1 median.

Each update step for sample q_j is:

```
q_j ← (Σ_i θ(|p_i − q_j|) · p_i / |p_i − q_j|) / (Σ_i θ(|p_i − q_j|) / |p_i − q_j|)
```

where θ is a compactly supported kernel and the sum is over point cloud points p_i within a support radius h. This is embarrassingly parallel across all samples {q_j} and across all supporting points {p_i} for a given sample, enabling a two-level GPU parallelism: one thread block per skeleton sample, threads within a block cooperating on the weighted sum over supporting points.

### 7.3 Reeb Graph Computation

A Reeb graph decomposes a shape using the level sets of a scalar function f: M → ℝ (e.g., the height coordinate or the first LBO eigenfunction). Arcs of the Reeb graph correspond to connected components of level sets; nodes correspond to topological changes. [Source: Biasotti et al., "Reeb Graphs for Shape Analysis and Applications," Theoretical Computer Science 2008, https://doi.org/10.1016/j.tcs.2007.10.017]

GPU acceleration applies to the per-vertex scalar function evaluation and the sorting of vertices by function value. The level-set connectivity tracking uses union-find, which is sequential but operates on at most N/height_resolution elements — manageable on the CPU after downloading the sorted vertex array.

### 7.4 Topology-Preserving Thinning on Voxel Grids

For voxelised shapes, topology-preserving thinning iteratively removes *simple points* — voxels whose removal does not change the Euler characteristic or connectivity of the shape. Parallel thinning alternates between even and odd sub-iterations: even iterations only mark voxels in one half of the voxel grid as candidates, preventing two adjacent simple points from both being removed simultaneously. [Source: Lee, Kashyap, Chu, "Building skeleton models via 3-D medial surface/axis thinning algorithms," Graphical Models and Image Processing 1994, https://doi.org/10.1006/gmip.1994.1042]

A GPU compute shader assigns one thread per voxel. Each thread checks the 26-neighbourhood for the simple-point criterion (typically encoded as a look-up table indexed by the binary pattern of occupied neighbours) and writes a removal flag. A prefix scan then compacts the removed voxels.

---

## 8. Shape Segmentation and Part Decomposition

Shape segmentation partitions a shape's vertices or faces into semantically or geometrically coherent parts, enabling animation rigging, part-based retrieval, and assembly planning.

### 8.1 Random Walk Graph Segmentation

Given a mesh graph with N vertices and edge weights w_ij proportional to geometric similarity (e.g., 1 − |dihedral angle|/π), the random walk algorithm assigns each unlabelled vertex to the label most likely reached by a random walk starting at that vertex. [Source: Grady, "Random Walks for Image Segmentation," IEEE Trans. PAMI 2006, https://doi.org/10.1109/TPAMI.2006.233]

For K labels and a set of seed vertices, the algorithm solves K sparse linear systems:

```
(D − αW) x^k = b^k
```

where D is the diagonal degree matrix, W is the weight matrix, and b^k encodes the seed constraints for label k. Each system has the same left-hand side matrix — a major optimisation: the factorisation is reused across all K right-hand sides.

On the GPU, each system is solved by a preconditioned conjugate gradient using cuSPARSE `cusparseSpMV` for the matrix-vector products. For K ≤ 8 labels and a mesh of 50K vertices, the K independent PCG solvers run concurrently via CUDA streams.

### 8.2 Normalised Cuts Segmentation

Normalised cuts partitions the mesh graph by finding the eigenvectors of the normalised graph Laplacian L_sym = D^{-1/2} L D^{-1/2}. The second and subsequent eigenvectors (the Fiedler vector and its extensions) encode soft partition boundaries; k-means clustering on these eigenvectors gives the final segmentation into K parts. [Source: Shi and Malik, "Normalized Cuts and Image Segmentation," IEEE Trans. PAMI 2000, https://doi.org/10.1109/34.868688]

The eigendecomposition uses `torch.lobpcg` (§4.4) on the GPU-resident sparse Laplacian, followed by a GPU k-means run on the resulting (N × K) eigenvector matrix.

### 8.3 PointNet++ Part Segmentation Inference on GPU

For data-driven segmentation, PointNet++ trains a hierarchical point set learning network that processes raw point coordinates. [Source: Qi et al., "PointNet++: Deep Hierarchical Feature Learning on Point Sets in a Metric Space," NeurIPS 2017, https://doi.org/10.5555/3295222.3295263] At inference time the network pipeline runs entirely on the GPU:

1. **Farthest point sampling** (GPU kernel): select M representative centroids from N input points.
2. **Ball query** (GPU kernel): for each centroid, find all points within radius r — PyTorch3D `ball_query()`.
3. **PointNet local feature extraction** (CUDA kernels for grouped convolutions): for each centroid group, run a shared MLP producing per-group features.
4. **Feature propagation** (GPU interpolation): upsample from M centroid features back to N point features.
5. **Per-point MLP**: classify each point into K part categories.

Using PyTorch Geometric:

```python
import torch
import torch.nn.functional as F
from torch_geometric.nn import PointNetConv, fps, radius, global_max_pool
from torch_geometric.nn import MLP

class PointNet2PartSeg(torch.nn.Module):
    def __init__(self, num_classes: int):
        super().__init__()
        # Set abstraction layer 1
        self.sa1_module = PointNetConv(
            local_nn=MLP([3 + 3, 64, 64, 128]),  # [xyz + features, …, out]
            global_nn=None,
            add_self_loops=False)
        # Set abstraction layer 2
        self.sa2_module = PointNetConv(
            local_nn=MLP([128 + 3, 128, 128, 256]),
            global_nn=None,
            add_self_loops=False)
        # Classification head
        self.mlp = MLP([256, 256, 128, num_classes], dropout=0.5)

    def forward(self, data):
        x, pos, batch = data.x, data.pos, data.batch
        # SA1: farthest point sampling + ball query + PointNet
        idx = fps(pos, batch, ratio=0.5)
        row, col = radius(pos, pos[idx], r=0.2, batch_x=batch, batch_y=batch[idx])
        edge_index = torch.stack([col, row], dim=0)
        x = self.sa1_module(x, (pos, pos[idx]), edge_index)
        pos, batch = pos[idx], batch[idx]
        # SA2
        idx = fps(pos, batch, ratio=0.25)
        row, col = radius(pos, pos[idx], r=0.4, batch_x=batch, batch_y=batch[idx])
        edge_index = torch.stack([col, row], dim=0)
        x = self.sa2_module(x, (pos, pos[idx]), edge_index)
        pos, batch = pos[idx], batch[idx]
        x = global_max_pool(x, batch)
        return F.log_softmax(self.mlp(x), dim=1)
```
[Source: PyTorch Geometric PointNetConv documentation, https://pytorch-geometric.readthedocs.io/en/latest/generated/torch_geometric.nn.conv.PointNetConv.html]

### 8.4 Approximate Convex Decomposition (VHACD)

Approximate convex decomposition decomposes a 3D shape into a small number of convex pieces whose union closely approximates the original shape. The VHACD algorithm (Volumetric Hierarchical Approximate Convex Decomposition) voxelises the shape and recursively splits voxel clusters along the plane that maximises the convexity of the resulting sub-clusters. [Source: Mamou and Ghorbel, "A simple and efficient approach for 3D mesh approximate convex decomposition," ICIP 2009, https://doi.org/10.1109/ICIP.2009.5413588]

GPU parallelism applies to: (1) the voxelisation step (one thread per triangle, writing into a 3D atomic bitmap), and (2) the per-cluster centroid and inertia computation (parallel reduction). The splitting plane search iterates over O(N_voxels^{1/3}) candidate planes per axis — a sequential inner loop but parallelisable across the three axes simultaneously.

Open3D exposes convex hull but not VHACD directly; the standalone [V-HACD library](https://github.com/kmammou/v-hacd) provides a CPU implementation widely integrated into physics engines. GPU-accelerated variants Note: needs verification for open-source availability.

---

## 9. Shape Completion and Inpainting

Sensor noise, occlusion, and limited field of view routinely produce incomplete point cloud scans. Shape completion fills in the missing geometry using local context (PDE methods) or learned shape priors (neural networks).

### 9.1 Harmonic Extension (PDE Diffusion)

For a mesh with known vertices V_known and unknown vertices V_unknown, harmonic extension minimises the Dirichlet energy:

```
E[f] = Σ_{edges (i,j)} w_ij (f_i − f_j)²
```

subject to f_i = boundary values at known vertices. This yields the Laplacian system:

```
L_{UU} f_U = −L_{UK} f_K
```

where the subscripts U and K denote the unknown and known sub-blocks of the cotangent Laplacian. The right-hand side involves only known values; the system matrix L_{UU} is symmetric positive-definite (SPD) and sparse — solvable via preconditioned conjugate gradient on the GPU. [Source: Botsch et al., "Polygon Mesh Processing," AK Peters 2010]

For point cloud hole filling, the boundary of the hole is detected as vertices with fewer than the expected number of neighbours, a Delaunay triangulation is grown inward from the boundary, and harmonic extension assigns positions to the new interior vertices.

### 9.2 Variational Thin-Plate Spline Hole Filling

Thin-plate spline (TPS) minimisation produces a smoother fill than harmonic extension by penalising second-order curvature:

```
E_TPS[f] = ∫ ||∇²f||² dA
```

The discrete version assembles a biharmonic system L^T M^{-1} L (where L is the cotangent Laplacian and M is the mass matrix), which is a larger sparse system than the Laplacian alone. The GPU SpMV cost doubles relative to harmonic extension, but convergence quality is substantially better for large holes with strong curvature variation.

### 9.3 Learning-Based Completion: PCN and FoldingNet

The Point Completion Network (PCN) encodes a partial point cloud into a global feature vector, then decodes it in two stages: coarse completion (a small set of seed points) followed by fine completion (folding a 2D grid onto each seed point). [Source: Yuan et al., "PCN: Point Completion Network," 3DV 2018, https://doi.org/10.1109/3DV.2018.00088]

At inference time the entire pipeline runs as GPU compute. FoldingNet's decoder generates N×M output points by applying an MLP to the concatenation of the global code and a 2D grid coordinate, with the grid broadcast across all M columns using tensor broadcasting — a natural GPU operation:

```python
import torch
import torch.nn as nn

class FoldingDecoder(nn.Module):
    """FoldingNet decoder: fold a 2D plane onto 3D points."""
    def __init__(self, latent_dim: int, grid_size: int = 45):
        super().__init__()
        self.grid_size = grid_size
        self.fold1 = nn.Sequential(
            nn.Linear(latent_dim + 2, 512), nn.ReLU(),
            nn.Linear(512, 512), nn.ReLU(),
            nn.Linear(512, 3))
        self.fold2 = nn.Sequential(
            nn.Linear(latent_dim + 3, 512), nn.ReLU(),
            nn.Linear(512, 512), nn.ReLU(),
            nn.Linear(512, 3))

    def forward(self, code: torch.Tensor) -> torch.Tensor:
        """
        code: (B, latent_dim) global shape code
        returns: (B, grid_size^2, 3) completed point cloud
        """
        B = code.shape[0]
        g = self.grid_size
        # 2D grid: (g^2, 2)
        linspace = torch.linspace(-1.0, 1.0, g, device=code.device)
        grid = torch.stack(torch.meshgrid(linspace, linspace, indexing='ij'), dim=-1)
        grid = grid.reshape(-1, 2).unsqueeze(0).expand(B, -1, -1)  # (B, g^2, 2)
        # Expand code to each grid point
        code_exp = code.unsqueeze(1).expand(-1, g*g, -1)  # (B, g^2, latent_dim)
        x = torch.cat([code_exp, grid], dim=-1)  # (B, g^2, latent_dim+2)
        x = self.fold1(x)                         # (B, g^2, 3)  — first fold
        x = self.fold2(torch.cat([code_exp, x], dim=-1))  # (B, g^2, 3)
        return x
```

### 9.4 Implicit Completion: Occupancy Networks

Occupancy networks (and signed-distance-field networks) represent a shape as a continuous function f_θ: ℝ³ → [0, 1] trained to predict whether a query point is inside the shape. At inference time, the network is queried on a dense 3D grid (e.g., 128³ = 2M points), extracting the iso-surface at f = 0.5 via Marching Cubes. [Source: Mescheder et al., "Occupancy Networks: Learning 3D Reconstruction in Function Space," CVPR 2019, https://doi.org/10.1109/CVPR.2019.00459]

GPU inference over a 128³ grid requires 2M forward passes through an MLP. Using ONNX Runtime on ROCm or CUDA, the batch can be processed in tiles of 64K points per forward pass. The GPU occupancy for a typical 8-layer MLP with 256 hidden units is compute-bound; a 128³ grid completes in under 100 ms on a mid-range discrete GPU.

---

## 10. Shape Retrieval and Classification

Given a query shape, retrieval finds the most similar shapes in a database; classification assigns a category label. Both tasks reduce to approximate nearest-neighbour (ANN) search in a descriptor space.

### 10.1 The Retrieval Pipeline

```
Query shape
     │
     ▼
Voxel downsample + normal estimation
     │
     ▼
Descriptor extraction (FPFH / SH / PointNet embedding)
     │
     ▼
ANN search in GPU index (FAISS)
     │
     ▼
Re-rank top-K candidates (pairwise ICP score)
     │
     ▼
Return retrieved shapes
```

### 10.2 FAISS GPU Index

FAISS provides GPU-accelerated ANN search for float descriptors. [Source: FAISS GPU Documentation, https://faiss.ai/] The two most common GPU index types for shape retrieval are:

**GpuIndexFlatL2**: exact L2 nearest-neighbour search, O(N·d) per query, optimal for databases up to ~1M shapes. Stores all descriptors on-GPU.

**GpuIndexIVFFlat**: inverted-file index with Voronoi-based coarse quantisation. Search visits only `nprobe` Voronoi cells, reducing per-query work to O(N/n_cells · nprobe · d). Suitable for databases > 1M shapes with a small recall penalty.

```python
import faiss
import numpy as np

d = 33          # FPFH descriptor dimension
n_db = 100000   # database size

# Build database descriptors: (n_db, d) float32 numpy array
db_descriptors = np.random.rand(n_db, d).astype(np.float32)  # placeholder

# Move to GPU
res = faiss.StandardGpuResources()
index_flat = faiss.GpuIndexFlatL2(res, d)
index_flat.add(db_descriptors)

# Query: (n_q, d) float32
query = np.random.rand(10, d).astype(np.float32)  # 10 query shapes
k = 5
distances, indices = index_flat.search(query, k)
# indices: (10, 5) int64 — top-5 database matches per query
```
[Source: FAISS Python API, https://faiss.ai/]

### 10.3 Deep Shape Retrieval: PointNet and DGCNN

PointNet extracts a global shape feature by applying per-point MLPs followed by a max-pool aggregation, producing a single 1024-dimensional embedding. DGCNN (Dynamic Graph CNN) improves on PointNet by dynamically constructing a k-NN graph in feature space and applying edge convolutions. [Source: Wang et al., "Dynamic Graph CNN for Learning on Point Clouds," ACM Trans. Graphics 2019, https://doi.org/10.1145/3326362]

Using PyTorch Geometric:

```python
import torch
import torch.nn.functional as F
from torch_geometric.nn import DynamicEdgeConv, global_max_pool
from torch_geometric.nn import MLP

class DGCNN(torch.nn.Module):
    """DGCNN for 3D shape classification/retrieval."""
    def __init__(self, out_channels: int, k: int = 20):
        super().__init__()
        self.conv1 = DynamicEdgeConv(MLP([2 * 3,  64,  64,  64]), k=k, aggr='max')
        self.conv2 = DynamicEdgeConv(MLP([2 * 64, 128, 128]), k=k, aggr='max')
        self.conv3 = DynamicEdgeConv(MLP([2 * 128, 256]), k=k, aggr='max')
        self.mlp   = MLP([256 + 128 + 64, 512, 256, out_channels], dropout=0.5)

    def forward(self, data):
        pos, batch = data.pos, data.batch
        x1 = self.conv1(pos, batch)           # (N, 64)
        x2 = self.conv2(x1, batch)            # (N, 128)
        x3 = self.conv3(x2, batch)            # (N, 256)
        out = global_max_pool(
            torch.cat([x1, x2, x3], dim=1), batch)  # (B, 448)
        return self.mlp(out)                          # (B, out_channels)
```
[Source: PyTorch Geometric DynamicEdgeConv documentation, https://pytorch-geometric.readthedocs.io/en/latest/generated/torch_geometric.nn.conv.DynamicEdgeConv.html]

At retrieval time, the network embedding (output of the penultimate layer, before the final classification head) is used as the shape descriptor. Cosine similarity is efficient on GPU: for a query embedding q ∈ ℝ^d and a database matrix D ∈ ℝ^{N×d} (L2-normalised), cosine similarity is simply the matrix-vector product D·q, reducible to a cuBLAS `Sgemv` call.

### 10.4 Compact Binary Descriptors and Hamming Embedding

For very large databases (> 10M shapes), float descriptors are compressed to binary hash codes of length B bits (B = 64–256). Similarity search in Hamming space is GPU-accelerated via popcount operations: comparing one B-bit query to N database codes costs N · B/64 popcount instructions, achievable in a single GPU kernel with 100× throughput advantage over float L2 for the same descriptor dimensionality.

---

## 11. Geometric Deep Learning on Meshes

Convolutional neural networks achieve translation equivariance on images by exploiting the regular grid structure of pixels. 3D meshes lack a canonical grid, requiring network architectures that operate on graphs or manifolds directly.

### 11.1 Spectral Graph Convolution (ChebNet)

ChebNet defines convolution on a graph by polynomial filters in the Laplacian eigenspace. A filter of order K approximates the convolution via the Chebyshev recurrence:

```
y = Σ_{k=0}^{K} θ_k T_k(L_sym) x
```

where T_k is the k-th Chebyshev polynomial, L_sym is the symmetrically normalised Laplacian, and θ_k are learnable coefficients. [Source: Defferrard et al., "Convolutional Neural Networks on Graphs with Fast Localized Spectral Filtering," NeurIPS 2016, https://doi.org/10.5555/3157382.3157527]

The K SpMVs (L_sym applied to running recurrence state) are the compute-dominant operations. Using PyTorch Geometric:

```python
from torch_geometric.nn import ChebConv
import torch

# Graph with N=5000 vertices, edge_index (2, E), node features x (N, in_channels)
conv = ChebConv(in_channels=16, out_channels=64, K=3).cuda()
x = torch.randn(5000, 16, device='cuda')
edge_index = torch.randint(0, 5000, (2, 50000), device='cuda')
x_out = conv(x, edge_index)  # (5000, 64)
```
[Source: PyTorch Geometric ChebConv API, https://pytorch-geometric.readthedocs.io/en/latest/modules/nn.html]

### 11.2 Spatial Convolution: GraphSAGE

GraphSAGE (SAmple and aggreGatE) defines convolution spatially: at each node, aggregate features from a sampled neighbourhood, then apply a learnable aggregator (mean, LSTM, or max). [Source: Hamilton et al., "Inductive Representation Learning on Large Graphs," NeurIPS 2017, https://doi.org/10.5555/3294771.3294869]

```python
from torch_geometric.nn import SAGEConv

conv = SAGEConv(in_channels=16, out_channels=64, aggr='max').cuda()
x_out = conv(x, edge_index)  # same call pattern as ChebConv
```

GraphSAGE is inductive (generalises to unseen meshes at test time) and more GPU-efficient than ChebConv for large K: it requires only one SpMM (sparse matrix multiply for all neighbours simultaneously) rather than K SpMV passes.

### 11.3 MeshCNN: Edge-Based Convolution

MeshCNN defines convolution on mesh *edges* rather than vertices, using the four edges incident to each edge (forming a diamond-shaped patch) as the convolutional kernel. [Source: Hanocka et al., "MeshCNN: A Network with an Edge," SIGGRAPH 2019, https://doi.org/10.1145/3306346.3322959] Edge features capture local curvature, dihedral angles, and edge length; pooling collapses mesh edges via half-edge collapse, simultaneously reducing graph size and pooling features.

The edge-based approach preserves mesh connectivity explicitly, which benefits tasks like segmentation where sharp edges correspond to semantic part boundaries. The GPU implementation represents edge adjacency as a static index buffer (updated after each pool operation), enabling SpMM-based message passing on edges.

### 11.4 DiffusionNet: Heat-Kernel Learned Features

DiffusionNet replaces the fixed heat kernel with a learned spectral filter applied via the pre-computed LBO eigenbasis. [Source: Sharp et al., "DiffusionNet: Discretization Agnostic Learning on Surfaces," ACM Trans. Graphics 2022, https://doi.org/10.1145/3450626.3459900] At each layer, features are diffused in the LBO eigenspace:

```
f_out = Σ_k learnable_weight(λ_k) · (Φ · Φ^T · f_in)
```

where Φ is the N×k eigenfunction matrix. The projection Φ^T · f_in is a k×d matrix multiply (d = feature channels); the re-projection Φ · (·) is another N×k × k×d multiply. Both are dense GEMMs on the GPU.

DiffusionNet achieves isometry equivariance by design (the LBO is isometry-invariant), making it robust to non-rigid deformations without requiring data augmentation with random rotations.

```python
# DiffusionNet layer — illustrative forward pass
import torch

class DiffusionNetLayer(torch.nn.Module):
    def __init__(self, in_ch: int, out_ch: int, k: int):
        super().__init__()
        self.spectral_weights = torch.nn.Parameter(torch.randn(k))
        self.mlp = torch.nn.Linear(in_ch, out_ch)

    def forward(self,
                x: torch.Tensor,       # (N, in_ch) node features
                phi: torch.Tensor,     # (N, k) LBO eigenvectors
                ) -> torch.Tensor:
        # Project to spectral domain: (k, in_ch)
        x_spec = phi.T @ x
        # Apply per-eigenvalue learned weight
        x_spec = x_spec * self.spectral_weights.unsqueeze(1)
        # Project back to spatial domain: (N, in_ch)
        x_diff = phi @ x_spec
        # Per-vertex MLP
        return self.mlp(x_diff)
```
[Source: DiffusionNet — official implementation, https://github.com/nmwsharp/diffusion-net]

---

## 12. Production Libraries for Shape Analysis on Linux

### 12.1 Open3D

Open3D is the primary Python/C++ library for 3D data processing on Linux, with both CPU and GPU tensor backends. [Source: Open3D Documentation, https://www.open3d.org/docs/latest/]

**Relevant shape analysis functionality:**

| Feature | API | Notes |
|---|---|---|
| Normal estimation | `pcd.estimate_normals(KDTreeSearchParamHybrid(...))` | CPU via FLANN KD-tree |
| FPFH extraction | `o3d.pipelines.registration.compute_fpfh_feature(pcd, search_param)` | CPU; 33-dim |
| Global registration (RANSAC) | `registration_ransac_based_on_feature_matching(...)` | Uses FPFH correspondences |
| ICP (tensor API) | `o3dt.pipelines.registration.icp(...)` | GPU-resident via tensor backend |
| Plane segmentation | `pcd.segment_plane(distance_threshold, ransac_n, num_iterations)` | RANSAC, CPU |
| DBSCAN clustering | `pcd.cluster_dbscan(eps, min_points)` | CPU |
| Outlier removal | `pcd.remove_statistical_outlier(nb_neighbors, std_ratio)` | CPU |

[Source: Open3D Python API — geometry.PointCloud, https://www.open3d.org/docs/latest/python_api/open3d.geometry.PointCloud.html]

The tensor-based registration API (`o3d.t.pipelines.registration`) stores point clouds as GPU tensors and runs ICP kernel steps on the GPU, avoiding CPU-GPU round-trips between iterations. [Source: Open3D Tensor ICP, https://www.open3d.org/docs/latest/tutorial/t_pipelines/t_icp_registration.html]

### 12.2 PCL: Point Cloud Library

PCL provides C++ modules targeting CUDA GPUs. The key GPU modules for shape analysis are:

| Module | GPU functionality |
|---|---|
| `gpu_features` | GPU-accelerated normal estimation, FPFH, SHOT, NARF |
| `gpu_kdtree` | CUDA KD-tree for k-NN queries on large point clouds |
| `gpu_segmentation` | GPU RANSAC plane segmentation, Euclidean cluster extraction |
| `gpu_octree` | Octree-based approximate NN search on GPU |

[Source: PCL GPU modules overview, https://pcl.readthedocs.io/projects/tutorials/en/latest/]

```cpp
// PCL GPU normal estimation
#include <pcl/gpu/features/features.hpp>
#include <pcl/gpu/containers/device_array.h>

pcl::gpu::NormalEstimation ne;
pcl::gpu::DeviceArray<pcl::PointXYZ> cloud_gpu;
pcl::gpu::DeviceArray<pcl::Normal>   normals_gpu;

ne.setInputCloud(cloud_gpu);
ne.setRadiusSearch(0.03f, 30);   // radius = 3 cm, at most 30 neighbours
ne.compute(normals_gpu);
// normals_gpu: device-resident normal buffer, ready for further GPU processing
```
[Source: PCL GPU Features Tutorial, https://pcl.readthedocs.io/projects/tutorials/en/latest/gpu_normal_estimation.html]

Note: PCL's GPU modules target CUDA; ROCm/HIP ports are community-maintained and may not cover all GPU features.

### 12.3 CGAL Shape Detection

CGAL's Shape Detection package implements two complementary algorithms for fitting primitive shapes to point sets:

**Efficient RANSAC** fits planes, spheres, cylinders, cones, and tori to unorganised point clouds with normals. Key parameters:

```cpp
#include <CGAL/Point_set_3.h>
#include <CGAL/Shape_detection/Efficient_RANSAC.h>

typedef CGAL::Exact_predicates_inexact_constructions_kernel  Kernel;
typedef CGAL::Point_set_3<Kernel::Point_3,
                          Kernel::Vector_3>                  Point_set;
typedef CGAL::Shape_detection::Efficient_RANSAC_traits<
    Kernel, Point_set,
    Point_set::Point_map,
    Point_set::Vector_map>                                   Traits;
typedef CGAL::Shape_detection::Efficient_RANSAC<Traits>      Ransac;

Point_set points;
// ... populate points with positions and normals ...

Ransac ransac;
ransac.set_input(points);
ransac.add_shape_factory<CGAL::Shape_detection::Plane>();
ransac.add_shape_factory<CGAL::Shape_detection::Cylinder>();

Ransac::Parameters params;
params.probability      = 0.05;   // probability of missing best shape
params.min_points       = 200;    // minimum points per detected shape
params.epsilon          = 0.002;  // point-to-shape distance tolerance (m)
params.normal_threshold = 0.9;    // cos(angle) threshold for normal agreement
params.cluster_epsilon  = 0.01;   // connectivity spacing

ransac.detect(params);
for (const auto& shape : ransac.shapes())
    std::cout << shape->info() << std::endl;
```
[Source: CGAL Shape Detection Documentation, https://doc.cgal.org/latest/Shape_detection/index.html]

**Region Growing** grows planar regions from a seed point by iteratively adding neighbours that satisfy a distance and normal-consistency criterion, without randomisation. It is more deterministic and faster for well-organised data.

### 12.4 PyTorch3D

PyTorch3D provides differentiable 3D operations integrated with PyTorch's autograd, enabling gradient-based shape optimisation. [Source: PyTorch3D Documentation, https://pytorch3d.readthedocs.io/en/latest/]

Key operations relevant to shape analysis:

```python
from pytorch3d.ops import (
    knn_points,
    sample_points_from_meshes,
    estimate_pointcloud_normals,
    cot_laplacian,
    iterative_closest_point,
)
from pytorch3d.loss import chamfer_distance
from pytorch3d.structures import Meshes, Pointclouds

# Sample 5000 points from each mesh in a batch
meshes: Meshes = ...   # batch of B meshes on GPU
pcl = sample_points_from_meshes(meshes, num_samples=5000)  # (B, 5000, 3)

# Estimate normals
normals = estimate_pointcloud_normals(pcl, neighborhood_size=50)  # (B, 5000, 3)

# k-NN between two point clouds
src = torch.randn(1, 1000, 3, device='cuda')
tgt = torch.randn(1, 2000, 3, device='cuda')
knn = knn_points(src, tgt, K=5)
# knn.dists: (1, 1000, 5), knn.idx: (1, 1000, 5)

# Chamfer distance loss (differentiable)
loss, _ = chamfer_distance(src, tgt)

# Cotangent Laplacian for spectral analysis
L, inv_areas = cot_laplacian(meshes.verts_packed(), meshes.faces_packed())
```
[Source: PyTorch3D ops module documentation, https://pytorch3d.readthedocs.io/en/latest/modules/ops.html]

### 12.5 libigl

libigl is a C++ header-only mesh processing library with Python bindings (`igl` on PyPI). It provides the most complete CPU reference implementation of spectral shape analysis algorithms on Linux. [Source: libigl tutorial, https://libigl.github.io/tutorial/]

```python
import igl
import numpy as np

V, _, _, F, _, _ = igl.read_obj("mesh.obj")  # V: (N,3), F: (M,3)

# Cotangent Laplacian (sparse, negative semi-definite)
L = igl.cotmatrix(V, F)          # scipy.sparse.csc_matrix (N, N)

# Mass matrix (diagonal, Voronoi area weights)
M = igl.massmatrix(V, F, igl.MASSMATRIX_TYPE_VORONOI)

# k smallest eigenpairs of generalised problem L φ = λ M φ
import scipy.sparse.linalg as spla
k = 50
vals, vecs = spla.eigsh(-L, k=k, M=M, sigma=0.0, which='LM')
# vals: (k,) eigenvalues ≥ 0; vecs: (N, k) eigenvectors
```
[Source: libigl Python bindings, https://libigl.github.io/libigl-python-bindings/]

libigl does not provide GPU acceleration directly; however, the sparse matrices it produces (in scipy.sparse format) can be converted to GPU-resident tensors for use with `torch.lobpcg` or cuSPARSE, bridging libigl's CPU geometry processing with GPU spectral computation.

---

## Integrations

**Ch208 — GPU Geometry Algorithms.** Ch208 covers mesh construction, subdivision, and GPU buffer management. The shape analysis pipeline in this chapter begins where Ch208 ends: the output mesh or point cloud from Ch208's GPU geometry kernels is the input to the descriptor extraction stages of §2–§4.

**Ch209 — GPU Spatial, Differential, and Animation Algorithms.** Ch209's mean curvature flow implementation is the inner loop of the MCF skeleton contraction algorithm described in §7.1. The cotangent Laplacian and mass matrix assembled in Ch209's differential geometry section are identical to those used for HKS and WKS (§4.2–§4.3).

**Ch210 — GPU Physics and Volumetric Algorithms.** The ARAP non-rigid shape matching energy (§5.3) is the same energy used in Ch210's physically based deformation. The sparse linear solvers (cuSPARSE CG / rocSPARSE CG) used for the ARAP global step appear in Ch210's cloth and soft-body simulation.

**Ch211 — GPU Terrain, Ray Tracing, and Point Cloud.** Ch211 §90 covers ICP registration and point cloud processing in depth. This chapter's §5.1 builds on that foundation by discussing correspondence quality metrics and functional map extensions. The k-NN GPU queries in §2.1 (FPFH neighbourhood lookup) use the BVH structures described in Ch211.

**Ch212 — GPU Neural and Specialised Primitives.** Ch212 covers neural primitives including implicit surface networks. The occupancy network completion in §9.4 and the PointNet++ inference in §8.3 use GPU inference infrastructure described in Ch212.

**Ch222 — Computational Geometry Algorithms.** The CGAL Shape Detection RANSAC of §12.3 complements Ch222's discussion of convex hull and computational geometry. The ANN search infrastructure (§10.2, FAISS GPU) is the high-dimensional analogue of Ch222's spatial query methods.

**Ch25 — GPU Compute.** The GPU execution model, subgroup operations, and compute shader dispatch patterns referenced in §§2, 3, 6, and 7 of this chapter are introduced in depth in Ch25. Readers unfamiliar with GLSL compute shaders or CUDA thread hierarchy should read Ch25 before the implementation sections here.

**Ch20 — AI Inference (GPU).** The PointNet++, DGCNN, DiffusionNet, and PCN inference pipelines in §§8–11 use the same GPU inference stack (PyTorch CUDA kernels, ONNX Runtime on ROCm/CUDA) described in Ch20's coverage of on-device neural network inference.
