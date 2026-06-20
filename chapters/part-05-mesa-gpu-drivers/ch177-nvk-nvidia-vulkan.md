# Chapter 177: NVK — NVIDIA Vulkan in Mesa

> **Part**: Part V — Mesa GPU Drivers
> **Audiences**: Systems and driver developers studying Mesa Vulkan driver architecture; graphics application developers targeting open NVIDIA hardware on Linux; contributors to the NVK driver in Mesa
> **Status**: First draft — 2026-06-20

## Table of Contents

- [Overview](#overview)
- [1. History and Driver Lineage](#1-history-and-driver-lineage)
- [2. Source Tree Layout and the NVKMD Abstraction](#2-source-tree-layout-and-the-nvkmd-abstraction)
- [3. Memory Architecture: GEM, TTM, VRAM, and GART](#3-memory-architecture-gem-ttm-vram-and-gart)
- [4. Command Encoding: Push Buffers, Class Headers, and the nv\_push API](#4-command-encoding-push-buffers-class-headers-and-the-nv_push-api)
- [5. NAK Integration: NIR to SASS in the Mesa Pipeline](#5-nak-integration-nir-to-sass-in-the-mesa-pipeline)
- [6. GSP-RM Firmware Interaction and Its Implications](#6-gsp-rm-firmware-interaction-and-its-implications)
- [7. Window System Integration: Kopper, DMA-BUF, and Wayland Explicit Sync](#7-window-system-integration-kopper-dma-buf-and-wayland-explicit-sync)
- [8. Timeline Semaphores and the drm\_syncobj Model](#8-timeline-semaphores-and-the-drm_syncobj-model)
- [9. Conformance Timeline and Known Gaps](#9-conformance-timeline-and-known-gaps)
- [10. Building and Testing NVK](#10-building-and-testing-nvk)
- [Integrations](#integrations)
- [References](#references)

---

## Overview

This chapter targets three overlapping audiences. **Systems and driver developers** will find a precise treatment of how NVK's NVKMD abstraction layer, push buffer encoding, and synchronisation model differ from AMD's RADV and Intel's ANV — the two other major production Mesa Vulkan drivers covered in Chapter 18. **Graphics application developers** will learn which GPU generations support which Vulkan versions, how the GSP-RM firmware dependency shapes runtime performance, and how to configure and test NVK for their workloads. **Contributors to NVK** will find the chapter a structured entry point to the source tree, covering the NVKMD function-pointer interface, the nv_push encoding layer, the NAK compiler handoff, and the dEQP testing workflow.

**NVK** is NVIDIA's open-source Vulkan driver in Mesa, located at `src/nouveau/vulkan/` in the Mesa repository. [Source](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/nouveau/vulkan) It is a clean-slate Mesa Vulkan driver — not a wrapper over the legacy Gallium nouveau driver, not a shim over NVIDIA's proprietary stack — designed from the ground up using Mesa's modern Vulkan common infrastructure. NVK provides a conformant Vulkan 1.4 implementation for all supported GPU generations, from Maxwell (GeForce GTX 750, 2014) through Blackwell (GeForce RTX 5090, 2025), with Kepler (GeForce GTX 660, 2012) providing conformant Vulkan 1.2.

Chapter 10b ("NVK: Building a Vulkan Driver from Scratch") tells the origin story of NVK — why Faith Ekstrand chose a clean-sheet design over extending the legacy Gallium nouveau driver, how the object model, memory heaps, NIR lowering, push buffer encoding, and synchronisation primitives are implemented, and what lessons the project holds for future Mesa Vulkan driver authors. Readers who have not yet read Chapter 10b should do so before this chapter, because ch177 builds on that foundation rather than repeating it.

This chapter's contribution relative to Chapter 10b is threefold: a detailed examination of the **NVKMD abstraction layer** that decouples NVK from any specific kernel ABI; a focused treatment of **WSI integration** via the Kopper layer, DMA-BUF, and the Wayland explicit sync protocol; and a complete **conformance and deployment history** tracking NVK from its Mesa 23.3 landing through Maxwell/Pascal/Volta and Kepler/Blackwell enablement in Mesa 25.x and 26.x. The chapter closes with a practical guide to building, debugging, and running dEQP conformance tests against NVK.

---

## 1. History and Driver Lineage

### Initial Development: 2022–2023

NVK was first shown publicly by Faith Ekstrand (then at Collabora, previously the maintainer of Intel's ANV driver) in October 2022. At that stage the driver passed approximately 98% of the Vulkan CTS for the subset of tests that it ran, but was running only a small fraction of the full CTS due to the limited feature set. [Source](https://www.collabora.com/news-and-blog/news-and-events/introducing-nvk.html) The initial hardware target was Turing (RTX 2000 series / GTX 1600 series, 2018–2019), which was the first NVIDIA generation with GSP-RM firmware support in the upstream nouveau kernel driver.

Karol Herbst, Dave Airlie, and approximately a dozen community contributors joined the project over 2022–2023. The kernel uAPI changes needed to support NVK — specifically the `DRM_IOCTL_NOUVEAU_EXEC` and `DRM_IOCTL_NOUVEAU_VM_BIND` ioctls described in Section 8 — were developed in parallel with the Mesa driver and landed in Linux 6.6. [Source](https://airlied.blogspot.com/2023/08/nvk-kernel-changes-needed.html) NVK was merged into the main Mesa repository in August 2023, shipping as an experimental driver in Mesa 23.3. [Source](https://www.gamingonlinux.com/2023/08/nvk-the-open-source-driver-for-nvidia-merged-into-mesa/)

### Mesa 24.x: NAK, Stability, and Widened Hardware Support

Mesa 24.0 (February 2024) brought NAK — the Rust-based NVIDIA shader compiler backend — into the release, making NVK's Turing path far more performant than the earlier Nouveau nv50_ir-derived code. The driver was still experimental in 24.0 but already passing nearly the full CTS. Mesa 24.1 (May 2024) promoted NVK to stable status and exposed Vulkan 1.3, enabling NVK for use in production Turing workloads. [Source](https://www.phoronix.com/news/NVK-Vulkan-1.3-Mesa-24.1) The meson build option changed from `nouveau-experimental` to `nouveau` at this point.

DXVK (D3D9/10/11 → Vulkan) was confirmed to work out of the box against NVK from Mesa 24.1, and VKD3D-Proton (D3D12 → Vulkan) also gained compatibility during this period, making NVK viable for Windows game compatibility through Steam's Proton layer. [Source](https://www.phoronix.com/news/Mesa-2024-Highlights)

### Mesa 25.x: Vulkan 1.4 and Architecture Breadth

Mesa 25.0 (early 2025) made NVK a day-zero Vulkan 1.4 conformant implementation for Turing and later hardware. Maxwell, Pascal, and Volta reached Vulkan 1.4 conformance in April 2025 and were enabled by default in Mesa 25.1 (May 2025). [Source](https://www.collabora.com/news-and-blog/news-and-events/nvk-enabled-for-maxwell,-pascal,-and-volta-gpus.html) At the same time, Mesa 25.1 switched Turing and later GPUs to use NVK+Zink as the default OpenGL implementation, ending the long tenure of the legacy Gallium nouveau GL driver as the primary OpenGL path for those GPUs. [Source](https://itsfoss.com/news/mesa-zink-nvk-switch/)

Mesa 25.2 added Blackwell (RTX 5000 series) and Kepler (GeForce 600/700 series) support, meaning NVK now covers every GPU generation that NVIDIA's proprietary driver ever served on Linux desktop. [Source](https://www.collabora.com/news-and-blog/news-and-events/mesa-25.2-brings-new-hardware-support-for-nouveau-users.html) Hopper support was also enabled experimentally (no consumer Hopper GPUs exist, but the driver enumerates correctly on server Hopper cards).

### Mesa 26.x: Extension Coverage

Mesa 26.0 (February 2026) brought additional extension coverage to NVK: `VK_KHR_maintenance10`, `VK_EXT_discard_rectangles`, `VK_EXT_shader_uniform_buffer_unsized_array`, `VK_KHR_robustness2`, and `VK_KHR_pipeline_binary`. NAK received pre-pass instruction scheduling across basic blocks and refined instruction latency models for Ampere and later architectures. Turing specifically gained improved warp occupancy by switching to the maximum warps-per-SM configuration for compute workloads. [Source](https://en.linuxadictos.com/Mesa-26.0-strengthens-Vulkan-support-and-adds-dozens-of-key-extensions-to-radv--anv--nvk--panvk--Venus--and-other-drivers..html)

Ray tracing (`VK_KHR_ray_tracing_pipeline`, `VK_KHR_acceleration_structure`) and hardware video decode (`VK_KHR_video_decode_queue`) remain active development areas as of mid-2026. Experimental H.264 and H.265 video decode is available behind the `NVK_I_WANT_A_BROKEN_VULKAN_DRIVER=1` environment variable, requiring Linux kernel 6.12 or later. [Source](https://blogs.igalia.com/scerveau/vulkan-video-with-nvk-driver/)

---

## 2. Source Tree Layout and the NVKMD Abstraction

### Directory Structure

NVK's source tree branches from `src/nouveau/` in the Mesa repository, which contains three major components:

```
src/nouveau/
├── compiler/           # NAK — the Rust-based NVIDIA shader compiler backend
│   └── nak/            # Core NAK Rust source (ir.rs, lower_*.rs, encode_*.rs)
├── headers/            # NVIDIA class headers and nv_push encoding API
│   ├── nv_push.h       # push buffer struct and nv_push_* helpers
│   ├── nv_push.c       # nv_push_dump disassembler
│   └── cl9097.h        # Kepler/Maxwell/Pascal 3D class methods (NV9097)
│   └── clc597.h        # Turing 3D class methods (NVC597)
│   └── ...             # Per-generation class headers (Ampere, Ada, Blackwell)
├── vulkan/             # NVK proper
│   ├── nvk_cmd_buffer.c / .h
│   ├── nvk_device.c / .h
│   ├── nvk_physical_device.c / .h
│   ├── nvk_heap.c / .h
│   ├── nvk_pipeline.c  # Graphics and compute pipeline compilation
│   ├── nvk_descriptor_set.c / .h
│   ├── nvk_wsi.c       # Kopper WSI integration
│   └── nvkmd/          # NVKMD abstraction layer
│       ├── nvkmd.h     # Function pointer table (vtable) definition
│       ├── nouveau/    # Standard nouveau kernel driver backend
│       └── ...         # Reserved for future alternative kernel backends
└── winsys/             # nouveau_ws_* — low-level winsys helpers
```

This layout makes a clear distinction between three concerns: the NVIDIA class-level command encoding (headers/), the Vulkan API implementation (vulkan/), and the kernel interface abstraction (vulkan/nvkmd/). The compiler lives in compiler/ because NAK serves both NVK and, potentially in future, the legacy Gallium nouveau driver.

### The NVKMD Abstraction Layer

The most architecturally significant internal boundary in NVK is **NVKMD** (Nouveau Kernel Mode Driver abstraction), defined in `src/nouveau/vulkan/nvkmd/nvkmd.h`. NVKMD is a function-pointer vtable that abstracts every interaction with the nouveau kernel driver behind a vendor-neutral interface. This means NVK's Vulkan implementation code never calls DRM ioctls directly; it calls through NVKMD, which dispatches to the appropriate backend.

```c
/* Source: src/nouveau/vulkan/nvkmd/nvkmd.h (conceptual representation) */
struct nvkmd_ops {
    /* Device lifetime */
    VkResult (*dev_create)(struct nvkmd_dev **dev_out, ...);
    void     (*dev_destroy)(struct nvkmd_dev *dev);
    VkResult (*dev_query_info)(struct nvkmd_dev *dev,
                               struct nvkmd_dev_info *info);

    /* Buffer object (BO) management */
    VkResult (*bo_alloc)(struct nvkmd_dev *dev,
                         uint64_t size_B,
                         uint32_t align_B,
                         enum nvkmd_mem_flags flags,
                         struct nvkmd_bo **bo_out);
    void     (*bo_free)(struct nvkmd_bo *bo);
    VkResult (*bo_map)(struct nvkmd_bo *bo, void **map_out);
    void     (*bo_unmap)(struct nvkmd_bo *bo);

    /* Virtual memory (VM) management */
    VkResult (*vm_bind)(struct nvkmd_dev *dev,
                        struct nvkmd_bo *bo,
                        uint64_t offset_B,
                        uint64_t size_B,
                        uint64_t va);
    VkResult (*vm_unbind)(struct nvkmd_dev *dev,
                          uint64_t va,
                          uint64_t size_B);

    /* Queue submission */
    VkResult (*queue_submit)(struct nvkmd_queue *queue,
                             uint32_t push_count,
                             const struct nvkmd_push_entry *pushes,
                             uint32_t wait_count,
                             const struct nvkmd_sync_entry *waits,
                             uint32_t sig_count,
                             const struct nvkmd_sync_entry *signals);
    VkResult (*queue_wait_idle)(struct nvkmd_queue *queue);
};
```

> **Note: needs verification** — The above struct layout is a conceptual representation based on public documentation. Exact field names and signatures should be verified against the current `nvkmd.h` in the Mesa source.

The NVKMD interface has one live backend today — the `nouveau/` subdirectory, which translates NVKMD calls to `DRM_IOCTL_NOUVEAU_EXEC`, `DRM_IOCTL_NOUVEAU_VM_BIND`, `DRM_NOUVEAU_GEM_NEW`, and related ioctls. The interface was designed with sufficient generality that a Nova-based backend (Nova is the Rust NVIDIA kernel driver described in Chapter 10a) could be plugged in without changing any code above the NVKMD layer. This is the NVK equivalent of RADV's `amdgpu` winsys layer or ANV's `anv_kmd_backend` vtable described in Chapter 18.

### Comparison with RADV's Winsys Model

The RADV driver uses a similar but independently evolved abstraction: the `radv_winsys` interface wraps the `amdgpu` kernel library (libdrm_amdgpu). The structural parallel is exact: RADV's `radv_amdgpu_winsys_alloc_buffer()` corresponds to NVKMD's `bo_alloc()`, and RADV's submission path through `amdgpu_cs_submit()` corresponds to NVKMD's `queue_submit()`. The key difference is the kernel protocol: RADV submits command streams as **Indirect Buffers** (IBs) referenced by PM4 `INDIRECT_BUFFER` packets, while NVK submits push buffer segments as virtual-address/length pairs to the nouveau channel. Both models result in the GPU reading a sequence of commands from memory, but the framing differs at the kernel-userspace interface level.

---

## 3. Memory Architecture: GEM, TTM, VRAM, and GART

### The GEM/TTM Relationship

A common point of confusion for readers coming from AMD or Intel driver backgrounds is how NVK relates to TTM (Translation Table Manager). The answer requires distinguishing two layers.

At the **kernel level**, the nouveau kernel driver uses TTM for physical GPU memory management. TTM is the kernel subsystem that allocates VRAM pages and system memory pages, manages eviction between VRAM and system RAM under memory pressure, and tracks per-BO placement. Every `nouveau_bo` (nouveau Buffer Object) is backed by a `ttm_buffer_object`; TTM decides whether it lives in VRAM or pinned system memory at any given moment, potentially evicting other BOs to satisfy a placement request.

At the **userspace level**, NVK (like all Mesa drivers) allocates GPU memory through the **GEM** (Graphics Execution Manager) interface. GEM is the DRM subsystem's user-facing API for buffer object lifecycle management: allocation (`DRM_NOUVEAU_GEM_NEW`), CPU mapping (`DRM_NOUVEAU_GEM_CPU_MAP`), and free (close on the GEM handle). NVK calls these GEM ioctls through the NVKMD backend; it does not interact with TTM directly. TTM is an implementation detail of the nouveau kernel driver, invisible above the NVKMD interface.

This is explicitly different from the situation on amdgpu, where libdrm_amdgpu provides a userspace library that makes BO management and scheduling simpler. Nouveau does not have an equivalent userspace library; NVK communicates with the kernel through raw DRM ioctls.

### Memory Regions and Vulkan Heap Mapping

NVIDIA GPU hardware presents three logical memory regions that NVK maps to Vulkan memory heaps and types:

**VRAM** is the GPU's on-board GDDR or HBM memory. It is fast for GPU access and inaccessible to the CPU except through the BAR1 aperture. NVK exposes VRAM as the `VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT` heap. Applications should place textures, render targets, and geometry buffers that the GPU reads frequently here.

**BAR1 aperture** is a PCIe window that maps a portion of VRAM into the CPU's address space. On systems without Resizable BAR (ReBAR), this window is limited to 256 MB — only a small fraction of total VRAM. On modern systems with ReBAR enabled in UEFI, the full VRAM extent is accessible, typically 8–24 GB. NVK exposes BAR1/ReBAR memory as `VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT | VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`. Resources placed here can be written directly by the CPU without an explicit upload copy — a significant benefit for streaming vertex data or frequently-updated uniform buffers when ReBAR is available.

**GART/system memory** is host RAM made accessible to the GPU through the IOMMU. NVK exposes two GART types: a coherent variant (`VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`) and a cached variant (`VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_CACHED_BIT`). The coherent type has write-combined CPU access and requires no explicit cache flush operations; the cached type has full cached CPU access but requires CPU cache flushes before the GPU reads updated data. Staging buffers, readback buffers, and uniform buffers frequently accessed by the CPU belong here.

```
Heap layout (discrete GPU, non-ReBAR example):

Heap 0: DEVICE_LOCAL                      — VRAM (e.g., 8 GB GDDR6)
Heap 1: HOST_VISIBLE | DEVICE_LOCAL       — BAR1 aperture (256 MB)
Heap 2: HOST_VISIBLE                      — System RAM (via GART)

Memory types:
  Type 0 → Heap 0  — DEVICE_LOCAL                               (VRAM, GPU-only)
  Type 1 → Heap 1  — DEVICE_LOCAL | HOST_VISIBLE | HOST_COHERENT (BAR1)
  Type 2 → Heap 2  — HOST_VISIBLE | HOST_COHERENT               (GART coherent)
  Type 3 → Heap 2  — HOST_VISIBLE | HOST_CACHED                 (GART cached)
```

The exact heap and type indices depend on hardware topology. On Tegra (integrated NVIDIA GPU sharing system memory), the heap layout is different: there is no dedicated VRAM, and system memory is device-local.

### The nvk_heap Suballocator

NVK does not call `DRM_NOUVEAU_GEM_NEW` for every small driver-internal allocation. Instead, it maintains a **suballocator** called `nvk_heap`, implemented in `src/nouveau/vulkan/nvk_heap.c`. The heap allocates large backing GEM BOs (typically 64 MB each) and serves driver-internal allocations from those slabs using a virtual memory free-list tracked by `util_vma_heap`. This pattern matches Mesa's standard practice in RADV and ANV: large pools are allocated infrequently; small allocations (push buffer segments, descriptor pool entries, compiled SASS binaries, QMD compute descriptors) are served from those pools without kernel round-trips.

```c
/* Source: src/nouveau/vulkan/nvk_heap.c — nvk_heap_alloc() */
VkResult
nvk_heap_alloc(struct nvk_device *dev,
               struct nvk_heap *heap,
               uint64_t size,
               uint32_t alignment,
               uint64_t *addr_out,
               void **map_out)
{
    simple_mtx_lock(&heap->mutex);
    /* Try to satisfy from existing slab */
    uint64_t addr = util_vma_heap_alloc(&heap->vma_heap, size, alignment);
    if (addr != 0) {
        simple_mtx_unlock(&heap->mutex);
        *addr_out = addr;
        if (map_out)
            *map_out = (char *)heap->bo_map + (addr - heap->bo_addr);
        return VK_SUCCESS;
    }
    /* Slab exhausted: grow by allocating a new backing BO */
    return nvk_heap_grow_locked(dev, heap, size, alignment, addr_out, map_out);
}
```

Application-visible `VkDeviceMemory` allocations are handled separately: allocations above a driver-tunable threshold receive a dedicated GEM BO, while smaller allocations are pooled to avoid GEM BO handle exhaustion. The nouveau kernel driver imposes a per-process limit on simultaneous open GEM handles; without pooling, an application with many small allocations would exhaust this limit.

---

## 4. Command Encoding: Push Buffers, Class Headers, and the nv\_push API

### NVIDIA's Channel and Push Buffer Model

NVIDIA GPUs execute work submitted via **channels**. A channel is a per-client GPU context that has its own set of hardware subchannel bindings and an associated DMA engine that reads command streams from memory. Each channel's command stream consists of **push buffer** segments — sequences of method/data pairs that the GPU's front-end hardware decodes and dispatches to the appropriate engine.

A method/data pair is a 64-bit unit (in most modern NVIDIA hardware): the upper 32 bits encode the subchannel, the method address (register offset within the class), and a count or flag indicating whether subsequent 32-bit words are additional data or a new header. The GPU's method processor interprets these as writes to the class registers on the targeted subchannel. Each subchannel is bound to an **object class** — for example, the 3D Kepler/Maxwell/Pascal class (`NV9097`), the 3D Turing class (`NVC597`), or the compute class (`NVC3C0`). The class name and method names within it define the hardware programming interface.

This model differs fundamentally from AMD's PM4 packet approach (used by RADV) and Intel's batch buffer approach (used by ANV). PM4 packets are variable-length structures with typed opcodes; the reader must understand the packet type before knowing its length. NVIDIA push buffers are simpler at the framing level — every token is either a method header or a data word — but require knowing the class's register layout to understand what a method does.

### Class Headers from open-gpu-doc

Prior to NVIDIA's open-sourcing efforts beginning around 2022, NVIDIA hardware's command encoding was documented entirely through reverse engineering by the Nouveau project, collected in the `envytools` repository. Beginning with the open-gpu-kernel-modules release (Turing and later), NVIDIA released official class header files through the `open-gpu-doc` repository that provide authoritative names and bit-field definitions for all hardware registers.

NVK uses these official class headers, available in `src/nouveau/headers/` after being generated by the Mesa build system from the upstream `open-gpu-doc` sources. Files such as `cl9097.h` define the Kepler/Maxwell/Pascal 3D class; `clc597.h` defines the Turing 3D class; newer generation classes are similarly named. Each header defines constants for every method address and every bit field within that method, allowing driver code to reference hardware registers by name rather than by raw hexadecimal offset.

### The nv\_push API

The mechanism through which NVK writes methods into a push buffer is the **nv_push** API, defined in `src/nouveau/headers/nv_push.h`. The `nv_push` structure tracks a current write pointer into an allocated push buffer segment:

```c
/* Source: src/nouveau/headers/nv_push.h */
struct nv_push {
    uint32_t *start; /* base of the push buffer segment */
    uint32_t *end;   /* exclusive end of the allocated region */
    uint32_t *cur;   /* current write position */
};

static inline void
nv_push_init(struct nv_push *push, uint32_t *start, size_t dw_count)
{
    push->start = start;
    push->end   = start + dw_count;
    push->cur   = start;
}
```

Methods are written using a family of macros generated from the class headers. The two primary forms are:

- **`P_MTHD(push, CLASS, METHOD)`** — emits an increasing-count method header for `METHOD` in `CLASS`, followed by N consecutive data words. Used when programming multiple sequential registers at once (e.g., viewport scale X, Y, Z in three consecutive writes).
- **`P_IMMD(push, CLASS, METHOD, VALUE)`** — emits a single-method header with an inline data word. Used when writing a single register with a compile-time constant or simple computed value.

```c
/* Source: src/nouveau/vulkan/nvk_cmd_draw.c — push buffer method usage */

/* Set viewport scale (3 consecutive registers via P_MTHD) */
P_MTHD(push, NV9097, SET_VIEWPORT_SCALE_X(0));
    P_NV9097_SET_VIEWPORT_SCALE_X(push, 0, fui(fb_width  * 0.5f));
    P_NV9097_SET_VIEWPORT_SCALE_Y(push, 0, fui(fb_height * 0.5f));
    P_NV9097_SET_VIEWPORT_SCALE_Z(push, 0, fui(0.5f));

/* Enable rasteriser (single register via P_IMMD) */
P_IMMD(push, NV9097, SET_RASTER_ENABLE, V_NV9097_SET_RASTER_ENABLE_V_TRUE);
```

The `P_MTHD` / `P_IMMD` names and the expanded per-method macros (`P_NV9097_SET_VIEWPORT_SCALE_X`, etc.) are auto-generated from the class header definitions. Every use of these macros is statically typed: the compiler enforces that the class name and method name are valid, that the value fits within the method's bit field, and that the number of consecutive data words matches the declared count. This compile-time checking is a major correctness improvement over raw hexadecimal register writes.

### Generating class for older hardware

For Kepler through Pascal hardware, NVK uses the reverse-engineered class headers from envytools that were adapted into the open-gpu-doc format when NVIDIA began releasing official documentation. These headers cover the 3D class (`NV9097`), the compute class (`NV_KEPLER_INLINE_TO_MEMORY_B` for inline copies), and the copy engine class (`NV90B5`). The naming convention encodes the class ID in hexadecimal: `NV9097` means class 0x9097. For Turing and later, the officially documented classes are `NVC597` (Turing 3D), `NVC3C0` (Turing compute), and `NVC5B5` (Turing copy engine), with per-generation successors for Ampere, Ada, and Blackwell.

### Comparing NVK's Push Buffer Model with RADV's IB Model

RADV submits work to the GPU via **Indirect Buffer** (IB) packets within AMD's PM4 command stream. A PM4 command stream is a sequence of packet headers with variable-length data; the `PKT3_INDIRECT_BUFFER` packet references a secondary buffer by GPU address, enabling arbitrarily deep chaining. RADV's `radv_cmd_buffer` maintains a list of IBs that are consumed by a single `amdgpu_cs_submit()` call.

NVK's submission model is structurally similar but lexically different. The `DRM_IOCTL_NOUVEAU_EXEC` ioctl accepts an array of `drm_nouveau_exec_push` entries, each specifying a GPU virtual address and byte length. The GPU reads these segments sequentially as if they were a single contiguous push buffer; chaining between segments is handled automatically by the GPU's front-end DMA engine. This means NVK can split a large command buffer across multiple GEM BOs without any special linking instruction — the EXEC ioctl's push array is the link list.

```
RADV (amdgpu):
  vkQueueSubmit → amdgpu_cs_submit → IBs → [PM4 PKT3_INDIRECT_BUFFER → GPU VA]

NVK (nouveau):
  vkQueueSubmit → DRM_IOCTL_NOUVEAU_EXEC → drm_nouveau_exec_push[] → [GPU VA, len]
```

Both models achieve the same result: the GPU reads a sequence of hardware commands from a set of memory buffers. The NVIDIA model is slightly simpler at the submission level because the IB chaining is implicit in the push array rather than requiring inline PKT3 packets.

### The nv_push_dump Debugger

NVK ships a standalone push buffer disassembler, `nv_push_dump`, built as part of the `tools=nouveau` meson target. This tool accepts a raw push buffer binary (e.g., captured from a `NVK_DEBUG=push` run) and decodes each method/data pair into human-readable form using the class header tables. The equivalent tools in AMD's stack are `amdgpu_cs_dump` and `umr`; Intel has the `intel_batchbuffer_dump` infrastructure.

---

## 5. NAK Integration: NIR to SASS in the Mesa Pipeline

NAK (the Nvidia Awesome Kompiler) is covered in depth in Chapter 118. This section focuses specifically on how NVK invokes NAK from within the Mesa pipeline object creation path, how the result is consumed, and how the interface between NVK and NAK is structured across the C/Rust FFI boundary.

### The Compilation Entry Point

Pipeline compilation in NVK begins at `vkCreateGraphicsPipeline` or `vkCreateComputePipeline` time. NVK's frontend (C code in `nvk_pipeline.c`) constructs a NIR shader by calling `spirv_to_nir()` and then applying a sequence of Mesa-common and NVK-specific NIR lowering passes via `nvk_lower_nir()`. The resulting NIR is handed to NAK through a narrow C-callable Rust FFI function:

```c
/* Source: src/nouveau/compiler/nak/lib.rs — C-callable entry point */
/* Defined in Rust, callable from C with bindgen-generated binding */
struct nak_shader_bin *
nak_compile_shader(const nir_shader *nir,
                   bool dump_asm,
                   const struct nak_compiler *nak,
                   nir_variable_mode robust2_modes,
                   const struct nak_fs_key *fs_key);
```

> **Note: needs verification** — The exact parameter list of `nak_compile_shader` evolves across Mesa releases (the `robust2_modes` parameter was renamed in Mesa 26.1 for panvk and may have changed for NAK concurrently). Verify the current signature against `src/nouveau/compiler/nak/lib.rs` in the active Mesa tree before relying on this listing.

The `nak_compiler` object (allocated once per device at `nvk_physical_device` init time) holds the target SM (streaming multiprocessor) version for the current hardware. This single value drives all architectural differences within NAK: SM 75 selects Turing instruction encoding with 128-bit instruction words and uniform register support; SM 80 adds Ampere-specific instructions; SM 86/89 cover Ada Lovelace; SM 90 covers Hopper; SM 120 targets consumer Blackwell (RTX 5000 series, GB202 and later), while SM 100 is the datacenter Blackwell (B100/B200) variant. [Source](https://github.com/NVIDIA/cutlass/issues/2800)

NAK returns a `nak_shader_bin` object containing:
- The compiled SASS binary (byte array)
- The register count (`num_gprs`) — the number of general-purpose registers consumed per thread, which determines warp occupancy
- The shared memory size for compute shaders
- A `nak_qmd` structure for compute dispatches, encoding the thread block configuration and resource requirements

NVK allocates a VRAM region via `nvk_heap` and uploads the SASS binary there. The `nak_qmd` is also uploaded to VRAM and referenced by the pipeline object.

### The NAK-NVK Interface in Context

The FFI boundary between NVK (C) and NAK (Rust) is deliberately narrow: seven public functions, defined with `#[no_mangle]` and `extern "C"` in Rust, and declared in a bindgen-generated C header. This narrowness was a prerequisite for NAK's acceptance into Mesa's C-dominated codebase — a larger FFI surface would have made build system integration and binary compatibility harder to reason about. The functions cover: creating/destroying the compiler, compiling a vertex/fragment/compute shader, and generating a QMD.

This design should be compared with ANV's shader compilation path, which calls into Intel's `brw_compile_*` functions in `src/intel/compiler/` directly as C-to-C calls, and RADV's path, which calls into the ACO compiler through `radv_shader_compile()`. The C/Rust boundary is NVK-specific and reflects NAK's implementation language choice rather than any Vulkan driver design principle.

### Disk Shader Cache Integration

NVK's compiled shader binaries are cached through Mesa's `vk_pipeline_cache` infrastructure. The cache key is a hash of the NIR shader after all frontend passes, combined with the driver version, NAK compiler version, and target SM. On a cache hit, the NAK compilation stage is skipped entirely; NVK retrieves the serialised SASS binary and `nak_qmd` from the cache and uploads them to VRAM directly. This makes repeated application launches with unchanged shaders much faster — the NAK compile cost is paid only once per unique (shader, target, driver version) triple.

The disk cache is stored at `$XDG_CACHE_HOME/mesa_shader_cache/` or `~/.cache/mesa_shader_cache/`. The sub-directory for NVK entries uses a hash of the device's UUID (derived from the NVIDIA GPU's PCI device ID and VRAM size) to prevent cache sharing between different hardware configurations.

---

## 6. GSP-RM Firmware Interaction and Its Implications

### What GSP-RM Is

The **GSP-RM** (GPU System Processor — Resource Manager) is NVIDIA's firmware-based GPU management subsystem, running on a dedicated ARM core (the GSP) embedded within Turing and later GPUs. GSP-RM manages GPU initialisation, power state transitions, clock frequency control, thermal protection, and various hardware resource management tasks that were previously performed directly by the host kernel driver. The nouveau kernel driver's GSP-RM integration is described in detail in Chapter 9.

From NVK's perspective, GSP-RM is an enabler. When GSP-RM is active (Turing and later with the `nouveau` module's `config=NvGspRm=1` option, or automatically on kernels that default to GSP mode), the nouveau kernel driver can:
- Set GPU clocks to rated performance frequencies (reclocking)
- Manage GPU power states correctly
- Allocate hardware channel resources through the firmware interface

Without GSP-RM — which affects all Maxwell, Pascal, and Volta hardware because those generations have no GSP hardware — the GPU runs permanently at its boot/idle clock frequency. The implications for performance are severe: a GTX 1080 (Pascal) without reclocking operates at roughly one-quarter to one-third of its rated compute throughput.

### NVK's Indirection Through Nouveau

NVK userspace (Mesa) does not communicate with GSP-RM directly. The path is:

```
NVK (Mesa userspace)
    ↓ NVKMD function calls
    ↓ DRM ioctls (DRM_IOCTL_NOUVEAU_EXEC, DRM_IOCTL_NOUVEAU_VM_BIND, ...)
Nouveau kernel driver
    ↓ firmware RPCs via the GSP message queue
GSP-RM (on Turing+ GPUs)
    ↓
GPU hardware
```

NVK's command buffers are submitted to nouveau through `DRM_IOCTL_NOUVEAU_EXEC`. The nouveau kernel driver's scheduler queues these against the appropriate hardware channel. For Turing and later, channel creation and management are mediated through GSP-RM firmware RPCs; for older hardware, the kernel driver manages channels directly using register writes (which requires the full set of hardware documentation that GSP-RM abstracted away on newer GPUs).

### What NVK Cannot Do Without GSP-RM

Three capabilities require GSP-RM and are therefore unavailable on Maxwell, Pascal, and Volta hardware:

1. **Reclocking to rated frequencies.** Performance is limited to boot clocks on pre-Turing hardware, a fundamental constraint that NVK cannot overcome from userspace.

2. **Display output.** On Turing and later with GSP-RM, the display pipeline is managed through GSP-RM firmware. On older hardware, the nouveau kernel driver handles display directly through KMS. NVK is a Vulkan driver and has no display path; display is separate from the 3D/compute rendering path. However, the GSP-RM dependency affects whether nouveau can serve as a complete graphics stack on a given GPU.

3. **Hardware video decode at full capability.** The experimental video decode path in NVK (H.264/H.265, described in Section 9) requires kernel-level support for NVDEC channel allocation that is only available when GSP-RM is active.

### The NVK_I_WANT_A_BROKEN_VULKAN_DRIVER Variable

NVK exposes hardware not considered production-ready behind an opt-in environment variable: `NVK_I_WANT_A_BROKEN_VULKAN_DRIVER=1`. Setting this variable enables enumeration of GPUs whose NVK support is experimental or known incomplete — primarily Kepler hardware (Vulkan 1.2, full performance not yet validated) and the video decode path on all hardware. This mechanism prevents unsuspecting users from picking up a subtly broken driver while still allowing early adopters and testers to exercise the code.

---

## 7. Window System Integration: Kopper, DMA-BUF, and Wayland Explicit Sync

### Mesa's WSI Architecture

Every Mesa Vulkan driver uses the shared WSI (Window System Integration) layer in `src/vulkan/wsi/` rather than implementing swapchain, surface, and present operations independently. This shared layer handles:
- Surface creation for X11 (`VK_KHR_xlib_surface`, `VK_KHR_xcb_surface`) via DRI3 and the X11 Present extension
- Surface creation for Wayland (`VK_KHR_wayland_surface`) via `zwp_linux_dmabuf_v1`
- Swapchain image management: allocating swapchain images, presenting to the compositor, managing acquire/release fences
- Format and modifier negotiation via `zwp_linux_dmabuf_feedback_v1`

NVK participates in the shared WSI layer through `nvk_wsi.c`, which provides the device-specific callbacks that the shared layer needs to allocate images with device-local memory, export them as DMA-BUF file descriptors, and import DMA-BUF fences from the compositor.

### Kopper: The Mesa OpenGL-on-Vulkan WSI Bridge

**Kopper** is a Mesa WSI abstraction layer that allows OpenGL applications (via Zink, the OpenGL-on-Vulkan Gallium driver) to use a Vulkan driver's swapchain implementation for window system presentation. From NVK's perspective, Kopper is relevant because — since Mesa 25.1 — Turing and later NVIDIA GPUs use NVK+Zink as the default OpenGL implementation. Every OpenGL application running on Turing+ via the open stack ultimately presents frames through NVK's Vulkan swapchain via Kopper.

Kopper's integration with NVK is mediated through the Mesa WSI layer: Zink creates a `VkSurfaceKHR` via `vkCreateWaylandSurfaceKHR` or `vkCreateXlibSurfaceKHR`, then creates a `VkSwapchainKHR` using NVK's swapchain implementation. Swapchain images are allocated as device-local Vulkan images, exported as DMA-BUF file descriptors, and imported by the compositor. This path is identical to what a native Vulkan application would do; Kopper is simply the bridge that feeds OpenGL's implicit present semantics into Vulkan's explicit swapchain model.

### DMA-BUF Import and Export

NVK allocates swapchain images in VRAM and exports them as DMA-BUF file descriptors through the standard Mesa WSI path. The DMA-BUF export calls `DRM_IOCTL_PRIME_HANDLE_TO_FD` on the GEM BO underlying the swapchain image's `VkDeviceMemory`. The compositor (e.g., Mutter/GNOME, KWin/KDE, or Sway) imports these DMA-BUF file descriptors via `zwp_linux_dmabuf_v1`, creates GBM/EGL images from them, and scans them out to the display or composites them into the scene.

Format and modifier negotiation follows the `zwp_linux_dmabuf_feedback_v1` protocol: the compositor advertises which format/modifier combinations it can scan out or texture from efficiently, and NVK's WSI layer selects among the advertised options, preferring scanout-capable formats (which avoid a composition copy). NVK advertises the same DRM format modifiers as the nouveau kernel driver uses for display; for Turing and later, this includes `DRM_FORMAT_MOD_NVIDIA_BLOCK_LINEAR_2D` variants.

### Wayland Explicit Synchronisation

Mesa 24.1 landed Vulkan explicit synchronisation support for Wayland via the `wp_linux_drm_syncobj_v1` protocol. NVK participates in this as a Vulkan driver: when the Wayland compositor supports explicit sync, NVK exports a `drm_syncobj` timeline point representing "this frame's rendering is complete" alongside the DMA-BUF image. The compositor waits on this timeline point before compositing or scanning out the image, eliminating the implicit synchronisation overhead that was previously required.

From NVK's implementation perspective, this is a straightforward extension of the existing `drm_syncobj` timeline semaphore support: NVK creates a timeline syncobj, signals it at queue submission time, and exports its file descriptor to the Wayland surface acquire/release mechanism. The kernel-level `drm_syncobj` can be exported as a sync file (`DRM_IOCTL_SYNCOBJ_EXPORT_SYNC_FILE`) and imported by any DRM client, including the compositor's KMS path. This fully explicit GPU-timeline synchronisation eliminates the last remaining implicit GPU-CPU synchronisation point in the NVK+Wayland frame delivery path.

---

## 8. Timeline Semaphores and the drm\_syncobj Model

### The DRM_IOCTL_NOUVEAU_EXEC Interface

Queue submission in NVK flows through `DRM_IOCTL_NOUVEAU_EXEC`, the kernel ioctl that schedules a set of push buffer segments for execution on a GPU channel, with explicit synchronisation specified as arrays of `drm_syncobj` handles. The ioctl structure is:

```c
/* Source: include/uapi/drm/nouveau_drm.h */
struct drm_nouveau_exec_push {
    __u64 va;      /* GPU virtual address of push buffer segment */
    __u32 va_len;  /* Length in bytes */
    __u32 flags;
};

struct drm_nouveau_sync {
    __u32 flags;
#define DRM_NOUVEAU_SYNC_SYNCOBJ          0x0  /* binary syncobj */
#define DRM_NOUVEAU_SYNC_TIMELINE_SYNCOBJ 0x1  /* timeline syncobj */
    __u32 handle;          /* DRM syncobj kernel handle */
    __u64 timeline_value;  /* For TIMELINE_SYNCOBJ: the timeline point */
};

struct drm_nouveau_exec {
    __u32 channel;      /* GPU channel handle */
    __u32 push_count;   /* Number of push segments */
    __u32 wait_count;   /* Number of wait sync entries */
    __u32 sig_count;    /* Number of signal sync entries */
    __u64 wait_ptr;     /* Pointer to wait drm_nouveau_sync[] */
    __u64 sig_ptr;      /* Pointer to signal drm_nouveau_sync[] */
    __u64 push_ptr;     /* Pointer to drm_nouveau_exec_push[] */
};
```

The semantic contract is: the GPU scheduler waits for all `wait_ptr` syncobjs to reach their specified timeline point (or to be signalled, for binary syncobjs), then schedules the push segments for execution, then signals all `sig_ptr` syncobjs when the GPU has finished processing the last push segment. This maps precisely to `VkSubmitInfo2`'s `pWaitSemaphoreInfos`/`pSignalSemaphoreInfos` arrays.

### Mapping Vulkan Synchronisation to drm_syncobj

NVK's `nvk_queue_submit()` translates `VkSubmitInfo2` directly to `DRM_IOCTL_NOUVEAU_EXEC` parameters:

| Vulkan primitive | drm_syncobj representation |
|---|---|
| `VkFence` | Binary `drm_syncobj`; signalled on GPU job completion |
| Binary `VkSemaphore` | Binary `drm_syncobj`; signal/wait via `sig_ptr`/`wait_ptr` |
| Timeline `VkSemaphore` at point P | Timeline `drm_syncobj` with `timeline_value = P` |
| `vkQueueSubmit` wait | `wait_ptr` entry in `DRM_NOUVEAU_EXEC` |
| `vkQueueSubmit` signal | `sig_ptr` entry in `DRM_NOUVEAU_EXEC` |

Timeline semaphores (`VK_SEMAPHORE_TYPE_TIMELINE`) are supported natively through the `DRM_NOUVEAU_SYNC_TIMELINE_SYNCOBJ` flag. NVK creates a `drm_syncobj` timeline for each Vulkan timeline semaphore and passes the specific timeline point value as `timeline_value` in the wait or signal entry. The nouveau kernel driver's DRM scheduler handles the wait-for-timeline-point logic in kernel space.

### Comparison with RADV's Timeline Semaphore Path

RADV implements timeline semaphores through the `amdgpu_cs_submit()` path using `AMDGPU_CHUNK_ID_SYNCOBJ_TIMELINE_WAIT` and `AMDGPU_CHUNK_ID_SYNCOBJ_TIMELINE_SIGNAL` chunk types in the CS submission structure. The underlying mechanism is identical — both use the kernel's DRM timeline syncobj infrastructure — but the framing differs at the winsys level: RADV passes timeline points as chunks within a structured CS submission, while NVK passes them as arrays of `drm_nouveau_sync` entries within `DRM_NOUVEAU_EXEC`.

ANV (Intel Vulkan) uses a similar explicit sync model through the `i915` or `xe` kernel drivers, depending on which backend is active. The common theme across all three drivers is that Vulkan's explicit synchronisation model maps naturally onto the DRM `drm_syncobj` timeline infrastructure, which was designed precisely for this purpose.

### Pipeline Barriers

Within a single command buffer, pipeline barriers (`vkCmdPipelineBarrier`, `vkCmdPipelineBarrier2`) translate to GPU memory barrier methods in the push buffer. On NVIDIA hardware, this means emitting a `NV9097_INVALIDATE_TEXTURE_DATA_CACHE` method (for texture caches), `NV9097_INVALIDATE_SHADER_CACHES` (for L1 shader instruction/data caches), or a compute-specific cache invalidation sequence. The specific methods emitted depend on the source and destination pipeline stages and access types in the `VkMemoryBarrier2` structure.

NVK uses the same approach as other Mesa drivers: it translates the access flags and stage flags into a set of hardware cache flush and invalidate operations. Because NVIDIA's cache hierarchy differs from AMD's and Intel's, NVK's barrier implementation in `nvk_cmd_buffer.c` is entirely specific to the NVIDIA hardware model.

---

## 9. Conformance Timeline and Known Gaps

### Generation-Stratified Conformance

NVK's conformance status as of mid-2026 is generation-stratified, reflecting both hardware capability differences and the order in which architecture support was implemented:

| GPU Generation | Release | Vulkan Version | Notes |
|---|---|---|---|
| Turing (SM 75, RTX 2000/GTX 1600) | Mesa 23.3 (Aug 2023) | Vulkan 1.4 | Primary target; GSP-RM reclocking |
| Ampere (SM 80/86, RTX 3000) | Mesa 24.x | Vulkan 1.4 | Full performance with GSP-RM |
| Ada Lovelace (SM 89, RTX 4000) | Mesa 24.x | Vulkan 1.4 | Full performance with GSP-RM |
| Blackwell (SM 120, RTX 5000) | Mesa 25.2 (Aug 2025) | Vulkan 1.4 | Consumer Blackwell (GB202+); GSP-RM required |
| Volta (SM 70, TITAN V) | Mesa 25.1 (May 2025) | Vulkan 1.4 | No reclocking (no GSP-RM) |
| Pascal (SM 61/62, GTX 1000) | Mesa 25.1 (May 2025) | Vulkan 1.4 | No reclocking (no GSP-RM) |
| Maxwell (SM 52/53, GTX 900/750) | Mesa 25.1 (May 2025) | Vulkan 1.4 | No reclocking (no GSP-RM) |
| Kepler (SM 30–37, GTX 600/700) | Mesa 25.2 (Aug 2025) | Vulkan 1.2 | Hardware lacks vulkanMemoryModel |
| Hopper (SM 90, H100) | Mesa 25.2 | Vulkan 1.4 | Experimental; no consumer GPUs |

Kepler is capped at Vulkan 1.2 because the hardware lacks the memory ordering semantics required by Vulkan's `vulkanMemoryModel` feature, which is mandatory for Vulkan 1.3 and later. [Source](https://www.collabora.com/news-and-blog/news-and-events/nvk-enabled-for-maxwell,-pascal,-and-volta-gpus.html)

### Extension Coverage

As of Mesa 26.0, NVK implements the following key extensions (non-exhaustive):

**Core Vulkan 1.4 mandatory extensions** (all present): `VK_KHR_dynamic_rendering`, `VK_KHR_synchronization2`, `VK_KHR_maintenance4`, `VK_KHR_maintenance5`, `VK_EXT_extended_dynamic_state` (all three versions), `VK_KHR_push_descriptor`, `VK_KHR_buffer_device_address`, `VK_EXT_descriptor_buffer`, `VK_KHR_pipeline_library`, `VK_EXT_mesh_shader`, `VK_KHR_cooperative_matrix` (Turing+).

**Additional extensions in NVK**: `VK_EXT_discard_rectangles`, `VK_EXT_shader_uniform_buffer_unsized_array`, `VK_KHR_robustness2`, `VK_KHR_maintenance10`, `VK_KHR_pipeline_binary`, `VK_KHR_external_semaphore_fd`, `VK_KHR_external_memory_fd`, `VK_MESA_image_alignment_control`.

**Not yet implemented (mid-2026)**:
- `VK_KHR_ray_tracing_pipeline` and `VK_KHR_acceleration_structure` — Ray tracing (BVH construction and traversal using Turing's RT cores) is the highest-priority missing feature. Development is underway but not yet merged as of mid-2026. [Source](https://www.phoronix.com/news/NVK-Status-Update-2025)
- `VK_KHR_video_decode_queue` (stable) — Hardware H.264/H.265 video decode via NVDEC is available experimentally behind `NVK_I_WANT_A_BROKEN_VULKAN_DRIVER=1` and requires Linux 6.12+. AV1 decode is not yet implemented. [Source](https://blogs.igalia.com/scerveau/vulkan-video-with-nvk-driver/)
- `VK_NV_*` proprietary NVIDIA extensions — Extensions from NVIDIA's proprietary stack (DLSS, Reflex, NGX) are not part of NVK, which is a clean-slate open-source implementation without NVIDIA proprietary middleware.

### DXVK and VKD3D-Proton Compatibility

NVK provides the Vulkan implementation that DXVK and VKD3D-Proton (the DirectX-to-Vulkan translation layers in Valve's Proton compatibility system for Windows games) run on top of when using the open NVIDIA driver stack. From Mesa 24.1 onward, DXVK (D3D9/10/11) works out of the box against NVK. VKD3D-Proton (D3D12) also runs on NVK, though the absence of ray tracing means D3D12 games that require DXR (DirectX Raytracing) cannot use their hardware RT paths. Most games fall back gracefully to software RT or simply disable RT effects, remaining playable. [Source](https://www.phoronix.com/news/Mesa-2024-Highlights)

The `NVK_EXPERIMENTAL` environment variable enables experimental features including initial DLSS support (`NVK_EXPERIMENTAL=dlss`), which provides basic NVIDIA DLSS 1.x support through a compatibility shim. DLSS 2.x and later (which use the NGX machine learning framework requiring NVIDIA's proprietary SDK) are not supported.

---

## 10. Building and Testing NVK

### Meson Configuration

NVK is built as part of Mesa by including `nouveau` in the `--vulkan-drivers` meson option. The minimum kernel version for NVK functionality is Linux 6.6 (for `DRM_IOCTL_NOUVEAU_EXEC` and `DRM_IOCTL_NOUVEAU_VM_BIND`). A kernel older than 6.6 will cause NVK to fail to enumerate any physical devices at runtime.

```bash
# Minimal NVK build (Vulkan only, no GL):
meson setup builddir \
    --prefix=/usr \
    -Dvulkan-drivers=nouveau \
    -Dgallium-drivers= \
    -Dbuildtype=debugoptimized

ninja -C builddir
```

To build NVK with Zink (OpenGL via Vulkan, default for Turing+ since Mesa 25.1):

```bash
meson setup builddir \
    --prefix=/usr \
    -Dvulkan-drivers=nouveau \
    -Dgallium-drivers=zink \
    -Dbuildtype=debugoptimized

ninja -C builddir
```

To build with the diagnostic tools including `nv_push_dump`:

```bash
meson setup builddir \
    --prefix=/usr \
    -Dvulkan-drivers=nouveau \
    -Dgallium-drivers=zink \
    -Dtools=nouveau \
    -Dbuildtype=debugoptimized

ninja -C builddir
# nv_push_dump is now at builddir/src/nouveau/tools/nv_push_dump
```

### NAK Build Dependencies

NAK is written in Rust and requires a Rust toolchain installed on the build host. Mesa pins a minimum Rust version (MSRV) that must be satisfied; the current MSRV as of Mesa 26.0 is 1.82.0. The Mesa build system (meson) detects the Rust toolchain via `rustc` and `cargo` in `$PATH` and compiles NAK as a Rust static library that is linked into the `libvulkan_nouveau.so` shared library at link time.

On distributions that do not ship a sufficiently recent Rust toolchain, the recommended approach is to install from [rustup](https://rustup.rs/) and then ensure the meson build uses the rustup-managed `rustc`:

```bash
# Install or update Rust toolchain
rustup update stable

# Verify Rust version meets Mesa's MSRV
rustc --version
```

### Environment Variables for Debugging and Testing

NVK exposes two families of environment variables for debugging:

**`NVK_DEBUG`** — Controls driver-level diagnostic output and behaviour modification:

| Flag | Effect |
|---|---|
| `push` | Dump all push buffer contents to stderr at submit time |
| `push_sync` | Insert synchronisation points after every push buffer submission (requires `push`) |
| `zero_memory` | Zero all allocated BOs before use (catches uninitialised-memory bugs) |
| `trash_memory` | Fill BOs with a recognisable pattern on free (catches use-after-free bugs) |
| `vm` | Log all virtual memory bind/unbind operations |
| `no_cbuf` | Disable constant buffer optimisations (useful for shader debugging) |
| `gart` | Force all BO allocations to GART/system memory instead of VRAM |
| `coherent` | Force all allocations to use coherent (non-cached) memory |

Multiple flags can be combined with commas: `NVK_DEBUG=push,push_sync,zero_memory`.

**`NAK_DEBUG`** — Controls NAK compiler diagnostic output:

| Flag | Effect |
|---|---|
| `print` | Print the final NAK IR (before binary encoding) to stderr |
| `serial` | Force all shader instructions to execute serially (no ILP) |
| `spill` | Minimise register usage by aggressively spilling (tests the spiller) |
| `annotate` | Annotate the output assembly with register allocation decisions |

**`NVK_EXPERIMENTAL`** — Enables experimental features:
- `dlss` — Enable DLSS 1.x compatibility shim
- `dlss_backwards_compat` — Enable backward compatibility for older DLSS binaries

**`NVK_I_WANT_A_BROKEN_VULKAN_DRIVER`** — Set to `1` or `true` to enable poorly-tested hardware and experimental features (Kepler, Hopper, video decode). This bypasses NVK's self-protection logic that would otherwise prevent the driver from enumerating on these configurations.

### Running dEQP-VK Against NVK

The standard conformance test suite for Vulkan is dEQP-VK (from the VK-GL-CTS project, maintained by Khronos). NVK's CI uses `deqp-runner` to execute tests and compare results against a per-generation expected pass list. To run dEQP-VK manually against NVK on a Turing GPU:

```bash
# Build dEQP-VK (from VK-GL-CTS):
git clone https://github.com/KhronosGroup/VK-GL-CTS
cd VK-GL-CTS
python3 external/fetch_sources.py
cmake -B build -GNinja -DDEQP_TARGET=default
ninja -C build

# Install NVK (or use LD_LIBRARY_PATH / VK_ICD_FILENAMES to use the local build):
export VK_ICD_FILENAMES=/path/to/mesa/build/src/nouveau/vulkan/nouveau_icd.x86_64.json

# Run the Vulkan 1.4 conformance test group:
./build/modules/vulkan/deqp-vk \
    --deqp-case=dEQP-VK.* \
    --deqp-log-filename=nvk-turing-results.qpa

# Or use deqp-runner for parallel execution with expected-failures handling:
deqp-runner run \
    --deqp ./build/modules/vulkan/deqp-vk \
    --caselist dEQP-VK.caselist.txt \
    --skips src/nouveau/ci/deqp-nvk-skips.txt \
    --baseline src/nouveau/ci/deqp-nvk-turing.fails.csv \
    --output nvk-results/
```

NVK's per-generation expected failure lists live in `src/nouveau/ci/` in the Mesa repository. Each file lists tests that are known to fail on that generation with a reason code. A merge request that introduces new failures will be caught by CI, which runs the full dEQP-VK suite against real hardware via Mesa's freedesktop.org CI infrastructure.

### RADV_DEBUG Analogues

NVK does not use `RADV_DEBUG` (which is AMD-specific) but the `NVK_DEBUG` flags cover the same diagnostic use cases. The mapping is approximate:

| RADV_DEBUG flag | NVK_DEBUG equivalent |
|---|---|
| `nodcc` | n/a (NVIDIA uses different compression mechanism) |
| `hang` | `push_sync` (isolates hanging push buffers) |
| `nofastclears` | n/a |
| `allbos` | `zero_memory` (different purpose but similar intent) |
| `nomemorycache` | `gart` (avoids VRAM, tests system memory paths) |

---

## Integrations

This chapter is part of the NVK/Nouveau sub-cluster of chapters. The following cross-references locate related content:

**[Chapter 7 — Reverse Engineering NVIDIA: History and Methodology](../../part-03-nouveau-story/ch07-reverse-engineering-nvidia.md)**: The envytools register XML that NVK's class headers derive from traces directly to the reverse engineering work described here. The hardware class identifiers (`NV9097`, `NVC597`) and method names in NVK's push buffer encoding come from this lineage.

**[Chapter 8 — The Nouveau Kernel Driver: nvkm Architecture](../../part-03-nouveau-story/ch08-nouveau-kernel-driver.md)**: NVK's NVKMD abstraction layer wraps the DRM ioctls exposed by the nouveau kernel driver described in that chapter. The `DRM_IOCTL_NOUVEAU_EXEC`, `DRM_IOCTL_NOUVEAU_VM_BIND`, and `DRM_NOUVEAU_GEM_NEW` ioctls are the kernel API that NVKMD translates NVKMD calls into.

**[Chapter 9 — GSP-RM, Firmware, and the nvidia-open Connection](../../part-03-nouveau-story/ch09-gsp-rm-firmware.md)**: Section 6 of this chapter summarises GSP-RM's effect on NVK's performance envelope. Chapter 9 provides the full GSP-RM architecture: the ARM core, the firmware binary, the message queue protocol between the host kernel driver and the GSP.

**[Chapter 10a — Nova: The Rust NVIDIA Kernel Driver](../../part-03-nouveau-story/ch10a-nova-rust-nvidia-driver.md)**: The NVKMD abstraction layer in NVK was designed with a Nova backend in mind. Nova is a new Rust-language NVIDIA kernel driver being developed as an alternative to nouveau; when Nova reaches maturity, an NVKMD backend for Nova would allow NVK to run against either kernel driver without any change to NVK's Vulkan code.

**[Chapter 10b — NVK: Building a Vulkan Driver from Scratch](../../part-03-nouveau-story/ch10b-nvk-vulkan-driver.md)**: The companion chapter to this one. Chapter 10b covers the object model (`nvk_instance`, `nvk_physical_device`, `nvk_device`), the GEM BO lifecycle and `nvk_heap` suballocator in detail, the full NIR lowering pipeline through `nvk_lower_nir()`, descriptor set implementation, command buffer push buffer recording, and the `DRM_NOUVEAU_EXEC` synchronisation model with code listings. Readers who want deep implementation detail on any of those topics should read Chapter 10b alongside this chapter.

**[Chapter 11 — Display, Reclocking, and Power Management](../../part-03-nouveau-story/ch11-display-reclocking-power.md)**: The reclocking limitation described in Section 6 of this chapter — NVK cannot improve GPU frequency on pre-Turing hardware — is explained at the kernel level in Chapter 11, which covers the nouveau clock management subsystem, the PMU firmware path, and the GSP-RM reclocking interface.

**[Chapter 16 — Mesa's Vulkan Common Infrastructure](../../part-04-mesa-architecture/ch16-mesa-vulkan-common.md)**: NVK uses the shared Vulkan runtime infrastructure from `src/vulkan/runtime/`: the `vk_render_pass` lowering layer, `vk_pipeline_cache`, `vk_command_buffer`, and the physical and logical device base types. Chapter 16 describes these foundations.

**[Chapter 18 — Vulkan Drivers](../ch18-vulkan-drivers.md)**: The companion survey of Mesa's other production Vulkan drivers — RADV (AMD) and ANV (Intel) — with which NVK's architecture can be compared. Sections 3 and 4 of Chapter 18 detail the push buffer models for AMD (PM4/IB) and Intel (batch buffers/genxml), enabling the comparison with NVK's nv_push model made in Section 4 of this chapter.

**[Chapter 118 — NAK: The Rust Shader Compiler for NVIDIA GPUs](../../part-03-nouveau-story/ch118-nak-rust-shader-compiler.md)**: NAK is covered in complete detail in Chapter 118: the SSA IR, register allocator, instruction scheduler, ISA encoders for each GPU generation, and the Rust-to-C FFI interface. Section 5 of this chapter provides the NVK-side view of that interface — how NVK invokes `nak_compile_shader()`, what it receives back, and how the SASS binary is uploaded and referenced — but refers readers to Chapter 118 for NAK internals.

---

## References

- [NVK Mesa Documentation](https://docs.mesa3d.org/drivers/nvk.html) — Official Mesa docs for NVK: supported GPUs, kernel requirements, environment variables.
- [NVK Source Tree](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/nouveau/vulkan) — `src/nouveau/vulkan/` in the Mesa repository.
- [NVK External Hardware Docs](https://docs.mesa3d.org/drivers/nvk/external_hardware_docs.html) — Links to open-gpu-doc class headers, envytools, envyhooks, nv_push_dump, and ISA references.
- [Introducing NVK — Collabora Blog](https://www.collabora.com/news-and-blog/news-and-events/introducing-nvk.html) — Faith Ekstrand's original NVK announcement post (October 2022).
- [NVK Has Landed — Collabora Blog](https://www.collabora.com/news-and-blog/news-and-events/nvk-has-landed.html) — August 2023 announcement of NVK's Mesa merge.
- [NVK Enabled for Maxwell, Pascal, and Volta — Collabora Blog](https://www.collabora.com/news-and-blog/news-and-events/nvk-enabled-for-maxwell,-pascal,-and-volta-gpus.html) — April 2025 coverage of Vulkan 1.4 conformance for older generations.
- [Mesa 25.2 Brings New Hardware Support for Nouveau Users — Collabora Blog](https://www.collabora.com/news-and-blog/news-and-events/mesa-25.2-brings-new-hardware-support-for-nouveau-users.html) — Blackwell and Kepler enablement announcement.
- [Vulkan Video with the NVK Driver — Igalia Blog](https://blogs.igalia.com/scerveau/vulkan-video-with-nvk-driver/) — Technical description of the experimental H.264/H.265 decode path and its kernel requirements.
- [nvk: the kernel changes needed — Dave Airlie's Blog](https://airlied.blogspot.com/2023/08/nvk-kernel-changes-needed.html) — Rationale and design of `DRM_IOCTL_NOUVEAU_EXEC` and `DRM_IOCTL_NOUVEAU_VM_BIND`.
- [NVIDIA open-gpu-doc](https://github.com/NVIDIA/open-gpu-doc) — Official NVIDIA hardware class header files used by NVK for push buffer encoding.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
