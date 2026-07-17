# Chapter 134: OpenCL on Linux

## Scope

This chapter targets:
- **GPU compute developers** — implementing OpenCL kernels for cross-vendor GPU acceleration
- **Image processing engineers** — using OpenCL for camera, video, and photo processing pipelines
- **ML practitioners** — needing vendor-neutral GPU acceleration across AMD, Intel, and NVIDIA hardware
- **Mesa contributors** — working on OpenCL support through the rusticl Gallium frontend

It covers the full OpenCL ecosystem on Linux:
- **ICD dispatch architecture** — the Installable Client Driver loader model and platform enumeration
- **Mesa's rusticl Gallium frontend** — OpenCL 3.0 via Gallium, replacing the legacy Clover implementation
- **Intel's NEO compute runtime** — the open-source OpenCL ICD for Intel GPUs from Gen8 through Xe3P
- **AMD's ROCm CLR OpenCL stack** — Compute Language Runtime providing OpenCL 2.0 on RDNA/CDNA hardware
- **pocl CPU/multi-target implementation** — portable OpenCL 3.0 via LLVM for CPU, CUDA, and Level Zero targets
- **OpenCL programming model** — including SVM, USM, NDRange dispatch, and SPIR-V ingestion
- **OpenCL–OpenGL/Vulkan interoperability via DMA-BUF** — zero-copy cross-API pipelines using `cl_khr_external_memory`
- **Real-world applications** — Darktable, hashcat, FFmpeg, BOINC, and vkFFT usage patterns
- **Debugging and profiling tools** — oclgrind, clpeak, `clGetEventProfilingInfo`, and vendor-specific profilers

## Table of Contents

