# Chapter 93: GPU Performance Analysis Methodology

This chapter targets **graphics application developers**, **game developers**, **ML engineers**, and **systems performance engineers** who need to systematically diagnose and improve GPU performance on Linux. Unlike Chapter 30 (which surveys debugging and profiling tools), this chapter focuses on *methodology*: how to structure an investigation, how to interpret hardware counter data, and how to move from a symptom to a root cause to a fix.

> **Scope:** Ch93 covers GPU performance analysis **methodology** — the investigation framework, Vulkan timestamp queries for frame time decomposition, hardware counter interpretation, occupancy analysis, and moving from symptom to root cause to fix. **Ch30** covers development-time debugging tools (RenderDoc, validation layers, Mesa debug vars). **Ch137** surveys the GPU profiling ecosystem (vendor profilers, MangoHud, Perfetto, shader-db). This chapter is deliberately tool-agnostic; see Ch137 for tool-specific instructions.

---

## Table of Contents

1. [Introduction: Performance Analysis vs Debugging](#1-introduction)
2. [Frame Time Decomposition with Vulkan Timestamp Queries](#2-frame-time-decomposition)
3. [VK_KHR_performance_query: Hardware Counter Access](#3-vk_khr_performance_query)
4. [GPU-Bound vs CPU-Bound Analysis](#4-gpu-bound-vs-cpu-bound)
5. [Occupancy and Wave/Warp Analysis](#5-occupancy-and-wavewarp-analysis)
6. [Memory Bandwidth Profiling](#6-memory-bandwidth-profiling)
7. [Pipeline Stall Taxonomy](#7-pipeline-stall-taxonomy)
8. [AMD-Specific: RGP and Mesa Counter Infrastructure](#8-amd-specific-rgp)
9. [NVIDIA-Specific: Nsight and ncu](#9-nvidia-specific-nsight-and-ncu)
10. [Intel-Specific: intel_gpu_top and EU Counters](#10-intel-specific)
11. [Worked Case Study: Optimising a Path-Traced Scene](#11-worked-case-study)
12. [eBPF-Based GPU Observability](#12-ebpf-based-gpu-observability)
13. [End-to-End Frame Delivery Latency](#13-end-to-end-frame-delivery-latency)
14. [Perfetto GPU Tracing on Linux](#perfetto-gpu-tracing-on-linux)
15. [Integrations](#14-integrations)

---

## 1. Introduction

Performance analysis differs from debugging: debugging finds *incorrect* behaviour; performance analysis finds *slow correct* behaviour. The distinction matters because the tools, mental models, and workflows diverge sharply.

GPU performance analysis on **Linux** is harder than on **Windows** or **macOS** for several reasons. **NVIDIA** does not ship a native **Linux** GUI profiler equivalent to **Nsight Graphics** (the GUI runs only on **Windows**, though it can capture remotely from **Linux**). **AMD**'s **Radeon GPU Profiler** (**RGP**) runs natively on **Linux** but requires specific kernel setup. **Intel**'s tools vary by generation. Regardless of vendor, the **Vulkan** API provides a portable counter collection path through **VK_KHR_performance_query** that works across all drivers that implement it.

This chapter covers the full methodology of GPU performance analysis, structured as a top-down investigation workflow. Section 2 explains **frame time decomposition** using **Vulkan** timestamp queries (**vkCmdWriteTimestamp2**) to isolate which render passes consume the most GPU time across the CPU simulation → **vkQueueSubmit** → GPU execution → swapchain present pipeline. Section 3 details **VK_KHR_performance_query**, including counter enumeration via **vkEnumeratePhysicalDeviceQueueFamilyPerformanceQueryCountersKHR**, **VkPerformanceCounterKHR** storage types, multi-pass collection, the **vkAcquireProfilingLockKHR** / **vkReleaseProfilingLockKHR** locking protocol, and enabling the extension on **RADV** via **RADV_PERFTEST=perf_counters**. Section 4 addresses **GPU-bound** vs **CPU-bound** analysis using **MangoHUD**, **nvtop**, **radeontop**, and **perf**, including the fence stall trap (where **vkWaitForFences** in a `perf` report is misread as CPU-bound), and **CPU**-side bottlenecks from **vkAllocateDescriptorSets**, **vkCreateGraphicsPipelines**, **VkPipelineCache**, and **VK_EXT_graphics_pipeline_library**. Section 5 analyses **occupancy** and wave/warp behaviour: **SM** occupancy on **NVIDIA**, **CU** occupancy on **AMD**, **VGPR**/**SGPR** register pressure, **LDS** (Local Data Share) shared memory limits, thread group sizing, **RDNA** **wave64** vs **wave32** modes via **VK_EXT_subgroup_size_control**, and strategies for reducing **VGPR** pressure using `barrier()`, **subgroupBallot**, and **subgroupShuffle**. Section 6 covers **memory bandwidth profiling**: the arithmetic intensity roofline, **DRAM** bandwidth measurement via `perf stat` AMD **PMU** events and **VK_KHR_performance_query** counters such as `GPU/MemUnitBusy` and `GPU/MemUnitStalled`, the **L1**/**L2**/**DRAM** cache hierarchy on **NVIDIA Ada** and **AMD RDNA 3**, and mitigations including **BC7**/**BC1**/**BC3** texture compression, **KTX2** with **BasisU**, packed vertex formats (**VK_FORMAT_R16G16_SFLOAT**), and **Morton**-order buffer layout. Section 7 provides a pipeline stall taxonomy covering **TMU** (texture fetch) latency stalls, **ALU** instruction-dependency stalls, **LDS** bank conflicts, **RDNA** export stalls (parameter cache saturation), and scalar-vs-vector **ALU** stalls; and shows how to identify them using **RGP**'s Pipeline Summary view, **ncu**'s `smsp__warp_issue_stalled_*` counters, and **VK_INTEL_performance_query** **EU** stall metrics.

Sections 8–10 treat vendor-specific tooling. For **AMD**: **RGP** thread-trace capture via **RADV_THREAD_TRACE**, the **RADV** performance counter infrastructure in **src/amd/vulkan/radv_perfcounter.c**, **radeon_gpu_analyzer** (**rga**) for **RDNA ISA** disassembly and **VGPR**/**SGPR** reporting, and **umr** (Userspace Register Debugger) for raw **MMIO** register access. For **NVIDIA**: **ncu** (NVIDIA Compute Profiler) for targeted metric collection and the **roofline model** via **ncu-ui**; **nsys** (**NVIDIA Nsight Systems**) for CPU+GPU timeline correlation; and **VK_NV_device_diagnostic_checkpoints** for diagnosing **GPU** hangs via **vkCmdSetCheckpointNV** / **vkGetQueueCheckpointDataNV**. For **Intel**: **intel_gpu_top** (from **intel-gpu-tools**) for real-time **Render/3D**, **Blitter**, **Video**, and **Compute** engine utilisation; `perf stat` with **Intel PMU** **EU** Active/Stall/Idle events; **VK_INTEL_performance_query** with **vkInitializePerformanceApiINTEL** and **vkAcquirePerformanceConfigurationINTEL**; **Intel VTune Profiler** for per-kernel **EU** utilisation; and **Xe** Slice/Subslice utilisation analysis.

Section 11 presents a worked case study optimising a **Vulkan** path tracer on an **NVIDIA GeForce RTX 3080** (**Ampere**) running **Ubuntu 24.04**: starting from 28 FPS, using frame decomposition with **vkCmdWriteTimestamp2**, **ncu** bandwidth analysis, secondary-ray **Morton**-code sorting with **GPU** radix sort to improve **L2** hit rate, callable shader refactoring via **vkCmdTraceRaysKHR** to reduce **VGPR** pressure and improve **SM** occupancy, and finally async **BVH** refit using **VK_BUILD_ACCELERATION_STRUCTURE_ALLOW_UPDATE_BIT_KHR** to remove **AccelerationStructureBuild** from the **CPU** critical path, ultimately reaching approximately 58–60 FPS.

### The measurement hierarchy

Sound performance work follows a top-down hierarchy:

1. **System-level**: wall-clock frame time, CPU/GPU utilisation (**MangoHUD**, **nvtop**, **radeontop**, **intel_gpu_top**)
2. **API-level**: Vulkan timestamp queries, **vkQueueSubmit** latency, fence wait time
3. **Hardware counter level**: GPU counter groups (SM occupancy, L2 hit rate, DRAM bandwidth)
4. **ISA level**: per-instruction cycle counts from **ncu** or **radeon_gpu_analyzer**

Start at level 1. Only descend to deeper levels once the higher level has confirmed a bottleneck exists. Skipping straight to hardware counters wastes time and confuses rather than clarifies.

### Common fallacies

- **"GPU-bound is the goal."** Being GPU-bound at 100% utilisation is only good if the GPU is doing the *right* work. A shader with a 90% texture stall rate saturates the GPU but delivers terrible throughput.
- **"VRAM size limits performance."** **VRAM** *bandwidth* is almost always the binding constraint; 24 GB at 50% bandwidth utilisation is worse than 16 GB at 95%.
- **"Fill rate is the bottleneck."** On modern GPUs with unified shaders, fill rate limits (**ROP**s) are rarely hit. Compute throughput, bandwidth, and latency dominate.

---

## 2. Frame Time Decomposition

### The frame pipeline

A rendered frame passes through: CPU simulation → render list construction → `vkQueueSubmit` → GPU execution → swapchain present. The critical measurement question is: *where does time go?*

```
CPU thread:    |-- simulate --|-- build cmdbuf --|-- submit --|
GPU timeline:                                       |-- GBuffer --|-- RT --|-- post --|
```

If the GPU idle gap between CPU submission and GPU start is large, submission latency is the problem (see §4). If GPU execution dominates, you are GPU-bound.

### Vulkan timestamp queries

`vkCmdWriteTimestamp2` (Vulkan 1.3 / `VK_KHR_synchronization2`) inserts a timestamp into the command buffer at a specified pipeline stage:

```cpp
// At the start of the GBuffer pass:
vkCmdWriteTimestamp2(cmd, VK_PIPELINE_STAGE_2_TOP_OF_PIPE_BIT, queryPool, 0);

// At the end:
vkCmdWriteTimestamp2(cmd, VK_PIPELINE_STAGE_2_BOTTOM_OF_PIPE_BIT, queryPool, 1);
```

After the frame completes, read back the timestamps:

```cpp
uint64_t timestamps[2];
vkGetQueryPoolResults(
    device, queryPool, 0, 2,
    sizeof(timestamps), timestamps, sizeof(uint64_t),
    VK_QUERY_RESULT_64_BIT | VK_QUERY_RESULT_WAIT_BIT);

double ns_per_tick = physicalDeviceProps.limits.timestampPeriod; // nanoseconds per tick
double gbuffer_ms = (timestamps[1] - timestamps[0]) * ns_per_tick * 1e-6;
```

`VkPhysicalDeviceLimits::timestampPeriod` converts GPU clock ticks to nanoseconds. On RDNA 3 it is typically 1 ns/tick. On Ada Lovelace it is approximately 1 ns/tick. Always verify with `vkGetPhysicalDeviceProperties`. [Source: Vulkan specification §20 — Queries](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#queries-timestamps)

### Which pipeline stages to use

- `TOP_OF_PIPE` / `BOTTOM_OF_PIPE` measure the wall-clock span of a command buffer section, including any stalls waiting for previous work. Use these for pass-level timing.
- `VERTEX_SHADER_BIT` / `FRAGMENT_SHADER_BIT` measure only the interval when those stages are active — useful for isolating specific stages within a renderpass.
- With Vulkan 1.3+ synchronization2, prefer `VK_PIPELINE_STAGE_2_TOP_OF_PIPE_BIT` (0x0000000000000001ULL) and `VK_PIPELINE_STAGE_2_BOTTOM_OF_PIPE_BIT`.

Build a per-frame timing table: GBuffer, shadow pass, lighting, ray tracing, post-process, UI. The largest entry is where optimisation effort pays off.

---

## 3. VK\_KHR\_performance\_query

`VK_KHR_performance_query` provides access to vendor hardware performance counters through a portable Vulkan API. [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_performance_query.html)

### Counter enumeration

```cpp
uint32_t counterCount = 0;
vkEnumeratePhysicalDeviceQueueFamilyPerformanceQueryCountersKHR(
    physDevice, graphicsQueueFamily, &counterCount, nullptr, nullptr);

std::vector<VkPerformanceCounterKHR> counters(counterCount);
std::vector<VkPerformanceCounterDescriptionKHR> descs(counterCount);
vkEnumeratePhysicalDeviceQueueFamilyPerformanceQueryCountersKHR(
    physDevice, graphicsQueueFamily, &counterCount, counters.data(), descs.data());
```

Each `VkPerformanceCounterKHR` has:
- `unit` — `VK_PERFORMANCE_COUNTER_UNIT_BYTES`, `CYCLES`, `NANOSECONDS`, `PERCENTAGE`, etc.
- `storage` — `UINT64`, `FLOAT32`, `FLOAT64`, `BOOL32`
- `uuid` — stable identifier across driver updates for the same hardware

Counter names (in `descs[i].name`) are vendor-specific strings. On RADV: `"GPU/CacheHit/L2CacheHit"`. On ANV: `"L3 Hit Rate"`. Parse `descs` at startup to find the counters you need.

### Query pool creation and collection

```cpp
uint32_t counterIndices[] = {l2HitCounterIdx, dramBytesCounterIdx};
VkQueryPoolPerformanceCreateInfoKHR perfInfo = {
    .sType = VK_STRUCTURE_TYPE_QUERY_POOL_PERFORMANCE_CREATE_INFO_KHR,
    .queueFamilyIndex = graphicsQueueFamily,
    .counterIndexCount = 2,
    .pCounterIndices = counterIndices,
};
// Determine how many passes the selected counter set requires:
uint32_t numPasses;
vkGetPhysicalDeviceQueueFamilyPerformanceQueryPassesKHR(physDevice, &perfInfo, &numPasses);
```

Some counter combinations require multiple render passes (the GPU replays the workload with different hardware mux settings). Build separate command buffers for each pass. [Source: Vulkan spec §22.5](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#queries-performance)

```cpp
// Before submission, acquire the profiling lock:
VkAcquireProfilingLockInfoKHR lockInfo = {
    .sType = VK_STRUCTURE_TYPE_ACQUIRE_PROFILING_LOCK_INFO_KHR,
    .timeout = UINT64_MAX,
};
vkAcquireProfilingLockKHR(device, &lockInfo);
// Submit all passes...
vkReleaseProfilingLockKHR(device);
```

### Enabling on RADV

```bash
export RADV_PERFTEST=perf_counters
```

This enables the Mesa RADV implementation of `VK_KHR_performance_query` on AMD hardware. Without it, the extension reports zero available counters. [Source: Mesa src/amd/vulkan/radv_perfcounter.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/vulkan/radv_perfcounter.c)

---

## 4. GPU-Bound vs CPU-Bound Analysis

### Identifying the bottleneck

The first question in any performance investigation: is the application GPU-bound or CPU-bound?

**GPU-bound**: GPU execution time equals or exceeds the wall-clock frame time. The GPU has no idle time between frames. Adding more GPU work increases frame time proportionally.

**CPU-bound**: The GPU has idle time waiting for the CPU to submit more work. The CPU (simulation, AI, physics, render list construction, Vulkan API calls) is the limiting factor.

Measurement approach:
```bash
# MangoHUD shows real-time CPU and GPU frame times:
MANGOHUD=1 ./my_game

# nvtop shows GPU engine utilisation:
nvtop

# radeontop for AMD:
radeontop
```

If MangoHUD's "GPU" bar is near 100% and "CPU" is lower, the application is GPU-bound. If "CPU" is near the frame budget and "GPU" shows idle gaps, it is CPU-bound.

### CPU-side profiling

When CPU-bound, profile the main thread:

```bash
perf record -g -F 1000 -- ./my_game
perf report
```

Look for time spent in:
- `vkQueueSubmit` — if significant, descriptor set allocation or command buffer recording is slow
- Game simulation / physics / AI — pure CPU work
- `vkWaitForFences` / `vkAcquireNextImageKHR` — CPU stalling on GPU completion (this is GPU-bound, not CPU-bound, despite appearing on the CPU)

### The fence stall trap

A common mistake: `vkWaitForFences` appears at the top of a `perf` report and developers conclude they are CPU-bound. In fact, waiting on a fence means the CPU is blocking on the GPU — the GPU is the bottleneck, and the CPU is idle, waiting for it to finish. This is GPU-bound, not CPU-bound.

### Descriptor set and pipeline cache pressure

If `vkAllocateDescriptorSets` or `vkCreateGraphicsPipelines` appears in CPU profiling with significant cost:
- Pre-allocate descriptor sets at load time using `VK_DESCRIPTOR_POOL_CREATE_FREE_DESCRIPTOR_SET_BIT`
- Use a pipeline cache (`VkPipelineCache`) serialised to disk to avoid runtime compilation
- Consider `VK_EXT_graphics_pipeline_library` (Ch76) for async pipeline creation

---

## 5. Occupancy and Wave/Warp Analysis

### What occupancy means

On NVIDIA hardware, the SM (Streaming Multiprocessor) can run multiple warps (groups of 32 threads) concurrently. **Occupancy** = active warps / maximum warps per SM. High occupancy enables latency hiding: when one warp stalls on a texture fetch, another warp can execute.

Low occupancy means the SM runs fewer warps than it could, and stalls cause the SM to go idle rather than switch to another warp.

On AMD hardware, the equivalent unit is the CU (Compute Unit), and thread groups are called **waves** (64 threads on RDNA2, optionally 32 on RDNA3 via wave32 mode).

### Occupancy limiters

Three common limiters:

**Register pressure**: Each SM has a fixed register file. If a shader uses many registers (VGPRs on AMD, registers on NVIDIA), fewer warps fit simultaneously.

```glsl
// High register pressure: many independent temporaries alive at once
vec4 a = texture(tex0, uv);
vec4 b = texture(tex1, uv);
vec4 c = texture(tex2, uv);
vec4 d = texture(tex3, uv);
// All 4 in-flight simultaneously → high register count
```

**Shared memory / LDS**: Compute shaders allocating large shared memory arrays limit how many thread groups fit per CU/SM.

**Thread group size**: Very small thread groups (e.g., 8 threads) waste SIMD lanes on RDNA (which prefers multiples of 64 or 32).

### RDNA wave64 vs wave32

RDNA3 supports both wave64 (64 threads per wave) and wave32 (32 threads per wave). Wave32 can improve occupancy for shaders with high register pressure by halving per-wave register consumption, but may underutilise the SIMD unit if the workload has divergent branches across the 64-lane vector width.

Control via GLSL:
```glsl
layout(local_size_x = 64) in;  // Prefer wave64 on RDNA
```

Or via Vulkan `VkPipelineShaderStageCreateInfo` with `VK_EXT_subgroup_size_control`.

### Measuring occupancy

**RADV/AMD**:
```bash
export RADV_DEBUG=shader
./my_app 2>&1 | grep -i "vgpr\|sgpr\|occupancy"
```

Mesa prints VGPR/SGPR counts for compiled shaders. Occupancy is a function of VGPR count and the CU's maximum: on RDNA3, max 256 VGPRs per CU, max 8 waves per CU in wave64 mode → if a shader uses >32 VGPRs, occupancy drops below 8 waves.

**NVIDIA Nsight**:
```
Metric: sm__warps_active.avg.pct_of_peak_sustained_active
```
Reports theoretical occupancy as percentage. Below 50% is a sign of register pressure or shared memory pressure. [Source: NVIDIA Nsight Compute documentation](https://docs.nvidia.com/nsight-compute/ProfilingGuide/)

**RGP (AMD Radeon GPU Profiler)**:
The occupancy lane view shows per-CU wave occupancy over time. Solid fills indicate high occupancy; gaps indicate stalls with no other waves to hide behind. [Source: https://gpuopen.com/rgp/](https://gpuopen.com/rgp/)

### Reducing VGPR pressure

- Restructure loops to reduce the number of live variables across iterations
- Use `barrier()` in compute shaders to reuse registers across phases
- Break large shaders into smaller passes
- On RDNA, use `subgroupBallot` / `subgroupShuffle` to reduce per-thread storage

---

## 6. Memory Bandwidth Profiling

### Why bandwidth dominates

GPU shader throughput (TFLOPS) has scaled faster than DRAM bandwidth (TB/s). A modern NVIDIA RTX 4090 delivers ~82.6 TFLOPS FP32 but only ~1 TB/s DRAM bandwidth. The arithmetic intensity (FLOPS/byte) required to be compute-bound is therefore ~82 ops/byte — most workloads, especially ML inference GEMM with batch size 1, are far below this and are therefore bandwidth-bound.

### Measuring DRAM bandwidth

**Via VK_KHR_performance_query (RADV)**:

Counter names on RDNA (verify with enumeration on your hardware — names vary):
- `"GPU/MemUnitBusy"` — percentage of time memory units are busy
- `"GPU/MemUnitStalled"` — percentage of time shaders are stalled waiting for memory

**Via nvtop**: The VRAM bandwidth bars show read and write utilisation in real time.

**Via perf (AMD)**:
```bash
perf stat -e amdgpu/mem-bw-read/,amdgpu/mem-bw-write/ ./my_app
```

### Cache hierarchy

Understanding the cache hierarchy is essential for interpreting bandwidth numbers:

| Level | NVIDIA Ada | AMD RDNA 3 | Size (typical) |
|-------|-----------|-----------|----------------|
| L1 / Texture cache | Per-SM | Per-CU | 128 KB / CU |
| L2 | Shared across GPU | Shared across GPU | 32–96 MB |
| DRAM | GDDR6X / HBM3 | GDDR6 / HBM3 | 16–192 GB |

L1 and L2 hits cost ~20–100 cycles. DRAM access costs ~500–1000 cycles. If L2 hit rate is low, DRAM bandwidth is saturated.

**AMD L2 hit rate via RGP**: The "Shader Engine" view shows per-SE cache hit rates. [Source: RGP documentation](https://gpuopen.com/learn/radeon-gpu-profiler-1-0/)

**NVIDIA L2 metrics via ncu**:
```
l2_global_load_bytes       # bytes loaded from L2 (L2 → L1)
dram_read_bytes            # bytes loaded from DRAM (DRAM → L2)
L2 hit rate = 1 - (dram_read_bytes / l2_global_load_bytes)
```

### Identifying bandwidth-bound shaders

Symptoms:
- High `MemUnitStalled` (AMD) or low `sm__pipe_alu_cycles_active` (NVIDIA)
- GPU utilisation at 100% but low TFLOPS throughput
- Adding compute work doesn't increase frame time (already bottlenecked elsewhere)

Mitigations:
- **Texture compression**: BC7 (high quality) or BC1/BC3 reduce bandwidth by 4–8×. ASTC on mobile. Use `KTX2` with BasisU for portable compressed textures (Ch63).
- **Packed vertex formats**: use `VK_FORMAT_R16G16_SFLOAT` instead of `R32G32_SFLOAT` for UV coordinates.
- **Access pattern optimisation**: ensure spatially adjacent fragments access spatially adjacent texels. Morton-order (Z-curve) layout for compute buffers.
- **Reduce redundant loads**: cache frequently-reused values in LDS/shared memory.

---

## 7. Pipeline Stall Taxonomy

### Stall types

Modern GPUs implement deep pipelines. A stall occurs when an instruction cannot issue because a dependency is not yet satisfied. Different stall types require different fixes.

**Texture fetch stall (TMU latency stall)**
The most common stall on fragment shaders. A texture sample takes ~500 cycles (uncached). The GPU hides this with warp/wave switching, but if occupancy is low, no other wave is available.

Fix: increase occupancy (§5) to improve latency hiding.

**ALU latency stall (instruction dependency)**
An arithmetic instruction must wait for a previous instruction's result. On RDNA3, FP32 FMA has a 4-cycle latency. The compiler schedules independent instructions between dependent ones, but if the shader has a long dependency chain with no independent work, stalls occur.

Fix: restructure shader to increase instruction-level parallelism. Avoid chains like `a = f(a); a = g(a); a = h(a);`.

**LDS (Local Data Share) / shared memory bank conflict**
Multiple threads in a warp access the same memory bank simultaneously, serialising accesses.

Fix: pad shared memory arrays. For 32 banks, pad stride by 1 element: `float lds[THREADS + 1]` instead of `float lds[THREADS]`.

**Export stall (RDNA-specific)**
The parameter cache between the vertex and fragment stages fills up. The vertex shader cannot export more attributes until the fragment shader consumes them.

Fix: reduce the number of varyings passed between vertex and fragment shaders. Pack multiple values into fewer vectors using `.xy`, `.zw` swizzles.

**Scalar vs vector ALU stall (RDNA)**
RDNA's scalar ALU handles control flow and uniform data; the vector ALU handles per-lane computation. If a shader performs excessive uniform-derived branching, the scalar ALU stalls the vector ALU.

Fix: hoist uniform computations outside loops. Use `subgroupBroadcast` for values uniform across a wave.

### Stall identification

**AMD RGP**: The "Pipeline Summary" view shows a per-stage stall breakdown. Hover over the stall bars for detailed counter names. [Source: https://gpuopen.com/rgp/](https://gpuopen.com/rgp/)

**NVIDIA Nsight Compute**:
```
sm__pipe_fma_cycles_active    # cycles where FMA pipe is doing useful work
sm__issue_active              # cycles where any instruction issued
stall fraction = 1 - (sm__pipe_fma_cycles_active / sm__issue_active)
```

Nsight also provides a warp stall breakdown: `smsp__warp_issue_stalled_*` counters identify the stall reason (memory dependency, execution dependency, synchronisation, etc.).

**Intel EU stalls**: `VK_INTEL_performance_query` exposes EU (Execution Unit) active/stall/idle breakdowns. [Source: Intel GPU tools](https://github.com/freedesktop/intel-gpu-tools)

---

## 8. AMD-Specific: RGP and Mesa Counter Infrastructure

### Radeon GPU Profiler setup

RGP captures GPU execution traces with sub-microsecond precision. Setup on Linux:

```bash
# Enable thread trace capture via RADV environment variable:
export RADV_THREAD_TRACE=1
export RADV_THREAD_TRACE_TRIGGER=F12   # press F12 in-app to capture

./my_vulkan_app
# A .rgp file is written to the working directory
# Open in RGP GUI (Linux native binary available):
./RadeonGpuProfiler frame_000001.rgp
```

RGP shows:
- **Event timeline**: draw call and dispatch timeline across all queues, with GFX/compute/copy engine lanes
- **Pipeline summary**: per-draw shader stage durations
- **Barriers**: VkMemoryBarrier, VkImageMemoryBarrier costs — often surprising (missed aliasing, unnecessary layout transitions)
- **Pipeline state**: Vulkan pipeline objects used per draw

[Source: https://gpuopen.com/rgp/](https://gpuopen.com/rgp/)

### RADV performance counters in Mesa

Mesa's RADV driver implements `VK_KHR_performance_query` via `src/amd/vulkan/radv_perfcounter.c`. The driver maps Vulkan counter indices to the hardware `PERF_SEL_*` register values defined in `src/amd/registers/`. [Source: Mesa RADV](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/amd/vulkan)

### radeon_gpu_analyzer

For ISA-level analysis:

```bash
rga -s vulkan -c gfx1100 --isa output_isa.txt --cfg output_cfg.dot my_shader.vert
```

`rga` (Radeon GPU Analyzer) disassembles SPIR-V or GLSL to RDNA ISA and reports VGPR/SGPR usage, instruction counts, and occupancy. [Source: https://github.com/GPUOpen-Tools/radeon_gpu_analyzer](https://github.com/GPUOpen-Tools/radeon_gpu_analyzer)

### umr: Userspace Register Debugger

For raw hardware register access (requires root or `CAP_SYS_RAWIO`):

```bash
umr -s gfx10 -r gfx/mmSQ_PERF_SEL_0
umr --gpu-id 0 --list-blocks
```

`umr` reads GPU MMIO registers directly, enabling custom counter collection outside the Vulkan layer. [Source: https://gitlab.freedesktop.org/tomstdenis/umr](https://gitlab.freedesktop.org/tomstdenis/umr)

---

## 9. NVIDIA-Specific: Nsight and ncu

### ncu: NVIDIA Compute Profiler

`ncu` is the command-line GPU performance profiler, available in the CUDA toolkit:

```bash
# Full metric set (slow, replays workload many times):
ncu --set full --export report.ncu-rep ./my_vulkan_app

# Targeted collection:
ncu --metrics sm__throughput.avg.pct_of_peak_sustained_elapsed,\
              dram__bytes.sum,\
              lts__t_bytes_equiv_l1sectmiss_pipe_lsu_mem_global_op_ld.sum \
    --export report.ncu-rep ./my_vulkan_app

# Open in Nsight Compute GUI:
ncu-ui report.ncu-rep
```

`ncu` replays each dispatch/draw multiple times with different hardware mux settings to collect all requested counters. This makes it slow for interactive applications — use it on isolated shader microbenchmarks where possible. [Source: https://docs.nvidia.com/nsight-compute/](https://docs.nvidia.com/nsight-compute/)

### Key ncu metrics

| Metric | Meaning |
|--------|---------|
| `sm__throughput.avg.pct_of_peak_sustained_elapsed` | Overall SM utilisation |
| `sm__warps_active.avg.pct_of_peak_sustained_active` | Occupancy |
| `dram__throughput.avg.pct_of_peak_sustained_elapsed` | DRAM bandwidth utilisation |
| `lts__throughput.avg.pct_of_peak_sustained_elapsed` | L2 bandwidth utilisation |
| `sm__pipe_fma_cycles_active.avg.pct_of_peak_sustained_active` | FP32 compute utilisation |

### The roofline model

Plot achieved FLOPS (y-axis) vs arithmetic intensity (FLOPS/byte, x-axis). The roofline is the minimum of: peak compute * intensity, and peak bandwidth. If your workload falls far below the roofline, there is a microarchitectural bottleneck beyond bandwidth and compute (latency-bound, occupancy-limited).

Nsight Compute generates roofline plots automatically in the "Roofline" analysis view.

### nsys: NVIDIA Nsight Systems

For system-level timeline capture:

```bash
nsys profile \
    --trace=vulkan,nvtx,cuda \
    --output timeline_report \
    ./my_vulkan_app

# Open:
nsys-ui timeline_report.nsys-rep
```

`nsys` shows the CPU timeline (threads, API calls) alongside the GPU timeline (kernel execution, memory transfers). Key use: identifying CPU→GPU submission gaps, queue depth, multi-queue parallelism. [Source: https://docs.nvidia.com/nsight-systems/](https://docs.nvidia.com/nsight-systems/)

### VK_NV_device_diagnostic_checkpoints

For diagnosing GPU hangs and TDRs:

```cpp
// Insert checkpoints at key points in the command buffer:
vkCmdSetCheckpointNV(cmd, reinterpret_cast<const void*>(42));  // "checkpoint 42"

// After a hang, query completed checkpoints:
uint32_t count;
vkGetQueueCheckpointDataNV(queue, &count, nullptr);
std::vector<VkCheckpointDataNV> checkpoints(count);
vkGetQueueCheckpointDataNV(queue, &count, checkpoints.data());
// The last completed checkpoint before the hang is reported
```

[Source: Vulkan extension VK_NV_device_diagnostic_checkpoints](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_NV_device_diagnostic_checkpoints.html)

---

## 10. Intel-Specific: intel\_gpu\_top and EU Counters

### intel_gpu_top

`intel_gpu_top` (from `intel-gpu-tools`) provides a real-time view of GPU engine utilisation:

```bash
sudo intel_gpu_top
```

Output shows:
- **Render/3D** engine utilisation percentage
- **Blitter** (copy engine) utilisation
- **Video** (VCS/VECS decode/encode) engine utilisation
- **Compute** engine utilisation (separate from render on Xe)
- Memory read/write bandwidth (on supported hardware)

[Source: https://github.com/freedesktop/intel-gpu-tools](https://github.com/freedesktop/intel-gpu-tools)

### perf-based GPU counters

```bash
# List available GPU events:
perf list | grep gpu

# Collect GPU counters for a workload:
sudo perf stat \
    -e gpu/metric=EU\ Active/,\
       gpu/metric=EU\ Stall/,\
       gpu/metric=EU\ Idle/ \
    ./my_vulkan_app
```

### VK_INTEL_performance_query

Intel ANV implements `VK_INTEL_performance_query` for hardware counter access from Vulkan:

```cpp
VkInitializePerformanceApiInfoINTEL initInfo = {
    .sType = VK_STRUCTURE_TYPE_INITIALIZE_PERFORMANCE_API_INFO_INTEL,
};
vkInitializePerformanceApiINTEL(device, &initInfo);

// Query configuration:
VkPerformanceConfigurationAcquireInfoINTEL configInfo = {
    .sType = VK_STRUCTURE_TYPE_PERFORMANCE_CONFIGURATION_ACQUIRE_INFO_INTEL,
    .type = VK_PERFORMANCE_CONFIGURATION_TYPE_COMMAND_QUEUE_METRICS_DISCOVERY_ACTIVATED_INTEL,
};
VkPerformanceConfigurationINTEL config;
vkAcquirePerformanceConfigurationINTEL(device, &configInfo, &config);
vkQueueSetPerformanceConfigurationINTEL(queue, config);
```

Key metrics: EU Active (EU executing instructions), EU Stall (EU waiting for data), EU Idle (EU doing nothing). EU Idle > 20% indicates serious under-utilisation. [Source: Vulkan extension VK_INTEL_performance_query](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_INTEL_performance_query.html)

### Intel VTune Profiler

Intel VTune (free download) supports GPU analysis on Linux for Intel hardware:

```bash
vtune -collect gpu-hotspots -result-dir vtune_result -- ./my_vulkan_app
vtune-gui vtune_result
```

VTune provides per-kernel EU utilisation, memory bandwidth, and Xe-specific throughput metrics. [Source: https://www.intel.com/content/www/us/en/developer/tools/oneapi/vtune-profiler.html](https://www.intel.com/content/www/us/en/developer/tools/oneapi/vtune-profiler.html)

### Xe-specific: Slice and Subslice utilisation

On Intel Xe (Arc and integrated), the GPU is organised into Slices → Subslices → EUs. `intel_gpu_top` shows aggregate utilisation; for per-subslice breakdown use `perf` with Intel-specific PMU events or VTune's "Microarchitecture" analysis. Imbalanced Subslice utilisation (one busy, others idle) indicates a serialisation point (e.g., a single large dispatch that doesn't distribute across all subslices).

---

## 11. Worked Case Study: Optimising a Path-Traced Scene

This section walks through a realistic optimisation investigation for a Vulkan path tracer. Numbers are illustrative; actual values will vary with hardware and workload.

### Starting point

Application: Vulkan path tracer, denoised with SVGF. Hardware: NVIDIA GeForce RTX 3080 on Ubuntu 24.04. Starting frame rate: approximately 28 FPS at 1920×1080, target 60 FPS.

### Step 1: Frame time decomposition

Insert `vkCmdWriteTimestamp2` around each pass:

```
GBuffer:           ~4 ms
Ray tracing (PT):  ~24 ms
SVGF denoise:      ~3 ms
TAA + tonemapper:  ~1 ms
Total GPU:         ~32 ms
```

The path tracing pass dominates at 24 ms (75% of frame time). Focus here.

### Step 2: ncu analysis of the ray tracing dispatch

```bash
ncu --metrics dram__throughput.avg.pct_of_peak_sustained_elapsed,\
              lts__throughput.avg.pct_of_peak_sustained_elapsed,\
              sm__throughput.avg.pct_of_peak_sustained_elapsed \
    --export pt_analysis.ncu-rep ./pathtracer
```

Results (approximate):
- DRAM throughput: ~67% of peak (544 GB/s of 912 GB/s theoretical)
- L2 throughput: ~89% of peak
- SM throughput: ~34% of peak

The GPU is **bandwidth-bound**: DRAM throughput is high while SM (compute) throughput is low. The shader is spending most of its time waiting for memory rather than computing.

### Step 3: Diagnose the bandwidth bottleneck

BVH traversal is the primary DRAM consumer in a path tracer. Each ray traverses the BVH, accessing node data (32 bytes/node). For incoherent secondary rays (scatter from glossy surfaces), BVH accesses are spatially incoherent — rays from adjacent pixels access distant parts of the BVH tree. The L2 miss rate confirms this: low temporal and spatial locality → L2 misses → DRAM access.

```
L2 hit rate = 1 - (dram_bytes / l2_bytes) ≈ 31%
```

A 31% L2 hit rate means 69% of BVH accesses miss the L2 and go to DRAM.

### Step 4: Ray sorting

Solution: sort secondary rays by BVH node morton code before dispatching the path tracing shader. Adjacent rays in the sorted order traverse similar BVH nodes, improving L2 hit rate.

Implementation: add a preprocessing compute pass that:
1. Traces primary rays, outputs hit normals and positions (already existing in GBuffer)
2. Generates secondary ray directions (given in the path tracer)
3. Computes a morton code for each secondary ray based on its direction in octahedral mapping
4. Sorts ray indices by morton code using a GPU radix sort (e.g., `VkRadixSort` from the Vulkan Samples, or CUB on NVIDIA)
5. Reorders the ray buffer before dispatching the main path tracing shader

Overhead of sort: approximately 1.5 ms per frame.

Results after sort (approximate):
- L2 hit rate: 31% → 61%
- DRAM throughput: 67% → 41% of peak
- Ray tracing pass: 24 ms → 15 ms
- Frame rate: 28 → 43 FPS

### Step 5: Residual bottleneck

After ray sorting, re-run frame decomposition:

```
GBuffer:           ~4 ms
Ray sorting:       ~1.5 ms
Ray tracing (PT):  ~15 ms
SVGF denoise:      ~3 ms
Total GPU:         ~23.5 ms → ~42 FPS
```

Still below 60 FPS. Check CPU vs GPU balance:

```bash
perf record -g -F 1000 -- ./pathtracer
perf report
```

CPU is 38% idle, GPU is 100% busy → still GPU-bound. The remaining bottleneck is in the ray tracing pass itself. A second `ncu` run shows SM throughput improved from 34% to 51%, but occupancy is still low at 28%.

VGPR check via shader source inspection: the ray tracing closest-hit shader uses approximately 80 VGPRs (complex material evaluation). On the RTX 3080 (Ampere), maximum VGPRs per thread is 255, maximum threads per SM is 1536 (48 warps). With 80 VGPRs per thread: 255/80 ≈ 3 warps per SM → 6.25% theoretical maximum occupancy. The SM runs only 3 warps instead of 48.

Fix: refactor the closest-hit shader — extract material evaluation into a separate callable shader (`vkCmdTraceRaysKHR` callable records), which reduces register pressure in the main closest-hit by offloading evaluation to a separate dispatch. This increases occupancy from 3 to approximately 8 warps per SM.

After callable shader refactor (approximate):
- Ray tracing pass: 15 ms → 10 ms
- Total GPU: approximately 18.5 ms → approximately 54 FPS

### Step 6: BVH build CPU bottleneck

At 54 FPS the GPU now has approximately 18% idle time (frame time 18.5 ms, GPU finishes at 18.5 ms but frame clock is 16.7 ms for 60 FPS). A CPU profile shows `AccelerationStructureBuild` consuming approximately 3 ms per frame on the main thread (dynamic scene with moving objects requiring full BVH rebuild).

Fix: switch from full rebuild to **refit** (`VK_BUILD_ACCELERATION_STRUCTURE_ALLOW_UPDATE_BIT_KHR`) for non-deforming objects, and async build for deforming objects using a separate compute queue. [Source: Vulkan Ray Tracing spec — AS updates](https://registry.khronos.org/vulkan/specs/latest/man/html/vkCmdBuildAccelerationStructuresKHR.html)

After async BVH refit (approximate):
- CPU BVH cost: removed from critical path
- Final frame rate: approximately 58–60 FPS

### Summary of optimisations

| Optimisation | Frame time change | Technique |
|---|---|---|
| Ray sorting (morton) | 24 ms → 15 ms | Improved L2 hit rate |
| Callable shader (VGPR) | 15 ms → 10 ms | Improved occupancy |
| Async BVH refit | ~1.5 ms CPU → overlap | CPU–GPU pipeline overlap |
| **Total** | **~32 ms → ~17 ms** | **28 → 58 FPS** |

---

## 12. eBPF-Based GPU Observability

This section targets **systems performance engineers** and **GPU driver developers** who need production-safe, kernel-level GPU visibility without modifying applications, recompiling drivers, or deploying heavyweight capture tools.

### Why eBPF for GPU Profiling

Traditional GPU profiling approaches impose significant overhead or require invasive setup. **RenderDoc** captures entire frames by injecting into the Vulkan/OpenGL call stream — it intercepts `vkQueueSubmit`, replays command buffers, and serialises GPU resources to disk. The capture overhead can exceed 10× normal execution time, making it unusable in production and unsuitable for workloads that rely on real-time timing (e.g., VR, cloud gaming). **Vendor profilers** (NVIDIA Nsight, AMD RGP) require specific driver modes and environment variables, are tied to particular GPU families, and cannot run simultaneously on multi-GPU or multi-tenant systems.

**eBPF** (Extended Berkeley Packet Filter) provides a fundamentally different approach. eBPF programs are loaded into the Linux kernel, verified for safety by the in-kernel verifier, and JIT-compiled to native machine code. They attach to **tracepoints**, **kprobes**, and **uprobes** at runtime — no recompile, no driver modification, no application restart. Overhead is sub-microsecond per probe firing; a typical bpftrace script adds fewer than 5 µs of total per-frame latency even at 240 Hz. Because eBPF programs cannot perform unbounded loops or dereference arbitrary pointers, they are safe to run in production environments, including cloud GPU instances. [Source: eBPF safety model, Linux kernel documentation](https://www.kernel.org/doc/html/latest/bpf/verifier.html)

The key contrast:

| Tool | Overhead | Requires recompile? | Multi-vendor? | Production-safe? |
|------|----------|--------------------|--------------|----|
| RenderDoc | 10–100× | No | Yes (API-level) | No |
| Nsight / RGP | 2–20× | No | No (vendor-locked) | No |
| intel_gpu_top | < 1% | No | Intel only | Yes |
| **bpftrace/eBPF** | **< 0.1%** | **No** | **Yes (DRM tracepoints)** | **Yes** |

### DRM Tracepoints: The Kernel's Observability Interface

The Linux kernel exports a stable set of GPU-related tracepoints across two subsystems. Stability is intentional: the DRM maintainers have explicitly designated certain `gpu_scheduler` tracepoints as stable uAPI so that tools like GPUVis and umr can rely on their field names without breaking across kernel updates. [Source: DRM Scheduler patch series, Jan 2025, LKML](https://lkml.iu.edu/hypermail/linux/kernel/2501.3/06940.html)

#### dma_fence tracepoints

Defined in `include/trace/events/dma_fence.h` in the kernel source. [Source: linux/include/trace/events/dma_fence.h](https://github.com/torvalds/linux/blob/master/include/trace/events/dma_fence.h)

All events share the same four fields via the `dma_fence` event class:
- `driver` (string) — the fence driver name (e.g., `"amdgpu"`, `"i915"`, `"nouveau"`)
- `timeline` (string) — timeline/context name
- `context` (unsigned int) — unique fence context identifier
- `seqno` (unsigned int) — sequence number within the context

The seven defined events are:

| Tracepoint | Fires when |
|-----------|-----------|
| `dma_fence:dma_fence_init` | A new fence is created |
| `dma_fence:dma_fence_emit` | A fence is attached to a submission |
| `dma_fence:dma_fence_enable_signal` | Signal callback is registered |
| `dma_fence:dma_fence_signaled` | GPU has finished the fenced work |
| `dma_fence:dma_fence_wait_start` | CPU begins waiting on a fence |
| `dma_fence:dma_fence_wait_end` | CPU wait completes |
| `dma_fence:dma_fence_destroy` | Fence reference count reaches zero |

The pair `dma_fence_emit` + `dma_fence_signaled` brackets the full GPU execution window. A bpftrace script can record submission timestamps and compute elapsed GPU time with no application modification:

```bpftrace
#!/usr/bin/env bpftrace
// gpu_fence_latency.bt — measure GPU fence round-trip time per driver

tracepoint:dma_fence:dma_fence_emit {
    @start[args->context] = nsecs;
}

tracepoint:dma_fence:dma_fence_signaled {
    if (@start[args->context] != 0) {
        $elapsed_us = (nsecs - @start[args->context]) / 1000;
        printf("%-12s fence ctx=%u signaled after %llu µs\n",
               str(args->driver), args->context, $elapsed_us);
        @latency_us[str(args->driver)] = hist($elapsed_us);
        delete(@start[args->context]);
    }
}

END {
    printf("\nGPU fence latency histograms (microseconds):\n");
    print(@latency_us);
    clear(@start);
}
```

#### DRM vblank tracepoints

Defined in the DRM core, `drm_vblank_event` fires each time a CRTC (display controller) completes a vertical blanking interval. Fields: `crtc` (integer, display index), `seq` (vblank sequence number), `time` (ktime_t timestamp). [Source: DRM internals documentation](https://www.kernel.org/doc/html/latest/gpu/drm-internals.html)

```bpftrace
// vblank_jitter.bt — measure display vblank interval jitter per CRTC
tracepoint:drm:drm_vblank_event {
    if (@prev_vblank[args->crtc] != 0) {
        $interval_us = (nsecs - @prev_vblank[args->crtc]) / 1000;
        @vblank_jitter[args->crtc] = hist($interval_us);
    }
    @prev_vblank[args->crtc] = nsecs;
}
END { print(@vblank_jitter); clear(@prev_vblank); }
```

A 60 Hz display should show intervals clustered around 16,667 µs; outliers indicate missed vblanks or compositor delays.

#### DRM GPU Scheduler tracepoints

Defined in `drivers/gpu/drm/scheduler/gpu_scheduler_trace.h`. [Source: linux/drivers/gpu/drm/scheduler/gpu_scheduler_trace.h](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/scheduler/gpu_scheduler_trace.h)

The scheduler events use two DEFINE_EVENT instances from the `drm_sched_job` class, plus a separate `drm_sched_job_done` event:

| Tracepoint | Fires when | Key fields |
|-----------|-----------|-----------|
| `gpu_scheduler:drm_sched_job_queue` | Job is queued in the software scheduler | `dev`, `name` (ring), `fence_context`, `fence_seqno`, `job_count`, `hw_job_count`, `client_id` |
| `gpu_scheduler:drm_sched_job_run` | Job is submitted to hardware | same as above |
| `gpu_scheduler:drm_sched_job_done` | GPU hardware completes the job | `fence_context`, `fence_seqno` |

The `job_count` field reports the number of jobs pending in the entity's software queue; `hw_job_count` reports jobs in-flight on hardware. Tracking these together reveals whether the GPU is starved (queue drains to zero) or overloaded (queue grows without bound).

List available GPU tracepoints on the running system:

```bash
sudo ls /sys/kernel/tracing/events/gpu_scheduler/
sudo ls /sys/kernel/tracing/events/dma_fence/
sudo ls /sys/kernel/tracing/events/drm/
# Inspect a specific tracepoint's fields:
sudo cat /sys/kernel/tracing/events/gpu_scheduler/drm_sched_job_run/format
```

### GPU Submission Latency Measurement

The full round-trip from a Vulkan application calling `vkQueueSubmit` to the corresponding GPU fence signalling can be measured by combining a kprobe on the DRM ioctl entry point with the `dma_fence_signaled` tracepoint. On AMD, the AMDGPU command submission ioctl is `AMDGPU_CS`; on Intel it is `DRM_IOCTL_I915_GEM_EXECBUFFER2`. The `drm_ioctl` kernel function is the common dispatch point for all DRM ioctls regardless of vendor.

```bpftrace
#!/usr/bin/env bpftrace
// gpu_submission_latency.bt
// Measures round-trip from drm_ioctl entry to dma_fence_signaled.
// Run as root: bpftrace gpu_submission_latency.bt

BEGIN {
    printf("Tracing GPU submission latency... Ctrl-C to print histogram.\n");
}

kprobe:drm_ioctl {
    // arg0 = struct file*, arg1 = unsigned int cmd, arg2 = unsigned long arg
    // Record entry time keyed by thread ID
    @submit_start[tid] = nsecs;
}

kretprobe:drm_ioctl {
    // Clear incomplete submissions (ioctls that don't produce fences)
    delete(@submit_start[tid]);
}

tracepoint:dma_fence:dma_fence_signaled {
    // For each fence signalled, find a recent submission from the same process.
    // Note: keying by pid correlates submission process to fence completion.
    if (@submit_start[pid] != 0) {
        $elapsed_us = (nsecs - @submit_start[pid]) / 1000;
        @latency_us = hist($elapsed_us);
        delete(@submit_start[pid]);
    }
}

END {
    printf("\nGPU submission-to-signal latency (µs):\n");
    print(@latency_us);
    clear(@submit_start);
}
```

> Note: The kprobe/tracepoint correlation shown above uses `pid` as a correlation key, which is correct for single-threaded submissions. In multi-threaded engines that submit from one thread and collect fences on another, extend the map key to include `args->context` (fence context) for precise correlation.

For vendor-specific ioctl filtering, check the ioctl command number:

```bpftrace
kprobe:drm_ioctl {
    // arg1 is the ioctl command number (e.g., 0xC0186440 = AMDGPU_CS)
    if (arg1 == 0xC0186440) {  // AMDGPU_CS, from uapi/drm/amdgpu_drm.h
        @amdgpu_cs_start[tid] = nsecs;
    }
}
```

On a healthy GPU at 60 FPS with simple geometry, `@latency_us` will cluster around 2,000–8,000 µs. Latencies above 33,000 µs (one frame at 30 Hz) indicate GPU scheduler stalls or fence signalling delays worth investigating.

### Shader Compilation Detection via Uprobes

One of the most common causes of frame-rate stutter in Vulkan applications is **runtime shader compilation** — a `vkCreateGraphicsPipelines` call that triggers ACO or LLVM SPIR-V→ISA compilation on the CPU, blocking the render thread for tens to hundreds of milliseconds. bpftrace uprobes can detect these events without modifying the application.

First, locate the Mesa RADV shared library and verify the symbol exists:

```bash
# Find the library path:
find /usr/lib -name "libvulkan_radeon.so" 2>/dev/null
# e.g., /usr/lib/x86_64-linux-gnu/libvulkan_radeon.so

# Verify the symbol is exported (RADV functions are not name-mangled):
nm -D /usr/lib/x86_64-linux-gnu/libvulkan_radeon.so | grep -i 'CreateGraphicsPipelines'
# Expected output: 0000000000123456 T radv_CreateGraphicsPipelines
```

Attach a uprobe to detect pipeline creation and measure compilation latency:

```bpftrace
#!/usr/bin/env bpftrace
// shader_compile_detect.bt
// Detect runtime Vulkan pipeline (shader) compilations in RADV.
// Usage: bpftrace shader_compile_detect.bt
//   (attach to specific PID with -p <pid> flag)

uprobe:/usr/lib/x86_64-linux-gnu/libvulkan_radeon.so:radv_CreateGraphicsPipelines {
    @compile_start[tid] = nsecs;
    printf("[%llu ms] PID %d (%s) calling CreateGraphicsPipelines\n",
           elapsed / 1000000, pid, comm);
    @compiles_by_process[comm] = count();
}

uretprobe:/usr/lib/x86_64-linux-gnu/libvulkan_radeon.so:radv_CreateGraphicsPipelines {
    if (@compile_start[tid] != 0) {
        $latency_ms = (nsecs - @compile_start[tid]) / 1000000;
        printf("  └─ compilation complete: %llu ms\n", $latency_ms);
        @compile_latency_ms = hist($latency_ms);
        delete(@compile_start[tid]);
    }
}

END {
    printf("\nShader compilation count by process:\n");
    print(@compiles_by_process);
    printf("\nCompilation latency distribution (ms):\n");
    print(@compile_latency_ms);
}
```

For the Intel ANV driver, the equivalent symbol is `anv_CreateGraphicsPipelines` in `libvulkan_intel.so`. For NVIDIA's open-source NVK driver in Mesa, the symbol is `nvk_CreateGraphicsPipelines` in `libvulkan_nouveau.so`.

For deeper introspection, attach to the internal compilation function to see individual shader stage compilations:

```bash
# RADV internal shader compilation entry point:
nm -D /usr/lib/.../libvulkan_radeon.so | grep 'radv_shader_spirv_to_nir\|radv_shader_compile'
```

A well-behaved Vulkan application should show zero compilations after the loading screen. Any `radv_CreateGraphicsPipelines` fires during gameplay indicate missing pipeline cache entries or VK_EXT_graphics_pipeline_library misuse (see Ch76).

To target a specific process rather than system-wide:

```bash
bpftrace -p $(pgrep -x my_game) shader_compile_detect.bt
```

### DRM Scheduler Queue Depth Analysis

The DRM GPU scheduler (`drivers/gpu/drm/scheduler/`) queues jobs per `drm_sched_entity` — an abstraction over a single Vulkan queue or OpenGL context. Under multi-process or multi-queue load, imbalanced queuing leads to GPU starvation: one entity's jobs dominate hardware while others wait.

The `drm_sched_job_run` tracepoint fires when a job leaves the software queue and is submitted to hardware. The `job_count` field at that moment reports the remaining depth of the entity's software queue:

```bpftrace
#!/usr/bin/env bpftrace
// gpu_sched_queue_depth.bt
// Monitor DRM scheduler queue depth per ring to detect GPU starvation.

tracepoint:gpu_scheduler:drm_sched_job_run {
    // args->name is the hardware ring name (e.g., "gfx_0.0", "comp_1.0")
    // args->job_count is pending software queue depth
    // args->hw_job_count is in-flight hardware job count
    @queue_depth[str(args->name)] = hist(args->job_count);
    @hw_depth[str(args->name)]    = hist(args->hw_job_count);

    if (args->job_count == 0) {
        // Queue just drained — GPU may stall waiting for next submission
        @starvation_events[str(args->name)] = count();
    }
}

interval:s:5 {
    printf("\n=== Queue depth snapshot (5s interval) ===\n");
    print(@queue_depth);
    printf("\nStarvation events (queue emptied):\n");
    print(@starvation_events);
    clear(@starvation_events);
}
```

A `job_count` that is consistently 0 when `drm_sched_job_run` fires means the GPU is receiving work one job at a time rather than batched — a CPU submission-rate bottleneck. A `job_count` that grows without bound indicates the GPU cannot keep up with CPU submission rate — a GPU compute bottleneck. Healthy operation shows a depth of 1–4 with occasional 0 readings on the primary graphics ring.

The `dev` field (added in kernel 6.14 via [commit in Jan 2025 LKML patch](https://lkml.iu.edu/hypermail/linux/kernel/2501.3/06940.html)) allows filtering by GPU device in multi-GPU systems: `if (str(args->dev) == "0000:03:00.0") { ... }`.

### Perfetto GPU Timeline via eBPF

**Perfetto** is the open-source system tracing framework used by ChromeOS, Android, and increasingly native Linux desktop tooling. Its `traced_probes` daemon uses Linux ftrace and eBPF to produce the GPU timeline track visible in the [Perfetto UI](https://ui.perfetto.dev). [Source: Perfetto documentation](https://perfetto.dev/docs/)

On Linux, `traced_probes` configures the kernel's tracefs interface to enable `gpu_scheduler`, `dma_fence`, and `drm` tracepoints, then converts the resulting event stream into Perfetto's protobuf trace format. The GPU timeline track in the Perfetto UI is constructed from these kernel tracepoints — no GPU vendor SDK is required.

A minimal `perfetto.cfg` that captures the GPU timeline alongside CPU scheduling:

```protobuf
# perfetto.cfg — GPU timeline + CPU scheduler trace
duration_ms: 10000

buffers {
  size_kb: 65536
  fill_policy: RING_BUFFER
}

# CPU scheduler events (for CPU–GPU correlation)
data_sources {
  config {
    name: "linux.ftrace"
    ftrace_config {
      ftrace_events: "sched/sched_switch"
      ftrace_events: "sched/sched_wakeup"
      # DRM GPU scheduler events (stable uAPI since kernel 6.14)
      ftrace_events: "gpu_scheduler/drm_sched_job_run"
      ftrace_events: "gpu_scheduler/drm_sched_job_done"
      ftrace_events: "gpu_scheduler/drm_sched_job_queue"
      # DMA-fence lifecycle (fence emit → signal = GPU execution window)
      ftrace_events: "dma_fence/dma_fence_emit"
      ftrace_events: "dma_fence/dma_fence_signaled"
      # Display vblank events (frame delivery timing)
      ftrace_events: "drm/drm_vblank_event"
      # GPU frequency scaling
      ftrace_events: "power/gpu_frequency"
    }
  }
}

# Mesa per-driver GPU render stage traces (requires -Dperfetto=true Mesa build)
data_sources {
  config {
    name: "gpu.renderstages.intel"   # Replace with gpu.renderstages.msm for AMD/Qualcomm
  }
}

# GPU hardware performance counters (driver-specific counter IDs)
data_sources {
  config {
    name: "gpu.counters.i915"
    gpu_counter_config {
      counter_period_ns: 1000000    # Sample every 1 ms
    }
  }
}

# Process metadata (maps PIDs to names in the UI)
data_sources {
  config {
    name: "linux.process_stats"
    process_stats_config {
      scan_all_processes_on_start: true
    }
  }
}
```

Run the capture:

```bash
# Start traced and traced_probes daemons:
sudo traced &
sudo traced_probes &

# Capture:
sudo perfetto -c perfetto.cfg -o gpu_trace.perfetto

# View: open gpu_trace.perfetto at https://ui.perfetto.dev
```

The resulting trace in the Perfetto UI shows a **GPU timeline track** with coloured slices for each GPU job (`drm_sched_job_run` → `drm_sched_job_done`), fence signal markers, vblank timing, and GPU frequency, all correlated against CPU thread scheduling on the same time axis. This is the most comprehensive view of GPU–CPU interaction available on Linux without vendor-specific tooling. Cross-reference with Ch137 for the broader GPU profiling ecosystem, including MangoHUD's overlay and AMD's ROCm Perfetto integration.

### bpftrace One-Liners Reference

The following one-liners cover the most common GPU observability tasks. Run each as root (`sudo bpftrace -e '...'`).

| Goal | bpftrace one-liner |
|------|-------------------|
| **Fence signal latency histogram** | `tracepoint:dma_fence:dma_fence_emit { @s[args->context]=nsecs; } tracepoint:dma_fence:dma_fence_signaled { @us=hist((nsecs-@s[args->context])/1000); delete(@s[args->context]); } END { print(@us); }` |
| **Vblank interval jitter** | `tracepoint:drm:drm_vblank_event { @j[args->crtc]=hist((nsecs-@p[args->crtc])/1000); @p[args->crtc]=nsecs; } END { print(@j); }` |
| **DRM ioctl rate by process** | `kprobe:drm_ioctl { @[comm]=count(); }` |
| **Shader compilation count (RADV)** | `uprobe:/usr/lib/x86_64-linux-gnu/libvulkan_radeon.so:radv_CreateGraphicsPipelines { @[comm]=count(); }` |
| **GPU scheduler queue depth** | `tracepoint:gpu_scheduler:drm_sched_job_run { @[str(args->name)]=hist(args->job_count); }` |
| **Hardware job count (in-flight)** | `tracepoint:gpu_scheduler:drm_sched_job_run { @hw[str(args->name)]=hist(args->hw_job_count); }` |
| **GPU fence wait time (CPU block)** | `tracepoint:dma_fence:dma_fence_wait_start { @w[args->context]=nsecs; } tracepoint:dma_fence:dma_fence_wait_end { @us=hist((nsecs-@w[args->context])/1000); delete(@w[args->context]); } END { print(@us); }` |
| **AMD CS ioctl rate** | `kprobe:drm_ioctl /arg1==0xC0186440/ { @[comm]=count(); }` |

> Note: The `gpu_scheduler` subsystem name in bpftrace tracepoints corresponds to the kernel's `gpu_scheduler` event class directory under `/sys/kernel/tracing/events/gpu_scheduler/`. Verify tracepoint availability on your kernel with `sudo bpftrace -l 'tracepoint:gpu_scheduler:*'` before use. The drm vblank tracepoint subsystem is `drm`, not `drm_vblank`. Uprobe paths must match your distribution's Mesa installation; adjust accordingly on distributions that install Mesa to `/usr/lib64/` or non-standard paths.

---

## 13. End-to-End Frame Delivery Latency

Understanding GPU performance in isolation misses a critical dimension: the time from when a rendered frame leaves the GPU until it reaches the user's eyes. This section quantifies every hop in the Linux graphics pipeline and shows how to measure each one.

### The Linux Frame Delivery Pipeline

A fully rendered frame traverses the following stages from application to display panel:

```
Application CPU
  └─ vkQueueSubmit() / eglSwapBuffers()
       └─ DRM scheduler queue                    ← 0.1–1 ms (fence wake latency)
            └─ GPU render                         ← 1–50 ms (workload dependent)
                 └─ compositor wl_surface.commit  ← 0.1–0.5 ms (Wayland IPC)
                      └─ compositor render pass   ← 0.5–3 ms (glass + overlay)
                           └─ KMS atomic commit   ← wait for next vblank
                                └─ scanout        ← 0 (atomic with vblank)
                                     └─ panel     ← 1–10 ms (IPS) / 0.1–1 ms (OLED)
```

#### Per-Hop Latency Budget (at 60 Hz / 16.67 ms frame budget)

| Hop | Typical latency | Measurement tool |
|-----|-----------------|-----------------|
| CPU command recording → `vkQueueSubmit` | 0.05–2 ms | `vkCmdWriteTimestamp2` before submit |
| Kernel scheduler wake-up (fence → GPU start) | 0.1–0.5 ms | `drm_sched_job_run` tracepoint |
| GPU render (vertex + fragment + post) | 1–50 ms | `vkCmdWriteTimestamp2` at pass boundaries |
| `eglSwapBuffers` / `wl_surface.commit` IPC | 0.05–0.3 ms | `wp_presentation_feedback` timestamps |
| Compositor compositing pass | 0.5–3 ms | DRM `drm_vblank_event` tracepoint |
| KMS atomic commit → vblank | 0–16.67 ms | `vblank_time` in `wp_presentation_feedback` |
| Panel pixel response | 1–10 ms (IPS), 0.1–1 ms (OLED) | Oscilloscope / manufacturer spec |

The **dominant variable** at 60 Hz is typically either GPU render time (if GPU-bound, as §4 diagnoses) or the vblank alignment penalty (if the frame misses a vblank and must wait for the next one). At 144 Hz (6.94 ms frame budget), the vblank alignment penalty becomes the most common latency source.

### Measuring Compositor Present Latency: wp\_presentation\_feedback

The `wp_presentation_feedback` protocol [Source](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/stable/presentation-time/presentation-time.xml) provides per-frame timestamps from the compositor:

```c
/* bind wp_presentation at setup */
struct wp_presentation *pres = wl_registry_bind(registry, name,
    &wp_presentation_interface, 1);
wp_presentation_add_listener(pres, &pres_listener, NULL);

/* before each commit, request feedback */
struct wp_presentation_feedback *fb =
    wp_presentation_feedback(pres, surface);
wp_presentation_feedback_add_listener(fb, &fb_listener, NULL);

static void feedback_presented(void *data,
    struct wp_presentation_feedback *feedback,
    uint32_t tv_sec_hi, uint32_t tv_sec_lo,
    uint32_t tv_nsec, uint32_t refresh,
    uint32_t seq_hi, uint32_t seq_lo,
    uint32_t flags)
{
    /* tv_sec_hi:tv_sec_lo + tv_nsec = actual scanout timestamp */
    /* refresh = display refresh interval in nanoseconds */
    /* flags: WP_PRESENTATION_FEEDBACK_KIND_VSYNC set if presented at vblank */
    uint64_t present_ns = ((uint64_t)tv_sec_hi << 32 | tv_sec_lo) * 1000000000ULL + tv_nsec;
    printf("presented at %" PRIu64 " ns, refresh %u ns\n", present_ns, refresh);
}
```

The difference between the application's `clock_gettime(CLOCK_MONOTONIC)` at `wl_surface.commit` and the `tv_sec/tv_nsec` timestamp in the feedback callback is the **compositor-to-scanout latency** for that frame.

### Measuring End-to-End with MangoHUD

MangoHUD's `present_timing` overlay [Source](https://github.com/flightlessmango/MangoHud) reports per-frame latency using `wp_presentation_feedback` internally. Enable it in `~/.config/MangoHud/MangoHud.conf`:

```ini
# MangoHud.conf
present_timing
frametime
```

The `present_timing` stat shows the interval from `vkQueuePresentKHR` to `wp_presentation_feedback`'s presented callback — the compositor pipeline latency including the vblank alignment cost. This is the most practical zero-instrumentation latency measurement available on Wayland.

### ftrace: Vblank and KMS Timing

To measure vblank interval precision and detect vblank jitter (a source of stutter even with correct frame pacing):

```bash
# Enable DRM vblank tracepoint
sudo trace-cmd record -e drm:drm_vblank_event -e drm:drm_atomic_commit_tail &
sleep 5
sudo trace-cmd stop
sudo trace-cmd report | grep drm_vblank_event | awk '{print $4}' | \
    awk 'NR>1{print $1-prev; prev=$1}1' | sort -n | uniq -c
```

The distribution of inter-vblank intervals shows whether the display is maintaining a stable refresh rate. A bimodal distribution (e.g., values clustering near 16.67 ms and 33.33 ms at 60 Hz) indicates missed vblanks.

### Input-to-Display Latency (Motion-to-Photon)

The full **motion-to-photon** latency includes input processing before the GPU render:

```
Input event (kernel evdev) → libinput → Wayland compositor → application
  → vkQueueSubmit → GPU → compositor → KMS → panel
```

On a well-configured Wayland system at 144 Hz:
- Input → application Wayland event: 1–3 ms
- Application CPU frame: 1–3 ms
- GPU render: 2–6 ms
- Compositor + KMS: 1–4 ms
- Panel: 0.5–5 ms
- **Total: 5–21 ms** (motion-to-photon)

Gamescope (Ch78) specifically targets reducing this latency via frame pacing, screen tearing control, and direct scanout plane assignment that bypasses compositor composition passes when the game is the only fullscreen surface.

### Optimisation Strategies per Hop

| Hop | Optimisation |
|-----|-------------|
| GPU render time | See §4–§11 (reduce GPU-bound bottlenecks) |
| vblank alignment | Use VRR/FreeSync (Ch112) to make the vblank follow the frame |
| Compositor pass | Use `VK_EXT_display_control` or `VK_KHR_display` for direct scanout bypassing compositor |
| Panel response | Select panels with MPRT (Moving Picture Response Time) mode enabled |
| Input latency | Enable `zwp_relative_pointer_v1`; use a compositor with low-latency input path (wlroots-based) |

## Roadmap

### Near-term (6–12 months)

- **RADV performance counter expansion for RDNA 4**: Mesa 26.0 added RADV support for new Radeon GPU Profiler 2.6 counters including LDS bank conflict counters, memory bytes counters, and memory counters as a percent of VRAM — expect continued counter coverage for RDNA 4 (GFX12) in Mesa 26.1 and 26.2. [Source](https://www.phoronix.com/news/AMD-RADV-RGP-2.6-Counters)
- **RADV Wave32 ray-tracing performance**: Mesa 26.0 switched RADV ray-tracing shaders to Wave32 execution on RDNA 3 and RDNA 4, replacing the older Wave64 path — methodology in §5 (occupancy analysis) now needs to account for this mode change when interpreting CU occupancy counters on affected hardware. [Source](https://www.gamingonlinux.com/2026/02/mesa-26-0-is-out-bringing-ray-tracing-performance-improvements-for-amd-radv/)
- **Nsight Systems 2026.x CLI/GUI parity on Linux**: NVIDIA's `nsys-ui` GUI is now distributed for Linux (Nsight Systems 2026.2.1 is current), improving the Linux-native workflow that previously required remote capture from a Windows host — reducing the gap noted in §1. [Source](https://docs.nvidia.com/nsight-systems/UserGuide/index.html)
- **eBPF GPU kernel tracepoint tooling**: Stable DRM tracepoints (`drm_run_job`, `drm_sched_job_timedout`, fence events) are increasingly being combined with eBPF programs for zero-overhead GPU job latency measurement — continued refinement of `drm:drm_vblank_event` and scheduler tracepoints is expected in Linux 6.12+. [Source](https://eunomia.dev/tutorials/xpu/gpu-kernel-driver/)
- **`VK_KHR_performance_query` broader driver coverage**: Intel ANV and Mesa RADV/TURNIP/V3DV already support the extension; ongoing work targets improving per-counter documentation and CTS coverage across all Mesa Vulkan drivers. [Source](https://www.phoronix.com/news/RADV-VK_KHR_performance_query)

### Medium-term (1–3 years)

- **eGPU: eBPF programmability extended onto GPU hardware**: The eGPU research prototype (presented at HCDS '25) dynamically compiles eBPF bytecode to PTX and injects instrumentation directly into running NVIDIA GPU kernels at runtime, enabling warp-level observability that today requires vendor tools. If upstreamed into the eBPF ecosystem, this would allow §12's eBPF-based GPU observability to reach inside shader execution rather than only the DRM scheduler boundary. [Source](https://dl.acm.org/doi/10.1145/3723851.3726984)
- **`gpu_ext`: extensible OS GPU scheduling policies via eBPF struct_ops**: A complementary research direction proposes BPF struct_ops hooks in the DRM GPU scheduler, allowing user-defined GPU scheduling policies without kernel patches — relevant to §13's frame delivery latency analysis since scheduling priority affects compositor-to-scanout latency. [Source](https://arxiv.org/html/2512.12615)
- **Unified cross-vendor GPU performance counter API above `VK_KHR_performance_query`**: Community interest (tracked on the Mesa and Khronos mailing lists) in a higher-level abstraction that normalises vendor counter semantics (e.g., mapping AMD `MemUnitBusy` and NVIDIA `l2_read_hit_rate` onto common bandwidth-utilisation concepts) — Note: needs verification as a formal working group proposal.
- **Perfetto GPU backend consolidation**: Perfetto (used in Android and increasingly on Linux desktop via traced) is expected to gain a stable DRM-backend counter source for GPU timeline events, enabling the §13 end-to-end latency analysis to be done entirely within a single Perfetto trace rather than stitching `nsys` + `wp_presentation_feedback` timestamps manually. Note: needs verification for desktop Linux target date.
- **RGP 3.x support for compute and mesh shader workloads**: Radeon GPU Profiler development is extending thread-trace coverage to task/mesh shader pipelines and compute workloads — aligning with the mesh shader extensions covered in Ch61 and the compute methodology in §5. [Source](https://www.phoronix.com/news/AMD-RADV-RGP-2.6-Counters)

### Long-term

- **GPU-internal eBPF (fully open): cross-vendor shader instrumentation**: The eGPU/bpftime prototype currently targets NVIDIA CUDA/PTX. A long-term goal in the open-source community is a vendor-agnostic eBPF-to-GPU-IR compiler path that works with RDNA (via ACO IR hooks) and Intel Xe — enabling §12's eBPF methodology to provide per-shader-invocation observability on all Linux GPU vendors without proprietary tooling. [Source](https://ebpf.foundation/ebpf-fellowship-update-tutorials-research-and-expanding-ebpf-into-gpu-and-ai/)
- **Automated performance diagnosis via ML counter analysis**: Tooling that ingests `VK_KHR_performance_query` counter streams and applies classification models to automatically identify the bottleneck category from §7's stall taxonomy (TMU latency, LDS bank conflict, export stall, bandwidth-bound) — analogous to `ncu`'s "top-down" recommendation engine but portable across vendors. Note: needs verification; early prototypes exist in academic literature but no production-quality open-source tool is shipping as of mid-2026.
- **KMS/DRM atomic commit latency tracing as a first-class kernel feature**: Current vblank latency measurement (§13) requires combining `ftrace`, `wp_presentation_feedback`, and `perf` — long-term kernel maintainers have discussed a unified `drm_atomic` latency histogram exported via `debugfs` or `sysfs` to make frame delivery latency measurement zero-setup for application developers.
- **VRR/FreeSync + GPU performance counter co-analysis**: As variable refresh rate becomes universal, frame time variability analysis must account for the display's refresh rate tracking the GPU's delivery time. Future tooling is expected to expose VRR range adherence counters alongside GPU performance counters so that per-frame counter data can be correlated with the actual scanout interval — bridging §13 and Ch112's VRR coverage. Note: needs verification.

## Perfetto GPU Tracing on Linux

This section targets **graphics application developers** and **systems performance engineers** who need cross-vendor, system-wide GPU tracing that correlates GPU execution with CPU threads, memory consumption, and display timing on Linux desktop. Section 12 above showed how Perfetto's `traced_probes` daemon consumes DRM kernel tracepoints to build the GPU timeline track; this section deepens that coverage with Perfetto's dedicated GPU data sources, the DRM FDINFO memory accounting model, and how to integrate the Perfetto C++ SDK directly into a Vulkan renderer.

### Perfetto Overview

[Perfetto](https://perfetto.dev) is Google's open-source, system-wide tracing framework. Its architecture separates concerns across three components:

- **`traced`**: the central tracing daemon that brokers connections between producers (data sources) and consumers (recording sessions). Producers register their data sources with `traced` and stream events into a shared memory ring buffer. Consumers issue `StartTracing`/`StopTracing` RPCs over a Unix socket to control sessions.
- **`traced_probes`**: the kernel data-source process. It configures Linux kernel tracing subsystems (tracefs ftrace, `perf_event`, procfs) and forwards events to `traced`. The GPU timeline track in the Perfetto UI is produced primarily by `traced_probes` reading DRM tracepoints — no GPU vendor SDK is required (see §12 for the full ftrace configuration).
- **`perfetto` CLI**: the command-line recording tool. Accepts a TextProto config file and writes a binary `trace.pb` output file. The resulting trace is viewed in the Perfetto UI at [ui.perfetto.dev](https://ui.perfetto.dev) or queried with `trace_processor_shell` via SQL.

Perfetto is not Android-only. It runs natively on Linux, is the system tracing backend for ChromeOS, and is used by Chrome/Chromium on desktop Linux to emit process-level events into the same trace as kernel GPU events. The Linux GPU tracing path (`traced_probes` + DRM tracepoints) is the same code path used in ChromeOS GPU performance work, and Perfetto is the recommended tracing framework for Mesa driver developers integrating GPU timeline data. [Source: Perfetto documentation](https://perfetto.dev/docs/)

### GPU-Specific Perfetto Data Sources on Linux

Perfetto defines several GPU-oriented data source types. The tracing service uses **exact name matching**, so the config must use the exact registered source name. For vendor-specific sources (counters, render stages), drivers register a hardware-suffixed name.

#### `linux.ftrace` — GPU Memory and Frequency

On Linux desktop, per-process GPU memory totals and GPU frequency scaling are collected via the ftrace subsystem through `linux.ftrace`, not through a dedicated GPU memory data source. The relevant ftrace event is `gpu_mem/gpu_mem_total`, which reports total GPU memory usage per process. GPU frequency is reported via `power/gpu_frequency`. [Source: Perfetto GPU data sources](https://perfetto.dev/docs/data-sources/gpu)

```protobuf
# Collect per-process GPU memory totals and GPU frequency via ftrace
data_sources {
  config {
    name: "linux.ftrace"
    ftrace_config {
      ftrace_events: "gpu_mem/gpu_mem_total"
      ftrace_events: "power/gpu_frequency"
    }
  }
}
```

> Note: The `gpu_mem/gpu_mem_total` ftrace event is supported on Android (where the kernel GPU driver emits it). On mainline Linux with DRM drivers (amdgpu, i915, nouveau), per-process GPU memory accounting is instead available via `/proc/PID/fdinfo` DRM FDINFO entries — see the "DRM FDINFO and GPU Memory Tracking" subsection below. Verify availability on your kernel with `sudo cat /sys/kernel/tracing/events/gpu_mem/gpu_mem_total/format`.

#### `gpu.renderstages` — GPU Command Buffer Timeline

`gpu.renderstages` provides a timeline of GPU render stage and compute activity: begin/end timestamps for graphics submissions, compute dispatches, and blitter operations. The data is structured as per-stage slice events (vertex, fragment, compute, etc.) in Perfetto's `GpuRenderStageEvent` proto format.

On Android, Mesa drivers (Turnip/Freedreno for Qualcomm Adreno, and Pan for Mali) export render stage events via Perfetto's producer API. On Linux desktop, Chrome's GPU process emits render stage events using the Perfetto C++ SDK (see "Integrating Perfetto in Vulkan Applications" below). The Mesa driver build option `-Dperfetto=true` enables Perfetto instrumentation in Mesa itself; when built with this option, Mesa drivers register GPU render stage producers that connect to `traced`. [Source: Mesa meson_options.txt](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/meson_options.txt)

GPU producers register with a hardware-specific suffix. For Intel ANV/i915:

```protobuf
data_sources {
  config {
    name: "gpu.renderstages.intel"
  }
}
```

For AMD (when Mesa is built with `-Dperfetto=true`), the registered name is `gpu.renderstages.msm` on Qualcomm or driver-specific on desktop. Consult `perfetto --query` on a running traced daemon to list registered producers. [Source: Perfetto data-sources documentation](https://perfetto.dev/docs/data-sources/gpu)

#### `gpu.counters` — Hardware Performance Counters

`gpu.counters` samples periodic or instrumented GPU hardware counter data — ALU utilization, cache miss rates, texture unit throughput, and similar metrics. The config specifies a sampling period in nanoseconds.

Like render stages, counter producers register with a hardware suffix. On Linux desktop systems with Intel hardware and the i915 driver:

```protobuf
data_sources {
  config {
    name: "gpu.counters.i915"
    gpu_counter_config {
      counter_period_ns: 1000000    # Sample every 1 ms
    }
  }
}
```

On AMD with RADV/amdgpu, `VK_KHR_performance_query` is the primary path for hardware counters (§3 above); a Perfetto `gpu.counters` data source for amdgpu on desktop Linux is a work in progress. On Android, Adreno and Mali counter access via `gpu.counters.adreno` and `gpu.counters.mali` is well established. [Source: Perfetto gpu.counters documentation](https://perfetto.dev/docs/data-sources/gpu)

### VK\_KHR\_performance\_query and Perfetto Integration

`VK_KHR_performance_query` (covered in depth in §3) exposes vendor hardware counters through a portable Vulkan API: `vkEnumeratePhysicalDeviceQueueFamilyPerformanceQueryCountersKHR` enumerates available counters; `VkQueryPoolPerformanceCreateInfoKHR` creates a query pool; `vkAcquireProfilingLockKHR` / `vkReleaseProfilingLockKHR` serializes collection passes. Refer to §3 for the full code pattern.

In the Perfetto integration layer, the `gpu.counters` data source on supported platforms reads hardware counters through the same underlying mechanism. The key driver support status as of mid-2026:

| Driver | VK_KHR_performance_query | Notes |
|--------|--------------------------|-------|
| AMD RADV (Mesa) | Partial | Enable with `RADV_PERFTEST=perf_counters`; RDNA 2/3 counters |
| Intel ANV (Mesa) | Partial | Via `VK_INTEL_performance_query` internally |
| ARM Panfrost (Mesa) | Partial | Limited counter set on Mali G57 and newer |
| NVIDIA NVK (Mesa) | In progress | NVK is the open-source Vulkan driver; counter support maturing |
| NVIDIA proprietary | Yes (via ncu) | Not exposed to Perfetto; use `ncu` for NVIDIA hardware (§9) |

[Source: Mesa VK_KHR_performance_query tracking](https://www.phoronix.com/news/RADV-VK_KHR_performance_query)

For applications that want to embed VK_KHR_performance_query counter data directly in a Perfetto trace, the approach is to collect counter values from the Vulkan API (§3) and emit them as Perfetto counter track events using the Perfetto C++ SDK (see below). This produces per-frame hardware counter slices on the GPU counter track in the Perfetto UI, correlated on the same time axis as CPU threads and DRM scheduler events.

### Linux Perfetto Setup and Configuration

#### Installation

The `perfetto` package is available in Ubuntu 23.10+ (Mantic Minotaur) and later:

```bash
sudo apt install perfetto
```

This installs the `perfetto` CLI, `traced`, and `traced_probes`. For older distributions or custom builds with Mesa `-Dperfetto=true` support:

```bash
git clone https://android.googlesource.com/platform/external/perfetto
cd perfetto
tools/install-build-deps
tools/gn args out/linux      # set is_debug=false
tools/ninja -C out/linux traced traced_probes perfetto trace_processor_shell
```

[Source: Perfetto build instructions](https://perfetto.dev/docs/contributing/build-instructions)

#### Full GPU Trace Configuration

The following `gpu_trace.cfg` captures GPU render stages, hardware counters (Intel i915), CPU scheduling, DRM tracepoints, and GPU memory — all on the same time axis for end-to-end frame delivery analysis as described in §13:

```protobuf
# gpu_trace.cfg — comprehensive GPU+CPU trace for Linux desktop
duration_ms: 15000

buffers {
  size_kb: 131072
  fill_policy: RING_BUFFER
}

# CPU scheduler events (for CPU-GPU correlation)
data_sources {
  config {
    name: "linux.ftrace"
    ftrace_config {
      ftrace_events: "sched/sched_switch"
      ftrace_events: "sched/sched_wakeup"
      # DRM GPU scheduler — GPU job queue/run/done lifecycle
      ftrace_events: "gpu_scheduler/drm_sched_job_queue"
      ftrace_events: "gpu_scheduler/drm_sched_job_run"
      ftrace_events: "gpu_scheduler/drm_sched_job_done"
      # DMA-fence lifecycle — fence emit to signal = GPU execution window
      ftrace_events: "dma_fence/dma_fence_emit"
      ftrace_events: "dma_fence/dma_fence_signaled"
      # Display vblank events
      ftrace_events: "drm/drm_vblank_event"
      # GPU frequency scaling
      ftrace_events: "power/gpu_frequency"
      # Per-process GPU memory totals (where available)
      ftrace_events: "gpu_mem/gpu_mem_total"
    }
  }
}

# GPU render stage timeline (Intel; replace with vendor suffix for other GPUs)
data_sources {
  config {
    name: "gpu.renderstages.intel"
  }
}

# GPU hardware performance counters (Intel i915)
data_sources {
  config {
    name: "gpu.counters.i915"
    gpu_counter_config {
      counter_period_ns: 2000000    # Sample every 2 ms
    }
  }
}

# Process metadata (maps PIDs to names in the UI)
data_sources {
  config {
    name: "linux.process_stats"
    process_stats_config {
      scan_all_processes_on_start: true
      proc_stats_poll_ms: 1000
    }
  }
}
```

#### Recording and Visualizing

```bash
# Start the daemons (if not running as system services):
sudo traced &
sudo traced_probes &

# Record:
sudo perfetto --config gpu_trace.cfg --out trace.pb

# Visualize: upload trace.pb to https://ui.perfetto.dev
# Or query with SQL:
trace_processor_shell trace.pb
# > SELECT ts, dur, name FROM slice WHERE category = 'gpu' LIMIT 20;
```

The Perfetto UI at [ui.perfetto.dev](https://ui.perfetto.dev) renders the GPU timeline track (from `gpu_scheduler` tracepoints), fence signal markers, GPU counter tracks, CPU thread lanes, and vblank events all on a single time-aligned axis. See §12 for the deeper eBPF-level GPU observability using the same tracepoints.

`trace_processor_shell` exposes the trace as a SQL database. Useful queries for GPU analysis:

```sql
-- GPU job duration histogram (from DRM scheduler tracepoints)
SELECT
  CAST((dur / 1e6) AS INT) AS dur_ms_bucket,
  COUNT(*) AS job_count
FROM slice
WHERE name GLOB 'drm_sched_job*'
GROUP BY dur_ms_bucket
ORDER BY dur_ms_bucket;

-- Per-process GPU memory over time (where gpu_mem/gpu_mem_total is available)
SELECT ts, CAST(value AS INT64) AS gpu_bytes, process.name
FROM counter
JOIN process_counter_track ON counter.track_id = process_counter_track.id
JOIN process USING (upid)
WHERE process_counter_track.name = 'GPU Memory'
ORDER BY ts;
```

[Source: Perfetto trace processor SQL documentation](https://perfetto.dev/docs/analysis/trace-processor)

### DRM FDINFO and GPU Memory Tracking

On mainline Linux with DRM drivers, per-process GPU resource usage is exposed through `/proc/PID/fdinfo/FD` for each open file descriptor pointing to a DRM device. This is the mechanism `traced_probes` and tools like `gfx-pps` read to track per-client GPU memory consumption. [Source: Linux kernel DRM usage stats documentation](https://www.kernel.org/doc/html/latest/gpu/drm-usage-stats.html)

#### Standardized Key Schema

The DRM FDINFO format uses a set of standardized keys (available since kernel 5.19 for i915 and amdgpu):

**Identification:**
- `drm-driver`: driver name string (e.g., `amdgpu`, `i915`, `nouveau`)
- `drm-pdev`: PCI device address (e.g., `0000:03:00.0`)
- `drm-client-id`: unique integer identifying this file descriptor

**Engine utilization** (per named engine):
- `drm-engine-<name>`: accumulated busy time in nanoseconds for the named engine. Common names: `render`, `copy`, `video`, `video-enhance`, `compute`
- `drm-engine-capacity-<name>`: number of identical hardware engines of this type
- `drm-cycles-<name>`: busy cycles in the engine's clock domain
- `drm-total-cycles-<name>`: total elapsed cycles (used to compute utilization fraction: `drm-cycles / drm-total-cycles`)

**Memory regions** (per named region, values in KiB or MiB with unit suffix):
- `drm-total-<region>`: all GEM buffers ever allocated by this client for this region
- `drm-resident-<region>`: buffers with instantiated backing store (physically present)
- `drm-shared-<region>`: buffers shared with other clients via DMA-BUF
- `drm-purgeable-<region>`: resident buffers eligible for eviction by the kernel
- `drm-active-<region>`: buffers currently being accessed by GPU engines

Common region names: `system` (system RAM, used for GTT-mapped memory), `gtt` (Graphics Translation Table aperture), `vram` (device-local VRAM on discrete GPUs), `stolen` (reserved memory on Intel integrated).

> Note: The older `drm-memory-<region>` key is a deprecated amdgpu alias for `drm-resident-<region>`. Newer kernels expose `drm-resident-vram`, `drm-resident-gtt`, etc. Use the standardized form. [Source: DRM usage stats kernel doc](https://www.kernel.org/doc/html/latest/gpu/drm-usage-stats.html)

#### Reading FDINFO from Shell

```bash
# List all open DRM fds for a process:
PID=$(pgrep -x my_vulkan_app)
for fd in /proc/$PID/fdinfo/*; do
    if grep -q '^drm-driver' "$fd" 2>/dev/null; then
        echo "=== $fd ==="
        grep '^drm-' "$fd"
    fi
done
```

Example output on an AMD system (VRAM = discrete memory, gtt = system RAM mapped into GPU address space):

```
=== /proc/12345/fdinfo/5 ===
drm-driver:      amdgpu
drm-pdev:        0000:03:00.0
drm-client-id:   42
drm-engine-render:   148367 ns
drm-engine-copy:     0 ns
drm-total-vram:      512 MiB
drm-resident-vram:   487 MiB
drm-shared-vram:     12 MiB
drm-purgeable-vram:  0 MiB
drm-active-vram:     256 MiB
drm-total-gtt:       128 MiB
drm-resident-gtt:    103 MiB
```

#### How Perfetto and gfx-pps Read FDINFO

`traced_probes` does not currently have a dedicated DRM FDINFO data source on Linux desktop; instead it relies on `gpu_mem/gpu_mem_total` ftrace events where available (Android/ChromeOS). For Linux desktop, per-process VRAM tracking can be done by polling FDINFO from a custom Perfetto producer or from `gfx-pps` (GPU Performance HUD), which reads `/proc/PID/fdinfo` on a configurable interval and reports `drm-resident-vram` as the primary VRAM usage metric.

Tools like `nvtop` and MangoHUD's GPU memory display also parse the same FDINFO keys, using `drm-resident-vram` (discrete GPUs) and `drm-resident-gtt` (integrated GPUs) for the "VRAM used" figure. [Source: nvtop FDINFO parsing](https://github.com/Syllo/nvtop)

### Integrating Perfetto in Vulkan Applications

The Perfetto C++ SDK lets a Vulkan renderer emit custom trace events that appear in the Perfetto UI on the same timeline as kernel GPU tracepoints, compositor events, and CPU thread scheduling. This is how Chrome's GPU process emits "RenderPass" and "DrawCall" slices that correlate with the DRM scheduler's `drm_sched_job_run` events.

#### SDK Setup

Download the single-file SDK from [perfetto releases](https://github.com/google/perfetto/releases/latest) (`perfetto-cpp-sdk-src.zip`) — it contains two files: `sdk/perfetto.h` and `sdk/perfetto.cc`.

CMake integration:

```cmake
# CMakeLists.txt
include_directories(perfetto/sdk)
add_library(perfetto STATIC perfetto/sdk/perfetto.cc)
target_link_libraries(my_vulkan_app perfetto ${CMAKE_THREAD_LIBS_INIT})
```

#### Defining Trace Categories

In one translation unit, declare categories and their storage:

```cpp
// trace_categories.h
#include <perfetto.h>

PERFETTO_DEFINE_CATEGORIES(
    perfetto::Category("gpu")
        .SetDescription("Vulkan GPU render stages"),
    perfetto::Category("gpu.memory")
        .SetDescription("GPU memory allocation events")
);

// trace_categories.cc
#include "trace_categories.h"
PERFETTO_TRACK_EVENT_STATIC_STORAGE();
```

#### Connecting to the traced Daemon

Initialize with the system backend so that GPU events merge with kernel tracepoints on the same timeline:

```cpp
#include "trace_categories.h"

void InitPerfetto() {
    perfetto::TracingInitArgs args;
    // System backend connects to the running traced daemon.
    // Events from this process will appear on the same timeline as
    // kernel DRM tracepoints captured by traced_probes.
    args.backends |= perfetto::kSystemBackend;
    perfetto::Tracing::Initialize(args);
    perfetto::TrackEvent::Register();
}
```

This requires `traced` to be running on the system. The connection is over a Unix socket (`/run/perfetto-producer`). When `traced` is not running, events are silently dropped — there is no recording overhead if no session is active.

#### Wrapping Vulkan Command Buffer Recording

Emit custom "GPU render stage" events around Vulkan command buffer recording to mark render pass boundaries in the trace:

```cpp
#include "trace_categories.h"

void RecordGBufferPass(VkCommandBuffer cmd, GBufferResources& res) {
    // TRACE_EVENT emits a begin-event at this line and an end-event
    // at end-of-scope (RAII destructor). The "gpu" category must match
    // a PERFETTO_DEFINE_CATEGORIES entry.
    TRACE_EVENT("gpu", "GBufferPass");

    VkRenderPassBeginInfo rpInfo = { /* ... */ };
    vkCmdBeginRenderPass(cmd, &rpInfo, VK_SUBPASS_CONTENTS_INLINE);
    // ... draw calls ...
    vkCmdEndRenderPass(cmd);
}

void RecordPathTracingDispatch(VkCommandBuffer cmd, uint32_t w, uint32_t h) {
    TRACE_EVENT("gpu", "PathTrace",
        // Attach key-value annotations visible on hover in the UI:
        "width", w, "height", h);
    vkCmdDispatch(cmd, (w + 7) / 8, (h + 7) / 8, 1);
}
```

Note that `TRACE_EVENT` timestamps are CPU-side (when the Vulkan API call is made), not GPU-side (when the GPU executes the work). For true GPU-side timestamps, use `vkCmdWriteTimestamp2` (§2) and emit the resulting GPU timestamps as Perfetto counter events:

```cpp
// After frame completion, read back GPU timestamps (§2) and emit:
void EmitGPUTimingToPerfetto(uint64_t gbuffer_start_ns,
                              uint64_t gbuffer_end_ns) {
    // Create a named track for the GPU timeline:
    auto gpu_track = perfetto::Track(0xGPU_TRACK_UUID);
    perfetto::TrackEvent::SetTrackDescriptor(gpu_track, [](
            perfetto::protos::gen::TrackDescriptor* desc) {
        desc->set_name("GPU Timeline (vkTimestamp)");
    });

    // Emit a slice using absolute GPU timestamps:
    TRACE_EVENT_BEGIN("gpu", "GBufferPass", gpu_track,
                      gbuffer_start_ns);
    TRACE_EVENT_END("gpu", gpu_track, gbuffer_end_ns);
}
```

[Source: Perfetto tracing SDK documentation](https://perfetto.dev/docs/instrumentation/tracing-sdk)

### Comparison with Other Linux GPU Profiling Tools

Perfetto occupies a distinct position in the Linux GPU profiling ecosystem. Understanding the boundaries between tools prevents duplication of effort.

| Tool | Scope | Vendor | Overhead | Primary use |
|------|-------|--------|----------|-------------|
| **Perfetto** | System-wide: CPU + GPU + memory + display | Cross-vendor | < 1% | Correlation across subsystems, production profiling |
| **RenderDoc** (Ch30) | Single-frame API capture + shader debug | Cross-vendor (API) | 10–100× | Rendering correctness + per-draw investigation |
| **AMD RGP** | GPU execution timeline, pipeline analysis | AMD only | 2–5× | Deep AMD pipeline analysis, barrier costs |
| **NVIDIA ncu/nsys** (§9) | Per-kernel counters + CPU timeline | NVIDIA only | 2–20× | NVIDIA occupancy, roofline, stall analysis |
| **Intel VTune / GPA** (§10) | EU utilization, per-kernel analysis | Intel only | 2–10× | Intel EU active/stall/idle breakdown |
| **gfx-pps** | Lightweight overlay (DRM FDINFO + counters) | Cross-vendor | < 0.5% | Real-time HUD; less detail than Perfetto |
| **MangoHUD** (Ch29) | Frame time + GPU/CPU utilization overlay | Cross-vendor | < 0.5% | In-game HUD; present timing via wp_presentation_feedback |
| **bpftrace / eBPF** (§12) | Kernel-level fence/scheduler events | Cross-vendor | < 0.1% | Production-safe scheduler and fence observability |

**Perfetto's key advantages over vendor profilers:**
- **System-wide correlation**: A Perfetto trace shows GPU execution (`drm_sched_job_run` → `drm_sched_job_done`), CPU thread scheduling, vblank timing, and Wayland compositor events all on one time axis. Vendor profilers show only their GPU's view.
- **Cross-vendor**: Works on AMD, Intel, NVIDIA (open NVK), ARM Mali (Panfrost) — any driver that implements DRM tracepoints.
- **Open-source**: The entire stack from `traced_probes` through `ui.perfetto.dev` is Apache-2.0 licensed and publicly developed.
- **Production safety**: `traced_probes` configures kernel ftrace; overhead is sub-percent and it can run on production systems without application modification.

**Where vendor tools remain superior:**
- **AMD RGP**: thread-trace capture (instruction-level GPU execution timeline, RADV `RADV_THREAD_TRACE=1`) has no Perfetto equivalent — it captures the actual ISA instruction stream executed on each CU. Perfetto sees job boundaries; RGP sees instructions.
- **NVIDIA ncu**: per-warp counter collection with workload replay enables the roofline model and occupancy analysis (§9). Perfetto's `gpu.counters` data source samples counters periodically without replay; it cannot produce roofline data.
- **RenderDoc**: full API-level frame capture and shader replay (Ch30) is complementary. Use RenderDoc when you need to inspect the rendering output (wrong pixel values, incorrect draw order); use Perfetto when you need to understand *timing* and *system interaction*.

The recommended workflow is Perfetto-first for framing the performance investigation (system-wide, low-overhead, identifies which subsystem and which timeframe), then vendor tools for the targeted deep-dive into the bottleneck confirmed by Perfetto.

---

## 14. Integrations

- **Ch15 (ACO shader compiler)** — register pressure (VGPR count) that limits occupancy is determined at ACO compilation time. `RADV_DEBUG=shaders` shows ACO's VGPR allocation decisions, and the ACO IR can be inspected to understand why pressure is high.

- **Ch24 (Vulkan compute)** — timestamp queries (`vkCmdWriteTimestamp2`) and `VK_KHR_performance_query` are Vulkan API features covered in depth here; §2 and §3 of Ch24 describe the underlying resource model.

- **Ch28 (DXVK/VKD3D-Proton)** — DXVK adds a translation layer above Vulkan. CPU profiling will show `d3d11_context::*` calls appearing as overhead in `vkQueueSubmit`. Translation overhead typically adds 1–3% CPU cost; visible in `perf report` as `libd3d11.so` frames.

- **Ch29 (MangoHUD)** — MangoHUD's CPU/GPU frame time bars use NVML / AMD PMC / DRM sysfs to read GPU timing data in real time. Understanding §4's GPU-vs-CPU analysis helps interpret MangoHUD output correctly (don't mistake fence-wait for CPU-bound).

- **Ch30 (Debugging and Profiling tools)** — Ch30 surveys available tools; this chapter provides the *methodology* for using them: what to measure, in what order, and what the numbers mean.

- **Ch56 (Ray Tracing on Linux)** — the worked case study (§11) uses ray tracing acceleration structures (`vkCmdTraceRaysKHR`, `vkCmdBuildAccelerationStructuresKHR`). Ch56 covers the API and driver implementation; this chapter covers the performance diagnosis.

- **Ch61 (Modern Vulkan Extensions)** — `VK_KHR_performance_query` is one of the extensions surveyed in Ch61's timeline extension group. This chapter provides the detailed usage pattern.

- **Ch67 (DLSS)** — before deciding to use DLSS (Ch67) or FSR3 (Ch72) to hit a frame rate target, Ch93's methodology should confirm the application is GPU-bound and identify which passes are the bottleneck. DLSS is most effective when the path tracer pass (§11 example) is the bottleneck, not post-processing or CPU work.

- **Ch87 (LLM Inference on Linux)** — bandwidth analysis (§6) applies directly to GEMM kernels in LLM inference. Arithmetic intensity of batch-1 token generation GEMM is typically 1–2 FLOPS/byte, far below the roofline boundary, confirming it as bandwidth-bound. The mitigation (quantisation reducing bytes loaded) mirrors texture compression in graphics workloads.

- **Ch137 (GPU Profiling Ecosystem)** — §12 (eBPF-Based GPU Observability) describes how Perfetto's `traced_probes` daemon uses DRM tracepoints to build the GPU timeline track visible in the Perfetto UI; Ch137 covers the broader profiling ecosystem including MangoHUD, shader-db, and vendor profiler GUIs that complement the eBPF-level instrumentation shown in §12.
