# Chapter 10: NVK: Building a Vulkan Driver from Scratch

> **Part**: Part III — The Nouveau Story
> **Audience**: Both — systems developers learn the internals of a new-generation Mesa Vulkan driver implementation; application developers learn what the NVK driver means for NVIDIA hardware support and what Vulkan guarantees it provides
> **Status**: First draft — 2026-06-06

## Table of Contents

- [Overview](#overview)
- [1. Motivation: Starting Fresh](#1-motivation-starting-fresh)
- [2. Object Model and Memory Heaps](#2-object-model-and-memory-heaps)
- [3. Shader Compilation: SPIR-V to NVIDIA ISA](#3-shader-compilation-spir-v-to-nvidia-isa)
- [4. Pipeline Objects and Command Recording](#4-pipeline-objects-and-command-recording)
- [5. Synchronisation on the Nouveau Channel Model](#5-synchronisation-on-the-nouveau-channel-model)
- [6. Vulkan Feature Coverage and Conformance Status](#6-vulkan-feature-coverage-and-conformance-status)
- [7. Lessons for New Mesa Vulkan Driver Authors](#7-lessons-for-new-mesa-vulkan-driver-authors)
- [Integrations](#integrations)
- [References](#references)

---

## Overview

NVK is the most technically ambitious project to emerge from the Nouveau ecosystem since the original kernel driver. Begun in 2022 by Faith Ekstrand — previously the lead developer of Intel's ANV Vulkan driver within Mesa — NVK is a clean-slate Vulkan driver for NVIDIA hardware built entirely within Mesa's modern Vulkan common infrastructure. It is not a wrapper over the legacy Gallium-based nouveau driver, nor an adaptation of NVIDIA's proprietary stack: it was designed from scratch with Vulkan semantics as the primary design constraint, and the entire Mesa GPU driver community as the intended audience.

This chapter examines NVK along three dimensions. First, it explains the architectural motivations: why starting over was preferable to extending the legacy GL driver, and what the design decisions were around object management, memory layout, and shader compilation. Second, it provides a detailed technical walkthrough of NVK's implementation — how the Vulkan object model maps to nouveau kernel objects, how shader compilation through SPIR-V to NIR to the NAK (Nouveau Assembler Kit) backend to NVIDIA ISA works, and how Vulkan synchronisation is grounded in the `drm_syncobj` kernel primitive. Third, it extracts transferable lessons applicable to anyone writing a new Mesa Vulkan driver: NVK was explicitly designed as a reference implementation for Mesa's Vulkan common infrastructure.

By the end of this chapter, the reader will understand the full NVK architecture end-to-end, be able to navigate the NVK source tree effectively, know the current conformance status spanning Kepler through Blackwell GPUs, and carry a mental model of Mesa Vulkan driver development that applies beyond NVIDIA hardware entirely.

---

## 1. Motivation: Starting Fresh

### The Legacy Driver and Its Constraints

To understand why NVK was written from scratch, it is necessary to understand what it replaced — and why replacement was preferable to evolution. The legacy Nouveau Gallium driver, located in `src/gallium/drivers/nouveau/` in the Mesa tree, was written in an era before NIR, before Vulkan, before any of the modern Mesa infrastructure that now makes new driver development tractable. Its shader compiler, `nv50_ir/`, is a bespoke IR whose design predates Mesa's common NIR framework entirely. The path from GLSL source to NVIDIA machine code inside the legacy driver runs entirely through this proprietary IR, with no connection to NIR's optimisation passes, no shared infrastructure with other Mesa drivers, and no obvious seam at which a NIR-consuming Vulkan front end could be grafted on.

The structural problem runs deeper than the compiler. The Gallium state tracker encodes OpenGL-era concepts throughout its interface: render target bindings, the fixed-function blend state machine, per-stage texture unit management. These concepts are not merely inconvenient for a Vulkan driver — they are actively counterproductive. Implementing Vulkan over Gallium means translating Vulkan's explicit, application-controlled state model into Gallium's implicit, driver-managed model and then immediately unwinding that translation on the hardware side. Every layer of indirection adds latency, increases correctness risk, and makes debugging harder.

The practical consequence was that implementing Vulkan over the Gallium nouveau driver would have required reimplementing most of `nv50_ir/` to consume NIR, rearchitecting the Gallium state tracker to handle Vulkan's render pass model, and then layering Vulkan's synchronisation primitives onto a subsystem designed for OpenGL's implicit sync. This was, in Faith Ekstrand's assessment, approximately as much work as writing a new driver — with the added disadvantage of inheriting the legacy driver's technical debt.

### Why NVK Rather Than Vulkan-over-Gallium

The decisive advantage of the clean-sheet approach was access to Mesa's modern Vulkan common infrastructure, located in `src/vulkan/` in the Mesa tree. This infrastructure, developed primarily through the ANV (Intel) and RADV (AMD) drivers, handles a surprising amount of the Vulkan specification on behalf of any driver that chooses to use it. The `vk_render_pass` implementation handles legacy `VkRenderPass` objects by lowering them to the dynamic rendering operations that modern hardware actually supports. The `vk_pipeline_cache` provides a correct, serialisable pipeline cache from day one. Common extension implementations for `VK_KHR_maintenance4`, `VK_KHR_synchronization2`, and the extended dynamic state family are available to any driver that registers the appropriate function pointers.

NVK was designed to exercise every piece of this infrastructure. Faith Ekstrand's stated goal was not merely to produce a working NVIDIA Vulkan driver but to validate and improve the common infrastructure simultaneously, so that subsequent driver authors — for embedded GPUs, for new architectures, for experimental hardware — would find a more capable foundation waiting for them. This dual purpose shaped NVK's architecture in observable ways: the driver is deliberately minimal in the places where the common infrastructure already handles a concern, and deliberately explicit in the places where NVIDIA hardware requires driver-specific logic.

The shader compilation path illustrates this cleanly. With NIR as the common IR, the path from SPIR-V to GPU binary becomes: `spirv_to_nir()` (from `src/compiler/spirv/`, shared by every Mesa driver) followed by a set of NIR optimisation passes (also shared), followed by NVK-specific lowering passes, followed by the NAK backend (NVIDIA-specific). The legacy `nv50_ir/` IR is entirely absent. Each stage of the pipeline has clear inputs, clear outputs, and clear ownership, making the compiler tractable to debug and extend without touching unrelated parts of the driver.

### Hardware Target Range and the GSP-RM Dependency

NVK targets NVIDIA GPUs from Kepler (GK100, 2012) through Ada Lovelace (AD102, 2022) and, as of Mesa 25.2, extending to Blackwell (GB202, 2025). The driver achieves different Vulkan version conformance across this range: Kepler reaches Vulkan 1.2 due to hardware limitations (the lack of `vulkanMemoryModel` semantics in the older memory subsystem), Maxwell through Volta reach Vulkan 1.3, and Turing through Blackwell achieve full Vulkan 1.4 conformance.

The practical performance story, however, is significantly more stratified. Older GPUs — Kepler, Maxwell, Pascal — lack support for GSP-RM (GPU System Processor — Resource Manager), the firmware-based GPU management subsystem described in Chapter 9. Without GSP-RM, the nouveau kernel driver cannot perform GPU reclocking, which means these GPUs run at their boot clocks rather than at their rated performance clocks. On a Maxwell GPU this typically means operating at roughly one-third to one-half of rated performance. NVK runs correctly on these GPUs and passes conformance, but the performance ceiling is fundamentally limited by the kernel, not by the Mesa driver.

For Turing and newer GPUs, GSP-RM is supported, reclocking works, and NVK delivers the full compute and rasterisation throughput of the hardware. This is the primary production target for NVK today, and it is where the performance story versus NVIDIA's proprietary driver is most meaningful.

The hardware breadth — spanning a decade of NVIDIA microarchitectures — was also a deliberate design choice. A driver that correctly handles Kepler through Blackwell must have a principled abstraction layer between the Vulkan API and the hardware differences across generations. NVK's architecture delivers this: the core object model, memory allocation path, and synchronisation primitives are hardware-generation-agnostic, while the NAK compiler backend encodes generation-specific instruction encoding and register file differences in clearly bounded modules.

---

## 2. Object Model and Memory Heaps

### Source Location and Object Hierarchy

NVK's Mesa source lives in `src/nouveau/vulkan/`. The top-level objects follow the naming convention of every Mesa Vulkan driver: `nvk_instance`, `nvk_physical_device`, `nvk_device`. Each of these wraps the corresponding Mesa Vulkan common object (`vk_instance`, `vk_physical_device`, `vk_device`) as its first member, enabling the pervasive Mesa pattern of safe upcasting via `container_of()`. The common object carries the function table dispatch, the enabled extension set, and other cross-driver state; the `nvk_` wrapper adds NVIDIA-specific fields.

The `nvk_instance` object holds global driver state: the list of enumerated physical devices, debug flag parsing (from the `NVK_DEBUG` environment variable), and DRI option overrides. Debug flags include `push_dump` (dump push buffer contents to stderr), `push_sync` (insert synchronisation points after every push buffer submission for GPU fault isolation), `zero_memory` (zero all allocated BOs to catch uninitialised-memory bugs), and `trash_memory` (fill BOs with a pattern to surface use-after-free errors). These flags are indispensable during driver development and reflect lessons learned from the ANV and RADV development histories.

The `nvk_physical_device` object represents one enumerable GPU. It holds the GPU's capability flags, the memory type and heap table, the queue family descriptions, and the NVKMD (Nouveau Kernel Mode Driver) handle used for all subsequent kernel interactions. NVKMD is NVK's abstraction layer over the kernel UAPI — it wraps GPU initialisation, buffer object allocation, virtual address space management, and queue submission behind a function-pointer interface, enabling the driver to run against both the in-tree DRM kernel driver and, in principle, alternative kernel back ends.

### Memory Types and Heaps

Vulkan's explicit memory model requires every driver to enumerate its memory heaps and memory types at physical device initialisation time. NVIDIA hardware presents three logical memory regions to the driver, each with distinct performance characteristics that map onto distinct Vulkan memory type properties.

The first region is VRAM — the GPU's on-board GDDR or HBM memory. This is the fastest memory for GPU access: full bandwidth, low latency, not accessible to the CPU except through a limited aperture. In Vulkan terms, VRAM corresponds to `VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT` without host visibility. Applications should place all GPU-only resources (textures, render targets, geometry buffers consumed only by draw calls) here.

The second region is the BAR1 aperture — a PCIe window into VRAM that is simultaneously mapped to the CPU's address space. On older systems without Resizable BAR (ReBAR), this window is limited to 256 MB, making it suitable only for streaming upload buffers. On modern systems with ReBAR enabled in the UEFI firmware, the full VRAM is exposed through this aperture, typically 8–24 GB. NVK exposes BAR1 memory as `VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT | VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`. Application developers who understand this distinction — and Chapter 24 discusses it in detail — can use ReBAR memory for resources that are written once by the CPU and read many times by the GPU, avoiding the explicit upload copy that non-ReBAR workflows require.

The third region is system memory, allocated through the GART/IOMMU mapping. This memory is host-visible and host-coherent but not device-local: the GPU accesses it over the PCIe bus with much higher latency than VRAM. NVK exposes two variants: a coherent type (immediate CPU-GPU visibility) and a cached type (CPU-cached, requiring explicit cache flush/invalidate operations). System memory is the correct placement for staging buffers, readback buffers, and any resource that the CPU writes frequently and the GPU reads rarely.

The memory type enumeration happens in `nvk_physical_device.c`. The precise heap and type indices depend on whether the GPU is a discrete card or an integrated Tegra device, since Tegra uses unified memory architecture and has different coherency semantics. The important invariant is that NVK always exposes the full set of placement options the hardware supports, giving applications the information they need to make optimal allocation decisions.

### GEM Object Lifecycle and Suballocation

Every piece of GPU memory in NVK ultimately comes from a GEM buffer object. GEM BO allocation goes through the `DRM_NOUVEAU_GEM_NEW` ioctl, which accepts placement flags specifying the desired memory domain (VRAM, GART, or pinned system memory) and alignment requirements. The nouveau kernel driver allocates from TTM (Translation Table Manager), which handles the actual physical memory allocation and the GPU page table mappings.

For CPU access, `nvk_bo_map()` uses `mmap()` on the DRM file descriptor, mapping the BO's BAR1 or GART backing into the process address space. The mapping is persistent — NVK does not unmap BOs between uses, trading virtual address space for the elimination of map/unmap overhead on the critical path of command buffer submission.

Rather than allocating a fresh GEM BO for every small driver-internal allocation, NVK uses a suballocator: `nvk_heap`, implemented in `src/nouveau/vulkan/nvk_heap.c`. The heap maintains a list of large GEM BOs and satisfies driver-internal allocation requests (descriptor pool entries, pipeline code storage, push buffer segments) from these pools using a simple free-list allocator. This pattern — a large backing allocation suballocated for driver-internal use — is standard practice in Mesa Vulkan drivers and avoids the per-allocation overhead of GEM ioctl calls for the high-frequency internal allocations that occur during command buffer recording.

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
    /* Try to satisfy from existing slabs first */
    simple_mtx_lock(&heap->mutex);
    uint64_t addr = util_vma_heap_alloc(&heap->vma_heap, size, alignment);
    if (addr != 0) {
        simple_mtx_unlock(&heap->mutex);
        *addr_out = addr;
        if (map_out)
            *map_out = (char *)heap->bo_map + (addr - heap->bo_addr);
        return VK_SUCCESS;
    }
    /* Grow the heap by allocating a new slab BO */
    /* ... */
    simple_mtx_unlock(&heap->mutex);
    return nvk_heap_grow_locked(dev, heap, size, alignment, addr_out, map_out);
}
```

For application-visible `VkDeviceMemory` allocations, NVK wraps the `vk_device_memory` base object. Large allocations (above a driver-tunable threshold) receive a dedicated GEM BO; small allocations are served from a pool allocator to avoid GEM BO proliferation, which would otherwise exhaust the kernel's per-process BO limit and degrade TTM performance.

---

## 3. Shader Compilation: SPIR-V to NVIDIA ISA

### The Compilation Pipeline Overview

Shader compilation in NVK spans two phases of the Vulkan API. At `vkCreateShaderModule` time, NVK simply stores the SPIR-V binary without processing it. The real compilation happens at `vkCreateGraphicsPipeline` or `vkCreateComputePipeline` time, when the driver has the full pipeline state available and can make informed decisions about register allocation, input/output layout, and hardware feature selection. This deferred-compilation approach is standard across Mesa Vulkan drivers and is fundamental to the correctness of the pipeline cache: the cache key includes not just the SPIR-V but the full pipeline state that influences code generation.

The compilation path proceeds through four major stages: SPIR-V to NIR translation, NIR optimisation, NVK-specific NIR lowering, and NAK code generation. Each stage has clear inputs and outputs, and each can be exercised independently for debugging purposes.

### SPIR-V to NIR

The first stage uses `spirv_to_nir()` from `src/compiler/spirv/`, the Mesa SPIR-V front end maintained by Faith Ekstrand and shared by every Mesa Vulkan driver. This function performs the structural translation from SPIR-V's typed SSA form to NIR's SSA form, resolving SPIR-V decorations (memory layout qualifiers, built-in variable identities, access qualifiers) into NIR intrinsics and variables.

NVK passes NVIDIA-specific capability flags to the SPIR-V front end: the driver advertises which SPIR-V extensions and capabilities it supports (such as `SPV_KHR_shader_draw_parameters`, `SPV_EXT_descriptor_indexing`, `SPV_KHR_vulkan_memory_model`), and the front end validates that the incoming SPIR-V does not use undeclared capabilities. The driver-level capability set is derived from the enabled Vulkan extensions and the hardware generation, ensuring that Kepler does not attempt to consume SPIR-V that requires Turing's bindless texture capabilities.

### NIR Optimisation Passes

After SPIR-V conversion, NVK applies a standard sequence of NIR optimisation passes. These passes are not NVK-specific; they are the same passes applied by RADV, ANV, and every other NIR-consuming Mesa driver. The key passes are: `nir_lower_vars_to_ssa()` (promotes variables to SSA values, enabling further optimisation), `nir_remove_dead_variables()` (eliminates unused interface variables), `nir_opt_algebraic()` (peephole algebraic simplifications), `nir_opt_constant_folding()` (folds constants at compile time), `nir_opt_copy_propagate()` (eliminates redundant copies), and `nir_opt_dead_cf()` (removes unreachable control flow). These passes are iterated until the NIR reaches a fixed point — typically two to four iterations suffice for typical shader workloads.

The NIR that emerges from this sequence is semantically equivalent to the input SPIR-V but structurally simpler: dead code is gone, constants are folded, and the variable model has been reduced to SSA. This simplification is valuable because the NVK-specific lowering passes and the NAK backend both work on NIR, and simpler NIR translates to faster compilation and better code quality.

### NVK-Specific NIR Lowering

Before handing NIR to the NAK backend, NVK applies a sequence of hardware-specific lowering passes through `nvk_lower_nir()` in `nvk_pipeline.c`. These passes transform NIR constructs that have no direct hardware equivalent into sequences that do.

The most significant lowering is `nvk_nir_lower_image_addrs()`, which translates Vulkan bindless image references into NVIDIA's hardware descriptor table format. NVIDIA GPUs use a global descriptor table (GDT) — a large buffer in VRAM that holds texture/sampler descriptors at fixed indices. When a shader accesses an image through a bindless descriptor, the hardware load instruction takes a 64-bit descriptor index; the lowering pass emits the NIR instructions that compute this index from the Vulkan descriptor set offset.

Other important lowering passes include: `nir_lower_clip_cull_distance_to_vec4()`, which maps Vulkan's per-component clip/cull distances to NVIDIA's hardware representation as packed vec4 values; `nir_lower_compute_system_values()`, which maps Vulkan's `gl_WorkGroupID`, `gl_LocalInvocationID`, and `gl_GlobalInvocationID` built-ins to the specific hardware registers where NVIDIA's compute engine deposits them; and the shared memory layout lowering, which translates NIR's abstract shared memory model to NVIDIA's banked LDS (Local Data Store) addressing.

### The NAK Compiler Backend

The NAK (Nouveau Assembler Kit) compiler, located in `src/nouveau/compiler/nak/`, is the most novel and technically ambitious component of NVK. It was added to Mesa in the 24.0 release cycle after approximately six months of development. It is notably written primarily in Rust — an unusual choice for Mesa, which is predominantly a C codebase — reflecting Faith Ekstrand's view that Rust's type system and memory safety guarantees reduce the class of compiler bugs that commonly afflict C-based compilers.

NAK defines its own intermediate representation, NAK IR, which sits between NIR's high-level SSA form and the final NVIDIA SASS (Shader ASSembly) binary encoding. NAK IR is an explicit-register IR: where NIR uses unlimited virtual SSA values, NAK IR uses named register files (GPR, predicate register, carry flag) with explicit operand slots. This representation is closer to the hardware and makes register allocation more straightforward because the allocator works on a representation that already encodes hardware constraints (register pair requirements for 64-bit values, predicate register separation from general registers).

The NAK compilation pipeline begins with `nak_compile_shader()`, which accepts a NIR shader and a set of driver options including the target architecture (Kepler SM 3.x through Blackwell SM 10.x). The function translates NIR instructions to NAK IR instructions using an instruction selector: each NIR opcode maps to one or more NAK IR instructions according to the capabilities of the target architecture. Memory instructions map to hardware load/store variants: `LDG` for global memory loads, `LDS` for shared memory loads (within a compute workgroup), `LDSM` for matrix loads on Turing and newer. Texture instructions map to the `TEX`, `TXD`, `TXL`, and `TXF` SASS instruction family depending on derivative and LOD requirements.

```c
/* Source: src/nouveau/compiler/nak/lib.rs — nak_compile_shader() conceptual flow */
/* NAK is primarily Rust; this is a conceptual C-style pseudocode representation */

/* Step 1: NIR → NAK IR (instruction selection) */
nak_ir = nak_select_instrs(nir_shader, target_sm);

/* Step 2: NIR-level optimizations on NAK IR */
nak_opt_copy_prop(nak_ir);
nak_opt_dce(nak_ir);

/* Step 3: Register allocation */
nak_ra_alloc(nak_ir, target_sm);     /* GPRs: up to 255 per thread */
nak_ra_alloc_pred(nak_ir);           /* Predicate registers: P0–P6 */

/* Step 4: Instruction scheduling (basic) */
nak_sched(nak_ir, target_sm);

/* Step 5: Binary encoding */
nak_encode(nak_ir, target_sm, out_binary);
```

Register allocation in NAK uses a variant of linear scan allocation. The NVIDIA GPU register file is organised as 255 general-purpose 32-bit registers per thread. 64-bit values occupy aligned pairs of GPRs (registers `n` and `n+1` where `n` is even). Texture coordinate vectors may require quadruples. NAK's allocator must respect these alignment constraints while also respecting the hardware limit on live register count — which determines warp occupancy and ultimately compute throughput. Predicate registers (P0 through P6 on most architectures) are allocated separately from GPRs, since they occupy a distinct hardware register file and are used exclusively for conditional instruction encoding.

One architecturally important feature of NVIDIA SASS that NAK must handle is the control code mechanism. On Maxwell and later architectures, the instruction stream contains scheduling control codes interspersed between groups of executable instructions. These control codes encode warp scheduling hints — stall counts, yield flags, read/write barriers — that the warp scheduler uses to determine when a warp can issue the next instruction without stalling. Correct control code generation is essential for both correctness (avoiding WAR/RAW hazards) and performance (maximising warp-level instruction-level parallelism). NAK generates conservative control codes that are correct but not maximally optimised compared to NVIDIA's proprietary compiler, which has access to detailed pipeline timing models.

### Shader Binary Upload and the QMD

After NAK produces the SASS binary, NVK must upload it to VRAM and make it accessible to the GPU. Shader binaries are allocated from the `nvk_heap` suballocator in a region of VRAM flagged as executable. The virtual address of the binary is recorded in the pipeline object and referenced in the pipeline's push buffer methods at dispatch time.

For compute shaders, NVK must also construct a QMD (Queue MetaData) descriptor: a 64-byte hardware structure that describes the compute shader's resource requirements to the GPU's compute dispatch engine. The QMD encodes the thread block dimensions, the number of registers per thread (from NAK's register allocation result), the shared memory size (from NIR's shared memory declarations), and the virtual address of the shader binary. The GPU's compute engine reads the QMD to set up warp state before launching the first thread block. Graphics shaders use an analogous descriptor format specific to each shader stage (vertex, tessellation, geometry, fragment).

### The Disk Shader Cache

NVK integrates with Mesa's `vk_pipeline_cache` from the Vulkan common layer. The cache maps a shader key — computed as the hash of the NIR shader after all frontend passes, combined with the driver version, hardware generation, and any driver-specific compilation flags — to the compiled SASS binary plus metadata (register count, QMD parameters). On a cache hit, the NAK compilation stage is skipped entirely, reducing pipeline creation latency by roughly the time of the NIR-to-binary compilation. Disk cache hit rates are high for repeated application runs with unchanged shaders, and the key includes enough hardware specificity that a cached binary for Turing is not incorrectly used on Ampere.

---

## 4. Pipeline Objects and Command Recording

### Descriptor Sets and the Bindless Model

NVIDIA hardware uses a fundamentally different descriptor model from AMD or Intel. Rather than per-stage descriptor tables with hardware-walked binding tables, NVIDIA GPUs use a global descriptor buffer: a contiguous region of VRAM that holds texture/sampler/buffer descriptors at indices used directly by shader instructions. This is the NVIDIA bindless model, and it predates Vulkan's `VK_EXT_descriptor_buffer` extension by years — the proprietary driver has always worked this way internally.

NVK maps Vulkan's descriptor set model onto this hardware reality. Each `VkDescriptorPool` allocates a region within a large driver-managed global descriptor buffer. When the application creates a descriptor set (`vkAllocateDescriptorSets`) and writes descriptors into it (`vkUpdateDescriptorSets`), NVK writes the hardware descriptor entries directly into the pre-allocated region of the global buffer. The `VkDescriptorSetLayout` records the byte offsets and descriptor formats for each binding, and `nvk_descriptor_set_layout.c` translates `VkDescriptorSetLayoutBinding` entries into NVIDIA's 32-byte texture header format and 16-byte sampler format.

Push descriptors (`VK_KHR_push_descriptor`) are handled by writing descriptors into a dedicated region of the command buffer's push buffer, which the GPU reads directly. This is more efficient for small, frequently-changing descriptor sets because it avoids the indirection through the global descriptor buffer.

### Graphics Pipeline State

The `nvk_graphics_pipeline` object stores the compiled shader binaries for all active stages, the hardware state register values for fixed-function pipeline stages (rasteriser, depth/stencil test, blend state), and the set of dynamic state flags indicating which state will be set at draw time rather than pipeline bind time.

Vulkan's extended dynamic state extensions (`VK_EXT_extended_dynamic_state`, `VK_EXT_extended_dynamic_state2`, `VK_EXT_extended_dynamic_state3`) allow applications to defer more and more pipeline state to draw time. NVK implements these extensions by recording state-setting method calls at `vkCmdDraw` time rather than at `vkCmdBindPipeline` time. The cost is slightly more work per draw call in exchange for fewer pipeline objects and better pipelining in applications that change state frequently. NVK's implementation records the full set of GPU register writes for dynamic state at draw time using the same push buffer mechanism as other commands.

### Compute Pipeline and Dispatch

The `nvk_compute_pipeline` object is simpler than the graphics pipeline: it holds the compiled compute shader binary, the QMD descriptor, and the shader's resource requirements. When the application calls `vkCmdDispatch(commandBuffer, groupCountX, groupCountY, groupCountZ)`, NVK writes a sequence of GPU methods into the command buffer that: sets the QMD virtual address, encodes the dispatch grid dimensions, and triggers the compute engine's launch mechanism through the appropriate GPU class method (class `NVC0C0` for Maxwell through Ampere, updated for later architectures).

### Command Buffers and the Push Buffer

The `nvk_cmd_buffer` object wraps Mesa's `vk_command_buffer` base with NVIDIA-specific push buffer management. NVIDIA's channel model sends work to the GPU by writing sequences of method–data pairs into a ring buffer (the push buffer or indirect buffer, IB). Each method encodes a hardware register address and subchannel; the data is the value to write into that register. The GPU processes these method writes sequentially within a channel, executing the hardware operations they trigger.

NVK's push buffer allocator is a simple linear allocator: each command buffer has a current BO from the `nvk_heap`, and writes methods at sequentially increasing offsets. When the current BO is exhausted, the allocator chains to a new BO by writing an indirect jump method. This design is simple, fast, and produces predictable memory usage patterns.

The method-writing API uses a family of macros that encode the hardware class and method name at compile time, providing type-checked access to the hardware register namespace. The macro system originated in envytools and was adapted for Mesa:

```c
/* Source: src/nouveau/vulkan/nvk_cmd_buffer.c — push buffer macro usage */

/* Begin a method sequence on subchannel 0 (3D class) */
P_MTHD(push, NV9097, SET_VIEWPORT_SCALE_X(0));
    P_NV9097_SET_VIEWPORT_SCALE_X(push, 0, fui(fb->width  * 0.5f));
    P_NV9097_SET_VIEWPORT_SCALE_Y(push, 0, fui(fb->height * 0.5f));
    P_NV9097_SET_VIEWPORT_SCALE_Z(push, 0, fui(0.5f));

/* Setting a single method with P_IMMD (immediate, one method) */
P_IMMD(push, NV9097, SET_RASTER_ENABLE, V_TRUE);
```

The `P_MTHD` macro emits an increasing-count method header followed by N consecutive data words. `P_IMMD` emits a single-method header and one data word. The method names (`NV9097_SET_VIEWPORT_SCALE_X`) are auto-generated from the `open-gpu-doc` hardware class headers that NVIDIA released publicly, ensuring correctness against the actual hardware specification.

### Render Passes

NVK has never implemented legacy `VkRenderPass` objects internally. Instead, it implements `VK_KHR_dynamic_rendering` natively and uses Mesa's `vk_render_pass` lowering layer from `src/vulkan/runtime/vk_render_pass.c` to convert legacy render pass API calls into dynamic rendering calls before they reach NVK's driver code. This is an example of the "never implement what the common infrastructure already handles" principle that shapes NVK's design throughout. The load/store operations specified in subpass attachments become explicit `vkCmdClearAttachments` or blit operations inserted by the lowering layer; NVK's rasteriser code only ever sees the lowered form. Multisampling resolve operations on NVIDIA hardware are handled through the copy engine (`NVC0B5_*` class methods), which has dedicated MSAA resolve capabilities.

---

## 5. Synchronisation on the Nouveau Channel Model

### The Fundamental Challenge

Vulkan's synchronisation model is explicit and rich: the application declares pipeline barriers, semaphore signals and waits, timeline semaphore points, and event completions, and the driver maps each of these to hardware primitives. The nouveau kernel driver, historically designed for OpenGL's implicit synchronisation model, did not originally provide the primitives needed for a correct Vulkan synchronisation implementation. The introduction of the `DRM_NOUVEAU_EXEC` interface and its `drm_syncobj`-based synchronisation was a prerequisite for NVK, developed in concert with NVK itself.

### drm_syncobj and the EXEC Interface

The foundational kernel primitive for NVK synchronisation is `drm_syncobj`, the DRM synchronisation object type available on all modern Linux DRM drivers. A `drm_syncobj` is a kernel object that holds a DMA fence, which transitions from unsignalled to signalled when a GPU operation completes. Timeline sync objects extend this with an integer payload: a monotonically increasing counter where different values represent different points in a timeline.

The `DRM_NOUVEAU_EXEC` ioctl is the interface through which NVK submits GPU work. Its parameters are:

```c
/* Source: include/uapi/drm/nouveau_drm.h */
struct drm_nouveau_exec_push {
    __u64 va;        /* GPU virtual address of the push buffer segment */
    __u32 va_len;    /* Length in bytes */
    __u32 flags;
#define DRM_NOUVEAU_EXEC_PUSH_NO_PREFETCH 0x1
};

struct drm_nouveau_sync {
    __u32 flags;
#define DRM_NOUVEAU_SYNC_SYNCOBJ          0x0
#define DRM_NOUVEAU_SYNC_TIMELINE_SYNCOBJ 0x1
#define DRM_NOUVEAU_SYNC_TYPE_MASK        0xf
    __u32 handle;          /* DRM syncobj handle */
    __u64 timeline_value;  /* For TIMELINE_SYNCOBJ: the timeline point */
};

struct drm_nouveau_exec {
    __u32 channel;
    __u32 push_count;
    __u32 wait_count;
    __u32 sig_count;
    __u64 wait_ptr;    /* pointer to array of drm_nouveau_sync (waits) */
    __u64 sig_ptr;     /* pointer to array of drm_nouveau_sync (signals) */
    __u64 push_ptr;    /* pointer to array of drm_nouveau_exec_push */
};
```

The kernel-side nouveau scheduler (`nouveau_sched`) processes this structure by first waiting for all `wait_ptr` sync objects to be signalled (blocking the job from executing until predecessor operations complete), then submitting the push buffer segments to the specified GPU channel for execution, then signalling all `sig_ptr` sync objects when the GPU completes the job. This maps exactly to Vulkan's `VkSubmitInfo2` structure, which NVK's `nvk_queue_submit()` translates into `DRM_NOUVEAU_EXEC` calls.

### Vulkan Synchronisation Primitive Mapping

NVK implements Vulkan's synchronisation primitives as follows. **Fences** (`VkFence`) are implemented as `drm_syncobj` handles; the GPU signals the syncobj by writing to it upon job completion, and `vkWaitForFences` blocks on the CPU until the signalled state is detected. **Binary semaphores** (`VkSemaphore` with type `VK_SEMAPHORE_TYPE_BINARY`) are also implemented as `drm_syncobj` handles; the signal and wait operations become `sig_ptr` and `wait_ptr` entries in `DRM_NOUVEAU_EXEC`. **Timeline semaphores** (`VkSemaphore` with type `VK_SEMAPHORE_TYPE_TIMELINE`) use `DRM_NOUVEAU_SYNC_TIMELINE_SYNCOBJ` with the `timeline_value` field encoding the specific timeline point being waited on or signalled.

**Events** (`VkEvent`) present a different challenge because they can be set and waited from both CPU and GPU sides. NVK implements GPU-set events by writing a sentinel value to a GPU-accessible memory location via a GPU method, and CPU-set events by writing the same sentinel from the CPU side. `vkCmdWaitEvents` polls this memory location from the GPU using a spin-wait method — a `while (mem_read(addr) != sentinel)` sequence encoded in the push buffer. This is correct but inefficient if events are waited for long periods; applications should prefer pipeline barriers for most synchronisation.

**Pipeline barriers** (`vkCmdPipelineBarrier`) are translated to GPU cache flush and invalidation methods. NVIDIA GPUs have a layered cache hierarchy: L1 per-SM (Streaming Multiprocessor) caches, an L2 unified cache shared across SMs, and the framebuffer compression engine. Different barrier types require different cache operations. A shader storage buffer write followed by a shader storage buffer read requires an L2 flush to ensure the writing SM's L1 data is visible to the reading SM's L2 view. A texture sample following a compute write requires additionally invalidating the texture cache (`NVA0C0_INVALIDATE_TEXTURE_HEADER_CACHE`) since the texture unit has its own cache separate from the compute data path.

### External Synchronisation and Wayland Integration

The `VK_KHR_external_semaphore_fd` and `VK_KHR_external_memory_fd` extensions allow NVK to export and import `drm_syncobj` handles as file descriptors. This capability is the foundation for NVK's integration with the Wayland `wp_linux_drm_syncobj_v1` protocol, described in Chapter 3.

The historical context is important here. NVIDIA's proprietary driver historically lacked `drm_syncobj` support, which blocked the adoption of the `wp_linux_drm_syncobj_v1` protocol for NVIDIA users on Wayland. The protocol allows the compositor to express explicit synchronisation requirements to the application: "you must signal this syncobj before I composite your buffer." Without syncobj export support in the Vulkan driver, applications on NVIDIA with Wayland had to rely on implicit synchronisation — which is slower, less correct under concurrent workloads, and fundamentally at odds with Vulkan's explicit synchronisation model.

NVK, by building on `drm_syncobj` from day one, provides correct `VK_KHR_external_semaphore_fd` support and thus full `wp_linux_drm_syncobj_v1` compatibility for open-driver NVIDIA users. This is one of the most immediately user-visible advantages of NVK over the legacy nouveau driver for Wayland desktop users, even before accounting for performance differences.

---

## 6. Vulkan Feature Coverage and Conformance Status

### Conformance Across the Hardware Generations

NVK's Khronos conformance story has progressed rapidly. The driver achieved Vulkan 1.3 conformance on Turing, Ampere, and Ada in early 2024, with the announcement explicitly framing this as "no hacks" — meaning the driver passes the full CTS without workarounds or test-category exclusions. By Mesa 25.0, NVK was advertising Vulkan 1.4 on Turing through Ada. By Mesa 25.1, Vulkan 1.4 conformance was extended back to Maxwell, Pascal, and Volta. Blackwell (RTX 50 series) joined the Vulkan 1.4 conformant list with Mesa 25.2.

The generation-stratified conformance reflects genuine hardware limitations. Kepler supports only Vulkan 1.2 because the Kepler memory model predates the `VK_KHR_vulkan_memory_model` extensions — the hardware does not provide the necessary memory ordering guarantees for the full Vulkan memory model. Kepler through Pascal and Volta lack `hostImageCopy` support, dropping them to Vulkan 1.3 at most. The full Vulkan 1.4 feature set, including all Vulkan 1.4 core promoted extensions, is available only on Turing and newer where GSP-RM provides reclocking and the hardware supports the full feature matrix.

Running the dEQP-VK conformance suite against NVK requires identifying the correct Vulkan physical device:

```bash
# Source: dEQP test runner invocation
# List available Vulkan devices first
deqp-vk --list-cases 2>&1 | head -5

# Run the full conformance suite against NVK on device 0
MESA_LOADER_DRIVER_OVERRIDE=nvk \
deqp-vk --deqp-surface-type=fbo \
         --deqp-vk-device-id=0 \
         --deqp-log-filename=nvk-results.qpa \
         dEQP-VK.*
```

The pass rate on Turing+ as of Mesa 24.x exceeded 99% of mandatory (non-skipped) test cases. Known failure categories included precision edge cases in transcendental shader math functions and occasional race conditions in synchronisation stress tests. Both categories shrank significantly across Mesa 24.x point releases as the NVK developers iterated on the test results.

### Extension Coverage

NVK supports over 40 Vulkan extensions. Notable supported extensions include the full `VK_KHR_synchronization2` family, `VK_KHR_timeline_semaphore`, `VK_KHR_buffer_device_address`, `VK_KHR_dynamic_rendering`, `VK_EXT_descriptor_buffer` (Maxwell A and later), `VK_EXT_extended_dynamic_state` through `VK_EXT_extended_dynamic_state3`, `VK_EXT_conservative_rasterization` (Maxwell B and later), `VK_KHR_fragment_shading_rate` (Turing and later), `VK_EXT_transform_feedback`, and `VK_KHR_cooperative_matrix` (Turing and later).

Hardware feature access that remains absent from NVK as of Mesa 25.x centres on the features that require undocumented or partially-documented hardware interfaces. **Ray tracing** (`VK_KHR_ray_tracing_pipeline`, `VK_KHR_acceleration_structure`) requires programming the RT cores present in Turing and later, and this portion of the hardware is not yet sufficiently documented even in the `open-gpu-doc` and `open-gpu-kernel-modules` repositories to enable a correct software implementation. This is an intrinsic limitation, not a driver maturity issue — the information needed to program the BVH traversal engine correctly is not yet available to the open-source community. **Video decode** (`VK_KHR_video_decode_queue`) requires interface to the NVDEC hardware, which similarly lacks sufficient public documentation. **Device-generated commands** (`VK_EXT_device_generated_commands`) has received preliminary work but was not complete as of Mesa 25.x.

### Performance Characteristics

NVK's performance on rasterisation and compute workloads as of Mesa 24.x–25.x is typically in the range of 60–80% of NVIDIA's proprietary Vulkan driver on Turing hardware for GPU-bound workloads. The primary sources of this gap are:

**Shader code quality**: The NAK compiler does not yet perform the instruction scheduling and pipeline filling optimisations that NVIDIA's proprietary compiler applies. NVIDIA's compiler has decades of GPU microarchitecture modelling; NAK is a few years old. The gap is largest on compute-heavy workloads where instruction scheduling matters most. On simple rasterisation, the gap is smaller because the hardware pipelines are naturally latency-tolerant.

**Pipeline creation latency**: NAK is faster than NVIDIA's proprietary compiler at creating pipelines, because it does fewer optimisation passes. This means lower stutter on first-time shader compilation but potentially lower peak GPU throughput. For titles that rely heavily on the proprietary driver's offline precompiled shader cache, NVK may exhibit more compilation hitches.

**Missing async compute features**: Some performance techniques that NVIDIA's proprietary driver uses internally (notably fine-grained preemption and compute/graphics interleaving) are not yet exposed through the open kernel interface in ways that NVK can exploit.

### DXVK and VKD3D-Proton

DXVK (the D3D9/D3D10/D3D11 to Vulkan translation layer) runs correctly on NVK, enabling Windows games through Proton. Faith Ekstrand invested specific effort during NVK's development to ensure DXVK's patterns work without driver workarounds — DXVK is an important compatibility target and its shader patterns stress test many Vulkan extension paths.

VKD3D-Proton (D3D12 to Vulkan translation) also runs on NVK with functional coverage for D3D12 feature level 12_1. The primary limitation is the absence of `VK_KHR_ray_tracing_pipeline`, which blocks D3D12 DXR ray tracing workloads. D3D12 games that do not require DXR typically run correctly on NVK through VKD3D-Proton, including titles that use mesh shaders, cooperative matrices, and other modern D3D12 features where NVK has extension coverage.

---

## 7. Lessons for New Mesa Vulkan Driver Authors

### Start with the Common Infrastructure

NVK's most important architectural lesson is also its simplest: use every piece of Mesa's Vulkan common infrastructure from day one, and implement custom code only where the hardware genuinely requires it. The common infrastructure in `src/vulkan/runtime/` handles more of the Vulkan specification than new driver authors often realise. The `vk_device`, `vk_physical_device`, and `vk_command_buffer` base objects carry the Vulkan API dispatch table, the enabled extension tracking, and a great deal of the API validation work. Embedding these as the first member of driver-specific structs and using the `container_of()` pattern to access the wrapping struct is the standard Mesa Vulkan idiom.

The `vk_render_pass` lowering layer is perhaps the single biggest piece of work the common infrastructure does on behalf of new drivers. Implementing legacy render pass support correctly — the load/store operation semantics, the subpass dependency model, the MSAA resolve rules — is a significant undertaking. New drivers should use `vk_render_pass` and implement only `VK_KHR_dynamic_rendering` natively. NVK demonstrated that this approach delivers full Vulkan render pass compatibility without a line of driver-specific render pass code.

The `vk_pipeline_cache` should be adopted from day one. Shader compilation is expensive; early-generation drivers that skip the cache save implementation time but create a user experience problem that is harder to fix later. The pipeline cache's key stability guarantees — that the same input NIR plus the same driver version produces a cache hit — are built into the common implementation, and preserving these guarantees requires only that the driver maintain a stable shader key format.

### Build Compute Before Rasterisation

NVK's development history makes a compelling case for implementing compute pipelines before graphics pipelines. Compute is simpler in almost every dimension: no vertex input, no rasterisation state, no render pass, no fragment output. A working compute pipeline validates the shader compilation pipeline end-to-end — SPIR-V ingestion, NIR optimisation, backend code generation, QMD construction, VRAM upload, queue submission, and result readback — without requiring the graphics pipeline infrastructure to be correct simultaneously. Once `vkCmdDispatch` produces correct results on a hello-world compute shader, the driver author has confidence in the compiler backend and can approach graphics pipeline complexity with a stable foundation.

### Use drm_syncobj Exclusively

NVK demonstrates that the full Vulkan synchronisation model — fences, binary semaphores, timeline semaphores, external semaphores — can be implemented entirely on top of `drm_syncobj` and timeline syncobj without any driver-private fence types. This matters because it keeps the kernel/userspace interface simple, enables automatic integration with the DRM scheduler's dependency tracking, and provides the foundation for external synchronisation with the compositor. New driver authors should resist the temptation to implement custom GPU-side fence objects; `drm_syncobj` is sufficient and correct.

### The Push Buffer as a Linear Allocator

NVK's push buffer allocator is deliberately simple: a linear allocator over a chain of GEM BOs, with no free list and no random-access modification. This design choice has important properties. It is simple enough to implement correctly in a few hundred lines of code. It is fast because allocation is a pointer increment. It is debuggable because the push buffer content is a sequential record of every GPU method write, making it easy to replay or diff for GPU fault analysis. And it is natural for command buffer semantics because Vulkan command buffers are write-once, execute-once structures — there is no need for the complexity of a general allocator.

### The Bringup Checklist

NVK's development history, from the first upstream commit in 2022 through Vulkan 1.3 conformance in 2024, implies a natural ordering for new Mesa Vulkan driver bringup:

1. Physical device enumeration: get `vkEnumeratePhysicalDevices` and `vkGetPhysicalDeviceProperties` returning correct values. This validates the NVKMD or equivalent kernel interface.
2. Memory allocation: implement GEM BO allocation and the Vulkan memory type table. Validate with `vkAllocateMemory` and map/unmap.
3. Command buffer recording and queue submission with fences: implement enough of the push buffer to submit a no-op command buffer. Validate with `vkQueueSubmit` and `vkWaitForFences`.
4. Compute pipeline: SPIR-V to backend compilation and `vkCmdDispatch`. Validate with a simple data-parallel computation and CPU readback.
5. Graphics pipeline: vertex input, rasterisation, fragment shader, render pass through dynamic rendering. Validate with a triangle.
6. Vulkan WSI: implement `VkSwapchainKHR` integration for at least one platform (Wayland or X11). The `nvk_wsi.c` file handles this for NVK, connecting to the `wsi_common` layer in Mesa's Vulkan runtime.
7. Descriptor sets: implement full descriptor pool and set management beyond push descriptors.
8. Advanced synchronisation: timeline semaphores, external semaphores, and compositor integration.

Each step builds on the previous ones and provides a clear go/no-go test. The ordering minimises the amount of unvalidated code that must be debugged simultaneously at any point in the development process.

### NVK as a Reference for Embedded GPU Authors

Authors developing Mesa Vulkan drivers for embedded GPUs — Qualcomm Adreno, ARM Mali, PowerVR, and future designs — can use NVK as a reference for how to structure the common infrastructure usage. The key patterns to study are: how `nvk_device` wraps `vk_device`; how `nvk_physical_device.c` enumerates capabilities and memory types; how `nvk_queue.c` maps Vulkan queue submission to the kernel UAPI; and how `nvk_pipeline.c` structures the NIR lowering sequence. These patterns are more directly applicable to new driver development than the NVIDIA-specific hardware details, and they represent the result of several years of Mesa Vulkan driver engineering lessons being encoded into reusable infrastructure.

---

## Integrations

**Chapter 8 (nvkm Architecture)**: NVK's entire hardware interface flows through the nvkm kernel driver described in Chapter 8. The `DRM_NOUVEAU_GEM_NEW` ioctl for BO allocation and the `DRM_NOUVEAU_EXEC` ioctl for queue submission are the two primary kernel interfaces NVK uses. NVK's NVKMD abstraction layer wraps these ioctls; the kernel-side implementation, including TTM integration and the `nouveau_sched` GPU scheduler that processes syncobj waits, is described in Chapter 8. NVK's channel allocation calls into nvkm's FIFO engine, which must be initialised (typically via GSP-RM on Turing+) before NVK can submit any GPU work.

**Chapter 9 (GSP-RM)**: The performance characteristics described in section 6 — 60–80% of proprietary driver performance on rasterisation workloads, full clock-speed execution on Turing+ — are conditional on GSP-RM being active, as described in Chapter 9. Without GSP-RM, the nouveau kernel driver cannot perform GPU reclocking on Turing and newer cards, severely constraining NVK's performance ceiling. The clean split between NVK (Mesa userspace) and GSP-RM (kernel firmware) is visible in the architecture: NVK makes no assumptions about clock management, leaving that entirely to the kernel driver layer.

**Chapter 14 (NIR)**: The shader compilation pipeline in section 3 — specifically the SPIR-V-to-NIR translation, the optimisation passes (`nir_opt_algebraic`, `nir_opt_constant_folding`, etc.), and the lowering passes (`nir_lower_io`, `nir_lower_vars_to_ssa`) — are all part of the NIR framework described in Chapter 14. NVK-specific lowering passes like `nvk_nir_lower_image_addrs()` follow the NIR lowering pass API described there. Readers who want to understand the NIR transformation model before reading section 3 should consult Chapter 14 first.

**Chapter 15 (ACO)**: NAK (Nouveau Assembler Kit) is structurally analogous to ACO (AMD Compiler for Optimised shaders), which is the RADV-specific NIR-consuming compiler described in Chapter 15. Both are Mesa-integrated hardware compilers that translate NIR to hardware-specific ISA, both replaced a less optimal LLVM-based backend, and both use a custom intermediate representation (NAK IR vs. ACO IR) that sits between NIR's SSA and the final binary encoding. Readers who understand ACO's architecture will find NAK's design choices immediately contextualised.

**Chapter 16 (Mesa Vulkan Common)**: Sections 2, 4, and 5 of this chapter build directly on the Mesa Vulkan common infrastructure described in Chapter 16. The `vk_device` base object, the `vk_render_pass` lowering layer, the `vk_pipeline_cache` implementation, and the common extension implementations (`VK_KHR_synchronization2`, maintenance4) are all from the common infrastructure. Chapter 16 provides the full specification of these components; the current chapter shows how NVK uses them.

**Chapter 3 (Explicit Sync)**: Section 5's discussion of `VK_KHR_external_semaphore_fd` and the Wayland `wp_linux_drm_syncobj_v1` protocol integration connects directly to Chapter 3's treatment of explicit synchronisation in the Linux graphics stack. NVK is the primary enabler of NVIDIA hardware's participation in the open-driver explicit sync ecosystem. `drm_syncobj` export from `nvk_queue_submit()` flows up through the WSI layer in `nvk_wsi.c` to the compositor, implementing the protocol that Chapter 3 describes from the compositor's perspective.

**Chapter 24 (Vulkan for Application Developers)**: The memory type discussion in section 2 — VRAM, BAR1/ReBAR, GART, and the performance implications of each — directly informs the application-level guidance in Chapter 24 on NVIDIA memory type selection. The ReBAR story is particularly important for application developers: on ReBAR-enabled systems, the `DEVICE_LOCAL | HOST_VISIBLE` memory type represents the full VRAM, making it the optimal allocation for resources written once per frame by the CPU and consumed many times by the GPU.

**Chapter 28 (Windows Compatibility)**: Section 6's discussion of DXVK and VKD3D-Proton on NVK connects to Chapter 28's coverage of those translation layers. The specific extension gaps discussed there — the absence of `VK_KHR_ray_tracing_pipeline` blocking D3D12 DXR workloads — are rooted in the hardware documentation situation described in this chapter: the BVH traversal engine in NVIDIA's RT cores is not yet sufficiently documented for open-source implementation.

**Chapter 31 (Conformance)**: Section 6's dEQP-VK discussion connects to Chapter 31's treatment of the Khronos CTS testing infrastructure and the Khronos conformance process. NVK's journey from zero to Vulkan 1.4 conformance is a case study in how a Mesa Vulkan driver moves through the CTS from early failures to submission-ready pass rates. The specific test runner invocation shown in section 6 corresponds to the workflow described in Chapter 31.

---

## References

1. [NVK source in Mesa](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/nouveau/vulkan) — The NVK Vulkan driver source tree: nvk_device.c, nvk_physical_device.c, nvk_cmd_buffer.c, nvk_queue.c, nvk_heap.c, nvk_pipeline.c, nvk_descriptor_set_layout.c, nvk_wsi.c

2. [NAK compiler source in Mesa](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/nouveau/compiler) — The Rust-based NAK (Nouveau Assembler Kit) compiler backend: instruction selection, register allocation, SASS binary encoding

3. [Mesa Vulkan common infrastructure](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/vulkan) — The shared Vulkan runtime used by NVK: vk_device, vk_render_pass, vk_pipeline_cache, common extension implementations

4. [NVK driver documentation — Mesa docs](https://docs.mesa3d.org/drivers/nvk.html) — Official NVK documentation: hardware support, debug flags, system requirements, extension status

5. [NVK external hardware docs — Mesa docs](https://docs.mesa3d.org/drivers/nvk/external_hardware_docs.html) — Hardware references for NVK development: NVIDIA architecture whitepapers, ISA documentation, envytools, open-gpu-doc

6. [Introducing NVK — Collabora blog](https://www.collabora.com/news-and-blog/news-and-events/introducing-nvk.html) — Faith Ekstrand's October 2022 introduction to NVK: motivation, design philosophy, common infrastructure approach

7. [NVK update: enabling extensions and conformance — Collabora blog](https://www.collabora.com/news-and-blog/news-and-events/nvk-update-enabling-new-extensions-conformance-status-more.html) — Detailed conformance status, extension enablement, and performance notes from early NVK development

8. [Rust-written NAK compiler merged for NVK in Mesa 24.0 — Phoronix](https://www.phoronix.com/news/NAK-Merged-Mesa-24.0) — Announcement of NAK's upstream merge: GPU support, status on Turing, relationship to NVK

9. [NVK open-source Vulkan driver ready for prime time — GamingOnLinux](https://www.gamingonlinux.com/2024/02/open-source-vulkan-driver-for-nvidia-hardware-in-mesa-nvk-is-now-ready-for-prime-time/) — Mesa 24.0 NVK stability announcement: Vulkan 1.3 conformance, DXVK support, hardware range

10. [NVK now Vulkan 1.4 conformant on Maxwell, Pascal, Volta — GamingOnLinux](https://www.gamingonlinux.com/2025/04/mesa-nvk-nvidia-vulkan-driver-now-vulkan-1-4-conformant-on-maxwell-pascal-and-volta-gpus/) — Mesa 25.1 extension of Vulkan 1.4 conformance to older GPU generations

11. [Mesa NVK Vulkan 1.4 for Blackwell — Phoronix](https://www.phoronix.com/news/Vulkan-1.4-NVK-Blackwell) — Mesa 25.2 Blackwell (RTX 50 series) support announcement

12. [DRM Nouveau kernel UAPI — nouveau_drm.h](https://github.com/torvalds/linux/blob/master/include/uapi/drm/nouveau_drm.h) — The kernel-userspace interface: DRM_NOUVEAU_EXEC ioctl, drm_nouveau_exec_push, drm_nouveau_sync structures

13. [Vulkan specification — synchronisation chapter](https://registry.khronos.org/vulkan/specs/1.3/html/) — The normative specification for Vulkan fences, semaphores, events, pipeline barriers, and the memory model

14. [NVIDIA PTX ISA guide](https://docs.nvidia.com/cuda/parallel-thread-execution/) — NVIDIA's parallel thread execution documentation: register file structure, memory instructions, warp model (useful cross-reference for NAK instruction mapping)

15. [NVIDIA CUDA binary utilities](https://docs.nvidia.com/cuda/cuda-binary-utilities/) — Hardware instruction descriptions via nvdisasm; reference for SASS encoding that NAK produces

16. [open-gpu-doc — NVIDIA](https://github.com/NVIDIA/open-gpu-doc) — NVIDIA's publicly released hardware class header files used by NVK for GPU method definitions (NV9097, NVC0C0, etc.)

17. [open-gpu-kernel-modules — NVIDIA](https://github.com/NVIDIA/open-gpu-kernel-modules) — NVIDIA's open-source kernel driver for Turing+: reference for hardware initialisation sequences and GSP-RM interaction

18. [envytools documentation](https://envytools.readthedocs.io/en/latest/hw/index.html) — Community reverse-engineered hardware documentation for Maxwell and earlier; foundational reference for pre-Turing NVK hardware support

19. [Vulkanised 2024: Faith Ekstrand and Iago Toral — Vulkan.org](https://vulkan.org/user/pages/09.events/vulkanised-2024/) — Presentation on NVK and Mesa Vulkan common infrastructure at the 2024 Vulkan Developer Conference

20. [dEQP-VK test runner documentation](https://android.googlesource.com/platform/external/deqp/) — The Khronos Vulkan Conformance Test Suite runner used to validate NVK conformance

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