1. [Introduction](#1-introduction)
2. [OpenCL Architecture and the ICD Loader](#2-opencl-architecture-and-the-icd-loader)
3. [Mesa rusticl — OpenCL 3.0 via Gallium](#3-mesa-rusticl--opencl-30-via-gallium)
   - [3.1 Legacy: Clover](#31-legacy-clover--mesas-pre-rusticl-opencl)
4. [Intel OpenCL: NEO / Compute Runtime](#4-intel-opencl-neo--compute-runtime)
5. [AMD ROCm OpenCL (CLR)](#5-amd-rocm-opencl-clr)
5.1. [NVIDIA OpenCL on Linux](#51-nvidia-opencl-on-linux)
6. [CPU/Software OpenCL: pocl](#6-cpusoftware-opencl-pocl)
7. [OpenCL Programming Model](#7-opencl-programming-model)
8. [OpenCL–OpenGL Interop and DMA-BUF](#8-openclopengl-interop-and-dma-buf)
9. [Real-World Applications](#9-real-world-applications)
10. [Debugging and Profiling](#10-debugging-and-profiling)
11. [Integrations](#11-integrations)

---

## 1. Introduction

OpenCL (Open Computing Language) is a Khronos standard for heterogeneous parallel compute, ratified in 2008 and predating CUDA's dominance on Linux workloads. Where CUDA is NVIDIA-only and AMD's HIP targets primarily CDNA/RDNA hardware with a CUDA-compatible API, OpenCL defines a single portable model that runs on any GPU, CPU, FPGA, or DSP for which a conformant Installable Client Driver (ICD) exists. This vendor-agnostic portability makes it uniquely valuable on Linux, where the diversity of hardware is highest.

OpenCL matters in practice for several Linux-native workloads:

- **Darktable**: RAW photo processing uses OpenCL for demosaicing, tone-mapping, and colour transforms, with adaptive GPU tiling to handle sensors exceeding available VRAM.
- **mpv and FFmpeg**: Video filter chains, colour-space conversion, and upscaling (e.g., `--vf=gpu` with OpenCL-backed filters) run on any OpenCL-capable GPU.
- **hashcat**: GPU-accelerated password cracking exploits OpenCL for cross-vendor portability — a single code-path runs on AMD, Intel, and NVIDIA hardware.
- **BOINC and scientific applications**: Distributed compute projects (Milkyway@home, Asteroids@home) use OpenCL to exploit volunteer GPU capacity across all vendors.
- **vkFFT** ([github.com/DTolm/VkFFT](https://github.com/DTolm/VkFFT)): A high-performance FFT library that supports Vulkan, OpenCL, and CUDA backends for scientific computing.
- **LibreOffice Calc**: Experimental GPU acceleration for spreadsheet computation targets OpenCL.

OpenCL 3.0, the current specification (published 2020, updated continuously), restructured the standard by making OpenCL 1.2 features mandatory and moving everything from OpenCL 2.x (SVM, pipes, device-side enqueue) into an optional-feature tier. This reduced the conformance burden for new implementations while preserving the extended feature set for platforms that support it. As of 2025–2026, three open-source Linux implementations have achieved official OpenCL 3.0 Khronos conformance: Intel NEO, Mesa rusticl, and pocl.

---

## 2. OpenCL Architecture and the ICD Loader

### The Dispatch Architecture

OpenCL on Linux is built around the Installable Client Driver (ICD) model. Applications link against `libOpenCL.so`, which is the ICD loader — not an OpenCL implementation itself. The loader's job is to enumerate available platform ICDs, dispatch every OpenCL API call to the correct underlying implementation, and present a unified API surface to the caller.

There are two ICD loaders in common use on Linux:

- **ocl-icd** ([github.com/OCL-dev/ocl-icd](https://github.com/OCL-dev/ocl-icd)): The de facto open-source standard, packaged as `ocl-icd-libopencl1` on Debian/Ubuntu. It is the loader shipped by essentially all Linux distributions.
- **Khronos OpenCL-ICD-Loader** ([github.com/KhronosGroup/OpenCL-ICD-Loader](https://github.com/KhronosGroup/OpenCL-ICD-Loader)): The reference implementation maintained by Khronos; wire-compatible with ocl-icd.

Both loaders implement the same ICD protocol. The dispatch mechanism relies on a subtle invariant: **every OpenCL opaque object (`cl_platform_id`, `cl_device_id`, `cl_context`, `cl_command_queue`, `cl_program`, `cl_kernel`, `cl_mem`, `cl_event`) begins with a pointer to a dispatch table as its first field.** When the ICD loader hands a handle to the caller, the handle's first word is a pointer to the platform's function pointers. The loader's stub implementations for every `clXxx()` call simply dereference this pointer and tail-call through the table.

This design has a critical performance implication: dispatching through the ICD table costs exactly one pointer dereference and one indirect call — essentially free compared to the kernel launch or memory operation that follows. There is no hash lookup, no string comparison, and no lock acquisition. The trick is that the ICD object and the underlying implementation object are the **same allocation**: the ICD inserts its dispatch pointer at offset zero of whatever struct the implementation uses for `cl_platform_id`. This is why the `_cl_platform_id` struct definition in the ICD header must always be opaque — user code is forbidden from dereferencing it directly. Any attempt to allocate or copy a `cl_platform_id` by value would break this invariant. The dispatch table itself is shared across all objects returned by the same platform (it is a static singleton per platform library), so the only per-object cost is one pointer field of storage.

```c
/* Conceptual ICD dispatch table (from OpenCL-ICD-Loader/loader/icd_dispatch.h) */
struct _cl_icd_dispatch {
    /* OpenCL 1.0 */
    void *clGetPlatformIDs;
    void *clGetPlatformInfo;
    void *clGetDeviceIDs;
    void *clGetDeviceInfo;
    void *clCreateContext;
    void *clCreateCommandQueue;
    void *clCreateBuffer;
    void *clCreateProgramWithSource;
    void *clBuildProgram;
    void *clCreateKernel;
    void *clSetKernelArg;
    void *clEnqueueNDRangeKernel;
    /* ... 200+ entries total through OpenCL 3.0 */
};

/* Every CL object starts with this pointer */
struct _cl_platform_id {
    struct _cl_icd_dispatch *dispatch;
    /* platform-private data follows */
};
```
[Source: KhronosGroup/OpenCL-ICD-Loader](https://github.com/KhronosGroup/OpenCL-ICD-Loader)

### ICD Registration

Platforms register themselves by installing a `.icd` file under `/etc/OpenCL/vendors/`. Each file is a one-line text file containing the path (or soname) of the platform's shared library:

```
# /etc/OpenCL/vendors/mesa.icd
/usr/lib/x86_64-linux-gnu/libMesaOpenCL.so.1

# /etc/OpenCL/vendors/intel.icd
/usr/lib/x86_64-linux-gnu/intel-opencl/libigdrcl.so

# /etc/OpenCL/vendors/amdocl64.icd
libamdocl64.so
```

The ICD loader also respects `OCL_ICD_VENDORS` (override the vendor directory path or filename) and `OPENCL_VENDOR_PATH` (override the base directory). This lets developers test local builds:

```bash
OCL_ICD_VENDORS=/path/to/my-dev-build.icd clinfo
```

### Platform Enumeration

```c
#include <CL/cl.h>

cl_uint num_platforms;
clGetPlatformIDs(0, NULL, &num_platforms);  /* query count */

cl_platform_id platforms[8];
clGetPlatformIDs(num_platforms, platforms, NULL);

for (cl_uint i = 0; i < num_platforms; i++) {
    char name[256];
    clGetPlatformInfo(platforms[i], CL_PLATFORM_NAME, sizeof(name), name, NULL);
    char version[256];
    clGetPlatformInfo(platforms[i], CL_PLATFORM_VERSION, sizeof(version), version, NULL);
    printf("Platform %u: %s [%s]\n", i, name, version);

    cl_uint num_devices;
    clGetDeviceIDs(platforms[i], CL_DEVICE_TYPE_ALL, 0, NULL, &num_devices);
    cl_device_id devices[8];
    clGetDeviceIDs(platforms[i], CL_DEVICE_TYPE_ALL, num_devices, devices, NULL);

    for (cl_uint j = 0; j < num_devices; j++) {
        char devname[256];
        clGetDeviceInfo(devices[j], CL_DEVICE_NAME, sizeof(devname), devname, NULL);
        printf("  Device %u: %s\n", j, devname);
    }
}
```

### clinfo

The `clinfo` tool ([github.com/Oblomov/clinfo](https://github.com/Oblomov/clinfo)) queries every registered platform and device and dumps the full capability set. Running `clinfo --list` gives a compact overview; `clinfo` with no flags prints hundreds of lines of device properties including CL version, extension strings, work-group limits, memory capacities, and sub-device support. It is the first diagnostic tool to reach for when troubleshooting an OpenCL setup.

---

## 3. Mesa rusticl — OpenCL 3.0 via Gallium

### Overview

rusticl is Mesa's OpenCL implementation, written in Rust, that sits as a Gallium frontend alongside the existing OpenGL/EGL state trackers. Its codebase lives at [`src/gallium/frontends/rusticl/`](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/frontends/rusticl) in the Mesa repository and is authored primarily by Karol Herbst (Red Hat). [Source: Mesa GitLab](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/frontends/rusticl)

The strategic value of rusticl is that any Mesa Gallium driver — **radeonsi** (AMD desktop), **iris** (Intel Gen8+), **llvmpipe** (CPU software renderer), **softpipe** — automatically gains an OpenCL interface without per-driver OpenCL work, because rusticl operates entirely through the `pipe_driver` / `pipe_context` abstraction layer. The Gallium pipe interface is the same API that the OpenGL state tracker (`st/mesa`) already uses, so rusticl re-uses all the memory management, command submission, and shader infrastructure that these drivers already implement.

### Architecture

```
OpenCL application
       │
       ▼
  libOpenCL.so (ocl-icd loader)
       │  (dispatch through ICD table)
       ▼
  libMesaOpenCL.so  ← rusticl (Rust)
       │
       ▼
  RusticlContext  →  pipe_context  (Gallium abstraction)
       │                    │
       │                    ▼
       │             radeonsi / iris / llvmpipe
       │
       ▼
  OpenCL C source
       │
  Clang/LLVM (libclc for builtins)
       │  → SPIR-V → LLVM IR → NIR
       ▼
  Gallium driver's NIR-to-ISA backend
       │
       ▼
  GPU ISA (GCN/RDNA shader ISA, GEN ISA, etc.)
```

The kernel compilation path: OpenCL C source → Clang frontend (with `--target=spirv64-unknown-unknown`) → SPIR-V → `spirv_to_nir()` → NIR optimizations → Gallium driver's `nir_to_tgsi()` or native NIR backend → GPU machine code. This is architecturally cleaner than maintaining a separate OpenCL compiler per driver.

### Enabling rusticl

rusticl devices are not advertised by default (for stability during development). Distributions may pre-enable specific drivers via the `gallium-rusticl-enable-drivers` meson build option. At runtime, `RUSTICL_ENABLE` controls which Gallium drivers expose OpenCL devices:

```bash
# Enable radeonsi device 0 as an OpenCL platform
RUSTICL_ENABLE=radeonsi:0 clinfo

# Enable iris (Intel) and llvmpipe simultaneously
RUSTICL_ENABLE=iris,llvmpipe clinfo

# Enable all available Gallium drivers
RUSTICL_ENABLE=auto clinfo
```

As of Mesa 25.x, some distributions (Fedora, openSUSE Tumbleweed) pre-enable radeonsi and iris by default via a distribution-level meson option.

### Build Requirements

Building rusticl requires:
- Rust 1.82+ (`rustc`, `rustfmt`, `bindgen 0.71.1+`)
- LLVM 15+ (LLVM 19 recommended) with `libclc` and `-DLLVM_ENABLE_DUMP=ON`
- SPIR-V Tools and SPIR-V-LLVM-Translator matching the LLVM version
- Meson 1.7.0+

Configure Mesa with:

```bash
meson setup build \
  -Dgallium-rusticl=true \
  -Dllvm=enabled \
  -Drust_std=2021 \
  -Dgallium-drivers=radeonsi,iris,llvmpipe
```

### Conformance and Extensions

rusticl achieved official Khronos OpenCL 3.0 conformance, with the submission running on **Intel Gen12 Xe (iris Gallium driver)**. [Source: Phoronix/en.ubunlog.com](https://en.ubunlog.com/rusticl-is-already-certified-and-is-compatible-with-opencl-3-0/) At XDC 2025 in Vienna, Karol Herbst presented a summary of rusticl's trajectory, highlighting several major achievements completed in 2024–2025:

- **Shared Virtual Memory (SVM)**: Coarse-grain SVM is now implemented, allowing host pointer sharing between CPU and GPU without explicit copy.
- **SPIR-V 1.6 support**: rusticl can consume the latest SPIR-V version, enabling interoperability with tools like DPC++ and SYCL compilers that emit SPIR-V 1.6.
- **Async and parallel program compilation**: Multiple `cl_program` objects can now be compiled in parallel, reducing application startup time for programs with many kernels.
- **Extended extension list**: As of Mesa 24.x–25.x, `CL_DEVICE_EXTENSIONS` on a rusticl device includes:

```
cl_khr_global_int32_base_atomics cl_khr_global_int32_extended_atomics
cl_khr_local_int32_base_atomics cl_khr_local_int32_extended_atomics
cl_khr_int64_base_atomics cl_khr_int64_extended_atomics
cl_khr_byte_addressable_store cl_khr_fp64
cl_khr_3d_image_writes cl_khr_image2d_from_buffer
cl_khr_depth_images cl_khr_mipmap_image
cl_khr_subgroups cl_khr_spirv_no_integer_wrap_decoration
cl_khr_gl_sharing cl_khr_external_memory
cl_khr_extended_async_copies
```

**Memory model mapping**: On AMD radeonsi, `CL_LOCAL` memory maps to LDS (Local Data Share); on Intel iris, it maps to SLM (Shared Local Memory). The `pipe_shader_cap_max_inputs` and related Gallium caps inform rusticl of per-driver workgroup constraints.

**Debugging**: `RUSTICL_DEBUG=1` (or `RUSTICL_DEBUG=program` / `RUSTICL_DEBUG=nir`) enables verbose logging of compilation events, NIR dumps, and kernel dispatch.

## 3.1 Legacy: Clover — Mesa's Pre-rusticl OpenCL

Clover is Mesa's original OpenCL state tracker, targeting Gallium3D drivers (radeonsi, iris, softpipe). It was the primary Mesa OpenCL implementation until rusticl superseded it in Mesa 22.x. Most distributions still ship `libMesaOpenCL.so` pointing to rusticl on recent Mesa builds; on older systems this file was Clover.

```bash
# Clover (if still present on an older system):
ls /usr/lib/x86_64-linux-gnu/libMesaOpenCL.so*
# Clover targets OpenCL 1.2 with limited 2.0 features
```

**Architecture**:

```
OpenCL host code (clCreateBuffer, clEnqueueNDRange, ...)
    │
    ▼
Clover (src/gallium/frontends/clover/)
    │ calls pipe_context directly
    ▼
Gallium pipe driver (radeonsi / iris / softpipe)
    │
    ▼
GPU ISA
```

Clover compiles OpenCL C via LLVM's OpenCL front-end → LLVM IR → SPIR-V (via `llvm-spirv`) → NIR → GPU ISA. The compilation path is functionally equivalent to rusticl's but implemented in C++ rather than Rust and limited to OpenCL 1.2 feature level.

**Limitations that drove rusticl adoption**:
- OpenCL 1.2 only — no 3.0 feature level
- No Shared Virtual Memory (SVM/USM)
- Incomplete `cl_khr_gl_sharing` on all targets
- `cl_khr_fp64` unavailable on softpipe

rusticl is the recommended Mesa OpenCL implementation for all new deployments. Clover receives maintenance fixes only and is expected to be removed from Mesa once rusticl reaches full CTS conformance on all supported drivers.

[Source: Mesa clover directory](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/frontends/clover)

---

## 4. Intel OpenCL: NEO / Compute Runtime

### Project Overview

Intel's open-source OpenCL (and Level Zero) implementation is the **Intel Graphics Compute Runtime for oneAPI**, colloquially called **NEO** ([github.com/intel/compute-runtime](https://github.com/intel/compute-runtime)). NEO is the production OpenCL ICD for all Intel GPUs from Gen8 (Broadwell) through the current Xe3P architecture (Nova Lake, Crescent Island). It is distributed under the MIT license and ships as `intel-opencl-icd` on Debian/Ubuntu and Fedora.

As of the 26.01 release series (January 2026), NEO supports:

| Architecture Family | Example Products | OpenCL Version |
|---|---|---|
| Gen8 (Broadwell) | Core 5th Gen | 3.0 |
| Gen9/9.5 (Skylake, Kaby Lake, Coffee Lake) | Core 6th–8th Gen | 3.0 |
| Gen11 (Ice Lake, Elkhart Lake) | Core 10th Gen | 3.0 |
| Gen12 LP (Tiger Lake, Rocket Lake, Alder Lake, Raptor Lake) | Core 11th–13th Gen | 3.0 |
| Xe HPG (Alchemist / Arc A-series) | Arc A770, A380 | 3.0 |
| Xe HPC (Ponte Vecchio) | Data center GPU Max | 3.0 |
| Xe2 LPG/HPG (Meteor Lake, Lunar Lake, Arrow Lake, Battlemage) | Core Ultra, Arc B-series | 3.0 |
| Xe3P (Panther Lake, Nova Lake, Crescent Island) | Next-gen, 2025+ | 3.0 (early) |

[Source: intel/compute-runtime README](https://github.com/intel/compute-runtime/blob/master/README.md)

### Architecture

NEO is structured around three main layers:

```
OpenCL API call
       │
       ▼
  NEO OpenCL Runtime  (libigdrcl.so / intel-opencl-icd)
  ┌─────────────────────────────────────────────────┐
  │  ClDevice  →  ClCommandQueue                    │
  │       └──►  CommandStreamReceiver (CSR)         │
  │                    │                            │
  │              Batch Buffer construction          │
  └─────────────────────────────────────────────────┘
       │
       ▼
  Intel Graphics Compiler (IGC)  ←─  libigc1, libigdfcl1
  (OpenCL C / SPIR-V → GEN ISA)
       │
       ▼
  i915 / Xe kernel driver  (DRM command submission)
```

The **CommandStreamReceiver** (CSR) is the core of NEO's scheduling: it builds GPU command streams (batch buffers), manages residency of `cl_mem` allocations, and submits work to the kernel driver via `execbuf` (i915) or the Xe DRM driver's job-submission interface.

**Kernel compilation** flows through the **Intel Graphics Compiler** ([github.com/intel/intel-graphics-compiler](https://github.com/intel/intel-graphics-compiler)), a standalone LLVM-based compiler:
- Input: OpenCL C source or SPIR-V bitcode
- Frontend: Clang with Intel OpenCL extensions
- Optimization: LLVM middle-end + IGC-specific passes (GenX backend)
- Output: ELF binary containing GEN ISA shader programs

The packages involved: `libigc1` (compiler runtime), `libigdfcl1` (frontend/builtins), `intel-opencl-icd` (NEO OpenCL ICD).

### Notable Intel-Specific Extensions

**`cl_intel_subgroups`**: Exposes SIMD lane-level programming inside a workgroup. A subgroup is a set of work-items that execute in lock-step on the EU (Execution Unit) SIMD lanes. This is the Intel-specific predecessor (and superset) of the portable `cl_khr_subgroups` extension. The Intel extension's primary value is the **block read/write** and **shuffle** operations that map directly to Gen EU SIMD instructions:

```c
__kernel void subgroup_shuffle_demo(__global const float *src,
                                    __global float *dst) {
    /* intel_sub_group_shuffle: broadcast lane i's value to all lanes */
    float my_val = src[get_global_id(0)];
    /* Broadcast lane 0's value to every lane in the subgroup */
    float lane0_val = intel_sub_group_shuffle(my_val, 0);
    dst[get_global_id(0)] = lane0_val;
}

__kernel void subgroup_block_read_demo(__global const uint *src,
                                       __global uint *dst) {
    /* intel_sub_group_block_read: coalesced 4-byte load per lane
     * using a single SIMD-width read instruction              */
    uint v = intel_sub_group_block_read(src + get_sub_group_id() * get_sub_group_size());
    dst[get_global_id(0)] = v;
}
```

For portable reductions across subgroup lanes, use `sub_group_reduce_add(val)` from the standard `cl_khr_subgroups` extension, which is available on Intel, AMD (RDNA2+), and rusticl.

**`cl_intel_unified_shared_memory` (USM)**: Intel's extension (predating the draft Khronos `cl_khr_unified_svm`) provides a pointer-based memory model where allocations are plain C pointers that the hardware migrates transparently. Unlike `cl_mem` handles, USM pointers support pointer arithmetic and can be embedded in data structures like linked lists.

USM defines three allocation types with distinct migration semantics:

- **Host allocations** (`clHostMemAllocINTEL`): Memory physically resident on the host (CPU) but accessible by the device. Device accesses cross the PCIe bus; useful for infrequently-written, host-side data.
- **Device allocations** (`clDeviceMemAllocINTEL`): Memory resident in device-local VRAM. The host can access it via implicit or explicit migration. Fast for GPU-intensive kernels on discrete GPUs.
- **Shared allocations** (`clSharedMemAllocINTEL`): Migrates automatically between host and device. On Intel integrated graphics (UMA), host and GPU share the same physical DRAM, making shared allocations zero-copy; on discrete GPU hardware, the runtime pages memory across PCIe on demand.

```c
/* cl_intel_unified_shared_memory USM allocation */
cl_mem_properties_intel props[] = {0};
void *host_ptr   = clHostMemAllocINTEL(ctx, props, size, 0, &err);
void *device_ptr = clDeviceMemAllocINTEL(ctx, device, props, size, 0, &err);
void *shared_ptr = clSharedMemAllocINTEL(ctx, device, props, size, 0, &err);

/* Pass as kernel argument by pointer value (not a cl_mem handle) */
clSetKernelArgMemPointerINTEL(kernel, 0, device_ptr);

/* Prefetch shared allocation to device before kernel launch */
clEnqueueMemPrefetchINTEL(queue, shared_ptr, size, device, 0, NULL, NULL);
```
[Source: Khronos cl_intel_unified_shared_memory spec](https://registry.khronos.org/OpenCL/extensions/intel/cl_intel_unified_shared_memory.html)

**`cl_intel_bfloat16`** (Xe HPC / XeHPC): Adds bfloat16 data type support for ML inference workloads on Intel Data Center GPU Max (Ponte Vecchio).

---

## 5. AMD ROCm OpenCL (CLR)

### Stack Structure

AMD's open-source OpenCL implementation is shipped as part of the **ROCm** software stack. Starting with ROCm 5.6, the formerly separate `ROCm-OpenCL-Runtime` and `hipamd` repositories were merged into a single repository: **CLR** (Compute Language Runtime, [github.com/ROCm/clr](https://github.com/ROCm/clr)). CLR implements both HIP and OpenCL on top of the common **ROCclr** (Radeon Open Compute Common Language Runtime) layer.

The runtime layers:

```
OpenCL application
       │
       ▼
  libamdocl64.so  (registered via /etc/OpenCL/vendors/amdocl64.icd)
       │  (CLR OpenCL ICD)
       ▼
  ROCclr  (common device/command abstraction)
       │
       ▼
  HSA Runtime (ROCr)  →  KFD (kernel fusion driver, amdgpu.ko)
```

The **KFD** (Kernel Fusion Driver) is the compute-specific interface in `amdgpu.ko` that exposes queue creation, signal/wait operations, and memory management for compute (as opposed to the `amdgpu_gem_*` paths used for graphics). ROCclr submits work to HSA queues which the KFD maps to SDMA and compute rings on the GPU.

### Kernel Compilation

AMD's ROCm OpenCL compiles kernels using LLVM with the **AMDGCN** target:

```
OpenCL C source
       │
  Clang (--target=amdgcn-amd-amdhsa -mcpu=gfx1100)
       │
  AMDGPU LLVM backend
       │  → ELF containing AMDGPU ISA
       ▼
  HSA Code Object (hsa_code_object_t / .hsaco format)
```

The compilation happens at kernel load time (`clBuildProgram`) or can use pre-compiled SPIR-V via `clCreateProgramWithIL`. Compiled kernels are cached by the runtime in `~/.cache/amdocl/` to avoid recompilation on subsequent runs.

### OpenCL Version and AMD Extensions

As of ROCm 7.x (2025), the AMD OpenCL runtime targets **OpenCL 2.0** — it does not yet claim full OpenCL 3.0 conformance. This is a significant differentiator: rusticl, Intel NEO, and pocl all have Khronos-certified OpenCL 3.0 conformance, while AMD's production OpenCL ICD is currently 2.0. AMD's recommendation for performance-critical AMD-specific work is HIP; OpenCL 2.0 remains available for cross-platform portability.

Key AMD-specific extensions:

**`cl_amd_device_attribute_query`**: Exposes hardware characteristics beyond the standard `clGetDeviceInfo`:

```c
cl_uint wavefront_width;
clGetDeviceInfo(device, CL_DEVICE_WAVEFRONT_WIDTH_AMD,
                sizeof(wavefront_width), &wavefront_width, NULL);
/* Returns 64 on CDNA/older GCN, 32 or 64 on RDNA (configurable) */

cl_uint max_cu;
clGetDeviceInfo(device, CL_DEVICE_MAX_COMPUTE_UNITS, sizeof(max_cu), &max_cu, NULL);
```

**`cl_amd_media_ops` / `cl_amd_media_ops2`**: Video codec and image processing operations using GCN/RDNA media instructions (SAD, bit manipulation for video encoding).

**`cl_khr_subgroups`**: Available on RDNA2+; subgroup sizes of 32 or 64 depending on the kernel compilation mode.

### RDNA/CDNA Specifics

- **Wavefront size**: CDNA (MI series, data center) uses 64-wide wavefronts. RDNA 1–3 supports both 32-wide and 64-wide wavefronts; the default is 32-wide. Use `cl_amd_device_attribute_query` to query at runtime.
- **Async compute**: AMD GPUs have multiple compute queues that can run independently of the graphics queue. ROCm OpenCL uses these for command-queue parallelism.

### Installation

```bash
# Ubuntu 22.04 / 24.04 — AMD ROCm 7.x
sudo apt install rocm-opencl-runtime

# Verify
clinfo --list
# Should show: AMD Accelerated Parallel Processing (amdocl)
```

The proprietary `amdgpu-pro-opencl` package remains available as a legacy alternative but is not recommended for new deployments on kernels with `amdgpu.ko` >= 6.x.

---

## 5.1 NVIDIA OpenCL on Linux

NVIDIA provides a proprietary OpenCL 3.0 implementation bundled with the standard NVIDIA driver package. No separate installation is required on systems with an NVIDIA driver installed.

```bash
# NVIDIA OpenCL ships with the NVIDIA driver:
ls /usr/lib/x86_64-linux-gnu/libnvidia-opencl.so*
# ICD registration:
cat /etc/OpenCL/vendors/nvidia.icd
# → libnvidia-opencl.so.1

# Verify platform detection:
clinfo --list | grep NVIDIA
# Platform: NVIDIA CUDA → CL_DEVICE_TYPE_GPU → Tesla/RTX ...
```

NVIDIA compiles OpenCL kernels via NVRTC (NVIDIA Runtime Compilation) → PTX assembly → GPU ISA. The OpenCL ICD is fully conformant to OpenCL 3.0 on all NVIDIA GPUs from Kepler onwards. On Volta+ hardware, Tensor Core operations are not exposed through OpenCL; use CUDA for those workloads.

[Source: NVIDIA OpenCL SDK](https://developer.nvidia.com/opencl)

---

## 6. CPU/Software OpenCL: pocl

### Overview

**pocl** (Portable Computing Language, [portablecl.org](https://portablecl.org/)) is an open-source, LLVM-based OpenCL implementation targeting CPUs and other non-GPU accelerators. Its architecture is explicitly compiler-first: pocl uses **Clang as the OpenCL C frontend** and **LLVM as the kernel compiler**, making it trivially portable to any ISA for which an LLVM backend exists.

As of **pocl 7.1** (released October 2025), pocl supports:
- **OpenCL 3.0 conformance**: Officially certified by Khronos for the CPU and Intel Level Zero GPU backends (conformance achieved January 2025). [Source: portablecl.org](https://portablecl.org/)
- **CPU targets**: x86-64 (SSE4.2, AVX, AVX-512), AArch64 (NEON, SVE), RISC-V (RVV — RISC-V Vector Extension)
- **Intel GPU via Level Zero**: pocl's `level0` device driver wraps Intel's Level Zero API, providing an alternative Intel GPU path alongside NEO
- **NVIDIA GPU via CUDA**: The `cuda` device driver translates OpenCL kernels to PTX through LLVM's NVPTX backend and submits them via libcuda
- **Remote backend**: Distributed OpenCL execution across a network of pocl instances

pocl 7.1 also adds support for **LLVM 21** for CPU targets, **Command Buffers** (experimental), and additional Tensor extensions. [Source: Phoronix PoCL 7.1](https://www.phoronix.com/news/PoCL-7.1-Released)

### CPU Backend Architecture

```
OpenCL C kernel source
       │
  Clang (--target=x86_64-pc-linux-gnu)
       │  (with OpenCL 3.0 builtins via pocl's libclc fork)
       ▼
  LLVM IR
       │
  pocl work-group loop generation
  (horizontal autovectorization across work-items)
       │
  LLVM vectorizer (SLP + loop vectorizer, AVX-512 if available)
       │
  x86-64 / AArch64 / RISC-V machine code (in-memory JIT or cached)
       │
  pthreads worker pool (one thread per CU, work-groups distributed)
```

The key insight is that pocl converts each kernel's inner loop over work-items into a SIMD-vectorized CPU loop, achieving good throughput on CPUs without per-platform GPU-specific code.

### Registration and Configuration

```bash
# Install pocl
sudo apt install pocl-opencl-icd  # Debian/Ubuntu
sudo dnf install pocl              # Fedora

# pocl registers itself at /etc/OpenCL/vendors/pocl.icd
cat /etc/OpenCL/vendors/pocl.icd
# libpocl.so.2

# Debug tracing
POCL_DEBUG=all clinfo

# Restrict to pocl only (exclude GPU ICDs) for testing
OCL_ICD_VENDORS=/etc/OpenCL/vendors/pocl.icd clinfo
```

**Kernel caching**: pocl caches compiled kernels under `~/.cache/pocl/` using a hash of the kernel source + build options + CPU target triple. This eliminates recompilation on repeated runs.

### CUDA Backend

The `pocl-cuda` backend translates OpenCL to PTX via LLVM's NVPTX backend:

```bash
# Build pocl with CUDA support
cmake -DWITH_CUDA=ON ..

# Runtime: CUDA devices appear alongside CPU
POCL_DEBUG=cuda clinfo
```

This enables OpenCL applications to run on NVIDIA GPUs without the proprietary CUDA OpenCL ICD — useful for portability testing and for environments where only the CUDA toolkit (not the full NVIDIA OpenCL ICD) is available.

### Use Cases

- **Portability and conformance testing**: Run the Khronos OpenCL CTS against the CPU backend to verify application correctness before deploying to a GPU.
- **CPU fallback**: Applications that probe `clGetDeviceIDs(..., CL_DEVICE_TYPE_GPU, ...)` and fall back to CPU when no GPU is found get accelerated CPU execution via SIMD vectorization.
- **Embedded and cross-compilation**: pocl's build system supports cross-compiling kernels for AArch64 or RISC-V targets.

---

## 7. OpenCL Programming Model

### Objects and Lifecycle

The canonical OpenCL object hierarchy:

```
cl_platform_id        ← registered ICD
    └── cl_device_id  ← GPU/CPU device
          └── cl_context  ← shared state (devices + memory)
                ├── cl_command_queue  ← ordered/unordered work submission
                ├── cl_mem            ← buffer or image allocation
                ├── cl_program        ← compiled kernel collection
                │       └── cl_kernel ← single kernel + bound args
                └── cl_event          ← dependency / profiling handle
```

### Memory Allocation

**Buffers** (`cl_mem` with `CL_MEM_OBJECT_BUFFER`): Raw byte ranges in device global memory.

```c
/* Allocate 64 MiB device buffer */
cl_int err;
cl_mem buf = clCreateBuffer(ctx,
                             CL_MEM_READ_WRITE,
                             64 * 1024 * 1024,
                             NULL,  /* no host pointer */
                             &err);

/* Copy from host */
clEnqueueWriteBuffer(queue, buf, CL_TRUE /*blocking*/,
                     0, size, host_ptr, 0, NULL, NULL);
```

**Images** (`cl_mem` with image type): Texture-backed memory with format/tiling managed by the driver:

```c
cl_image_format fmt = { CL_RGBA, CL_UNORM_INT8 };
cl_image_desc desc = {
    .image_type   = CL_MEM_OBJECT_IMAGE2D,
    .image_width  = 1920,
    .image_height = 1080,
};
cl_mem img = clCreateImage(ctx, CL_MEM_READ_ONLY,
                            &fmt, &desc, NULL, &err);
```

### Memory Address Spaces

OpenCL kernels operate across four address spaces:

| Qualifier | Location | Scope |
|---|---|---|
| `__global` | Device global memory (VRAM / system RAM) | All work-items |
| `__local` | On-chip LDS (AMD) / SLM (Intel) — ~32–128 KiB per CU | Work-group |
| `__constant` | Read-only constant cache | All work-items (read-only) |
| `__private` | Per-lane registers / stack | Single work-item |

```c
__kernel void mat_mul(__global const float *A,
                      __global const float *B,
                      __global       float *C,
                      __local        float *scratch,
                      int N) {
    int row = get_global_id(0);
    int col = get_global_id(1);
    int lid = get_local_id(0);

    /* Load tile into __local for fast reuse */
    scratch[lid] = A[row * N + lid];
    barrier(CLK_LOCAL_MEM_FENCE);  /* synchronise within work-group */

    float acc = 0.0f;
    for (int k = 0; k < N; k++)
        acc += scratch[k] * B[k * N + col];
    C[row * N + col] = acc;
}
```

### Kernel Dispatch and the NDRange Execution Model

OpenCL's execution model is built around three nested concepts that every GPU compute developer needs to understand precisely:

- **Work-item**: The fundamental unit of parallelism — a single invocation of the kernel function. Each work-item has its own private registers and stack. On AMD hardware, 32 or 64 work-items execute in lock-step as a **wavefront**; on Intel hardware, 8–32 work-items form a **subgroup** on a single EU SIMD lane.

- **Work-group**: A rectangular block of work-items (up to three dimensions) that share a `__local` memory region and can synchronise with each other using `barrier()`. All work-items in a work-group are guaranteed to execute on the same Compute Unit (CU) / Execution Unit cluster. The work-group size is bounded by `CL_DEVICE_MAX_WORK_GROUP_SIZE` (typically 256–1024 work-items on current GPUs) and must divide evenly into the global work size.

- **NDRange**: The total iteration space — a 1D, 2D, or 3D grid of work-items. `clEnqueueNDRangeKernel` decomposes this grid into work-groups and dispatches them across the GPU's available CUs. Work-groups execute in any order; there is no inter-work-group synchronisation within a single kernel dispatch (use multiple kernel enqueues with event dependencies for that).

Inside a kernel, the built-in functions provide the position coordinates:

```c
get_global_id(dim)   /* absolute position in the NDRange (0 to global_size-1) */
get_local_id(dim)    /* position within the work-group (0 to local_size-1) */
get_group_id(dim)    /* which work-group (0 to num_groups-1) */
get_global_size(dim) /* total global work size */
get_local_size(dim)  /* work-group size */
get_num_groups(dim)  /* number of work-groups (= global_size / local_size) */
```

```c
size_t global_work_size[2] = { 1024, 1024 };  /* total threads */
size_t local_work_size[2]  = { 16, 16 };      /* work-group (256 work-items) */

cl_event evt;
clEnqueueNDRangeKernel(queue,
                        kernel,
                        2,                   /* work dimensions */
                        NULL,                /* global work offset */
                        global_work_size,
                        local_work_size,
                        0, NULL,             /* wait list */
                        &evt);              /* output event */
```

### Event-Based Dependency Chains

`cl_event` objects are OpenCL's equivalent of timeline semaphores — they express happens-before relationships between operations on the same or different queues:

```c
cl_event upload_done, kernel_done;

/* Async upload */
clEnqueueWriteBuffer(queue, buf, CL_FALSE, 0, size, host, 0, NULL, &upload_done);

/* Kernel waits for upload */
clEnqueueNDRangeKernel(queue, kernel, 1, NULL, &gws, &lws,
                        1, &upload_done, &kernel_done);

/* Readback waits for kernel */
clEnqueueReadBuffer(queue, out, CL_FALSE, 0, size, result,
                    1, &kernel_done, NULL);

clFinish(queue);  /* block host until all complete */
```

Out-of-order queues (`CL_QUEUE_OUT_OF_ORDER_EXEC_MODE_ENABLE`) let the GPU scheduler reorder independent operations:

```c
cl_command_queue_properties props = CL_QUEUE_OUT_OF_ORDER_EXEC_MODE_ENABLE
                                  | CL_QUEUE_PROFILING_ENABLE;
cl_command_queue ooo_queue = clCreateCommandQueueWithProperties(
    ctx, device,
    (cl_queue_properties[]){ CL_QUEUE_PROPERTIES, props, 0 },
    &err);
```

### Shared Virtual Memory (SVM) and Unified Shared Memory (USM)

**SVM** (OpenCL 2.0, optional in 3.0) allows pointers to be shared between host and device without explicit `cl_mem` handles:

```c
/* Coarse-grain SVM — allocate, then explicitly sync */
void *svm_ptr = clSVMAlloc(ctx,
                            CL_MEM_READ_WRITE | CL_MEM_SVM_FINE_GRAIN_BUFFER,
                            size, 0);
clSetKernelArgSVMPointer(kernel, 0, svm_ptr);
clEnqueueSVMMap(queue, CL_TRUE, CL_MAP_WRITE, svm_ptr, size, 0, NULL, NULL);
memcpy(svm_ptr, host_data, size);
clEnqueueSVMUnmap(queue, svm_ptr, 0, NULL, NULL);
```

**USM** via `cl_intel_unified_shared_memory` (Intel extension; a draft Khronos `cl_khr_unified_svm` is in development) goes further, treating all allocations as pointers with implicit migration — no `clEnqueueSVMMap`/`Unmap` required for shared allocations on UMA (Unified Memory Architecture) Intel platforms.

### SPIR-V Ingestion

Applications can bypass OpenCL C source entirely and submit pre-compiled SPIR-V:

```c
/* Load SPIR-V binary */
FILE *f = fopen("kernel.spv", "rb");
fseek(f, 0, SEEK_END); size_t spv_size = ftell(f); rewind(f);
unsigned char *spv = malloc(spv_size);
fread(spv, 1, spv_size, f); fclose(f);

/* Create program from SPIR-V IL */
cl_program prog = clCreateProgramWithIL(ctx, spv, spv_size, &err);
clBuildProgram(prog, 1, &device, NULL, NULL, NULL);
cl_kernel k = clCreateKernel(prog, "my_kernel", &err);
```
[Source: Khronos Offline SPIR-V Compilation Guide](https://www.khronos.org/blog/offline-compilation-of-opencl-kernels-into-spir-v-using-open-source-tooling)

This is the preferred deployment model for shipping kernels without source: compile offline with `clang --target=spirv64-unknown-unknown` or a SYCL/DPC++ compiler, then ingest at runtime. SPIR-V 1.6 is supported by rusticl (Mesa 25.x+) and Intel NEO (IGC).

---

## 8. OpenCL–OpenGL Interop and DMA-BUF

### cl_khr_gl_sharing

The `cl_khr_gl_sharing` extension ([Khronos registry](https://registry.khronos.org/OpenCL/sdk/3.0/docs/man/html/cl_khr_gl_sharing.html)) allows OpenGL buffer objects, textures, and renderbuffers to be directly accessed as OpenCL `cl_mem` objects, eliminating a CPU round-trip through host memory. The objects share the underlying GPU memory allocation.

**Context creation with GL sharing**:

```c
/* EGL context sharing (Wayland/EGL path) */
cl_context_properties props[] = {
    CL_GL_CONTEXT_KHR,   (cl_context_properties)eglGetCurrentContext(),
    CL_EGL_DISPLAY_KHR,  (cl_context_properties)eglGetCurrentDisplay(),
    CL_CONTEXT_PLATFORM, (cl_context_properties)platform,
    0
};
cl_context ctx = clCreateContext(props, 1, &device, NULL, NULL, &err);
```

**Sharing objects**:

```c
/* Share an OpenGL VBO as a CL buffer */
GLuint vbo;
glGenBuffers(1, &vbo);
glBindBuffer(GL_ARRAY_BUFFER, vbo);
glBufferData(GL_ARRAY_BUFFER, size, NULL, GL_DYNAMIC_DRAW);

cl_mem cl_buf = clCreateFromGLBuffer(ctx, CL_MEM_READ_WRITE, vbo, &err);

/* Share an OpenGL texture as a CL image */
GLuint tex;
glGenTextures(1, &tex);
glBindTexture(GL_TEXTURE_2D, tex);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA8, width, height, 0,
             GL_RGBA, GL_UNSIGNED_BYTE, NULL);
cl_mem cl_img = clCreateFromGLTexture(ctx, CL_MEM_READ_WRITE,
                                       GL_TEXTURE_2D, 0, tex, &err);
```

**Acquire/release for OpenCL access**:

```c
/* Acquire: transition ownership from GL to CL */
clEnqueueAcquireGLObjects(queue, 1, &cl_img, 0, NULL, NULL);

/* Process in OpenCL kernel */
clEnqueueNDRangeKernel(queue, edge_detect_kernel, 2,
                        NULL, global, local, 0, NULL, NULL);

/* Release: return ownership to GL for rendering */
clEnqueueReleaseGLObjects(queue, 1, &cl_img, 0, NULL, NULL);
clFinish(queue);

/* Now GL can use the texture for rendering */
```

**Synchronization with `cl_khr_gl_event`**: Creates a `cl_event` from a GL sync object, enabling fine-grained cross-API synchronization without `glFinish()`:

```c
GLsync gl_sync = glFenceSync(GL_SYNC_GPU_COMMANDS_COMPLETE, 0);
cl_event cl_ev = clCreateEventFromGLsyncKHR(ctx, gl_sync, &err);
/* Pass cl_ev as a wait event for the acquire */
clEnqueueAcquireGLObjects(queue, 1, &cl_buf, 1, &cl_ev, NULL);
```

### Note on Wayland/EGL Compatibility

A caveat documented in 2026: on Wayland compositors, `cl_khr_gl_sharing` requires EGL context sharing (`CL_EGL_DISPLAY_KHR` / `CL_GL_CONTEXT_KHR`), not GLX. Applications that hard-code the GLX path (`CL_GLX_DISPLAY_KHR`) will fail. This is an active portability concern as GLX usage on Wayland (via XWayland) differs from native EGL.

### DMA-BUF External Memory Import

For interoperability beyond OpenGL — particularly for V4L2 camera capture → OpenCL processing → display pipelines — the modern approach uses the **`cl_khr_external_memory`** and **`cl_khr_external_memory_dma_buf`** extensions ([Khronos blog, 2021](https://www.khronos.org/blog/khronos-releases-opencl-3.0-extensions-for-neural-network-inferencing-and-opencl-vulkan-interop)):

```c
/* Import a DMA-BUF FD as a cl_mem */
cl_mem_properties props[] = {
    CL_EXTERNAL_MEMORY_HANDLE_DMA_BUF_KHR,
    (cl_mem_properties)dma_buf_fd,
    0
};
cl_mem imported = clCreateBufferWithProperties(ctx, props,
                                                CL_MEM_READ_WRITE,
                                                buf_size, NULL, &err);
```

These extensions, part of the 2021 OpenCL 3.0 extension set, enable the following pipeline without any CPU copies:

```
V4L2 device  →  DMA-BUF FD  →  clCreateBufferWithProperties (import)
                                        │
                              OpenCL processing kernel
                                        │
                              Export DMA-BUF FD  →  EGL image  →  display
```

**`cl_khr_external_semaphore`** (part of the same 2021 set) allows OpenCL wait/signal operations to synchronise with Vulkan timeline semaphores or DRM syncobj, enabling heterogeneous pipelines where OpenCL compute and Vulkan rendering alternate without stalling either command stream.

---

## 9. Real-World Applications

### Darktable

Darktable ([darktable.org](https://www.darktable.org/)) uses OpenCL for its pixel pipeline: demosaicing, exposure adjustment, colour calibration, filmic RGB tone mapping, and noise reduction all have GPU-accelerated paths. The key architecture decisions:

**Adaptive tiling**: When the RAW file's pixel buffer exceeds available GPU VRAM (common with 45+ MP sensors), Darktable splits the image into overlapping tiles, processes each tile independently, and re-assembles them. The overlap handles edge effects in convolution kernels. Tiling introduces a performance penalty (sometimes 5–10×) due to repeated boundary transfers, but allows large-format RAW files to be processed on GPUs with 4–8 GiB VRAM.

**Memory management**: Darktable maintains a GPU memory budget and tracks allocation via internal metrics. When a module's `cl_mem` requirements exceed the budget, it automatically falls back to CPU for that module while keeping GPU acceleration for others.

**Debug options**: Starting darktable with `darktable -d opencl` enables verbose OpenCL diagnostics — device selection, kernel compilation, timing per module, and fallback events. The `darktable-cltest` utility (separate binary) verifies the OpenCL setup independently of the GUI.

### hashcat

hashcat ([hashcat.net](https://hashcat.net/)) uses OpenCL (alongside CUDA and HIP) for GPU-accelerated password recovery. Its cross-GPU portability is the core value proposition: the same hashcat binary, using OpenCL, runs identically on AMD, Intel, NVIDIA, and even CPU (via pocl). The performance-critical inner loops (MD5, SHA-256, bcrypt, etc.) are written as OpenCL kernels compiled at runtime for the specific device.

hashcat uses out-of-order command queues and multiple in-flight kernel dispatches to keep GPU utilization high, and exploits `cl_khr_subgroups` for SIMD-optimised reduction paths on AMD and Intel.

### Note on Blender Cycles

Blender Cycles used to support OpenCL as a GPU backend for rendering on AMD hardware. This support was **removed in Blender 3.0** (released 2021), with AMD GPUs now handled by **HIP** and NVIDIA GPUs by **CUDA and OptiX**. The removal was driven by maintenance burden (Cycles X rewrote the rendering core) and performance parity gaps between OpenCL and native GPU APIs on modern hardware.

This is a cautionary data point: for production rendering workloads, application developers increasingly prefer vendor-native compute APIs (HIP, CUDA) over OpenCL when targeting a specific hardware family, sacrificing portability for performance and feature velocity. OpenCL remains the pragmatic choice for applications that genuinely need to target all GPU vendors.

### BOINC and Scientific Computing

The **BOINC** distributed computing platform ([boinc.berkeley.edu](https://boinc.berkeley.edu/)) uses OpenCL to leverage volunteer GPU capacity across all vendors. Projects like Einstein@Home, Milkyway@home, and Asteroids@home distribute OpenCL kernels that the BOINC client compiles and runs on whatever GPU the volunteer has available.

**vkFFT** ([github.com/DTolm/VkFFT](https://github.com/DTolm/VkFFT)) deserves special mention: it implements high-performance FFT on Vulkan, OpenCL, CUDA, HIP, and Metal from a single source base using a template metaprogramming approach. Its OpenCL backend makes it a practical choice for scientific applications needing FFT without tying to CUDA.

### FFmpeg OpenCL Filters

FFmpeg includes several OpenCL-accelerated video filters (`avgblur_opencl`, `overlay_opencl`, `unsharp_opencl`, `tonemap_opencl`). These are invoked via the filter graph:

```bash
ffmpeg -init_hw_device opencl=ocl:0.0 \
       -filter_hw_device ocl \
       -i input.mkv \
       -vf 'hwupload,tonemap_opencl=tonemap=hable,hwdownload,format=yuv420p' \
       -c:v libx264 output.mp4
```

The `hwupload` filter uploads frames to the OpenCL device memory (`cl_mem`), the tonemapping kernel executes on the GPU, and `hwdownload` brings the result back. When combined with VA-API decode (`-hwaccel vaapi`), the pipeline can keep pixel data on-GPU from decode through processing to encode, with DMA-BUF as the transfer mechanism between subsystems.

---

## 10. Debugging and Profiling

### Kernel Timing with clGetEventProfilingInfo

The most direct profiling mechanism is event-based timing. Enable profiling on the command queue and query events:

```c
cl_command_queue queue = clCreateCommandQueueWithProperties(
    ctx, device,
    (cl_queue_properties[]){ CL_QUEUE_PROPERTIES,
                              CL_QUEUE_PROFILING_ENABLE, 0 },
    &err);

cl_event evt;
clEnqueueNDRangeKernel(queue, kernel, 1, NULL, &gws, &lws,
                        0, NULL, &evt);
clFinish(queue);

cl_ulong t_start, t_end;
clGetEventProfilingInfo(evt, CL_PROFILING_COMMAND_START,
                         sizeof(t_start), &t_start, NULL);
clGetEventProfilingInfo(evt, CL_PROFILING_COMMAND_END,
                         sizeof(t_end), &t_end, NULL);
printf("Kernel time: %.3f ms\n", (t_end - t_start) / 1e6);
```

The timestamps are in nanoseconds from an implementation-defined epoch. The interval `CL_PROFILING_COMMAND_END - CL_PROFILING_COMMAND_START` is the GPU execution time; `CL_PROFILING_COMMAND_SUBMIT - CL_PROFILING_COMMAND_QUEUED` captures host-side queuing overhead.

### Implementation-Specific Debug Variables

| Implementation | Debug Variable | Effect |
|---|---|---|
| rusticl | `RUSTICL_DEBUG=1` | Verbose logging |
| rusticl | `RUSTICL_DEBUG=nir` | Dump NIR IR for each kernel |
| rusticl | `RUSTICL_DEBUG=program` | Log program compilation |
| pocl | `POCL_DEBUG=all` | Full trace (compilation, dispatch) |
| pocl | `POCL_DEBUG=opencl` | API-level trace only |
| Intel NEO | (see NEO docs) | Various INTEL_* vars |
| AMD CLR | `AMD_OCL_WAIT_COMMAND=1` | Force synchronous execution for easier error attribution — *Note: verify this variable name in the current CLR release* |

### oclgrind

**oclgrind** ([github.com/jrprice/Oclgrind](https://github.com/jrprice/Oclgrind)) is an OpenCL device simulator built on an LLVM interpreter. It intercepts all OpenCL API calls and executes kernels in a simulator that can detect:

- **Out-of-bounds memory accesses**: Catches reads/writes past buffer ends.
- **Data races**: With `--data-races` (or `OCLGRIND_DATA_RACES=1`), detects when two work-items access the same address without a barrier, reporting both work-item IDs and source locations.
- **Barrier divergence**: Flags workgroups where some work-items reach a `barrier()` call while others do not (undefined behaviour per the OpenCL spec).
- **Invalid API usage**: Checks that `clSetKernelArg` types match kernel argument types.

```bash
# Run an application through oclgrind
oclgrind --data-races ./my_opencl_app

# Or via ICD replacement
OCLGRIND_OCL_ICD=/etc/OpenCL/vendors/oclgrind.icd ./my_opencl_app
```

oclgrind requires LLVM 18+ to build and operates independently of the system's OpenCL ICD by acting as its own platform.

### clpeak

**clpeak** ([github.com/krrishnarraj/clpeak](https://github.com/krrishnarraj/clpeak)) benchmarks the raw performance limits of OpenCL devices: single/double-precision FLOPS, integer throughput, memory bandwidth (global, constant, local), and image throughput.

```bash
clpeak
# Schematic output format (numbers vary by device):
# Platform: <Platform Name>
#   Device: <Device Name>
#     Single-precision compute (GFLOPS): <value>
#     Double-precision compute (GFLOPS): <value>
#     Half-precision compute (GFLOPS):   <value>
#     Integer compute (GIOPS):           <value>
#     Global memory bandwidth (GB/s):    <value>
#     Constant memory bandwidth (GB/s):  <value>
#     Local memory bandwidth (GB/s):     <value>
#     Kernel launch latency (us):        <value>
```

clpeak reports results in standardized units that make cross-device comparisons straightforward. Actual numbers depend entirely on the specific GPU and driver version; benchmark databases like [openbenchmarking.org/test/pts/clpeak](https://openbenchmarking.org/test/pts/clpeak) aggregate real measurements across hardware configurations. This tool is useful for establishing a baseline before measuring application-level kernel efficiency — a kernel achieving 30% of clpeak's peak FLOPS has headroom for optimization; one at 80% is likely memory-bound.

### Intel GPU Tools

`intel_gpu_top` (from the `intel-gpu-tools` package) displays real-time GPU utilisation, render/compute/copy engine activity, and memory bandwidth. When running OpenCL workloads via NEO, the "CS" (Compute Shader / Command Streamer) column shows utilisation:

```bash
intel_gpu_top
# Displays: render%, compute%, media%, copy%, interrupt rate, VRAM usage
```

### AMD ROCm Profiler

For AMD GPUs with the ROCm stack, `rocprof` can capture OpenCL kernel dispatch timings and hardware counter data. As of ROCm 6.x, the command is:

```bash
rocprof --stats --hsa-trace ./my_opencl_app
# Generates results.csv with kernel dispatch start/end times and hardware counters
```

### OpenCL Conformance Test Suite (CTS)

The Khronos OpenCL CTS ([github.com/KhronosGroup/OpenCL-CTS](https://github.com/KhronosGroup/OpenCL-CTS)) is the normative test suite for conformance. It is structured as individual test binaries under `test_conformance/`:

```
test_conformance/
  basic/          ← fundamental operations
  images/         ← image read/write/copy
  atomics/        ← global/local/64-bit atomics
  math_brute_force/ ← ULP accuracy of math builtins
  subgroups/      ← cl_khr_subgroups
  spirv_new/      ← SPIR-V ingestion via clCreateProgramWithIL
  SVM/            ← Shared Virtual Memory
```

Running a specific test against a specific platform:

```bash
./test_basic --platform-index 0 --device-index 0
./test_spirv_new computeAdd --platform-index 0
```

Implementations seeking Khronos conformance must pass all mandatory tests and report the optional-feature test results for extensions they claim to support.

---

## 10.1 Choosing Between OpenCL, CUDA, Vulkan Compute, and HIP

| Criteria | OpenCL | CUDA | Vulkan Compute | HIP |
|---|---|---|---|---|
| Vendor support | Intel, AMD, NVIDIA, ARM | NVIDIA only | All Vulkan GPUs | AMD (+ limited NVIDIA via HIPIFY) |
| Performance | Good | Best on NVIDIA | Very good | Good on AMD |
| Portability | Excellent | None | Excellent | Limited |
| Tooling | clinfo, oclgrind, clpeak | Nsight, nvvp | RenderDoc, spirv-val | rocprof, roc-gdb |
| Ecosystem | Libraries (OpenCV, FFmpeg, Darktable) | Largest (cuBLAS, cuDNN) | Growing (WGSL compute) | ROCm libs (MIOpen, rocBLAS) |
| Ease of use | Verbose C | Clean CUDA C++ | Very verbose GLSL/SPIR-V | CUDA-like C++ |
| Kernel persistence | Runtime-compiled (`.cl` → JIT) | Pre-compiled + JIT (PTX) | Pre-compiled SPIR-V | Pre-compiled + JIT |

**Decision guide**:
- **Cross-hardware portability is the priority**: OpenCL via rusticl (Mesa) or Vulkan compute + SPIR-V (best long-term)
- **NVIDIA-only ML/HPC workloads**: CUDA
- **AMD GPU compute and ML**: ROCm/HIP + MIOpen
- **Embedded/automotive Linux, non-CUDA GPU**: OpenCL (widest ICD availability)
- **New cross-platform project, Vulkan already in use**: Vulkan compute shares the same command queue and memory as graphics — zero extra driver overhead

---

## 11. Integrations

- **Ch14 (NIR)**: rusticl compiles OpenCL C through SPIR-V into NIR — the same IR used by all Mesa Gallium drivers for graphics and Vulkan workloads. The `spirv_to_nir()` function in Mesa is the shared entry point.

- **Ch15 (ACO)**: When rusticl targets AMD hardware via radeonsi, compiled NIR passes through the ACO register allocator and instruction scheduler just as Vulkan compute shaders do. The entire ACO optimization pipeline applies equally to OpenCL and Vulkan kernels.

- **Ch17 (Software Renderers — llvmpipe/softpipe)**: llvmpipe is one of the Gallium drivers that rusticl can target via `RUSTICL_ENABLE=llvmpipe`. This gives a fully software-rendered OpenCL implementation useful for testing, CI, and systems without a GPU, built on the same LLVM JIT that llvmpipe uses for its fragment shader execution.

- **Ch25 (GPU Compute)**: OpenCL is one of three major GPU compute APIs on Linux alongside CUDA and ROCm HIP. Chapter 25 surveys the GPU compute landscape; this chapter provides the deep OpenCL implementation detail. The choice between OpenCL (portable), HIP (AMD-optimised), and CUDA (NVIDIA-optimised) depends on target hardware diversity.

- **Ch62 (SYCL)**: SYCL 2020 implementations such as Intel DPC++ and AdaptiveCpp (formerly hipSYCL) can target OpenCL as a backend: SYCL kernels compile to SPIR-V, which is then ingested via `clCreateProgramWithIL`. The `cl_khr_unified_svm` extension (in development) is designed to match the SYCL 2020 USM model.

- **Ch108 (ROCm/HIP)**: The AMD ROCm stack runs HIP and OpenCL in parallel — both use the CLR (Compute Language Runtime) above the HSA/KFD kernel layer. A single `amdgpu.ko` kernel module and KFD serve both API stacks.

- **Ch110 (SPIR-V)**: `clCreateProgramWithIL` is one of the primary SPIR-V consumers on Linux alongside Vulkan's `vkCreateShaderModule`. The SPIR-V Tools (`spirv-val`, `spirv-opt`, `spirv-dis`) are equally useful for debugging OpenCL kernel SPIR-V as they are for Vulkan shaders.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*

## Roadmap

### Near-term (6–12 months)
- **rusticl fine-grain SVM and full OpenCL 3.0 CTS pass on radeonsi**: Karol Herbst's ongoing Mesa MRs target completing fine-grain SVM buffer support on radeonsi, the last major gap before rusticl can achieve conformance certification on AMD hardware to match the existing Intel iris submission.
- **AMD ROCm OpenCL 3.0 conformance**: AMD's CLR roadmap targets OpenCL 3.0 certification for CDNA3 (MI300X) and RDNA3 hardware; the work involves completing the optional-feature tier (fine-grain SVM, pipes, device-side enqueue) required for a Khronos-accepted submission.
- **`cl_khr_unified_svm` extension ratification**: The Khronos OpenCL working group has a draft specification for a portable USM extension (`cl_khr_unified_svm`) that standardises what Intel's `cl_intel_unified_shared_memory` currently provides; Intel NEO and rusticl are expected to ship initial implementations once the spec is ratified.
- **pocl LLVM 21 stabilisation and RVV (RISC-V Vector) improvements**: pocl 7.x stabilises LLVM 21 support for the CPU backend, with active upstream work on scalable-vector (SVE/RVV) autovectorisation paths that improve throughput on AArch64 servers and RISC-V SBCs.

### Medium-term (1–3 years)
- **rusticl as the default Mesa OpenCL on all major distributions**: As rusticl achieves CTS conformance on radeonsi, Fedora, Ubuntu, and Arch are expected to pre-enable it by default (via `gallium-rusticl-enable-drivers=radeonsi,iris`) and begin the deprecation process for Clover, which is then expected to be removed from Mesa entirely.
- **Clover removal from Mesa**: Clover is officially on a maintenance-only trajectory; once rusticl passes the mandatory CTS on the drivers Clover currently targets (radeonsi, iris, softpipe), the removal MR is expected to land, reducing the Gallium frontend maintenance surface significantly.
- **`cl_khr_external_semaphore` and DMA-BUF interop becoming first-class in all major ICDs**: Adoption of the 2021 OpenCL 3.0 interop extension set (`cl_khr_external_memory_dma_buf`, `cl_khr_external_semaphore`) across Intel NEO, AMD CLR, and rusticl will enable zero-copy V4L2-to-OpenCL-to-display pipelines without application-specific workarounds, aligning with the broader Linux graphics stack's DMA-BUF-first memory model.
- **OpenCL 3.1 specification work**: The Khronos OpenCL working group has active proposals for a 3.1 revision incorporating work-graph extensions (analogous to D3D12 work graphs), extended subgroup operations, and tighter SPIR-V 1.7 alignment; open-source implementations are tracking the provisional specifications.

### Long-term
- **Convergence of OpenCL and SYCL kernel representation around SPIR-V**: The long-term Khronos trajectory is for OpenCL C to remain supported but for SPIR-V (from SYCL/DPC++, Clang, or offline compilation) to become the dominant ingestion path; this will deepen the integration between `clCreateProgramWithIL`, the SPIR-V Tools ecosystem, and Mesa's `spirv_to_nir()` pipeline — blurring the line between OpenCL and Vulkan compute at the IR level.
- **Unified kernel interface for heterogeneous compute (HSA 2.0 / Vulkan SC alignment)**: As ROCm's HSA runtime, Vulkan compute, and OpenCL converge on DMA-BUF and timeline semaphore synchronisation primitives, the architectural distinction between submitting an OpenCL NDRange and a Vulkan compute dispatch may reduce to a thin API layer above a shared kernel command-submission interface (DRM GEM + syncobj), enabling tighter cross-API scheduling in compositors and media pipelines.
- **OpenCL on RISC-V and emerging accelerator architectures**: pocl's LLVM-first architecture positions it as the implementation most likely to support next-generation RISC-V GPU and AI-accelerator targets (e.g., OpenHW Group's CVA6-based GPU IP, Milk-V and StarFive RVV platforms); long-term, OpenCL's vendor-neutral ICD model may be the path of least resistance for bringing heterogeneous compute to open-hardware ecosystems.
