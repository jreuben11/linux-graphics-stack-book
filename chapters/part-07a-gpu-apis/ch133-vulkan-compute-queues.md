# Chapter 133: Vulkan Compute Queues and Task Graphs

## Scope

This chapter targets four audiences. **Vulkan application developers** seeking to move beyond single-queue rendering will learn how to discover, select, and synchronize multiple queue families. **Game engine developers** will find a concrete multi-queue frame structure and the rationale behind the "frame graph" / "task graph" abstraction. **GPU compute engineers** will see how Vulkan's queue model maps to real AMD, Intel, and NVIDIA hardware engines and how to measure overlap with GPU timestamps. **Graphics architects** designing multi-queue workloads will learn how timeline semaphores, synchronization2 barriers, and render graph compilers compose into a coherent system.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [GPU Hardware Queue Model](#2-gpu-hardware-queue-model)
3. [Queue Families and Discovery](#3-queue-families-and-discovery)
4. [Timeline Semaphores](#4-timeline-semaphores)
5. [Pipeline Barriers in Depth](#5-pipeline-barriers-in-depth)
6. [Async Compute: Overlapping Graphics and Compute](#6-async-compute-overlapping-graphics-and-compute)
7. [DMA Queue and Streaming](#7-dma-queue-and-streaming)
8. [Frame Graphs / Task Graphs](#8-frame-graphs--task-graphs)
9. [Practical Multi-Queue Frame Structure](#9-practical-multi-queue-frame-structure)
10. [Vulkan Video Queues](#10-vulkan-video-queues)
11. [Integrations](#11-integrations)

---

## 1. Introduction

OpenGL gives the application a single implicit command stream: every draw call and compute dispatch goes into one ring buffer and executes in the order it was recorded. Drivers may reorder under the hood, but the application sees a sequentially consistent GPU. Vulkan discards this fiction and exposes the GPU's actual hardware queue structure directly to the programmer.

Modern discrete GPUs contain several independent command processors. On AMD RDNA hardware there are one or more Compute/Graphics front-ends, multiple Asynchronous Compute Engines (ACEs) that drive compute work independently, and one or two System DMA (SDMA) engines for memory copies. On NVIDIA, a unified graphics/compute engine coexists with dedicated copy engines and dedicated video decode/encode engines. On Intel Xe2, separate render, compute, copy, and video queue types are exposed.

Vulkan maps these hardware engines to **queue families**: groups of queues that share the same capability flags. An application that uses only one queue family leaves performance on the table. A well-structured frame can run TAA compute and particle simulation on an ACE queue while the main GFX queue is processing the Z-prepass; simultaneously a DMA engine streams next-frame textures over PCIe. Getting all three to cooperate without data races or redundant stalls is what this chapter is about.

The tools are: `VkQueueFamilyProperties` for capability discovery, `VkDeviceQueueCreateInfo` for queue reservation, timeline semaphores (`VK_KHR_timeline_semaphore`, core in Vulkan 1.2) for cross-queue and CPU–GPU signalling, `vkCmdPipelineBarrier2` (Vulkan 1.3 / `VK_KHR_synchronization2`) for cache and execution ordering within a queue, and the **frame graph** pattern for coordinating dozens of render and compute passes without hand-inserting every barrier.

---

## 2. GPU Hardware Queue Model

### 2.1 AMD RDNA 3 and RDNA 4

AMD's command processor hierarchy has remained stable from GCN through RDNA 4. Each GPU die contains:

- **One GFX ring** (Graphics Command Processor, CP). This ring can submit both graphics and compute workloads to the Compute Unit (CU / WGP) array. It is exposed as the primary `VK_QUEUE_GRAPHICS_BIT | VK_QUEUE_COMPUTE_BIT | VK_QUEUE_TRANSFER_BIT` queue family.
- **Multiple Asynchronous Compute Engines (ACEs)**. Each ACE is an independent command processor that dispatches compute work without touching the GFX state. On RDNA 3 consumer cards (RX 7000-series) there are typically 2 ACEs visible to software, though the hardware may have more. Each ACE manages up to 8 independent queues. [Source](https://gpuopen.com/amd-gpu-architecture-programming-documentation/)
- **One or two SDMA engines**. The System DMA engines move data between host and device memory and between device memory regions. They have no shader dispatch capability. On RDNA 3 discrete GPUs two SDMA engines are present. [Source](https://docs.mesa3d.org/drivers/radv.html)

The Micro Engine Scheduler (MES) firmware arbitrates between the GFX ring and ACE queues, enabling true simultaneous execution on the CU array when their workloads do not contend for the same resources. [Source](https://gpuopen.com/amd-gpu-architecture-programming-documentation/)

### 2.2 Intel Xe2 (Battlemage / Lunar Lake)

Intel's Xe2 architecture, debuting in Lunar Lake (2024) and Arc B-series discrete GPUs, exposes distinct hardware engines:

- **Render Engine**: 3D pipeline, geometry, rasterisation.
- **Compute Engine**: GPGPU dispatch without 3D state.
- **Copy Engine**: Blitter for buffer and image copies.
- **Video Engine**: Hardware H.264, H.265, AV1 decode and encode.

Vulkan's ANV driver exposes these as queue families with appropriate capability flags. Intel's OneAPI L0 v2 adapter, introduced in 2025, restructures queue submission to minimise host overhead for each engine independently. [Source](https://www.intel.com/content/www/us/en/developer/articles/release-notes/oneapi-dpcpp/2025.html)

### 2.3 NVIDIA Ampere and Ada Lovelace

NVIDIA does not expose multiple graphics front-ends to Vulkan the way AMD does with ACEs. Instead:

- **Unified Graphics/Compute Engine**: A single engine handles all graphics and compute work. NVIDIA exposes two Vulkan queue types within the same queue family — a `DIRECT` queue (graphics + compute) and `ASYNC_COMPUTE` queues — that can overlap on the SM array when their register file footprints allow it. [Source](https://developer.nvidia.com/blog/advanced-api-performance-async-compute-and-overlap/)
- **Copy Engines**: Ampere has five CE (Copy Engine) units. These handle `cudaMemcpy` style transfers and are exposed to Vulkan as transfer queues. [Source](https://developer.nvidia.com/blog/nvidia-ampere-architecture-in-depth/)
- **Video Decode/Encode Engines**: Separate NVDEC and NVENC hardware engines handle video codec work.

### 2.4 The Overlap Opportunity

The key insight is that multiple engines run **physically simultaneously** when they do not contend for the same CU/SM resources or memory bandwidth:

```
Timeline:
  GFX queue:     [Z-prepass]──────[Shadow maps]──[Lighting]──────────────[Forward+]
  Async compute: [Particle sim]──[Culling]──[TAA / tonemapping]
  DMA:           [Upload tex LOD k+1]─────────────[Upload tex LOD k+2]
```

The GFX queue is vertex-bound during Z-prepass, leaving compute resources idle on AMD hardware. An ACE queue can fill those idle compute slots with particle simulation work, improving GPU utilisation without adding to the GFX queue's critical path.

Hardware preemption interacts with this model at the kernel level. AMD's `drm_sched` (covered in Ch102) may preempt long-running compute dispatches to serve the compositor's flip deadlines. Applications should keep compute dispatches short enough (< 1 ms) to remain responsive to preemption. [Source](https://www.kernel.org/doc/html/latest/gpu/drm-mm.html)

---

## 3. Queue Families and Discovery

### 3.1 Enumeration

```c
// Enumerate queue families
uint32_t queueFamilyCount = 0;
vkGetPhysicalDeviceQueueFamilyProperties(physicalDevice, &queueFamilyCount, NULL);

VkQueueFamilyProperties *props = malloc(queueFamilyCount * sizeof(*props));
vkGetPhysicalDeviceQueueFamilyProperties(physicalDevice, &queueFamilyCount, props);
```

[Source](https://docs.vulkan.org/guide/latest/queues.html)

Each `VkQueueFamilyProperties` entry carries:

| Field | Meaning |
|---|---|
| `queueFlags` | Bitfield of supported operations |
| `queueCount` | Max simultaneous queues from this family |
| `timestampValidBits` | Non-zero means timestamps are reliable |
| `minImageTransferGranularity` | Minimum granularity for image copy regions |

The `queueFlags` values of interest:

- `VK_QUEUE_GRAPHICS_BIT` — `vkCmdDraw*`, render passes, graphics pipelines
- `VK_QUEUE_COMPUTE_BIT` — `vkCmdDispatch*`, `vkCmdTraceRays*`
- `VK_QUEUE_TRANSFER_BIT` — `vkCmdCopy*`, blits; implicitly set on GRAPHICS and COMPUTE queues
- `VK_QUEUE_SPARSE_BINDING_BIT` — `vkQueueBindSparse` for sparse resources
- `VK_QUEUE_VIDEO_DECODE_BIT_KHR` — `vkCmdDecodeVideoKHR` (requires `VK_KHR_video_decode_queue`)
- `VK_QUEUE_VIDEO_ENCODE_BIT_KHR` — `vkCmdEncodeVideoKHR` (requires `VK_KHR_video_encode_queue`)

[Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VkQueueFlagBits.html)

### 3.2 Typical Family Layouts per Driver

**RADV (AMD RDNA 3, Mesa 26.x):**

| Family | Flags | Count |
|---|---|---|
| 0 | GRAPHICS \| COMPUTE \| TRANSFER \| SPARSE | 1 |
| 1 | COMPUTE (ACE — async compute) | 2 |
| 2 | TRANSFER (SDMA) | 1 |
| 3 | VIDEO_DECODE | 1 |

> **Note:** The dedicated TRANSFER family (family 2) backed by SDMA landed in Mesa 26.0 and requires RDNA2 or newer hardware. Older Mesa releases and older hardware may not expose this family. [Source](https://phoronix.com/news/Mesa-26.0-RADV-Transfer-SDMA)

**ANV (Intel Xe2 / Arc B-series):**

Typically one general-purpose family (`GRAPHICS | COMPUTE | TRANSFER | SPARSE`) plus a separate video family (`VIDEO_DECODE | VIDEO_ENCODE`). The number of queues in the primary family reflects Intel's hardware execution submission rings.

**NVK (NVIDIA, open kernel driver, Mesa):**

NVIDIA exposes a single family (`GRAPHICS | COMPUTE | TRANSFER | SPARSE`) with a `queueCount` that reflects the number of hardware submission rings available.

### 3.3 Creating Queues

```c
float priorities[] = { 1.0f, 0.5f };  // graphics high priority, ACE lower

VkDeviceQueueCreateInfo queueInfos[2] = {
    {
        .sType            = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO,
        .queueFamilyIndex = graphicsFamilyIndex,   // family 0
        .queueCount       = 1,
        .pQueuePriorities = &priorities[0],
    },
    {
        .sType            = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO,
        .queueFamilyIndex = computeFamilyIndex,    // family 1 (ACE)
        .queueCount       = 1,
        .pQueuePriorities = &priorities[1],
    },
};

VkDeviceCreateInfo deviceInfo = {
    .sType                = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO,
    .queueCreateInfoCount = 2,
    .pQueueCreateInfos    = queueInfos,
    // ... features, extensions ...
};
vkCreateDevice(physicalDevice, &deviceInfo, NULL, &device);

VkQueue graphicsQueue, asyncComputeQueue;
vkGetDeviceQueue(device, graphicsFamilyIndex, 0, &graphicsQueue);
vkGetDeviceQueue(device, computeFamilyIndex,  0, &asyncComputeQueue);
```

Queue priorities (0.0–1.0) are a hint to the driver scheduler; their exact effect is implementation-defined. On AMD hardware, setting the present queue at priority 1.0 ensures it can preempt other work to meet VSync deadlines. [Source](https://docs.vulkan.org/guide/latest/queues.html)

---

## 4. Timeline Semaphores

### 4.1 The Problem with Binary Semaphores

Before `VK_KHR_timeline_semaphore`, developers synchronised multi-queue workloads with binary `VkSemaphore` objects. Binary semaphores have severe limitations:

- Each signal must be consumed by exactly one wait; the semaphore resets to unsignalled after the wait.
- A signal must be submitted before the corresponding wait can be submitted.
- The CPU can only poll via a fence, adding latency and allocation overhead.
- "Frames in flight" management required a distinct pool of semaphores for each in-flight index.

Three frames in flight across two queues meant allocating and carefully routing at least six binary semaphores — and any mistake caused a device-lost or hang.

### 4.2 Timeline Semaphore API

Timeline semaphores are a monotonically increasing 64-bit counter. They were promoted to Vulkan 1.2 core (requiring the `timelineSemaphore` feature). [Source](https://www.khronos.org/blog/vulkan-timeline-semaphores)

**Creation:**

```c
VkSemaphoreTypeCreateInfo typeInfo = {
    .sType         = VK_STRUCTURE_TYPE_SEMAPHORE_TYPE_CREATE_INFO,
    .semaphoreType = VK_SEMAPHORE_TYPE_TIMELINE,
    .initialValue  = 0,
};
VkSemaphoreCreateInfo semInfo = {
    .sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO,
    .pNext = &typeInfo,
};
VkSemaphore timeline;
vkCreateSemaphore(device, &semInfo, NULL, &timeline);
```

**GPU-side signal (queue submission):**

```c
uint64_t signalValue = frameIndex;  // e.g. 7 for frame 7

VkSemaphoreSubmitInfo signalInfo = {
    .sType     = VK_STRUCTURE_TYPE_SEMAPHORE_SUBMIT_INFO,
    .semaphore = timeline,
    .value     = signalValue,
    .stageMask = VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT,
};
VkSubmitInfo2 submit2 = {
    .sType                   = VK_STRUCTURE_TYPE_SUBMIT_INFO_2,
    .commandBufferInfoCount  = 1,
    .pCommandBufferInfos     = &cbSubmitInfo,
    .signalSemaphoreInfoCount = 1,
    .pSignalSemaphoreInfos   = &signalInfo,
};
vkQueueSubmit2(graphicsQueue, 1, &submit2, VK_NULL_HANDLE);
```

**GPU-side wait (second queue):**

```c
uint64_t waitValue = frameIndex;  // wait for graphics to finish frame

VkSemaphoreSubmitInfo waitInfo = {
    .sType     = VK_STRUCTURE_TYPE_SEMAPHORE_SUBMIT_INFO,
    .semaphore = timeline,
    .value     = waitValue,
    .stageMask = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
};
VkSubmitInfo2 asyncSubmit = {
    .sType                  = VK_STRUCTURE_TYPE_SUBMIT_INFO_2,
    .waitSemaphoreInfoCount = 1,
    .pWaitSemaphoreInfos    = &waitInfo,
    .commandBufferInfoCount = 1,
    .pCommandBufferInfos    = &asyncCbSubmitInfo,
};
vkQueueSubmit2(asyncComputeQueue, 1, &asyncSubmit, VK_NULL_HANDLE);
```

**CPU-side wait:**

```c
VkSemaphoreWaitInfo cpuWait = {
    .sType          = VK_STRUCTURE_TYPE_SEMAPHORE_WAIT_INFO,
    .semaphoreCount = 1,
    .pSemaphores    = &timeline,
    .pValues        = &signalValue,
};
vkWaitSemaphores(device, &cpuWait, UINT64_MAX);
```

**CPU-side signal** (for CPU→GPU synchronisation):

```c
VkSemaphoreSignalInfo cpuSignal = {
    .sType     = VK_STRUCTURE_TYPE_SEMAPHORE_SIGNAL_INFO,
    .semaphore = timeline,
    .value     = signalValue,
};
vkSignalSemaphore(device, &cpuSignal);
```

**Polling:**

```c
uint64_t currentValue;
vkGetSemaphoreCounterValue(device, timeline, &currentValue);
```

[Source](https://docs.vulkan.org/samples/latest/samples/extensions/timeline_semaphore/README.html)

### 4.3 Replacing the Fence + Binary Semaphore Dance

With timeline semaphores, a "frame in flight" reduces to a single integer:

```
Frame 7 graphics work signals timeline → 7
Frame 7 async compute waits ≥ 7, signals → 7 (separate timeline or same with offset)
CPU: if vkGetSemaphoreCounterValue ≥ 5, recycle frame 5's resources
```

This replaces the traditional `VkFence` per frame + multiple binary semaphores with one or two timeline semaphores. The counter never resets, so a "wait before signal" submission is legal: you can submit the async-compute wait before submitting the graphics signal, and the driver will defer the wait. This enables concurrent recording on multiple CPU threads without lock-step synchronisation.

### 4.4 The WSI Exception

**Important:** Window System Integration (WSI) operations — `vkAcquireNextImageKHR` and `vkQueuePresentKHR` — do **not** support timeline semaphores. `vkQueuePresentKHR` requires a binary semaphore as its wait signal. [Source](https://docs.vulkan.org/samples/latest/samples/extensions/timeline_semaphore/README.html)

The canonical pattern is: use timeline semaphores to coordinate all compute, graphics, and transfer work; then signal a binary semaphore from the final queue submission that the present operation will consume:

```c
// Final graphics submit: signals both the timeline AND a binary semaphore for present
VkSemaphoreSubmitInfo finalSignals[2] = {
    { .semaphore = timeline,   .value = frameIndex,  /* timeline */ },
    { .semaphore = renderDone, .value = 0,           /* binary */  },
};
```

---

## 5. Pipeline Barriers in Depth

### 5.1 Why Barriers Are Necessary

Vulkan's execution model is deliberately relaxed: the GPU may execute commands in any order that does not violate explicit synchronisation. Without barriers, a compute shader writing to a storage image and a fragment shader sampling that image may genuinely run in the wrong order, producing a data race. Barriers make ordering constraints explicit and also flush and invalidate GPU caches.

### 5.2 The synchronization2 API (Vulkan 1.3)

`VK_KHR_synchronization2` (promoted to Vulkan 1.3 core) replaced the old `vkCmdPipelineBarrier` with a unified `vkCmdPipelineBarrier2` that accepts a `VkDependencyInfo`. The key structural improvement: source stage, source access, destination stage, and destination access are now co-located in a single barrier struct instead of being split across the function call and the barrier array. [Source](https://docs.vulkan.org/guide/latest/extensions/VK_KHR_synchronization2.html)

```c
// Example: compute shader writes storage image; fragment shader samples it
VkImageMemoryBarrier2 imgBarrier = {
    .sType            = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER_2,
    .srcStageMask     = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
    .srcAccessMask    = VK_ACCESS_2_SHADER_WRITE_BIT,
    .dstStageMask     = VK_PIPELINE_STAGE_2_FRAGMENT_SHADER_BIT,
    .dstAccessMask    = VK_ACCESS_2_SHADER_SAMPLED_READ_BIT,
    .oldLayout        = VK_IMAGE_LAYOUT_GENERAL,
    .newLayout        = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL,
    .srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED,
    .dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED,
    .image            = storageImage,
    .subresourceRange = { VK_IMAGE_ASPECT_COLOR_BIT, 0, 1, 0, 1 },
};

VkDependencyInfo dep = {
    .sType                   = VK_STRUCTURE_TYPE_DEPENDENCY_INFO,
    .imageMemoryBarrierCount = 1,
    .pImageMemoryBarriers    = &imgBarrier,
};
vkCmdPipelineBarrier2(commandBuffer, &dep);
```

### 5.3 Stage Masks

Stage masks (64-bit in synchronization2) specify **when** in the GPU pipeline the dependency takes effect:

| Stage | Meaning |
|---|---|
| `VK_PIPELINE_STAGE_2_TOP_OF_PIPE_BIT` | Beginning of queue; no-op for waits |
| `VK_PIPELINE_STAGE_2_DRAW_INDIRECT_BIT` | Indirect parameter reads |
| `VK_PIPELINE_STAGE_2_VERTEX_INPUT_BIT` | Index and vertex buffer reads |
| `VK_PIPELINE_STAGE_2_VERTEX_SHADER_BIT` | Vertex shader execution |
| `VK_PIPELINE_STAGE_2_FRAGMENT_SHADER_BIT` | Fragment shader execution |
| `VK_PIPELINE_STAGE_2_COLOR_ATTACHMENT_OUTPUT_BIT` | Render target writes |
| `VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT` | Compute dispatch execution |
| `VK_PIPELINE_STAGE_2_TRANSFER_BIT` | Copy and blit commands |
| `VK_PIPELINE_STAGE_2_VIDEO_DECODE_BIT_KHR` | Video decode engine |
| `VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT` | All stages (convenience, but kills overlap) |

Using `ALL_COMMANDS_BIT` as a source stage is a common mistake that eliminates any possibility of async overlap. Prefer the narrowest applicable stage.

### 5.4 Access Masks

Access masks specify **what kind of memory** was produced or will be consumed:

| Access | Meaning |
|---|---|
| `VK_ACCESS_2_INDIRECT_COMMAND_READ_BIT` | Indirect draw/dispatch parameters |
| `VK_ACCESS_2_INDEX_READ_BIT` | Index buffer reads |
| `VK_ACCESS_2_VERTEX_ATTRIBUTE_READ_BIT` | Vertex attribute reads |
| `VK_ACCESS_2_UNIFORM_READ_BIT` | UBO reads |
| `VK_ACCESS_2_SHADER_SAMPLED_READ_BIT` | Sampled image reads |
| `VK_ACCESS_2_SHADER_STORAGE_READ_BIT` | Storage buffer/image reads |
| `VK_ACCESS_2_SHADER_WRITE_BIT` | Storage buffer/image writes |
| `VK_ACCESS_2_COLOR_ATTACHMENT_WRITE_BIT` | Colour attachment writes |
| `VK_ACCESS_2_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT` | Depth/stencil writes |
| `VK_ACCESS_2_TRANSFER_WRITE_BIT` | DMA/copy engine writes |
| `VK_ACCESS_2_TRANSFER_READ_BIT` | DMA/copy engine reads |

### 5.5 Image Layout Transitions

Vulkan images have a current **layout** that describes how their texels are arranged in memory. The common sequence for a GBuffer colour attachment:

```
VK_IMAGE_LAYOUT_UNDEFINED
  → VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL   (rendering)
  → VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL   (sampled in lighting pass)
  → VK_IMAGE_LAYOUT_PRESENT_SRC_KHR            (WSI presentation)
```

Each transition is expressed as a `VkImageMemoryBarrier2` with `oldLayout` and `newLayout`. The driver generates whatever hardware commands are needed to reformat texel data (often a no-op on tiling-based GPUs if the underlying memory does not change).

### 5.6 The Minimal Barrier Principle

A common beginner pattern is to insert `VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT` as both `srcStageMask` and `dstStageMask` on every barrier. This is semantically correct but performance-devastating: it forces the GPU to idle every pipeline stage, flush all caches, and wait for every outstanding command to complete before any subsequent command can start. The GPU behaves like a sequential CPU.

The correct approach is to express the **narrowest dependency** that is actually true:

```c
// BAD: blocks the entire GPU
VkMemoryBarrier2 badBarrier = {
    .srcStageMask  = VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT,
    .srcAccessMask = VK_ACCESS_2_MEMORY_WRITE_BIT,
    .dstStageMask  = VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT,
    .dstAccessMask = VK_ACCESS_2_MEMORY_READ_BIT,
};

// GOOD: only stalls fragment shaders waiting for compute writes
VkImageMemoryBarrier2 goodBarrier = {
    .srcStageMask  = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
    .srcAccessMask = VK_ACCESS_2_SHADER_WRITE_BIT,
    .dstStageMask  = VK_PIPELINE_STAGE_2_FRAGMENT_SHADER_BIT,
    .dstAccessMask = VK_ACCESS_2_SHADER_SAMPLED_READ_BIT,
    // ... image fields ...
};
```

The narrower barrier allows the GPU's vertex shader stage to continue processing independent draw calls while the fragment shader stage waits for the compute output to become visible.

A useful mental model: think of barriers as arrows in the dependency DAG — they connect a specific producing operation to a specific consuming operation. The barrier should name exactly those two operations, nothing more.

### 5.7 Queue Family Ownership Transfer

When a resource must move from one queue family to another (e.g., from the async compute family to the graphics family after a compute dispatch), a two-phase ownership transfer is required:

```c
// Phase 1: Release on compute queue
VkImageMemoryBarrier2 release = {
    // ... same src stage/access as before ...
    .srcQueueFamilyIndex = computeFamilyIndex,
    .dstQueueFamilyIndex = graphicsFamilyIndex,
    .oldLayout = VK_IMAGE_LAYOUT_GENERAL,
    .newLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL,
    .image = computeOutputImage,
    .subresourceRange = { ... },
};
// submitted with signalSemaphore

// Phase 2: Acquire on graphics queue (after waiting on the semaphore)
VkImageMemoryBarrier2 acquire = {
    .srcStageMask  = VK_PIPELINE_STAGE_2_NONE,   // no src ops needed
    .srcAccessMask = VK_ACCESS_2_NONE,
    .dstStageMask  = VK_PIPELINE_STAGE_2_FRAGMENT_SHADER_BIT,
    .dstAccessMask = VK_ACCESS_2_SHADER_SAMPLED_READ_BIT,
    .srcQueueFamilyIndex = computeFamilyIndex,
    .dstQueueFamilyIndex = graphicsFamilyIndex,
    .oldLayout = VK_IMAGE_LAYOUT_GENERAL,
    .newLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL,
    .image = computeOutputImage,
    // ...
};
```

The semaphore between the two submissions ensures the release has been submitted before the acquire executes. [Source](https://github.com/khronosgroup/vulkan-docs/wiki/synchronization-examples)

If the resource will only be read on the destination queue (no layout transition needed, same queue family), and the resource was allocated with `VK_SHARING_MODE_CONCURRENT`, the ownership transfer can be omitted entirely — `CONCURRENT` mode allows all specified queue families to access the resource without ownership. The trade-off: `CONCURRENT` may incur a small performance penalty on some hardware because the driver cannot assume exclusive ownership for cache optimisation. For frequently transferred resources, prefer `EXCLUSIVE` mode with explicit ownership transfer; for rarely-shared resources or where the code simplification matters more, `CONCURRENT` is acceptable.

---

## 6. Async Compute: Overlapping Graphics and Compute

### 6.1 The Opportunity

On AMD RDNA hardware, the GFX front-end handles vertex processing, and ACE queues handle compute dispatches. When the GFX queue is processing the Z-prepass (highly vertex-bound, few compute resources occupied), ACE queues can simultaneously dispatch particle simulation on the idle compute units. This is not a software fiction: the hardware MES firmware genuinely runs these in parallel.

The Vulkan Samples async compute benchmark measured an **8.2% frame time improvement** (21.8 ms vs 22.9 ms) on real hardware by properly interleaving compute post-processing with graphics work. [Source](https://docs.vulkan.org/samples/latest/samples/performance/async_compute/README.html)

NVIDIA's async compute model is architecturally different — there is one unified SM array rather than a separate ACE — but the driver's ASYNC_COMPUTE queue can still preempt or interleave with DIRECT queue work on a per-warp basis, and NVIDIA's own testing shows benefits especially when combining math-limited compute with shadow rasterisation or when combining ray tracing with denoising. [Source](https://developer.nvidia.com/blog/advanced-api-performance-async-compute-and-overlap/)

### 6.2 Setup

```c
// graphicsFamilyIndex: family with GRAPHICS|COMPUTE bit (family 0 on RADV)
// computeFamilyIndex:  family with COMPUTE only       (family 1 on RADV — ACE)

VkQueue graphicsQueue, asyncComputeQueue;
vkGetDeviceQueue(device, graphicsFamilyIndex, 0, &graphicsQueue);
vkGetDeviceQueue(device, computeFamilyIndex,  0, &asyncComputeQueue);
```

### 6.3 A Concrete Overlap: Particles During Z-Prepass

```
Graphics thread records:     [Z-prepass cmd buf]
                                 signals timeline value 1 after VERTEX_SHADER stage

Async compute thread records: [Particle simulate cmd buf]
                                 waits for timeline ≥ 1 (particles need last frame depth)
                                 signals timeline value 2 after COMPUTE_SHADER stage

Graphics thread (continued):  [Shadow maps, lighting ...]
                                 waits for timeline ≥ 2 (lights need updated particles)
```

```c
// Submit Z-prepass; signal timeline value = FRAME_BASE + 1
VkSemaphoreSubmitInfo zSignal = {
    .sType     = VK_STRUCTURE_TYPE_SEMAPHORE_SUBMIT_INFO,
    .semaphore = frameSemaphore,
    .value     = frameBase + 1,
    .stageMask = VK_PIPELINE_STAGE_2_VERTEX_SHADER_BIT,
};
VkSubmitInfo2 zSubmit = {
    .sType                    = VK_STRUCTURE_TYPE_SUBMIT_INFO_2,
    .commandBufferInfoCount   = 1,
    .pCommandBufferInfos      = &zCBInfo,
    .signalSemaphoreInfoCount = 1,
    .pSignalSemaphoreInfos    = &zSignal,
};
vkQueueSubmit2(graphicsQueue, 1, &zSubmit, VK_NULL_HANDLE);

// Submit async particle simulation; waits for value 1, signals value 2
VkSemaphoreSubmitInfo particleWait = {
    .semaphore = frameSemaphore, .value = frameBase + 1,
    .stageMask = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
};
VkSemaphoreSubmitInfo particleSignal = {
    .semaphore = frameSemaphore, .value = frameBase + 2,
    .stageMask = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
};
VkSubmitInfo2 particleSubmit = {
    .sType                    = VK_STRUCTURE_TYPE_SUBMIT_INFO_2,
    .waitSemaphoreInfoCount   = 1, .pWaitSemaphoreInfos   = &particleWait,
    .commandBufferInfoCount   = 1, .pCommandBufferInfos   = &particleCBInfo,
    .signalSemaphoreInfoCount = 1, .pSignalSemaphoreInfos = &particleSignal,
};
vkQueueSubmit2(asyncComputeQueue, 1, &particleSubmit, VK_NULL_HANDLE);
```

### 6.4 Candidate Workloads for Async Compute

Good candidates have low register pressure and are not immediately in the critical path of the next draw call:

- Particle simulation / physics integration
- GPU-driven occlusion culling and draw-call generation
- SSAO (screen-space ambient occlusion) in a deferred pipeline
- TAA (temporal anti-aliasing) when it reads the previous frame's buffer
- Tone mapping and bloom when run after opaque geometry
- DLSS / FSR 4 upscaling passes

Poor candidates:
- Passes that the GFX queue immediately needs as input (the wait negates the gain)
- Heavy texture-sampling workloads (competes for TMU bandwidth with GFX)
- Dispatches that use `VK_PIPELINE_STAGE_ALL_COMMANDS_BIT` as wait stage

### 6.5 Pitfalls

1. **Over-broad barriers**: Using `ALL_COMMANDS_BIT` as `srcStageMask` blocks the GPU from overlapping any subsequent commands.
2. **Ownership transfer forgotten**: Reading a resource written by a different queue family without the release/acquire pair is undefined behaviour.
3. **Memory dependency without barrier**: On NVIDIA RTX 3090, a missing barrier after async compute caused sporadic image corruption even though the timeline semaphore provided the correct execution order. Execution and memory ordering are separate. [Source](https://github.com/nvpro-samples/vk_timeline_semaphore)
4. **Too many short submits**: Each `vkQueueSubmit2` has CPU overhead (~10–30 µs). Batch multiple command buffers into a single submit where possible.

---

## 7. DMA Queue and Streaming

### 7.1 The Transfer Queue

Queue families that expose only `VK_QUEUE_TRANSFER_BIT` (no GRAPHICS or COMPUTE) map to dedicated DMA engines. On AMD RDNA 3, this is the SDMA engine. On NVIDIA it maps to copy engines. The key property: these engines run **entirely independently** of the shader array, so a DMA transfer does not consume CU/SM time.

As of Mesa 26.0, RADV exposes the SDMA engine as a dedicated transfer queue family backed by SDMA image copies. [Source](https://phoronix.com/news/Mesa-26.0-RADV-Transfer-SDMA)

### 7.2 The Staging Buffer Pattern

Device-local memory (`VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT`) is fastest for the GPU but is not visible to the CPU. Uploads require a **staging buffer** in host-visible memory:

```c
// 1. Allocate staging buffer (HOST_VISIBLE | HOST_COHERENT)
VkBufferCreateInfo stagingInfo = { .size = dataSize, .usage = VK_BUFFER_USAGE_TRANSFER_SRC_BIT };
// ... allocate via VMA with VMA_MEMORY_USAGE_CPU_ONLY ...

// 2. Write data to staging buffer (CPU)
memcpy(stagingMappedPtr, textureData, dataSize);

// 3. Record copy on transfer queue
VkBufferImageCopy region = {
    .bufferOffset      = 0,
    .imageSubresource  = { VK_IMAGE_ASPECT_COLOR_BIT, mipLevel, 0, 1 },
    .imageExtent       = { width, height, 1 },
};
vkCmdCopyBufferToImage(transferCB, stagingBuffer, destImage,
                        VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, 1, &region);
```

### 7.3 Streaming with Timeline Semaphores

```
Transfer queue:
  [Upload LOD k+1 for texture T]  →  signals timeline value N+1

GFX queue:
  [Render frame N using LOD k]    →  signals timeline value N
  waits for N+1 before using LOD k+1 in frame N+1
```

PCIe 4.0 x16 provides approximately 32 GB/s aggregate bandwidth (16 GB/s each direction). GPU VRAM bandwidth is 1+ TB/s (GDDR6X on RDNA 4 or HBM3 on MI series). Streaming must respect this 30× asymmetry: only upload what will actually be sampled, prefetch several frames ahead, and avoid contention between the DMA engine and the GPU's own VRAM accesses.

### 7.4 VK_EXT_external_memory_host

`VK_EXT_external_memory_host` allows importing a CPU allocation directly as a Vulkan buffer, avoiding the host→staging copy:

```c
VkImportMemoryHostPointerInfoEXT importInfo = {
    .sType        = VK_STRUCTURE_TYPE_IMPORT_MEMORY_HOST_POINTER_INFO_EXT,
    .handleType   = VK_EXTERNAL_MEMORY_HANDLE_TYPE_HOST_ALLOCATION_BIT_EXT,
    .pHostPointer = cpuAllocatedTextureData,
};
// Use importInfo in VkMemoryAllocateInfo.pNext
// Then vkCmdCopyBufferToImage directly from the imported buffer
```

The alignment requirements are exposed via `vkGetPhysicalDeviceExternalBufferProperties` and are typically the system page size (4096 bytes). [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_external_memory_host.html)

### 7.5 Sparse Textures

`VK_QUEUE_SPARSE_BINDING_BIT` enables **sparse images**: images whose mip levels are backed by individually-bindable memory pages. `vkQueueBindSparse` (which requires a queue with `SPARSE_BINDING_BIT`) submits page-in and page-out requests. Combined with the transfer queue, this enables streaming virtual textures: the transfer queue uploads new pages, then `vkQueueBindSparse` maps them into the sparse image address space.

---

## 8. Frame Graphs / Task Graphs

### 8.1 Motivation

A modern game frame might contain 50–100 render and compute passes. Each pass has read dependencies on earlier passes and write dependencies that subsequent passes consume. Correctly inserting pipeline barriers at every transition — and doing so optimally — is impractical by hand. Worse, barrier placement changes whenever the render pipeline is reorganised.

The **Frame Graph** (introduced by Yuriy O'Donnell at Frostbite, GDC 2017) [Source](https://www.gdcvault.com/play/1024612/FrameGraph-Extensible-Rendering-Architecture-in) and subsequently refined in Themaister's Granite renderer [Source](https://themaister.net/blog/2017/08/15/render-graphs-and-vulkan-a-deep-dive/) solve this by lifting synchronisation to a compile-time step over a directed acyclic graph (DAG).

### 8.2 Core Concepts

```
             ┌──────────────────────┐
             │   Frame Graph DAG    │
             │                      │
 [Z-prepass] ─► [Shadow maps] ─────►┐
     │                               ├─► [Lighting] ─► [TAA] ─► [Present]
     └──► [Particle sim (ACE)] ─────►┘
                    │
             [Culling (ACE)]
```

**Nodes** (passes): A pass declares its read and write resources as virtual handles. The pass has an `Execute(VkCommandBuffer)` callback where it records actual Vulkan commands.

**Edges** (resources): A `RenderResource` handle is a logical resource (image or buffer) that does not correspond to a real `VkImage` or `VkBuffer` until the graph is compiled. This allows the compiler to alias resources that have non-overlapping lifetimes and share memory.

### 8.3 Compilation

The frame graph compiler performs:

1. **Topological sort**: Determine execution order from the dependency DAG.
2. **Pass culling**: Any pass whose outputs are not consumed by the current frame's output (e.g., the final swapchain image) is eliminated.
3. **Resource aliasing**: If resource A is last read at pass 12 and resource B is first written at pass 15, they can share the same `VkDeviceMemory` allocation. Granite reports that aliasing saves 30–40% of render target memory in typical deferred pipelines.
4. **Queue assignment**: Passes with no dependency on the GFX queue's outputs can be assigned to the async compute queue.
5. **Barrier insertion**: At every resource state transition, the compiler inserts the minimal correct `VkImageMemoryBarrier2` or `VkBufferMemoryBarrier2`. Cross-queue transitions become queue family ownership transfers with semaphores.
6. **Physical resource creation**: `RenderResource` handles resolve to real `VkImage`s and `VkBuffer`s, possibly with aliased `VkDeviceMemory`.

### 8.4 Granite's Implementation

Granite (Themaister, [https://github.com/Themaister/Granite](https://github.com/Themaister/Granite)) is an open-source reference implementation. Key aspects of its render graph:

- Aims for one graphics queue, one async compute queue (separate family if available), and one DMA transfer queue.
- Uses `VkEvent` for dependencies within a single queue (lower overhead than `vkCmdPipelineBarrier2` when the producing pass finishes before the consuming pass starts on the same queue).
- Uses `VkSemaphore` for cross-queue resource transitions.
- A `physical_pass_transfer_ownership` function handles the release/acquire pair automatically.
- Resources that cross queue families are tracked per-family with separate semaphore sets.

[Source](https://github.com/Themaister/Granite/blob/master/renderer/render_graph.hpp)

### 8.5 Production Engine Implementations

**Unreal Engine 5 (RDG)**: Unreal's Render Dependency Graph (RDG) is the production evolution of this pattern. It integrates with the Vulkan RHI to automatically insert pipeline barriers and uses async compute queues for TAA, shadow depth renders, and various compute passes. [Source](https://docs.unrealengine.com/5.0/en-US/render-dependency-graph-in-unreal-engine/)

**azhirnov/FrameGraph** ([https://github.com/azhirnov/FrameGraph](https://github.com/azhirnov/FrameGraph)): A standalone Vulkan abstraction where the entire frame is represented as a task graph with nodes for draw, dispatch, copy, and present tasks.

**bgfx**: The `bgfx` library implements a simpler frame graph on top of Vulkan and other APIs, focusing on render pass management rather than full barrier automation (see Ch84).

### 8.6 VK_EXT_attachment_feedback_loop_layout

Some passes read from and write to the same attachment (e.g., a TAA accumulation buffer). Normally this requires a layout transition barrier. `VK_EXT_attachment_feedback_loop_layout` introduces `VK_IMAGE_LAYOUT_ATTACHMENT_FEEDBACK_LOOP_OPTIMAL_EXT`, allowing simultaneous read (as a sampled image) and write (as a colour attachment) within the same render pass, eliminating the round-trip barrier. Frame graph compilers should detect this pattern and apply it automatically. [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_attachment_feedback_loop_layout.html)

---

## 9. Practical Multi-Queue Frame Structure

### 9.1 A Modern Game Frame

The following timeline illustrates how the three queue types interleave in a contemporary game frame. Vertical columns are time; rows are queues.

```
Time →    t0           t1              t2              t3              t4       t5
GFX:  [UpldCmd] [Z-prepass+Shadow] [Lighting+RT] [Forward+ shade] [Blit/UI] [Present]
ACE:              [Particle sim]   [GPU Culling]   [TAA + Bloom]
DMA:  [Upload LOD k+1]─────────────────────────────────────────────────────────►
```

Dependencies:

| Producer | Consumer | Semaphore value |
|---|---|---|
| DMA upload complete | GFX first frame using new texture | N+1 |
| GFX Z-prepass | ACE particle sim (reads depth) | N+2 |
| ACE particle sim | GFX lighting (particles in scene) | N+3 |
| GFX lighting | ACE culling (reads visibility buffer) | N+4 |
| ACE culling + GFX | GFX Forward+ (draw indirect) | N+5 |
| GFX Forward+ | ACE TAA (reads colour + velocity) | N+6 |
| ACE TAA | GFX blit/UI | N+7 |
| GFX blit done | WSI present (binary semaphore) | (binary) |

### 9.2 VkSubmitInfo2 for Each Stage

Each logical stage maps to one or more `vkQueueSubmit2` calls. For the lighting pass:

```c
VkSemaphoreSubmitInfo lightWaits[] = {
    // Wait for Z-prepass (value N+2) on FRAGMENT_SHADER stage
    { .semaphore = frame.timeline, .value = frameBase + 2,
      .stageMask = VK_PIPELINE_STAGE_2_FRAGMENT_SHADER_BIT },
    // Wait for particle sim (value N+3) on VERTEX_SHADER_BIT (indirect draws)
    { .semaphore = frame.timeline, .value = frameBase + 3,
      .stageMask = VK_PIPELINE_STAGE_2_DRAW_INDIRECT_BIT },
};
VkSemaphoreSubmitInfo lightSignal = {
    .semaphore = frame.timeline, .value = frameBase + 4,
    .stageMask = VK_PIPELINE_STAGE_2_COLOR_ATTACHMENT_OUTPUT_BIT,
};
VkCommandBufferSubmitInfo lightCBInfo = {
    .sType         = VK_STRUCTURE_TYPE_COMMAND_BUFFER_SUBMIT_INFO,
    .commandBuffer = lightingCB,
};
VkSubmitInfo2 lightSubmit = {
    .sType                    = VK_STRUCTURE_TYPE_SUBMIT_INFO_2,
    .waitSemaphoreInfoCount   = 2, .pWaitSemaphoreInfos   = lightWaits,
    .commandBufferInfoCount   = 1, .pCommandBufferInfos   = &lightCBInfo,
    .signalSemaphoreInfoCount = 1, .pSignalSemaphoreInfos = &lightSignal,
};
vkQueueSubmit2(graphicsQueue, 1, &lightSubmit, VK_NULL_HANDLE);
```

### 9.3 Profiling Multi-Queue Execution

To verify that queues actually overlap, measure GPU timestamps on each queue:

```c
// Create one query pool per queue family
VkQueryPoolCreateInfo qpInfo = {
    .sType      = VK_STRUCTURE_TYPE_QUERY_POOL_CREATE_INFO,
    .queryType  = VK_QUERY_TYPE_TIMESTAMP,
    .queryCount = 64,
};
VkQueryPool gfxQP, acePQ;
vkCreateQueryPool(device, &qpInfo, NULL, &gfxQP);
vkCreateQueryPool(device, &qpInfo, NULL, &acePQ);

// In Z-prepass command buffer (GFX queue)
vkCmdWriteTimestamp2(zCB, VK_PIPELINE_STAGE_2_TOP_OF_PIPE_BIT, gfxQP, 0);
// ... draw calls ...
vkCmdWriteTimestamp2(zCB, VK_PIPELINE_STAGE_2_BOTTOM_OF_PIPE_BIT, gfxQP, 1);

// In particle sim command buffer (ACE queue)
vkCmdWriteTimestamp2(particleCB, VK_PIPELINE_STAGE_2_TOP_OF_PIPE_BIT, acePQ, 0);
// ... dispatches ...
vkCmdWriteTimestamp2(particleCB, VK_PIPELINE_STAGE_2_BOTTOM_OF_PIPE_BIT, acePQ, 1);
```

After the frame, read back timestamps and correlate using the `timestampPeriod` from `VkPhysicalDeviceLimits` (nanoseconds per tick). If the ACE start time is earlier than the GFX Z-prepass end time, overlap occurred.

**RenderDoc** (version 1.28+) has a multi-queue timeline view that shows all queues on a shared axis, making overlap immediately visible. Tools like AMD Radeon GPU Profiler and NVIDIA Nsight Graphics provide more detailed per-engine traces.

### 9.4 The Present Semaphore Chain

Because WSI does not accept timeline semaphores, the transition from timeline-based synchronisation to presentation requires special handling. The standard pattern is:

1. The final graphics or compute submit signals **two** semaphores: the timeline (for CPU-side tracking and next-frame async waits) and a binary semaphore (for `vkQueuePresentKHR`).
2. `vkAcquireNextImageKHR` signals a second binary semaphore (`imageAcquired`) when the swapchain image is safe to render to.
3. The first "real" submit of the frame waits on `imageAcquired`.

```c
VkSemaphore imageAcquired;      // binary — signalled by vkAcquireNextImageKHR
VkSemaphore renderComplete;     // binary — signalled by final submit, consumed by present
VkSemaphore frameSemaphore;     // timeline — for all internal queue coordination

// Acquire swapchain image
uint32_t imageIndex;
vkAcquireNextImageKHR(device, swapchain, UINT64_MAX, imageAcquired, VK_NULL_HANDLE, &imageIndex);

// ... (all the multi-queue rendering, using frameSemaphore for coordination) ...

// Final submit: signals both timeline AND binary renderComplete
VkSemaphoreSubmitInfo finalSignals[2] = {
    {
        .sType     = VK_STRUCTURE_TYPE_SEMAPHORE_SUBMIT_INFO,
        .semaphore = frameSemaphore,
        .value     = frameBase + 8,           // timeline
        .stageMask = VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT,
    },
    {
        .sType     = VK_STRUCTURE_TYPE_SEMAPHORE_SUBMIT_INFO,
        .semaphore = renderComplete,
        .value     = 0,                        // binary (value ignored)
        .stageMask = VK_PIPELINE_STAGE_2_COLOR_ATTACHMENT_OUTPUT_BIT,
    },
};

// Present: consumes renderComplete binary semaphore
VkPresentInfoKHR presentInfo = {
    .sType              = VK_STRUCTURE_TYPE_PRESENT_INFO_KHR,
    .waitSemaphoreCount = 1,
    .pWaitSemaphores    = &renderComplete,
    .swapchainCount     = 1,
    .pSwapchains        = &swapchain,
    .pImageIndices      = &imageIndex,
};
vkQueuePresentKHR(graphicsQueue, &presentInfo);
```

This ensures the timeline semaphore tracks the frame's completion for CPU-side resource recycling, while the WSI binary semaphore path remains correct.

### 9.5 Frames in Flight Management

With timeline semaphores, frames-in-flight management is clean:

```c
#define MAX_FRAMES_IN_FLIGHT 3

VkSemaphore frameSemaphore;  // one per device, monotonic counter

uint64_t frameCounter = 0;  // increments each frame

// Start of frame N:
uint64_t waitForValue = (frameCounter > MAX_FRAMES_IN_FLIGHT)
    ? frameCounter - MAX_FRAMES_IN_FLIGHT
    : 0;

VkSemaphoreWaitInfo fi = {
    .sType = VK_STRUCTURE_TYPE_SEMAPHORE_WAIT_INFO,
    .semaphoreCount = 1, .pSemaphores = &frameSemaphore, .pValues = &waitForValue,
};
vkWaitSemaphores(device, &fi, UINT64_MAX);
// Now safe to recycle resources from frame (frameCounter - MAX_FRAMES_IN_FLIGHT)

frameCounter++;
```

---

## 10. Vulkan Video Queues

### 10.1 Video Queue Families

`VK_KHR_video_decode_queue` and `VK_KHR_video_encode_queue` add capability bits for hardware video codec engines. Queue families exposing `VK_QUEUE_VIDEO_DECODE_BIT_KHR` can process `vkCmdDecodeVideoKHR` commands. [Source](https://docs.vulkan.org/features/latest/features/proposals/VK_KHR_video_decode_queue.html)

Codec support is discovered via `VkVideoCapabilitiesKHR` and profile structures:

```c
VkVideoDecodeH264ProfileInfoKHR h264Profile = {
    .sType         = VK_STRUCTURE_TYPE_VIDEO_DECODE_H264_PROFILE_INFO_KHR,
    .stdProfileIdc = STD_VIDEO_H264_PROFILE_IDC_HIGH,
    .pictureLayout = VK_VIDEO_DECODE_H264_PICTURE_LAYOUT_INTERLACED_INTERLEAVED_LINES_BIT_KHR,
};
VkVideoProfileInfoKHR profile = {
    .sType               = VK_STRUCTURE_TYPE_VIDEO_PROFILE_INFO_KHR,
    .pNext               = &h264Profile,
    .videoCodecOperation = VK_VIDEO_CODEC_OPERATION_DECODE_H264_BIT_KHR,
    .chromaSubsampling   = VK_VIDEO_CHROMA_SUBSAMPLING_420_BIT_KHR,
    .lumaBitDepth        = VK_VIDEO_COMPONENT_BIT_DEPTH_8_BIT_KHR,
    .chromaBitDepth      = VK_VIDEO_COMPONENT_BIT_DEPTH_8_BIT_KHR,
};
VkVideoCapabilitiesKHR caps = { .sType = VK_STRUCTURE_TYPE_VIDEO_CAPABILITIES_KHR };
vkGetPhysicalDeviceVideoCapabilitiesKHR(physicalDevice, &profile, &caps);
```

Supported codecs (as of Vulkan SDK June 2026): H.264, H.265, AV1 decode; H.264, H.265 encode; VP9 decode (Vulkan Video Decode VP9 extension, June 2025). [Source](https://www.khronos.org/blog/khronos-releases-vulkan-video-av1-decode-extension-vulkan-sdk-now-supports-h.264-h.265-encode)

### 10.2 Video Session Setup

```c
VkVideoSessionCreateInfoKHR sessionInfo = {
    .sType            = VK_STRUCTURE_TYPE_VIDEO_SESSION_CREATE_INFO_KHR,
    .queueFamilyIndex = videoDecodeFamilyIndex,
    .pVideoProfile    = &profile,
    .pictureFormat    = VK_FORMAT_G8_B8R8_2PLANE_420_UNORM,  // NV12
    .maxCodedExtent   = { 3840, 2160 },
    .referencePictureFormat = VK_FORMAT_G8_B8R8_2PLANE_420_UNORM,
    .maxDpbSlots      = 17,  // H.264 max DPB size
    .maxActiveReferencePictures = 16,
    .pStdHeaderVersion = &h264StdHeaderVersion,
};
VkVideoSessionKHR videoSession;
vkCreateVideoSessionKHR(device, &sessionInfo, NULL, &videoSession);

// Bind memory to the video session
uint32_t memReqCount;
vkGetVideoSessionMemoryRequirementsKHR(device, videoSession, &memReqCount, NULL);
VkVideoSessionMemoryRequirementsKHR *memReqs = ...;
vkGetVideoSessionMemoryRequirementsKHR(device, videoSession, &memReqCount, memReqs);
// ... allocate and bind each memory requirement ...
```

### 10.3 Recording Decode Commands

```c
VkVideoBeginCodingInfoKHR beginInfo = {
    .sType          = VK_STRUCTURE_TYPE_VIDEO_BEGIN_CODING_INFO_KHR,
    .videoSession   = videoSession,
    .videoSessionParameters = sessionParams,
    .referenceSlotCount = dpbSlotCount,
    .pReferenceSlots    = dpbSlots,
};
vkCmdBeginVideoCodingKHR(videoDecodeCB, &beginInfo);

VkVideoDecodeInfoKHR decodeInfo = {
    .sType              = VK_STRUCTURE_TYPE_VIDEO_DECODE_INFO_KHR,
    .srcBuffer          = bitstreamBuffer,
    .srcBufferOffset    = nalOffset,
    .srcBufferRange     = nalSize,
    .dstPictureResource = { .imageViewBinding = outputImageView, ... },
    .pSetupReferenceSlot = &reconstructedSlot,
    .referenceSlotCount  = referenceCount,
    .pReferenceSlots     = referenceSlots,
};
vkCmdDecodeVideoKHR(videoDecodeCB, &decodeInfo);

vkCmdEndVideoCodingKHR(videoDecodeCB, &endInfo);
```

[Source](https://docs.vulkan.org/features/latest/features/proposals/VK_KHR_video_decode_queue.html)

### 10.4 Video Queue as an Async Peer

The video decode queue operates entirely independently of the graphics and compute queues. In a video playback + overlay application:

```
Video queue:   [Decode frame N] ─► signals timeline V+N
GFX queue:     waits V+N ─► [Composite video + UI] ─► present
```

Access flags for video/graphics synchronisation:

```c
VkImageMemoryBarrier2 videoToGraphics = {
    .srcStageMask  = VK_PIPELINE_STAGE_2_VIDEO_DECODE_BIT_KHR,
    .srcAccessMask = VK_ACCESS_2_VIDEO_DECODE_WRITE_BIT_KHR,
    .dstStageMask  = VK_PIPELINE_STAGE_2_FRAGMENT_SHADER_BIT,
    .dstAccessMask = VK_ACCESS_2_SHADER_SAMPLED_READ_BIT,
    .srcQueueFamilyIndex = videoDecodeFamilyIndex,
    .dstQueueFamilyIndex = graphicsFamilyIndex,
    .oldLayout = VK_IMAGE_LAYOUT_VIDEO_DECODE_DST_KHR,
    .newLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL,
    .image = decodedFrame,
    .subresourceRange = { VK_IMAGE_ASPECT_COLOR_BIT, 0, 1, 0, 1 },
};
```

### 10.5 Driver Status (2026)

**RADV**: H.264 and H.265 decode supported from TONGA through NAVI 3x (RDNA 3). AV1 decode on RDNA 3. H.264/H.265 encode in active development; `VK_KHR_video_encode_av1` merged mid-2025. [Source](https://airlied.blogspot.com/2025/07/radv-vkkhrvideoencodeav1-support.html)

**ANV (Intel)**: Vulkan Video decode supported on Gen12+ (Ice Lake discrete, Tiger Lake, and newer). Encode support in progress.

**NVK (NVIDIA open driver)**: Video queue implementation ongoing; the proprietary driver has full codec support.

The Vulkan Video ecosystem page at [rastergrid.com/vulkan-video/](https://www.rastergrid.com/vulkan-video/) maintains a current compatibility matrix. [Source](https://www.rastergrid.com/vulkan-video/)

---

## Roadmap

### Near-term (6–12 months)

- **`VK_KHR_internally_synchronized_queues` promotion to KHR.** Published as a KHR extension in January 2026 alongside Vulkan 1.4.340, this allows game engines to opt-in to driver-managed queue synchronization, eliminating the mandatory external locking on `vkQueueSubmit` calls and reducing CPU overhead in multi-threaded submission paths. [Source](https://docs.vulkan.org/features/latest/features/proposals/VK_KHR_internally_synchronized_queues.html)
- **`VK_EXT_descriptor_heap` driver adoption.** The collaborative descriptor heap extension (NVIDIA, AMD, Arm, Nintendo, Valve, Google) shipped in Vulkan 1.4.340 (January 2026) and Vulkan SDK Q1 2026. Descriptor heaps allow pre-allocated GPU-visible descriptor memory, reducing runtime reallocation stalls in multi-queue workloads where compute and graphics passes share resource views. [Source](https://www.khronos.org/blog/vulkan-introduces-roadmap-2026-and-new-descriptor-heap-extension)
- **Vulkan Roadmap 2026 milestone baseline adoption.** The Roadmap 2026 milestone formalises mandatory extensions for mid-range and high-end GPUs shipping in 2026 and later, including synchronization2 and timeline semaphores as guaranteed baseline features rather than optional extensions — reducing driver query boilerplate in multi-queue engines. [Source](https://www.phoronix.com/news/Vulkan-Roadmap-2026)
- **RADV and ANV video encode stabilisation.** `VK_KHR_video_encode_av1` merged into RADV mid-2025; remaining encode-queue plumbing (rate control, multi-slice) is expected to reach feature-complete status in Mesa 25.x. [Source](https://airlied.blogspot.com/2025/07/radv-vkkhrvideoencodeav1-support.html)
- **`VK_NV_external_compute_queue` ecosystem expansion.** NVIDIA's extension for joining external compute APIs (CUDA, OpenCL) to a `VkDevice` enables simultaneous execution between CUDA streams and Vulkan compute queues on the same hardware engine. Broader multi-vendor adoption is anticipated as the pattern proves out on the NV driver. [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_NV_external_compute_queue.html)

### Medium-term (1–3 years)

- **GPU Work Graphs (`VK_AMDX_shader_enqueue`) Mesh Node standardisation.** AMD's experimental extension, extended with mesh node support in 2025–2026, allows GPU-driven task dispatch graphs entirely driven by shader-generated payloads — eliminating CPU readback for culling and dispatch count decisions. A multi-vendor KHR equivalent is under discussion in the Khronos working group. [Source](https://www.khronos.org/news/archives/amd-blog-gpu-work-graphs-mesh-node-are-now-in-vulkan)
- **Vulkan ML extensions for compute-queue ML workloads.** The Vulkan ML Task Sub Group has published KHR extensions for 8-bit and 16-bit type support; follow-on work targets fused ML operators and memory layout hints that interact with async compute queue scheduling for inference-during-rendering workloads. [Source](https://www.khronos.org/blog/vulkan-continuing-to-forge-ahead-siggraph-2025)
- **Standardised render-graph / task-graph ABI.** Multiple engine vendors (Epic, Unity, id Software) have independently implemented frame-graph compilers on top of Vulkan timeline semaphores. The Khronos Vulkan Working Group has ongoing discussions about a lower-level "sync graph" API that would let drivers inspect the full frame dependency graph for hardware-level optimisation (e.g., eliminating L2 flushes between passes that share a cache line). Note: needs verification on formal proposal status.
- **NVK video queue completion.** The open-source NVIDIA Vulkan driver (NVK, in Mesa) has video queue implementation ongoing; full H.264/H.265/AV1 decode on NVK is expected to track RADV's implementation timeline within one to two Mesa release cycles.
- **`drm_sched` priority API extension.** Kernel-side work to expose finer-grained queue priorities via `drm_sched` (beyond the current three-level hint) would allow Vulkan `VK_DEVICE_QUEUE_CREATE_PROTECTED_BIT` and real-time compositor queues to be scheduled with tighter latency guarantees. [Source](https://www.kernel.org/doc/html/latest/gpu/drm-mm.html)

### Long-term

- **Hardware-accelerated dependency tracking.** Future GPU architectures may expose hardware barrier coalescing units that accept a DAG of resource dependencies rather than per-barrier pipeline stage masks, enabling finer-grained overlap without API-level barrier explosion in deeply pipelined frame graphs. Note: needs verification — speculative based on current hardware trajectory.
- **Unified GPU scheduling across Vulkan, CUDA, and media APIs.** Long-term kernel work targets a single `drm_sched` arbiter that can schedule Vulkan queues, CUDA compute streams, and V4L2/VA-API media queues with unified priority and preemption semantics — eliminating priority inversion when GPU-accelerated video decode competes with Vulkan async compute.
- **Ray tracing + async compute integration.** As ray tracing workloads mature, architectural guidance is expected to evolve around running BVH traversal on the dedicated RT cores while async compute queues simultaneously execute denoising (DLSS/XeSS/FSR) — requiring new pipeline stage flags and memory domain definitions in the synchronization2 framework.
- **Vulkan compute queue abstractions for heterogeneous compute.** Multi-chip module (MCM) GPUs and CPU+GPU tiled architectures raise questions about queue family semantics across chiplets with distinct L3 cache domains. Khronos is anticipated to address per-chiplet queue family annotations and cross-chiplet coherence domains in a future extension.

---

## 11. Integrations

**Ch24 — Vulkan API Fundamentals (`ch24-vulkan-egl-application-developers.md`):** Queue families, `VkDevice` creation, and basic command buffer recording are introduced there. This chapter builds on that foundation with multi-family queue setups and timeline semaphore synchronisation.

**Ch25 — GPU Compute (`ch25-gpu-compute.md`):** Async compute queues are the hardware substrate for compute shaders. Ch25 covers `vkCmdDispatch` semantics and descriptor sets; this chapter covers the scheduling and synchronisation layer that lets compute dispatches run concurrently with graphics work.

**Ch26 — Hardware Video (`ch26-hardware-video.md`):** VA-API and V4L2 approaches to hardware video decode on Linux. The Vulkan Video queue (Section 10) is Vulkan's native interface to the same hardware; Ch26 covers the KMS/DRM display side.

**Ch50 — Vulkan Video (`ch50-vulkan-video.md`):** Dedicated chapter on `VK_KHR_video_decode_queue`, session management, DPB (Decoded Picture Buffer) handling, and codec profiles. Section 10 of this chapter introduces the video queue model; Ch50 provides the full application-level treatment.

**Ch75 — Explicit GPU Sync (`ch75-explicit-gpu-sync.md` — DRM syncobj):** `drm_syncobj` is the kernel primitive underlying Vulkan semaphores and fences. The kernel-level timeline semaphore (`drm_syncobj` with timeline support) maps directly to `VK_SEMAPHORE_TYPE_TIMELINE`.

**Ch84 — bgfx (`ch84-bgfx.md`):** bgfx implements a simplified frame graph on top of Vulkan and other backends. Comparing bgfx's abstraction with the full Granite render graph illustrates the trade-off between portability and control.

**Ch97 — Unreal Engine 5 (`ch97-ue5.md`):** UE5's Render Dependency Graph (RDG) uses async compute queues extensively — TAA, shadow map atlas, and various post-process passes run on the async compute queue when the Vulkan RHI detects a suitable queue family. Ch97 covers the RHI layer; this chapter provides the Vulkan primitives it uses.

**Ch102 — DRM GPU Scheduler (`ch102-drm-sched.md`):** `drm_sched` is the kernel job scheduler that arbitrates between Vulkan queue submissions at the hardware level. It implements preemption, timeout detection, and fair scheduling between processes. Understanding `drm_sched` explains why Vulkan queue priorities are hints rather than guarantees, and why compositor submissions can preempt long-running application compute shaders.

**Ch127 — Mesh Shaders and VRS (`ch127-mesh-shaders-vrs.md`):** Mesh shader task/mesh pipeline stages introduce new `VK_PIPELINE_STAGE_2_TASK_SHADER_BIT_EXT` and `MESH_SHADER_BIT_EXT` that must be accounted for in barrier `srcStageMask`/`dstStageMask` values when mesh shader output feeds a subsequent compute pass.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
