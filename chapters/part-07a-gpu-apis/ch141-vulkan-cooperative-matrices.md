# Chapter 141: Vulkan Cooperative Matrices and GPU ML Acceleration

**Target audiences**: ML engineers using Vulkan compute for inference without CUDA/ROCm lock-in, Vulkan compute developers integrating matrix-multiply acceleration, GPU ML framework authors targeting Linux multi-vendor hardware, and graphics engineers adding ML-accelerated effects (denoising, upscaling, super-resolution) to rendering pipelines.

---

## Table of Contents

1. [Introduction](#introduction)
2. [The Matrix Multiply Problem and Hardware MMA Units](#the-matrix-multiply-problem-and-hardware-mma-units)
3. [VK_KHR_cooperative_matrix API](#vk_khr_cooperative_matrix-api)
4. [GLSL Cooperative Matrix Shaders](#glsl-cooperative-matrix-shaders)
5. [Hardware Mapping](#hardware-mapping)
6. [RADV Implementation on RDNA3+](#radv-implementation-on-rdna3)
7. [ANV Implementation on Intel Xe-HPG+](#anv-implementation-on-intel-xe-hpg)
8. [Quantization and Mixed Precision](#quantization-and-mixed-precision)
9. [Practical ML Use Cases on Linux](#practical-ml-use-cases-on-linux)
10. [Performance Tuning](#performance-tuning)
11. [Integrations](#integrations)

---

## Introduction

Modern GPUs contain dedicated matrix multiply-accumulate (MMA) hardware units: NVIDIA's Tensor Cores (Volta, 2017), AMD's Matrix Cores (CDNA, 2020; RDNA3, 2022), and Intel's XMX units (Xe-HPG, 2021). These units are the primary compute workhorses for AI/ML workloads, delivering 10–100× the throughput of regular floating-point units for matrix operations at the heart of deep learning: GEMM (general matrix multiply), convolutional layers, and the attention mechanism in transformers.

`VK_KHR_cooperative_matrix` (promoted to KHR status in 2023) exposes these units through the Vulkan API, enabling ML inference and linear algebra acceleration without CUDA, ROCm/HIP, or SYCL. On Linux, RADV (AMD RDNA3, RX 7000 series) and ANV (Intel DG2/Arc and later) implement this extension. This portability means an LLM inference engine or stable diffusion compute shader can run on AMD, Intel, and NVIDIA GPUs through the same code path.

This chapter covers:

- **Cooperative matrix programming model** — where a subgroup of shader invocations cooperates to compute a small matrix tile
- **Vulkan and GLSL API** — the extension's host and shader interfaces
- **Hardware ISA instructions** — AMD WMMA and Intel DPAS, which back the extension
- **RADV and ANV Mesa implementations** — driver-level implementation on AMD RDNA3 and Intel Xe-HPG
- **Quantization support** — INT8, mixed precision, FP8, BF16, and block quantization
- **Real-world usage in LLM inference tools on Linux** — practical examples including llama.cpp and stable-diffusion.cpp

[VK_KHR_cooperative_matrix specification](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_cooperative_matrix.html)

---

## The Matrix Multiply Problem and Hardware MMA Units

### GEMM: The Core Operation

The fundamental operation is:

```
C[M×N] += A[M×K] × B[K×N]
```

For a transformer FFN layer: M = batch size × sequence length, K = embedding dim (e.g. 4096), N = hidden dim (e.g. 16384). This requires M×N×K multiply-accumulate operations — at 4096×4096×16384, that is 274 billion MACs per forward pass through one FFN layer.

Naïve GPU parallelism assigns one output element C[i,j] per thread, requiring each thread to accumulate K values. This is extremely memory-bandwidth bound: K reads from A row i and K reads from B column j per output element.

### Tensor Core / Matrix Core Insight

Instead of one thread per output element, **a group of 32–128 threads cooperates** to compute a small tile of C (e.g. 16×16) from a tile of A (16×K_tile) and B (K_tile×16). Each thread holds a fragment (slice) of A, B, and C in registers. The hardware MMA instruction performs the multiply-accumulate for all threads simultaneously using dedicated pipelined multiplier arrays.

NVIDIA Ampere A100: `mma.sync.aligned.m16n8k16` in PTX — one instruction, 16×8×16 = 2048 multiplications, per 4 clock cycles, per SM warp.

AMD RDNA3 RX 7900 XTX: `v_wmma_f32_16x16x16_f16` — one 16×16×16 WMMA instruction per 4 clock cycles, per CU wavefront.

Intel Arc A770 (Xe-HPG): `dpas` (Dot Product Accumulate Systolic) — processes 8×8 tiles per instruction in a systolic array.

### Why This Matters for ML

At FP16 precision:
- RDNA3 RX 7900 XTX: 82.6 TFLOPS ML (WMMA)
- Intel Arc A770: 34.1 TFLOPS ML (XMX)
- RDNA3 RX 7600 (entry level): 21.5 TFLOPS ML

These peak rates are achievable for GEMM workloads with correctly tiled shaders using `VK_KHR_cooperative_matrix`.

---

## VK_KHR_cooperative_matrix API

### Feature and Property Queries

```c
VkPhysicalDeviceCooperativeMatrixFeaturesKHR coopMatFeatures = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_COOPERATIVE_MATRIX_FEATURES_KHR,
};
/* Check: coopMatFeatures.cooperativeMatrix == VK_TRUE */

/* Query supported configurations: */
uint32_t count;
vkGetPhysicalDeviceCooperativeMatrixPropertiesKHR(physDevice, &count, NULL);

std::vector<VkCooperativeMatrixPropertiesKHR> props(count);
for (auto &p : props) p.sType =
    VK_STRUCTURE_TYPE_COOPERATIVE_MATRIX_PROPERTIES_KHR;
vkGetPhysicalDeviceCooperativeMatrixPropertiesKHR(physDevice, &count, props.data());

/* Each entry describes one supported (M, N, K, type) combination: */
for (auto &p : props) {
    printf("M=%d N=%d K=%d AType=%d BType=%d CType=%d ResultType=%d scope=%d\n",
        p.MSize, p.NSize, p.KSize,
        p.AType, p.BType, p.CType, p.ResultType,
        p.scope);
}
```

### Typical Supported Configurations

| Hardware | M×N×K | AType | BType | CType | ResultType |
|---|---|---|---|---|---|
| RDNA3 (RX 7900) | 16×16×16 | FLOAT16 | FLOAT16 | FLOAT32 | FLOAT32 |
| RDNA3 | 16×16×16 | FLOAT16 | FLOAT16 | FLOAT16 | FLOAT16 |
| RDNA3 | 16×16×16 | INT8 | INT8 | INT32 | INT32 |
| Intel Arc (Xe-HPG) | 8×8×32 | INT8 | INT8 | INT32 | INT32 |
| Intel Arc | 8×8×16 | FLOAT16 | FLOAT16 | FLOAT32 | FLOAT32 |
| NVIDIA (via VK_NV_cooperative_matrix) | 16×16×16 | FLOAT16 | FLOAT16 | FLOAT32 | FLOAT32 |

### Component Types

```c
/* VkComponentTypeKHR */
VK_COMPONENT_TYPE_FLOAT16_KHR
VK_COMPONENT_TYPE_FLOAT32_KHR
VK_COMPONENT_TYPE_FLOAT64_KHR
VK_COMPONENT_TYPE_SINT8_KHR
VK_COMPONENT_TYPE_SINT16_KHR
VK_COMPONENT_TYPE_SINT32_KHR
VK_COMPONENT_TYPE_UINT8_KHR
VK_COMPONENT_TYPE_UINT32_KHR
/* Added via extension VK_KHR_cooperative_matrix rev 2 (2024): */
VK_COMPONENT_TYPE_BFLOAT16_KHR
```

### Scope

```c
/* VkScopeKHR */
VK_SCOPE_SUBGROUP_KHR  /* 32 invocations on AMD, 8 or 16 on Intel */
VK_SCOPE_WORKGROUP_KHR /* entire workgroup (less common) */
```

Most implementations use subgroup scope: the 32 shader invocations in a warp/wavefront collectively hold one or more tile fragments.

---

## GLSL Cooperative Matrix Shaders

### Declaring Cooperative Matrix Types

```glsl
#version 460
#extension GL_KHR_cooperative_matrix : require
#extension GL_EXT_shader_explicit_arithmetic_types_float16 : require

const int M = 16, N = 16, K = 16;

/* Matrix use enum: */
/* gl_MatrixUseA, gl_MatrixUseB, gl_MatrixUseAccumulator */

/* Declare tile types: */
coopmat<float16_t, gl_ScopeSubgroup, 16, 16, gl_MatrixUseA>       tileA;
coopmat<float16_t, gl_ScopeSubgroup, 16, 16, gl_MatrixUseB>       tileB;
coopmat<float32_t, gl_ScopeSubgroup, 16, 16, gl_MatrixUseAccumulator> tileC;
```

### GEMM Kernel: Tiled Matrix Multiply

```glsl
#version 460
#extension GL_KHR_cooperative_matrix : require
#extension GL_EXT_shader_explicit_arithmetic_types_float16 : require

/* Input: A[M_TOTAL×K_TOTAL] float16, B[K_TOTAL×N_TOTAL] float16
   Output: C[M_TOTAL×N_TOTAL] float32 */

layout(local_size_x = 32, local_size_y = 1, local_size_z = 1) in;

layout(set=0, binding=0) readonly buffer BufferA { float16_t A[]; };
layout(set=0, binding=1) readonly buffer BufferB { float16_t B[]; };
layout(set=0, binding=2) writeonly buffer BufferC { float C[]; };

layout(push_constant) uniform Dims {
    uint M, N, K;
} dims;

const uint TILE_M = 16, TILE_N = 16, TILE_K = 16;

void main() {
    uint row_tile = gl_WorkGroupID.y;
    uint col_tile = gl_WorkGroupID.x;

    /* Base indices for this workgroup's output tile */
    uint base_m = row_tile * TILE_M;
    uint base_n = col_tile * TILE_N;

    /* Accumulator tile: initialise to zero */
    coopmat<float32_t, gl_ScopeSubgroup, TILE_M, TILE_N,
            gl_MatrixUseAccumulator> acc;
    coopMatConstruct(acc, 0.0f);

    /* Loop over K tiles */
    for (uint k = 0; k < dims.K; k += TILE_K) {
        /* Load A tile: row-major, offset [base_m, k] */
        coopmat<float16_t, gl_ScopeSubgroup, TILE_M, TILE_K,
                gl_MatrixUseA> tileA;
        coopMatLoad(tileA, A, base_m * dims.K + k,
                    dims.K,   /* column stride (elements per row) */
                    gl_CooperativeMatrixLayoutRowMajor);

        /* Load B tile: row-major, offset [k, base_n] */
        coopmat<float16_t, gl_ScopeSubgroup, TILE_K, TILE_N,
                gl_MatrixUseB> tileB;
        coopMatLoad(tileB, B, k * dims.N + base_n,
                    dims.N,
                    gl_CooperativeMatrixLayoutRowMajor);

        /* Multiply-Accumulate: acc += tileA × tileB */
        acc = coopMatMulAdd(tileA, tileB, acc);
    }

    /* Store result tile: C[base_m .. base_m+TILE_M, base_n .. base_n+TILE_N] */
    coopMatStore(acc, C, base_m * dims.N + base_n,
                 dims.N,
                 gl_CooperativeMatrixLayoutRowMajor);
}
```

**Slang equivalent** — `typealias` names each tile type once (eliminating repeated `coopmat<T,scope,M,N,use>` arguments), `[[vk::binding]]` moves SSBO layout metadata to function parameters instead of global blocks, and the method-syntax `.fill()` / `.load()` / `.store()` replaces the GLSL free-function calls for a more self-documenting tiling loop.

```slang
// File: gemm.slang
// Tiled FP16×FP16→FP32 GEMM using Vulkan cooperative matrices
// Slang improvements:
//   - typealias TileA/TileB/TileAcc — CoopMatrix type arguments written once, reused everywhere
//   - [[vk::binding]] / [[vk::push_constant]] on parameters — no global layout blocks
//   - .fill() / .load() / .store() method syntax — load layout enum is scoped, not a bare integer

static const uint TILE_M = 16;
static const uint TILE_N = 16;
static const uint TILE_K = 16;

struct Dims { uint M, N, K; }

typealias TileA   = CoopMatrix<float16_t, CoopMatrixScope.Subgroup, TILE_M, TILE_K, CoopMatrixUse.A>;
typealias TileB   = CoopMatrix<float16_t, CoopMatrixScope.Subgroup, TILE_K, TILE_N, CoopMatrixUse.B>;
typealias TileAcc = CoopMatrix<float,     CoopMatrixScope.Subgroup, TILE_M, TILE_N, CoopMatrixUse.Accumulator>;

[numthreads(32, 1, 1)]
[shader("compute")]
void main(
    uint3                                           wgID  : SV_GroupID,
    [[vk::binding(0, 0)]] StructuredBuffer<float16_t>   A,
    [[vk::binding(1, 0)]] StructuredBuffer<float16_t>   B,
    [[vk::binding(2, 0)]] RWStructuredBuffer<float>     C,
    [[vk::push_constant]]  ConstantBuffer<Dims>          dims)
{
    uint base_m = wgID.y * TILE_M;
    uint base_n = wgID.x * TILE_N;

    // Single call fills all per-invocation accumulator registers with zero
    TileAcc acc;
    acc.fill(0.0f);

    for (uint k = 0; k < dims.K; k += TILE_K) {
        // Static method on the tile type — layout enum is scoped, not a bare gl_ constant
        TileA tileA = TileA.load(A, base_m * dims.K + k, dims.K,
                                 CoopMatrixLayout.RowMajor);
        TileB tileB = TileB.load(B, k * dims.N + base_n,  dims.N,
                                 CoopMatrixLayout.RowMajor);
        // coopMatMulAdd maps to the same OpCooperativeMatrixMulAddKHR SPIR-V opcode
        acc = coopMatMulAdd(tileA, tileB, acc);
    }

    // Method-syntax store — no need to repeat the type name or layout enum prefix
    acc.store(C, base_m * dims.N + base_n, dims.N, CoopMatrixLayout.RowMajor);
}
```

Compile with:

```bash
glslc --target-env=vulkan1.3 -o gemm.spv gemm.comp.glsl
spirv-val --target-env vulkan1.3 gemm.spv
```

### Key GLSL Functions

| Function | Description |
|---|---|
| `coopMatConstruct(mat, value)` | Fill matrix with constant |
| `coopMatLoad(mat, buffer, offset, stride, layout)` | Load from SSBO |
| `coopMatStore(mat, buffer, offset, stride, layout)` | Store to SSBO |
| `coopMatMulAdd(A, B, C)` | C += A × B; returns result |
| `coopMatElementwise(mat, func)` | Per-element operation (GLSL extension) |

### Layout Enum

```glsl
gl_CooperativeMatrixLayoutRowMajor      /* standard row-major C layout */
gl_CooperativeMatrixLayoutColumnMajor   /* column-major (Fortran-style) */
```

### SPIR-V Opcodes

The GLSL cooperative matrix constructs lower to:

- `OpTypeCooperativeMatrixKHR` — type declaration
- `OpCooperativeMatrixLoadKHR` — load from pointer
- `OpCooperativeMatrixStoreKHR` — store to pointer
- `OpCooperativeMatrixMulAddKHR` — matrix multiply-accumulate

[SPIR-V KHR_cooperative_matrix spec](https://htmlpreview.github.io/?https://github.com/KhronosGroup/SPIRV-Registry/blob/main/extensions/KHR/SPV_KHR_cooperative_matrix.html)

---

## Hardware Mapping

### NVIDIA: Tensor Cores and wmma

NVIDIA Volta+ Tensor Cores are exposed through PTX `mma.sync.aligned` instructions. In Vulkan, the NVIDIA proprietary driver maps `OpCooperativeMatrixMulAddKHR` to these instructions. NVIDIA also provides `VK_NV_cooperative_matrix2` (a superset of KHR) with additional features:

- FP8 E4M3 / E5M2 (Ada/Hopper)
- 8-bit integer saturating accumulation
- Tensor memory layout hints for better cache usage

```ptx
/* PTX: NVIDIA Ampere sm_80 matrix multiply FP16×FP16+FP32 */
mma.sync.aligned.m16n8k16.row.col.f32.f16.f16.f32
    {d0, d1, d2, d3},
    {a0, a1, a2, a3},
    {b0, b1},
    {c0, c1, c2, c3};
```

### AMD RDNA3: WMMA

AMD RDNA3 introduces the WMMA (Wave Matrix Multiply Accumulate) instruction family, distinct from CDNA's MFMA. WMMA targets graphics + compute workloads on mainstream GPU (RX 7000 series):

```
v_wmma_f32_16x16x16_f16     v_dst[8], v_src_a[8], v_src_b[8], v_src_c[8]
v_wmma_f16_16x16x16_f16     v_dst[4], v_src_a[8], v_src_b[8], v_src_c[4]
v_wmma_i32_16x16x16_iu8     v_dst[8], v_src_a[4], v_src_b[4], v_src_c[8]
v_wmma_f32_16x16x16_bf16    v_dst[8], v_src_a[8], v_src_b[8], v_src_c[8]
```

Each `v_wmma_f32_16x16x16_f16` instruction processes a 16×16×16 tile across 32 shader invocations. The 32 VGPRs in `v_src_a[8]` and `v_src_b[8]` hold 32 half-precision values (8 VGPRs × 2 fp16/VGPR × 2 matrix fragments = 32 fp16 values per invocation... with the full 32-lane subgroup completing the 16×16 tiles).

Note: AMD CDNA2 (MI250X) uses `v_mfma_f32_16x16x16f16` with AGPR registers (distinct from RDNA WMMA). Vulkan's `VK_KHR_cooperative_matrix` targets both, but the VGPR vs AGPR distinction is invisible to the Vulkan programmer — RADV handles it per-target.

### Intel Xe-HPG: DPAS

Intel Arc Alchemist (Xe-HPG) uses **DPAS** (Dot Product Accumulate Systolic), a systolic array instruction exposed via the EU (Execution Unit) message interface:

```
dpas.8x8.f32.f16.f16  dst(8), src0(8), src1(1×8), src2(8×1)
```

DPAS on Xe-HPG processes 8 rows × 8 cols with configurable K depth. Intel Xe2 (Battlemage) extends DPAS to larger tiles and adds FP8 support.

[Intel Xe-HPG DPAS reference](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-xe-gpu-architecture.html)

---

## RADV Implementation on RDNA3+

### Platform Requirements

`VK_KHR_cooperative_matrix` in RADV requires RDNA3 or later (`gfx11+`, i.e. RX 7000 series). RDNA2 (RX 6000) does not have WMMA and cannot support the extension in hardware (software emulation would be prohibitively slow).

### Feature Detection

```c
/* src/amd/vulkan/radv_physical_device.c */
void radv_physical_device_get_cooperative_matrix_properties(
    struct radv_physical_device *pdev,
    uint32_t *count,
    VkCooperativeMatrixPropertiesKHR *props)
{
    if (pdev->rad_info.gfx_level < GFX11) {
        *count = 0;
        return;
    }
    /* Populate WMMA-backed configurations */
}
```

### NIR Lowering

RADV lowers `OpCooperativeMatrixMulAddKHR` in NIR through `nir_lower_cooperative_matrix`:

```
SPIR-V OpCooperativeMatrixMulAddKHR
  → nir_intrinsic_cooperative_matrix_muladd
  → nir_lower_cooperative_matrix() [per-invocation register slicing]
  → ACO backend: emit v_wmma_* instructions
```

Key file: `src/amd/vulkan/radv_nir_lower_cooperative_matrix.c` [Source](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/amd/vulkan).

### ACO WMMA Emission

The ACO (AMD compiler) backend maps NIR cooperative matrix intrinsics to WMMA instructions:

```c
/* ACO instruction selection for RDNA3 WMMA: */
case nir_intrinsic_cooperative_matrix_muladd: {
    if (element_type == FLOAT16 && acc_type == FLOAT32)
        bld.vop3p(aco_opcode::v_wmma_f32_16x16x16_f16, dst, a, b, c);
    else if (element_type == INT8 && acc_type == INT32)
        bld.vop3p(aco_opcode::v_wmma_i32_16x16x16_iu8, dst, a, b, c);
    break;
}
```

### Debugging RADV Cooperative Matrix

```bash
# Dump shader before/after cooperative matrix lowering:
RADV_DEBUG=spirv,shaders NIR_PRINT=1 ./llama_vulkan

# Check feature support:
vulkaninfo --json 2>/dev/null | jq '.[] | .features |
    .VkPhysicalDeviceCooperativeMatrixFeaturesKHR'

# List supported coop mat configurations on RADV:
# (build vulkan-samples or write a small query program)
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/radeon_icd.x86_64.json \
    ./coop_mat_query
```

---

## ANV Implementation on Intel Xe-HPG+

### Platform Requirements

`VK_KHR_cooperative_matrix` in ANV requires Intel DG2 (Arc Alchemist, `gfx1255`+) or later. Integrated graphics (Iris Gen12) lack XMX units and do not support the extension.

### Property Query

```c
/* src/intel/vulkan/anv_extensions.c */
VK_FROM_HANDLE(anv_physical_device, pdev, physicalDevice);

if (pdev->info.has_cooperative_matrix) {
    /* Supported on DG2+: 8x8xK INT8/FP16 tiles */
    props[n++] = (VkCooperativeMatrixPropertiesKHR){
        .MSize = 8, .NSize = 8, .KSize = 16,
        .AType = VK_COMPONENT_TYPE_FLOAT16_KHR,
        .BType = VK_COMPONENT_TYPE_FLOAT16_KHR,
        .CType = VK_COMPONENT_TYPE_FLOAT32_KHR,
        .ResultType = VK_COMPONENT_TYPE_FLOAT32_KHR,
        .scope = VK_SCOPE_SUBGROUP_KHR,
    };
}
```

### NIR to DPAS

ANV's cooperative matrix lowering maps to Intel DPAS:

```
OpCooperativeMatrixMulAddKHR (FLOAT16, 8×8×16)
  → nir_intrinsic_cooperative_matrix_muladd
  → nir_lower_cooperative_matrix()
  → intel_nir_lower_dpas.c: emit DPAS instruction
  → EU ISA: dpas.8x8.f32.hf.hf [systolic array]
```

Intel's XMX (Xe Matrix Extensions) in Arc Alchemist supports INT8 at 274 TOPS and FP16 at 137 TFLOPS (peak, for A770 160W).

### Debugging ANV Cooperative Matrix

```bash
# Enable Intel driver debug:
INTEL_DEBUG=cs,spirv ./llama_vulkan 2>&1 | grep -i "dpas\|wmma\|coop"

# Check features:
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/intel_icd.x86_64.json \
    vulkaninfo --json 2>/dev/null | jq '.[] | .features |
    .VkPhysicalDeviceCooperativeMatrixFeaturesKHR'
```

---

## Quantization and Mixed Precision

### Why Quantize

A 7B parameter LLM in FP32 requires 28 GB of VRAM — exceeding most consumer GPUs. INT8 quantization halves this to 14 GB; INT4 (4-bit) reduces to 7 GB, fitting on an 8 GB RX 7600.

Cooperative matrices natively support INT8 GEMM:

```glsl
/* INT8 GEMM accumulating into INT32: */
coopmat<int8_t,  gl_ScopeSubgroup, 16, 16, gl_MatrixUseA>          tileA_i8;
coopmat<int8_t,  gl_ScopeSubgroup, 16, 16, gl_MatrixUseB>          tileB_i8;
coopmat<int32_t, gl_ScopeSubgroup, 16, 16, gl_MatrixUseAccumulator> acc_i32;

coopMatLoad(tileA_i8, quantA, offset_a, stride_a,
            gl_CooperativeMatrixLayoutRowMajor);
coopMatLoad(tileB_i8, quantB, offset_b, stride_b,
            gl_CooperativeMatrixLayoutColumnMajor);
acc_i32 = coopMatMulAdd(tileA_i8, tileB_i8, acc_i32);
/* Then scale by quantization factors and convert to FP16 for output */
```

**Slang equivalent** — a single `IMatrixPrecision` interface with associated `ElementType`/`AccumType` types lets one generic `tiled_gemm<P>()` helper cover both FP16→FP32 and INT8→INT32 without duplicating the tiling loop; the compiler generates separate SPIR-V entry points for each concrete specialisation.

```slang
// File: gemm_mixed_precision.slang
// Generic GEMM demonstrating Slang interface generics for mixed-precision variants
// Slang improvements:
//   - IMatrixPrecision interface: associated ElementType + AccumType unify FP16 and INT8 paths
//   - Single tiled_gemm<P>() helper — the K-loop is written once; both HW WMMA variants generated
//   - Concrete entry points gemm_fp16 / gemm_int8 carry correct [[vk::binding]] types

static const uint TILE_M = 16, TILE_N = 16, TILE_K = 16;

struct Dims { uint M, N, K; }

// ── Precision interface ───────────────────────────────────────────────────
interface IMatrixPrecision {
    associatedtype ElementType;   // e.g. float16_t or int8_t
    associatedtype AccumType;     // e.g. float or int
    static AccumType zero();      // typed zero for acc.fill()
}

struct FP16Precision : IMatrixPrecision {
    typedef float16_t ElementType;
    typedef float     AccumType;
    static float zero() { return 0.0f; }
}

struct INT8Precision : IMatrixPrecision {
    typedef int8_t  ElementType;
    typedef int     AccumType;
    static int zero() { return 0; }
}

// ── Generic tiling loop — NOT an entry point ──────────────────────────────
void tiled_gemm<P : IMatrixPrecision>(
    uint3                           wgID,
    StructuredBuffer<P.ElementType> A,
    StructuredBuffer<P.ElementType> B,
    RWStructuredBuffer<P.AccumType> C,
    Dims                            dims)
{
    // typealias inside a generic function — rows/cols fixed; element/accum types flow from P
    typealias TileA   = CoopMatrix<P.ElementType, CoopMatrixScope.Subgroup, TILE_M, TILE_K, CoopMatrixUse.A>;
    typealias TileB   = CoopMatrix<P.ElementType, CoopMatrixScope.Subgroup, TILE_K, TILE_N, CoopMatrixUse.B>;
    typealias TileAcc = CoopMatrix<P.AccumType,   CoopMatrixScope.Subgroup, TILE_M, TILE_N, CoopMatrixUse.Accumulator>;

    uint base_m = wgID.y * TILE_M;
    uint base_n = wgID.x * TILE_N;

    TileAcc acc;
    acc.fill(P.zero());   // P.zero() resolves to 0.0f (FP16) or 0 (INT8) at compile time

    for (uint k = 0; k < dims.K; k += TILE_K) {
        TileA tileA = TileA.load(A, base_m * dims.K + k, dims.K, CoopMatrixLayout.RowMajor);
        TileB tileB = TileB.load(B, k * dims.N + base_n,  dims.N, CoopMatrixLayout.ColumnMajor);
        acc = coopMatMulAdd(tileA, tileB, acc);
    }
    // Scale and type-convert after the loop (caller or a follow-on shader)
    acc.store(C, base_m * dims.N + base_n, dims.N, CoopMatrixLayout.RowMajor);
}

// ── Concrete FP16 → FP32 entry point ─────────────────────────────────────
[numthreads(32, 1, 1)]
[shader("compute")]
void gemm_fp16(
    uint3                                           wgID  : SV_GroupID,
    [[vk::binding(0, 0)]] StructuredBuffer<float16_t>   A,
    [[vk::binding(1, 0)]] StructuredBuffer<float16_t>   B,
    [[vk::binding(2, 0)]] RWStructuredBuffer<float>     C,
    [[vk::push_constant]]  ConstantBuffer<Dims>          dims)
{
    tiled_gemm<FP16Precision>(wgID, A, B, C, dims);
}

// ── Concrete INT8 → INT32 entry point ────────────────────────────────────
[numthreads(32, 1, 1)]
[shader("compute")]
void gemm_int8(
    uint3                                          wgID  : SV_GroupID,
    [[vk::binding(0, 0)]] StructuredBuffer<int8_t>    A,
    [[vk::binding(1, 0)]] StructuredBuffer<int8_t>    B,
    [[vk::binding(2, 0)]] RWStructuredBuffer<int>     C,
    [[vk::push_constant]]  ConstantBuffer<Dims>        dims)
{
    tiled_gemm<INT8Precision>(wgID, A, B, C, dims);
}
```

### Saturating Accumulation

```c
/* VkCooperativeMatrixPropertiesKHR */
VkBool32 saturatingAccumulation; /* If VK_TRUE, INT32 overflow saturates */
```

Saturating accumulation prevents INT32 overflow in deep INT8 GEMMs without requiring software overflow checks.

### FP8 (E4M3 / E5M2)

FP8 (8-bit floating point) halves memory bandwidth vs FP16 at near-equivalent quality for inference. Two formats:
- **E4M3**: 4-bit exponent, 3-bit mantissa — higher dynamic range, used for weights
- **E5M2**: 5-bit exponent, 2-bit mantissa — better for gradients

FP8 Vulkan support (via `VK_COMPONENT_TYPE_FLOAT8_E4M3_EXT` and `VK_COMPONENT_TYPE_FLOAT8_E5M2_EXT`) requires `VK_EXT_shader_float8` which is being implemented for RDNA3 in RADV as of mid-2025. NVIDIA Ada (sm_89) and Hopper (sm_90) hardware support FP8 natively; AMD MI300 (CDNA3) supports FP8.

### BFloat16 (BF16)

BF16 shares the same 8-bit exponent as FP32 (no re-normalisation needed) with a 7-bit mantissa. Useful for training (larger dynamic range than FP16). `VK_COMPONENT_TYPE_BFLOAT16_KHR` is available in the KHR extension:

```glsl
/* BF16 accumulation: */
coopmat<bfloat16_t, gl_ScopeSubgroup, 16, 16, gl_MatrixUseA> tileA_bf16;
```

RDNA3 supports BF16 via `v_wmma_f32_16x16x16_bf16`.

### Block Quantization (GGUF Format)

llama.cpp's GGUF format uses block quantization: every 32 or 64 elements share a scale factor. The Vulkan GEMM kernel must dequantize A and B tiles before loading into cooperative matrices:

```glsl
/* Dequantize INT4 block to FP16 before cooperative matrix load: */
void dequant_q4_k(in uint packed, in float scale, out float16_t vals[8]) {
    for (int i = 0; i < 8; i++) {
        vals[i] = float16_t(float((packed >> (i*4)) & 0xF) * scale);
    }
}
```

---

## Practical ML Use Cases on Linux

### llama.cpp Vulkan Backend

[llama.cpp](https://github.com/ggerganov/llama.cpp) is the leading open-source LLM inference framework. Its Vulkan compute backend uses `VK_KHR_cooperative_matrix` on supported hardware:

```bash
# Build with Vulkan support:
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
cmake -B build -DGGML_VULKAN=ON -DCMAKE_BUILD_TYPE=Release
cmake --build build -j$(nproc)

# Download a model (e.g. Llama 3.1 8B Q4_K_M):
# huggingface-cli download bartowski/Meta-Llama-3.1-8B-Instruct-GGUF

# Run on AMD GPU:
GGML_VK_VISIBLE_DEVICES=0 ./build/bin/llama-cli \
    -m Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf \
    -p "The Linux graphics stack consists of" \
    --gpu-layers 35

# Check Vulkan device selection:
GGML_VK_VERBOSE=1 ./build/bin/llama-cli -m model.gguf -p "test" -n 1
```

On an RX 7900 XTX (RDNA3), llama.cpp achieves ~45–70 tokens/second on Llama 3.1 8B Q4_K_M via Vulkan cooperative matrices.

### Stable Diffusion via Vulkan

[stable-diffusion.cpp](https://github.com/leejet/stable-diffusion.cpp) provides Vulkan acceleration for Stable Diffusion inference:

```bash
git clone https://github.com/leejet/stable-diffusion.cpp
cd stable-diffusion.cpp
cmake -B build -DSD_VULKAN=ON
cmake --build build -j$(nproc) --target sd

./build/bin/sd --diffusion-model sd3-medium.safetensors \
    --vae vae.safetensors \
    -p "A scenic mountain landscape" \
    --steps 20 --output output.png
```

### VkFFT

[VkFFT](https://github.com/DTolm/VkFFT) is a GPU Fast Fourier Transform library for Vulkan, supporting AMD, NVIDIA, and Intel. While not using cooperative matrices directly, it benefits from the same compute shader infrastructure and can complement cooperative matrix workloads (e.g. frequency-domain convolutions).

```bash
# Build and benchmark VkFFT on AMD:
git clone https://github.com/DTolm/VkFFT
cd VkFFT
cmake -B build -DVKFFT_BACKEND=0  # 0 = Vulkan
cmake --build build
./build/VkFFT_TestSuite -d 0
```

### tinygrad Vulkan Backend

[tinygrad](https://github.com/tinygrad/tinygrad) is a minimalist ML framework with a Vulkan compute backend (`tinygrad/runtime/ops_gpu.py`). When cooperative matrix support is available, tinygrad emits optimised GEMM shaders.

```bash
GPU=VULKAN python3 -c "
from tinygrad import Tensor
a = Tensor.randn(4096, 4096).realize()
b = Tensor.randn(4096, 4096).realize()
c = (a @ b).realize()
print(c.numpy()[0, :5])
"
```

### KompVis / Vulkan Upscaling

Cooperative matrices enable real-time neural network-based image upscaling in graphics applications. AMD's FidelityFX Super Resolution 3 (FSR3) uses temporal upscaling in a Vulkan compute shader; a cooperative-matrix-accelerated neural network (DLSS-equivalent) is a natural extension point.

---

## Performance Tuning

### Register Pressure and Occupancy

Cooperative matrix tiles consume many VGPRs. On RDNA3, `v_wmma_f32_16x16x16_f16` uses 8 src VGPRs for A, 8 for B, and 8 for C (all fp32). With 256 VGPRs per CU wavefront slot and 512 VGPRs total per CU, a shader using 64 VGPRs for one 16×16×16 tile leaves room for only 4 concurrent wavefronts (vs the ideal 8–16 for latency hiding).

**Tuning approach**: balance tile size vs VGPR pressure. Two 16×16×16 tiles (more ILP) may stall for latency if occupancy drops.

```bash
# RADV shader statistics (shows VGPR count):
RADV_DEBUG=shaderinfo ./llama_vulkan 2>&1 | grep -i "vgpr\|wavefront"
```

### L1 Cache Tiling

RDNA3 L1 cache: 32 KB per shader array. For optimal throughput:
- A tile: 16×16 fp16 = 512 bytes
- B tile: 16×16 fp16 = 512 bytes
- C accumulator: 16×16 fp32 = 1024 bytes

Total: 2 KB per tile — well within L1. Shared memory (LDS) can stage tiles from global memory to further improve bandwidth:

```glsl
shared float16_t shmA[16][16];
shared float16_t shmB[16][16];

/* Cooperative load from global to shared memory, then from shared to coop mat */
```

### Subgroup Size

```glsl
/* Require exact subgroup size 32 (RDNA3 wavefront size): */
#extension GL_EXT_subgroup_size_control : require
layout(local_size_x = 32, local_size_y = 1) in;
layout(subgroup_size_control = SUBGROUP_SIZE_CONTROL_FULL_EXT) in;
```

RDNA3 can run in wave32 or wave64 mode. Cooperative matrices require wave32 for WMMA on RDNA3; the GLSL `subgroup_size_control` extension ensures this.

### Peak TFLOPS Calculation

For RX 7900 XTX (96 CUs, RDNA3):

```
v_wmma_f32_16x16x16_f16: 16×16×16 × 2 = 8192 FLOPs per instruction
Issue rate: 1 instruction per 4 clocks (RDNA3 WMMA throughput)
CU count: 96
Clock: 2.5 GHz

Peak = 96 × (8192 / 4) × 2.5e9 = 96 × 2048 × 2.5e9 ≈ 492 TFLOPS

(Real-world GEMM: 60–80% efficiency → ~300–390 TFLOPS achieved)
```

Verify with:

```bash
# AMD GPU Profiler (RGP) via rgpviewer — shows WMMA utilisation
# Or use radeon-profile for clock and occupancy monitoring

# VK_AMD_shader_core_properties:
vulkaninfo --json 2>/dev/null | jq '.[] | .properties |
    .VkPhysicalDeviceShaderCorePropertiesAMD |
    {shaderEngines, computeUnitsPerShaderArray, shaderArraysPerEngineCount}'
```

---

## Roadmap

### Near-term (6–12 months)

- **`VK_NV_cooperative_matrix2` broader adoption in RADV**: RADV landed partial support for `VK_NV_cooperative_matrix2` in Mesa 25.2 (gated behind the `radv_cooperative_matrix2_nv` DriConf option), initially targeting FidelityFX Super Resolution 4 and VKD3D-Proton; full production enablement and expanded `CooperativeMatrixConversionsNV` coverage is expected to follow. [Source](https://www.phoronix.com/news/RADV-NV-Coperative-Matrix2)
- **`VK_QCOM_cooperative_matrix_conversion`**: Qualcomm's vendor extension adds optimised data-type conversion instructions that let shaders move data between invocation and subgroup scope without explicit shared-memory round-trips; adoption by other IHVs or promotion to KHR is under discussion. [Source](https://github.khronos.org/Vulkan-Site/features/latest/features/proposals/VK_QCOM_cooperative_matrix_conversion.html)
- **RDNA4 (GFX12) cooperative matrix stabilisation in RADV**: `VK_KHR_cooperative_matrix` was merged for RDNA4 ahead of the RX 9000 series launch; expect ongoing tuning of tile sizes and VGPR scheduling for the new WMMA ISA variant on GFX12. [Source](https://www.phoronix.com/news/RADV-Lands-RDNA4-Coop-Matrix)
- **ANV restricts cooperative matrix to hardware-backed devices**: Mesa 25.2 tightened `VK_KHR_cooperative_matrix` exposure on ANV so it is only advertised when genuine XMX/DPAS hardware instructions are present, removing the earlier software-emulation path; further Intel Xe2 (Battlemage) tuning is expected. [Source](https://docs.mesa3d.org/relnotes/25.2.0.html)
- **Lavapipe `VK_NV_cooperative_matrix2` per-element operations**: Mesa 26.1 added per-element operation support in lavapipe (CPU Vulkan), providing a software reference implementation for testing cooperative matrix shaders without GPU hardware. [Source](https://docs.mesa3d.org/relnotes/26.1.0.html)

### Medium-term (1–3 years)

- **`VK_NV_cooperative_matrix2` KHR promotion**: The NVIDIA v2 extension introduces workgroup-scope matrices with compiler-managed shared-memory staging, flexible (non-power-of-two) tile sizes, and per-element callbacks — features missing from the base KHR extension. A Khronos working-group proposal to promote these capabilities to `VK_KHR_cooperative_matrix2` is anticipated once multi-vendor implementation experience is gathered. [Source](https://docs.vulkan.org/features/latest/features/proposals/VK_NV_cooperative_matrix2.html)
- **AMD Instinct MI400 / CDNA 5 Vulkan support**: AMD's MI450X and MI430X data-centre accelerators (targeted for second half 2026) implement CDNA Next architecture with new matrix-core variants; AMDVLK/RADV will need ISA-level mapping of the new MFMA/WMMA instructions to `VK_KHR_cooperative_matrix` property queries. Note: needs verification for exact ISA details. [Source](https://wccftech.com/amd-to-battle-nvidia-ai-dominance-instinct-mi400-accelerators-2026-mi500-2027/)
- **`cl_khr_cooperative_matrix` OpenCL 3.0 finalisation**: The parallel OpenCL path through rusticl (RADV) and Intel NEO shares the same GFX back-end as Vulkan cooperative matrices; spec finalisation and CTS coverage will unlock portability between Vulkan and OpenCL compute on Linux without duplicating matrix kernels. Note: needs verification on current draft status.
- **SPIR-V cooperative matrix type extensions for FP8/INT4**: As LLM inference shifts to sub-byte quantisation (FP8, INT4, MX formats), SPIR-V and Vulkan will need new `VkComponentTypeKHR` enumerants and cooperative matrix property entries; AMD RDNA4 and Intel Xe2 already expose limited FP8 WMMA/DPAS paths at the ISA level. Note: needs verification on exact Khronos proposal status.

### Long-term

- **Graph-compiler integration (torch-mlir / IREE)**: ML compilers that target Vulkan (IREE's HAL Vulkan backend, torch-mlir) will increasingly drive the tile-size selection and layout transformation above the cooperative matrix level; long-term the expectation is that hand-written GLSL GEMM shaders give way to compiler-generated SPIR-V with auto-tuned tile strategies, similar to CUDA's cuBLAS vs. CUTLASS split.
- **Hardware-managed matrix prefetch and caching**: Future GPU architectures may expose dedicated matrix-tile caches (analogous to NVIDIA H100's L2 sector cache for WGMMA), removing the need for shader-managed LDS staging entirely; this would change the cooperative matrix programming model from explicit tile management to a higher-level "load → compute → store" without shared-memory boilerplate.
- **Unified ML inference API convergence**: Competitive pressure from DirectML (Windows) and Metal Performance Shaders (macOS) may drive Khronos toward a higher-level `VK_KHR_ml_inference` extension that sits above cooperative matrices, accepting operator graphs (ONNX-style) and compiling them to tiled GEMM internally — reducing the gap between Vulkan's low-level tile model and framework-level operator dispatch.

---

## Integrations

- **Ch24 (Vulkan API)** — cooperative matrices are a Vulkan compute extension; the pipeline model (compute pipelines, SSBO storage, push constants) is established in Ch24
- **Ch25 (GPU Compute on Linux)** — cooperative matrix acceleration is a specialised form of GPU compute; shares the Vulkan compute queue infrastructure
- **Ch108 (ROCm / HIP)** — `rocBLAS` on AMD uses MFMA instructions (CDNA) and WMMA (RDNA3) — the underlying hardware is the same as Vulkan cooperative matrices on RDNA3
- **Ch124 (OpenCL on Linux)** — `cl_khr_cooperative_matrix` (OpenCL 3.0 draft) is a parallel path to the same hardware; RADV's rusticl and Intel NEO implement this
- **Ch127 (Mesh Shaders and VRS)** — geometry (mesh) + ML (cooperative matrix) in the same frame: e.g. ray-traced GI (Ch135) followed by cooperative-matrix denoising
- **Ch133 (Vulkan Compute Queues)** — GEMM workloads dispatch on async compute queues alongside graphics; timeline semaphores synchronise ML inference with render pipeline
- **Ch135 (Vulkan Ray Tracing)** — neural denoising (OIDN-style): 1 spp RT → cooperative matrix network → denoised frame; same pipeline

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
