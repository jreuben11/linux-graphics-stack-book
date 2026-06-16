# Chapter 66: CUDA Runtime, Streams, and NVRTC

> **Part**: Part XV — NVIDIA Proprietary Graphics Stack
> **Audience**: Systems developers and ML engineers using NVIDIA on Linux
> **Status**: First draft — 2026-06-15

---

## Table of Contents

1. [Overview](#overview)
2. [CUDA Library Layering: Driver API vs. Runtime API](#1-cuda-library-layering-driver-api-vs-runtime-api)
3. [CUDA Streams: Ordered GPU Work Queues](#2-cuda-streams-ordered-gpu-work-queues)
4. [CUDA Events: Asynchronous Timing and Synchronization](#3-cuda-events-asynchronous-timing-and-synchronization)
5. [Memory Model: Device, Pinned, Unified, and Stream-Ordered](#4-memory-model-device-pinned-unified-and-stream-ordered)
6. [NVRTC: Runtime Compilation of CUDA C++ to PTX](#5-nvrtc-runtime-compilation-of-cuda-c-to-ptx)
7. [CUDA Graphs: Capturing and Replaying GPU Work](#6-cuda-graphs-capturing-and-replaying-gpu-work)
8. [Multi-Process Service (MPS) and MIG Partitioning](#7-multi-process-service-mps-and-mig-partitioning)
9. [NVML: GPU Monitoring API](#8-nvml-gpu-monitoring-api)
10. [CUDA on Linux: Kernel Interfaces and Diagnostic Paths](#9-cuda-on-linux-kernel-interfaces-and-diagnostic-paths)
11. [Integrations](#integrations)
12. [References](#references)

---

## Overview

This chapter examines the CUDA software stack as it operates on Linux: from the layered shared libraries that expose the Driver and Runtime APIs, through streams and events as the fundamental concurrency and synchronization primitives, to NVRTC for runtime kernel compilation and CUDA Graphs for eliminating CPU dispatch overhead in iterative workloads. It then covers the operational concerns that matter most on production Linux systems—MPS and MIG for multi-tenant GPU sharing, NVML for monitoring, and the procfs/debugfs interfaces exposed by the nvidia kernel modules.

Readers of this chapter should already be comfortable with the Linux graphics stack at the depth covered in earlier parts: DRM scheduling (Ch4), the nvidia kernel module and open-gpu-kernel-modules (Ch9), Vulkan queue and synchronization model (Ch25), and container-level CUDA device exposure (Ch55). The goal here is not a CUDA tutorial but a precise map of how the CUDA programming model is realized in the Linux kernel driver and userspace libraries, with emphasis on the interfaces, version constraints, and Linux-specific behaviours that trap experienced practitioners.

---

## 1. CUDA Library Layering: Driver API vs. Runtime API

### 1.1 libcuda.so and libcudart.so

The CUDA userspace stack on Linux is split across two shared libraries with distinct ownership and version cycles.

**`libcuda.so`** — the CUDA Driver API — is installed as part of the NVIDIA graphics driver package, not the CUDA Toolkit. It lives on the standard dynamic linker path (`/usr/lib/x86_64-linux-gnu/libcuda.so.1` on Ubuntu, symlinked from the versioned name). Its header is `cuda.h`. Driver API functions carry the `cu` prefix: `cuInit`, `cuCtxCreate`, `cuModuleLoadData`, `cuLaunchKernel`.

**`libcudart.so`** — the CUDA Runtime API — is installed as part of the CUDA Toolkit. It translates every Runtime call into one or more Driver API calls which `libcuda.so` dispatches into the kernel driver (`nvidia.ko` / `nvidia-uvm.ko`). Its C++ header is `cuda_runtime.h`; the C-compatible header is `cuda_runtime_api.h`. Runtime API functions carry the `cuda` prefix: `cudaEventCreate`, `cudaLaunchKernel`, `cudaMalloc`.

Both APIs are feature-equivalent for common use. The Driver API additionally exposes explicit context (`CUcontext`) and module management primitives used by NVRTC, nvJitLink, and plugin frameworks that need to load compiled kernels into a running process without a full Toolkit dependency.

### 1.2 Version Compatibility and CUDA_ERROR_INVALID_PTX

The driver API version installed on a system must be greater than or equal to the runtime API version the application was built against. This is enforced by CUDA's forward-compatibility layer (`libcuda.so` exposes a compatibility interface for toolkit versions newer than the driver, within limits documented per-major-version).

When a PTX binary is loaded via `cuModuleLoadData()`, the driver JIT-compiles it to native SASS for the installed GPU. This succeeds only if the driver's JIT understands the PTX ISA version embedded in the module. If an application was compiled with a newer nvcc that emits `ptx7.8` directives but the installed driver only understands up to `ptx7.4`, the driver returns `CUDA_ERROR_INVALID_PTX`. The error message in userspace libraries is typically the unhelpful string "a PTX JIT compilation failed." The underlying cause is always a toolkit/driver version mismatch: `nvcc --ptx-version` shows what ISA level nvcc targets; `nvidia-smi` shows the driver version which implies the maximum supported PTX ISA. [Source: NVIDIA developer forums thread on CUDA_UNSUPPORTED_PTX_VERSION][1]

### 1.3 Loading PTX and CUBIN via the Driver API

The classic module-based path:

```c
// file: driver_load_ptx_example.c
// Loads PTX text compiled by NVRTC; launches kernel without full Toolkit dependency.
#include <cuda.h>

CUdevice   dev;
CUcontext  ctx;
CUmodule   mod;
CUfunction fn;

cuInit(0);
cuDeviceGet(&dev, 0);
cuCtxCreate(&ctx, 0, dev);

// ptx: null-terminated PTX string from NVRTC or offline nvcc --ptx
cuModuleLoadData(&mod, ptx);          // JIT-compiles to SASS for current GPU
cuModuleGetFunction(&fn, mod, "saxpy"); // retrieve by mangled name
cuLaunchKernel(fn, gridX, 1, 1, blockX, 1, 1, sharedMem, stream, args, NULL);
cuCtxSynchronize();
cuModuleUnload(mod);
cuCtxDestroy(ctx);
```

[Source: CUDA Driver API — 3.3 The CUDA Driver API][2]

CUDA 12.0 introduced a parallel **library API** that is context-independent: a `CUlibrary` handle is loaded once and the driver automatically propagates the compiled code into each `CUcontext` as it is created or destroyed, removing the need for per-context module management in multi-context applications:

```c
// file: driver_library_api_example.c — CUDA 12.0+ context-independent loading
CUlibrary lib;
CUkernel  kernel;
CUfunction fn;

cuLibraryLoadData(&lib, ptx, NULL, NULL, 0, NULL, NULL, 0);
cuLibraryGetKernel(&kernel, lib, "saxpy");
cuKernelGetFunction(&fn, kernel);   // obtains a CUfunction for current context
cuLaunchKernel(fn, gridX, 1, 1, blockX, 1, 1, 0, stream, args, NULL);
```

[Source: CUDA Driver API — 6.12 Library Management][3]

The library API is preferred for frameworks that create and destroy CUDA contexts dynamically (e.g., multi-tenant inference servers). The module API remains appropriate for short-lived tools and NVRTC-driven compilation pipelines where a single context is sufficient.

### 1.4 Context and the Primary Context Model

Every CUDA process automatically acquires the **primary context** for each device it uses. The Runtime internally calls `cuDevicePrimaryCtxRetain()`. Code mixing Driver and Runtime API calls must retrieve the primary context via `cuDevicePrimaryCtxRetain()` and activate it with `cuCtxSetCurrent()` before making Driver API calls; otherwise the Driver operates on a separate context and unified memory and peer access APIs are unavailable.

The CUDA programming guide explicitly discourages multiple `CUcontext` objects per device per process: the GPU scheduler must time-slice between contexts even within the same process, degrading throughput. Green Contexts (CUDA 12.4+, `cudaGreenCtxCreate`) provide an intra-process SM-partitioning alternative without multi-context overhead.

---

## 2. CUDA Streams: Ordered GPU Work Queues

### 2.1 The Default Stream and Implicit Barriers

A CUDA stream is an ordered queue of GPU operations. Work within a stream executes in issue order; work across streams may execute concurrently, subject to hardware resource availability and explicit synchronization.

Stream 0 — the **null stream** or **default stream** — is a synchronization barrier. Any non-blocking operation issued to stream 0 implicitly synchronizes with every other default stream in the process: before stream 0 operations execute, all previously-issued work on all other streams completes; after stream 0 operations complete, subsequent work on all other streams may proceed. This implicit barrier is the largest source of unintentional serialization in CUDA codebases.

### 2.2 Creating and Destroying Streams

```c
// file: stream_create_example.c — paraphrased from CUDA RT API 13.3 docs
#include <cuda_runtime.h>

cudaStream_t stream;

// Default: synchronizes with the null stream (cudaStreamDefault = 0)
cudaStreamCreate(&stream);

// Non-blocking: no implicit synchronization with null stream
cudaStreamCreateWithFlags(&stream, cudaStreamNonBlocking);

// With priority: lower integer = higher priority
int leastPri, greatestPri;
cudaDeviceGetStreamPriorityRange(&leastPri, &greatestPri);
// On current hardware: greatestPriority = -1, leastPriority = 0
cudaStreamCreateWithPriority(&stream, cudaStreamNonBlocking, greatestPri);

// Query stream ID (CUDA 12.0+)
unsigned long long streamId;
cudaStreamGetId(stream, &streamId);

// Destroy: does NOT wait for in-flight work; GPU ops complete asynchronously
cudaStreamDestroy(stream);
```

[Source: CUDA RT API — Stream Management][4]

### 2.3 Cross-Stream Synchronization

The canonical inter-stream dependency uses events as lightweight GPU-side timestamps:

```c
// file: cross_stream_sync_example.c
cudaEvent_t event;
cudaEventCreateWithFlags(&event, cudaEventDisableTiming); // no timestamp overhead

// Stream A issues work, then records event
myKernelA<<<grid, block, 0, streamA>>>(d_in, d_mid);
cudaEventRecord(event, streamA);

// Stream B waits for the event before proceeding — no host involvement
cudaStreamWaitEvent(streamB, event, 0);
myKernelB<<<grid, block, 0, streamB>>>(d_mid, d_out);
```

`cudaStreamWaitEvent` inserts a GPU-side dependency: stream B will not begin any subsequently-enqueued work until the event recorded in stream A has been reached. This is purely asynchronous from the host's perspective — no blocking, no CPU spinwait.

### 2.4 Stream Priority and the DRM Scheduler Analogy

CUDA stream priority is the CUDA user-space analogue of the DRM GPU scheduler priority described in Ch4. Just as `drm_gpu_scheduler` maintains multiple priority run-queues (high, normal, low, kernel) that the GuC or firmware scheduler drains in priority order, CUDA streams with different priorities feed priority queues in the NVIDIA GPU hardware scheduler. On NVIDIA hardware, two priority levels are typically exposed (greatestPriority and leastPriority). The driver maps streams to the hardware work queues that correspond to those levels. Pending work in higher-priority streams preempts lower-priority streams at kernel granularity on Pascal and later architectures (not mid-warp).

Key difference from DRM: DRM priority is typically fixed at queue-creation time and maps to VkQueue priority (`VK_EXT_global_priority`); on Intel and AMD open-source drivers, these priorities flow through `VK_EXT_global_priority` all the way to the DRM scheduler entity. CUDA stream priority is per-stream, per-process, and set entirely in user space — the CUDA driver communicates the priority to the kernel module (`nvidia.ko`) via ioctl when creating the hardware channel, rather than through a kernel scheduler policy. The `sched_ext` BPF-programmable scheduler framework (Linux 6.15) applies to CPU scheduling only; GPU work scheduling is controlled by the hardware's own command processor and remains outside its scope. The conceptual mapping is:
- DRM `DRM_SCHED_PRIORITY_HIGH` → CUDA `greatestPriority` (-1 on current hardware)
- DRM `DRM_SCHED_PRIORITY_NORMAL` → CUDA `leastPriority` (0 on current hardware)

### 2.5 Host-Function Callbacks in Streams

```c
// file: stream_host_func_example.c
// cudaLaunchHostFunc: preferred over deprecated cudaStreamAddCallback
typedef void (*cudaHostFn_t)(void *userData);

void myCallback(void *data) {
    // Safe to call non-CUDA APIs here; cannot call CUDA synchronization APIs
    printf("GPU work complete, iteration %d\n", *(int *)data);
}

cudaLaunchHostFunc(stream, myCallback, &iteration);
```

`cudaLaunchHostFunc` (CUDA 10.0+) runs a host function inline in the stream after all preceding GPU operations complete. Unlike the deprecated `cudaStreamAddCallback`, it does not prevent the stream from making further GPU-side progress while the callback runs. The restriction on calling CUDA synchronization APIs (`cudaStreamSynchronize`, `cudaEventSynchronize`) from within the callback still applies.

---

## 3. CUDA Events: Asynchronous Timing and Synchronization

Events are GPU-side timestamps that can be recorded into streams and queried or synchronized from the host. They are the primitive underlying `cudaStreamWaitEvent` cross-stream synchronization, IPC synchronization, and kernel timing.

### 3.1 Creation Flags

```c
// file: event_create_example.c — paraphrased from CUDA RT API 13.3 docs
cudaEvent_t event;

// Standard event: default spin/block behavior on cudaEventSynchronize
cudaEventCreate(&event);

// Non-busy-wait: CPU blocks (sleeps) during synchronize, lower CPU utilization
cudaEventCreateWithFlags(&event, cudaEventBlockingSync);

// No timestamp: lower overhead, required for IPC events
cudaEventCreateWithFlags(&event, cudaEventDisableTiming);

// IPC-capable (Linux only): combine with DisableTiming
cudaEventCreateWithFlags(&event, cudaEventInterprocess | cudaEventDisableTiming);
```

### 3.2 Recording and Timing

```c
// file: event_timing_pattern.c — canonical kernel timing pattern
cudaEvent_t start, stop;
cudaEventCreate(&start);
cudaEventCreate(&stop);

cudaEventRecord(start, stream);
myKernel<<<grid, block, 0, stream>>>(args...);
cudaEventRecord(stop, stream);

cudaEventSynchronize(stop);   // blocks host until stop event is reached

float ms;
cudaEventElapsedTime(&ms, start, stop);
// Resolution: approximately 0.5 µs on current hardware

cudaEventDestroy(start);
cudaEventDestroy(stop);
```

`cudaEventRecord` with `cudaEventRecordExternal` flag (CUDA 11.x+) creates an external event node during stream capture for CUDA Graphs — discussed in Section 6.

### 3.3 IPC Events (Linux Only)

IPC events allow GPU-side synchronization across OS process boundaries without CPU involvement. They require `cudaEventInterprocess | cudaEventDisableTiming` at creation and Unified Virtual Addressing (compute capability ≥ 2.0). The event handle is an opaque struct that can be transmitted via a Unix domain socket or shared memory segment:

```c
// file: ipc_event_example.c — Linux only; requires UVA (CC >= 2.0)
// Process A (producer):
cudaIpcEventHandle_t handle;
cudaIpcGetEventHandle(&handle, event);
// Transmit handle to process B via socket/shmem

// Process B (consumer):
cudaEvent_t remoteEvent;
cudaIpcOpenEventHandle(&remoteEvent, handle);
cudaStreamWaitEvent(localStream, remoteEvent, 0);
// localStream will not proceed past this point until process A records remoteEvent
```

[Source: CUDA Programming Guide — Inter-Process Communication][5]

"The CUDA IPC API is only currently supported on Linux." [Source: NVIDIA CUDA Programming Guide][5]

---

## 4. Memory Model: Device, Pinned, Unified, and Stream-Ordered

### 4.1 Device Memory

```c
// file: device_memory_example.c — paraphrased from CUDA RT API 13.3
void *devPtr;
cudaMalloc(&devPtr, size);       // synchronous: host blocks until allocation complete
cudaMemset(devPtr, 0, size);
cudaMemcpy(host, devPtr, size, cudaMemcpyDeviceToHost);  // synchronous transfer
cudaFree(devPtr);                // synchronous: host blocks
```

`cudaMalloc` and `cudaFree` serialize against all preceding GPU work in the process — they behave like a global barrier. For performance-sensitive code, replace them with the stream-ordered allocator described in Section 4.4.

### 4.2 Pinned (Page-Locked) Host Memory

```c
// file: pinned_memory_example.c
void *hostPtr;
cudaMallocHost(&hostPtr, size);               // page-locked allocation

// Variant with flags:
cudaHostAlloc(&hostPtr, size,
    cudaHostAllocPortable |    // accessible from all CUDA contexts in process
    cudaHostAllocWriteCombined // faster for GPU reads, slower for CPU writes
);

// Async transfer over stream (DMA, no CPU involvement during transfer):
cudaMemcpyAsync(devPtr, hostPtr, size, cudaMemcpyHostToDevice, stream);

cudaFreeHost(hostPtr);
```

Pinned memory enables the PCIe DMA engine to transfer directly between host RAM and GPU VRAM without a staging copy through kernel bounce buffers. `cudaMemcpyAsync` over a non-blocking stream with pinned memory is the standard pattern for overlapping H2D transfers with compute on a different stream. For existing allocations (e.g., from `malloc`), `cudaHostRegister`/`cudaHostUnregister` pins and unpins pages in-place.

### 4.3 Unified Memory and HMM

```c
// file: unified_memory_example.c
void *ptr;
cudaMallocManaged(&ptr, size, cudaMemAttachGlobal);

// Prefetch to GPU device 0 before kernel launch (CUDA 13.x v2 signature)
cudaMemLocation loc;
loc.type = cudaMemLocationTypeDevice;
loc.id   = 0;
cudaMemPrefetchAsync(ptr, size, loc, 0 /*flags*/, stream);

// Advise: data read far more often than written; system may replicate pages
cudaMemAdvise(ptr, size, cudaMemAdviseSetReadMostly, loc);
// Advise: prefer residency at device 0
cudaMemAdvise(ptr, size, cudaMemAdviseSetPreferredLocation, loc);

cudaFree(ptr);
```

**BREAKING CHANGE — CUDA 13.0**: `cudaMemAdvise` and `cudaMemPrefetchAsync` changed their device parameter from `int deviceId` to `cudaMemLocation` (an internally renamed `_v2` signature). Code written for CUDA ≤ 12.x using bare `int deviceId` fails to compile under CUDA 13.1+ with:

```text
error: no suitable constructor exists to convert from "int" to "cudaMemLocation"
```

[Source: NVIDIA Developer Forum — cudaMemPrefetchAsync compilation error with CUDA 13.1][6]

**Unified Memory on Linux with HMM.** NVIDIA's HMM (Heterogeneous Memory Management) support, available when using the Open Kernel Modules (driver ≥ 535) on Linux kernel ≥ 6.1.24, enables GPU access to all system-allocated memory (`malloc`, `mmap`) without requiring `cudaMallocManaged`. The GPU MMU mirrors host page tables; a GPU page fault triggers the Linux kernel's HMM migration engine which moves the physical page to GPU memory. Requirements: compute capability ≥ 7.5 (Turing); Open Kernel Modules; Linux ≥ 6.1.24 (stable) or ≥ 6.3 (upstream). This is the same HMM kernel infrastructure shared with amdgpu SVM (Ch5). Query addressing mode:

```bash
# Check HMM / UVA addressing mode
nvidia-smi -q | grep -i "Addressing Mode"
```

On Linux with HMM or compute capability ≥ 6.0, unified memory allocations can exceed GPU VRAM capacity; the driver pages out GPU data to host memory under pressure, enabling out-of-core GPU algorithms. [Source: NVIDIA Blog — Simplifying GPU Application Development with HMM][7]

**Comparison table — page migration mechanisms:**

| System | Coherence | Page-table model | Oversubscription |
|--------|-----------|-----------------|------------------|
| Discrete GPU + Linux HMM | Software | Separate, mirrored via HMM | Yes (CC ≥ 6.0) |
| Grace Hopper NVLink-C2C | Hardware | Unified | Yes |
| Windows (any) | Software | Separate | Limited (CC < 6.0: none) |

### 4.4 Stream-Ordered Memory Allocator (CUDA 11.2+)

The stream-ordered allocator eliminates the implicit global synchronization of `cudaMalloc`/`cudaFree`:

```c
// file: stream_ordered_alloc_example.c — requires cudaDevAttrMemoryPoolsSupported
cudaMemPool_t pool;
cudaMemPoolProps props = {};
props.allocType     = cudaMemAllocationTypePinned;
props.location.type = cudaMemLocationTypeDevice;
props.location.id   = 0;
cudaMemPoolCreate(&pool, &props);

// Prevent pool from releasing physical memory back to OS between iterations:
uint64_t threshold = UINT64_MAX;
cudaMemPoolSetAttribute(pool, cudaMemPoolAttrReleaseThreshold, &threshold);

void *ptr;
// Allocation is ordered after prior work in stream:
cudaMallocFromPoolAsync(&ptr, size, pool, stream);
myKernel<<<grid, block, 0, stream>>>(ptr);
// Free is ordered before subsequent allocations can reuse this memory:
cudaFreeAsync(ptr, stream);

cudaStreamSynchronize(stream);
cudaMemPoolDestroy(pool);
```

[Source: CUDA Programming Guide — Stream-Ordered Memory Allocation][8]

A device-default pool is available via `cudaMallocAsync`/`cudaFreeAsync` without explicit pool creation. Cross-stream reuse of freed memory requires either explicit event-based ordering or enabling `cudaMemPoolReuseFollowEventDependencies`.

---

## 5. NVRTC: Runtime Compilation of CUDA C++ to PTX

NVRTC (NVIDIA Runtime Compilation) compiles device-side CUDA C++ source strings into PTX or CUBIN at application runtime. It ships as `libnvrtc.so` with header `nvrtc.h`. NVRTC accepts device code only — no `__host__` functions, no `#include <stdio.h>` in global scope — and is distinct from `nvcc`, which handles full translation units mixing host and device code.

### 5.1 Core API

```c
// file: nvrtc_core_api_example.c — signatures from NVRTC 13.3 docs (confirmed)
#include <nvrtc.h>

nvrtcProgram prog;

// Create a program from a source string
nvrtcCreateProgram(
    &prog,
    src,            // CUDA C++ source string (device code only)
    "saxpy.cu",     // name for diagnostic messages
    0,              // numHeaders
    NULL,           // header source strings
    NULL            // header names
);

// Register C++ template instantiations before compilation
nvrtcAddNameExpression(prog, "saxpy<float>");

// Compile with options
const char *opts[] = {
    "--gpu-architecture=compute_86",  // PTX for sm_86 family
    "--std=c++17",
    "--use_fast_math"
};
nvrtcResult res = nvrtcCompileProgram(prog, 3, opts);

// Always retrieve log (warnings appear even on success)
size_t logSize;
nvrtcGetProgramLogSize(prog, &logSize);
char *log = malloc(logSize);
nvrtcGetProgramLog(prog, log);
if (res != NVRTC_SUCCESS) {
    fprintf(stderr, "NVRTC compile error:\n%s\n", log);
    exit(1);
}
free(log);

// Get PTX
size_t ptxSize;
nvrtcGetPTXSize(prog, &ptxSize);
char *ptx = malloc(ptxSize);
nvrtcGetPTX(prog, ptx);  // null-terminated PTX string

// Get lowered (mangled) name for the template instantiation
const char *loweredName;
nvrtcGetLoweredName(prog, "saxpy<float>", &loweredName);

nvrtcDestroyProgram(&prog);

// Load via Driver API (no full Toolkit dependency)
CUmodule mod;
CUfunction fn;
cuModuleLoadData(&mod, ptx);
cuModuleGetFunction(&fn, mod, loweredName);
free(ptx);
```

[Source: NVRTC 13.3 Documentation][9]

### 5.2 Compiler Options

Architecture targets: `--gpu-architecture=compute_75` through `compute_121` for PTX; `--gpu-architecture=sm_75` for device-specific CUBIN.

C++ standards supported (CUDA 13.3): `--std=c++03`, `c++11`, `c++14`, `c++17`, `c++20`, `c++23`. The `c++23` flag is new in CUDA 13.3 and enables C++23 language features in device code. [Source: CUDA 13.3 Release Notes][10]

Key flags:
- `--use_fast_math`: enables flush-to-zero, reciprocal approximations, and other precision trade-offs
- `--maxrregcount=<N>`: cap register usage per thread (increases occupancy at the cost of spills)
- `--device-c`: relocatable device code (required for multi-TU link with nvJitLink)
- `--dlto`: generate LTO IR for nvJitLink link-time optimization
- `-G`: device-side debug info (disables optimizations)
- `-lineinfo`: source-level line information (compatible with optimization)

### 5.3 Bundled Headers (CUDA 13.3)

Previously, NVRTC required the user to point `-I` flags at a full CUDA Toolkit installation to access CUDA C++ and CCCL (CUDA C++ Core Libraries) headers. CUDA 13.3 adds bundled header support:

```c
// file: nvrtc_bundled_headers.c — CUDA 13.3+
// Extract bundled headers to a temp directory once at startup.
// Signature: nvrtcResult nvrtcInstallBundledHeaders(const char *installPath,
//                                                   unsigned int flags,
//                                                   const char **errorLog)
const char *errLog = NULL;
nvrtcResult res = nvrtcInstallBundledHeaders(
    "/tmp/nvrtc_headers",              // destination directory (created if absent)
    NVRTC_INSTALL_HEADERS_SKIP_IF_EXISTS,  // skip if already installed (default)
    &errLog                            // optional error detail on failure
);
if (res != NVRTC_SUCCESS) {
    fprintf(stderr, "bundled headers install failed: %s\n",
            errLog ? errLog : nvrtcGetErrorString(res));
}

// Compile with -I pointing to the extracted paths:
const char *opts[] = {
    "--gpu-architecture=compute_90",
    "--std=c++20",
    "-I/tmp/nvrtc_headers",
    "-I/tmp/nvrtc_headers/cccl"  // CCCL sub-tree (thrust, cub, libcudacxx)
};
```

A convenience compiler flag `--use-bundled-headers=<dir>` combines the install call and the `-I` additions in one step — pass it as an option to `nvrtcCompileProgram` and NVRTC extracts and includes the headers automatically. This enables header-only Toolkit deployment: distribute `libnvrtc.so` (which bundles the headers) and applications can compile CUDA C++ kernels without a separate Toolkit install. [Source: CUDA 13.3 Release Notes][10] [Source: NVRTC 13.3 Documentation — Bundled Headers][9]

### 5.4 PTX to cubin via nvJitLink

For production deployments with multiple separately-compiled kernel libraries, nvJitLink (CUDA 12.0+) links multiple GPU code objects at runtime with LTO across them:

```c
// file: nvjitlink_lto_example.c — paraphrased from nvJitLink 13.3 docs
#include <nvJitLink.h>

// Compile each kernel TU with --dlto to get LTO IR:
// nvrtcGetLTOIR(prog, ltoIR);  nvrtcGetLTOIRSize(prog, &ltoIRSize);

nvJitLinkHandle handle;
const char *options[] = {"-arch=sm_90", "-O3"};
nvJitLinkCreate(&handle, 2, options);

// Add each LTO IR blob:
nvJitLinkAddData(handle, NVJITLINK_INPUT_LTOIR, ltoIR1, size1, "kernel_a");
nvJitLinkAddData(handle, NVJITLINK_INPUT_LTOIR, ltoIR2, size2, "kernel_b");

nvJitLinkComplete(handle);  // performs LTO across all inputs

size_t cubinSize;
nvJitLinkGetLinkedCubinSize(handle, &cubinSize);
char *cubin = malloc(cubinSize);
nvJitLinkGetLinkedCubin(handle, cubin);

// Load the linked cubin via Driver API:
CUmodule mod;
cuModuleLoadData(&mod, cubin);
```

[Source: nvJitLink 13.3 Documentation][11]

OptiX 9 uses NVRTC as its kernel compilation path: shader programs are C++ source strings compiled via `nvrtcCreateProgram` to PTX, which OptiX then translates into `OptixModule` objects. See Ch67 for the OptiX-specific compilation pipeline.

---

## 6. CUDA Graphs: Capturing and Replaying GPU Work

CUDA Graphs capture a sequence of GPU operations into a directed acyclic graph (DAG) that can be instantiated and relaunched with a single API call, eliminating the repeated CPU dispatch overhead of individual stream submissions. For iterative workloads — ML training loops, physics simulation, iterative solvers — graphs reduce per-iteration CPU cost from microseconds-per-launch to a single `cudaGraphLaunch` call. [Source: CUDA Programming Guide — CUDA Graphs][12]

### 6.1 Stream Capture

```c
// file: graph_stream_capture_example.c — paraphrased from CUDA RT API 13.3
cudaGraph_t     graph;
cudaGraphExec_t graphExec;

// Begin capture on a non-blocking stream
// cudaStreamCaptureModeGlobal: any uncaptured CUDA API call from any thread = error
cudaStreamBeginCapture(stream, cudaStreamCaptureModeGlobal);

// All operations issued to stream (and streams joined via cudaStreamWaitEvent)
// are captured as graph nodes — not executed yet
cudaMemcpyAsync(d_in, h_in, size, cudaMemcpyHostToDevice, stream);
myKernel<<<grid, block, sharedMem, stream>>>(d_in, d_out);
cudaMemcpyAsync(h_out, d_out, size, cudaMemcpyDeviceToHost, stream);

cudaStreamEndCapture(stream, &graph);  // returns the captured graph

// Instantiate once
cudaGraphInstantiate(&graphExec, graph, 0);
cudaGraphDestroy(graph);  // template no longer needed after instantiation

// Launch N times — single API call per iteration
for (int i = 0; i < N; i++) {
    updateHostInputs(h_in);
    cudaGraphLaunch(graphExec, stream);
    cudaStreamSynchronize(stream);
}

cudaGraphExecDestroy(graphExec);
```

For capture modes: `cudaStreamCaptureModeGlobal` is the safe default — any CUDA API call from any thread that is not captured generates an error. `cudaStreamCaptureModeThreadLocal` allows uncaptured work on other threads concurrently. Use `cudaStreamBeginCaptureToGraph` (available since CUDA 12.x; signature: `cudaStreamBeginCaptureToGraph(stream, graph, dependencies, dependencyData, numDependencies, mode)`) to capture into an existing graph with specified node dependencies, enabling incremental graph construction — unlike `cudaStreamBeginCapture` which always creates a new graph.

### 6.2 Updating Graph Parameters Without Reinstantiation

Two update paths exist with different granularities:

**Per-node update** (`cudaGraphExecKernelNodeSetParams`): updates a single kernel node's launch parameters (grid dimensions, block dimensions, shared memory, kernel arguments) in-place on the instantiated graph. Topology is unchanged:

```c
// file: graph_node_update_example.c — CUDA 11.2+
cudaKernelNodeParams newParams = {
    .func           = (void*)myKernel,
    .gridDim        = {newGridX, 1, 1},
    .blockDim       = {blockX, 1, 1},
    .sharedMemBytes = sharedMem,
    .kernelParams   = newArgs,
    .extra          = NULL
};
cudaGraphExecKernelNodeSetParams(graphExec, kernelNode, &newParams);
```

**Whole-graph update** (`cudaGraphExecUpdate`): takes a new `cudaGraph_t` with the same topology as the instantiated graph and updates all node parameters at once. If the topology has changed (different number of nodes, different edge structure), the update fails and the caller must destroy and reinstantiate. This is useful when the capture loop produces a new graph each iteration (e.g., dynamic batch sizes) but the topology is stable:

```c
// file: graph_exec_update_example.c — CUDA 11.2+
cudaGraphExecUpdateResultInfo updateResult;
cudaError_t err = cudaGraphExecUpdate(graphExec, newGraph, &updateResult);
if (err != cudaSuccess) {
    // Topology changed; reinstantiate
    cudaGraphExecDestroy(graphExec);
    cudaGraphInstantiate(&graphExec, newGraph, 0);
}
```

### 6.3 Manual Graph Construction

For topology that cannot be captured from a stream (e.g., conditionally included nodes), build the graph programmatically:

```c
// file: graph_manual_build_example.c
cudaGraph_t graph;
cudaGraphCreate(&graph, 0);

cudaGraphNode_t memcpyNode, kernelNode;
cudaMemcpy3DParms memcpyParams = { /* ... */ };
cudaGraphAddMemcpyNode(&memcpyNode, graph, NULL, 0, &memcpyParams);

cudaKernelNodeParams kernelParams = { /* func, grid, block, args... */ };
cudaGraphAddKernelNode(&kernelNode, graph, &memcpyNode, 1, &kernelParams);
// Second argument array carries node dependencies (topological edges)
```

### 6.4 Conditional Nodes (CUDA 12.4+)

Conditional nodes enable on-device branching and looping without returning to the CPU. The device kernel sets a condition variable; the graph node uses it to decide whether to execute a subgraph:

```c
// file: graph_conditional_example.c — CUDA 12.4+; enhanced IF-ELSE/SWITCH in 12.8
cudaGraphConditionalHandle condHandle;
cudaGraphConditionalHandleCreate(&condHandle, graph,
    0 /*defaultValue*/, 0 /*flags*/);

cudaGraphNodeParams cParams = {};
cParams.type                   = cudaGraphNodeTypeConditional;
cParams.conditional.handle     = condHandle;
cParams.conditional.type       = cudaGraphCondTypeWhile;  // loop
cParams.conditional.size       = 1;  // 1 subgraph body; 2=IF-ELSE, N=SWITCH
cudaGraphAddNode(&condNode, graph, deps, numDeps, &cParams);
```

In a device kernel (the "condition setter"), before returning:
```c
// device code — sets loop-continue condition
__device__ void updateCondition(cudaGraphConditionalHandle h, bool cont) {
    cudaGraphSetConditional(h, cont ? 1u : 0u);
}
```

CUDA 12.8 added IF-ELSE (size=2, body[0] for true, body[1] for false) and SWITCH (size=N, selects body[conditionValue]) variants, primarily targeting Blackwell. [Source: NVIDIA Blog — Dynamic Control Flow in CUDA Graphs][13]

**Thread safety note**: "Graph objects are not thread-safe." Concurrent modification of the same `cudaGraph_t` from multiple threads requires external synchronization.

---

## 7. Multi-Process Service (MPS) and MIG Partitioning

### 7.1 GPU Sharing Without MPS: Time-Sliced Contexts

Without any partitioning mechanism, each OS process that initializes CUDA acquires its own GPU context. The GPU hardware time-slices between contexts via preemptive context switching at kernel granularity (Pascal+) or block granularity (pre-Pascal). Context switches are expensive: the driver must save and restore the full GPU state. On a system with eight concurrent ML inference processes, seven processes are idle (context-switched out) at any moment.

### 7.2 MPS Architecture

The CUDA Multi-Process Service replaces per-process contexts with a shared server context. Three components:

1. **`nvidia-cuda-mps-control`** — control daemon managing server lifecycle and client routing; ensures at most one server per GPU per user account
2. **`nvidia-cuda-mps-server`** — owns the single GPU context; creates worker threads for each client; mediates shared scheduling resources
3. **`libcuda.so` client runtime** — transparent to the application; CUDA API calls route to the server instead of the driver directly

On Volta (sm_70) and later, MPS enables truly concurrent kernel execution from multiple client processes in the same GPU context. On pre-Volta, MPS still reduces context-switch overhead but does not enable hardware-level spatial sharing.

### 7.3 Linux MPS Configuration

```bash
# file: mps_setup.sh — Linux only; requires 64-bit; CC >= 3.5 with UVA

# Set exclusive compute mode (required pre-Ampere; enforces single MPS server)
sudo nvidia-smi -i 0 -c EXCLUSIVE_PROCESS

# Configure pipe and log directories (unique per node in cluster deployments)
export CUDA_VISIBLE_DEVICES=0
export CUDA_MPS_PIPE_DIRECTORY=/tmp/nvidia-mps
export CUDA_MPS_LOG_DIRECTORY=/tmp/nvidia-log

# Start the control daemon (background)
nvidia-cuda-mps-control -d

# Verify: running CUDA processes show "M+C" in Type column of nvidia-smi

# Per-client SM limits (set before the client process starts):
export CUDA_MPS_ACTIVE_THREAD_PERCENTAGE=25  # limit client to 25% of SMs

# Shutdown gracefully
echo quit | nvidia-cuda-mps-control
```

[Source: NVIDIA MPS Documentation][14]

Pipes and sockets reside in `CUDA_MPS_PIPE_DIRECTORY`. On multi-node clusters, set a node-unique path (e.g., incorporate `$(hostname)`) to prevent cross-node socket collisions.

**Fault isolation (Volta+)**: If a client faults (illegal memory access, timeout), the server isolates the fault to that client. The server returns to `ACTIVE` state once faulting clients disconnect, without terminating other clients' work. On pre-Volta, a client fault terminates the server and all clients.

**CUDA 13.1 static partitioning**: `nvidia-cuda-mps-control -d -S` starts the server in static partitioning mode. Each client must declare its SM partition at launch via `CUDA_MPS_SM_PARTITION`. This gives deterministic isolation guarantees comparable to MIG for software-partitioned workloads.

**Limitations for interactive graphics**: MPS is incompatible with OpenGL and Vulkan rendering contexts. A process using MPS cannot simultaneously perform CUDA-Vulkan interop (Ch25 external memory) from the same GPU. MPS is a compute-only multiplexing mechanism.

### 7.4 MIG: Multi-Instance GPU (Ampere and Later)

MIG partitions the physical GPU into up to 7 **GPU Instances (GI)**, each with dedicated hardware slices of SMs, L2 cache, DRAM controllers, and PCIe bandwidth. Unlike MPS, which shares all internal GPU resources among clients, each MIG instance has physically isolated memory paths:

| Property | Time-slicing | MPS | MIG |
|----------|-------------|-----|-----|
| Isolation | None (context switch) | SM partitioning only | Full physical isolation |
| Memory isolation | None | Logical (no hardware isolation) | Dedicated DRAM + L2 slices |
| QoS guarantees | None | Throughput hint only | Hard bandwidth guarantees |
| Multi-tenant security | None | Limited (shared context) | Strong (separate instances) |
| Graphics API support | Yes | No | No (compute only) |
| Hardware support | All | CC ≥ 3.5 | Ampere A100, A30; Hopper H100, H200; Blackwell |

Configuration:

```bash
# Enable MIG mode on device 0 (requires driver ≥ 450; Ampere+ hardware)
sudo nvidia-smi -i 0 --mig-enable
sudo reboot  # MIG mode change requires reboot on some systems

# Create a 3g.40gb GPU instance on A100-40GB (half the GPU)
sudo nvidia-smi mig -cgi 3g.40gb -C
# -C flag automatically creates the corresponding Compute Instance

# List MIG instances
nvidia-smi mig -lgi
```

[Source: NVIDIA MIG User Guide][15]

Profile names encode the SM fraction and memory: `1g.10gb` = 1/7th SMs, 10 GB DRAM; `7g.80gb` = full A100 80 GB in a single instance. Applications target a MIG instance by setting `CUDA_VISIBLE_DEVICES=MIG-<uuid>`.

MIG does not support Vulkan or OpenGL graphics. In Kubernetes GPU node deployments (Ch55), MIG instances appear as individual `nvidia.com/gpu: 1` resources per instance, allowing the scheduler to allocate sub-GPU fractions to pods. The NVIDIA Container Toolkit configures the device files exposed per container, mapping MIG instance UUIDs to isolated device nodes. [Source: NVIDIA Container Toolkit, Ch55][16]

### 7.5 Choosing Between Time-Slicing, MPS, and MIG

- **Latency-sensitive inference, homogeneous workloads**: MPS — reduces context-switch overhead with concurrent kernel execution; SM limits prevent a single client from starving others.
- **Security-sensitive multi-tenant ML cloud deployments**: MIG — hardware-enforced memory isolation prevents any cross-tenant data leakage; hard bandwidth guarantees enable SLA compliance.
- **Interactive + compute mixed workloads (rendering + CUDA)**: Time-slicing — only mode compatible with graphics APIs.
- **Single-node training on large models**: None of the above — use the full GPU with a single process.

---

## 8. NVML: GPU Monitoring API

NVML (NVIDIA Management Library) is the C API underlying `nvidia-smi`. On Linux it installs as `libnvidia-ml.so` on the standard library path with the driver. The header is `nvml.h`. NVML is thread-safe; multiple threads may call NVML functions concurrently.

### 8.1 Core Monitoring Functions

```c
// file: nvml_monitor_example.c — paraphrased from NVML API Reference R610 (May 2026)
#include <nvml.h>

nvmlDevice_t dev;
nvmlInit();
nvmlDeviceGetHandleByIndex(0, &dev);

// SM utilization % and memory bandwidth utilization %
nvmlUtilization_t util;
nvmlDeviceGetUtilizationRates(dev, &util);
printf("GPU: %u%%  MEM BW: %u%%\n", util.gpu, util.memory);

// Memory: total, used, free (bytes)
nvmlMemory_t mem;
nvmlDeviceGetMemoryInfo(dev, &mem);
printf("Used: %llu MB / Total: %llu MB\n",
       mem.used >> 20, mem.total >> 20);

// Temperature (degrees Celsius)
unsigned int temp;
nvmlDeviceGetTemperature(dev, NVML_TEMPERATURE_GPU, &temp);

// Power draw (milliwatts)
unsigned int power;
nvmlDeviceGetPowerUsage(dev, &power);
printf("Power: %.2f W\n", power / 1000.0);

// Clock frequencies (MHz)
unsigned int smClk, memClk;
nvmlDeviceGetClockInfo(dev, NVML_CLOCK_SM,  &smClk);
nvmlDeviceGetClockInfo(dev, NVML_CLOCK_MEM, &memClk);

nvmlShutdown();
```

[Source: NVML API Reference R610][17]

### 8.2 Per-Process Utilization

```c
// file: nvml_per_process_example.c
nvmlProcessesUtilizationInfo_t info;
nvmlDeviceGetProcessesUtilizationInfo(dev, &info);
// info.processSamplesCount contains per-PID utilization samples
// Each sample: pid, smUtil, memUtil, encUtil, decUtil
```

### 8.3 Hopper+ GPM Metrics

Hopper (H100) and later GPUs expose fine-grained performance metrics via the GPU Performance Monitoring (GPM) API, a subset of which is accessible through NVML:

```c
// file: nvml_gpm_example.c — Hopper (CC 9.0) and later
nvmlGpmSample_t sample1, sample2;
nvmlGpmSampleAlloc(&sample1);
nvmlGpmSampleAlloc(&sample2);
nvmlGpmSampleGet(dev, sample1);
// ... time passes, GPU work runs ...
nvmlGpmSampleGet(dev, sample2);

nvmlGpmMetricsGet_t metricsGet = {};
metricsGet.version         = nvmlGpmMetricsGet_v1;
metricsGet.numMetrics      = 1;
metricsGet.sample1         = sample1;
metricsGet.sample2         = sample2;
metricsGet.metrics[0].metricId = NVML_GPM_METRIC_SM_OCCUPANCY;
nvmlGpmMetricsGet(&metricsGet);
printf("SM Occupancy: %.2f%%\n", metricsGet.metrics[0].value);

nvmlGpmSampleFree(sample1);
nvmlGpmSampleFree(sample2);
```

### 8.4 Compilation

```bash
# Link against libnvidia-ml.so (installed with the NVIDIA driver, not the Toolkit)
gcc -o monitor monitor.c \
    -I/usr/local/cuda/include \
    -lnvidia-ml
```

`libnvidia-ml.so` is version-locked to the driver: a given `nvidia-smi` binary calls NVML functions in the `libnvidia-ml.so` that ships with the same driver version. CUDA 13.3 NVML additions include `nvmlDeviceGetRemappedRows_v2` (reports inactive row remapping) and Dynamic Boost support for Grace Blackwell (GB200/GB300). [Source: CUDA 13.3 Release Notes][10]

---

## 9. CUDA on Linux: Kernel Interfaces and Diagnostic Paths

### 9.1 DKMS and the Open Kernel Modules

The NVIDIA kernel module stack (Ch9) consists of `nvidia.ko` (core driver), `nvidia-uvm.ko` (Unified Memory / HMM), `nvidia-modeset.ko` (modesetting), and `nvidia-drm.ko` (DRM integration). On most Linux distributions, these are managed via DKMS (Dynamic Kernel Module Support): `dkms install -m nvidia -v <version>` rebuilds the modules for a new kernel. The Open Kernel Modules (`NVIDIA/open-gpu-kernel-modules` on GitHub) are source-available replacements for `nvidia.ko` and `nvidia-uvm.ko`, required for HMM support (Section 4.3) and supporting Turing (sm_75) and later. Older GPUs (Kepler, Maxwell, Pascal) require the proprietary closed-source modules.

When a kernel update breaks the DKMS build (common on rolling-release distributions), CUDA applications fail at driver initialization with `CUDA_ERROR_NO_DEVICE`. The diagnostic sequence is:

```bash
# Verify module load
lsmod | grep nvidia
dmesg | grep -i nvidia  # look for module load errors

# Rebuild modules for current kernel
sudo dkms autoinstall

# Verify CUDA device visibility
nvidia-smi
```

### 9.2 /proc/driver/nvidia Diagnostic Interface

The NVIDIA driver exposes a procfs interface for runtime diagnostics:

```text
/proc/driver/nvidia/version
    — driver revision and GNU C compiler version used to build the module

/proc/driver/nvidia/warnings/
    — text files containing warning messages about host kernel configuration
      (e.g., IOMMU conflicts, PCIe ACS policy)

/proc/driver/nvidia/gpus/<domain>:<bus>:<device>.<function>/information
    — model name, IRQ assignment, BIOS version, driver state
    — populated after GPU initialization; some fields available pre-init

/proc/driver/nvidia/gpus/<domain>:<bus>:<device>.<function>/power
    — runtime D3 (RTD3) power management status
    — shows current power state, constraint sources
```

[Source: NVIDIA /proc Interface Documentation, driver 535.98][18]

```bash
# Read driver version
cat /proc/driver/nvidia/version

# List all detected GPUs
ls /proc/driver/nvidia/gpus/

# Check a specific GPU's state (PCI address format: domain:bus:device.function)
cat /proc/driver/nvidia/gpus/0000:01:00.0/information
```

The `warnings` directory is particularly useful for diagnosing subtle configuration issues that cause non-obvious CUDA failures: IOMMU passthrough misconfiguration causes `cudaMemcpy` to fail silently with DMA errors logged here.

### 9.3 /sys/kernel/debug/nvidia (debugfs)

The NVIDIA driver mounts a debugfs subtree at `/sys/kernel/debug/nvidia/` when the kernel is built with `CONFIG_DEBUG_FS=y`. This interface is driver-version-specific and explicitly not a stable ABI — entries, names, and formats change across driver major versions.

```bash
# Ensure debugfs is mounted (on most systemd distros it is auto-mounted)
mount | grep debugfs
# If absent: sudo mount -t debugfs none /sys/kernel/debug

# Explore what the installed driver version exposes:
ls /sys/kernel/debug/nvidia/
```

> **Note: needs verification** — The specific subdirectory names and file paths within `/sys/kernel/debug/nvidia/` are not documented in the public NVIDIA driver release notes or in the procfs interface documentation ([Source: NVIDIA /proc Interface Documentation][18]). The entries that exist depend on the driver version (proprietary vs. Open Kernel Modules), the kernel configuration, and which subsystems are active. Key areas to explore include per-GPU PCI device directories (named by PCI address), and the `nvidia-uvm` module's subtree which is known to contain Unified Memory page-fault accounting data useful for diagnosing `cudaMallocManaged` performance pathologies — but exact paths should be verified against the running driver by inspection. Do not rely on any path documented in third-party guides; always `ls` the live tree on the target system before scripting against it.

The `/proc/driver/nvidia/` paths described in Section 9.2 are the NVIDIA-documented stable diagnostic interface; prefer those over debugfs for any production monitoring or automated tooling.

---

## Integrations

**Ch4 — DRM Scheduling**: CUDA stream priorities are the user-space analogue of DRM GPU scheduler priority queues. Both map user-supplied priority levels to hardware work queues; the preemption semantics (kernel-granularity on Pascal/Ampere, like Pascal-era DRM) are structurally equivalent. See Section 2.4.

**Ch5 — amdgpu SVM and HMM**: CUDA Unified Memory uses the same Linux HMM (Heterogeneous Memory Management) kernel infrastructure as amdgpu's SVM (System Virtual Memory) support. The migration paths, page-fault handling, and oversubscription model are the same kernel subsystem; the driver-side policy differs. See Section 4.3.

**Ch9 — NVIDIA Kernel Modules**: The CUDA Driver API (`libcuda.so`) dispatches all work into `nvidia.ko` and `nvidia-uvm.ko` via ioctl. HMM requires the Open Kernel Modules (driver ≥ 535). DKMS management of module builds is the Linux-level mechanism that keeps the kernel-user boundary operational across kernel updates. See Section 9.1.

**Ch25 — CUDA–Vulkan Interop**: CUDA streams are the compute analogue of Vulkan queues: both are ordered GPU work submission channels with priority levels and cross-channel synchronization via timeline primitives (CUDA events vs. Vulkan timeline semaphores). CUDA external memory (`cudaImportExternalMemory`, `cudaExternalMemoryHandleType_OpaqueFd`) and external semaphore (`cudaImportExternalSemaphore`) use the memory and synchronization primitives described in this chapter — device allocations, events — to share GPU buffers and timeline semaphores with Vulkan. MPS is incompatible with CUDA-Vulkan interop.

**Ch48 — ROCm/HIP**: HIP provides a portable alternative to the CUDA Runtime API at the programming model level, mapping `hipStream_t` to CUDA streams, `hipEvent_t` to CUDA events, and `hipMalloc` to `cudaMalloc`. The underlying ROCm runtime communicates with `amdgpu` in the same pattern that `libcudart.so` communicates with `nvidia.ko`.

**Ch55 — NVIDIA Container Toolkit**: The Container Toolkit controls which `/dev/nvidia*` device files and CUDA driver libraries are injected into containers. MIG instance UUIDs map to isolated device nodes per pod. NVML runs inside containers using the injected `libnvidia-ml.so` to monitor per-container GPU utilization.

**Ch67 — OptiX 9**: OptiX wraps a CUDA context (`OptixDeviceContext`). Shader programs compile via NVRTC (`nvrtcCreateProgram` → PTX → `optixModuleCreate`) — the same pipeline described in Section 5. Cooperative Vectors use `cudaMalloc`-allocated weight matrices. See Ch67 for the OptiX-specific build and launch model.

**Ch68 — DLSS 4 and Neural Rendering**: Gaussian Splatting training and inference run on the CUDA compute model (streams, events, managed memory) described in this chapter. The NGX SDK evaluates DLSS features as CUDA kernel dispatches managed through the Runtime API. See Ch68 for the NGX API surface.

---

## References

1. NVIDIA Developer Forum — "cudaMemPrefetchAsync compilation error with CUDA 13.1 on RTX 5070 Ti: no suitable constructor exists to convert from int to cudaMemLocation": https://forums.developer.nvidia.com/t/cudamemprefetchasync-compilation-error-with-cuda-13-1-on-rtx-5070-ti-no-suitable-constructor-exists-to-convert-from-int-to-cudamemlocation/357462

2. NVIDIA CUDA Programming Guide — 3.3 The CUDA Driver API: https://docs.nvidia.com/cuda/cuda-programming-guide/03-advanced/driver-api.html

3. NVIDIA CUDA Driver API — 6.12 Library Management: https://docs.nvidia.com/cuda/cuda-driver-api/group__CUDA__LIBRARY.html

4. NVIDIA CUDA Runtime API — Stream Management: https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__STREAM.html

5. NVIDIA CUDA Programming Guide — Inter-Process Communication: https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/inter-process-communication.html

6. NVIDIA Developer Forum — cudaMemPrefetchAsync v2 API signature change: https://forums.developer.nvidia.com/t/cudamemprefetchasync-compilation-error-with-cuda-13-1-on-rtx-5070-ti-no-suitable-constructor-exists-to-convert-from-int-to-cudamemlocation/357462

7. NVIDIA Blog — Simplifying GPU Application Development with Heterogeneous Memory Management: https://developer.nvidia.com/blog/simplifying-gpu-application-development-with-heterogeneous-memory-management

8. NVIDIA CUDA Programming Guide — Stream-Ordered Memory Allocation: https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/stream-ordered-memory-allocation.html

9. NVIDIA NVRTC 13.3 Documentation: https://docs.nvidia.com/cuda/nvrtc/index.html

10. NVIDIA CUDA Toolkit 13.3 Release Notes: https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html

11. NVIDIA nvJitLink 13.3 Documentation: https://docs.nvidia.com/cuda/nvjitlink/index.html

12. NVIDIA CUDA Programming Guide — CUDA Graphs: https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/cuda-graphs.html

13. NVIDIA Developer Blog — Dynamic Control Flow in CUDA Graphs with Conditional Nodes: https://developer.nvidia.com/blog/dynamic-control-flow-in-cuda-graphs-with-conditional-nodes

14. NVIDIA MPS Documentation: https://docs.nvidia.com/deploy/mps/index.html

15. NVIDIA MIG User Guide: https://docs.nvidia.com/datacenter/tesla/mig-user-guide/

16. NVIDIA Container Toolkit (referenced in Ch55): https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/

17. NVIDIA NVML API Reference R610: https://docs.nvidia.com/deploy/nvml-api/

18. NVIDIA Driver /proc Interface Documentation (driver 535.98): https://download.nvidia.com/XFree86/Linux-x86_64/535.98/README/procinterface.html

19. NVIDIA CUDA Runtime API — CUDA Graphs: https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__GRAPH.html

20. NVIDIA CUDA Runtime API — Memory Management: https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__MEMORY.html

21. NVIDIA CUDA Programming Guide — Unified Memory: https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/unified-memory.html

22. Linux Kernel HMM Documentation: https://www.kernel.org/doc/html/v5.0/vm/hmm.html

23. CUDA Context-Independent Module Loading (NVIDIA Technical Blog): https://developer.nvidia.com/blog/cuda-context-independent-module-loading/

24. nebuly-ai/nos — GPU partitioning modes comparison (MPS vs MIG): https://nebuly-ai.github.io/nos/dynamic-gpu-partitioning/partitioning-modes-comparison/
