# Chapter 161: OpenCL on Linux: Clover, Intel, and ROCm

**Target audiences**: Engineers using GPU compute for scientific computing, ML inference, and image processing on Linux; developers choosing between OpenCL, CUDA, and Vulkan compute; and systems architects evaluating vendor compute stacks.

---

## Table of Contents

1. [Introduction](#introduction)
2. [OpenCL Architecture and ICD Loader](#opencl-architecture-and-icd-loader)
3. [Clover: Mesa OpenCL for Gallium Drivers](#clover-mesa-opencl-for-gallium-drivers)
4. [RustiCL: Rust OpenCL in Mesa](#rusticl-rust-opencl-in-mesa)
5. [Intel OpenCL: compute-runtime and NEO](#intel-opencl-compute-runtime-and-neo)
6. [AMD ROCm: HIP and OpenCL](#amd-rocm-hip-and-opencl)
7. [NVIDIA OpenCL on Linux](#nvidia-opencl-on-linux)
8. [OpenCL Programming Model](#opencl-programming-model)
9. [Choosing Between OpenCL, CUDA, and Vulkan Compute](#choosing-between-opencl-cuda-and-vulkan-compute)
10. [Integrations](#integrations)

---

## Introduction

OpenCL (Open Computing Language) is the cross-platform GPU compute API standardised by the Khronos Group. On Linux, multiple implementations coexist via the ICD (Installable Client Driver) loader mechanism: Mesa's Clover (for RadeonSI), Mesa's RustiCL (Rust-based, for RADV/RadeonSI), Intel's compute-runtime (NEO), AMD's ROCm (HIP + OpenCL), and NVIDIA's proprietary OpenCL.

OpenCL has lost ground to CUDA (NVIDIA) and ROCm (AMD) for ML workloads, but remains important for cross-platform compute (video processing, scientific computing, image processing libraries like OpenCV).

Sources: [OpenCL spec](https://www.khronos.org/opencl/) | [Mesa RustiCL](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/frontends/rusticl) | [Intel compute-runtime](https://github.com/intel/compute-runtime) | [ROCm](https://rocm.docs.amd.com/)

---

## OpenCL Architecture and ICD Loader

### ICD Loader Mechanism

Multiple OpenCL implementations coexist on one system via the ICD loader (`libOpenCL.so`). Each implementation registers an ICD file:

```bash
# List installed OpenCL ICDs:
ls /etc/OpenCL/vendors/
# mesa.icd       → Mesa (Clover/RustiCL)
# intel.icd      → Intel compute-runtime (NEO)
# amdocl64.icd   → AMD ROCm
# nvidia.icd      → NVIDIA proprietary

# List available platforms:
clinfo --list
# Platform #0: Intel(R) OpenCL Graphics
# Platform #1: AMD ROCm OpenCL ...
# Platform #2: Clover
```

### Platform and Device Hierarchy

```c
/* OpenCL host code: discover platforms and devices */
cl_uint num_platforms;
clGetPlatformIDs(0, NULL, &num_platforms);
cl_platform_id *platforms = malloc(num_platforms * sizeof(*platforms));
clGetPlatformIDs(num_platforms, platforms, NULL);

for (cl_uint p = 0; p < num_platforms; p++) {
    char name[256];
    clGetPlatformInfo(platforms[p], CL_PLATFORM_NAME, sizeof(name), name, NULL);
    printf("Platform: %s\n", name);

    cl_uint num_devices;
    clGetDeviceIDs(platforms[p], CL_DEVICE_TYPE_GPU, 0, NULL, &num_devices);
    cl_device_id *devices = malloc(num_devices * sizeof(*devices));
    clGetDeviceIDs(platforms[p], CL_DEVICE_TYPE_GPU, num_devices, devices, NULL);

    for (cl_uint d = 0; d < num_devices; d++) {
        clGetDeviceInfo(devices[d], CL_DEVICE_NAME, sizeof(name), name, NULL);
        cl_ulong mem;
        clGetDeviceInfo(devices[d], CL_DEVICE_GLOBAL_MEM_SIZE, sizeof(mem), &mem, NULL);
        printf("  Device: %s (%.1f GB)\n", name, mem / 1e9);
    }
}
```

---

## Clover: Mesa OpenCL for Gallium Drivers

### What Clover Is

Clover is Mesa's OpenCL state tracker, targeting Gallium3D drivers (RadeonSI, Iris, softpipe). It was the primary Mesa OpenCL implementation until RustiCL superseded it.

```bash
# Check Clover:
ls /usr/lib/x86_64-linux-gnu/libMesaOpenCL.so*
# Clover targets OpenCL 1.2 / 2.0 (limited)
```

### Clover Architecture

```
OpenCL host code (clCreateBuffer, clEnqueueNDRange, ...)
    │
    ▼
Clover (src/gallium/frontends/clover/)
    │ calls pipe_context directly
    ▼
Gallium pipe driver (RadeonSI / Iris / softpipe)
    │
    ▼
GPU ISA
```

Clover compiles OpenCL C via LLVM's OpenCL front-end → LLVM IR → SPIRV (via llvm-spirv) → NIR → GPU ISA.

### Clover Limitations

Clover supports OpenCL 1.2 with limited 2.0 features. It lacks:
- OpenCL 3.0 features
- SVM (Shared Virtual Memory)
- `cl_khr_gl_sharing` (OpenGL interop) on all targets
- `cl_khr_fp64` on softpipe

RustiCL is Clover's replacement.

---

## RustiCL: Rust OpenCL in Mesa

### What RustiCL Is

RustiCL (by Karol Herbst, Red Hat) is a full OpenCL 3.0 implementation written in Rust, using Mesa's Gallium pipe drivers (primarily RADV for AMD, Iris for Intel):

```bash
# Enable RustiCL:
RUSTICL_ENABLE=radeonsi,iris clinfo
# Or globally:
echo 'RUSTICL_ENABLE=radeonsi,iris' | sudo tee /etc/environment
```

### Architecture

```
OpenCL application
    │
    ▼
RustiCL (src/gallium/frontends/rusticl/)
    │ Rust bindings to pipe_context
    ▼
Gallium pipe driver (RADV Gallium / RadeonSI / Iris Gallium / Zink)
    │
    ▼
GPU (via RADV or Iris)
```

RustiCL targets RADV (AMD Vulkan backend) via Zink or RadeonSI/Iris directly.

### OpenCL 3.0 Features in RustiCL

```bash
# Check OpenCL 3.0 support:
clinfo | grep "Device OpenCL C Version"
# Should show: OpenCL C 3.0

# Key 3.0 features in RustiCL:
# - Unified SVM (shared virtual memory)
# - Program scope global variables
# - Sub-groups
# - cl_khr_fp16, cl_khr_fp64
```

### RustiCL Rust API Structure

```rust
/* src/gallium/frontends/rusticl/core/context.rs */
pub struct RusticlContext {
    pub base: cl_context,
    pub devs: Vec<Arc<Device>>,
    pub pipes: HashMap<*const Device, PipeContext>,
    /* Memory objects: */
    pub svm_ptrs: Mutex<HashMap<usize, Arc<Mem>>>,
}

impl RusticlContext {
    pub fn create_buffer(&self, size: usize, flags: cl_mem_flags)
        -> CLResult<Arc<Mem>>
    {
        /* Allocate via Gallium pipe_screen::resource_create: */
        let pipe_res = self.screen.resource_create_buffer(size, flags)?;
        Ok(Arc::new(Mem::Buffer { res: pipe_res, size }))
    }
}
```

---

## Intel OpenCL: compute-runtime and NEO

### Architecture

Intel's OpenCL stack for Gen9/Gen12/Xe:

```
OpenCL application
    │
    ▼
Intel OpenCL ICD (libigdrcl.so / libOpenCL.so stub)
    │
    ▼
compute-runtime (NEO) — github.com/intel/compute-runtime
    │
    ▼
Level Zero (backend)    ← i915 / xe kernel driver
    │                   ← GPU command submission
    ▼
Intel GPU (Gen12 / Xe / Xe2)
```

```bash
# Install Intel compute-runtime:
sudo apt install intel-opencl-icd  # Ubuntu
# or:
sudo dnf install intel-compute-runtime  # Fedora

# Verify:
clinfo | grep Intel
```

### Level Zero

Intel's compute-runtime uses **Level Zero** as its low-level backend (analogous to Vulkan for compute):

```c
/* Level Zero API (lower level than OpenCL): */
#include <level_zero/ze_api.h>

ze_context_desc_t ctx_desc = { ZE_STRUCTURE_TYPE_CONTEXT_DESC };
ze_context_handle_t context;
zeContextCreate(driver, &ctx_desc, &context);

ze_command_queue_desc_t cq_desc = { ZE_STRUCTURE_TYPE_COMMAND_QUEUE_DESC };
ze_command_queue_handle_t cq;
zeCommandQueueCreate(context, device, &cq_desc, &cq);
```

Level Zero is used by Intel's oneAPI toolkit (oneDNN, oneMKL) and is the preferred compute API for Intel Xe GPUs.

### Intel Compiler: IGC

The Intel Graphics Compiler (IGC) compiles OpenCL C / SPIR-V → Intel GPU ISA (GEN Assembly / EU ISA):

```bash
# IGC is invoked internally by NEO:
# OpenCL C → clang (SPIR-V) → IGC optimizer → GEN ISA binary
# Debug: set OCL_ICD_VENDORS=/usr/local/lib/OpenCL/vendors/intel.icd
```

---

## AMD ROCm: HIP and OpenCL

### ROCm Stack

AMD's ROCm (Radeon Open Compute) is AMD's heterogeneous compute platform:

```
Applications (HIP, OpenCL, MIOpen)
    │
    ▼
ROCm Runtime (hip-runtime-amd, rocclr)
    │
    ▼
ROCr (ROCm Runtime, formerly HSA Runtime)
    │
    ▼
KFD (Kernel Fusion Driver) — amdkfd in Linux kernel
    │
    ▼
AMD GPU (RDNA2/RDNA3/CDNA)
```

```bash
# Install ROCm:
sudo apt install rocm-hip-libraries rocm-opencl-runtime  # Ubuntu 22.04+
# Check:
rocminfo | grep "gfx\|Name"
clinfo --list | grep AMD
```

### HIP: CUDA-like API

HIP (Heterogeneous Integrated Platform) is AMD's CUDA-compatible API:

```cpp
#include <hip/hip_runtime.h>

__global__ void vector_add(const float *a, const float *b, float *c, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) c[i] = a[i] + b[i];
}

int main() {
    float *d_a, *d_b, *d_c;
    hipMalloc(&d_a, n * sizeof(float));
    hipMalloc(&d_b, n * sizeof(float));
    hipMalloc(&d_c, n * sizeof(float));
    hipMemcpy(d_a, h_a, n * sizeof(float), hipMemcpyHostToDevice);

    dim3 threads(256), blocks((n + 255) / 256);
    hipLaunchKernelGGL(vector_add, blocks, threads, 0, 0, d_a, d_b, d_c, n);

    hipMemcpy(h_c, d_c, n * sizeof(float), hipMemcpyDeviceToHost);
    hipFree(d_a); hipFree(d_b); hipFree(d_c);
}
```

HIP compiles via `hipcc` → AMD's `clang` with ROCm backend → AMDGPU ISA.

### ROCm OpenCL

ROCm includes `rocm-opencl-runtime` which provides an OpenCL 2.0 implementation using the same hardware path as HIP:

```bash
# ROCm OpenCL example:
OCL_ICD_VENDORS=/etc/OpenCL/vendors/amdocl64.icd clinfo
# Shows: AMD Accelerated Parallel Processing
```

---

## NVIDIA OpenCL on Linux

NVIDIA provides a proprietary OpenCL 3.0 implementation via the standard driver package:

```bash
# NVIDIA OpenCL is included with the NVIDIA driver:
ls /usr/lib/x86_64-linux-gnu/libnvidia-opencl.so*
# ICD file:
cat /etc/OpenCL/vendors/nvidia.icd
# → libnvidia-opencl.so.1

# Check:
clinfo --list | grep NVIDIA
# Platform: NVIDIA CUDA → CL_DEVICE_TYPE_GPU → Tesla/RTX ...
```

OpenCL on NVIDIA compiles kernels via NVRTC (NVIDIA Runtime Compilation) → PTX → GPU ISA.

---

## OpenCL Programming Model

### Basic Kernel and Execution

```c
/* OpenCL vector addition (portable across all vendors): */
const char *kernel_src =
"__kernel void vadd(__global const float *a, __global const float *b,\n"
"                   __global float *c, int n) {\n"
"    int i = get_global_id(0);\n"
"    if (i < n) c[i] = a[i] + b[i];\n"
"}\n";

/* Compile and run: */
cl_context ctx = clCreateContext(NULL, 1, &device, NULL, NULL, &err);
cl_command_queue queue = clCreateCommandQueueWithProperties(ctx, device, NULL, &err);
cl_program program = clCreateProgramWithSource(ctx, 1, &kernel_src, NULL, &err);
clBuildProgram(program, 1, &device, "-cl-std=CL3.0", NULL, NULL);
cl_kernel kernel = clCreateKernel(program, "vadd", &err);

cl_mem buf_a = clCreateBuffer(ctx, CL_MEM_READ_ONLY  | CL_MEM_COPY_HOST_PTR,
    n * sizeof(float), h_a, &err);
cl_mem buf_c = clCreateBuffer(ctx, CL_MEM_WRITE_ONLY, n * sizeof(float), NULL, &err);

clSetKernelArg(kernel, 0, sizeof(cl_mem), &buf_a);
clSetKernelArg(kernel, 2, sizeof(cl_mem), &buf_c);
clSetKernelArg(kernel, 3, sizeof(int), &n);

size_t global_work_size = n;
size_t local_work_size  = 256;
clEnqueueNDRangeKernel(queue, kernel, 1, NULL,
    &global_work_size, &local_work_size, 0, NULL, NULL);

clEnqueueReadBuffer(queue, buf_c, CL_TRUE, 0, n * sizeof(float), h_c,
    0, NULL, NULL);
```

### Shared Virtual Memory (SVM)

OpenCL 2.0+ SVM lets the GPU access CPU pointers directly:

```c
/* Coarse-grained SVM — pointer valid on both CPU and GPU: */
float *svm_ptr = clSVMAlloc(ctx,
    CL_MEM_READ_WRITE | CL_MEM_SVM_FINE_GRAIN_BUFFER,
    n * sizeof(float), 0);

/* GPU kernel can access svm_ptr directly (no explicit transfer): */
clSetKernelArgSVMPointer(kernel, 0, svm_ptr);
clEnqueueNDRangeKernel(queue, kernel, 1, NULL, &global, &local, 0, NULL, NULL);
clFinish(queue);
/* CPU reads svm_ptr without a separate copy */
```

---

## Choosing Between OpenCL, CUDA, and Vulkan Compute

| Criteria | OpenCL | CUDA | Vulkan Compute | HIP |
|---|---|---|---|---|
| Vendor support | Intel, AMD, NVIDIA, ARM | NVIDIA only | All Vulkan GPUs | AMD, some NVIDIA |
| Performance | Good | Best (NVIDIA) | Very good | Good (AMD) |
| Portability | Excellent | None | Excellent | Limited |
| Tooling | clinfo, gdb-ocl | nsight, nvvp | RenderDoc | rocprof |
| Ecosystem | Libraries exist | Largest | Growing | ROCm libs |
| Ease of use | Verbose C | Clean C++ | Very verbose | CUDA-like |

**Recommendation**:
- Cross-platform GPU compute: OpenCL via RustiCL (Mesa) or Vulkan compute
- NVIDIA-only: CUDA
- AMD compute/ML: ROCm/HIP + MIOpen
- New cross-platform: Vulkan compute + GLSL/SPIR-V (best long-term)

---

## Integrations

- **Ch14 (Gallium3D)** — Clover and RustiCL are Gallium frontends; RustiCL calls `pipe_context::launch_grid` for compute dispatch, the same interface used by OpenGL compute shaders
- **Ch19 (Vulkan)** — Vulkan compute (`vkCmdDispatch`) is the modern alternative to OpenCL; RustiCL uses RADV's Vulkan internals via Zink
- **Ch26 (Hardware Video)** — VA-API and V4L2 video decode pipelines can output buffers consumed by OpenCL for GPU-accelerated post-processing (e.g. OpenCV `UMat`)
- **Ch152 (Rust GPU)** — wgpu compute and OpenCL both target GPU SPIR-V; RustiCL is implemented in Rust, paralleling wgpu's Rust-first approach
- **Ch154 (GPU-Driven Rendering)** — GPU culling shaders (compute) are an alternative use case to OpenCL; Vulkan compute pipelines share the same hardware ALUs as OpenCL kernels
