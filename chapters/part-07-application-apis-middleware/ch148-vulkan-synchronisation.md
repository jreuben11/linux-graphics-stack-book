# Chapter 148: Vulkan Synchronisation: A Complete Developer Reference

This chapter targets **graphics application developers** writing Vulkan renderers and **systems developers** diagnosing GPU hangs, corruption, and race conditions caused by synchronisation errors. It assumes familiarity with the Vulkan object model and command buffer recording but treats synchronisation from first principles, because the Vulkan synchronisation model is fundamentally different from OpenGL and from most concurrent-programming models that developers encounter elsewhere. Incorrect synchronisation is not slow — it is undefined behaviour: pixels may be corrupt, the GPU may hang, or results may appear correct on one vendor's driver while crashing on another. This chapter covers every synchronisation primitive from `VkFence` to timeline semaphores, every barrier type from `vkCmdPipelineBarrier2` to render pass subpass dependencies, kernel-level `drm_syncobj` infrastructure, Wayland explicit sync, and the Vulkan Validation Layer's synchronisation mode.

---

## Table of Contents

- [Why Vulkan Sync Is Hard](#why-vulkan-sync-is-hard)
- [The Three Synchronisation Primitives](#the-three-synchronisation-primitives)
- [Fences in Depth](#fences-in-depth)
- [Binary Semaphores](#binary-semaphores)
- [Timeline Semaphores](#timeline-semaphores)
- [Pipeline Barriers](#pipeline-barriers)
- [Image Layout Transitions](#image-layout-transitions)
- [Render Pass Implicit Synchronisation](#render-pass-implicit-synchronisation)
- [Events: Split Barriers](#events-split-barriers)
- [Queue Families and Ownership Transfer](#queue-families-and-ownership-transfer)
- [Explicit Sync with the OS and Compositor](#explicit-sync-with-the-os-and-compositor)
- [Synchronisation Validation](#synchronisation-validation)
- [Real-World Patterns](#real-world-patterns)
- [Integrations](#integrations)

---

## Why Vulkan Sync Is Hard

OpenGL managed synchronisation implicitly. The driver tracked which draw calls had written to a texture before it was sampled, which buffers were in-flight before a `glBufferData` overwrite, and which render targets needed cache flushes before a `glReadPixels`. None of that tracking was free: OpenGL drivers contained — and still contain — elaborate dependency-tracking machinery. The performance cost was hidden but real: spurious flushes, unnecessary stalls, serialisation of work that could have overlapped.

Vulkan's design premise is that **the application knows the synchronisation graph**. A renderer knows that the shadow map pass always completes before the lighting pass samples it; a compute shader knows it writes to a buffer before the vertex shader reads it. Vulkan makes the application express these relationships explicitly so the driver can schedule them optimally without guesswork. The tradeoff is that expressing them incorrectly, or not at all, produces undefined behaviour rather than a runtime error.

Four properties of the GPU command model make explicit synchronisation necessary:

1. **The GPU is a pipeline.** A GPU executes thousands of shader invocations in parallel across many units. Commands submitted in order to a queue may execute in an order that maximises occupancy, not submission order. Memory written by draw N is not automatically visible to draw N+1.

2. **Caches are per-unit and incoherent.** Compute units have L1 and L2 caches. The framebuffer engine has its own tile memory on tile-based renderers. A write to GPU memory by the vertex shader is not automatically visible in the L1 cache of the fragment shader, and especially not in the tile memory of the ROP (raster operations pipeline). Making a write visible requires a cache flush (make-available) and cache invalidation (make-visible).

3. **CPU and GPU are separate processors with separate memory views.** When the CPU writes to a Vulkan buffer, those writes may still be in CPU cache and not yet flushed to physical memory. When the GPU writes to a Vulkan buffer, those writes may be in GPU L2 cache and not yet visible to the CPU. Synchronisation must span both the execution ordering and the memory visibility.

4. **Wrong sync is not slow — it is undefined behaviour.** The Vulkan specification [Source](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#synchronization) states that violating synchronisation requirements makes results undefined. A validation layer can sometimes catch violations. A correct driver on one vendor may silently hide a missing barrier while another vendor's driver crashes. GPU hangs, torn frames, and non-deterministic corruption are all symptomatic of missing or incorrect synchronisation.

The Vulkan specification models memory operations in terms of **availability** and **visibility** operations [Source](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#synchronization-dependencies-available-and-visible):

> *"A memory dependency is an execution dependency which includes availability and visibility operations such that: the first set of operations happens-before the availability operation; the availability operation happens-before the visibility operation; the visibility operation happens-before the second set of operations."*

The `srcAccessMask` in a barrier specifies which writes to **flush and make available** (cache writeback from the writing unit). The `dstAccessMask` specifies which reads to **invalidate and make visible** (cache invalidation so the reading unit sees fresh data). A critical insight: a write-after-read hazard (WAR) requires only an **execution dependency** — reads do not dirty caches, so no cache flush is needed, only ordering. This is why WAR barriers may have empty `srcAccessMask` and `dstAccessMask` in the sync2 API, using `VK_ACCESS_2_NONE` for both, with only the stage masks set.

---

## The Three Synchronisation Primitives

Vulkan provides three primitives, each addressing a different ordering relationship:

| Primitive | Scope | Direction | Typical Use |
|-----------|-------|-----------|-------------|
| **VkFence** | CPU ↔ GPU | GPU signals → CPU observes | Frame pacing; CPU waits for N-in-flight frames to complete |
| **VkSemaphore** | GPU queue ↔ GPU queue | One queue signals → another queue waits | Present semaphore; async compute handoff |
| **VkEvent** | Within a queue or CPU → GPU | GPU command signals → later GPU command waits | Split barrier within a command buffer; fine-grained pipelining |

**Fences** are the coarsest tool: a fence signaled by `vkQueueSubmit2` can be waited on by `vkWaitForFences` on the CPU. They carry no memory ordering by themselves — they only indicate that the GPU reached the end of a submit batch. Any memory ordering needed before the CPU reads results must be expressed through barriers inside the submitted command buffer.

**Semaphores** are queue-to-queue synchronisation objects. A semaphore signal operation on one queue establishes a happens-before relationship with the semaphore wait operation on another queue (or the same queue). Binary semaphores operate in strict 1:1 pairs: signal then wait, with the wait consuming the signal. Timeline semaphores extend this to a monotonically increasing counter that multiple waiters can observe.

**Events** are split barriers within a single queue. `vkCmdSetEvent2` inserts the producer-side synchronisation scope into the command stream; `vkCmdWaitEvents2` inserts the consumer-side scope. The GPU can execute commands between the two, potentially overlapping work on different shader units. Events cannot cross queues [Source](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#vkCmdWaitEvents2): *"vkCmdWaitEvents2 must not be used to wait on event signal operations occurring on other queues."*

---

## Fences in Depth

A `VkFence` starts in either unsignaled or signaled state. It is signaled by the GPU at the end of a queue submission and polled or waited on by the CPU.

```c
// vk_fence_example.c — fence creation and frame-pacing pattern

// Create a fence pre-signaled so the first frame loop iteration doesn't block.
VkFenceCreateInfo fenceCI = {
    .sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO,
    .flags = VK_FENCE_CREATE_SIGNALED_BIT   // Pre-signaled for first iteration
};
VkFence inFlightFence;
vkCreateFence(device, &fenceCI, NULL, &inFlightFence);
```

`VK_FENCE_CREATE_SIGNALED_BIT` is essential for the N-buffering frame loop: on frame 0, the fence for slot 0 must appear already complete so `vkWaitForFences` does not block before any GPU work has been submitted [Source](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/03_Drawing/03_Frames_in_flight.html).

### The N-Buffering Frame Loop

The canonical pattern for double- or triple-buffering maintains one fence per in-flight slot:

```c
// vk_frame_loop.c — double-buffered frame pacing

const int MAX_FRAMES_IN_FLIGHT = 2;
VkSemaphore acquireSemaphores[MAX_FRAMES_IN_FLIGHT];
VkSemaphore renderSemaphores[MAX_FRAMES_IN_FLIGHT];
VkFence     inFlightFences[MAX_FRAMES_IN_FLIGHT];
uint32_t    frameIndex = 0;

// --- Per frame ---
// 1. CPU waits until slot frameIndex is free on the GPU.
vkWaitForFences(device, 1, &inFlightFences[frameIndex],
                VK_TRUE /*waitAll*/, UINT64_MAX);

// 2. Reset before reuse — a fence cannot be reset while signaled and in-use.
vkResetFences(device, 1, &inFlightFences[frameIndex]);

// 3. Acquire swapchain image (may block if no image is available).
uint32_t imageIndex;
vkAcquireNextImageKHR(device, swapchain, UINT64_MAX,
                      acquireSemaphores[frameIndex],
                      VK_NULL_HANDLE, &imageIndex);

// 4. Record command buffer, then submit.
VkCommandBufferSubmitInfo cmdInfo = {
    .sType         = VK_STRUCTURE_TYPE_COMMAND_BUFFER_SUBMIT_INFO,
    .commandBuffer = commandBuffers[frameIndex]
};
VkSemaphoreSubmitInfo waitSI = {
    .sType     = VK_STRUCTURE_TYPE_SEMAPHORE_SUBMIT_INFO,
    .semaphore = acquireSemaphores[frameIndex],
    .stageMask = VK_PIPELINE_STAGE_2_COLOR_ATTACHMENT_OUTPUT_BIT
};
VkSemaphoreSubmitInfo signalSI = {
    .sType     = VK_STRUCTURE_TYPE_SEMAPHORE_SUBMIT_INFO,
    .semaphore = renderSemaphores[frameIndex],
    .stageMask = VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT
};
VkSubmitInfo2 submitInfo = {
    .sType                    = VK_STRUCTURE_TYPE_SUBMIT_INFO_2,
    .waitSemaphoreInfoCount   = 1,
    .pWaitSemaphoreInfos      = &waitSI,
    .commandBufferInfoCount   = 1,
    .pCommandBufferInfos      = &cmdInfo,
    .signalSemaphoreInfoCount = 1,
    .pSignalSemaphoreInfos    = &signalSI
};
// inFlightFences[frameIndex] is signaled when all commands complete.
vkQueueSubmit2(graphicsQueue, 1, &submitInfo, inFlightFences[frameIndex]);

// 5. Present; wait on renderSemaphores so present doesn't start before render.
VkPresentInfoKHR presentInfo = {
    .sType              = VK_STRUCTURE_TYPE_PRESENT_INFO_KHR,
    .waitSemaphoreCount = 1,
    .pWaitSemaphores    = &renderSemaphores[frameIndex],
    .swapchainCount     = 1,
    .pSwapchains        = &swapchain,
    .pImageIndices      = &imageIndex
};
vkQueuePresentKHR(presentQueue, &presentInfo);

frameIndex = (frameIndex + 1) % MAX_FRAMES_IN_FLIGHT;
```

`vkWaitForFences` with `waitAll = VK_TRUE` blocks the calling thread until all listed fences are signaled. With `waitAll = VK_FALSE` it returns as soon as any one fence is signaled. The `timeout` parameter is in nanoseconds; `UINT64_MAX` means indefinite wait. A timeout of 0 polls without blocking.

Fences must be reset with `vkResetFences` before they can be reused. Resetting a fence that is not signaled, or resetting it while it is referenced by a pending GPU submit, is undefined behaviour.

---

## Binary Semaphores

Binary semaphores express queue-to-queue ordering within or across queue families. A semaphore has two states: unsignaled and signaled. A signal operation transitions it to signaled; a wait operation transitions it back to unsignaled and establishes the ordering relationship.

The critical constraint is 1:1 signal/wait pairing. From the Vulkan specification [Source](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#synchronization-semaphores-signaling):

> *"Binary semaphore waits and signals should thus occur in discrete 1:1 pairs."*

And on exclusivity:

> *"There must be no other queue waiting on the same binary semaphore when the operation executes."*

The primary use of binary semaphores in a Vulkan renderer is the swapchain acquire/render/present chain:

- `vkAcquireNextImageKHR` signals a semaphore when the swapchain image is available for rendering.
- The render submission waits on that semaphore at `VK_PIPELINE_STAGE_2_COLOR_ATTACHMENT_OUTPUT_BIT` — the stage that first writes to the image.
- The render submission signals a separate semaphore when all rendering completes.
- `vkQueuePresentKHR` waits on that render-complete semaphore before the presentation engine samples the image.

The choice of `VK_PIPELINE_STAGE_2_COLOR_ATTACHMENT_OUTPUT_BIT` as the wait stage is significant: the GPU can execute all earlier pipeline stages (vertex processing, geometry shading, early depth tests) on other draws while blocked on the acquire semaphore, because those stages do not access the swapchain image. Choosing `VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT` as the wait stage instead would stall the entire pipeline unnecessarily.

Binary semaphores **cannot** be used with timeline semaphore operations (`vkWaitSemaphores`, `vkSignalSemaphore`), and timeline semaphores **cannot** be used directly with `vkAcquireNextImageKHR` or in `VkPresentInfoKHR.waitSemaphores` — the WSI interface requires binary semaphores, which is what forces the binary-to-timeline bridging pattern discussed in [Timeline Semaphores](#timeline-semaphores).

A common mistake: if `vkAcquireNextImageKHR` returns `VK_SUBOPTIMAL_KHR` or `VK_ERROR_OUT_OF_DATE_KHR`, the semaphore it was given may or may not have been signaled (the spec allows it to be signaled even on error). Applications that skip the submit but still pass the acquire semaphore to a future wait will deadlock or trigger a validation error. The safe pattern is to handle errors by recreating the swapchain and signaling or replacing the semaphore.

---

## Timeline Semaphores

Timeline semaphores, promoted to Vulkan 1.2 core from `VK_KHR_timeline_semaphore` [Source](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/appendices/VK_KHR_timeline_semaphore.adoc), replace the fence+binary-semaphore combination in many scenarios with a single monotonically increasing counter.

A timeline semaphore has an integer value. GPU and CPU operations can signal it to a new value (which must be strictly greater than the current value) and wait until it reaches or exceeds a target value. Multiple waiters can observe the same value; no "consumption" occurs. The GPU can wait on a timeline semaphore value that has not yet been signaled, blocking until it is — even if the signal comes from a CPU `vkSignalSemaphore` call.

### Creating a Timeline Semaphore

```c
// vk_timeline_semaphore.c — creation

VkSemaphoreTypeCreateInfo typeCI = {
    .sType         = VK_STRUCTURE_TYPE_SEMAPHORE_TYPE_CREATE_INFO,
    .semaphoreType = VK_SEMAPHORE_TYPE_TIMELINE,
    .initialValue  = 0          // Counter starts at 0
};
VkSemaphoreCreateInfo semCI = {
    .sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO,
    .pNext = &typeCI
};
VkSemaphore timelineSem;
vkCreateSemaphore(device, &semCI, NULL, &timelineSem);
```

### GPU Signal and CPU Wait

```c
// vk_timeline_semaphore.c — GPU signal at frame N, CPU wait replacing vkWaitForFences

uint64_t frameValue = currentFrame;  // Monotonically increasing per frame

// Signal from GPU: semaphore reaches frameValue when all commands complete.
VkSemaphoreSubmitInfo gpuSignal = {
    .sType     = VK_STRUCTURE_TYPE_SEMAPHORE_SUBMIT_INFO,
    .semaphore = timelineSem,
    .value     = frameValue,
    .stageMask = VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT
};
VkSubmitInfo2 submitInfo = {
    .sType                    = VK_STRUCTURE_TYPE_SUBMIT_INFO_2,
    .signalSemaphoreInfoCount = 1,
    .pSignalSemaphoreInfos    = &gpuSignal,
    // ... command buffer, wait semaphores ...
};
vkQueueSubmit2(graphicsQueue, 1, &submitInfo, VK_NULL_HANDLE);

// CPU wait: block until frameValue - MAX_FRAMES_IN_FLIGHT has been signaled.
// This replaces vkWaitForFences for the corresponding slot.
uint64_t waitValue = frameValue >= MAX_FRAMES_IN_FLIGHT
                   ? frameValue - MAX_FRAMES_IN_FLIGHT
                   : 0;
VkSemaphoreWaitInfo waitInfo = {
    .sType          = VK_STRUCTURE_TYPE_SEMAPHORE_WAIT_INFO,
    .semaphoreCount = 1,
    .pSemaphores    = &timelineSem,
    .pValues        = &waitValue
};
vkWaitSemaphores(device, &waitInfo, UINT64_MAX);
```

### CPU Signal Unblocking GPU

```c
// vk_timeline_semaphore.c — CPU signals a value; GPU was waiting on it

VkSemaphoreSignalInfo sigInfo = {
    .sType     = VK_STRUCTURE_TYPE_SEMAPHORE_SIGNAL_INFO,
    .semaphore = timelineSem,
    .value     = triggerValue   // Must be > current counter value
};
vkSignalSemaphore(device, &sigInfo);
```

Non-blocking query of current value: `vkGetSemaphoreCounterValue(device, semaphore, &value)` — useful for adaptive scheduling decisions on the CPU without blocking.

### Constraints on Timeline Semaphores

The spec imposes two constraints that differ from binary semaphores:

1. **Strict monotonicity per batch:** from the spec [Source](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#synchronization-semaphores-signaling): *"it is invalid to signal the same timeline semaphore twice within a single batch."* All signal operations in a single `VkSubmitInfo2` are unordered with respect to each other, so two signals in the same batch would race. Use separate `vkQueueSubmit2` calls for different timeline values.

2. **Host signal ordering:** if two threads call `vkSignalSemaphore`, the call with the lower value must return before the call with the higher value is made. Signal values must be strictly increasing over time.

The Khronos timeline semaphore blog [Source](https://www.khronos.org/blog/vulkan-timeline-semaphores) describes the primary use case: replacing separate `VkFence` and `VkSemaphore` objects for multi-frame-in-flight with a single timeline counter, reducing object count and eliminating `vkResetFences`.

---

## Pipeline Barriers

Pipeline barriers are the mechanism for GPU-to-GPU memory ordering within a queue. `vkCmdPipelineBarrier2` (sync2, promoted to Vulkan 1.3 core from `VK_KHR_synchronization2`) [Source](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/appendices/VK_KHR_synchronization2.adoc) takes a `VkDependencyInfo` that aggregates all barrier types:

```c
// vk_pipeline_barrier.c — vkCmdPipelineBarrier2 with VkDependencyInfo

VkImageMemoryBarrier2 imgBarrier = { ... };  // See sections below
VkBufferMemoryBarrier2 bufBarrier = { ... };
VkMemoryBarrier2 memBarrier = { ... };

VkDependencyInfo depInfo = {
    .sType                    = VK_STRUCTURE_TYPE_DEPENDENCY_INFO,
    .dependencyFlags          = 0,
    .memoryBarrierCount       = 1,
    .pMemoryBarriers          = &memBarrier,
    .bufferMemoryBarrierCount = 1,
    .pBufferMemoryBarriers    = &bufBarrier,
    .imageMemoryBarrierCount  = 1,
    .pImageMemoryBarriers     = &imgBarrier,
};
vkCmdPipelineBarrier2(commandBuffer, &depInfo);
```

### Stage and Access Masks

The sync2 API uses 64-bit `VkPipelineStageFlags2` and `VkAccessFlags2`, removing the awkward splitting of `VK_PIPELINE_STAGE_ALL_GRAPHICS_BIT` required in the original 32-bit API:

- `srcStageMask`: stages at which the **producing** operations execute (the stages whose output we need to make available).
- `srcAccessMask`: the access types written by those producing stages (the caches to flush).
- `dstStageMask`: stages at which the **consuming** operations will execute (the stages that must not run until data is visible).
- `dstAccessMask`: the access types read by those consuming stages (the caches to invalidate).

A concrete shadow map example:

```c
// vk_shadow_barrier.c — depth attachment write → fragment shader sampled read

VkImageMemoryBarrier2 shadowBarrier = {
    .sType         = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER_2,
    // Producer: depth/stencil attachment writes in early and late fragment tests
    .srcStageMask  = VK_PIPELINE_STAGE_2_EARLY_FRAGMENT_TESTS_BIT |
                     VK_PIPELINE_STAGE_2_LATE_FRAGMENT_TESTS_BIT,
    .srcAccessMask = VK_ACCESS_2_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT,
    // Consumer: fragment shader samples the shadow map as a texture
    .dstStageMask  = VK_PIPELINE_STAGE_2_FRAGMENT_SHADER_BIT,
    .dstAccessMask = VK_ACCESS_2_SHADER_SAMPLED_READ_BIT,
    // Layout transition combined with the barrier
    .oldLayout = VK_IMAGE_LAYOUT_ATTACHMENT_OPTIMAL,
    .newLayout = VK_IMAGE_LAYOUT_READ_ONLY_OPTIMAL,
    .srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED,
    .dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED,
    .image = shadowMap,
    .subresourceRange = {
        .aspectMask     = VK_IMAGE_ASPECT_DEPTH_BIT,
        .baseMipLevel   = 0, .levelCount = 1,
        .baseArrayLayer = 0, .layerCount = 1
    }
};
```

### TOP_OF_PIPE and BOTTOM_OF_PIPE Deprecation

The original Vulkan 1.0 API included `VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT` and `VK_PIPELINE_STAGE_BOTTOM_OF_PIPE_BIT` as convenience stages. Their semantics were confusing because they depend on context (src vs dst). The sync2 spec [Source](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/appendices/VK_KHR_synchronization2.adoc) deprecates them with explicit replacements:

| Old | Context | sync2 Replacement |
|-----|---------|-------------------|
| `TOP_OF_PIPE` | as `srcStageMask` (no-op sync) | `VK_PIPELINE_STAGE_2_NONE` |
| `TOP_OF_PIPE` | as `dstStageMask` (wait for all prior) | `VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT` |
| `BOTTOM_OF_PIPE` | as `srcStageMask` (flush all) | `VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT` |
| `BOTTOM_OF_PIPE` | as `dstStageMask` (no-op sync) | `VK_PIPELINE_STAGE_2_NONE` |

`VK_ACCESS_2_NONE` is used wherever an access mask is required but no memory dependency is needed — for example, a pure execution barrier where `srcAccessMask = VK_ACCESS_2_NONE` and `dstAccessMask = VK_ACCESS_2_NONE` express a WAR ordering without any cache operations.

`VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT` is a heavy hammer: it covers all pipeline stages including those not relevant to the resource. Prefer specific stage flags wherever possible to allow the driver to overlap unrelated work.

---

## Image Layout Transitions

Vulkan images exist in layouts that describe how their texels are arranged in memory. Different operations require different layouts: the framebuffer engine may use a tiled compression layout for render targets, while the sampler expects a specific layout for texture reads, and the presentation engine expects `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`. Transitioning between layouts is not merely a metadata change; on many GPU architectures it involves decompressing or recompressing tile memory, so the driver needs explicit notification to insert the right commands.

Image layout transitions are specified inside `VkImageMemoryBarrier2` (or `VkImageMemoryBarrier` in the original API) via `oldLayout` and `newLayout`. The transition happens **inside** the memory dependency ordering: after the availability operations (flush) but before the visibility operations (invalidation) [Source](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#synchronization-image-layout-transitions):

> *"when a layout transition is specified in a memory dependency, it happens-after the availability operations in the memory dependency, and happens-before the visibility operations."*

The full lifecycle of a typical swapchain image:

```
VK_IMAGE_LAYOUT_UNDEFINED
         │  (transition at first use, oldLayout=UNDEFINED discards content)
         ▼
VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL
         │  (transition after render pass)
         ▼
VK_IMAGE_LAYOUT_PRESENT_SRC_KHR
         │  (transition at next frame, after acquire)
         ▼
VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL
```

### VK_IMAGE_LAYOUT_UNDEFINED

Setting `oldLayout = VK_IMAGE_LAYOUT_UNDEFINED` explicitly discards the image contents [Source](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#synchronization-image-layout-transitions):

> *"If the old layout is VK_IMAGE_LAYOUT_UNDEFINED, the contents of that range may be discarded."*

This is intentional for resources that will be fully overwritten (such as a color attachment that begins with a clear operation), and tile-based GPU architectures exploit it to avoid loading tile memory from off-chip DRAM at the start of a render pass, reducing memory bandwidth.

### VK_IMAGE_LAYOUT_GENERAL

`VK_IMAGE_LAYOUT_GENERAL` is a valid layout for any image usage, but provides no layout optimisation guarantees. It is useful when an image is used simultaneously for multiple purposes (read and write in a compute shader, for example), but performance will be worse than using the optimal layout for each access type. Prefer `VK_IMAGE_LAYOUT_READ_ONLY_OPTIMAL` (for both depth sampling and color sampling) and `VK_IMAGE_LAYOUT_ATTACHMENT_OPTIMAL` (for render targets) introduced in sync2.

### No-Op Transition

In the sync2 API, if `oldLayout == newLayout`, the layout transition is a no-op (contents are preserved). This allows using an `VkImageMemoryBarrier2` purely for its memory barrier semantics without any layout change.

---

## Render Pass Implicit Synchronisation

Render passes provide implicit synchronisation through their attachment load/store operations and subpass dependencies. The render pass represents a coherent block of GPU work; the driver can implement it as a scheduling unit, potentially keeping intermediate results in tile memory across subpasses without any round-trip to main DRAM.

### Attachment Load and Store Operations

`VK_ATTACHMENT_LOAD_OP_CLEAR` and `VK_ATTACHMENT_LOAD_OP_LOAD` imply the GPU must resolve any outstanding writes to the attachment before the render pass reads it. `VK_ATTACHMENT_STORE_OP_STORE` means the rendered data must be flushed to memory so it can be read by subsequent commands. `VK_ATTACHMENT_STORE_OP_DONT_CARE` tells the driver that the contents do not need to persist, allowing tile memory to be discarded — critical for depth/stencil attachments on tile-based renderers.

### VkSubpassDependency2 and VK_SUBPASS_EXTERNAL

Subpass dependencies express execution and memory ordering between subpasses, or between a subpass and commands outside the render pass:

```c
// vk_subpass_dependency.c — subpass 0 color output → subpass 1 input attachment read
// Using sync2 VkMemoryBarrier2 chained in pNext for 64-bit stage/access masks

VkMemoryBarrier2 memBarrier2 = {
    .sType         = VK_STRUCTURE_TYPE_MEMORY_BARRIER_2,
    .srcStageMask  = VK_PIPELINE_STAGE_2_COLOR_ATTACHMENT_OUTPUT_BIT,
    .srcAccessMask = VK_ACCESS_2_COLOR_ATTACHMENT_WRITE_BIT,
    .dstStageMask  = VK_PIPELINE_STAGE_2_FRAGMENT_SHADER_BIT,
    .dstAccessMask = VK_ACCESS_2_INPUT_ATTACHMENT_READ_BIT
};
VkSubpassDependency2 dep = {
    .sType       = VK_STRUCTURE_TYPE_SUBPASS_DEPENDENCY_2,
    .pNext       = &memBarrier2,   // Overrides 32-bit masks below when present
    .srcSubpass  = 0,
    .dstSubpass  = 1,
    // When pNext contains VkMemoryBarrier2, the 32-bit fields are ignored:
    .srcStageMask = 0, .dstStageMask = 0,
    .srcAccessMask = 0, .dstAccessMask = 0,
    .dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT
};
```

`VK_SUBPASS_EXTERNAL` is a pseudo-subpass index meaning "all commands outside this render pass":
- `srcSubpass = VK_SUBPASS_EXTERNAL, dstSubpass = 0`: ordering all commands before `vkCmdBeginRenderPass` relative to the first subpass.
- `srcSubpass = N, dstSubpass = VK_SUBPASS_EXTERNAL`: ordering the last subpass relative to commands after `vkCmdEndRenderPass`.

`VK_DEPENDENCY_BY_REGION_BIT` tells the driver that the dependency is only between framebuffer regions: fragment output for region X only depends on fragment input from region X, not on the entire attachment. Tile-based GPUs can exploit this to keep data in tile memory across subpasses without flushing to main memory at all.

Within a single subpass, there are no ordering guarantees between draw calls for the same attachment except those provided by `VkSubpassDependency2` with `srcSubpass == dstSubpass` (a self-dependency). Self-dependencies allow feedback loops such as reading back results of previous draws within the same subpass via input attachments, with appropriate synchronisation.

---

## Events: Split Barriers

A pipeline barrier is a single command that inserts both the producer-side and consumer-side synchronisation at the same point in the command buffer. An event is a split barrier: `vkCmdSetEvent2` inserts only the producer-side scope; `vkCmdWaitEvents2` inserted later inserts the consumer-side scope.

Between the two commands, the GPU can execute unrelated work. If the shadow map pass completes its depth writes in 5ms and the lighting pass's fragment shader only runs 10ms later, an event allows the 5ms gap to be filled with compute work or geometry processing for other draws. A full barrier at the shadow-to-lighting transition point would stall the pipeline at that exact point.

From the sync2 spec on split-barrier efficiency [Source](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#vkCmdSetEvent2):

> *"The extra information provided by vkCmdSetEvent2 compared to vkCmdSetEvent allows implementations to more efficiently schedule the operations required to satisfy the requested dependencies. With vkCmdSetEvent, the full dependency information is not known until vkCmdWaitEvents is recorded, forcing implementations to insert the required operations at that point and not before."*

```c
// vk_events.c — split barrier: shadow pass done → main lighting pass samples

// After shadow map render commands, on the same queue:
VkImageMemoryBarrier2 setEventBarrier = {
    .sType         = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER_2,
    .srcStageMask  = VK_PIPELINE_STAGE_2_LATE_FRAGMENT_TESTS_BIT,
    .srcAccessMask = VK_ACCESS_2_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT,
    .dstStageMask  = VK_PIPELINE_STAGE_2_FRAGMENT_SHADER_BIT,
    .dstAccessMask = VK_ACCESS_2_SHADER_SAMPLED_READ_BIT,
    .oldLayout = VK_IMAGE_LAYOUT_ATTACHMENT_OPTIMAL,
    .newLayout = VK_IMAGE_LAYOUT_READ_ONLY_OPTIMAL,
    .image = shadowMap, .subresourceRange = { ... }
};
VkDependencyInfo setDepInfo = {
    .sType = VK_STRUCTURE_TYPE_DEPENDENCY_INFO,
    .imageMemoryBarrierCount = 1,
    .pImageMemoryBarriers = &setEventBarrier
};
vkCmdSetEvent2(cmdBuf, shadowCompleteEvent, &setDepInfo);

// Unrelated draw calls here may execute while shadow flush proceeds...
vkCmdDraw(cmdBuf, ...);   // Geometry not using the shadow map

// Before the lighting pass needs the shadow map:
VkDependencyInfo waitDepInfo = { /* must match setDepInfo exactly */ };
vkCmdWaitEvents2(cmdBuf, 1, &shadowCompleteEvent, &waitDepInfo);
// Now the lighting pass can sample the shadow map safely.
```

The **exact-match constraint**: the `VkDependencyInfo` passed to `vkCmdWaitEvents2` must match exactly the one passed to `vkCmdSetEvent2` [Source](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#vkCmdWaitEvents2):

> *"future vkCmdWaitEvents2 commands rely on all values of each element in pDependencyInfo matching exactly with those used to signal the corresponding event"*

This means events cannot be used with dynamically composed dependency descriptions. For that pattern, use pipeline barriers instead.

A CPU can also signal an event with `vkSetEvent` (without Cmd), which unblocks a `vkCmdWaitEvents2` on the GPU — useful for signaling the GPU to proceed after CPU-side resource upload completes. Events cannot replace semaphores across queue boundaries.

---

## Queue Families and Ownership Transfer

Vulkan queues are grouped into queue families, where each family supports a set of operations (graphics, compute, transfer, video, etc.) and may run on physically separate hardware units. Images and buffers created with `VK_SHARING_MODE_EXCLUSIVE` (the default) can be accessed by only one queue family at a time. To move a resource from one family to another, a two-step ownership transfer is required.

### EXCLUSIVE vs CONCURRENT Sharing Mode

`VK_SHARING_MODE_CONCURRENT` allows simultaneous access from multiple queue families, but requires the driver to assume all participating families may access the resource at any time, which typically means using uncached or more conservative memory layouts. On discrete GPU hardware with a separate transfer engine, this is usually more expensive than explicit ownership transfer.

`VK_SHARING_MODE_EXCLUSIVE` with explicit release/acquire pairs allows the driver to apply queue-family-specific optimisations (tile compression for the graphics family, uncached writes for the transfer family) and transitions between them only at ownership transfer time.

### Release and Acquire Barriers

The ownership transfer is asymmetric in its synchronisation scopes. This is one of the most non-obvious aspects of the Vulkan synchronisation model [Source](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#synchronization-queue-transfers):

- **Release barrier** (submitted to the source queue): only the **first** synchronisation scope is active. The barrier flushes and makes available the writes from the source queue. The second scope (dstStage/dstAccess) does not apply on the source queue.
- **Acquire barrier** (submitted to the destination queue): only the **second** synchronisation scope is active. The barrier makes the data visible to the destination queue. The first scope (srcStage/srcAccess) does not apply on the destination queue.

The cross-queue ordering that connects release to acquire is provided by a **semaphore**: the source queue signals a semaphore after the release command, and the destination queue waits on that semaphore before the acquire command. The acquire barrier itself does no hazard avoidance; it only makes data visible after the semaphore guarantees the release has completed.

```c
// vk_ownership_transfer.c — async texture upload: transfer → graphics

// === Transfer queue: upload data, then release ownership ===

// 1. Transition to TRANSFER_DST_OPTIMAL (image starts undefined, discard contents)
VkImageMemoryBarrier2 toTransfer = {
    .sType         = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER_2,
    .srcStageMask  = VK_PIPELINE_STAGE_2_NONE,    // No prior writes
    .srcAccessMask = VK_ACCESS_2_NONE,
    .dstStageMask  = VK_PIPELINE_STAGE_2_COPY_BIT,
    .dstAccessMask = VK_ACCESS_2_TRANSFER_WRITE_BIT,
    .oldLayout = VK_IMAGE_LAYOUT_UNDEFINED,
    .newLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
    .srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED,
    .dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED,
    .image = texture, .subresourceRange = { ... }
};
// ... vkCmdCopyBufferToImage(cmdBuf, stagingBuffer, texture, ...) ...

// 2. Release ownership from transfer → graphics
VkImageMemoryBarrier2 releaseBarrier = {
    .sType         = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER_2,
    .srcStageMask  = VK_PIPELINE_STAGE_2_COPY_BIT,
    .srcAccessMask = VK_ACCESS_2_TRANSFER_WRITE_BIT,
    .dstStageMask  = VK_PIPELINE_STAGE_2_NONE,    // Second scope inactive on release
    .dstAccessMask = VK_ACCESS_2_NONE,
    .oldLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
    .newLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL,
    .srcQueueFamilyIndex = transferQueueFamily,
    .dstQueueFamilyIndex = graphicsQueueFamily,
    .image = texture, .subresourceRange = { ... }
};
// Submit to transfer queue, signal uploadSemaphore

// === Graphics queue: acquire ownership, then sample texture ===
// Submit must wait on uploadSemaphore before this command executes.

VkImageMemoryBarrier2 acquireBarrier = {
    .sType         = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER_2,
    .srcStageMask  = VK_PIPELINE_STAGE_2_NONE,    // First scope inactive on acquire
    .srcAccessMask = VK_ACCESS_2_NONE,
    .dstStageMask  = VK_PIPELINE_STAGE_2_FRAGMENT_SHADER_BIT,
    .dstAccessMask = VK_ACCESS_2_SHADER_SAMPLED_READ_BIT,
    .oldLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
    .newLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL,
    .srcQueueFamilyIndex = transferQueueFamily,
    .dstQueueFamilyIndex = graphicsQueueFamily,
    .image = texture, .subresourceRange = { ... }
};
// After this barrier, the fragment shader can sample the texture.
```

Skipping the acquire barrier is undefined behaviour: the graphics queue may see stale or corrupt data. Syncval will report a missing barrier in this case. Note that the layout transition (TRANSFER_DST_OPTIMAL → SHADER_READ_ONLY_OPTIMAL) is specified identically in both the release and acquire barriers; the driver applies it exactly once as part of the ownership transfer.

### Async Compute Queue Patterns

For typical async compute postprocessing (such as DLSS-style upscaling or temporal antialiasing running in a compute queue alongside rendering in a graphics queue), the pattern is:

1. Graphics queue renders scene to intermediate image and signals a timeline semaphore at value N.
2. Compute queue waits on timeline semaphore value N, then acquires the image and runs compute.
3. Compute queue signals timeline semaphore at value N+1.
4. Graphics queue (or present queue) waits on value N+1 before presenting or further compositing.

The timeline semaphore replaces the fence+binary-semaphore pair required before Vulkan 1.2, reducing object count and removing the need for `vkResetFences`.

---

## Explicit Sync with the OS and Compositor

On Linux, the Vulkan runtime must interoperate with kernel DRM synchronisation objects and the Wayland compositor's explicit sync protocol.

### DRM Syncobj

The kernel `drm_syncobj` [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_syncobj.c) is a reference-counted object that wraps a `dma_fence*`. It was introduced in kernel 4.13 (binary) [Source](https://www.phoronix.com/news/DRM-Sync-Objects-4.13) and extended to timeline semantics in kernel 5.2 (commit `48197bc564c7`). The corresponding capability flags are `DRM_CAP_SYNCOBJ` (0x13) and `DRM_CAP_SYNCOBJ_TIMELINE` (0x14), queryable via `drmGetCap`.

Key UAPI ioctls from `include/uapi/drm/drm.h` [Source](https://github.com/torvalds/linux/blob/master/include/uapi/drm/drm.h):

| IOCTL | Purpose |
|-------|---------|
| `DRM_IOCTL_SYNCOBJ_CREATE` | Allocate a new syncobj (optionally pre-signaled with `DRM_SYNCOBJ_CREATE_SIGNALED`) |
| `DRM_IOCTL_SYNCOBJ_DESTROY` | Free the syncobj |
| `DRM_IOCTL_SYNCOBJ_HANDLE_TO_FD` | Export as fd; `DRM_SYNCOBJ_HANDLE_TO_FD_FLAGS_EXPORT_SYNC_FILE` exports as a `sync_file` |
| `DRM_IOCTL_SYNCOBJ_FD_TO_HANDLE` | Import from fd; `DRM_SYNCOBJ_FD_TO_HANDLE_FLAGS_IMPORT_SYNC_FILE` imports a `sync_file` |
| `DRM_IOCTL_SYNCOBJ_WAIT` | Block until signaled; flags: `WAIT_ALL`, `WAIT_FOR_SUBMIT`, `WAIT_AVAILABLE` |
| `DRM_IOCTL_SYNCOBJ_TIMELINE_WAIT` | Timeline-aware wait with per-syncobj `uint64_t points[]` |
| `DRM_IOCTL_SYNCOBJ_SIGNAL` | Signal from userspace |
| `DRM_IOCTL_SYNCOBJ_QUERY` | Query current timeline value |
| `DRM_IOCTL_SYNCOBJ_TRANSFER` | Transfer a point from one timeline syncobj to another |

From `drm_syncobj.c` line 101 [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_syncobj.c):

> *"The use of binary syncobj is supported through the timeline set of ioctl() by using a point value of 0, this will reproduce the behavior of the binary set of ioctl()."*

In Mesa's Vulkan runtime, `VkFence` and `VkSemaphore` are ultimately backed by `drm_syncobj` handles through the `vk_sync_type` abstraction in `src/vulkan/runtime/vk_drm_syncobj.h` [Source](https://gitlab.freedesktop.org/mesa/mesa). Drivers set `vk_physical_device.supported_sync_types` to include the `vk_drm_syncobj` type, and the common `vk_queue.c` translates `vkQueueSubmit2` semaphore operations into the corresponding DRM ioctls.

### VK_KHR_external_semaphore_fd

`VK_KHR_external_semaphore_fd` [Source](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#VK_KHR_external_semaphore_fd) provides two handle types for exporting and importing semaphores as file descriptors:

- **`VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_OPAQUE_FD_BIT`**: Reference transference. The fd represents a `drm_syncobj` handle. Supports both permanent and temporary import. Supports timeline semaphores. The fd must be closed by the importer.
- **`VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_SYNC_FD_BIT`**: Copy transference. The fd represents a Linux `sync_file` wrapping a single `dma_fence`. Temporary import only. Binary semaphores only — `sync_file` does not carry a timeline. The fence is consumed on wait.

```c
// vk_external_semaphore.c — exporting a semaphore as OPAQUE_FD for cross-process use

VkSemaphoreGetFdInfoKHR getFdInfo = {
    .sType      = VK_STRUCTURE_TYPE_SEMAPHORE_GET_FD_INFO_KHR,
    .semaphore  = renderCompleteSemaphore,
    .handleType = VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_OPAQUE_FD_BIT
};
int semaphoreFd;
vkGetSemaphoreFdKHR(device, &getFdInfo, &semaphoreFd);
// Pass semaphoreFd to another process via Unix socket or similar IPC.

// Importing in the other process:
VkImportSemaphoreFdInfoKHR importInfo = {
    .sType      = VK_STRUCTURE_TYPE_IMPORT_SEMAPHORE_FD_INFO_KHR,
    .semaphore  = importedSemaphore,
    .handleType = VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_OPAQUE_FD_BIT,
    .fd         = receivedFd,
    .flags      = 0   // VK_SEMAPHORE_IMPORT_TEMPORARY_BIT for temporary import
};
vkImportSemaphoreFdKHR(device, &importInfo);
```

### wp_linux_drm_syncobj_v1 Wayland Protocol

The `wp_linux_drm_syncobj_v1` protocol [Source](https://wayland.app/protocols/linux-drm-syncobj-v1) (in `wayland-protocols/staging/linux-drm-syncobj/`) replaces implicit fence-based synchronisation between a Vulkan Wayland client and the compositor with explicit timeline syncobj points.

The protocol provides three interfaces:
- `wp_linux_drm_syncobj_manager_v1`: global factory; `import_timeline(fd)` imports a `drm_syncobj` by fd; `get_surface(wl_surface)` binds explicit sync to a surface.
- `wp_linux_drm_syncobj_timeline_v1`: represents an imported timeline syncobj.
- `wp_linux_drm_syncobj_surface_v1`: per-surface object with:
  - `set_acquire_point(timeline, point_hi, point_lo)`: the compositor waits on this timeline value before sampling the buffer.
  - `set_release_point(timeline, point_hi, point_lo)`: the compositor signals this timeline value when it finishes with the buffer, allowing the client to reuse it.

The 64-bit point is split into two `uint32_t` fields because the Wayland wire protocol does not have a native 64-bit integer type.

Mesa 24.1 implemented Vulkan WSI explicit sync for both Wayland and X11 using this protocol. The `wp_linux_drm_syncobj_surface_v1` is stored on the `VkSurface` in the WSI code in `src/vulkan/wsi/`. The acquire point corresponds to the Vulkan render-complete semaphore signal value; the release point corresponds to the next acquire semaphore that will be signaled when the compositor returns the buffer.

---

## Synchronisation Validation

Synchronisation errors are among the hardest GPU bugs to diagnose because they are non-deterministic, vendor-specific, and often manifest as pixel corruption or hangs rather than API errors. The primary tool is the Vulkan Validation Layer's synchronisation mode.

### Enabling SyncVal

SyncVal is part of `VK_LAYER_KHRONOS_validation` and is disabled by default because it has significant CPU overhead (it tracks every command's access to every resource). Enable it using one of three mechanisms:

**Environment variable** (simplest for development):
```bash
export VK_VALIDATION_VALIDATE_SYNC=1
# For verbose output:
export VK_LAYER_ENABLES=VK_VALIDATION_FEATURE_ENABLE_SYNCHRONIZATION_VALIDATION_EXT
```

**Programmatic enable** (for automated test infrastructure):
```c
// vk_syncval.c — programmatic SyncVal enable via VkLayerSettingEXT

const VkBool32 enable = VK_TRUE;
VkLayerSettingEXT setting = {
    .pLayerName   = "VK_LAYER_KHRONOS_validation",
    .pSettingName = "validate_sync",
    .type         = VK_LAYER_SETTING_TYPE_BOOL32_EXT,
    .valueCount   = 1,
    .pValues      = &enable
};
VkLayerSettingsCreateInfoEXT layerSettingsCI = {
    .sType        = VK_STRUCTURE_TYPE_LAYER_SETTINGS_CREATE_INFO_EXT,
    .settingCount = 1,
    .pSettings    = &setting
};
VkInstanceCreateInfo instanceCI = {
    .sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO,
    .pNext = &layerSettingsCI,
    // ...
};
// Then enable "VK_LAYER_KHRONOS_validation" in ppEnabledLayerNames.
```

**vkconfig GUI**: select the Synchronisation Preset in the Vulkan Configurator tool.

Pair with a debug messenger to receive errors as callbacks:
```c
// vk_debug_messenger.c — catch syncval errors in-process

VkDebugUtilsMessengerCreateInfoEXT messengerCI = {
    .sType           = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT,
    .messageSeverity = VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT |
                       VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT,
    .messageType     = VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT,
    .pfnUserCallback = debugCallback
};
vkCreateDebugUtilsMessengerEXT(instance, &messengerCI, NULL, &messenger);
// Set a breakpoint inside debugCallback to catch hazards at record time.
```

### Hazard Types

SyncVal identifies five hazard types [Source](https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/main/docs/syncval_usage.md):

| Hazard | Meaning | Fix |
|--------|---------|-----|
| **RAW** (Read-After-Write) | A read uses the result of a prior write without waiting for it | Add barrier: `srcAccess` = write type, `dstAccess` = read type |
| **WAR** (Write-After-Read) | A write starts before a prior read has completed | Add execution dependency (stage masks only; no access masks needed) |
| **WAW** (Write-After-Write) | A write races with or precedes a prior write | Add barrier: `srcAccess` = write type, `dstAccess` = write type |
| **WRW** (Write-Racing-Write) | Writes from concurrent subpasses or queues race | Add subpass dependency or semaphore |
| **RRW** (Read-Racing-Write) | Reads and writes from concurrent operations race | Add subpass dependency, barrier, or semaphore |

A typical syncval error message:
```
SYNC-HAZARD-READ-AFTER-WRITE: vkCmdDraw: Hazard READ_AFTER_WRITE
for image VkImage 0x... in VkCommandBuffer 0x...
Access info (write): stage = VK_PIPELINE_STAGE_2_LATE_FRAGMENT_TESTS_BIT,
access = VK_ACCESS_2_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT in
prior vkCmdDraw at renderpass #0, subpass #0, op #42.
Access info (read): stage = VK_PIPELINE_STAGE_2_FRAGMENT_SHADER_BIT,
access = VK_ACCESS_2_SHADER_SAMPLED_READ_BIT.
```

The `[Extra properties]` block in the error message contains structured fields (`hazard_type`, `prior_access`, `write_barriers`, `command`, `prior_command`) that remain stable across validation layer versions and can be used for programmatic suppression in test frameworks.

### Known SyncVal Limitations

SyncVal does not validate [Source](https://vulkan.lunarg.com/doc/view/latest/windows/synchronization_usage.html):
- Descriptor-level resource tracking (use GPU-AV for per-descriptor validation)
- Memory aliasing scenarios
- Indirect buffer contents
- Host memory access ordering between CPU and GPU

### RenderDoc Sync Analysis

RenderDoc captures the full frame and annotates each command with its resource state. Image layout transitions are visible in the resource inspector, showing which layout each image was in at capture time versus what a given draw expects. This complements syncval (which catches errors at record time) with post-hoc analysis of complex frame graphs.

---

## Real-World Patterns

### Shadow Map Render → Sample in Main Pass

The shadow pass writes to a depth image as a `DEPTH_STENCIL_ATTACHMENT`. The lighting pass reads it as a sampled texture. Between these two render passes, exactly one barrier is required:

```c
// vk_shadow_sample_barrier.c — depth write → sampled read transition

// After vkCmdEndRenderPass for the shadow pass:
VkImageMemoryBarrier2 shadowToSample = {
    .sType         = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER_2,
    .srcStageMask  = VK_PIPELINE_STAGE_2_EARLY_FRAGMENT_TESTS_BIT |
                     VK_PIPELINE_STAGE_2_LATE_FRAGMENT_TESTS_BIT,
    .srcAccessMask = VK_ACCESS_2_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT,
    .dstStageMask  = VK_PIPELINE_STAGE_2_FRAGMENT_SHADER_BIT,
    .dstAccessMask = VK_ACCESS_2_SHADER_SAMPLED_READ_BIT,
    .oldLayout     = VK_IMAGE_LAYOUT_ATTACHMENT_OPTIMAL,
    .newLayout     = VK_IMAGE_LAYOUT_READ_ONLY_OPTIMAL,
    .srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED,
    .dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED,
    .image = shadowDepthImage,
    .subresourceRange = {
        .aspectMask = VK_IMAGE_ASPECT_DEPTH_BIT,
        .baseMipLevel = 0, .levelCount = 1,
        .baseArrayLayer = 0, .layerCount = 1
    }
};
VkDependencyInfo dep = {
    .sType = VK_STRUCTURE_TYPE_DEPENDENCY_INFO,
    .imageMemoryBarrierCount = 1,
    .pImageMemoryBarriers = &shadowToSample
};
vkCmdPipelineBarrier2(cmdBuf, &dep);
// Now begin the lighting render pass; shadow map is accessible as VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER.
```

### Compute Postprocess → Present

After a compute shader writes to a swapchain image (e.g., for a full-screen FXAA or tonemapping pass), a barrier transitions the image to `PRESENT_SRC_KHR`:

```c
// vk_compute_to_present.c — compute write → present

VkImageMemoryBarrier2 computeToPresent = {
    .sType         = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER_2,
    .srcStageMask  = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
    .srcAccessMask = VK_ACCESS_2_SHADER_WRITE_BIT,
    .dstStageMask  = VK_PIPELINE_STAGE_2_NONE,   // Present engine is outside Vulkan pipeline
    .dstAccessMask = VK_ACCESS_2_NONE,
    .oldLayout = VK_IMAGE_LAYOUT_GENERAL,
    .newLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR,
    .srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED,
    .dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED,
    .image = swapchainImages[imageIndex], .subresourceRange = { ... }
};
// The render submission signals renderSemaphore;
// vkQueuePresentKHR waits on renderSemaphore before the present engine reads the image.
```

The present engine's read is not a Vulkan pipeline stage, so `dstStageMask = VK_PIPELINE_STAGE_2_NONE` with `dstAccessMask = VK_ACCESS_2_NONE` is correct: the ordering is provided by the present semaphore wait in `VkPresentInfoKHR`, not by the barrier's dst scope.

### Async Texture Upload (Transfer Queue + Ownership Transfer)

See the full code example in [Queue Families and Ownership Transfer](#queue-families-and-ownership-transfer). The key ordering is:

1. Transfer queue records: `UNDEFINED → TRANSFER_DST_OPTIMAL` (barrier), `vkCmdCopyBufferToImage`, release barrier (`TRANSFER_DST_OPTIMAL → SHADER_READ_ONLY_OPTIMAL`, transfer family → graphics family).
2. Transfer queue submission signals `uploadSemaphore`.
3. Graphics queue submission waits on `uploadSemaphore` at `VK_PIPELINE_STAGE_2_FRAGMENT_SHADER_BIT`.
4. Graphics queue records: acquire barrier (`TRANSFER_DST_OPTIMAL → SHADER_READ_ONLY_OPTIMAL`, transfer family → graphics family).
5. Fragment shader samples texture.

The staging buffer holding the texel data must remain valid until the transfer queue submission completes. A fence on the transfer queue submission signals the CPU to recycle the staging buffer.

### Multi-Frame-in-Flight with Timeline Semaphores

Replacing `N` separate fences and `2N` binary semaphores with two timeline semaphores (one for acquire, one for render):

```c
// vk_timeline_frame_pacing.c — N-buffering with timeline semaphores only

// frameTimeline signals at value N (frame N render complete).
// wsiBridge[i] is a binary semaphore per swapchain slot,
//   bridging vkAcquireNextImageKHR → frameTimeline.
// This is necessary because vkAcquireNextImageKHR requires a binary semaphore.

uint64_t currentFrame = 0;

// --- Per frame ---
// Wait for the frame MAX_FRAMES_IN_FLIGHT ago to have completed.
uint64_t waitFrame = (currentFrame >= MAX_FRAMES_IN_FLIGHT)
                   ? currentFrame - MAX_FRAMES_IN_FLIGHT : 0;
VkSemaphoreWaitInfo waitInfo = {
    .sType = VK_STRUCTURE_TYPE_SEMAPHORE_WAIT_INFO,
    .semaphoreCount = 1,
    .pSemaphores = &frameTimeline,
    .pValues = &waitFrame
};
vkWaitSemaphores(device, &waitInfo, UINT64_MAX);

uint32_t imageIndex;
vkAcquireNextImageKHR(device, swapchain, UINT64_MAX,
                      wsiBridge[currentFrame % MAX_FRAMES_IN_FLIGHT],
                      VK_NULL_HANDLE, &imageIndex);

// Submit: wait on binary wsiBridge semaphore and signal frameTimeline at currentFrame+1.
uint64_t signalValue = currentFrame + 1;
VkSemaphoreSubmitInfo waitBinary = {
    .sType     = VK_STRUCTURE_TYPE_SEMAPHORE_SUBMIT_INFO,
    .semaphore = wsiBridge[currentFrame % MAX_FRAMES_IN_FLIGHT],
    .stageMask = VK_PIPELINE_STAGE_2_COLOR_ATTACHMENT_OUTPUT_BIT
};
VkSemaphoreSubmitInfo signalTimeline = {
    .sType     = VK_STRUCTURE_TYPE_SEMAPHORE_SUBMIT_INFO,
    .semaphore = frameTimeline,
    .value     = signalValue,
    .stageMask = VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT
};
// Also signal a binary semaphore for vkQueuePresentKHR (binary required by WSI).
VkSemaphoreSubmitInfo signalPresentBinary = {
    .sType     = VK_STRUCTURE_TYPE_SEMAPHORE_SUBMIT_INFO,
    .semaphore = presentSemaphores[imageIndex],
    .stageMask = VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT
};
// ... assemble VkSubmitInfo2 with both signalTimeline and signalPresentBinary ...

currentFrame++;
```

The need to bridge between binary semaphores (required by WSI) and timeline semaphores is an acknowledged limitation. A proposal to allow timeline semaphores in WSI exists in the Vulkan ecosystem but has not yet been standardised as of 2026.

---

## Integrations

**[Chapter 16 — Mesa's Vulkan Common Infrastructure]:** The `VkFence`, `VkSemaphore`, and queue submission pipeline discussed here are implemented in Mesa's common Vulkan runtime at `src/vulkan/runtime/vk_fence.c`, `vk_semaphore.c`, and `vk_queue.c`. The `vk_sync_type` abstraction table in `src/vulkan/runtime/vk_sync.h` defines the operations (`wait`, `signal`, `import_opaque_fd`, `export_opaque_fd`) that individual GPU drivers fill in. The `vk_drm_syncobj` sync type in `src/vulkan/runtime/vk_drm_syncobj.h` is the standard implementation backed by kernel `drm_syncobj` ioctls, used by most Mesa drivers. For drivers that lack kernel timeline support, `src/vulkan/runtime/vk_sync_timeline.c` provides a software-emulated timeline. Barrier and layout transition logic lives above this layer in the application's command buffer recording.

**[Chapter 24 — Vulkan and EGL for Application Developers]:** The swapchain present path described in the EGL chapter uses binary `VkSemaphore` for `vkAcquireNextImageKHR` and `VkPresentInfoKHR.waitSemaphores`. The `VK_PIPELINE_STAGE_2_COLOR_ATTACHMENT_OUTPUT_BIT` wait stage in the render submission is the connection point where the render queue waits for image availability. Timeline semaphores cannot be used directly in this path; the binary-to-timeline bridging pattern in this chapter covers how to integrate both in one frame loop.

**[Chapter 20 — Wayland Protocol Fundamentals] / [Chapter 46 — Evolving Wayland]:** The `wp_linux_drm_syncobj_v1` Wayland protocol bridges Vulkan WSI semaphores to the kernel `drm_syncobj` objects seen by the compositor. Mesa 24.1's WSI code attaches a `wp_linux_drm_syncobj_surface_v1` to each `VkSurface`. The acquire point set on the surface corresponds to the timeline value that the compositor must wait for before sampling the buffer (i.e., the Vulkan render-complete timeline signal); the release point is the value the compositor signals when the buffer is no longer in use. This replaces the implicit synchronisation that older Wayland clients relied on through `dma_resv` reservation objects.

**[Chapter 133 — Vulkan Compute Queues]:** Async compute is the primary scenario requiring multi-queue synchronisation. The async compute queue lives in a different queue family from the graphics queue on many GPU architectures. Resource ownership transfer (release on graphics, acquire on compute, or vice versa) is required for every image or buffer exchanged between the queues. Timeline semaphores are the modern synchronisation mechanism for the graphics→compute→present pipeline: a single timeline expresses "graphics frame N complete → compute postprocess can start → present can proceed" without separate fence objects. The exact-stage selection for the compute wait semaphore (which pipeline stage in the compute queue first reads the graphics output) directly determines how much the compute work can overlap with unrelated graphics work.

**[Chapter 1 — DRM Architecture] / [Chapter 4 — GPU Memory Management]:** `drm_syncobj` is a DRM core object defined in `drivers/gpu/drm/drm_syncobj.c`, not driver-specific. It wraps a `dma_fence*` from `drivers/dma-buf/dma-fence.c`. When a GPU driver submits work to the hardware, it creates a `dma_fence` and plants it in the associated `drm_syncobj`; the fence is signaled in an interrupt handler when the GPU completes the work. The same `dma_fence` infrastructure underlies `dma_resv` (reservation objects on GEM buffers), which provides OpenGL's implicit synchronisation. Vulkan's `VK_SHARING_MODE_EXCLUSIVE` with explicit barriers is the application-level alternative to the automatic `dma_resv` tracking the kernel performs for OpenGL. Using explicit sync avoids the kernel's conservative assumption that any GEM buffer might be accessed by any engine at any time.

**[Chapter 30 — Debugging and Profiling the Graphics Stack]:** RenderDoc complements syncval by capturing post-execution frame state: it shows which image layout each resource was in at capture time, which barriers were active per command, and can highlight transitions that appear wrong (such as an image still in `UNDEFINED` layout when sampled). syncval catches errors at command recording time (before any GPU work runs); RenderDoc catches the consequences of errors that syncval missed or that only manifest under specific timing conditions. Together they cover the two failure modes of synchronisation bugs: structural errors (missing barriers, wrong access masks) and timing-dependent races.

---

*Sources: [Vulkan Specification — Synchronisation](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#synchronization); [VK_KHR_synchronization2 appendix](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/appendices/VK_KHR_synchronization2.adoc); [VK_KHR_timeline_semaphore appendix](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/appendices/VK_KHR_timeline_semaphore.adoc); [Khronos Vulkan Timeline Semaphores blog](https://www.khronos.org/blog/vulkan-timeline-semaphores); [Frames in Flight tutorial](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/03_Drawing/03_Frames_in_flight.html); [syncval usage documentation](https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/main/docs/syncval_usage.md); [LunarG syncval guide](https://vulkan.lunarg.com/doc/view/latest/windows/synchronization_usage.html); [linux/drm_syncobj.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_syncobj.c); [linux/include/uapi/drm/drm.h](https://github.com/torvalds/linux/blob/master/include/uapi/drm/drm.h); [linux/include/linux/dma-fence.h](https://github.com/torvalds/linux/blob/master/include/linux/dma-fence.h); [wp_linux_drm_syncobj_v1 protocol](https://wayland.app/protocols/linux-drm-syncobj-v1); [DRM Sync Objects kernel 4.13](https://www.phoronix.com/news/DRM-Sync-Objects-4.13); [Mesa Vulkan driver guide](https://www.collabora.com/news-and-blog/blog/2022/03/23/how-to-write-vulkan-driver-in-2022/).*
