# Chapter 200: Vulkan Memory Allocation and Resource Management

> **Part**: Part VII-A — GPU APIs
> **Audience**: Graphics application developers writing Vulkan renderers and engines who need to move past `vkAllocateMemory`-per-object toy code; systems and driver developers who want to see how RADV, ANV, and NVK turn a `VkMemoryAllocateInfo` into a GEM object and a GPU virtual-address binding.
> **Status**: First draft — 2026-08-21

Vulkan hands memory management to the application by design. There is no driver-side allocator standing between a `VkBuffer` and the DRAM or VRAM it lives in — the application decides which of a handful of heaps to allocate from, how large each `VkDeviceMemory` object is, and how resources are suballocated within it. This chapter covers that whole surface: querying what memory the device actually has (`VkPhysicalDeviceMemoryProperties`, `VK_EXT_memory_budget`), the allocation and binding calls themselves, why virtually every real Vulkan codebase reaches for a suballocator such as AMD's Vulkan Memory Allocator (VMA) rather than calling `vkAllocateMemory` per resource, and the newer mechanisms — buffer device addresses, sparse residency, host-image copy, explicit aliasing — that change what "memory management" means in a GPU-driven or bindless renderer. It closes by tracing an allocation down through RADV, ANV, and NVK to the GEM object and BAR-window constraints underneath.

---

## Table of Contents

