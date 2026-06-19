# Chapter 93: GPU Performance Analysis Methodology

This chapter targets **graphics application developers**, **game developers**, **ML engineers**, and **systems performance engineers** who need to systematically diagnose and improve GPU performance on Linux. Unlike Chapter 30 (which surveys debugging and profiling tools), this chapter focuses on *methodology*: how to structure an investigation, how to interpret hardware counter data, and how to move from a symptom to a root cause to a fix.

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
12. [Integrations](#12-integrations)

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

## 12. Integrations

- **Ch15 (ACO shader compiler)** — register pressure (VGPR count) that limits occupancy is determined at ACO compilation time. `RADV_DEBUG=shaders` shows ACO's VGPR allocation decisions, and the ACO IR can be inspected to understand why pressure is high.

- **Ch24 (Vulkan compute)** — timestamp queries (`vkCmdWriteTimestamp2`) and `VK_KHR_performance_query` are Vulkan API features covered in depth here; §2 and §3 of Ch24 describe the underlying resource model.

- **Ch28 (DXVK/VKD3D-Proton)** — DXVK adds a translation layer above Vulkan. CPU profiling will show `d3d11_context::*` calls appearing as overhead in `vkQueueSubmit`. Translation overhead typically adds 1–3% CPU cost; visible in `perf report` as `libd3d11.so` frames.

- **Ch29 (MangoHUD)** — MangoHUD's CPU/GPU frame time bars use NVML / AMD PMC / DRM sysfs to read GPU timing data in real time. Understanding §4's GPU-vs-CPU analysis helps interpret MangoHUD output correctly (don't mistake fence-wait for CPU-bound).

- **Ch30 (Debugging and Profiling tools)** — Ch30 surveys available tools; this chapter provides the *methodology* for using them: what to measure, in what order, and what the numbers mean.

- **Ch56 (Ray Tracing on Linux)** — the worked case study (§11) uses ray tracing acceleration structures (`vkCmdTraceRaysKHR`, `vkCmdBuildAccelerationStructuresKHR`). Ch56 covers the API and driver implementation; this chapter covers the performance diagnosis.

- **Ch61 (Modern Vulkan Extensions)** — `VK_KHR_performance_query` is one of the extensions surveyed in Ch61's timeline extension group. This chapter provides the detailed usage pattern.

- **Ch67 (DLSS)** — before deciding to use DLSS (Ch67) or FSR3 (Ch72) to hit a frame rate target, Ch93's methodology should confirm the application is GPU-bound and identify which passes are the bottleneck. DLSS is most effective when the path tracer pass (§11 example) is the bottleneck, not post-processing or CPU work.

- **Ch87 (LLM Inference on Linux)** — bandwidth analysis (§6) applies directly to GEMM kernels in LLM inference. Arithmetic intensity of batch-1 token generation GEMM is typically 1–2 FLOPS/byte, far below the roofline boundary, confirming it as bandwidth-bound. The mitigation (quantisation reducing bytes loaded) mirrors texture compression in graphics workloads.
