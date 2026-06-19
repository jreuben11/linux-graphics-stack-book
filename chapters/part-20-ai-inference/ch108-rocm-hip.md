# Chapter 108: ROCm and HIP — AMD's GPU Compute Stack

**Target audiences**: ML engineers deploying workloads on AMD GPUs, GPGPU developers porting from CUDA, systems engineers building ROCm-based infrastructure on Linux.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [ROCm Architecture Overview](#2-rocm-architecture-overview)
3. [KFD: Kernel Fusion Driver](#3-kfd-kernel-fusion-driver)
4. [HIP Programming Model](#4-hip-programming-model)
5. [AMDGPU ISA and Code Objects](#5-amdgpu-isa-and-code-objects)
6. [ROCm Libraries](#6-rocm-libraries)
7. [ROCm Compiler: LLVM/Clang with AMDGPU Backend](#7-rocm-compiler-llvmclang-with-amdgpu-backend)
8. [PyTorch on ROCm](#8-pytorch-on-rocm)
9. [OpenCL on ROCm](#9-opencl-on-rocm)
10. [Debugging and Profiling ROCm Workloads](#10-debugging-and-profiling-rocm-workloads)
11. [Integrations](#11-integrations)

---

## 1. Introduction

ROCm (Radeon Open Compute platform) is AMD's open-source GPU compute stack for Linux, first released in 2016 and now in its seventh major generation. Where NVIDIA's CUDA ecosystem is proprietary and tightly coupled to NVIDIA hardware, ROCm is a Linux-native alternative composed almost entirely of open-source software under the MIT, Apache 2.0, and GPL-2.0 licenses. As of ROCm 6.x (2024) and ROCm 7.x (2025–2026), the platform officially supports PyTorch, JAX, TensorFlow, and Triton on AMD GPUs, making it a credible CUDA alternative for ML training and inference workloads. [Source](https://rocm.docs.amd.com/en/latest/)

The primary programming model is **HIP** (Heterogeneous-Compute Interface for Portability), a C++ runtime API and kernel language intentionally designed to be source-portable with CUDA. A single HIP source file can compile to both AMD (via the ROCm backend) and NVIDIA (via the CUDA backend), enabling gradual migration away from CUDA-only code.

**Supported hardware as of ROCm 7.2 (January 2026)**:
- **RDNA2** (gfx1030/1031/1032/1034/1035/1036): Radeon RX 6000 series, including RX 6700 XT, RX 6800 XT, RX 6900 XT
- **RDNA3** (gfx1100/1101/1102): Radeon RX 7000 series, including RX 7900 XTX (24 GB VRAM), RX 7700, RX 7600
- **RDNA4** (gfx1200/1201): Radeon RX 9060 XT, Radeon AI PRO R9600D (added ROCm 7.2)
- **CDNA1** (gfx908): Instinct MI100
- **CDNA2** (gfx90a): Instinct MI200 series (MI250, MI250X)
- **CDNA3** (gfx942): Instinct MI300X (192 GB HBM3, the primary datacenter target)

[Source](https://rocm.docs.amd.com/en/docs-7.2.0/about/release-notes.html)

The distinction between RDNA (consumer/workstation graphics) and CDNA (datacenter compute) architectures is fundamental: CDNA sacrifices display output and ray-tracing hardware for higher VRAM capacity, wider SIMD, matrix FMA units, and Infinity Fabric high-bandwidth interconnects. For ML training at scale, CDNA accelerators like the MI300X are the target; for single-GPU inference and hobbyist ML, RDNA3 and RDNA4 consumer GPUs are the pragmatic choice.

---

## 2. ROCm Architecture Overview

The ROCm stack is organized in layers from Linux kernel to application-level frameworks:

```
┌──────────────────────────────────────────────────────────┐
│  Application: PyTorch, JAX, TensorFlow, llama.cpp        │
├──────────────────────────────────────────────────────────┤
│  ML Libraries: MIOpen, rocBLAS, hipBLASLt, RCCL          │
├──────────────────────────────────────────────────────────┤
│  HIP Runtime API (libamdhip64.so)                        │
├──────────────────────────────────────────────────────────┤
│  ROCclr / CLR — ROCm Common Language Runtime             │
│  (device memory management, command queue abstraction)   │
├──────────────────────────────────────────────────────────┤
│  HSA Runtime (libhsa-runtime64.so, ROCR-Runtime)         │
│  (HSA signal/queue management, agent enumeration)        │
├──────────────────────────────────────────────────────────┤
│  libhsakmt.so — userspace KFD interface                  │
├──────────────────────────────────────────────────────────┤
│  /dev/kfd  ←  amdkfd kernel module                       │
│  /dev/dri/renderD*  ←  amdgpu DRM driver                 │
├──────────────────────────────────────────────────────────┤
│  Linux kernel: amdgpu.ko (DRM + KFD dual personality)    │
└──────────────────────────────────────────────────────────┘
```

### 2.1 The Dual Personality of `amdgpu.ko`

The single kernel module `amdgpu.ko` serves two conceptually separate roles:

1. **amdgpu DRM driver**: Implements KMS (Kernel Mode Setting) for display, manages GEM buffer objects, and exposes `/dev/dri/card0` and `/dev/dri/renderD128` for graphics clients (Mesa, Vulkan, OpenGL). This is covered in depth in Ch5.

2. **amdkfd compute extension**: The KFD (Kernel Fusion Driver) component exposes `/dev/kfd` for compute-only access. KFD manages HSA compute queues, process isolation via PASID, and GPU virtual memory for compute workloads. KFD is compiled as a submodule within `drivers/gpu/drm/amd/amdkfd/` and initialized alongside the DRM driver. [Source](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/amd/amdkfd)

The practical consequence: a ROCm compute workload communicates through `/dev/kfd` (for queue management and memory allocation) and `/dev/dri/renderD128` (for DMA-BUF operations and GEM buffer handles). The two paths share the underlying GPU MMU context but present distinct IOCTL interfaces.

### 2.2 `/dev/kfd` vs `/dev/dri/renderD*`

| Property | `/dev/kfd` | `/dev/dri/renderD*` |
|---|---|---|
| Interface | KFD IOCTLs (HSA-spec queue management) | DRM IOCTLs (GEM, KMS) |
| Primary consumers | ROCm runtime, OpenCL | Mesa, Vulkan, graphics apps |
| Queue type | SDMA + compute queues (AQL ring) | CS (command stream) via `drm_sched` |
| Memory model | HSA SVM, per-process PASID | GEM BOs, DMA-BUF export |
| Permissions | Group `render` (`/dev/kfd` mode 0660) | Group `render` (`renderD*`) |

For a ROCm container deployment, both device nodes must be passed: `--device /dev/kfd --device /dev/dri/renderD128`.

---

## 3. KFD: Kernel Fusion Driver

### 3.1 Overview

The KFD implements AMD's support for the HSA (Heterogeneous System Architecture) specification, providing compute queue management, process isolation, and memory management for GPU compute workloads without involving the display subsystem.

The key KFD source files in the Linux kernel are:

- `drivers/gpu/drm/amd/amdkfd/kfd_chardev.c` — character device `/dev/kfd` and IOCTL dispatch
- `kfd_device_queue_manager.c` — compute queue (MQD) lifecycle management
- `kfd_svm.c` — Shared Virtual Memory implementation using Linux HMM
- `kfd_process.c` — per-process GPU context (PASID binding)
- `kfd_doorbell.c` — doorbell register mapping for queue submission

[Source](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/amd/amdkfd)

### 3.2 PASID: Process Address Space ID

Each process that opens `/dev/kfd` receives a **PASID** (Process Address Space ID), a hardware concept from PCIe ATS (Address Translation Services) and IOMMU. The PASID enables the GPU MMU to distinguish between different process virtual address spaces, providing isolation analogous to CPU process isolation.

When `amdkfd` binds a process via `kfd_process_create()`, it allocates a PASID and programs it into the GPU's VMID (Virtual Memory ID) tables. The GPU MMU then uses PASID→page-table mappings to translate GPU virtual addresses, allowing multiple processes to share the GPU without accessing each other's memory.

### 3.3 KFD IOCTLs

The KFD exposes IOCTLs defined in `include/uapi/linux/kfd_ioctl.h` under the `AMDKFD_IOC_*` prefix (using the `'K'` ioctl base). The three most important for compute workloads are: [Source](https://github.com/torvalds/linux/blob/master/include/uapi/linux/kfd_ioctl.h)

```c
#define AMDKFD_IOCTL_BASE 'K'
#define AMDKFD_IOWR(nr, type) _IOWR(AMDKFD_IOCTL_BASE, nr, type)

/* nr=0x02: Queue creation — allocates an AQL ring and maps doorbells */
#define AMDKFD_IOC_CREATE_QUEUE \
        AMDKFD_IOWR(0x02, struct kfd_ioctl_create_queue_args)

/* nr=0x16: Memory allocation — GPU VRAM or system memory accessible from GPU */
#define AMDKFD_IOC_ALLOC_MEMORY_OF_GPU \
        AMDKFD_IOWR(0x16, struct kfd_ioctl_alloc_memory_of_gpu_args)

/* nr=0x18: Map allocated memory into the GPU virtual address space */
#define AMDKFD_IOC_MAP_MEMORY_TO_GPU \
        AMDKFD_IOWR(0x18, struct kfd_ioctl_map_memory_to_gpu_args)
```

The `kfd_ioctl_alloc_memory_of_gpu_args` struct includes a `flags` field combining:

- `KFD_IOC_ALLOC_MEM_FLAGS_VRAM` — GPU-local VRAM (fastest for GPU access, not directly CPU-accessible without staging)
- `KFD_IOC_ALLOC_MEM_FLAGS_GTT` — system memory mapped through the GART, GPU-accessible via PCIe (lower bandwidth, useful for CPU↔GPU transfers)
- `KFD_IOC_ALLOC_MEM_FLAGS_USERPTR` — CPU-side pinned memory exposed to the GPU (zero-copy for unified memory scenarios)
- `KFD_IOC_ALLOC_MEM_FLAGS_DOORBELL` — doorbell page for queue submission
- `KFD_IOC_ALLOC_MEM_FLAGS_COHERENT` — forces cache-coherent access (required for APU SVM)

### 3.4 HSA Queue Submission (AQL)

The HSA specification defines an **AQL (Architected Queuing Language)** ring buffer for submitting work to the GPU. The user process writes 64-byte AQL packets directly into a queue in system memory; the GPU's command processor polls the ring tail via a doorbell register.

```c
/* Dispatch packet layout — user writes this to the AQL ring */
typedef struct hsa_kernel_dispatch_packet_s {
    uint16_t header;              /* packet type, fence scopes */
    uint16_t setup;               /* number of dimensions */
    uint16_t workgroup_size_x;
    uint16_t workgroup_size_y;
    uint16_t workgroup_size_z;
    uint16_t reserved0;
    uint32_t grid_size_x;
    uint32_t grid_size_y;
    uint32_t grid_size_z;
    uint32_t private_segment_size; /* scratch memory per workitem */
    uint32_t group_segment_size;   /* LDS per workgroup */
    uint64_t kernel_object;        /* GPU VA of kernel descriptor */
    uint64_t kernarg_address;      /* GPU VA of kernel arguments */
    uint64_t reserved2;
    hsa_signal_t completion_signal; /* HSA signal for completion */
} hsa_kernel_dispatch_packet_t;
```

[Source: HSA Runtime Specification 1.2, `hsa.h` in ROCR-Runtime]

### 3.5 libhsakmt: Userspace KFD Interface

`libhsakmt.so` (the HSA kernel mode thunk library) is the thin userspace layer that wraps KFD IOCTLs, analogous to `libdrm` for DRM. The HSA runtime (`libhsa-runtime64.so`) calls `libhsakmt` for queue creation, memory allocation, and topology discovery.

### 3.6 Shared Virtual Memory (SVM)

On APUs (CPU+GPU integrated, such as the Ryzen AI series), ROCm supports **SVM** (Shared Virtual Memory): a unified address space in which the same virtual address is valid on both CPU and GPU. This eliminates explicit `hipMemcpy` calls for data that both processor types must access.

The implementation in `kfd_svm.c` uses Linux HMM (Heterogeneous Memory Management) — specifically `hmm_range_fault()` to pin CPU pages and migrate them to GPU VRAM on demand. GPU page faults trigger the `amdgpu_mn` migration notifier to migrate pages back to system memory, enabling transparent demand-paging. [Source](https://lwn.net/Articles/841990/)

---

## 4. HIP Programming Model

### 4.1 Overview

HIP (Heterogeneous-Compute Interface for Portability) is AMD's primary GPU programming model. Its design mirrors CUDA: `__global__` kernel functions, chevron launch syntax (`<<<grid, block>>>`), and a runtime API that closely follows `cudaXxx` naming with `hipXxx` equivalents.

A minimal HIP kernel:

```cpp
// vadd.hip — vector addition kernel
#include <hip/hip_runtime.h>

__global__ void vadd(const float* a, const float* b, float* c, int n) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < n) {
        c[idx] = a[idx] + b[idx];
    }
}

int main() {
    const int N = 1 << 20;  // 1M elements
    const size_t bytes = N * sizeof(float);

    float *d_a, *d_b, *d_c;
    hipMalloc(&d_a, bytes);
    hipMalloc(&d_b, bytes);
    hipMalloc(&d_c, bytes);

    /* fill d_a, d_b from host ... */

    dim3 block(256);
    dim3 grid((N + block.x - 1) / block.x);

    // Triple-chevron launch (HIP language extension)
    vadd<<<grid, block>>>(d_a, d_b, d_c, N);

    // Alternatively, hipLaunchKernelGGL (portable, no language extension)
    // hipLaunchKernelGGL(vadd, grid, block, 0, 0, d_a, d_b, d_c, N);

    hipDeviceSynchronize();

    hipFree(d_a);
    hipFree(d_b);
    hipFree(d_c);
    return 0;
}
```

[Source](https://github.com/ROCm/HIP)

### 4.2 `hipcc` Compiler Driver

`hipcc` is the HIP compiler driver that wraps `amdclang++` (Clang with AMD GPU extensions). It selects the correct target triple and links the HIP runtime automatically:

```bash
# Compile for RDNA3 RX 7900 XTX
hipcc -O3 --offload-arch=gfx1100 vadd.hip -o vadd

# Multi-arch fat binary (RDNA2 + RDNA3 + MI300X)
hipcc -O3 --offload-arch=gfx1030 --offload-arch=gfx1100 \
         --offload-arch=gfx942   vadd.hip -o vadd_fat

# Enable binary compression (ROCm 6.4+)
hipcc -O3 --offload-arch=gfx1100 --offload-compress vadd.hip -o vadd
```

Internally, `hipcc` invokes `amdclang++` with the target triple `amdgcn-amd-amdhsa` and the device libraries linked via `-mlink-bitcode-file`. The host code compiles to native x86-64; the GPU code compiles to an ELF code object embedded in the host binary.

### 4.3 Runtime API Reference

Core HIP runtime functions (all synchronous unless noted):

```cpp
/* Device management */
hipError_t hipGetDeviceCount(int* count);
hipError_t hipSetDevice(int deviceId);
hipError_t hipGetDeviceProperties(hipDeviceProp_t* prop, int deviceId);

/* Memory management */
hipError_t hipMalloc(void** ptr, size_t size);
hipError_t hipFree(void* ptr);
hipError_t hipMallocManaged(void** ptr, size_t size);  // unified memory
hipError_t hipMemcpy(void* dst, const void* src, size_t size,
                     hipMemcpyKind kind);               // synchronous
hipError_t hipMemcpyAsync(void* dst, const void* src, size_t size,
                           hipMemcpyKind kind, hipStream_t stream);

/* Streams (FIFO command queues on the GPU) */
hipError_t hipStreamCreate(hipStream_t* stream);
hipError_t hipStreamDestroy(hipStream_t stream);
hipError_t hipStreamSynchronize(hipStream_t stream);

/* Events (timestamps, inter-stream synchronization) */
hipError_t hipEventCreate(hipEvent_t* event);
hipError_t hipEventRecord(hipEvent_t event, hipStream_t stream);
hipError_t hipEventElapsedTime(float* ms, hipEvent_t start, hipEvent_t stop);
```

### 4.4 CUDA Portability: hipify

The `hipify-perl` and `hipify-clang` tools automate CUDA→HIP source translation:

```bash
# Perl-based regex translation (fast, handles most common patterns)
hipify-perl cuda_kernel.cu -o hip_kernel.hip

# Clang-based AST translation (accurate for complex templates)
hipify-clang cuda_kernel.cu --cuda-path=/usr/local/cuda \
             -- -std=c++17
```

`hipify-perl` performs textual substitutions: `cudaMalloc` → `hipMalloc`, `__CUDA_ARCH__` → `__HIP_DEVICE_COMPILE__`, `cublasHandle_t` → `rocblas_handle`, etc. `hipify-clang` parses the CUDA AST for higher accuracy with complex template metaprogramming.

**HIP on CUDA**: The same HIP source can compile to NVIDIA hardware when `hipcc` is invoked with `--platform=nvidia`. In this mode, `hipcc` becomes a thin wrapper over `nvcc`, and the HIP header macros expand to their CUDA equivalents. This bidirectional portability is the key differentiator of HIP over pure ROCm-specific code.

### 4.5 Managed Memory and Unified Memory

`hipMallocManaged` allocates memory visible to both CPU and GPU. On discrete GPUs, this triggers page migration via the GPU IOMMU and HMM. On APUs with Shared Virtual Memory enabled, it maps the same physical pages directly:

```cpp
float* data;
hipMallocManaged(&data, N * sizeof(float));

// CPU can write...
for (int i = 0; i < N; i++) data[i] = (float)i;

// GPU kernel can read the same pointer without explicit hipMemcpy
vadd<<<grid, block>>>(data, data, data, N);
hipDeviceSynchronize();

// CPU can read back without copy
printf("data[0] = %f\n", data[0]);
hipFree(data);
```

On MI300X with its unified HBM3 memory pool (192 GB shared between CPU and GPU compute dies), managed memory is particularly efficient as data never crosses a PCIe link.

---

## 5. AMDGPU ISA and Code Objects

### 5.1 ISA Architecture Families

AMD GPUs relevant to ROCm belong to two ISA lineages:

**GCN (Graphics Core Next, gfx6–gfx9)**: The original AMD GPU compute architecture. CDNA descended from GCN, but GCN itself is not CDNA — CDNA begins at gfx908 (Instinct MI100). The GCN characteristics relevant to ROCm:
- 64-wide SIMD (wavefront of 64 threads)
- 4 SIMD units per Compute Unit (CU), each 16-wide, executing in lock-step to form a 64-wide wave
- LDS (Local Data Share): 64 KB per CU, shared among all wavefronts
- VGPR file: 256 registers × 64 threads per SIMD = 64 KB per SIMD unit
- Memory access: VMEM instructions (global/scratch/LDS loads/stores), SMEM for constant cache reads

**RDNA (gfx10–gfx12)**: Introduced in 2019, consumer-focused. Key differences from GCN:
- **wave32 default**: 32-wide SIMD; waves are 32 threads instead of 64 (halves VGPR pressure)
- **wave64 optional**: Can run 64-wide for compatibility, but splits into two wave32 sub-wavefronts
- **WGP (Work Group Processor)**: RDNA groups two CUs into a WGP sharing a front-end. In WGP mode, LDS is 128 KB (two 64 KB banks shared across the WGP). In CU mode (per-CU), each half has 64 KB
- Dual-issue scalar and vector instructions in some units
- Ray-tracing BVH traversal hardware (RDNA2+)

**CDNA (gfx908, gfx90a, gfx942)**: Datacenter-optimized:
- Returns to 64-wide SIMD (wave64)
- **MFMA (Matrix Fused Multiply-Accumulate)**: Hardware matrix FMA for FP64/FP32/FP16/BF16/INT8, the analogue of NVIDIA Tensor Cores
- **AGPR (Accumulation Registers)**: Additional 512×64 AGPR file for MFMA accumulation, separate from VGPR
- Infinity Fabric: high-bandwidth GPU-to-GPU interconnect (up to 900 GB/s on MI300X inter-die)
- No display output hardware, no ray-tracing units

### 5.2 ISA Instruction Classes

The AMDGPU ISA organizes instructions by execution unit and memory type:

| Class | Description | Example |
|---|---|---|
| `VALU` | Vector ALU — executes once per SIMD lane | `v_add_f32 v0, v1, v2` |
| `SALU` | Scalar ALU — executes once per wavefront | `s_add_u32 s0, s1, s2` |
| `VMEM` | Vector memory — global/scratch/LDS loads per lane | `global_load_dword v0, v1, off` |
| `SMEM` | Scalar memory — constant/descriptor cache reads | `s_load_dword s0, s2, 0x0` |
| `DS` | Data share — LDS load/store, atomic ops | `ds_read_b32 v0, v1` |
| `FLAT` | Flat addressing — unified global/LDS/scratch | `flat_load_dword v0, v[2:3]` |
| `MFMA` | Matrix FMA — CDNA only | `v_mfma_f32_32x32x8_f16 a[0:15], v[0:3], v[4:7], a[0:15]` |

[Source](https://llvm.org/docs/AMDGPUUsage.html)

### 5.3 Code Object Format

ROCm uses ELF-based GPU code objects (Code Object v4/v5/v6 as of ROCm 6.x). The format is defined by the AMDGPU ABI:

- `.text` section: raw AMDGPU ISA binary
- `.rodata` section: kernel descriptor table (one 64-byte `amd_kernel_code_t` per kernel)
- `.note` section: ELF note with GPU target ID (e.g., `amdgcn-amd-amdhsa--gfx1100`)
- `.symtab`: kernel symbol table with offsets into `.text`

**Kernel Descriptor** (64 bytes, Code Object v3+): encodes the GPU register programming needed to launch a kernel. This is the compact descriptor read by the CP (Command Processor) at dispatch time, distinct from the older `amd_kernel_code_t` (256-byte, Code Object v2):

```c
/* Code Object v3+ kernel descriptor (64 bytes).
 * Located in the ELF .rodata section; one entry per kernel symbol.
 * See: LLVM AMDGPUUsage.html §"Kernel Descriptor" */
struct amdgpu_kernel_descriptor_v3 {
    uint32_t group_segment_fixed_size;    /* LDS bytes required by kernel */
    uint32_t private_segment_fixed_size;  /* Scratch bytes per work-item */
    uint32_t kernel_code_properties;      /* Wave32/64, SGPR preload flags */
    uint32_t kernarg_size;                /* Kernel argument buffer size */
    uint8_t  reserved0[4];
    int64_t  kernel_code_entry_byte_offset; /* Offset from descriptor to .text */
    uint8_t  reserved1[20];
    uint32_t compute_pgm_rsrc3;           /* gfx10+ shared VGPR, inst prefetch */
    uint32_t compute_pgm_rsrc1;           /* VGPR alloc, SGPR alloc, FP mode */
    uint32_t compute_pgm_rsrc2;           /* LDS/SH_MEM sizes, user SGPR count */
    uint32_t reserved2;
};
```

`COMPUTE_PGM_RSRC1` packs the VGPR count (bits [5:0]), SGPR count (bits [9:6]), floating-point mode (bits [13:12]), and exception handling flags. The hardware reads this register during kernel dispatch to configure the compute unit before the first instruction executes.

### 5.4 Inspecting GPU Binaries

```bash
# Disassemble GPU code object
llvm-objdump --disassemble --mcpu=gfx1100 vadd.co

# Extract embedded code object from host binary
llvm-objcopy --dump-section=.hip_fatbin=fatbin.bin vadd
llvm-objcopy --dump-section=.text=kernel.co fatbin.bin
llvm-objdump --disassemble --mcpu=gfx1100 kernel.co

# Print kernel resource usage (VGPR/SGPR/LDS/occupancy)
hipcc -O3 --offload-arch=gfx1100 -Rpass-analyze=kernel-resource-usage \
      vadd.hip -o vadd 2>&1 | grep "kernel resource"
```

---

## 6. ROCm Libraries

ROCm ships a comprehensive suite of GPU-accelerated libraries covering linear algebra, FFT, deep learning primitives, and communications. As of ROCm 6.4/7.x, these are organized under the `rocm-libraries` umbrella repository.

### 6.1 rocBLAS

rocBLAS is AMD's BLAS (Basic Linear Algebra Subprograms) implementation on ROCm. It provides Level 1/2/3 BLAS routines, with the most ML-relevant being:

```cpp
#include <rocblas/rocblas.h>

rocblas_handle handle;
rocblas_create_handle(&handle);

/* Single-precision GEMM: C = alpha * A * B + beta * C */
rocblas_sgemm(handle,
              rocblas_operation_none,   /* transa */
              rocblas_operation_none,   /* transb */
              M, N, K,
              &alpha,
              d_A, lda,
              d_B, ldb,
              &beta,
              d_C, ldc);

/* Half-precision GEMM (FP16, critical for ML inference) */
rocblas_hgemm(handle, rocblas_operation_none, rocblas_operation_none,
              M, N, K, &alpha_h, d_A_h, lda, d_B_h, ldb,
              &beta_h, d_C_h, ldc);

rocblas_destroy_handle(handle);
```

[Source: rocBLAS 4.4+ — ROCm 6.4]

### 6.2 hipBLASLt

hipBLASLt is AMD's implementation of the "lightweight BLAS" API (analogous to cublasLt), providing user-tunable GEMM with explicit algorithm selection. This is the preferred backend for ML frameworks needing mixed-precision and custom epilogue operations:

```cpp
#include <hipblaslt/hipblaslt.h>

hipblasLtHandle_t handle;
hipblasLtCreate(&handle);

hipblasLtMatmulDesc_t matmul_desc;
hipblasLtMatmulDescCreate(&matmul_desc, HIPBLASLT_COMPUTE_F32,
                           HIP_R_16F);  /* FP16 inputs, FP32 accumulate */

/* User-tunable: enumerate and select from available algorithms */
hipblasLtMatmulPreference_t pref;
hipblasLtMatmulPreferenceCreate(&pref);

hipblaslt_ext::GemmInstance gemm(handle);
gemm.setProblem(M, N, K, /* batch */ 1);
/* ... run heuristic, select best algo, execute ... */
```

PyTorch uses hipBLASLt for `torch.matmul` on ROCm when enabled (default on ROCm 6.3+ for gfx908+). The `TunableOp` feature evaluates thousands of algorithm variants at first execution and caches the winner, achieving up to 7.2× throughput improvement for LLM GEMM workloads on MI300X. [Source](https://rocm.docs.amd.com/en/docs-6.1.2/how-to/tuning-guides/mi300x/workload.html)

### 6.3 MIOpen

MIOpen is AMD's deep learning primitives library — the ROCm equivalent of cuDNN. It implements:

- **Convolution** (forward, backward data, backward weights): using FFT, Winograd, direct, and implicit GEMM algorithms; auto-tuning selects the fastest for each problem shape
- **Pooling** (max, average, adaptive)
- **Batch normalization** (training and inference modes, supports BF16)
- **Activation functions**: ReLU, GELU, SiLU (Swish), Tanh, Sigmoid
- **Recurrent layers**: LSTM, GRU via CK (Composable Kernel) backends
- **Attention**: fused attention via AOTriton (integrated ROCm 7.0+)

```cpp
#include <miopen/miopen.h>

miopenHandle_t handle;
miopenCreate(&handle);

miopenTensorDescriptor_t input_desc, filter_desc, output_desc;
miopenCreateTensorDescriptor(&input_desc);
miopenSet4dTensorDescriptor(input_desc, miopenFloat,
                             N, C, H, W);

/* MIOpen auto-finds the fastest convolution algorithm */
miopenConvAlgoPerf_t perf_results[4];
int returned_algo_count;
miopenFindConvolutionForwardAlgorithm(
    handle, input_desc, d_input, filter_desc, d_filter,
    conv_desc, output_desc, d_output,
    4, &returned_algo_count, perf_results,
    workspace, workspace_size, /* exhaustive_search */ 1);

miopenConvolutionForward(
    handle, &alpha,
    input_desc, d_input, filter_desc, d_filter,
    conv_desc, perf_results[0].fwd_algo,
    &beta, output_desc, d_output,
    workspace, workspace_size);
```

MIOpen has been moved to the `rocm-libraries` monorepo as of 2025. [Source](https://github.com/ROCm/MIOpen)

### 6.4 Composable Kernel (CK)

The AMD Composable Kernel library provides highly optimized templated GPU kernels for ML operators. Rather than hand-writing one kernel per configuration, CK uses C++ template metaprogramming to compose building blocks (tile loops, memory layouts, threadblock configurations) at compile time:

```cpp
// CK GEMM: DeviceGemmMultipleD with configurable precision and layout
using DeviceGemmInstance = ck::tensor_operation::device::DeviceGemmMultipleD<
    ck::tensor_layout::gemm::RowMajor,   // ALayout
    ck::tensor_layout::gemm::ColumnMajor, // BLayout
    ck::Tuple<>,                           // DsLayout (no extra outputs)
    ck::tensor_layout::gemm::RowMajor,   // ELayout
    ck::half_t,                           // ADataType (FP16)
    ck::half_t,                           // BDataType
    ck::Tuple<>,                           // DsDataType
    ck::half_t,                           // EDataType
    float,                                 // AccDataType (FP32 accumulation)
    /* ... BlockSize, MPerBlock, NPerBlock, KPerBlock, WMMA config ... */
>;
```

CK is the backend for MIOpen's GEMM/convolution paths and for Flash Attention on ROCm (via AOTriton). [Source](https://github.com/ROCm/MIOpen/wiki/Notes-on-Composable-Kernel-library)

### 6.5 Other ROCm Libraries

| Library | Purpose | CUDA analogue |
|---|---|---|
| rocFFT | FFT (1D/2D/3D, batch, R2C/C2C) | cuFFT |
| rocSPARSE | Sparse BLAS (SpMM, SpGEMM, CSR/COO) | cuSPARSE |
| rocSOLVER | Dense linear solver (LU, QR, SVD) | cuSOLVER |
| rocRAND | Pseudo and quasi-random number generation | cuRAND |
| rocALUTION | Sparse iterative solvers (PCG, AMG) | — |
| RCCL | Collective communications (AllReduce, AllGather, Broadcast) | NCCL |
| MIGraphX | ML graph compiler and inference engine | TensorRT |
| rocPRIM | Device-level parallel primitives (scan, sort, reduce) | CUB |
| rocThrust | High-level parallel algorithms (C++ STL style) | Thrust |

### 6.6 RCCL: Collective Communications

RCCL (ROCm Communication Collectives Library) implements multi-GPU and multi-node collective operations. It uses ring-based and tree-based algorithms and leverages AMD Infinity Fabric for intra-node GPU-to-GPU transfers (bypassing host memory) and RDMA-capable NICs for inter-node operations.

```bash
# PyTorch distributed training with RCCL backend
torchrun --nproc_per_node=8 train.py \
    --backend=nccl   # 'nccl' here maps to RCCL on ROCm
```

As of RCCL 2.22+ (ROCm 6.4), socket configuration parameters allow RCCL to select optimal NIC paths in multi-node clusters. [Source](https://github.com/ROCm/rccl)

---

## 7. ROCm Compiler: LLVM/Clang with AMDGPU Backend

### 7.1 Target Triple and GFX Targets

ROCm uses a customized LLVM/Clang toolchain (shipped separately from distribution LLVM as `amdclang`/`amdclang++`). The GPU target triple is `amdgcn-amd-amdhsa`:

- **arch**: `amdgcn` — AMD GCN/RDNA/CDNA ISA family
- **vendor**: `amd`
- **OS**: `amdhsa` — HSA (Heterogeneous System Architecture) ABI

Key GFX target codes:

| GFX Target | GPU | Architecture | SIMD Width |
|---|---|---|---|
| gfx906 | Radeon VII / MI50 | GCN5 (Vega20) | 64 |
| gfx90a | Instinct MI250X | CDNA2 | 64 |
| gfx942 | Instinct MI300X | CDNA3 | 64 |
| gfx1030 | Radeon RX 6800 XT | RDNA2 (Navi21) | 32 |
| gfx1100 | Radeon RX 7900 XTX | RDNA3 (Navi31) | 32 |
| gfx1200 | Radeon RX 9060 XT | RDNA4 (Navi44) | 32 |

[Source](https://llvm.org/docs/AMDGPUUsage.html)

### 7.2 LLVM AMDGPU Backend Passes

The key LLVM backend passes for AMDGPU compilation:

```
HIP C++ source
    │
    ▼  amdclang++ -std=c++17 --offload-arch=gfx1100
Clang AST → LLVM IR (host + device, one TU)
    │
    ▼  AMDGPULowerKernelArguments
Device LLVM IR (kernargs lowered to explicit pointer loads)
    │
    ▼  AMDGPUAnnotateKernelFeatures
Annotated IR (kernel_dynamic_stack_call, uses-dispatch-ptr, etc.)
    │
    ▼  AMDGPUPropagateAttributes
Propagated attributes (compute-mode, wave-size)
    │
    ▼  SIInsertWaitcnts (critical: inserts s_waitcnt instructions)
IR with memory hazard barriers (prevents out-of-order load/store)
    │
    ▼  AMDGPURegisterBankInfo → GISel / DAG instruction selection
AMDGPU MachineIR → ISA binary
    │
    ▼  lld (LLVM linker) + device-libs bitcode link
ELF code object (.co) embedded in host binary
```

`SIInsertWaitcnts` is particularly important: the AMDGPU memory model allows out-of-order memory operations, so the compiler must insert `s_waitcnt` instructions to enforce ordering for loads that depend on prior stores. Missing wait counts are a common source of correctness bugs in hand-written assembly.

### 7.3 ROCm Device Libraries

Every GPU compilation links a set of pre-compiled LLVM bitcode libraries from the `rocm-device-libs` package:

```bash
/opt/rocm/lib/ockl.bc     # OpenCL Kernel Library: atomics, work-item builtins
/opt/rocm/lib/ocml.bc     # OpenCL Math Library: sin, cos, sqrt, exp, erf (IEEE-correct)
/opt/rocm/lib/oclc_isa_version_1100.bc  # ISA feature flags for gfx1100
/opt/rocm/lib/hip.bc      # HIP built-ins: __syncthreads, warp shuffles, ballot
```

These are linked with `-mlink-bitcode-file` during compilation and optimized together with application code via LLVM LTO (Link-Time Optimization), enabling inlining of math functions into kernels. [Source](https://github.com/ROCm/ROCm-Device-Libs/blob/amd-stg-open/doc/OCKL.md)

### 7.4 Compilation Example

```bash
# Compile HIP kernel for RDNA3, with resource usage stats
hipcc -O3 --offload-arch=gfx1100 \
      -Rpass-analyze=kernel-resource-usage \
      vadd.hip -o vadd

# Expected output excerpt:
# remark: [kernel-resource-usage] vadd: NumSgprs: 30, NumVgprs: 8,
#         NumAgprs: 0, SpilledSgprs: 0, SpilledVgprs: 0,
#         LDSSize: 0, ScratchSize: 0, Occupancy: 10

# Check what gets embedded
llvm-readelf --sections vadd | grep hip
# .hip_fatbin   ... (embedded multi-arch fat binary)
```

The `Occupancy: 10` means 10 wavefronts can be resident simultaneously per SIMD unit on this CU. For RDNA3 in wave32 mode, each SIMD unit has 1024 VGPRs (32 lanes × 32 VGPR slots per allocation granule). With 8 VGPRs used per work-item and wave32 (32 lanes), each wave occupies 8 × 32 = 256 VGPR slots; 10 waves × 256 = 2560 VGPR slots, well within the SIMD unit's capacity, so LDS (0 bytes here) is the practical ceiling at 16 waves maximum. Adding LDS or more VGPRs per thread would reduce this occupancy figure.

---

## 8. PyTorch on ROCm

### 8.1 Installation

PyTorch ships pre-built ROCm wheels via the official PyTorch index:

```bash
# ROCm 6.2 wheel (for RDNA3, MI300X)
pip install torch torchvision torchaudio \
    --index-url https://download.pytorch.org/whl/rocm6.2

# ROCm 7.0+ wheel
pip install torch --index-url https://download.pytorch.org/whl/rocm7.0

# Verify ROCm detection
python -c "import torch; print(torch.cuda.is_available()); print(torch.version.hip)"
# True
# 6.2.41133-...
```

PyTorch on ROCm uses a **HIP-as-CUDA masquerade**: at the API level, `torch.cuda.*` functions work on ROCm by mapping to HIP internally. `torch.cuda.is_available()` returns `True` on a ROCm system. This compatibility layer means virtually all CUDA-targeted PyTorch code runs unmodified on ROCm. [Source](https://rocm.docs.amd.com/en/latest/compatibility/ml-compatibility/pytorch-compatibility.html)

### 8.2 GPU Architecture Selection

```bash
# Build PyTorch from source targeting specific architectures
export PYTORCH_ROCM_ARCH="gfx1100;gfx942"  # RDNA3 + MI300X
python setup.py install

# At runtime, restrict visible GPUs
export HIP_VISIBLE_DEVICES=0,1    # analogous to CUDA_VISIBLE_DEVICES
export ROCR_VISIBLE_DEVICES=GPU-<uuid>  # by UUID
```

For RDNA3 GPUs (gfx1100), some users need:
```bash
export HSA_OVERRIDE_GFX_VERSION=11.0.0  # override version detection
```
This is sometimes required for consumer RDNA3 cards when ROCm's official support list doesn't include the exact gfx variant (e.g., gfx1101 on RX 7800 XT). [Source](https://rocm.docs.amd.com/en/docs-6.4.0/about/release-notes.html)

### 8.3 Matrix Operations: hipBLASLt and TunableOp

PyTorch dispatches `torch.matmul` on ROCm through two paths:

1. **rocBLAS** (legacy): stable, broad compatibility
2. **hipBLASLt** (preferred on ROCm 6.3+): algorithm-tunable, FP8/BF16 support

```python
import torch

# Enable hipBLASLt (default on supported architectures)
torch.backends.cuda.matmul.allow_tf32 = True   # BF16-like approximation

# TunableOp: benchmark and cache the fastest GEMM algorithm
# API: torch.cuda.tunable (PyTorch 2.3+)
torch.cuda.tunable.enable()
torch.cuda.tunable.tuning_enable()

# First run benchmarks multiple algorithms; subsequent runs use the cache
A = torch.randn(4096, 4096, dtype=torch.float16, device='cuda')
B = torch.randn(4096, 4096, dtype=torch.float16, device='cuda')
C = torch.matmul(A, B)  # first call: tuning; subsequent calls: fast path
```

Alternatively via environment variables (no code change required):

```bash
PYTORCH_TUNABLEOP_ENABLED=1 \
PYTORCH_TUNABLEOP_FILENAME=tunableop_results.csv \
    python my_model.py
```

### 8.4 Flash Attention on ROCm

Flash Attention (memory-efficient attention) on ROCm is implemented via **AOTriton** (AMD OpenAI Triton fork), which provides pre-compiled Triton kernels for common attention configurations:

```python
import torch
import torch.nn.functional as F

# scaled_dot_product_attention automatically uses AOTriton Flash Attention
# on ROCm (PyTorch 2.7+ with ROCm 7.0+)
q = torch.randn(batch, heads, seq_len, head_dim,
                dtype=torch.float16, device='cuda')
k = torch.randn_like(q)
v = torch.randn_like(q)

# Uses AOTriton FA2-class kernel via AOTriton on ROCm 7.0+
out = F.scaled_dot_product_attention(q, k, v, is_causal=True)
```

AOTriton ships pre-compiled kernels for common (batch, heads, seq_len, head_dim) configurations, eliminating the JIT compilation latency of pure Triton. For configurations not covered by pre-compiled kernels, it falls back to the Triton JIT path. Note: FlashAttention v3's Hopper-specific optimizations (warp specialization, TMA) do not translate to RDNA/CDNA; AOTriton implements FA2-class algorithms adapted for the AMDGPU memory model. [Source](https://rocm.docs.amd.com/en/latest/compatibility/ml-compatibility/pytorch-compatibility.html)

### 8.5 `torch.compile` on ROCm

```python
import torch

model = MyTransformer().to('cuda').half()
# torch.compile with Inductor backend (uses triton-rocm for GPU kernels)
compiled_model = torch.compile(model, backend='inductor')

# First forward pass triggers kernel compilation; subsequent passes are fast
output = compiled_model(input_ids)
```

`torch.compile` on ROCm uses the **Triton-ROCm** backend (OpenAI Triton targeting HIP/AMDGPU) for generated kernels. The Triton compiler generates Triton IR → TritonGPU IR → LLVM IR → AMDGPU ISA via `amdclang`. Compilation artifacts are cached in `~/.triton/cache/` to amortize JIT cost across runs.

Known limitation as of ROCm 7.2: FP16/BF16 reduction accumulation types are not fully supported on all RDNA architectures.

### 8.6 Distributed Training with RCCL

```python
import torch
import torch.distributed as dist

# Initialize RCCL process group (identical API to NCCL)
dist.init_process_group(backend='nccl')  # 'nccl' uses RCCL on ROCm

rank = dist.get_rank()
device = torch.device(f'cuda:{rank}')

model = MyModel().to(device)
ddp_model = torch.nn.parallel.DistributedDataParallel(model, device_ids=[rank])

# AllReduce gradients via RCCL (Infinity Fabric intra-node, RDMA inter-node)
loss = ddp_model(inputs).sum()
loss.backward()  # triggers RCCL AllReduce under the hood
```

```bash
# Launch 8-GPU training on a single MI300X node
torchrun --nproc_per_node=8 train_llm.py \
    --model llama3-70b --batch-size 4 --grad-accum 8
```

### 8.7 Practical: LLaMA 3 Inference on RX 7900 XTX

The RX 7900 XTX (24 GB GDDR6, gfx1100) is capable of running 70B parameter models quantized to 4-bit (Q4_K_M ≈ 35 GB fits on dual-GPU setups; 8B models fit in 24 GB single-GPU):

```bash
# Using llama.cpp with ROCm backend (hipBLAS)
git clone https://github.com/ggerganov/llama.cpp
mkdir build && cd build
cmake .. -DGGML_HIPBLAS=ON \
         -DAMDGPU_TARGETS=gfx1100 \
         -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)

# Run LLaMA 3 8B at 4-bit quantization
./llama-cli -m llama-3-8b-q4_k_m.gguf \
            --n-gpu-layers 99 \   # offload all layers to GPU
            -p "The Linux graphics stack consists of"
```

For 70B inference with ROCm, `vLLM` with ROCm support provides better throughput via PagedAttention:

```bash
pip install vllm  # ROCm wheel available
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3-70b-Instruct \
    --dtype float16 \
    --tensor-parallel-size 2  # splits across 2 RX 7900 XTX GPUs
```

[Source](https://github.com/ggerganov/llama.cpp)

---

## 9. OpenCL on ROCm

### 9.1 ROCm OpenCL via CLR

ROCm ships an OpenCL 2.0 implementation through the **CLR** (Common Language Runtime, formerly rocclr). The OpenCL ICD (Installable Client Driver) is loaded by `libOpenCL.so` when `cl_amd_platform` is detected:

```bash
# Enumerate OpenCL platforms
clinfo | grep -E "Platform|Device|Compute|Memory"

# Build and run an OpenCL program against ROCm ICD
gcc -o mandelbrot mandelbrot.c -lOpenCL
./mandelbrot
```

CLR compiles OpenCL C kernels through the same LLVM AMDGPU backend as HIP: `clBuildProgram` invokes `amdclang` with the OpenCL front-end, producing an ELF code object identical in format to HIP code objects.

### 9.2 ROCm OpenCL vs Mesa Rusticl

Mesa **Rusticl** (introduced in Mesa 22.3, Rust-based OpenCL 3.0 via Gallium/NIR) provides an alternative OpenCL implementation that does **not** require KFD. The two implementations target different use cases:

| Property | ROCm CLR | Mesa Rusticl |
|---|---|---|
| OpenCL version | 2.0 (SVM on APU) | 3.0 |
| Kernel driver | amdkfd (`/dev/kfd`) | amdgpu DRM render node |
| Hardware support | CDNA2+, RDNA2+ (limited list) | Any Gallium-supported GPU |
| Dependencies | Full ROCm install | Mesa only |
| SVM | Yes (gfx9+) | No |
| Performance | Higher (KFD direct queue) | Competitive (HMM) |
| Use case | HPC/ML compute | Desktop compute, image processing |

```bash
# Enable Rusticl for a specific GPU
RUSTICL_ENABLE=radeonsi clinfo   # use Rusticl via radeonsi Gallium
AMD_VULKAN_ICD=RADV RUSTICL_ENABLE=radeonsi darktable  # Darktable OpenCL
```

Benchmarks on the Ryzen AI Max+ "Strix Halo" APU (ROCm 7.0 vs Mesa Rusticl) show the two implementations are competitive, with ROCm winning on bandwidth-bound workloads and Rusticl occasionally winning on latency-sensitive short kernels due to lower queue setup overhead. [Source](https://www.phoronix.com/review/rocm-7-rusticl-opencl)

### 9.3 When to Use Each

- **ROCm CLR**: Compute-intensive ML/HPC workloads requiring OpenCL 2.0 SVM, or when existing ROCm infrastructure is already deployed. Required for KFD-based queue scheduling.
- **Mesa Rusticl**: General desktop compute (image editing, Darktable), broader hardware support without a ROCm installation, or when only a Mesa driver is available (older GPUs not on ROCm's official support list).

---

## 10. Debugging and Profiling ROCm Workloads

### 10.1 ROCm-SMI: System Management

`rocm-smi` (or the newer `amd-smi`) provides GPU management analogous to `nvidia-smi`:

```bash
# Show GPU utilization and memory
rocm-smi --showuse --showmeminfo vram

# Show temperature and power
rocm-smi --showtemp --showpower

# Select which GPUs are visible to ROCm applications
export HIP_VISIBLE_DEVICES=0,2   # expose GPU 0 and GPU 2 only

# Continuous monitoring (1-second refresh)
watch -n1 rocm-smi
```

### 10.2 rocprofv3: Tracing and Counter Collection

`rocprofv3` (ROCm 6.3+, replaces the deprecated `rocprof`) collects hardware counter events and kernel execution statistics:

```bash
# Collect kernel timing statistics
rocprofv3 --stats ./my_app

# Collect specific hardware counters
rocprofv3 --counter TCC_HIT[0],TCC_MISS[0],SQ_WAVES \
           ./my_app

# Output: CSV with kernel name, duration, counter values
```

The `--stats` output includes per-kernel GPU time, number of invocations, and average/min/max duration. The hardware counters (`TCC_*` for L2 cache, `SQ_WAVES` for wave count) map to AMD's PMU (Performance Monitoring Unit) register set.

### 10.3 rocprof-compute (formerly Omniperf): Kernel Performance Analysis

`rocprof-compute` (ROCm 6.3+) provides system-level kernel performance analysis with roofline modeling:

```bash
# Profile a kernel with hardware counters
rocprof-compute profile --name my_profile -- ./my_app

# Analyze the collected data
rocprof-compute analyze --path ./workloads/my_profile

# Generate roofline model: identify memory vs compute bound kernels
rocprof-compute analyze --path ./workloads/my_profile --report roofline
```

The roofline analysis plots achieved FLOPS/s against arithmetic intensity (FLOPs/byte), making it immediately visible whether a kernel is memory-bandwidth bound (left of the roofline ridge point) or compute bound (right of the ridge point). [Source](https://rocm.docs.amd.com/projects/rocprofiler-compute/en/docs-6.2.4/what-is-omniperf.html)

### 10.4 rocgdb: GPU Kernel Debugger

`rocgdb` extends GDB for GPU debugging, enabling single-step execution, breakpoints, and variable inspection inside running GPU kernels:

```bash
# Debug a HIP application
rocgdb ./my_app

(rocgdb) break vadd      # set breakpoint at GPU kernel
(rocgdb) run
# Breakpoint hit on GPU kernel vadd
(rocgdb) info threads    # show all GPU waves (threads)
(rocgdb) print threadIdx.x  # inspect GPU built-in variables
(rocgdb) print v0        # inspect VGPR v0 for current lane
(rocgdb) next            # step one GPU instruction
```

`rocgdb` communicates with the GPU via the KFD debug interface (`kfd_debug.c`). Single-step stalls the entire GPU, so it is suitable for correctness debugging but not performance analysis.

### 10.5 roctx: Application-Level Markers

`roctx` provides markers for annotating application timelines, which appear in profiling tools (rocprofv3, rocprof-compute):

```cpp
#include <roctx.h>

void train_step(/* ... */) {
    roctxRangePush("forward_pass");
    // ... forward pass code ...
    roctxRangePop();

    roctxRangePush("backward_pass");
    // ... backward pass code ...
    roctxRangePop();
}
```

### 10.6 Environment Variables for Debugging

```bash
# Disable deferred loading (catch driver errors earlier)
HIP_ENABLE_DEFERRED_LOADING=0 ./my_app

# Enable AMD ROCm runtime logging
AMD_LOG_LEVEL=4 ./my_app 2>rocm.log   # 4 = verbose

# Print each HIP API call (like CUDA_LAUNCH_BLOCKING + verbose)
HIP_TRACE_API=1 ./my_app

# Force wave64 on RDNA3 (usually wave32 by default) via hipcc flag
# hipcc --offload-arch=gfx1100 -mwavefrontsize64 kernel.hip -o kernel_w64
```

### 10.7 Common Performance Pitfalls

**VGPR Spilling**: When a kernel uses more VGPRs than the hardware allocates (e.g., more than 256 per lane on RDNA3 in wave32 mode), the compiler spills excess registers to **scratch memory** (GPU private memory backed by VRAM). Scratch accesses are 10–100× slower than register access. Detect spilling with:

```bash
hipcc -Rpass-analyze=kernel-resource-usage ... 2>&1 | grep Spilled
# remark: SpilledVgprs: 12  ← bad: 12 VGPRs spilled to scratch
```

Mitigation: reduce loop unrolling, split the kernel into smaller stages, or use `__launch_bounds__(max_threads_per_block, min_warps_per_eu)` to give the compiler VGPR budget hints (this is the HIP/CUDA-portable form); alternatively the AMDGPU-specific attribute `__attribute__((amdgpu_waves_per_eu(min, max)))` gives the register allocator an explicit wave-count target.

**Low Occupancy**: Occupancy measures how many wavefronts are simultaneously active on a CU. Low occupancy prevents hiding memory latency:
- Too many VGPRs → fewer concurrent waves (each wave needs its VGPR allocation)
- Too much LDS → fewer work-groups fit per CU
- Check with `rocprof-compute analyze --report occupancy`

**LDS Bank Conflicts** (RDNA3 WGP mode): In WGP mode, all 32-lane waves on the WGP share one 128 KB LDS bank. Accessing the same bank from multiple lanes in the same cycle causes serialization. Use padding or swizzled addressing:

```cpp
// Pad LDS arrays to avoid bank conflicts (32-bank LDS)
__shared__ float smem[BLOCK_SIZE][BLOCK_SIZE + 1]; // +1 padding
```

---

## 11. Integrations

**Ch1 — DRM Architecture**: ROCm's KFD (`/dev/kfd`) is an extension of the amdgpu DRM driver implemented in the same `amdgpu.ko` module. The DRM infrastructure (GEM object management, DMA fence synchronization, DMA-BUF export) is shared between the graphics and compute paths — a HIP kernel producing a texture for Vulkan rendering uses DMA-BUF to pass the GEM buffer handle from the ROCm stack to the Mesa Vulkan driver.

**Ch5 — amdgpu Driver**: The amdgpu kernel driver is the foundation for both the graphics display stack (KMS, GEM, ring buffers for CS command submission) and the ROCm compute stack (KFD queue management, PASID assignment, SVM via HMM). Ch5 covers the DRM side; this chapter covers the compute side of the same driver.

**Ch87 — LLM Inference on Linux**: llama.cpp's ROCm backend (`ggml-hipblas.cpp`) uses HIP and hipBLAS for matrix-vector multiply in transformer inference. Ollama's `gpu.go` detects ROCm via `/dev/kfd` presence. vLLM's ROCm support uses `torch.distributed` with RCCL for tensor-parallel multi-GPU inference.

**Ch88 — NPU and AI Accelerators**: The AMD XDNA NPU driver (for Ryzen AI) uses the DRM `accel` subsystem (`/dev/accel/accel0`) rather than KFD, and targets different hardware (AIE2 tiles vs. CDNA/RDNA GPU arrays). The two coexist: XDNA handles power-efficient sustained inference; ROCm/KFD handles peak-throughput GPU compute. The boundary is the `amdxdna` vs. `amdkfd` kernel module boundary.

**Ch91 — MLIR and GPU Compiler Infrastructure**: MLIR's `rocdl` dialect generates LLVM IR targeting `amdgcn-amd-amdhsa`, the same target as `hipcc`. OpenAI Triton compiles `@triton.jit` kernels through TritonGPU IR → AMDGPU LLVM IR → ISA, using the same LLVM AMDGPU backend described in §7. Flash Attention's ROCm implementation in AOTriton demonstrates the Triton→AMDGPU compilation pipeline in production.

**Ch93 — GPU Performance Analysis Methodology**: The general performance analysis methodology (roofline model, memory bandwidth vs. compute throughput, occupancy analysis) applies directly to ROCm workloads. `rocprof-compute`/Omniperf is the ROCm-specific tool implementing this methodology, sitting alongside RGP (Radeon GPU Profiler, Ch93) which targets graphics workloads. Both use the same KFD hardware counter infrastructure.

---

*Sources consulted: [ROCm documentation](https://rocm.docs.amd.com/), [ROCm GitHub](https://github.com/ROCm/ROCm), [HIP documentation](https://rocm.docs.amd.com/projects/HIP/en/latest/), [LLVM AMDGPU Backend](https://llvm.org/docs/AMDGPUUsage.html), [Linux kernel amdkfd sources](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/amd/amdkfd), [MIOpen](https://github.com/ROCm/MIOpen), [RCCL](https://github.com/ROCm/rccl), [ROCm 6.4 release notes](https://rocm.docs.amd.com/en/docs-6.4.0/about/release-notes.html), [ROCm 7.2 release notes](https://rocm.docs.amd.com/en/docs-7.2.0/about/release-notes.html), [Phoronix Rusticl vs ROCm benchmark](https://www.phoronix.com/review/rocm-7-rusticl-opencl), [Omniperf/rocprof-compute documentation](https://rocm.docs.amd.com/projects/rocprofiler-compute/en/docs-6.2.4/what-is-omniperf.html), [PyTorch ROCm compatibility matrix](https://rocm.docs.amd.com/en/latest/compatibility/ml-compatibility/pytorch-compatibility.html), [AMD GPUOpen occupancy guide](https://gpuopen.com/learn/occupancy-explained/).*