- [1. `VkPhysicalDeviceMemoryProperties`: Types and Heaps](#1-vkphysicaldevicememoryproperties-types-and-heaps)
  - [1.1 Memory Property Flags](#11-memory-property-flags)
  - [1.2 `vkGetPhysicalDeviceMemoryProperties2` and the pNext Chain](#12-vkgetphysicaldevicememoryproperties2-and-the-pnext-chain)
  - [1.3 `VK_EXT_memory_budget`: Heap Budget and Usage](#13-vk_ext_memory_budget-heap-budget-and-usage)
- [2. `VkDeviceMemory` Allocation](#2-vkdevicememory-allocation)
  - [2.1 `vkAllocateMemory` and `VkMemoryAllocateInfo`](#21-vkallocatememory-and-vkmemoryallocateinfo)
  - [2.2 `VkMemoryDedicatedAllocateInfo`](#22-vkmemorydedicatedallocateinfo)
  - [2.3 `maxMemoryAllocationCount` and the Suballocation Requirement](#23-maxmemoryallocationcount-and-the-suballocation-requirement)
- [3. Memory Binding](#3-memory-binding)
  - [3.1 `vkBindBufferMemory2` / `vkBindImageMemory2`](#31-vkbindbuffermemory2--vkbindimagememory2)
  - [3.2 Alignment from `VkMemoryRequirements2`](#32-alignment-from-vkmemoryrequirements2)
  - [3.3 `VkBindImagePlaneMemoryInfo` for Disjoint YCbCr Images](#33-vkbindimageplanememoryinfo-for-disjoint-ycbcr-images)
- [4. VMA (Vulkan Memory Allocator) in Depth](#4-vma-vulkan-memory-allocator-in-depth)
  - [4.1 Blocks, Pools, and the TLSF Suballocator](#41-blocks-pools-and-the-tlsf-suballocator)
  - [4.2 `VmaAllocationCreateInfo` and Allocation Strategies](#42-vmaallocationcreateinfo-and-allocation-strategies)
  - [4.3 `vmaCreateBuffer` / `vmaCreateImage` and `VmaAllocationInfo`](#43-vmacreatebuffer--vmacreateimage-and-vmaallocationinfo)
  - [4.4 The Linear Pool Algorithm](#44-the-linear-pool-algorithm)
  - [4.5 Defragmentation](#45-defragmentation)
- [5. Buffer Device Address](#5-buffer-device-address)
  - [5.1 `VK_KHR_buffer_device_address` and `vkGetBufferDeviceAddress`](#51-vk_khr_buffer_device_address-and-vkgetbufferdeviceaddress)
  - [5.2 `PhysicalStorageBuffer` in Shaders](#52-physicalstoragebuffer-in-shaders)
  - [5.3 Capture/Replay Addresses](#53-capturereplay-addresses)
  - [5.4 Intersection with GPU-Driven Rendering and Descriptor-Less Binding](#54-intersection-with-gpu-driven-rendering-and-descriptor-less-binding)
- [6. Sparse Resources](#6-sparse-resources)
  - [6.1 `VkSparseImageMemoryRequirements` and Mip-Tail Packing](#61-vksparseimagememoryrequirements-and-mip-tail-packing)
  - [6.2 `vkQueueBindSparse` Timeline Semantics](#62-vkqueuebindsparse-timeline-semantics)
  - [6.3 Virtual Texture Streaming on RADV/ANV](#63-virtual-texture-streaming-on-radvanv)
- [7. Host-Image Copy](#7-host-image-copy)
  - [7.1 `VK_EXT_host_image_copy` and Promotion to Vulkan 1.4](#71-vk_ext_host_image_copy-and-promotion-to-vulkan-14)
  - [7.2 `vkCopyMemoryToImageEXT` / `vkCopyImageToMemoryEXT`](#72-vkcopymemorytoimageext--vkcopyimagetomemoryext)
  - [7.3 What It Does and Does Not Eliminate](#73-what-it-does-and-does-not-eliminate)
- [8. Memory Aliasing](#8-memory-aliasing)
  - [8.1 `VK_IMAGE_CREATE_ALIAS_BIT` and Overlapping Bindings](#81-vk_image_create_alias_bit-and-overlapping-bindings)
  - [8.2 Transient Attachment Aliasing](#82-transient-attachment-aliasing)
  - [8.3 Hazard and Synchronization Requirements](#83-hazard-and-synchronization-requirements)
- [9. Residency and Device Loss](#9-residency-and-device-loss)
  - [9.1 `VK_ERROR_OUT_OF_DEVICE_MEMORY` vs `VK_ERROR_OUT_OF_HOST_MEMORY`](#91-vk_error_out_of_device_memory-vs-vk_error_out_of_host_memory)
  - [9.2 Heap Eviction Under Pressure](#92-heap-eviction-under-pressure)
  - [9.3 `VK_EXT_device_fault` and Post-Mortem Diagnosis](#93-vk_ext_device_fault-and-post-mortem-diagnosis)
- [10. RADV/ANV/NVK Allocator Internals](#10-radvanvnvk-allocator-internals)
  - [10.1 RADV: `radv_alloc_memory` and the amdgpu Winsys](#101-radv-radv_alloc_memory-and-the-amdgpu-winsys)
  - [10.2 ANV: `anv_device_alloc_bo` and the KMD Backend Abstraction](#102-anv-anv_device_alloc_bo-and-the-kmd-backend-abstraction)
  - [10.3 NVK: `nvk_AllocateMemory` and `nvkmd`](#103-nvk-nvk_allocatememory-and-nvkmd)
  - [10.4 BAR Window Constraints on Discrete GPUs](#104-bar-window-constraints-on-discrete-gpus)
- [Integrations](#integrations)
- [References](#references)

---

## 1. `VkPhysicalDeviceMemoryProperties`: Types and Heaps

Every Vulkan device exposes its memory as a small, fixed set of **heaps** — physical memory pools, each with a size and a bit indicating whether it is device-local — and **types**, which are combinations of a heap index plus a set of property flags describing how the CPU and GPU can access memory allocated from it. `vkGetPhysicalDeviceMemoryProperties` fills a `VkPhysicalDeviceMemoryProperties` struct with `memoryHeapCount` heaps (at most `VK_MAX_MEMORY_HEAPS`, 16) and `memoryTypeCount` types (at most `VK_MAX_MEMORY_TYPES`, 32) [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VkPhysicalDeviceMemoryProperties.html). On a discrete GPU this typically yields three to five types spread across two or three heaps: a device-local VRAM heap, a host-visible system-memory heap, and — on GPUs with resizable BAR — a device-local *and* host-visible heap that is a CPU-mapped window into VRAM (§10.4).

### 1.1 Memory Property Flags

The `VkMemoryPropertyFlagBits` that matter for allocation decisions:

- `VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT` — the memory is on the GPU's own bus (VRAM on discrete parts, main DRAM on integrated ones); accessing it from the CPU, if possible at all, is slow.
- `VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT` — `vkMapMemory` is legal on allocations of this type.
- `VK_MEMORY_PROPERTY_HOST_COHERENT_BIT` — writes through the mapped pointer become visible to the device without an explicit `vkFlushMappedMemoryRanges` call (and vice versa for reads), simplifying but not eliminating synchronization at the command-buffer level.
- `VK_MEMORY_PROPERTY_HOST_CACHED_BIT` — the mapping goes through the CPU cache hierarchy, which is a large win for host *reads* (readback buffers) and a potential loss for host *writes* if the memory is not also coherent.
- `VK_MEMORY_PROPERTY_LAZILY_ALLOCATED_BIT` — used with `VK_IMAGE_USAGE_TRANSIENT_ATTACHMENT_BIT`; on tile-based architectures the implementation may never back the allocation with physical pages at all, since transient attachments (an MSAA resolve source, a depth buffer that is written and read only within a render pass and never stored) can live entirely in on-chip tile memory (§8.2).

These flags compose: a type can be `DEVICE_LOCAL | HOST_VISIBLE | HOST_COHERENT` simultaneously — the resizable-BAR case above — or `HOST_VISIBLE | HOST_COHERENT | HOST_CACHED` without `DEVICE_LOCAL`, a good choice for readback staging buffers.

### 1.2 `vkGetPhysicalDeviceMemoryProperties2` and the pNext Chain

`vkGetPhysicalDeviceMemoryProperties2` wraps the same `VkPhysicalDeviceMemoryProperties` struct inside `VkPhysicalDeviceMemoryProperties2` and adds a `pNext` chain, which is how extension structs — most importantly `VkPhysicalDeviceMemoryBudgetPropertiesEXT` — attach to the query [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/vkGetPhysicalDeviceMemoryProperties2.html). Applications targeting Vulkan 1.1+ should call the `2` variant unconditionally even when they do not need any chained struct, both for API consistency and because some memory-related extensions only define behavior in terms of the `2` entry point.

### 1.3 `VK_EXT_memory_budget`: Heap Budget and Usage

`VK_EXT_memory_budget` (core-optional in Vulkan 1.4 promotion terms is not applicable — it remains an EXT extension, widely supported) adds `VkPhysicalDeviceMemoryBudgetPropertiesEXT`, with two arrays indexed by heap: `heapBudget` and `heapUsage`. `heapBudget[i]` is "a guideline for how much total memory from each heap the current process can use at any given time, before allocations may start failing or causing performance degradation," and `heapUsage[i]` is the implementation's estimate of what the current process has already committed from that heap [Source](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/appendices/VK_EXT_memory_budget.adoc). Both numbers are explicitly *not* guarantees: the spec text warns that budget "can depend on a variety of factors including operating system and system load" and may shift due to other processes' GPU memory pressure, entirely outside the querying application's control.

The intended usage pattern is periodic re-querying — once per frame or once every few frames is typical — followed by a policy decision when `heapUsage` approaches `heapBudget`: stop streaming in new mip levels, evict LRU textures, or fall back to a lower-resolution asset tier. `VK_EXT_memory_budget` is commonly paired with `VK_EXT_memory_priority`, which lets an application hint per-allocation eviction priority to the driver so that, when the OS or another process forces something out of VRAM, the driver evicts the application's own least-important allocations first rather than an arbitrary one [Source](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/appendices/VK_EXT_memory_budget.adoc).

RADV's implementation of the budget query is not a pass-through of a single kernel counter — it reconstructs a budget from `amdgpu` heap-usage ioctls, apportioning free space between the visible-VRAM and GTT (GPU-accessible system memory) heaps in a fixed 2:1 ratio when the two heaps are drawn from a shared underlying pool, and separately when resizable BAR gives the GPU an independent VRAM-invisible heap [Source, gitlab.freedesktop.org/mesa/mesa @ e7b3fdae](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/amd/vulkan/radv_physical_device.c#L3290-3327). This is a concrete illustration of why the spec refuses to promise budget stability: the number an application receives is itself a heuristic computed on top of kernel accounting, not a hardware register.

---

## 2. `VkDeviceMemory` Allocation

### 2.1 `vkAllocateMemory` and `VkMemoryAllocateInfo`

`vkAllocateMemory(device, &allocateInfo, pAllocator, &memory)` takes a `VkMemoryAllocateInfo` naming `allocationSize` and `memoryTypeIndex` — the index into `VkPhysicalDeviceMemoryProperties::memoryTypes` chosen in §1 — and returns an opaque `VkDeviceMemory` handle representing a single contiguous device allocation [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/vkAllocateMemory.html). The `pNext` chain on `VkMemoryAllocateInfo` is where most of the interesting extension behavior in this chapter attaches: `VkMemoryDedicatedAllocateInfo` (§2.2), `VkMemoryAllocateFlagsInfo` for buffer-device-address allocations (§5), `VkMemoryOpaqueCaptureAddressAllocateInfo` for capture/replay (§5.3), `VkExportMemoryAllocateInfo` and `VkImportMemoryFdInfoKHR` for cross-process or cross-API sharing, and `VkMemoryPriorityAllocateInfoEXT` for the eviction hint mentioned above.

### 2.2 `VkMemoryDedicatedAllocateInfo`

By default a `VkDeviceMemory` allocation is meant to back *multiple* resources through suballocation — the whole reason §4 exists. `VkMemoryDedicatedAllocateInfo`, chained onto `VkMemoryAllocateInfo` with an `image` or `buffer` handle, instead ties the entire allocation to exactly one resource [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VkMemoryDedicatedAllocateInfo.html). This exists for cases where suballocation is impossible or actively harmful:

- **External memory export/import.** Sharing a `VkDeviceMemory` across process or API boundaries (a Vulkan-to-CUDA interop buffer, a Vulkan/OpenGL shared texture, a Wayland `linux-dmabuf` swapchain image) generally requires the whole allocation to correspond to one dma-buf/GEM object with one set of external semantics; suballocating a shared buffer would expose unrelated application data to the importer. The spec makes dedicated allocation mandatory in exactly this situation: certain `VkExportMemoryAllocateInfo::handleTypes` require `VkMemoryDedicatedAllocateInfo` to be present in the same `pNext` chain [Source](https://raw.githubusercontent.com/KhronosGroup/Vulkan-Docs/main/chapters/memory.adoc).
- **Implementation-defined layout requirements.** Some image formats or tiling modes on some hardware carry metadata (compression state, auxiliary surfaces) that the driver can only place correctly if it controls the whole backing allocation rather than a slice of a larger one.
- **Large, long-lived resources.** A single 4K render target or a multi-gigabyte compute buffer gains nothing from suballocation and only adds bookkeeping overhead; dedicating the allocation is both simpler and, for VMA-managed applications, the library's own default heuristic above a size threshold (§4.2).

### 2.3 `maxMemoryAllocationCount` and the Suballocation Requirement

`VkPhysicalDeviceLimits::maxMemoryAllocationCount` caps how many `VkDeviceMemory` objects can exist simultaneously on a device — every `vkAllocateMemory` call past that count returns `VK_ERROR_TOO_MANY_OBJECTS`. The spec is explicit that suballocation exists to work around this limit, not merely as an optimization: "the maximum number of valid memory allocations that can exist simultaneously within a `VkDevice` may be restricted by implementation- or platform-dependent limits... The `maxMemoryAllocationCount` feature describes the number of allocations that can exist simultaneously before encountering these internal limits" [Source](https://raw.githubusercontent.com/KhronosGroup/Vulkan-Docs/main/chapters/memory.adoc). Desktop drivers commonly report values in the low thousands (historically 4096 was a widespread minimum on Windows drivers reported through GPUInfo/vulkan.gpuinfo.org submissions, though the exact value is implementation-defined and applications must query it rather than assume it — Note: needs verification for any specific current RADV/ANV/NVK value, since this is queried per-device and changes across driver versions). A renderer that calls `vkAllocateMemory` once per `VkBuffer`/`VkImage` — the naive pattern every Vulkan tutorial demonstrates for pedagogical clarity — will exhaust this limit on any scene with more than a few thousand distinct resources, which is why §4's suballocator is not optional engineering polish but a load-bearing requirement for any non-trivial Vulkan application.

---

## 3. Memory Binding

Allocating a `VkDeviceMemory` object and creating a `VkBuffer` or `VkImage` are independent steps; **binding** is the third step that associates a range of an allocation with a resource, and only after a successful bind is the resource legal to use.

### 3.1 `vkBindBufferMemory2` / `vkBindImageMemory2`

The original `vkBindBufferMemory`/`vkBindImageMemory` entry points bind one resource at a time. `vkBindBufferMemory2` and `vkBindImageMemory2` (core since Vulkan 1.1, originally `VK_KHR_bind_memory2`) instead take an array of `VkBindBufferMemoryInfo` / `VkBindImageMemoryInfo` structs and bind all of them in a single call [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/vkBindBufferMemory2.html). Two reasons to prefer the batched form even for engines that could get away with single binds:

- **Driver-side batching.** A single call gives the driver visibility into the whole set of binds up front, which matters on drivers that must serialize GPU virtual-address-space updates (RADV and ANV both funnel binds through a kernel VM-bind path in modern configurations; batching reduces the number of separate kernel round-trips).
- **`pNext` extensibility per bind.** Each element of the array carries its own `pNext` chain, which is how `VkBindImagePlaneMemoryInfo` (§3.3) and `VkBindImageMemorySwapchainInfoKHR` attach — extension behavior that needs to vary per-resource within one batched call.

### 3.2 Alignment from `VkMemoryRequirements2`

Before binding, an application queries `vkGetBufferMemoryRequirements2` / `vkGetImageMemoryRequirements2`, which fill a `VkMemoryRequirements2` (itself extensible via `pNext`, e.g. for `VkMemoryDedicatedRequirements` telling the application whether the implementation *requires* a dedicated allocation for this specific resource). The embedded `VkMemoryRequirements::alignment` field gives the byte alignment the resource's binding offset must satisfy within the `VkDeviceMemory` — this is why a suballocator cannot simply pack allocations back-to-back and must round each requested offset up to the resource's own alignment, which varies by resource type, format, and tiling mode and is frequently larger than the natural size alignment (RADV, for instance, reports alignments up to the hardware page-table entry granularity for compressed or tiled images).

### 3.3 `VkBindImagePlaneMemoryInfo` for Disjoint YCbCr Images

Multi-planar formats (`VK_FORMAT_G8_B8_R8_3PLANE_420_UNORM` and similar YCbCr formats used for video decode output and camera capture, covered from the decode side in the video/streaming chapters) can be created with `VK_IMAGE_CREATE_DISJOINT_BIT`, meaning each plane is backed by an *independent* memory binding rather than all planes sharing one allocation — the natural representation when a plane originates from a separate DMA-BUF handed over by a V4L2 or VA-API decode pipeline. For a disjoint image, `vkBindImageMemory2` is called once per plane, with a `VkBindImagePlaneMemoryInfo` chained onto each `VkBindImageMemoryInfo` naming which `VkImageAspectFlagBits` plane (`VK_IMAGE_ASPECT_PLANE_0_BIT`, `_PLANE_1_BIT`, ...) that particular bind targets [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VkBindImagePlaneMemoryInfo.html). Correspondingly, `vkGetImageMemoryRequirements2` needs a `VkImagePlaneMemoryRequirementsInfo` in its own `pNext` chain to ask for per-plane requirements rather than the (meaningless, for a disjoint image) whole-image requirements. The spec is explicit that dedicated allocation and disjoint images do not mix: a `VkMemoryDedicatedAllocateInfo::image` reference is invalid if that image was created with `VK_IMAGE_CREATE_DISJOINT_BIT` set [Source](https://raw.githubusercontent.com/KhronosGroup/Vulkan-Docs/main/chapters/memory.adoc) — each plane gets its own (possibly dedicated) allocation instead.

---

## 4. VMA (Vulkan Memory Allocator) in Depth

Chapter 82 covers VMA's application-facing API surface in full — `VmaAllocatorCreateInfo`, the staging-upload pattern, custom pools, `VmaVirtualBlock`, statistics export — and readers who have not yet used VMA should start there. This section instead focuses on the allocation-strategy internals that determine *how* VMA places suballocations, which matters when diagnosing fragmentation or choosing a strategy flag deliberately.

### 4.1 Blocks, Pools, and the TLSF Suballocator

VMA never calls `vkAllocateMemory` once per resource. It groups allocations of compatible memory-type index into **blocks** — individual `VkDeviceMemory` objects, sized by default up to `VMA_DEFAULT_LARGE_HEAP_BLOCK_SIZE`, 256 MiB, for heaps larger than 1 GiB (overridable per-pool via `VmaPoolCreateInfo::blockSize`) [Source](https://raw.githubusercontent.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator/master/include/vk_mem_alloc.h) — and suballocates individual buffer/image bindings out of those blocks. `vmaCreateBuffer`/`vmaCreateImage` (§4.3) either find free space in an existing block or, when none fits, allocate a new block and place the suballocation at its start.

As of VMA 3.0.0, the placement algorithm within each block is a **Two-Level Segregated Fit (TLSF)** allocator, which replaced the library's earlier default algorithm; the same release removed the older `VMA_POOL_CREATE_BUDDY_ALGORITHM_BIT` buddy-system pool mode entirely [Source](https://raw.githubusercontent.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator/master/CHANGELOG.md). TLSF maintains a two-level index (a coarse free-list bucketed by size class, and within each bucket a finer second-level index) so that finding a free range that fits a given request is close to O(1) rather than requiring a linear scan of a free list, while still coalescing adjacent free ranges on release to control fragmentation — a materially better fragmentation/latency trade-off than either a first-fit linked list or a rigid power-of-two buddy scheme for the highly irregular allocation-size distributions a real Vulkan application produces (small uniform buffers alongside multi-hundred-megabyte textures in the same memory type).

### 4.2 `VmaAllocationCreateInfo` and Allocation Strategies

`VmaAllocationCreateInfo::flags` accepts one of a small set of mutually exclusive strategy bits that bias TLSF's search within a block, layered inside the broader `VmaAllocationCreateFlagBits` enum used for usage/mapping hints:

- `VMA_ALLOCATION_CREATE_STRATEGY_MIN_MEMORY_BIT` (alias: `..._BEST_FIT_BIT`) — "chooses smallest possible free range for the allocation to minimize memory usage and fragmentation, possibly at the expense of allocation time."
- `VMA_ALLOCATION_CREATE_STRATEGY_MIN_TIME_BIT` — "chooses first suitable free range for the allocation — not necessarily in terms of the smallest offset but the one that is easiest and fastest to find to minimize allocation time, possibly at the expense of allocation quality."
- `VMA_ALLOCATION_CREATE_STRATEGY_MIN_OFFSET_BIT` — "chooses always the lowest offset in available space. This is not the most efficient strategy but achieves highly packed data. Used internally by defragmentation, not recommended in typical usage."

[Source](https://raw.githubusercontent.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator/master/include/vk_mem_alloc.h)

For most engine code the default (unset, which behaves like `MIN_MEMORY`) is the right choice; `MIN_TIME` is worth reaching for explicitly in hot allocation paths — streaming systems that create and destroy many short-lived suballocations per frame — where allocation latency itself becomes measurable, at the cost of somewhat worse packing over the block's lifetime.

### 4.3 `vmaCreateBuffer` / `vmaCreateImage` and `VmaAllocationInfo`

`vmaCreateBuffer(allocator, &bufferCreateInfo, &allocCreateInfo, &buffer, &allocation, &allocationInfo)` collapses `vkCreateBuffer` + `vkGetBufferMemoryRequirements2` + block search/creation + `vkBindBufferMemory2` into one call (`vmaCreateImage` is the analogous wrapper for images). The `VmaAllocationInfo` output struct gives back exactly the information an application needs to reason about where its data physically lives: `memoryType`, the underlying `deviceMemory` handle (nullable, and subject to change if the allocation is later moved by defragmentation — §4.5), the suballocation's `offset` and `size` within that `VkDeviceMemory`, and `pMappedData` if the allocation is persistently mapped [Source](https://raw.githubusercontent.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator/master/include/vk_mem_alloc.h). Chapter 82 §2.3–2.4 walks through the full staging-upload pattern built on these calls; it is not repeated here.

### 4.4 The Linear Pool Algorithm

A custom `VmaPool` created with `VMA_POOL_CREATE_LINEAR_ALGORITHM_BIT` opts out of TLSF entirely in favor of a bump allocator: "always creates new allocations after last one and doesn't reuse space from allocations freed in between." Combined with `VMA_ALLOCATION_CREATE_UPPER_ADDRESS_BIT` for allocating from the opposite end, this single flag implements four different classic allocation patterns without additional bookkeeping — free-at-once, stack, double-stack, and ring buffer — depending on whether the application frees the whole pool at once, only ever frees from the top, allocates from both ends toward the middle, or continuously wraps [Source](https://raw.githubusercontent.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator/master/include/vk_mem_alloc.h). A per-frame upload ring buffer — the dominant use case for host-visible staging memory in a streaming renderer — maps directly onto the ring-buffer mode: allocate forward every frame, and because the pool never needs to search for free space (only ever appending), allocation cost is O(1) with none of TLSF's bookkeeping overhead, which is the right trade for a pool whose entire contents are recycled every N frames rather than held for arbitrary lifetimes.

### 4.5 Defragmentation

Suballocation inevitably fragments over an application's lifetime as resources of varying sizes are created and destroyed within shared blocks. VMA's defragmentation API — substantially reworked in the same 3.0.0 release that introduced TLSF [Source](https://raw.githubusercontent.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator/master/CHANGELOG.md) — runs as an explicit, application-driven pass rather than automatically in the background, because moving a live GPU resource requires the application's cooperation to update whatever CPU-side handles (descriptor sets, cached device addresses) point at it. The flow is: call `vmaBeginDefragmentation` to start a context, repeatedly call `vmaBeginDefragmentationPass`/`vmaEndDefragmentationPass` in a loop bounded by `VmaDefragmentationInfo::maxBytesPerPass` and `maxAllocationsPerPass`, and for each `VmaDefragmentationMove` the pass reports, the application performs the actual data copy (a `vkCmdCopyBuffer` / `vkCmdCopyBuffer2` recorded by the application, or a host `memcpy` for host-visible memory) before calling `vmaEndDefragmentationPass` to commit the move [Source](https://raw.githubusercontent.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator/master/include/vk_mem_alloc.h). Internally, VMA's own move-selection uses the `MIN_OFFSET` strategy from §4.2 to pack surviving allocations toward the low end of each block, after which empty blocks can be freed back to the driver. Engines that hold long-lived device addresses (§5) into VMA allocations must re-fetch those addresses after a defragmentation pass moves the underlying `VkDeviceMemory` binding, since a `VkDeviceAddress` is only valid until the buffer it was queried from is rebound.

---

## 5. Buffer Device Address

### 5.1 `VK_KHR_buffer_device_address` and `vkGetBufferDeviceAddress`

`VK_KHR_buffer_device_address`, core in Vulkan 1.2 (with the `PhysicalStorageBuffer` SPIR-V capability becoming a required feature once an implementation advertises Vulkan 1.3 support), adds "pointers to buffer memory in shaders." A buffer created with `VK_BUFFER_USAGE_SHADER_DEVICE_ADDRESS_BIT` can have its GPU virtual address queried on the host via `vkGetBufferDeviceAddress(device, &VkBufferDeviceAddressInfo{ .buffer = buf })`, which returns a raw 64-bit `VkDeviceAddress` [Source](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/appendices/VK_KHR_buffer_device_address.adoc). That address is a plain integer the application can store anywhere — in a uniform buffer, in a push constant, embedded as a field inside another buffer to build a linked or indexed structure entirely in GPU memory — and it remains valid for the buffer's lifetime as long as the buffer's binding does not change.

### 5.2 `PhysicalStorageBuffer` in Shaders

On the shader side, `PhysicalStorageBuffer` is a SPIR-V storage class (surfaced in GLSL via `GL_EXT_buffer_reference`) that lets a shader dereference a raw 64-bit address as if it were a pointer to a typed buffer block, without that block having been bound through a descriptor at all. This is the mechanism that makes GPU-resident, pointer-chasing data structures practical in Vulkan: a scene's mesh data, material tables, and BVH nodes can each be a plain buffer whose address is threaded through other buffers as an ordinary struct field, and shader code walks the resulting graph with `->`-style dereferences compiled down to loads at `PhysicalStorageBuffer` addresses rather than indirect descriptor lookups.

### 5.3 Capture/Replay Addresses

Because a `VkDeviceAddress` is chosen by the implementation at allocation time and is not guaranteed to be the same value on a second run, graphics-debugging and frame-capture tools (RenderDoc and similar) need a way to force a *specific* address to be reproduced on replay so that recorded GPU-side pointers inside captured buffers remain valid. `VK_BUFFER_CREATE_DEVICE_ADDRESS_CAPTURE_REPLAY_BIT` on buffer creation, together with `VkMemoryOpaqueCaptureAddressAllocateInfo`/`VkBufferOpaqueCaptureAddressCreateInfo` on the backing allocation, opts a resource into this behavior; `vkGetBufferOpaqueCaptureAddress` and `vkGetDeviceMemoryOpaqueCaptureAddress` retrieve the values a capture tool needs to record and later replay verbatim. The spec is explicit that "this mechanism is intended only to support capture/replay tools, and is not recommended for use in other applications" [Source](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/appendices/VK_KHR_buffer_device_address.adoc) — ordinary engine code should let the implementation choose addresses freely and simply re-query them each run, which is also why the VMA defragmentation caveat in §4.5 matters: a buffer device address is not stable across a rebind unless the application has specifically opted into capture/replay semantics for it.

### 5.4 Intersection with GPU-Driven Rendering and Descriptor-Less Binding

Buffer device address is one of the two pillars — alongside descriptor buffers and bindless descriptor indexing (Chapter 157) — that let a GPU-driven renderer (Chapter 154) avoid CPU-side descriptor-set churn entirely: an indirect-draw-generating compute shader can build per-draw parameter buffers containing raw `VkDeviceAddress` values pointing at per-object data, with the draw-executing shader dereferencing those addresses directly via `PhysicalStorageBuffer` rather than indexing into a bound descriptor array. Device Generated Commands (`VK_EXT_device_generated_commands`, covered in Chapter 154's indirect-draw material) leans on the same mechanism: a GPU-authored command stream can embed buffer addresses computed entirely on-device, with no CPU round-trip to allocate or bind a descriptor for each generated draw.

---

## 6. Sparse Resources

Sparse resources decouple a `VkBuffer`/`VkImage`'s *virtual* extent from its *physical* memory backing: the resource is created at full logical size, but individual regions are bound to (or left unbound from) actual `VkDeviceMemory` independently and can be rebound at any time via `vkQueueBindSparse`, rather than requiring the whole resource to be resident before use. This underlies virtual/megatexture streaming, where a texture far larger than any single allocation can host only its currently-visible mip levels and tiles.

### 6.1 `VkSparseImageMemoryRequirements` and Mip-Tail Packing

`vkGetImageSparseMemoryRequirements2` returns, per image aspect, a `VkSparseImageMemoryRequirements` describing the sparse block granularity (`VkSparseImageFormatProperties::imageGranularity`, the texel dimensions of one sparse-binding tile) plus the parameters of the **mip tail** — the run of the smallest mip levels, below `imageMipTailFirstLod`, that the implementation packs into one region (`imageMipTailSize` bytes at `imageMipTailOffset`) rather than exposing as individually bindable tiles, since below some size the per-tile bookkeeping overhead exceeds the memory saved by partial residency [Source](https://raw.githubusercontent.com/KhronosGroup/Vulkan-Docs/main/chapters/sparsemem.adoc). Two packing models exist, distinguished by `VK_SPARSE_IMAGE_FORMAT_SINGLE_MIPTAIL_BIT` in `VkSparseImageFormatProperties::flags`: with the bit set, every array layer of the image shares one mip-tail region; without it, each array layer gets its own mip tail at `imageMipTailOffset + arrayLayer * imageMipTailStride`. A megatexture streaming system needs to branch on this flag, since it changes whether unloading one array layer's mip tail is possible independently of the others.

### 6.2 `vkQueueBindSparse` Timeline Semantics

`vkQueueBindSparse` submits an array of `VkBindSparseInfo` batches, each naming opaque and/or image-subresource binds to attach or detach memory ranges, guarded by ordinary semaphore wait/signal lists (`VkTimelineSemaphoreSubmitInfo` in the `pNext` chain works here exactly as it does for `vkQueueSubmit`). The spec's ordering guarantee is deliberately weak: "batches begin execution in the order they appear... but may complete out of order," and bind operations do not themselves constitute a memory-access synchronization point — "no operation to `vkQueueBindSparse` causes any pipeline stage to access memory" [Source](https://raw.githubusercontent.com/KhronosGroup/Vulkan-Docs/main/chapters/sparsemem.adoc). Practically: binding a tile and then submitting a draw that samples it still requires the ordinary semaphore or timeline dependency between the bind batch and the consuming submission: a sparse bind is a virtual-address-space edit, not a barrier.

### 6.3 Virtual Texture Streaming on RADV/ANV

Both RADV and ANV expose the `sparseResidencyImage2D`/`sparseResidencyAliased` feature set on their respective discrete-GPU targets, and both implement sparse binding as an extension of the same GPU virtual-address-space machinery §10 describes for ordinary allocations — a sparse-bound image is, from the kernel's perspective, a page table whose entries are populated and depopulated by `vkQueueBindSparse` rather than fixed at creation time. A streaming engine typically layers its own residency manager on top: a background thread tracks a feedback signal (rendered UV footprint from a feedback-map pass, or simple camera-distance heuristics) to decide which tiles of a virtual texture atlas should be resident, issues `vkQueueBindSparse` to attach freshly-loaded tiles and detach evicted ones, and updates an indirection texture the shader samples to translate a virtual UV into a physical tile address — none of which is provided by the API itself; Vulkan's sparse-resource machinery is the binding substrate, not a streaming system.

---

## 7. Host-Image Copy

### 7.1 `VK_EXT_host_image_copy` and Promotion to Vulkan 1.4

`VK_EXT_host_image_copy` lets the CPU write image data directly into an implementation's chosen image layout without going through a GPU-visible staging buffer at all: "this extension allows applications to copy data between host memory and images on the host processor, without staging the data through a GPU-accessible buffer" [Source](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/appendices/VK_EXT_host_image_copy.adoc). The functionality was promoted into Vulkan 1.4 core (as an *optional* feature — an implementation can be Vulkan-1.4-conformant without supporting it, and applications must still query `VkPhysicalDeviceHostImageCopyFeatures`/`...Properties` before relying on it), with the original `EXT`-suffixed types and function names retained as aliases for source compatibility [Source](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_host_image_copy.html).

### 7.2 `vkCopyMemoryToImageEXT` / `vkCopyImageToMemoryEXT`

An image created with `VK_IMAGE_USAGE_HOST_TRANSFER_BIT_EXT` can be the target of `vkCopyMemoryToImageEXT` (upload) or the source of `vkCopyImageToMemoryEXT` (readback), both issued as plain host function calls rather than recorded into a command buffer — there is no queue submission, no fence to wait on, and no command-buffer recording overhead for the transfer itself. `vkTransitionImageLayoutEXT` performs the accompanying layout transition on the host side for images whose usage does not otherwise require a queue-recorded barrier. `VK_HOST_IMAGE_COPY_MEMCPY_BIT_EXT` on the copy region signals that the implementation's chosen host-visible layout is a byte-for-byte linear layout, allowing the application to skip its own row-pitch bookkeeping when it can be sure a `memcpy` is faithful.

### 7.3 What It Does and Does Not Eliminate

Host-image copy removes the staging-buffer *allocation* and the GPU-side *copy command* for one-shot asset upload — precisely the pattern of loading a texture once at level-load time and never touching it again, which previously required a temporary `VkBuffer`, a `vkCmdCopyBufferToImage`, and a fence/semaphore round trip purely to move bytes the CPU already had in hand. It does not eliminate the driver-side layout conversion the GPU would otherwise have performed in that copy command: if the image's optimal tiling layout differs from a linear byte layout (true for essentially all compressed or tiled formats on real hardware), the driver performs that conversion on the CPU during `vkCopyMemoryToImageEXT` instead, which can be markedly slower per byte than the equivalent GPU-side blit — host-image copy trades transfer-queue GPU time and synchronization complexity for CPU time, and is a net win specifically for upload patterns that are latency- or complexity-bound rather than throughput-bound (level-load-time texture population, editor tooling, small one-off uploads), not a blanket replacement for a streaming engine's steady-state GPU upload path.

---

## 8. Memory Aliasing

### 8.1 `VK_IMAGE_CREATE_ALIAS_BIT` and Overlapping Bindings

Multiple `VkImage` or `VkBuffer` objects can be bound to overlapping — or identical — ranges of the same `VkDeviceMemory`, provided their lifetimes of *use* (not object lifetime; the `VkImage`/`VkBuffer` handles can all exist simultaneously) do not overlap. `VK_IMAGE_CREATE_ALIAS_BIT` at image-creation time additionally permits two images with *identical* creation parameters (same format, extent, tiling, usage) to alias the same memory range and both be treated as validly containing each other's last-written data, which is the mechanism used when a driver or engine wants two type-identical view objects over one physical allocation rather than one image reinterpreted through format-compatible image views.

### 8.2 Transient Attachment Aliasing

The primary motivation for deliberate aliasing in ordinary (non-tile-based) engines is transient render-target memory: a deferred renderer's G-buffer attachments, a temporary MSAA resolve target, and a shadow-pass depth buffer are each live only within a narrow span of the frame and never simultaneously with each other's peak memory need, so backing them with the *same* aliased `VkDeviceMemory` range instead of three separate allocations can cut the transient-attachment memory footprint substantially — this is precisely the case §1.1's `VK_MEMORY_PROPERTY_LAZILY_ALLOCATED_BIT` addresses for tile-based mobile GPUs, where the implementation may back `VK_IMAGE_USAGE_TRANSIENT_ATTACHMENT_BIT` images with on-chip tile memory rather than DRAM at all; explicit aliasing via `VK_IMAGE_CREATE_ALIAS_BIT` is the desktop-GPU analogue, achieving the same memory reduction through explicit application-managed overlap rather than an implementation-managed on-chip resource.

### 8.3 Hazard and Synchronization Requirements

Aliasing shifts the correctness burden from the API to the application in a way ordinary Vulkan resource usage does not: the application alone is responsible for ensuring that at any point where the GPU accesses one alias, no other alias's contents are assumed live, and for inserting whatever barriers are necessary to make the transition from "image A's contents are the ones that matter" to "image B's contents are the ones that matter" visible — Vulkan's ordinary synchronization primitives (pipeline barriers, `VkImageMemoryBarrier2`) still apply, but the *dependency graph* the application must construct now includes edges between aliased resources that have no other relationship, and getting this wrong produces exactly the kind of read-after-write hazard the GPU-Assisted and Synchronization Validation layers (Chapter 201) are built to catch. In practice, most engines that alias transient attachments do so through a render-graph abstraction that tracks resource lifetimes automatically and computes the alias assignment and the necessary barriers as a byproduct of topologically sorting the frame's passes, rather than hand-managing aliasing per resource.

---

## 9. Residency and Device Loss

### 9.1 `VK_ERROR_OUT_OF_DEVICE_MEMORY` vs `VK_ERROR_OUT_OF_HOST_MEMORY`

The two allocation-failure error codes are not interchangeable and applications that treat them identically lose diagnostic information. `VK_ERROR_OUT_OF_HOST_MEMORY` means the *CPU-side* allocation the driver needed to service the call — for the internal bookkeeping structure behind a `VkDeviceMemory` handle, not the GPU memory itself — could not be satisfied from system RAM or from the application-supplied `VkAllocationCallbacks`; it can occur on almost any Vulkan call, not just `vkAllocateMemory`. `VK_ERROR_OUT_OF_DEVICE_MEMORY` specifically means the GPU-visible heap named by `memoryTypeIndex` could not satisfy the request — the failure §1.3's `VK_EXT_memory_budget` query exists to help applications anticipate and avoid.

### 9.2 Heap Eviction Under Pressure

A `VK_ERROR_OUT_OF_DEVICE_MEMORY` from `vkAllocateMemory` does not necessarily mean the heap is permanently full from the application's own allocations — on a shared, multi-process GPU, another process's VRAM footprint (a second game, a compositor's own GPU-composited surfaces, a separate ML workload) can push total system demand for a heap above its physical size, at which point the kernel driver (`amdgpu`, `i915`/`xe`, `nvidia`/`nouveau`) evicts colder allocations from VRAM to system memory to make room. That eviction is transparent to the API — the evicted `VkDeviceMemory` object remains valid and its contents are preserved, but subsequent GPU access to it now pays a PCIe round-trip until it is paged back in, which surfaces to the application only as an unexplained frame-time spike rather than as an explicit Vulkan-level signal. `VK_EXT_memory_priority`, mentioned in §1.3, is the API-level lever applications have to influence *which* of their own allocations get evicted first when the kernel driver needs to reclaim VRAM.

### 9.3 `VK_EXT_device_fault` and Post-Mortem Diagnosis

Where §9.1–9.2 describe soft allocation failures, `VK_EXT_device_fault` addresses the hard failure: `VK_ERROR_DEVICE_LOST`. After receiving that error from any Vulkan call, `vkGetDeviceFaultInfoEXT` "allows developers to query for additional information on GPU faults which may have caused device loss, and to generate binary crash dumps, which may be loaded into external tools for further diagnosis" [Source](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/appendices/VK_EXT_device_fault.adoc). The returned `VkDeviceFaultInfoEXT` carries a `VkDeviceFaultAddressInfoEXT` array classifying faulting addresses by `VkDeviceFaultAddressTypeEXT` (an invalid pointer versus a genuine page fault versus an out-of-bounds access, when the implementation can distinguish them) and a `VkDeviceFaultVendorInfoEXT` array of implementation-defined name/code/value triples — this is where RADV's AMD-specific UVM (unified virtual memory) fault-register decoding and ANV's equivalent Intel diagnostics surface vendor detail the core struct has no vocabulary for. Correlating a fault address back to a specific `VkBuffer`/`VkImage` and draw call is the job of `VK_EXT_debug_utils` object names and command-buffer debug labels (Chapter 201 §4) recorded *before* the hang — `VK_EXT_device_fault` tells you where the GPU faulted; the debug-label breadcrumb trail tells you which draw call was executing when it did. `VK_KHR_present_wait` is the complementary mechanism for the non-fault case: it lets an application block until a specific present operation has actually been displayed, which is useful for reasoning about frame-pacing-related memory pressure (how many frames of swapchain/upload-ring memory are genuinely in flight) without implying anything about device loss.

---

## 10. RADV/ANV/NVK Allocator Internals

Everything above this section is portable Vulkan API surface. This section follows one `vkAllocateMemory` call down through each of the three open-source Linux Vulkan drivers to the kernel object it produces, using Mesa's `main` branch as of commit `e7b3fdae` (2026-08-21) throughout.

### 10.1 RADV: `radv_alloc_memory` and the amdgpu Winsys

`vkAllocateMemory` on RADV is a thin wrapper: `radv_AllocateMemory` calls `radv_alloc_memory(device, pAllocateInfo, pAllocator, pMem, /* is_internal */ false)` [Source, gitlab.freedesktop.org/mesa/mesa @ e7b3fdae](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/amd/vulkan/radv_device_memory.c#L327-331), with the internal function shared by driver-internal allocations (scratch buffers, internal descriptor tables) that pass `is_internal = true`. `radv_alloc_memory` walks the `VkMemoryAllocateInfo` `pNext` chain for every extension struct this chapter has covered — `VkMemoryDedicatedAllocateInfo`, `VkExportMemoryAllocateInfo`, `VkImportMemoryFdInfoKHR`, `VkMemoryPriorityAllocateInfoEXT`, `VkMemoryOpaqueCaptureAddressAllocateInfo`, `VkMemoryAllocateFlagsInfo` — translating each into a `radeon_bo_flag`/`radeon_bo_domain` pair that gets passed to `radv_bo_create` [Source, gitlab.freedesktop.org/mesa/mesa @ e7b3fdae](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/amd/vulkan/radv_device_memory.c#L75-269). Two details worth calling out:

- **Priority is a float, quantized to an internal fixed range.** `VkMemoryPriorityAllocateInfoEXT::priority` (a `[0,1]` float, defaulting to 0.5) is mapped to `MIN2(RADV_BO_PRIORITY_APPLICATION_MAX - 1, priority_float * RADV_BO_PRIORITY_APPLICATION_MAX)` before being handed to the winsys, which is the input to the kernel-level eviction ordering §9.2 describes.
- **`overallocation_disallowed` is an opt-in soft cap.** When enabled, RADV tracks `allocated_memory_size[heap_index]` under a mutex and fails the allocation with `VK_ERROR_OUT_OF_DEVICE_MEMORY` before ever calling into the winsys if the requested size would exceed the heap's reported total — a driver-side enforcement of the exact number `VK_EXT_memory_budget` reports, for applications and driver configurations that want a hard rather than a soft signal.

The actual GEM object comes from the `amdgpu` winsys, not this file: `radv_amdgpu_winsys_bo_create` builds an `amdgpu_bo_alloc_request`, submits it through libdrm's `ac_drm` wrapper to get a kernel GEM buffer handle, and then separately allocates and binds a GPU virtual-address range for it via `ac_drm_va_range_alloc`/`amdgpu_va_handle` [Source, gitlab.freedesktop.org/mesa/mesa @ e7b3fdae](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/amd/vulkan/winsys/amdgpu/radv_amdgpu_bo.c#L514-549) — the GEM handle and the GPU VA binding are two separate kernel operations, mirroring the general DRM model (Chapter 3/Chapter 24) where a buffer object's existence and its placement in a given VM address space are independent.

### 10.2 ANV: `anv_device_alloc_bo` and the KMD Backend Abstraction

ANV's allocation path is organized around a single core function, `anv_device_alloc_bo`, that every higher-level allocation path (`vkAllocateMemory`'s handler, internal scratch/descriptor-pool allocations, the BO cache) funnels through [Source, gitlab.freedesktop.org/mesa/mesa @ e7b3fdae](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/intel/vulkan/anv_allocator.c#L1644-1650). Notable structure:

- **A slab allocator sits in front of the kernel path.** `anv_device_alloc_bo` first tries `anv_slab_bo_alloc` — a sub-allocator for small BOs carved out of larger kernel allocations — and only falls through to a fresh `gem_create` call when the slab allocator cannot satisfy the request, after re-aligning the size to 4 KiB (or up to 2 MiB when the platform benefits from huge-page-backed oversubscription) [Source, gitlab.freedesktop.org/mesa/mesa @ e7b3fdae](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/intel/vulkan/anv_allocator.c#L1676-1705). This means ANV performs its own sub-block suballocation beneath the kernel even before VMA-style application-level suballocation enters the picture — a second layer applications generally never observe.
- **The kernel driver is abstracted behind `device->kmd_backend`.** `gem_create`, `gem_close`, and `vm_bind_bo` are function-pointer members of a `kmd_backend` struct rather than direct ioctl calls, letting the same ANV code target both the legacy `i915` and the newer `xe` kernel driver without a compile-time branch [Source, gitlab.freedesktop.org/mesa/mesa @ e7b3fdae](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/intel/vulkan/anv_allocator.c#L1735-1771).
- **Local-memory region selection is explicit.** On platforms with discrete VRAM, `anv_device_alloc_bo` builds a `regions[]` array choosing between `vram_non_mappable` and the system-memory region depending on `ANV_BO_ALLOC_NO_LOCAL_MEM`/`ANV_BO_ALLOC_MAPPED`/`ANV_BO_ALLOC_LOCAL_MEM_CPU_VISIBLE` flags derived from the Vulkan memory-type request, mirroring §1's visible-VRAM/invisible-VRAM heap split at the kernel-region level rather than only at the `VkMemoryHeap` level [Source, gitlab.freedesktop.org/mesa/mesa @ e7b3fdae](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/intel/vulkan/anv_allocator.c#L1707-1732).

### 10.3 NVK: `nvk_AllocateMemory` and `nvkmd`

NVK follows the same overall shape but defers the kernel-facing work to a separate internal library, `nvkmd`, abstracting NVK's two possible kernel backends (upstream `nouveau` and NVIDIA's open GPU kernel modules). `nvk_AllocateMemory` resolves the requested `VkMemoryType`, computes alignment starting from `pdev->nvkmd->bind_align_B` and widening it for dedicated image allocations that need 64 KiB or 2 MiB alignment for compression/tiling metadata, then dispatches to one of three `nvkmd_dev_*` entry points depending on whether the allocation is a dma-buf import, a tiled/compressed dedicated-image allocation (carrying a `pte_kind`/`tile_mode` pair from the image's NIL — NVIDIA Image Layout — description), or a plain allocation [Source, gitlab.freedesktop.org/mesa/mesa @ e7b3fdae](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/nouveau/vulkan/nvk_device_memory.c#L129-241). Two points of contrast with RADV/ANV:

- **`VK_MEMORY_ALLOCATE_ZERO_INITIALIZE_BIT_EXT` is implemented in software.** Where a kernel driver might zero pages as part of allocation, NVK's handler for that flag explicitly `memset`s host-visible memory or issues an `nvk_upload_queue_fill` GPU clear for device-local memory, with a debug-only counterpart (`NVK_DEBUG_TRASH_MEMORY`) that instead fills freshly allocated memory with a poison pattern (`0xF1`/`0xCAFEF00D`) to surface use-of-uninitialized-memory bugs during development [Source, gitlab.freedesktop.org/mesa/mesa @ e7b3fdae](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/nouveau/vulkan/nvk_device_memory.c#L243-289).
- **Heap accounting is a simple atomic counter, not a kernel query.** `nvk_AllocateMemory`/`nvk_FreeMemory` maintain `pdev->mem_heaps[heapIndex].used` via `p_atomic_add`, a userspace-only running total rather than a value sourced from the kernel driver on every allocation [Source, gitlab.freedesktop.org/mesa/mesa @ e7b3fdae](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/nouveau/vulkan/nvk_device_memory.c#L305-333) — a materially simpler accounting model than RADV's `VK_EXT_memory_budget` reconciliation logic in §1.3, reflecting NVK's comparative youth as a driver rather than any difference mandated by the hardware.

### 10.4 BAR Window Constraints on Discrete GPUs

A discrete GPU's VRAM is only CPU-addressable through its PCIe **Base Address Register (BAR)** window — historically a fixed, often small (256 MiB) aperture regardless of total VRAM size, because BAR sizes are constrained by 32-bit PCI address decoding on older platforms. "Resizable BAR" (PCIe spec-level Resizable BAR combined with platform/firmware support) lets the BAR be sized up to the GPU's full VRAM capacity, which is what makes an `all_vram_visible` memory-type configuration — a single heap that is simultaneously `DEVICE_LOCAL` and `HOST_VISIBLE` across all of VRAM — possible at all; without it, only the (small, historically 256 MiB) BAR-covered slice of VRAM can ever be a `HOST_VISIBLE | DEVICE_LOCAL` type, and the remainder is `DEVICE_LOCAL`-only.

RADV's heap construction makes this concrete: `radv_physical_device_init_mem_types` computes `visible_vram_size` from the BAR-mapped portion the kernel reports and a separate `vram_size` for the non-mapped remainder, with an explicit debug path (`hide_rebar_on_dgpu`) that artificially clamps the visible-VRAM heap back down to a synthetic 256 MiB "virtual carveout" — reproducing pre-ReBAR behavior — specifically to work around games whose allocation heuristics misbehave when handed a much larger `HOST_VISIBLE | DEVICE_LOCAL` heap than they were tuned against [Source, gitlab.freedesktop.org/mesa/mesa @ e7b3fdae](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/amd/vulkan/radv_physical_device.c#L447-482). ANV's physical-device code carries the identical split under different names — `vram_mappable` versus `vram_non_mappable` regions, with `vram_non_mappable.region` populated only when the platform reports unmappable VRAM beyond the BAR window [Source, gitlab.freedesktop.org/mesa/mesa @ e7b3fdae](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/intel/vulkan/anv_physical_device.c#L2371-2423). The practical consequence for application authors: an allocation from the `DEVICE_LOCAL | HOST_VISIBLE` type is not free to make large regardless of how big VRAM is — it competes for a physically limited mapping window that the driver may have deliberately capped even on hardware capable of exposing more, and the §4.1 VMA block-size default (256 MiB) is itself sized with the historical, pre-ReBAR BAR window in mind.

---

## Integrations

**Chapter 24 (Vulkan and EGL for Application Developers)** covers Vulkan initialization and the presentation stack this chapter's allocation and binding material sits underneath; readers unfamiliar with basic `VkDevice`/queue setup should start there. Its WSI/swapchain material is also where multi-planar `linux-dmabuf` images (§3.3 of this chapter) most commonly appear as import targets.

**Chapter 106 (The Vulkan Memory Model)** covers the orthogonal concern this chapter deliberately does not: once memory is allocated and bound, what ordering and visibility guarantees govern concurrent GPU access to it. §3's alignment and binding material here is a prerequisite for reasoning correctly about Ch106's atomics and barrier scopes, and §8's aliasing hazards are a case where this chapter's allocation-level rules (which alias is "live") and Ch106's execution-level rules (what a barrier actually orders) must both be satisfied simultaneously.

**Chapter 154 (GPU-Driven Rendering)** is where §5's buffer device addresses and §6's sparse-residency material do the most work in practice: large indirect-draw and indirect-dispatch parameter buffers are exactly the case where per-object suballocation (§4) and raw-pointer traversal (§5.2) replace per-draw descriptor updates, and GPU culling systems that stream visibility data at scale are natural sparse-residency or virtual-texture-style consumers of §6.

**Chapter 157 (Vulkan Descriptor Binding in Depth)** covers descriptor buffers, the GPU-visible-heap-backed alternative to descriptor sets, whose backing memory is allocated and bound through exactly the mechanisms in §§1–3 of this chapter — a descriptor buffer is, from the memory-management side, an ordinary `VkBuffer` with `VK_BUFFER_USAGE_RESOURCE_DESCRIPTOR_BUFFER_BIT_EXT` and nothing else new.

**Chapter 82 (Vulkan Ecosystem Toolkit: VMA, volk, vk-bootstrap, and Friends)** is the companion chapter for VMA's application-facing API — bootstrapping, the staging-upload pattern, custom pools, and statistics/JSON export are covered there in full; §4 of this chapter assumes that material and instead goes one level deeper, into the TLSF suballocator and strategy flags that determine how VMA places allocations within its blocks.

---

## References

- [`VkPhysicalDeviceMemoryProperties`(3) — Vulkan Registry](https://registry.khronos.org/vulkan/specs/latest/man/html/VkPhysicalDeviceMemoryProperties.html) — memory heap/type structure and limits (§1)
- [`vkGetPhysicalDeviceMemoryProperties2`(3) — Vulkan Registry](https://registry.khronos.org/vulkan/specs/latest/man/html/vkGetPhysicalDeviceMemoryProperties2.html) — pNext-extensible query entry point (§1.2)
- [`VK_EXT_memory_budget.adoc` — KhronosGroup/Vulkan-Docs, `main`](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/appendices/VK_EXT_memory_budget.adoc) — `VkPhysicalDeviceMemoryBudgetPropertiesEXT`, heapBudget/heapUsage semantics and caveats (§1.3, §9.2)
- [`vkAllocateMemory`(3) — Vulkan Registry](https://registry.khronos.org/vulkan/specs/latest/man/html/vkAllocateMemory.html) — `VkMemoryAllocateInfo` and allocation entry point (§2.1)
- [`VkMemoryDedicatedAllocateInfo`(3) — Vulkan Registry](https://registry.khronos.org/vulkan/specs/latest/man/html/VkMemoryDedicatedAllocateInfo.html) — dedicated allocation for external memory and layout-constrained resources (§2.2)
- [`memory.adoc` — KhronosGroup/Vulkan-Docs, `main`](https://raw.githubusercontent.com/KhronosGroup/Vulkan-Docs/main/chapters/memory.adoc) — `maxMemoryAllocationCount`, dedicated-allocation VUIDs, disjoint-image/dedicated-allocation exclusion (§2.3, §3.3)
- [`vkBindBufferMemory2`(3) — Vulkan Registry](https://registry.khronos.org/vulkan/specs/latest/man/html/vkBindBufferMemory2.html) — batched bind via `VkBindBufferMemoryInfo` array (§3.1)
- [`VkBindImagePlaneMemoryInfo`(3) — Vulkan Registry](https://registry.khronos.org/vulkan/specs/latest/man/html/VkBindImagePlaneMemoryInfo.html) — per-plane binding for disjoint multi-planar images (§3.3)
- [VulkanMemoryAllocator `vk_mem_alloc.h` — GPUOpen-LibrariesAndSDKs, `master`](https://raw.githubusercontent.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator/master/include/vk_mem_alloc.h) — `VmaAllocationCreateInfo` strategy flags, `VmaAllocationInfo`, linear-pool algorithm doc comments, default block size (§4.1, §4.2, §4.3, §4.4)
- [VulkanMemoryAllocator `CHANGELOG.md` — GPUOpen-LibrariesAndSDKs, `master`](https://raw.githubusercontent.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator/master/CHANGELOG.md) — VMA 3.0.0 TLSF adoption, buddy-algorithm removal, defragmentation API rework (§4.1, §4.5)
- [`VK_KHR_buffer_device_address.adoc` — KhronosGroup/Vulkan-Docs, `main`](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/appendices/VK_KHR_buffer_device_address.adoc) — `vkGetBufferDeviceAddress`, `PhysicalStorageBuffer`, capture/replay addressing (§5.1, §5.3)
- [`sparsemem.adoc` — KhronosGroup/Vulkan-Docs, `main`](https://raw.githubusercontent.com/KhronosGroup/Vulkan-Docs/main/chapters/sparsemem.adoc) — `VkSparseImageMemoryRequirements`, mip-tail packing models, `vkQueueBindSparse` ordering (§6.1, §6.2)
- [`VK_EXT_host_image_copy.adoc` — KhronosGroup/Vulkan-Docs, `main`](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/appendices/VK_EXT_host_image_copy.adoc) — host-side image transfer without staging buffers (§7.1, §7.2)
- [VK_EXT_host_image_copy proposal — Vulkan Documentation Project](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_host_image_copy.html) — Vulkan 1.4 core promotion as an optional feature (§7.1)
- [`VK_EXT_device_fault.adoc` — KhronosGroup/Vulkan-Docs, `main`](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/appendices/VK_EXT_device_fault.adoc) — `vkGetDeviceFaultInfoEXT`, `VkDeviceFaultVendorInfoEXT`, post-`VK_ERROR_DEVICE_LOST` diagnostics (§9.3)
- [`radv_device_memory.c` — Mesa `mesa/mesa` @ `e7b3fdae`](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/amd/vulkan/radv_device_memory.c) — `radv_alloc_memory`, pNext extension handling, priority quantization, `overallocation_disallowed` (§10.1)
- [`radv_amdgpu_bo.c` — Mesa `mesa/mesa` @ `e7b3fdae`](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/amd/vulkan/winsys/amdgpu/radv_amdgpu_bo.c) — `radv_amdgpu_winsys_bo_create`, GEM handle allocation and separate VA-range binding (§10.1)
- [`radv_physical_device.c` — Mesa `mesa/mesa` @ `e7b3fdae`](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/amd/vulkan/radv_physical_device.c) — `radv_physical_device_init_mem_types` visible/invisible VRAM heap split, resizable-BAR carveout workaround, `VK_EXT_memory_budget` reconciliation (§1.3, §10.4)
- [`anv_allocator.c` — Mesa `mesa/mesa` @ `e7b3fdae`](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/intel/vulkan/anv_allocator.c) — `anv_device_alloc_bo`, slab sub-allocator, `kmd_backend` abstraction, local-memory region selection (§10.2)
- [`anv_physical_device.c` — Mesa `mesa/mesa` @ `e7b3fdae`](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/intel/vulkan/anv_physical_device.c) — `vram_mappable`/`vram_non_mappable` region split (§10.4)
- [`nvk_device_memory.c` — Mesa `mesa/mesa` @ `e7b3fdae`](https://gitlab.freedesktop.org/mesa/mesa/-/blob/e7b3fdaefa65239e09a5eb7e23d7133a5862ef15/src/nouveau/vulkan/nvk_device_memory.c) — `nvk_AllocateMemory`, `nvkmd` dispatch, zero-initialize/trash-memory debug paths, atomic heap accounting (§10.3)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
