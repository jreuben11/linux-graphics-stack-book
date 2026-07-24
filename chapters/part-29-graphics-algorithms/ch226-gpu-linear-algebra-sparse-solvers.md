# Chapter 226: GPU Linear Algebra and Sparse Solvers

**Audiences:** Systems and driver developers optimising GPU workload throughput on ROCm/Vulkan Linux stacks; graphics application developers embedding physics simulation, geometry processing, or ML inference pipelines that feed vertex buffers and render passes.

**Relationship to the series.** Chapter 221 covers GPU algorithm performance fundamentals — occupancy, bandwidth ceilings, and kernel fusion. Chapter 210 builds physics simulations whose stiffness matrices are the typical input to the sparse solvers described here. Chapters 224 and 225 apply eigensolvers and sparse Laplacians to shape analysis and topology respectively, and can treat this chapter as the numerical engine underpinning those algorithms. Chapters 227 and 229 consume the dense GEMM throughput covered in §2–§3 for Monte Carlo sampling and ML inference respectively.

---

## Table of Contents

1. [Linear Algebra as a GPU Algorithm Domain](#1-linear-algebra-as-a-gpu-algorithm-domain)
2. [Dense BLAS Operations](#2-dense-blas-operations)
3. [Dense Matrix Factorizations](#3-dense-matrix-factorizations)
4. [Eigensolvers](#4-eigensolvers)
5. [Sparse Matrix Formats](#5-sparse-matrix-formats)
6. [SpMV and SpGEMM](#6-spmv-and-spgemm)
7. [Iterative Linear Solvers](#7-iterative-linear-solvers)
8. [Multigrid and AMG](#8-multigrid-and-amg)
9. [Structured, Toeplitz, and Circulant Systems](#9-structured-toeplitz-and-circulant-systems)
10. [Production Libraries](#10-production-libraries)
11. [Mixed Precision and Low-Precision Arithmetic](#11-mixed-precision-and-low-precision-arithmetic)
12. [Integration with the Graphics Pipeline](#12-integration-with-the-graphics-pipeline)
- [Integrations](#integrations)

---

## 1. Linear Algebra as a GPU Algorithm Domain

GPU hardware is an arithmetic throughput machine: thousands of shader processors fed by a high-bandwidth memory subsystem. Linear algebra maps naturally because its dominant kernels — matrix-matrix multiply, triangular solve, and sparse matrix-vector product — have regular, predictable memory access patterns that expose maximum parallelism.

### 1.1 Arithmetic Intensity and the Roofline

The roofline model (introduced by Williams et al., 2009) characterises a kernel by its **arithmetic intensity**: floating-point operations performed per byte of memory traffic. The hardware ridge point — the intensity at which the kernel transitions from memory-bound to compute-bound — divides the performance landscape.

For an m×n×k double-precision GEMM (C = αAB + βC):
- **Flops:** 2mnk multiply-adds
- **Memory:** (mn + mk + nk) × 8 bytes read/written — if each matrix is accessed once
- **Arithmetic intensity:** 2mnk / ((mn + mk + nk) × 8) → as m = n = k = n, intensity → 2n³ / (24n²) = **n/12** FLOP/byte

At n = 1024 on an AMD Instinct MI250X (HBM2e, 3.28 TB/s peak bandwidth, ~95.7 TFLOPS FP64 matrix), the ridge point is approximately 95.7e12 / 3.28e12 ≈ **29 FLOP/byte**. A 1024×1024 DGEMM achieves intensity ≈ 85 FLOP/byte — comfortably above the ridge, firmly compute-bound. This is why GEMM routinely achieves 80–90% of peak on well-tuned GPU implementations.

**SpMV** (sparse matrix-vector product, y = Ax) for a matrix with nnz non-zeros has intensity ≈ 2·nnz / (nnz × 12) ≈ 0.17 FLOP/byte for CSR with 32-bit indices — deeply memory-bound on all current GPUs. The challenge for sparse algorithms is not compute throughput but memory access efficiency: irregular column indices cause cache misses, and short rows leave warps underutilised.

[Source: "Roofline: An Insightful Visual Performance Model," Williams et al., CACM 2009]

### 1.2 Memory Hierarchy Impact

RDNA/CDNA GPUs present a multi-level hierarchy:
- **L0 (VGPR file):** 256–512 32-bit registers per lane; spilling to LDS is expensive
- **L1 (64 KB per CU):** shared across the 64-lane wavefront; coalesced access is critical
- **L2 (per shader array, 2–8 MB):** shared across CUs in one shader engine
- **Infinity Cache (RDNA3+):** up to 96 MB on-chip last-level cache; dramatically improves bandwidth for working sets that fit
- **HBM / GDDR6:** 900 GB/s–3.28 TB/s; capacity limit for large matrices

Dense GEMM tiling strategies deliberately stage sub-tiles through L1 and L2 to reuse data. Sparse algorithms — especially those with random column access — rarely achieve L1 reuse, making bandwidth the hard constraint.

---

## 2. Dense BLAS Operations

The BLAS standard (Basic Linear Algebra Subprograms) organises routines into three levels based on the ratio of computation to data. [Source: BLAS Technical Forum Standard, https://www.netlib.org/blas/]

### 2.1 Level 1, 2, and 3 Taxonomy

| Level | Operation | Ratio (flops/bytes) | Example |
|-------|-----------|---------------------|---------|
| 1 | vector-vector | O(1) | AXPY, DOT, NRM2 |
| 2 | matrix-vector | O(n) | GEMV, TRSV |
| 3 | matrix-matrix | O(n) | GEMM, TRSM, SYRK |

Level 1 and 2 operations are memory-bound on GPU; they are used as building blocks within iterative solvers but are not where GPU throughput is won. Level 3 operations — GEMM especially — are the workhorses of dense linear algebra.

### 2.2 rocBLAS Batched GEMM

rocBLAS [Source: ROCm Libraries, https://github.com/ROCm/rocm-libraries] provides two batched GEMM variants for processing independent same-size matrices in a single kernel launch.

**Pointer-array batched:** each matrix at an independent address — suitable when matrices are scattered in memory:

```c
rocblas_status rocblas_dgemm_batched(
    rocblas_handle     handle,
    rocblas_operation  transA,
    rocblas_operation  transB,
    rocblas_int        m,
    rocblas_int        n,
    rocblas_int        k,
    const double      *alpha,
    const double      *const A[],   // array of batch_count device pointers
    rocblas_int        lda,
    const double      *const B[],
    rocblas_int        ldb,
    const double      *beta,
    double            *const C[],
    rocblas_int        ldc,
    rocblas_int        batch_count
);
```

**Strided batched:** matrices stored contiguously with uniform stride — lower pointer-chasing overhead, preferred when all matrices are allocated together:

```c
rocblas_status rocblas_dgemm_strided_batched(
    rocblas_handle     handle,
    rocblas_operation  transA,
    rocblas_operation  transB,
    rocblas_int        m,   rocblas_int n,   rocblas_int k,
    const double      *alpha,
    const double      *A,   rocblas_int lda,   rocblas_stride stride_a,
    const double      *B,   rocblas_int ldb,   rocblas_stride stride_b,
    const double      *beta,
    double            *C,   rocblas_int ldc,   rocblas_stride stride_c,
    rocblas_int        batch_count
);
```

[Source: rocBLAS Level 3 API Reference, https://rocm.docs.amd.com/projects/rocBLAS/en/latest/reference/level-3.html]

**Grouped GEMM** — processing batches of *variable-size* matrices — is not available in core rocBLAS. It is provided by:
- MAGMA `magma_dgemm_vbatched` (per-matrix m/n/k arrays, GPU-side dispatch)
- oneMKL `oneapi::mkl::blas::column_major::gemm_batch` with `group_count` and per-group size arrays (SYCL backend, Intel GPU)
- hipBLASLt (AMD): supports grouped GEMM for variable-size batches via its matmul pipeline API [Note: verify exact function name against current hipBLASLt documentation]

### 2.3 TRSM and SYRK

Triangular solve with multiple right-hand sides (TRSM) is the other critical Level 3 routine, appearing in every factorisation update:

```c
rocblas_status rocblas_dtrsm(
    rocblas_handle  handle,
    rocblas_side    side,    // left or right
    rocblas_fill    uplo,    // upper or lower triangle
    rocblas_operation transA,
    rocblas_diagonal diag,   // unit or non-unit diagonal
    rocblas_int     m,  rocblas_int n,
    const double   *alpha,
    const double   *A,  rocblas_int lda,
    double         *B,  rocblas_int ldb
);
```

Symmetric rank-k update (SYRK, C = αAAᵀ + βC) appears in Cholesky and least-squares computations. rocBLAS provides `rocblas_dsyrk`. [Source: rocBLAS Level 3, https://rocm.docs.amd.com/projects/rocBLAS/en/latest/reference/level-3.html]

### 2.4 Packing, Tiling, and Register Blocking

Achieving >80% peak GEMM throughput requires explicit data staging. The standard approach (used in Tensile, the code-generation backend for rocBLAS):

1. **Panel packing:** copy an m×k panel of A into a contiguous LDS-aligned buffer, transposing to maximise bank access. Similarly pack k×n of B.
2. **Thread block tiles:** each workgroup computes a 64×64 or 128×64 output tile of C.
3. **Register blocking:** each thread accumulates a 4×4 or 8×4 register tile to maximise FMA instruction-level parallelism while hiding HBM latency.
4. **Double-buffering:** overlap the next LDS load with the current VALU (MFMA) computation.

On CDNA2 (MI200), the MFMA (Matrix Fused Multiply-Add) instruction operates on 16×16 or 32×32 matrix tiles directly in registers, replacing the scalar FMA loop. MFMA throughput is the ceiling that well-tuned GEMM approaches.

---

## 3. Dense Matrix Factorizations

rocSOLVER provides LAPACK-compatible factorizations running on ROCm GPUs. [Source: rocSOLVER documentation, https://rocm.docs.amd.com/projects/rocSOLVER/en/latest/]

### 3.1 LU Factorization with Partial Pivoting

The panel-update algorithm factorises a block column (panel) at a time, then applies a TRSM and trailing GEMM update:

```c
rocblas_status rocsolver_dgetrf(
    rocblas_handle handle,
    rocblas_int    m,
    rocblas_int    n,
    double        *A,      // m×n matrix, overwritten by LU factors
    rocblas_int    lda,
    rocblas_int   *ipiv,   // pivot indices, length min(m,n)
    rocblas_int   *info    // 0 = success, >0 = singular pivot column
);
```

The panel factorisation (a tall-skinny column block) is compute-limited on GPU because it is BLAS-2 in nature. rocSOLVER handles this by recursively blocking into smaller panels and calling `rocsolver_dgetf2` (unblocked, BLAS-2) on the CPU-accessible portion while launching the trailing GEMM update asynchronously. For large matrices (n > 2048) the trailing GEMM dominates and GPU efficiency is high.

### 3.2 QR Factorization

```c
rocblas_status rocsolver_dgeqrf(
    rocblas_handle handle,
    rocblas_int    m,
    rocblas_int    n,
    double        *A,    // m×n matrix, overwritten by Q (Householder reflectors) and R
    rocblas_int    lda,
    double        *tau   // length min(m,n), scalar factors of Householder reflectors
);
```

**Tall-Skinny QR (TSQR):** when m ≫ n (tall matrix), the standard blocked Householder approach leaves the trailing GEMM too narrow to be efficient. TSQR splits the matrix into horizontal tiles, factorises each tile independently (embarrassingly parallel), then hierarchically merges the R factors. This maps well to GPU with one workgroup per tile. MAGMA implements TSQR as `magma_dtsqrf_gpu`. [Source: MAGMA 2.x documentation, https://icl.utk.edu/magma/]

### 3.3 Cholesky Factorization

```c
rocblas_status rocsolver_dpotrf(
    rocblas_handle handle,
    rocblas_fill   uplo,   // ROCBLAS_FILL_MODE_UPPER or LOWER
    rocblas_int    n,
    double        *A,      // n×n SPD matrix, overwritten by triangular factor
    rocblas_int    lda,
    rocblas_int   *info    // 0 = success, >0 = non-positive definite
);
```

The right-looking (trailing-matrix) algorithm is preferred on GPU: after factorising column j, update the trailing (n−j)×(n−j) submatrix with a rank-1 SYRK update. This keeps the GEMM-like update in the GPU-efficient path. For batched small Cholesky (physics simulation — per-element SPD systems), `magma_dpotrf_batched` dispatches all at once.

### 3.4 SVD and One-Sided Jacobi

The divide-and-conquer SVD (`rocsolver_dgesvd`) is the workhorse for large matrices (note: verify exact function signature against rocSOLVER release notes; the function exists in rocSOLVER ≥ 3.0 but parameter names for left/right singular vector computation modes should be confirmed against current headers).

For **batched small SVD** (e.g., per-triangle deformation gradient in FEM), the one-sided Jacobi method is preferred: it applies a sequence of 2×2 Givens rotations to annihilate off-diagonal elements, with each rotation sweep operating on independent column pairs. MAGMA exposes this as `magma_dgesvj_batched`, parallelising across the batch dimension. The per-matrix cost is O(n³) but the constant is low for small n (n ≤ 32) and the batch dimension exploits warp-level parallelism.

---

## 4. Eigensolvers

### 4.1 Symmetric Eigenvalue Problem

For a real symmetric matrix A, the eigenvalue problem Ax = λx is relevant in shape analysis (Ch224), topology (Ch225), and physics (modal analysis). rocSOLVER provides `rocsolver_dsyevd` for divide-and-conquer symmetric eigensolver and `rocsolver_dsyevj` for Jacobi-based batched variants (note: verify availability in the current rocSOLVER version against https://rocm.docs.amd.com/projects/rocSOLVER/en/latest/). Both operate on device memory with `rocblas_handle`.

### 4.2 Krylov Subspace Methods — Lanczos and Arnoldi

When only a few extreme eigenvalues of a large sparse matrix are needed, the Lanczos algorithm (symmetric) or Arnoldi algorithm (general) builds a Krylov subspace K_m = span{v, Av, A²v, …, A^{m−1}v}. The projection of A onto K_m is a tridiagonal (Lanczos) or upper Hessenberg (Arnoldi) matrix whose eigenvalues approximate those of A. Convergence to extremal eigenvalues is rapid.

GPU implementation:
1. Each Lanczos step requires one SpMV (`rocsparse_spmv`) and one AXPY + inner products (`rocblas_daxpy`, `rocblas_ddot`)
2. The Krylov basis vectors (m vectors of length n) are stored in an n×m device matrix
3. Reorthogonalisation (full or selective) stabilises the iteration; uses a Level 2 GEMV

The thick-restart Lanczos (TRLAN) algorithm [Source: Wu and Simon, https://doi.org/10.1137/S1064827500374093] periodically compresses the Krylov basis by retaining only the k best Ritz pairs ("locking") and restarting from the deflated subspace. This bounds memory at k + p vectors (k locked, p active) while allowing effective basis dimension larger than memory permits.

### 4.3 LOBPCG for Symmetric Problems

Locally Optimal Block Preconditioned Conjugate Gradient (LOBPCG) [Source: Knyazev 2001, https://doi.org/10.1137/S1064827500366124] simultaneously optimises a block of trial eigenvectors, making it GPU-friendly because the inner loop is a batched dense GEMM rather than a sequence of SpMV calls on single vectors. rocALUTION includes a LOBPCG implementation. The preconditioned variant applies an approximate inverse (Jacobi, polynomial Chebyshev) as a preconditioner within each block step, dramatically accelerating convergence for ill-conditioned problems.

---

## 5. Sparse Matrix Formats

Format choice is the first performance decision for any sparse computation. The key trade-off axis is regularity: formats that assume uniform row length are efficient when that assumption holds and wasteful when it does not.

### 5.1 Format Survey

| Format | Key fields | Best for |
|--------|-----------|----------|
| **CSR** | `row_ptr[m+1]`, `col_ind[nnz]`, `val[nnz]` | General SpMV; row-major access |
| **CSC** | `col_ptr[n+1]`, `row_ind[nnz]`, `val[nnz]` | Column-major access; SpSolve |
| **COO** | `row_ind[nnz]`, `col_ind[nnz]`, `val[nnz]` | Construction; format conversion |
| **ELL/ELLPACK** | `col_ind[m×w]`, `val[m×w]` (w = max row NNZ) | Uniform sparsity (FEM stencils) |
| **HYB** | ELL (regular rows) + COO (overflow rows) | Mixed sparsity |
| **BSR** | `row_ptr`, `col_ind`, `val[nnz_blocks × rb × cb]` | Block-structured (FEM assemblies) |

**Format selection heuristics:**
- **High row-length variance** (power-law graphs, adaptive meshes): HYB or merge-based CSR SpMV
- **Near-uniform row length** (regular stencils, structured grids): ELL
- **Dense local blocks** (velocity-pressure FEM, material point method): BSR with block size matching the element DOF count
- **Dynamic construction followed by solve**: build in COO, convert to CSR/BSR with `rocsparse_coo2csr`

### 5.2 Creating rocSPARSE Descriptors

The modern rocSPARSE generic API creates format-agnostic handles that are passed to unified SpMV and SpGEMM kernels:

```c
// Create a CSR matrix descriptor on the GPU
rocsparse_spmat_descr A_descr;
rocsparse_status status = rocsparse_create_csr_descr(
    &A_descr,
    (int64_t)m,            // rows
    (int64_t)n,            // cols
    (int64_t)nnz,          // non-zeros
    (void *)csr_row_ptr,   // device pointer, int32 or int64
    (void *)csr_col_ind,   // device pointer
    (void *)csr_val,       // device pointer, float or double
    rocsparse_indextype_i32,
    rocsparse_indextype_i32,
    rocsparse_index_base_zero,
    rocsparse_datatype_f64_r
);
```

A dense vector descriptor is created analogously with `rocsparse_create_dnvec_descr`. Once created, descriptors survive across multiple SpMV calls, amortising setup cost. [Source: rocSPARSE Generic API, https://rocm.docs.amd.com/projects/rocSPARSE/en/latest/reference/generic.html]

---

## 6. SpMV and SpGEMM

### 6.1 Load-Balancing Strategies for CSR SpMV

CSR SpMV assigns row i to a thread or warp. Four progressively more sophisticated schemes:

| Strategy | Assignment | Problem |
|----------|-----------|---------|
| Scalar | 1 thread per row | Long rows saturate one thread; short rows waste warps |
| Vector | 1 warp per row | Short rows idle most lanes; very long rows still serialised |
| Warp | 1 block per row | Viable for matrices with uniformly long rows |
| Merge-based | 1 thread segment of the CSR (row_ptr, val) merge path | Provably load-balanced regardless of row-length distribution |

The **merge-based SpMV** (Merrill and Garland, 2016) [Source: https://doi.org/10.1145/2967938.2967940] views the SpMV as a merge of two monotone sequences — the row pointer array and the value array — and partitions this merge path evenly across threads. Each thread then independently processes its contiguous segment, with a prefix-sum across the partial row sums at segment boundaries. This achieves near-perfect load balance for arbitrary sparsity patterns.

**CSR Adaptive** (Bell and Garland; also the basis of rocSPARSE's adaptive SpMV) dynamically selects scalar, vector, or warp strategy per row-block based on a prefix-sum histogram of row lengths computed at first call.

### 6.2 rocsparse_spmv — Generic API

```c
rocsparse_status rocsparse_spmv(
    rocsparse_handle            handle,
    rocsparse_operation         trans,        // transpose or not
    const void                 *alpha,        // device or host scalar
    rocsparse_const_spmat_descr mat,          // sparse matrix descriptor
    rocsparse_const_dnvec_descr x,            // input dense vector
    const void                 *beta,
    const rocsparse_dnvec_descr y,            // output dense vector
    rocsparse_datatype          compute_type, // rocsparse_datatype_f64_r
    rocsparse_spmv_alg          alg,          // e.g. ROCSPARSE_SPMV_ALG_CSR_ADAPTIVE
    rocsparse_spmv_stage        stage,        // BUFFER_SIZE → PREPROCESS → COMPUTE
    size_t                     *buffer_size,  // temp buffer query and use
    void                       *temp_buffer
);
```

The three-stage pattern (buffer size query, optional pre-processing, compute) amortises symbolic analysis cost over multiple solves with the same sparsity pattern:

```c
// Stage 1: query buffer size
rocsparse_spmv(handle, trans, &alpha, mat, x, &beta, y,
               rocsparse_datatype_f64_r,
               ROCSPARSE_SPMV_ALG_CSR_ADAPTIVE,
               rocsparse_spmv_stage_buffer_size,
               &buf_size, NULL);
hipMalloc(&temp_buf, buf_size);

// Stage 2: preprocess (sparsity analysis, permutation)
rocsparse_spmv(handle, trans, &alpha, mat, x, &beta, y,
               rocsparse_datatype_f64_r,
               ROCSPARSE_SPMV_ALG_CSR_ADAPTIVE,
               rocsparse_spmv_stage_preprocess,
               &buf_size, temp_buf);

// Stage 3: compute (called in the solver loop)
rocsparse_spmv(handle, trans, &alpha, mat, x, &beta, y,
               rocsparse_datatype_f64_r,
               ROCSPARSE_SPMV_ALG_CSR_ADAPTIVE,
               rocsparse_spmv_stage_compute,
               &buf_size, temp_buf);
```

[Source: rocSPARSE API Reference, https://rocm.docs.amd.com/projects/rocSPARSE/en/latest/reference/generic.html]

### 6.3 SpGEMM: Symbolic and Numeric Phases

Sparse matrix-matrix multiply (C = A × B) requires computing both the sparsity pattern of C (symbolic phase) and its values (numeric phase). The symbolic phase determines which entries of C are non-zero — this requires scanning all pairs (A[i,k], B[k,j]) for each row i. The challenge is that the output size is unknown a priori.

Three approaches to the symbolic phase:
1. **Hash-map per row:** each thread maintains a small hash table of (column, value) pairs for one output row. Efficient for short rows; suffers hash collisions on wide rows.
2. **Heap-based merge:** maintains a min-heap over active row iterators of B, merging in sorted order. Memory-efficient but irregular access.
3. **Expansion-Sort-Compress (ESC):** emit all (i, j, a_ik × b_kj) triples, sort by (i, j), then compress equal (i, j) pairs. Highly parallelisable; memory cost is O(nnz(A) × max_row_nnz(B)).

rocSPARSE provides `rocsparse_spgemm` with symbolic and numeric stages, supporting CSR inputs and outputs with an intermediate work buffer for the symbolic result. [Note: The exact multi-parameter signature of `rocsparse_spgemm` — including `rocsparse_spgemm_alg` and `rocsparse_spgemm_stage` — should be verified against the current rocSPARSE headers at https://github.com/ROCm/rocm-libraries before production use.]

---

## 7. Iterative Linear Solvers

For large sparse systems Ax = b where direct factorisation is too expensive, iterative methods produce successive approximations converging to the solution.

### 7.1 Conjugate Gradient

CG requires A to be symmetric positive definite (SPD). Each iteration:

```
r = b - A·x         (SpMV + AXPY)
p = r + β·p         (AXPY)
α = (rᵀr) / (pᵀAp) (two dots and one SpMV)
x = x + α·p         (AXPY)
r = r - α·Ap        (AXPY)
```

**GPU-resident implementation:** the critical design decision is to keep all vectors (x, r, p, Ap) on the GPU and copy only the two scalar numerics (rᵀr, pᵀAp) to the host for the division. rocBLAS `rocblas_ddot` leaves the result in a device scalar by default (set pointer mode with `rocblas_set_pointer_mode(handle, rocblas_pointer_mode_device)`), allowing the α computation to remain on the GPU:

```c
// PCIe-free α computation via device-side scalars
rocblas_set_pointer_mode(blas_handle, rocblas_pointer_mode_device);
rocblas_ddot(blas_handle, n, r, 1, r, 1, &rtr_dev);  // result in device memory
// α = rtr_old / pAp — computed in a tiny kernel, never touches host
```

Convergence is checked against a relative residual threshold ‖r_k‖₂ / ‖b‖₂ < ε, with ‖r_k‖₂ computed via `rocblas_dnrm2`. To avoid a PCIe round-trip each iteration, the check is performed only every K iterations (K = 50 is common), trading convergence detection latency for bandwidth.

**Preconditioned CG** applies a preconditioner M ≈ A⁻¹ at each step: z = M⁻¹r. Common choices on GPU:
- **Jacobi (diagonal):** z_i = r_i / A_ii — trivial, one EDIV kernel
- **ILU(0):** incomplete LU with no fill; rocSPARSE provides `rocsparse_csrilu0` and the subsequent triangular solves via `rocsparse_spsv`
- **AMG:** see §8; one AMG V-cycle as preconditioner, applied inside each CG step

### 7.2 BiCGSTAB

BiCGSTAB (van der Vorst, 1992) handles non-symmetric systems. Unlike the earlier BiCG algorithm, BiCGSTAB is **transpose-free**: it applies A twice per iteration and never requires a multiply with Aᵀ. A fixed shadow residual vector r̂₀ (chosen at startup) enters only through inner products, which are cheap device-scalar dot products. The two SpMVs with A per iteration mean BiCGSTAB costs roughly twice CG per step but handles the non-symmetric case. GPU implementation follows the same device-scalar pointer mode as CG, with two `rocblas_daxpy` calls and two `rocsparse_spmv` calls per iteration.

### 7.3 GMRES with Restarts

Generalised Minimal Residual (GMRES) is effective for highly non-symmetric or indefinite systems. GMRES(m) builds a Krylov basis of m vectors, solves a small m×m least-squares problem (QR on the upper Hessenberg matrix), updates x, then restarts. GPU considerations:
- The Arnoldi orthogonalisation loop is a sequence of Level 1 dot products and AXPYs — low arithmetic intensity but unavoidable
- The m×m Hessenberg solve is so small it runs on CPU without significant overhead
- Restart basis storage: m vectors of length n; for n = 10⁶ and m = 100, this is 800 MB of device memory

rocALUTION implements GMRES(m) as part of its Krylov solver suite, using HIP-accelerated SpMV and BLAS primitives. [Source: rocALUTION documentation, https://rocm.docs.amd.com/projects/rocALUTION/en/latest/]

---

## 8. Multigrid and AMG

Iterative solvers like CG converge slowly when A is ill-conditioned — when the condition number κ(A) = λ_max/λ_min is large. Multigrid methods achieve condition-number-independent convergence by combining a smoother (that rapidly damps high-frequency error) with a coarse-grid correction (that reduces low-frequency error).

### 8.1 Geometric Multigrid Operators

On a structured grid, the restriction operator (fine → coarse) is a weighted averaging stencil — equivalent to a sparse matrix-vector product with a fixed, regular pattern. The prolongation operator (coarse → fine) is bilinear (2D) or trilinear (3D) interpolation, again a sparse matvec. Both are highly efficient on GPU using `rocsparse_spmv` with pre-built CSR restriction and prolongation matrices. For a 3-level V-cycle:

```
level 0 (fine):   pre-smooth (2 Jacobi/CG steps)
                  compute residual r = b - A·x        [SpMV]
                  restrict r to level 1:  r1 = R·r    [SpMV]
level 1:          pre-smooth, restrict → r2 = R1·r1   [SpMV]
level 2 (coarse): solve A2·e2 = r2 exactly (small, on GPU or CPU)
level 1:          prolong e1 = P1·e2                   [SpMV]
                  post-smooth
level 0:          prolong e0 = P0·e1                   [SpMV]
                  update: x += e0
                  post-smooth
```

### 8.2 Algebraic Multigrid (AMG)

AMG constructs the coarse-grid hierarchy automatically from the matrix coefficients, without requiring geometric information. This makes it applicable to unstructured meshes and arbitrary PDE discretisations.

Two families dominate:
- **Classical (Ruge-Stüben) AMG:** defines "strong connections" (|A_ij| > θ · max_k|A_ik|) and constructs coarse sets by a colouring-based algorithm. Prolongation weights are derived from the smoothing property. Produces accurate but dense coarse grids.
- **Aggregation-based AMG (smoothed aggregation):** clusters strongly connected nodes into aggregates; constructs a piecewise-constant tentative prolongator; smoothes it with a damped Jacobi step. Coarser hierarchy, lower setup cost, slightly less accurate.

**Smoothers on GPU:**
- *Damped Jacobi:* x ← x + ω · D⁻¹ · r — diagonal scaling plus SpMV; trivially parallel
- *Gauss-Seidel:* inherently sequential in serial; on GPU requires graph colouring. Colour the graph of A such that no two adjacent nodes share a colour, then update all nodes of one colour simultaneously. rocSPARSE `rocsparse_csrcolor` computes a distance-k colouring.
- *Chebyshev polynomial smoother:* applies a degree-k Chebyshev polynomial in A to the residual, computed as a sequence of SpMVs and AXPYs. GPU-friendly because it is entirely BLAS-like.

### 8.3 rocALUTION AMG

rocALUTION's AMG is configured through its C++ object hierarchy:

```cpp
#include <rocalution/rocalution.hpp>
using namespace rocalution;

LocalMatrix<double> A;
LocalVector<double> x, b;
A.ReadFileMTX("matrix.mtx");   // Matrix Market format

AMGCG<LocalMatrix<double>, LocalVector<double>, double> amg_solver;
amg_solver.SetOperator(A);
amg_solver.SetCoarseningStrategy(Aggregation);  // or PMIS
amg_solver.SetSmootherPreIter(1);
amg_solver.SetSmootherPostIter(1);
amg_solver.Build();   // constructs AMG hierarchy on GPU

amg_solver.Solve(b, &x);
```

The `MoveToAccelerator()` call on `A`, `x`, `b` moves data to the HIP device. All subsequent operations — coarsening, smoothing, prolongation — run on the GPU. [Source: rocALUTION 3.x documentation, https://rocm.docs.amd.com/projects/rocALUTION/en/latest/]

AMGX (NVIDIA, reference) provides equivalent functionality on CUDA with classical and aggregation AMG, Krylov smoothers, and multi-GPU support via MPI. [Source: AMGX GitHub, https://github.com/NVIDIA/AMGX]

---

## 9. Structured, Toeplitz, and Circulant Systems

When the matrix A has special structure — its entries depend only on the index difference A_ij = f(i−j) — a direct sparse representation wastes structure and masks algorithmic opportunity.

### 9.1 Toeplitz Matvec via FFT

A Toeplitz matrix T of size n is fully determined by its first row and first column (2n−1 values). The matrix-vector product y = Tx is equivalent to a 1D convolution: y_i = Σ_j T_{i−j} x_j. The standard fast algorithm:

1. **Embed** T in a 2n×2n circulant matrix C (Toeplitz is a submatrix of a circulant)
2. **Pad** x to length 2n
3. **Forward FFT** of both the first row of C (the kernel) and the padded x
4. **Pointwise multiply** of the two frequency-domain vectors
5. **Inverse FFT** to recover the convolution; extract the first n entries

Cost: O(n log n) vs O(n²) for direct computation. On GPU, all three FFT steps run as a single `VkFFTAppend` call appended to a Vulkan command buffer. VkFFT [Source: https://github.com/DTolm/VkFFT] supports six backends (Vulkan=0, CUDA=1, HIP=2, OpenCL=3, Level Zero=4, Metal=5) and is configured by setting `FFTdim`, `size[3]`, and the backend-specific `device`/`queue`/`buffer` fields in a `VkFFTConfiguration` struct before calling `initializeVkFFT` and `VkFFTAppend`.

### 9.2 Circulant Preconditioner

For a general non-Toeplitz system, the best circulant approximation (minimising the Frobenius norm of the difference) is constructed by averaging the diagonals of A. The resulting circulant system is solved in O(n log n) via FFT and used as a preconditioner for CG or GMRES. Circulant preconditioning is particularly effective for convection-diffusion PDEs on periodic domains and image deblurring with a spatially uniform PSF.

### 9.3 Applications

| Application | Linear system | Structure |
|-------------|--------------|-----------|
| Image deblurring (uniform PSF) | A = convolution with PSF | Toeplitz (2D block Toeplitz) |
| Heat equation on regular grid | A = finite difference Laplacian | Toeplitz in each spatial direction |
| Audio processing | A = room impulse response convolution | Toeplitz |
| PDE on periodic domain | A = circulant | Circulant (diagonalised by DFT) |

---

## 10. Production Libraries

### 10.1 rocBLAS and hipBLAS

rocBLAS [Source: https://github.com/ROCm/rocm-libraries] implements BLAS 1/2/3 for AMD GPUs via the ROCm/HIP stack. Key characteristics:
- Tensile code-generation backend: generates ISA-level GEMM kernels tuned per (m, n, k, transA, transB, datatype) tuple at library build time
- `rocblas_handle` carries the HIP stream; operations are async relative to the host
- `hipBLAS` is the compatibility layer: `hipblasDgemm` dispatches to `rocblas_dgemm` on AMD, to cuBLAS on NVIDIA — enabling portable code

### 10.2 rocSOLVER

rocSOLVER [Source: https://rocm.docs.amd.com/projects/rocSOLVER/en/latest/] provides LAPACK-compatible dense factorizations reusing the same `rocblas_handle`. All device pointers are passed directly, eliminating PCIe copies. Verified functions include `rocsolver_dgetrf` (LU+pivot), `rocsolver_dpotrf` (Cholesky), `rocsolver_dgeqrf` (QR), and symmetric eigensolvers. rocSOLVER uses the same handle as rocBLAS, so both can share one stream without extra synchronisation.

### 10.3 rocSPARSE

rocSPARSE [Source: https://rocm.docs.amd.com/projects/rocSPARSE/en/latest/] provides:
- **Format conversion:** `rocsparse_csr2coo`, `rocsparse_coo2csr`, `rocsparse_csr2bsr`
- **SpMV (generic API):** `rocsparse_spmv` with CSR, COO, BSR, ELL; adaptive algorithm
- **SpGEMM:** `rocsparse_spgemm` with symbolic/numeric stages (see §6.3)
- **SpSV:** triangular solve `rocsparse_spsv` — used after ILU(0) factorisation
- **ILU(0):** `rocsparse_csrilu0` with `rocsparse_csrilu0_zero_pivot` check
- **Incomplete Cholesky:** `rocsparse_csric0`

The deprecated typed API (`rocsparse_dcsrmv`, `rocsparse_dcsrgemm`) remains for legacy code but the generic descriptor API is preferred for new development.

### 10.4 Intel oneMKL (SYCL Backend)

Intel oneMKL on Linux supports Intel Arc, Ponte Vecchio, and Battlemage GPUs via its SYCL backend. The `oneapi::mkl::blas::column_major::gemm_batch` function provides grouped GEMM with a `group_count` parameter, enabling variable-size matrix groups in a single launch — the feature absent from core rocBLAS. [Source: Intel oneAPI Math Kernel Library, https://www.intel.com/content/www/us/en/developer/tools/oneapi/onemkl.html]

### 10.5 MAGMA

MAGMA (Matrix Algebra on GPU and Multi-core Architectures) [Source: https://icl.utk.edu/magma/] provides:
- **Multi-GPU factorizations:** `magma_dgetrf_mgpu`, `magma_dpotrf_mgpu` — panel factorisation on one GPU, TRSM and GEMM updates distributed across all GPUs
- **Mixed-precision:** FP32 factorisation with FP64 iterative refinement (`magma_dsgesv` for double-single)
- **Variable-size batched:** `magma_dgemm_vbatched`, `magma_dgetrf_vbatched` — per-matrix size arrays, GPU-side dispatch
- **Batched Jacobi SVD:** `magma_dgesvj_batched` — essential for per-element deformation gradient in FEM

MAGMA supports CUDA and HIP backends, making it portable across NVIDIA and AMD hardware.

### 10.6 VkFFT

VkFFT [Source: https://github.com/DTolm/VkFFT] is a header-only GPU FFT library (MIT licence) supporting Vulkan, CUDA, HIP, OpenCL, Level Zero, and Metal. Its key advantage for graphics-pipeline integration is native Vulkan support: the FFT is appended to an existing `VkCommandBuffer`, sharing synchronisation with render passes and other compute dispatches without backend API crossing. Benchmarks show VkFFT matching or exceeding cuFFT performance on NVIDIA hardware while supporting AMD and Intel GPUs natively.

---

## 11. Mixed Precision and Low-Precision Arithmetic

### 11.1 Iterative Refinement

The standard FP64→FP32→FP64 iterative refinement scheme reduces FP64 factorisation cost (the dominant expense) by performing the factorisation in FP32 and correcting the error with FP64 residual iterations:

```
[FP32] factor A = LU                   (rocSOLVER in float)
[FP32] solve LU·x₀ = b
for k = 0, 1, 2, …:
    [FP64] r_k = b - A·x_k            (FP64 SpMV or GEMM)
    [FP32] solve LU·d = r_k            (triangular solve in float)
    [FP64] x_{k+1} = x_k + d
    if ‖r_k‖/‖b‖ < ε: break
```

MAGMA `magma_dsgesv` implements this with typically 2–3 refinement steps needed, yielding FP64 accuracy at roughly half the factorisation cost of a full FP64 LU. [Source: MAGMA documentation, https://icl.utk.edu/magma/]

### 11.2 BF16 and FP16 Accumulation

For GEMM blocks where full FP64 accuracy is unnecessary (graph Laplacian eigenvalue preconditioning, neural-network weight matrices), BF16 (bfloat16) offers the same 8-bit exponent range as FP32 with only 7-bit mantissa. This halves register pressure and doubles the effective MFMA throughput on CDNA2/CDNA3 hardware. rocBLAS `rocblas_gemm_ex` accepts mixed input/compute/output types, allowing A and B in BF16 with accumulation in FP32.

INT8 arithmetic is applicable to graph Laplacian spectral problems where all matrix entries are integers (the Laplacian of an unweighted graph has integer entries bounded by the maximum degree). The product of two INT8 matrices is accumulated in INT32. rocBLAS `rocblas_gemm_ex` supports INT8 inputs with INT32 accumulation.

### 11.3 VK_KHR_cooperative_matrix

`VK_KHR_cooperative_matrix` [Source: Vulkan Extension Registry, https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#VK_KHR_cooperative_matrix] exposes the hardware matrix tile instructions (WMMA, MFMA, dpas) through a GLSL/SPIR-V intrinsic available in compute shaders. Unlike the vendor extensions (`VK_NV_cooperative_matrix`, `VK_INTEL_shader_integer_functions2`), the KHR form is cross-vendor and is the long-term standard.

The query API enumerates supported tile configurations at runtime:

```c
// Query supported cooperative matrix tile configurations
uint32_t property_count = 0;
vkGetPhysicalDeviceCooperativeMatrixPropertiesKHR(
    physicalDevice, &property_count, NULL);

VkCooperativeMatrixPropertiesKHR *props =
    malloc(property_count * sizeof(*props));
for (uint32_t i = 0; i < property_count; i++)
    props[i].sType = VK_STRUCTURE_TYPE_COOPERATIVE_MATRIX_PROPERTIES_KHR;
vkGetPhysicalDeviceCooperativeMatrixPropertiesKHR(
    physicalDevice, &property_count, props);
// Each entry specifies MSize, NSize, KSize, AType, BType, CType,
// ResultType, and scope (VK_SCOPE_SUBGROUP_KHR)
```

In GLSL (with `#extension GL_KHR_cooperative_matrix : enable`):

```glsl
coopmat<float16_t, gl_ScopeSubgroup, 16, 16, gl_MatrixUseA> matA;
coopmat<float16_t, gl_ScopeSubgroup, 16, 16, gl_MatrixUseB> matB;
coopmat<float32_t, gl_ScopeSubgroup, 16, 16, gl_MatrixUseAccumulator> matC;

coopMatLoad(matA, bufferA, offset, stride, gl_CooperativeMatrixLayoutRowMajor);
coopMatLoad(matB, bufferB, offset, stride, gl_CooperativeMatrixLayoutColumnMajor);
coopMatMulAdd(matA, matB, matC);
coopMatStore(matC, bufferC, offset, stride, gl_CooperativeMatrixLayoutRowMajor);
```

Chapter 141 covers `VK_KHR_cooperative_matrix` in detail as a Vulkan extension; this chapter treats it as the mechanism for integrating GEMM-tile throughput into graphics-pipeline compute shaders.

---

## 12. Integration with the Graphics Pipeline

The sections above treat linear algebra as a standalone workload. This section covers how it connects to a Vulkan graphics pipeline where the computed solution feeds vertex attributes, physics state, or deformation maps for the next rendered frame.

### 12.1 Vulkan Compute Dispatch for GPU-Resident GEMM

A physics simulation step (constraint resolution, FEM stiffness solve) executes as a Vulkan compute dispatch. The result — updated positions, velocities — lives in a `VkBuffer`. A subsequent render pass reads the same buffer as a vertex buffer or via a shader storage buffer binding. No CPU round-trip is needed if the compute and render passes are in the same command buffer or linked by a Vulkan semaphore.

The pipeline barrier between the compute dispatch (STORE of position buffer) and the vertex input stage (LOAD of the same buffer):

```c
VkBufferMemoryBarrier barrier = {
    .sType         = VK_STRUCTURE_TYPE_BUFFER_MEMORY_BARRIER,
    .srcAccessMask = VK_ACCESS_SHADER_WRITE_BIT,
    .dstAccessMask = VK_ACCESS_VERTEX_ATTRIBUTE_READ_BIT,
    .srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED,
    .dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED,
    .buffer        = physics_result_buf,
    .offset        = 0,
    .size          = VK_WHOLE_SIZE,
};
vkCmdPipelineBarrier(
    cmd,
    VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT,  // srcStageMask
    VK_PIPELINE_STAGE_VERTEX_INPUT_BIT,    // dstStageMask
    0,
    0, NULL,
    1, &barrier,
    0, NULL
);
```

If the physics solver runs on a separate async compute queue (for overlap with the previous frame's render), a `VkSemaphore` with `VK_KHR_timeline_semaphore` replaces the pipeline barrier:

```c
// Async compute queue: signal timeline semaphore at physics_done_value
VkTimelineSemaphoreSubmitInfo ts_info = {
    .sType                     = VK_STRUCTURE_TYPE_TIMELINE_SEMAPHORE_SUBMIT_INFO,
    .signalSemaphoreValueCount = 1,
    .pSignalSemaphoreValues    = &physics_done_value,
};
vkQueueSubmit(compute_queue, 1, &submit_info, VK_NULL_HANDLE);

// Graphics queue: wait for physics_done_value before vertex fetch
VkTimelineSemaphoreSubmitInfo wait_ts = {
    .waitSemaphoreValueCount = 1,
    .pWaitSemaphoreValues    = &physics_done_value,
};
vkQueueSubmit(graphics_queue, 1, &render_submit, VK_NULL_HANDLE);
```

### 12.2 rocSPARSE SpMV Inside a Physics Step

A finite-element solver computes f = K·u (global stiffness matrix times displacement vector) as a GPU SpMV. The result f (force residual) feeds the next CG iteration. The same buffer backing f is then (after convergence) read by a vertex shader to displace mesh vertices:

```c
// 1. Solve K·u = f_ext using rocSPARSE + rocBLAS CG (GPU-resident)
//    u lives in d_u (HIP device buffer)

// 2. Copy d_u → Vulkan VkBuffer via DMA-BUF or
//    via hipMemcpy if they share the same GPU allocation
//    (on ROCm, HIP and Vulkan can share device memory — see below)

// 3. Record a VkCmdPipelineBarrier and issue the render pass
//    with the position buffer reading from d_u's Vulkan alias
```

On ROCm, HIP device allocations and Vulkan device allocations share the same GPU VRAM. A HIP allocation can be imported as a Vulkan external memory object using `VK_EXT_external_memory_host` (for host-accessible memory) or via DRM prime handles:

```c
// Export HIP allocation as a DRM prime file descriptor (ROCm-specific)
hipMemGetAddressRange(...);  // get base of HIP allocation
// Use VK_EXT_external_memory_dma_buf to import:
VkImportMemoryFdInfoKHR import_info = {
    .sType      = VK_STRUCTURE_TYPE_IMPORT_MEMORY_FD_INFO_KHR,
    .handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
    .fd         = drm_prime_fd,   // from rocm / DRM export
};
```

[Note: The exact ROCm API for exporting a HIP allocation as a DRM prime FD needs verification against the current ROCm Interop documentation at https://rocm.docs.amd.com/. The `VK_EXT_external_memory_dma_buf` + `VK_KHR_external_memory_fd` path for importing DMA-BUF handles into Vulkan is standardised.]

### 12.3 OpenCL Interop with DRM Prime Handles

OpenCL on Linux uses the `cl_khr_dx9_media_sharing` and `cl_khr_gl_sharing` extensions for interop with graphics APIs, and `cl_khr_external_memory_dma_buf` for DRM prime-based zero-copy sharing. A SpMV result computed in OpenCL can be passed to a Vulkan render pass via:

```
OpenCL kernel writes to cl_mem  →  export as DRM-prime fd
Vulkan: import fd via VK_EXT_external_memory_dma_buf  →  bind as VkBuffer
VkPipelineBarrier  →  render pass reads vertex positions
```

The `cl_khr_external_memory_dma_buf` extension requires a GPU driver that exposes DRM prime import/export at the GEM level — all Mesa-based drivers (radeonsi, anv, iris) support this. [Source: OpenCL Extension Registry, https://registry.khronos.org/OpenCL/extensions/]

### 12.4 Practical Latency Budget

For a real-time physics + render loop at 60 fps (16.7 ms budget):
- Render pass: ~8 ms
- Physics CG solve (50 iterations, 100k DOF): ~4 ms (well-tuned)
- Pipeline barrier overhead: <0.1 ms
- Total: ~12 ms with 4.7 ms margin for other workloads

Using the async compute queue for the physics step and double-buffering the solution vectors (one being read by the renderer while the next is being computed) recovers the physics solve latency entirely, allowing the renderer to see zero added latency from the physics computation.

---

## Integrations

**Ch210 (GPU Physics and Volumetric Algorithms).** The physics simulation stiffness matrices constructed in Ch210 are the primary input to the sparse solvers of §7 (CG, BiCGSTAB) and the AMG preconditioners of §8. The GPU-resident iteration patterns of §7 connect directly to Ch210's constraint projection loops. The Vulkan timeline semaphore integration of §12 synchronises the physics compute result with Ch210's volume rendering pass.

**Ch221 (GPU Algorithm Performance).** The arithmetic intensity analysis of §1 applies the roofline methodology developed in Ch221. GEMM register-blocking decisions (§2.4), SpMV load-balancing strategies (§6.1), and the GPU-resident iteration decision (§7.1) are all validated using the occupancy and bandwidth measurement tools of Ch221 (rocprof, Radeon GPU Profiler).

**Ch224 (3D Shape Analysis Algorithms).** Laplacian eigenvectors for shape descriptors (heat kernel signature, wave kernel signature) require solving the generalised eigenvalue problem (L − λM)v = 0 where L is the cotan-Laplacian sparse matrix. The Lanczos eigensolver (§4.2), LOBPCG (§4.3), and sparse format selection (§5) for L apply directly to the geometry processing pipeline of Ch224.

**Ch225 (Computational Topology Algorithms).** The discrete Morse complex and persistent homology boundary matrix reduction of Ch225 produce sparse matrices whose eigenstructure is analysed using the solvers here. The graph Laplacian of the simplicial complex adjacency, used in §4 of Ch225, is an integer sparse matrix suitable for INT8 or FP32 treatment (§11).

**Ch227 (Monte Carlo Sampling Using Linear Algebra).** Monte Carlo importance sampling requires solving sparse systems (Laplacian smoothing of density estimates, transport equations on graphs). The CG and AMG solvers of §7–§8, and the Toeplitz FFT convolution of §9, provide the numerical engine that Ch227 calls for its density-estimation and sampling correction steps.

**Ch229 (ML Inference GEMM).** The batched GEMM infrastructure of §2 — `rocblas_dgemm_batched`, MAGMA `_vbatched`, cooperative matrix tiles (§11.3) — is the same throughput path used in Ch229 for neural network layer evaluation. The mixed-precision iterative refinement (§11.1) and BF16 MFMA acceleration (§11.2) connect the linear algebra precision engineering of this chapter to ML inference quantisation discussed in Ch229.
