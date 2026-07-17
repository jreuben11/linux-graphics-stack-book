# Chapter 137: GPU Performance Profiling Ecosystem on Linux

This chapter targets the following practitioners conducting GPU profiling and tuning:

- **Graphics performance engineers**
- **Game developers** optimizing GPU workloads
- **Mesa driver developers**
- **Systems engineers**

It covers every layer of the Linux GPU profiling stack — from hardware performance counters exposed through the kernel's `perf` subsystem, through driver-level instrumentation in Mesa, to vendor-specific frame profilers and Vulkan API-level queries — and shows how these tools compose into a systematic optimization workflow.

> **Scope:** Ch137 surveys the **Linux GPU profiling tool ecosystem**:
> - **MangoHud**
> - **RenderDoc** frame analysis
> - **Mesa shader-db**
> - Vendor profilers: **AMD RGP**, **Intel GPA**, **NVIDIA Nsight**
> - **Perfetto** GPU timeline
> - Linux **`perf`** GPU counter path
>
> **Ch30** covers development-time debugging (validation layers, gdb, Mesa debug environment variables). **Ch93** covers performance analysis methodology (investigation structure, counter interpretation, bottleneck root-cause analysis). Readers who want to know *what tool to use* are in the right place; readers who want to know *how to interpret the results* should also read Ch93.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [GPU Performance Counter Architecture](#2-gpu-performance-counter-architecture)
3. [AMD Radeon GPU Profiler (RGP)](#3-amd-radeon-gpu-profiler-rgp)
4. [Intel GPU Profiling Tools](#4-intel-gpu-profiling-tools)
5. [NVIDIA Nsight and perf on Linux](#5-nvidia-nsight-and-perf-on-linux)
6. [Mesa Built-in Instrumentation](#6-mesa-built-in-instrumentation)
7. [Vulkan Timestamp and Statistics Queries](#7-vulkan-timestamp-and-statistics-queries)
8. [System-Level GPU Profiling](#8-system-level-gpu-profiling)
9. [Flame Graphs and Profiling Workflows](#9-flame-graphs-and-profiling-workflows)
10. [Integrations](#10-integrations)

---

## 1. Introduction

GPU performance profiling on Linux is a layered discipline. At the bottom, GPU silicon exposes fixed-function hardware performance counters — cycle counts, cache hit rates, memory bandwidth bytes — that the kernel makes accessible through its `perf` Performance Monitoring Unit (PMU) framework or through driver-level ioctls. Above that sits the graphics API layer: Vulkan's `VK_KHR_performance_query` extension and timestamp query pools give applications a portable way to measure GPU-side timing without vendor-specific code. Higher still, Mesa and each vendor driver expose their own instrumentation — environment variables that toggle heads-up displays, generate CSV timing files, or trigger frame-level hardware traces. Finally, vendor desktop tools — AMD's Radeon GPU Profiler, Intel's Graphics Performance Analyzers, NVIDIA's Nsight Graphics — tie all of this together into interactive frame debuggers with wavefront-level or EU-level granularity.

The Linux landscape differs meaningfully from Windows. On Windows, PIX for Windows and NVIDIA Nsight Graphics are the dominant frame profilers. On Linux, no single tool occupies that position: AMD ships a native Qt-based RGP client; Intel's tooling straddles open-source (`intel_gpu_top`, `INTEL_MEASURE`) and proprietary (GPA, VTune); NVIDIA's tooling requires the proprietary driver; and cross-vendor tooling (`perf`, Perfetto, Vulkan query pools) fills the gaps. Understanding when to reach for each layer — and how to compose the outputs — is the core skill this chapter develops.

A useful mental model: CPU profiling gives you the *call graph*; GPU profiling tells you what the *GPU was doing* during each CPU-side call. Both signals are needed. A workload that looks CPU-bound in `perf` may be waiting for GPU timeline semaphores; a workload that shows high GPU utilization may be hiding severe shader occupancy problems invisible to system-level monitors.

---

## 2. GPU Performance Counter Architecture

### 2.1 Fixed-Function Hardware Counters

Every modern GPU contains hundreds of fixed-function hardware performance counters. Unlike CPU performance counters, which are generally uniform across cores, GPU counters are architecturally heterogeneous: there are counters on the shader array, the rasterizer, the texture unit, the L2 cache, and the memory interface — each with their own sampling constraints. Understanding the counter taxonomy is prerequisite to interpreting any profiler output.

**AMD GCN/RDNA counters.** AMD's RDNA 3 and RDNA 4 GPU architectures expose counters through the SQ (Shader Sequencer) block, the GL2 (generalized L2 cache), the SPI (Shader Processor Input), and the memory interface. Key counters include:

- `SQ_PERFCOUNTER_*`: Instruction issue cycles, wavefront counts, ALU busy cycles, VMEM (vector memory) latency.
- `SPI_PERFCOUNTER_*`: Wavefronts dispatched per compute unit, resource utilization stalls.
- `GL2C_PERFCOUNTER_*`: L2 cache read/write hit and miss counts.
- `MC_PERFCOUNTER_*`: VRAM read/write bandwidth in 64-byte transactions.

**Intel Xe counters.** Intel Xe (Graphics) exposes Execution Unit (EU) utilization, sampler throughput (texels per clock), render engine stall cycles, and L3 cache traffic. The i915 OA (Observation Architecture) subsystem, and its successor under the Xe driver, exposes pre-defined *metric sets* (groups of counters that can be sampled together) rather than raw counter addresses.

**NVIDIA SM counters.** NVIDIA GPUs expose Streaming Multiprocessor (SM) utilization, warp occupancy, L2 cache read/write bandwidth in bytes, DRAM bytes read/written, and NVLink bandwidth. These are accessible through the NVIDIA Performance SDK (`nv_perf_sdk`) and through `nvmlDeviceGetPerformanceState()` in NVML.

### 2.2 The Counter Access Problem

Hardware performance counters are privileged resources. Uncontrolled counter access allows timing side-channel attacks across security domains. Consequently, most GPU drivers gate counter access:

- AMD: The `amdgpu` kernel driver requires `CAP_PERFMON` (Linux 5.8+) or root to enable hardware counters via perf.
- Intel: i915 OA counters are gated by the `perf_stream_paranoid` sysctl (`/proc/sys/dev/i915/perf_stream_paranoid=1` by default; set to 0 for developer-mode unprivileged access).
- NVIDIA: Counter access through `nv_perf_sdk` requires either root or the driver's `NvPerfSdk` privilege grant mechanism.

Vulkan's `VK_KHR_performance_query` extension handles this at the driver level by acquiring an exclusive *profiling lock* that serializes counter collection.

### 2.3 VK_KHR_performance_query

`VK_KHR_performance_query` ([Khronos Vulkan spec](https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_performance_query.html)) is the cross-vendor, standardized Vulkan API for GPU performance counter access. Note: the extension name ends in `KHR`, not `EXT`; there is no `VK_EXT_performance_query`. The extension was promoted from vendor-specific extensions (including `VK_INTEL_performance_query`) into a cross-vendor KHR extension.

**Counter discovery:**

```c
// Enumerate available counters for a queue family
uint32_t counterCount = 0;
vkEnumeratePhysicalDeviceQueueFamilyPerformanceQueryCountersKHR(
    physicalDevice,
    queueFamilyIndex,
    &counterCount, NULL, NULL);

VkPerformanceCounterKHR *counters =
    malloc(counterCount * sizeof(VkPerformanceCounterKHR));
VkPerformanceCounterDescriptionKHR *counterDescs =
    malloc(counterCount * sizeof(VkPerformanceCounterDescriptionKHR));

vkEnumeratePhysicalDeviceQueueFamilyPerformanceQueryCountersKHR(
    physicalDevice,
    queueFamilyIndex,
    &counterCount, counters, counterDescs);
```

Each `VkPerformanceCounterKHR` carries:

- `.unit`: `VK_PERFORMANCE_COUNTER_UNIT_CYCLES_KHR`, `VK_PERFORMANCE_COUNTER_UNIT_BYTES_KHR`, `VK_PERFORMANCE_COUNTER_UNIT_NANOSECONDS_KHR`, `VK_PERFORMANCE_COUNTER_UNIT_PERCENTAGE_KHR`, and others.
- `.storage`: `VK_PERFORMANCE_COUNTER_STORAGE_UINT64_KHR`, `VK_PERFORMANCE_COUNTER_STORAGE_FLOAT64_KHR`, etc. — this determines how to read `VkPerformanceCounterResultKHR` union members.
- `.scope`: `VK_PERFORMANCE_COUNTER_SCOPE_COMMAND_BUFFER_KHR`, `VK_PERFORMANCE_COUNTER_SCOPE_RENDER_PASS_KHR`, `VK_PERFORMANCE_COUNTER_SCOPE_COMMAND_KHR`.

**Creating a performance query pool and acquiring the lock:**

```c
// Determine how many submission passes are required
VkQueryPoolPerformanceCreateInfoKHR perfCreateInfo = {
    .sType = VK_STRUCTURE_TYPE_QUERY_POOL_PERFORMANCE_CREATE_INFO_KHR,
    .queueFamilyIndex = queueFamilyIndex,
    .counterIndexCount = selectedCounterCount,
    .pCounterIndices = selectedCounterIndices,
};

uint32_t numPasses = 0;
vkGetPhysicalDeviceQueueFamilyPerformanceQueryPassesKHR(
    physicalDevice, &perfCreateInfo, &numPasses);

VkQueryPoolCreateInfo poolCI = {
    .sType = VK_STRUCTURE_TYPE_QUERY_POOL_CREATE_INFO,
    .pNext = &perfCreateInfo,
    .queryType = VK_QUERY_TYPE_PERFORMANCE_QUERY_KHR,
    .queryCount = 1,
};
VkQueryPool perfPool;
vkCreateQueryPool(device, &poolCI, NULL, &perfPool);

// Acquire the exclusive profiling lock
VkAcquireProfilingLockInfoKHR lockInfo = {
    .sType = VK_STRUCTURE_TYPE_ACQUIRE_PROFILING_LOCK_INFO_KHR,
    .timeout = UINT64_MAX,
};
vkAcquireProfilingLockKHR(device, &lockInfo);
```

The workload must be submitted `numPasses` times, each time with a `VkPerformanceQuerySubmitInfoKHR` specifying the pass index. This is because a single GPU submission can only simultaneously program a limited number of physical counter mux paths.

**Reading results:**

```c
VkPerformanceCounterResultKHR results[MAX_COUNTERS];
vkGetQueryPoolResults(device, perfPool, 0, 1,
    selectedCounterCount * sizeof(VkPerformanceCounterResultKHR),
    results, sizeof(VkPerformanceCounterResultKHR),
    VK_QUERY_RESULT_WAIT_BIT);

vkReleaseProfilingLockKHR(device);
```

### 2.4 Linux Kernel perf PMU for GPU Counters

The Linux `perf` subsystem provides a unified interface to hardware PMU events. GPU drivers expose their counters through the PMU framework under `/sys/bus/event_source/devices/`:

```bash
# AMD GPU events (requires CONFIG_PERF_EVENTS_AMD in kernel)
ls /sys/bus/event_source/devices/amdgpu/events/

# Sample a specific AMD counter during workload execution
perf stat -e amdgpu/ta_busy_cnt/ -- ./vulkan-app

# Intel i915 events
ls /sys/bus/event_source/devices/i915/events/
perf stat -e i915/rcs-busy/ -e i915/vcs-busy/ -- ./vulkan-app
```

GPU PMU counters through `perf` are system-wide rather than per-process, and GPU ring counters cannot be attributed to individual draw calls — they are best used to establish system-level baselines and detect GPU frequency throttling.

---

## 3. AMD Radeon GPU Profiler (RGP)

### 3.1 Overview

The Radeon GPU Profiler ([GPUOpen](https://gpuopen.com/rgp/)) is AMD's primary GPU profiling tool for RDNA and GCN hardware. RGP captures a complete frame-level GPU trace with **wavefront-level granularity**, meaning it can show exactly which wavefronts ran on which Compute Unit, when they started and ended, why they stalled, and what instructions they were executing. This distinguishes it from system-level monitors that show only aggregate utilization.

RGP 2.7 (June 2026) supports:
- AMD RDNA 4 (RX 9000 series), RDNA 3 (RX 7000 series), RDNA 2 (RX 6000 series), RDNA (RX 5000 series)
- Vulkan API on Linux (Ubuntu 24.04 LTS)
- DirectX 12 and Vulkan on Windows

The Linux client is a native Qt application available in the Radeon Developer Tool Suite (`RadeonDeveloperToolSuite-2026-*.tgz` from [GitHub releases](https://github.com/GPUOpen-Tools/radeon_gpu_profiler/releases)).

### 3.2 SQ Thread Trace (SQTT)

The core of RGP's wavefront-level data is **SQ Thread Trace** (SQTT), a hardware mechanism in AMD's SQ (Shader Sequencer) block that records wavefront dispatch and retire events to an in-GPU memory ring buffer. SQTT data includes:

- Wavefront start/end timestamps per Compute Unit
- Instruction issue stall reasons (VMEM latency, LDS bank conflicts, barrier waits)
- Register occupancy events
- Draw/dispatch boundaries from the command processor

Because SQTT records at the hardware level, it captures activity invisible to API-level tracing — for example, shader pre-emption events, SPI occupancy cliffs caused by register pressure, and inter-wavefront barrier synchronization costs.

SQTT is also the file format (`*.rgp`) consumed by the RGP GUI; RADV generates these files directly on Linux. The configurable **SQTT Buffer Size** (default 32 MB per shader engine) can be increased for long frames:

```bash
# Increase SQTT buffer via Radeon Developer Panel, or set via env:
MESA_VK_TRACE_BUFFER_SIZE=64 ./vulkan-app
```

### 3.3 Capture Mechanism

RGP captures are triggered through the **Radeon Developer Panel** (RDP), a GUI control panel that communicates with the `RadeonDeveloperServiceCLI` daemon:

```bash
# Start the developer service (listens on port 27300 by default)
RadeonDeveloperServiceCLI --port 27300

# In another terminal, launch RGP and connect
RGP --connect localhost
```

RDP acts as a bridge: it instruments the AMDGPU driver to enable SQTT on the next frame, captures the resulting ring buffer, and streams the `.rgp` file to the RGP GUI.

**RADV (Mesa AMD Vulkan) trigger mechanism.** When using RADV, captures can also be triggered directly without RDP, using Mesa's trace infrastructure:

```bash
# Capture frame 50 automatically:
MESA_VK_TRACE_FRAME=50 ./vulkan-app

# Trigger interactively with F1 key (X11 only):
MESA_VK_TRACE_TRIGGER=./trigger_file ./vulkan-app
# then: touch ./trigger_file
```

The older environment variable `RADV_THREAD_TRACE_TRIGGER` is superseded by `MESA_VK_TRACE_TRIGGER` in current Mesa (25.x+). The output is a `.rgp` file openable in the RGP GUI.

### 3.4 AMD Vulkan Extensions for Profiling

Two vendor extensions enrich RGP traces with application-supplied structure:

**`VK_AMD_buffer_marker`**: Insert 32-bit marker values into the command stream, visible in the RGP timeline as labeled events:

```c
// Write a 32-bit marker at the pipeline stage where the previous
// work is guaranteed complete — useful for pinning GPU crash locations
vkCmdWriteBufferMarkerAMD(
    cmd,
    VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT,  // or BOTTOM_OF_PIPE
    markerBuffer,
    offsetInBuffer,
    0xDEADBEEFu);                        // user-defined marker value
```

**`VK_AMD_shader_info`**: Query post-compilation shader statistics — VGPR/SGPR register usage, scratch memory, LDS bytes, and wavefront size:

```c
VkShaderStatisticsInfoAMD stats;
size_t dataSize = sizeof(stats);
vkGetShaderInfoAMD(
    device, pipeline,
    VK_SHADER_STAGE_FRAGMENT_BIT,
    VK_SHADER_INFO_TYPE_STATISTICS_AMD,
    &dataSize, &stats);

printf("VGPRs used: %u (physical: %u)\n",
    stats.resourceUsage.numUsedVgprs,
    stats.numPhysicalVgprs);
```

High VGPR usage (`numUsedVgprs`) is the single most important predictor of low wavefront occupancy on AMD. On RDNA 3, 256 VGPRs per CU can hold either 8 wavefronts × 32 VGPRs each or 1 wavefront × 256 VGPRs — a shader using 256 VGPRs will run at 1/8th theoretical occupancy.

### 3.5 Supporting AMD Tools

**`umr` (Userspace Micro Register Debugger)** ([GitLab](https://gitlab.freedesktop.org/tomstdenis/umr)) provides low-level register inspection and shader disassembly. Useful for verifying hardware state without a full RGP capture:

```bash
# Disassemble shaders currently running on gfx10 (RDNA 1)
umr -s gfx10

# Read a specific MMIO register
umr -r 0x2010  # example register address

# Dump active shader programs
umr --wave
```

**`amdgpu_top`** ([GitHub](https://github.com/Umio-Yasuno/amdgpu_top)) is a Rust TUI tool for continuous AMDGPU monitoring, reading from performance counters (GRBM, GRBM2 blocks), `fdinfo`, and GPU metrics firmware tables. It provides per-process GPU utilization, VRAM usage, engine frequency, and temperature — a fast first-look tool before reaching for RGP:

```bash
# TUI mode (default): refreshes every 1 second
amdgpu_top

# SMI-style one-shot output
amdgpu_top --smi

# JSON output for scripting
amdgpu_top --json
```

**`radeontop`** ([GitHub](https://github.com/clbr/radeontop)) is an older tool that reads GRBM block counters via `/dev/mem` or debugfs; `amdgpu_top` is preferred for RDNA hardware.

---

## 4. Intel GPU Profiling Tools

### 4.1 intel_gpu_top

`intel_gpu_top` is part of [igt-gpu-tools](https://gitlab.freedesktop.org/drm/igt-gpu-tools) and provides a terminal-based, `top`-like GPU utilization view driven by the i915 PMU. It is the standard first-stop tool for Intel GPU profiling on Linux:

```bash
# Monitor i915 GPU engines
intel_gpu_top

# Specify device explicitly
intel_gpu_top -d i915:/dev/dri/card0

# JSON output mode for scripting / piping to other tools
intel_gpu_top -J

# Output to CSV file
intel_gpu_top -o /tmp/gpu.csv
```

`intel_gpu_top` displays render engine busy %, blitter busy %, video decode/encode engine busy %, current GPU frequency (actual vs. requested), memory bandwidth (when i915 Uncore IMC PMU events are available), and interrupt rate.

**Important limitation:** `intel_gpu_top` reads i915 PMU events and **does not support the Xe kernel driver**. For systems using the Xe driver (Meteor Lake, Battlemage/Arc B-series and later), the `gputop` tool ([freedesktop.org](https://github.com/rib/gputop)) or Mesa's Perfetto integration should be used instead. The Xe driver exposes PMU events under `/sys/bus/event_source/devices/xe/` rather than `i915/`.

### 4.2 INTEL_MEASURE

`INTEL_MEASURE` is Mesa's built-in GPU timestamp collection infrastructure, implemented for both the ANV (Vulkan) and Iris (OpenGL) Intel drivers. It inserts `VkQueryPool` GPU timestamp queries at configurable boundaries and writes results to a CSV file — no external tool or vendor daemon required:

```bash
# Collect per-draw GPU timestamps (default boundary)
INTEL_MEASURE=draw ./vulkan-app

# Collect per-batch timestamps (coarser, lower overhead)
INTEL_MEASURE=batch ./vulkan-app

# Collect per-render-pass timestamps
INTEL_MEASURE=rt ./vulkan-app

# Collect per-frame timestamps
INTEL_MEASURE=frame ./vulkan-app

# Write to a specific file
INTEL_MEASURE=batch,file=/tmp/intel_measure.csv ./vulkan-app

# Start capturing at frame 10, collect 20 frames
INTEL_MEASURE=batch,start=10,count=20 ./vulkan-app

# Collect pipeline statistics queries alongside timestamps
INTEL_MEASURE=pipeline_stats ./vulkan-app
```

The CSV output contains columns: `frame`, `event`, `cpu_start_ns`, `gpu_start_ns`, `gpu_end_ns`, `duration_ns`. Because GPU timestamps are captured in hardware, `INTEL_MEASURE` can identify idle gaps between draw calls and render passes that cannot be seen in CPU-side instrumentation.

Source: [Mesa environment variables documentation](https://docs.mesa3d.org/envvars.html).

### 4.3 i915 OA and VK_INTEL_performance_query

Intel's i915 OA (Observation Architecture) exposes hardware performance counter metric sets — groups of correlated counters that the GPU can sample simultaneously into a ring buffer. Metric sets include `RenderBasic`, `L3_1`, `ComputeBasic`, `MemoryReads`, and others. OA access requires either root or `sysctl dev.i915.perf_stream_paranoid=0`:

```bash
sudo sysctl dev.i915.perf_stream_paranoid=0

# Sample OA counters via perf
perf stat -e i915/rcs-busy/ -e i915/vcs-busy/ -e i915/bcs-busy/ \
    -- ./vulkan-app
```

`VK_INTEL_performance_query` is a vendor Vulkan extension that exposed OA metric sets through the Vulkan API. It has been superseded in practice by `VK_KHR_performance_query` (Section 2.3), which is the preferred cross-vendor interface now supported by ANV.

### 4.4 Intel GPA and VTune

**Intel Graphics Performance Analyzers (GPA)** ([intel.com](https://www.intel.com/content/www/us/en/developer/tools/graphics-performance-analyzers/overview.html)) provides a frame analyzer GUI and the Graphics Monitor background service. On Linux, GPA supports Vulkan frame capture via an interposing layer; the frame analyzer runs as a standalone GUI application.

**Intel oneAPI VTune Profiler** provides `gpu-hotspots` analysis for OpenCL, SYCL, and Vulkan compute workloads:

```bash
# Collect GPU hotspots (kernel execution time)
vtune --collect gpu-hotspots -- ./compute-app

# Open results in VTune GUI
vtune-gui r000gh/
```

VTune's GPU analysis reports OpenCL/Vulkan kernel execution time, EU array utilization, L3 cache hit rates, and DRAM bandwidth — at the granularity of GPU kernels (compute dispatches or render passes), not individual draw calls.

**`INTEL_DEBUG=perf`** is a Mesa driver debug flag that enables verbose logging of performance-relevant driver decisions:

```bash
INTEL_DEBUG=perf ./vulkan-app 2>&1 | grep -i stall
```

---

## 5. NVIDIA Nsight and perf on Linux

### 5.1 Nsight Systems

[Nsight Systems](https://developer.nvidia.com/nsight-systems) is NVIDIA's system-level profiler covering CUDA, OpenGL, and Vulkan on Linux. It captures CPU + GPU timelines together in a single trace, making it the correct first tool for CPU/GPU interaction analysis:

```bash
# Capture a Vulkan/OpenGL application trace
nsys profile --trace=opengl,vulkan,cuda \
    --output=./report \
    ./vulkan-app

# Generate text summary statistics from a capture file
nsys stats report.nsys-rep

# Launch the GUI analysis application
nsys-ui report.nsys-rep &
```

The `.nsys-rep` file is internally a SQLite database, enabling programmatic querying:

```bash
sqlite3 report.nsys-rep \
    "SELECT text, start, end FROM ANALYSIS_DETAILS LIMIT 20;"
```

Nsight Systems 2025.x visualizes Vulkan API calls on the CPU timeline, command buffer submission boundaries, GPU workload intervals (with debug label names from `VK_EXT_debug_utils`), and NVTX user markers alongside CUDA/OpenGL streams.

### 5.2 Nsight Graphics

[Nsight Graphics](https://developer.nvidia.com/nsight-graphics) is NVIDIA's frame-level GPU profiler, analogous to RGP for AMD. It captures per-draw-call metrics including SM occupancy, warp execution efficiency, L2 cache statistics, and memory transaction counts:

```bash
# Frame capture via Nsight Graphics UI, or command-line headless:
ngfx-cli --exe ./vulkan-app --capture-frame 5 \
    --output ./frame_capture
```

Linux limitations: Nsight Graphics requires the proprietary NVIDIA driver (`nvidia`) and does not work with Nouveau. The frame capture mechanism uses NVIDIA's proprietary interceptor layers.

### 5.3 NVTX User Markers

NVTX (NVIDIA Tools Extension, [docs](https://docs.nvidia.com/nvtx/index.html)) is a C API for annotating GPU workloads with named ranges and instantaneous markers. NVTX annotations appear in both Nsight Systems and Nsight Graphics, making it possible to correlate GPU work with application-level concepts:

```c
#include <nvtx3/nvToolsExt.h>

// Push a named range — nested ranges are supported
nvtxRangePushA("Shadow Map Pass");
vkCmdDraw(cmd, ...);   // shadow geometry
nvtxRangePop();

// Instantaneous marker at a point in time
nvtxMarkA("Frame boundary");

// With color and payload:
nvtxEventAttributes_t attrib = {
    .version = NVTX_VERSION,
    .size = NVTX_EVENT_ATTRIB_STRUCT_SIZE,
    .colorType = NVTX_COLOR_ARGB,
    .color = 0xFF0000FFu,  // blue
    .messageType = NVTX_MESSAGE_TYPE_ASCII,
    .message = { .ascii = "Geometry Pass" },
};
nvtxRangePushEx(&attrib);
```

Link against `libnvToolsExt.so` or the header-only `nvtx3` version. At zero overhead when no profiler is attached, NVTX annotations have essentially no cost in production builds.

### 5.4 VK_NV_device_diagnostic_checkpoints

`VK_NV_device_diagnostic_checkpoints` allows inserting per-draw checkpoint markers into a command buffer that survive GPU hangs. When a TDR (timeout detection and recovery) occurs, the last completed checkpoint can be recovered, pinpointing the crashing draw:

```c
// Insert checkpoint before a suspicious draw
vkCmdSetCheckpointNV(cmd, (void*)"draw_147_terrain_shadow");
vkCmdDraw(cmd, ...);

// After a GPU hang, query completed checkpoints:
uint32_t checkpointCount = 0;
vkGetQueueCheckpointDataNV(queue, &checkpointCount, NULL);
VkCheckpointDataNV checkpoints[MAX_CHECKPOINTS];
vkGetQueueCheckpointDataNV(queue, &checkpointCount, checkpoints);
printf("Last checkpoint: %s\n",
    (const char *)checkpoints[checkpointCount - 1].pCheckpointMarker);
```

### 5.5 NVML and nvidia-smi

NVML (NVIDIA Management Library) provides programmatic access to GPU utilization, memory usage, temperature, power draw, and clock frequencies:

```c
#include <nvml.h>
nvmlInit();
nvmlDevice_t device;
nvmlDeviceGetHandleByIndex(0, &device);

nvmlMemory_t memInfo;
nvmlDeviceGetMemoryInfo(device, &memInfo);
printf("VRAM used: %zu MiB / %zu MiB\n",
    memInfo.used >> 20, memInfo.total >> 20);

nvmlUtilization_t util;
nvmlDeviceGetUtilizationRates(device, &util);
printf("GPU: %u%%  Memory: %u%%\n", util.gpu, util.memory);
nvmlShutdown();
```

For terminal-based streaming monitoring:

```bash
# Stream GPU metrics every 100ms: power, utilization, clocks,
# VRAM, encoder/decoder activity, temperature
nvidia-smi dmon -s pucvmet -d 0.1

# Per-process GPU memory
nvidia-smi --query-compute-apps=pid,used_gpu_memory \
    --format=csv --loop=1
```

---

## 6. Mesa Built-in Instrumentation

### 6.1 GALLIUM_HUD

`GALLIUM_HUD` is Mesa's built-in heads-up display (HUD) overlay, available for all Gallium3D drivers (RadeonSI, Nouveau NVC0, LLVMpipe, softpipe, etc.) and rendered directly into the swapchain surface. It requires no external tool or privileged access:

```bash
# Show FPS, CPU load, and GPU load overlay
GALLIUM_HUD=fps,cpu,gpu-load ./opengl-app

# Multiple panels, semicolon-separated
GALLIUM_HUD="fps;draw-calls;primitives-generated;VRAM-usage" ./app

# Update every 0.25 seconds (default: 0.5)
GALLIUM_HUD_PERIOD=0.25 GALLIUM_HUD=fps ./app

# Dump CSV data files alongside the display
GALLIUM_HUD_DUMP_DIR=/tmp/hud_data/ GALLIUM_HUD=fps,gpu-load ./app

# Toggle visibility with SIGUSR1 (start hidden)
GALLIUM_HUD_VISIBLE=false GALLIUM_HUD_TOGGLE_SIGNAL=10 \
    GALLIUM_HUD=fps ./app
# then: kill -10 $(pidof app)
```

Available metric strings (query with `GALLIUM_HUD=help`):

| Metric | Description |
|--------|-------------|
| `fps` | Frames per second |
| `cpu` or `cpu0`–`cpuN` | CPU utilization per-core |
| `gpu-load` | GPU command processor busy % (via driver query) |
| `draw-calls` | Draw calls per frame |
| `primitives-generated` | Primitives from geometry/vertex stage |
| `pixels-rendered` | Pixels reaching the fragment stage |
| `buffer-wait-time` | CPU stall waiting for GPU buffer |
| `VRAM-usage` | Video RAM occupied (MB) |
| `GTT-usage` | Graphics Translation Table memory (MB) |
| `screen-shots` | Screenshot frame counter |

`GALLIUM_HUD_DUMP_DIR` writes per-metric CSV files in the form `fps.tsv`, `draw-calls.tsv`, etc., enabling post-session analysis in spreadsheet tools or scripts.

To force the software renderer for comparison:

```bash
MESA_LOADER_DRIVER_OVERRIDE=llvmpipe GALLIUM_HUD=fps,draw-calls ./app
```

Source: [Mesa Gallium HUD documentation](https://gallium.readthedocs.io/en/latest/envvars.html).

### 6.2 MESA_DEBUG

`MESA_DEBUG` accepts a comma-separated list of debug modes. The `context` flag is the most performance-relevant: it creates an OpenGL debug context and routes performance warnings — cache misses, synchronous texture uploads, deprecated API usage — to stderr or `MESA_LOG_FILE`:

```bash
# Enable performance warnings via debug context
MESA_DEBUG=context ./opengl-app 2>&1 | grep -i perf

# Flush after every draw (dramatically slows rendering,
# but reveals per-draw GPU timing via CPU-side blocking):
MESA_DEBUG=flush ./opengl-app

# Combine: flush + debug context
MESA_DEBUG=flush,context ./opengl-app 2>/tmp/mesa_debug.log
```

### 6.3 RADV Driver Profiling Variables

RADV (the Mesa AMD Vulkan driver) exposes several profiling-specific environment variables:

```bash
# Log shader compilation events with timing and VGPR/SGPR counts
RADV_DEBUG=shaderstats ./vulkan-app

# Force all shaders to recompile (disable the pipeline cache)
MESA_SHADER_CACHE_DISABLE=1 ./vulkan-app

# Dump NIR IR at each optimization pass (very verbose)
RADV_DEBUG=nir ./vulkan-app

# Log all Vulkan calls with their parameters (API trace)
RADV_DEBUG=startup ./vulkan-app

# Synchronize the GPU after every draw/dispatch (enables
# per-draw GPU timing with CPU-side wall clock)
RADV_DEBUG=syncshaders ./vulkan-app

# Dump PSO cache statistics (hit/miss rates)
RADV_DEBUG=psocachestats ./vulkan-app
```

Source: [Mesa RADV environment variables](https://docs.mesa3d.org/envvars.html).

### 6.4 Shader Compiler Profiling

Mesa includes `shader_db`, a tool for measuring shader compile time across a collection of NIR shader files. It is used in Mesa CI for regression detection:

```bash
# Measure compile time for all shaders in a directory
shader_db -p radv /path/to/shader-db/shaders/ 2>&1 | \
    sort -k3 -n -r | head -20
```

`RADV_DEBUG=compile` logs per-shader compile events to stderr, while `MESA_SHADER_CACHE_DISABLE=1` forces cold compilation on every run — together they benchmark the hot-path shader compiler without warm-cache interference.

---

## 7. Vulkan Timestamp and Statistics Queries

### 7.1 Timestamp Queries

`VK_QUERY_TYPE_TIMESTAMP` is the most portable Vulkan GPU timing mechanism. Unlike `INTEL_MEASURE` (Mesa-specific) or NVTX (NVIDIA-specific), timestamp queries work on every Vulkan implementation:

```c
// Create timestamp query pool for begin + end of a render pass
VkQueryPoolCreateInfo poolCI = {
    .sType = VK_STRUCTURE_TYPE_QUERY_POOL_CREATE_INFO,
    .queryType = VK_QUERY_TYPE_TIMESTAMP,
    .queryCount = 2,
};
VkQueryPool tsPool;
vkCreateQueryPool(device, &poolCI, NULL, &tsPool);

// In the command buffer:
vkCmdResetQueryPool(cmd, tsPool, 0, 2);

// Write timestamp at start of render pass (after vkCmdBeginRenderPass)
vkCmdWriteTimestamp2(cmd,
    VK_PIPELINE_STAGE_2_TOP_OF_PIPE_BIT,
    tsPool, 0);

// ... draw calls ...

// Write timestamp after all fragment work completes
vkCmdWriteTimestamp2(cmd,
    VK_PIPELINE_STAGE_2_BOTTOM_OF_PIPE_BIT,
    tsPool, 1);
```

`vkCmdWriteTimestamp2` (from `VK_KHR_synchronization2`, core in Vulkan 1.3) is preferred over the older `vkCmdWriteTimestamp` because it uses `VkPipelineStageFlags2`, which has finer-grained pipeline stage names including `VK_PIPELINE_STAGE_2_FRAGMENT_SHADER_BIT` and `VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT`.

**Reading results and converting to nanoseconds:**

```c
uint64_t timestamps[2];
vkGetQueryPoolResults(device, tsPool, 0, 2,
    sizeof(timestamps), timestamps, sizeof(uint64_t),
    VK_QUERY_RESULT_64_BIT | VK_QUERY_RESULT_WAIT_BIT);

// Convert raw ticks to nanoseconds
VkPhysicalDeviceProperties props;
vkGetPhysicalDeviceProperties(physicalDevice, &props);

double elapsedNs = (double)(timestamps[1] - timestamps[0])
                 * props.limits.timestampPeriod;
printf("Render pass took %.3f ms\n", elapsedNs * 1e-6);
```

`VkPhysicalDeviceLimits.timestampPeriod` is the number of nanoseconds per GPU clock tick. On AMD RDNA hardware it is typically 1.0 ns (1 GHz counter); on Intel Xe it can be fractional (e.g., 0.5 ns for a 2 GHz counter). `timestampComputeAndGraphics` in `VkPhysicalDeviceLimits` indicates whether timestamps are comparable across both graphics and compute queues.

### 7.2 Calibrated Timestamps

`VK_KHR_calibrated_timestamps` ([Vulkan spec](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_calibrated_timestamps.html)) — note: originally `VK_EXT_calibrated_timestamps`, promoted to KHR — allows atomically reading GPU timestamp and CPU `CLOCK_MONOTONIC` together, enabling correlation of GPU events with CPU-side profiling data (perf, flamegraphs, etc.):

```c
VkCalibratedTimestampInfoKHR domains[2] = {
    { .sType = VK_STRUCTURE_TYPE_CALIBRATED_TIMESTAMP_INFO_KHR,
      .timeDomain = VK_TIME_DOMAIN_DEVICE_KHR },
    { .sType = VK_STRUCTURE_TYPE_CALIBRATED_TIMESTAMP_INFO_KHR,
      .timeDomain = VK_TIME_DOMAIN_CLOCK_MONOTONIC_KHR },
};
uint64_t timestamps[2];
uint64_t maxDeviation;

vkGetCalibratedTimestampsKHR(device, 2, domains,
    timestamps, &maxDeviation);

uint64_t gpuTs  = timestamps[0];  // GPU device timestamp
uint64_t cpuNs  = timestamps[1];  // CLOCK_MONOTONIC nanoseconds
// maxDeviation: upper bound on measurement error
```

With calibrated timestamps, a GPU render pass's start/end GPU ticks can be converted to wall-clock nanoseconds and overlaid on a `perf` trace or a Perfetto trace.

### 7.3 Pipeline Statistics Queries

`VK_QUERY_TYPE_PIPELINE_STATISTICS` collects hardware counters — input vertex count, primitive count, invocation counts per shader stage — with zero GPU overhead (they are computed from the geometry pipeline, not added counter hardware):

```c
VkQueryPoolCreateInfo statsPoolCI = {
    .sType = VK_STRUCTURE_TYPE_QUERY_POOL_CREATE_INFO,
    .queryType = VK_QUERY_TYPE_PIPELINE_STATISTICS,
    .queryCount = 1,
    .pipelineStatistics =
        VK_QUERY_PIPELINE_STATISTIC_INPUT_ASSEMBLY_VERTICES_BIT |
        VK_QUERY_PIPELINE_STATISTIC_INPUT_ASSEMBLY_PRIMITIVES_BIT |
        VK_QUERY_PIPELINE_STATISTIC_VERTEX_SHADER_INVOCATIONS_BIT |
        VK_QUERY_PIPELINE_STATISTIC_FRAGMENT_SHADER_INVOCATIONS_BIT |
        VK_QUERY_PIPELINE_STATISTIC_COMPUTE_SHADER_INVOCATIONS_BIT,
};
VkQueryPool statsPool;
vkCreateQueryPool(device, &statsPoolCI, NULL, &statsPool);

// In command buffer:
vkCmdBeginQuery(cmd, statsPool, 0, 0);
// ... draw calls ...
vkCmdEndQuery(cmd, statsPool, 0);
```

Interpreting results:

```c
struct PipelineStats {
    uint64_t verticesIn;
    uint64_t primitivesIn;
    uint64_t vsInvocations;
    uint64_t fsInvocations;
    uint64_t csInvocations;
} stats;

vkGetQueryPoolResults(device, statsPool, 0, 1, sizeof(stats),
    &stats, sizeof(stats),
    VK_QUERY_RESULT_64_BIT | VK_QUERY_RESULT_WAIT_BIT);

// High fsInvocations relative to primitivesIn → overdraw
double pixelsPerPrimitive =
    (double)stats.fsInvocations / (double)stats.primitivesIn;
printf("Average overdraw factor: %.1fx\n", pixelsPerPrimitive);
```

If `fsInvocations / primitivesIn` greatly exceeds the average triangle screen size in pixels, severe overdraw is present. If `vsInvocations` is high relative to `primitivesIn`, post-transform vertex reuse is poor (index buffer cache misses).

---

## 8. System-Level GPU Profiling

### 8.1 drm_fdinfo: Per-Process GPU Utilization

The Linux kernel DRM subsystem exposes per-process GPU utilization through `/proc/<pid>/fdinfo/<fd>` for each open DRM file descriptor ([kernel documentation](https://docs.kernel.org/gpu/drm-usage-stats.html)). This is the mechanism used by `MangoHud`, `amdgpu_top`, `nvtop`, and `all-smi` for per-process GPU metrics without root privileges.

Key `fdinfo` keys:

| Key | Type | Meaning |
|-----|------|---------|
| `drm-driver` | string | Driver name (`amdgpu`, `i915`, `xe`, `nouveau`) |
| `drm-pdev` | string | PCI slot (`0000:0a:00.0`) |
| `drm-client-id` | uint | Unique per-fd identifier |
| `drm-engine-render` | ns | Cumulative nanoseconds on render engine |
| `drm-engine-compute` | ns | Cumulative nanoseconds on compute engine |
| `drm-engine-copy` | ns | Cumulative nanoseconds on copy engine |
| `drm-cycles-render` | uint | Busy GPU cycles (render) |
| `drm-total-cycles-render` | uint | Total GPU cycles (for utilization %) |
| `drm-memory-vram` | bytes | VRAM allocated |
| `drm-memory-gtt` | bytes | System (GTT) memory allocated |
| `drm-memory-cpu` | bytes | CPU-visible memory allocated |

Computing utilization from two `fdinfo` snapshots:

```python
import time, os, re

def read_fdinfo(pid, fd):
    path = f"/proc/{pid}/fdinfo/{fd}"
    data = {}
    with open(path) as f:
        for line in f:
            m = re.match(r'(drm-\S+):\s+(\d+)', line)
            if m:
                data[m.group(1)] = int(m.group(2))
    return data

s1 = read_fdinfo(pid, fd_num)
time.sleep(1.0)
s2 = read_fdinfo(pid, fd_num)

render_busy = s2['drm-engine-render'] - s1['drm-engine-render']
wall_ns = 1_000_000_000  # 1 second in nanoseconds
util_pct = render_busy / wall_ns * 100
print(f"Render engine utilization: {util_pct:.1f}%")
```

### 8.2 perf GPU Events

```bash
# List all GPU-related perf events
perf list | grep -i 'gpu\|amdgpu\|i915\|xe'

# AMD: count texture-addresser busy cycles during workload
perf stat -e amdgpu/ta_busy_cnt/ -- ./vulkan-app

# Intel i915: measure render, video, blitter engine occupancy
perf stat \
    -e i915/rcs-busy/ \
    -e i915/vcs-busy/ \
    -e i915/bcs-busy/ \
    -- ./vulkan-app

# Record with CPU callstack for correlation
perf record -e amdgpu/ta_busy_cnt/ -g -- ./vulkan-app
perf report --stdio

# Intel Xe events (driver: xe, not i915)
perf stat -e xe/rcs-busy/ -- ./vulkan-app
```

`perf record` with GPU PMU events records the CPU callstack at each hardware counter overflow — enabling correlation between application code paths and GPU counter activity, even though the counter is system-wide.

### 8.3 gputop for Intel OA

`gputop` ([GitHub](https://github.com/rib/gputop)) is Intel-specific and accesses the i915 OA (Observation Architecture) PMU, which exposes correlated counter metric sets that cannot be accessed through standard `perf stat`:

```bash
# Web-based UI: start gputop server, connect browser to localhost:7890
gputop --remote

# List available OA metric sets
gputop --list

# Sample RenderBasic metric set for 10 seconds
gputop --metric=RenderBasic --duration=10 --output=metrics.json
```

OA metric sets for Intel Xe are accessible through separate tooling; `gputop` in its original form targets i915.

### 8.4 Continuous Monitoring Tools

```bash
# radeontop: classic AMD GPU monitor (GRBM block counters via /dev/mem or DRM)
radeontop -d /dev/dri/card0

# nvtop: interactive TUI for NVIDIA, AMD, Intel, and other GPUs via fdinfo
nvtop

# MangoHud: Vulkan/OpenGL overlay (uses fdinfo + driver queries)
MANGOHUD=1 ./vulkan-app
# See Ch29 for MangoHud's complete metric set and configuration

# Correlate GPU load with power consumption
powertop                              # CPU power; GPU power via RAPL
nvidia-smi -q -d POWER                # NVIDIA power draw
cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_average  # AMD/Intel
```

---

## 9. Flame Graphs and Profiling Workflows

### 9.1 Perfetto on Linux

[Perfetto](https://perfetto.dev/) is Google's system tracing framework, originally developed for Android but now supported on Linux. Mesa has experimental Perfetto integration for driver-level GPU counter and render stage traces:

```bash
# Capture a 10-second system trace with GPU counters (AMD/Intel)
tracebox -- perfetto \
    -o /tmp/trace.pb \
    -t 10s \
    -c - <<EOF
buffers { size_kb: 65536 }
data_sources { config { name: "gpu.counters.amdgpu" } }
data_sources { config { name: "gpu.renderstages.amdgpu" } }
data_sources { config { name: "linux.process_stats" } }
EOF

# Open in Perfetto UI
# Upload /tmp/trace.pb to https://ui.perfetto.dev/
```

Mesa driver datasource names:

| Driver | Counter source | Render stage source |
|--------|----------------|---------------------|
| RadeonSI / RADV | `gpu.counters.amdgpu` | `gpu.renderstages.amdgpu` |
| ANV / Iris (Intel) | `gpu.counters.i915` | `gpu.renderstages.intel` |
| Freedreno / Turnip | `gpu.counters.msm` | `gpu.renderstages.msm` |
| Panfrost / PanVK | `gpu.counters.panfrost` | `gpu.renderstages.panfrost` |

The **pps-producer** daemon collects system-wide GPU counters (requires root for AMD and Intel OA):

```bash
sudo pps-producer --backend=i915 &
# then capture a perfetto trace as above
```

Source: [Mesa Perfetto documentation](https://docs.mesa3d.org/perfetto.html).

### 9.2 CPU Flamegraph + GPU Timestamp Correlation

Combining CPU flamegraphs with GPU timestamps produces the most informative picture of a frame's cost:

```bash
# 1. Capture CPU flamegraph while app runs
perf record -F 999 -g --call-graph dwarf -- ./vulkan-app &
APP_PID=$!

# 2. Simultaneously collect GPU load via amdgpu_top / intel_gpu_top

# 3. Let the app run for 10 seconds, then stop
sleep 10; kill $APP_PID; wait $APP_PID

# 4. Generate flamegraph
perf script | stackcollapse-perf.pl | flamegraph.pl > cpu_flame.svg
```

GPU timestamps (from `VkQueryPool`) recorded in the application show exactly which render passes consumed the GPU time. The CPU flamegraph shows the application-level code that generated those workloads. Together they answer: "The shadow pass takes 8ms GPU time, and the CPU-side code building the shadow command buffer calls `VkCmdDraw` 847 times from `scene::draw_shadow_casters` — that function needs batching."

### 9.3 Systematic Optimization Workflow

A structured GPU profiling methodology for Linux:

**Step 1: Classify the bottleneck.** Use `GALLIUM_HUD=fps,gpu-load` or `MangoHud` to determine whether the frame is CPU-bound (CPU > ~16ms at 60 Hz, GPU < 95% utilized) or GPU-bound (GPU at 100%, CPU stalling on presents).

**Step 2: If GPU-bound, find the expensive pass.** Insert `vkCmdWriteTimestamp2` queries at render pass boundaries, or enable `INTEL_MEASURE=rt` / `RADV_DEBUG=syncshaders`. Identify the render pass consuming the most GPU time.

**Step 3: Characterize the bottleneck type.** Use pipeline statistics (`VK_QUERY_TYPE_PIPELINE_STATISTICS`) to determine:
- High `fsInvocations / primitivesIn` → overdraw → add depth pre-pass or better culling
- High `vsInvocations` with low reuse → vertex fetch bound → optimize index ordering
- Low occupancy (from RGP or `VK_AMD_shader_info`) → register pressure → reduce VGPR usage

**Step 4: Drill into shader performance.** Use RGP (AMD) or Nsight Graphics (NVIDIA) for wavefront/warp-level analysis. Check:
- SM or CU occupancy (low occupancy → VGPR/LDS pressure or launch latency)
- ALU efficiency (idle ALU cycles → long-latency memory dependencies)
- Memory transaction patterns (scattered VRAM accesses → stride optimization or texture sampling)

**Step 5: Bandwidth analysis.** Use `VK_KHR_performance_query` bandwidth counters to measure L2 and VRAM bytes/frame. High VRAM bandwidth relative to vertex count → uncoalesced vertex fetches or overly wide vertex attributes.

**Step 6: Validate the fix.** Re-run with `GALLIUM_HUD=frame-time` to verify the improvement. Use `RADV_DEBUG=psocachestats` to confirm pipeline state objects are being cached rather than recompiled.

**Step 7: Identify stutter.** Frame time spikes (visible in `GALLIUM_HUD=frame-time` CSV dumps or Perfetto traces) are often caused by:
- Synchronous GPU-CPU buffer readbacks (map of in-flight buffer)
- Shader compilation stalls (`MESA_SHADER_CACHE_DISABLE=1` triggers worst case)
- Descriptor set pool exhaustion or fence/semaphore pool thrashing

Use `MESA_DEBUG=context` to surface OpenGL synchronization warnings; for Vulkan, `RADV_DEBUG=syncshaders` forces a CPU-GPU sync after every dispatch, isolating which dispatch is stalling.

---

## Roadmap

### Near-term (6–12 months)

- **AMD RDNA 4 SQTT support in RADV.** AMD's Radeon Developer Tool Suite 2026 releases (RGP 2.7) have expanded hardware support to RDNA 4 (Radeon RX 9000 series). Corresponding RADV-side SQTT capture support for RDNA 4 wavefront tracing is expected to follow in Mesa as RADV becomes the default Vulkan driver on Linux following AMDVLK's discontinuation. [Source](https://gpuopen.com/learn/radeon-developer-tool-suite-update-rgp-2-6/)
- **RGP shader source-code viewing and instruction-level divergence metrics.** RGP 2.7 introduced shader source code viewing and instruction-level divergence metrics for identifying warp/wavefront divergence hotspots. These features surface inside Linux `.rgp` captures when RADV SQTT is used. [Source](https://gpuopen.com/learn/radeon-developer-tool-suite-update-rgp-2-6/)
- **Expanded `drm_fdinfo` counter coverage across drivers.** Panfrost, Nouveau, and other drivers have active patchsets to extend `drm-cycles-*` and `drm-engine-*` fdinfo keys to cover more engine types (video decode, media engines), making MangoHud and `drm_info` more informative across a broader GPU landscape. [Source](https://lkml.iu.edu/hypermail/linux/kernel/2403.0/03640.html)
- **eBPF-backed GPU perf event sampling.** The Linux `perf` subsystem's eBPF integration has enabled `perf stat` to read PMU counters into BPF maps; near-term work focuses on extending this path to GPU PMU drivers (amdgpu, i915/Xe) so that eBPF programs can co-locate GPU counter sampling with CPU stack traces without switching tools. Note: needs verification for specific upstream commit status.
- **Mesa Perfetto `gpu_memory` data source stabilization.** Mesa's Perfetto integration currently covers CPU-side Mesa state tracing; work is underway to stabilize the `gpu_memory` Perfetto data source (ftrace-backed) so that GPU VRAM pressure shows up natively in Perfetto timelines alongside CPU traces. [Source](https://docs.mesa3d.org/perfetto.html)

### Medium-term (1–3 years)

- **Vulkan `VK_KHR_performance_query` broader driver coverage.** Intel's ANV and NVIDIA's NVK (the open-source Vulkan driver) have partial or in-progress `VK_KHR_performance_query` implementations. Full coverage across all Mesa-hosted drivers would enable cross-vendor benchmark harnesses to use a single Vulkan API rather than vendor-specific perf paths. Note: needs verification for specific NVK implementation status.
- **Kernel `perf` GPU counter unification.** Current GPU counter access is fragmented — AMD uses amdgpu's own ioctl path, Intel i915/Xe uses OA streams, NVIDIA uses proprietary SDK. A longer-term direction discussed in kernel mailing list threads is a unified `perf_event_open`-based GPU PMU interface so that all GPU counter collection is accessible through the same syscall as CPU profiling. Note: needs verification for specific RFC status.
- **Perfetto GPU timeline first-class support in trace_processor.** Perfetto's `trace_processor` currently handles GPU counter tracks from Android (via `GpuCounterEvent` proto) but Linux GPU profiling data from `drm_fdinfo` requires custom importers. Upstream work to add a standard Linux GPU track importer would make Linux GPU profiling data queryable with the same SQL-based `trace_processor` tooling used for Android. [Source](https://lkml.iu.edu/hypermail/linux/kernel/2403.0/03640.html)
- **Intel Xe driver OA stream stabilization.** The `i915` OA subsystem underpins Intel's `INTEL_MEASURE` and `VK_INTEL_performance_query` paths. The Xe kernel driver (i915's successor for Xe-generation hardware) is re-implementing the OA subsystem under a new driver model; stabilization of `xe_oa` will restore full counter access for Meteor Lake and Lunar Lake on the new driver. Note: needs verification for specific Xe OA upstream status.
- **Shader occupancy visualization in open-source tooling.** AMD RGP's wavefront occupancy view has no direct open-source equivalent; RADV and Mesa developers have discussed adding occupancy estimation to `VK_AMD_shader_info` output and to Mesa's internal shader statistics, so that the shader compiler can report register pressure and theoretical occupancy without proprietary tooling.

### Long-term

- **Standardized cross-vendor GPU profiling protocol.** The profiling ecosystem is fragmented by vendor: RGP files are AMD-specific, Nsight traces are NVIDIA-specific, and there is no open interchange format analogous to SPIR-V for GPU trace data. A long-term goal in the open-source profiling community is a vendor-neutral GPU trace format (potentially built on Perfetto's proto schema) that tools like RenderDoc, GPA, and Nsight could all emit.
- **AI-assisted performance diagnosis.** Both NVIDIA Nsight and AMD's tooling have begun integrating guided analysis (automated bottleneck identification). In the open-source Linux stack, similar guidance could emerge from LLM-backed tooling that interprets `VK_KHR_performance_query` counter dumps or Perfetto traces and suggests optimization strategies — though this remains speculative and raises questions about counter semantic accuracy.
- **GPU profiling integration in system-wide observability platforms.** As platforms like Grafana, OpenTelemetry, and Prometheus expand into GPU infrastructure monitoring (primarily for ML/HPC workloads), the same `drm_fdinfo`-based metrics exported for desktop GPU profiling will likely feed into standardized observability pipelines, blurring the line between frame-level profiling tools and data-center infrastructure monitoring.
- **Hardware-accelerated counter sampling.** Current GPU counter sampling interrupts the GPU pipeline or requires pass-multiplexing (as in `VK_KHR_performance_query`'s multi-pass requirement). Future GPU architectures may expose dedicated on-chip sampling hardware that can record counter snapshots between draw calls without disturbing execution, enabling always-on profiling with negligible overhead — analogous to ARM's SPE (Statistical Profiling Extension) for CPUs. Note: needs verification for specific hardware roadmap announcements.

---

## 10. Integrations

**Ch24 — Vulkan Internals.** `VK_KHR_performance_query` and `VK_QUERY_TYPE_TIMESTAMP` are both Vulkan core extension mechanisms described in Ch24's coverage of Vulkan query pools and extension promotion. The `timestampPeriod` value used to convert GPU ticks to nanoseconds is part of `VkPhysicalDeviceLimits`, detailed in Ch24.

**Ch18 — RADV (Mesa AMD Vulkan Driver).** RADV implements SQTT capture and emits `.rgp` files consumed by the Radeon GPU Profiler. The MESA_VK_TRACE_TRIGGER mechanism, RADV's `VK_AMD_buffer_marker` support, and shader statistics via `VK_AMD_shader_info` are implemented in RADV's command recording paths covered in Ch18.

**Ch19 — ANV (Intel Vulkan Driver).** `INTEL_MEASURE` is implemented in ANV's batch generation layer. The i915 OA integration and `VK_INTEL_performance_query` history are discussed in the context of ANV's query implementation in Ch19.

**Ch25 — GPU Compute (Vulkan/OpenCL).** Compute shader profiling — CS invocations via pipeline statistics, occupancy analysis via RGP or Nsight, and `RADV_DEBUG=syncshaders` for per-dispatch timing — applies equally to graphics and compute workloads. The interaction between compute queue timestamps and graphics queue timestamps (and whether they are comparable, per `timestampComputeAndGraphics`) is relevant to mixed workloads covered in Ch25.

**Ch29 — MangoHud.** MangoHud reads `drm_fdinfo` keys (`drm-engine-render`, `drm-memory-vram`, `drm-cycles-render`) to display per-process GPU metrics in its overlay. The `fdinfo` schema described in Section 8.1 is the kernel interface MangoHud relies on, documented in Ch29's coverage of MangoHud's data sources.

**Ch30 — Graphics Debugging.** GPU performance profiling and GPU debugging are complementary disciplines that share tools (RenderDoc, RGP, Nsight Graphics) and concepts (command buffer markers, validation layers). Ch30 covers the debugging perspective — detecting incorrect rendering — while this chapter focuses on the performance perspective. The same `VK_AMD_buffer_marker` extension used for crash pinpointing (debugging) is also used for render pass timing (profiling).

**Ch75 — Explicit GPU Synchronization.** Accurate GPU timestamp measurement requires understanding the synchronization model: `vkCmdWriteTimestamp2` with `VK_PIPELINE_STAGE_2_BOTTOM_OF_PIPE_BIT` measures when all prior work completes on the GPU, not when the CPU submitted the command. Timeline semaphores (Ch75) enable multi-queue timestamp correlation. Calibrated timestamps (`VK_KHR_calibrated_timestamps`) depend on the CPU/GPU clock domain relationships analyzed in Ch75.

**Ch109 — Mesa Testing and CI.** Mesa's performance regression tracking uses `shader_db` for compiler throughput benchmarks and Phoronix Test Suite for rendering throughput. The `RADV_DEBUG=compile` and `MESA_SHADER_CACHE_DISABLE=1` flags described in Section 6.4 are also used in Mesa CI to detect shader compiler regressions. Ch109 describes the automated benchmark infrastructure that surfaces performance regressions before they reach users.

**Ch125 — RenderDoc on Linux.** RenderDoc and RGP are complementary: RenderDoc captures API-level frame data (draw calls, resources, pipeline state) while RGP captures hardware-level wavefront traces. `AMDRenderDoc` is an integration layer that bridges the two: a `.rdc` file captured in RenderDoc can be re-submitted to the driver with SQTT enabled, and the resulting `.rgp` file opened in RGP — combining API-level visibility with hardware-level timing. Ch125 covers RenderDoc's Linux capture architecture and the technical mechanism of this bridge.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
