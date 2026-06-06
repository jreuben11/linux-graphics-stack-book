# Appendix A: Glossary of Linux Graphics Stack Terms

**Audiences**: All readers — systems and driver developers, graphics application developers, and browser/web platform engineers.

This appendix provides a single authoritative reference for the specialised vocabulary used throughout the book. Terms are drawn from five layers of the Linux graphics stack: the kernel DRM/KMS subsystem, GPU hardware architecture, Mesa and its compiler infrastructure, the Wayland protocol ecosystem, and the Vulkan/OpenGL application-layer APIs. Because many terms are overloaded across layers — "fence" means something different in DRM, in Vulkan, and in the Linux `sync_file` framework — this glossary explicitly disambiguates each usage and flags the most common misconceptions. Readers who encounter an unfamiliar term mid-chapter can turn here for a concise definition and a pointer to the chapter that treats the concept in depth. Entry format: **Term** — one-sentence definition; chapter reference; misconception note where relevant; cross-references in the _See also_ line.

---

## Table of Contents

1. [Section 1: Kernel / DRM Layer Terms](#section-1-kernel--drm-layer-terms)
2. [Section 2: GPU Hardware Terms](#section-2-gpu-hardware-terms)
3. [Section 3: Mesa / Compiler Terms](#section-3-mesa--compiler-terms)
4. [Section 4: Wayland / Compositor Terms](#section-4-wayland--compositor-terms)
5. [Section 5: Application / API Terms](#section-5-application--api-terms)
6. [Integrations](#integrations)
7. [References](#references)

---

## Section 1: Kernel / DRM Layer Terms

The terms in this section originate in the Linux kernel's Direct Rendering Manager subsystem (`drivers/gpu/drm/`), which provides the unified kernel interface for GPU hardware, display output, memory management, and synchronisation. These are the concepts that every GPU driver author and display compositor must understand regardless of which higher-level API they target. The DRM UAPI is kernel-stable — ioctls are never removed — but the internal object model and feature set evolve with each kernel release.

---

**atomic modesetting**
The DRM API mode in which all display-state changes — CRTC configuration, plane source and destination rectangles, connector routing, colour management properties — are submitted as a single atomic transaction that is either committed in full or rejected wholesale; it supersedes the legacy per-object ioctl interface (`DRM_IOCTL_MODE_SETCRTC`, `DRM_IOCTL_MODE_SETPLANE`, and friends) where partial updates could leave hardware in an inconsistent state. See Chapter 2 (KMS Display Pipeline). The kernel implements this via `drm_atomic_commit()` in `drivers/gpu/drm/drm_atomic.c`, with driver-specific commit helpers in `drm_atomic_helper.c`. A test-only path (`DRM_MODE_ATOMIC_TEST_ONLY` flag) lets compositors probe whether a configuration is hardware-feasible before actually applying it.

_Misconception_: Atomic modesetting does not guarantee tear-free presentation on its own. It guarantees that hardware register updates are applied coherently across all display objects within a single vblank window, but vblank synchronisation and page-flip timing are separate concerns handled by the `DRM_MODE_ATOMIC_NONBLOCK` path and vblank event callbacks.

_See also_: CRTC, plane, connector, vblank, page flip.

---

**connector**
A DRM object (`struct drm_connector`, `include/drm/drm_connector.h`) representing one physical output port on the GPU — HDMI, DisplayPort, DSI, LVDS, VGA, or others — that carries EDID retrieval over DDC/I2C or DisplayPort AUX, hot-plug detect (HPD) interrupt registration, and the list of modes advertised by the attached display. See Chapter 2.

_See also_: encoder, CRTC, EDID, HPD, MST.

---

**CRTC** (Cathode Ray Tube Controller)
A DRM abstraction (`struct drm_crtc`, `include/drm/drm_crtc.h`) for the display pipeline stage that blends pixel data from one or more planes, applies colour transformations (gamma, CTM, degamma via `drm_color_mgmt`), and drives the scan-out timing generator (pixel clock, sync polarity, blanking intervals); the name is purely historical and has no relation to cathode ray tube technology. See Chapter 2.

_Misconception_: A CRTC does not correspond one-to-one with a physical output. One CRTC can drive multiple connectors simultaneously (clone mode), and many GPU families expose fewer CRTCs than connectors. The mapping between CRTCs and connectors goes through encoders and is constrained by hardware routing matrices.

_See also_: plane, connector, encoder, framebuffer, vblank.

---

**DMA-BUF** (Direct Memory Access Buffer)
A Linux kernel mechanism introduced in kernel 3.3 (`include/linux/dma-buf.h`) for sharing a buffer, represented as a file descriptor obtained via `dma_buf_export()`, between multiple kernel subsystems or devices without copying; the exporting device retains ownership and lifecycle management while importers attach via `dma_buf_attach()` and obtain scatter-gather mappings through `dma_buf_map_attachment()`. See Chapter 4 (GPU Memory Management).

_Misconception_: DMA-BUF does not imply that the buffer resides in system RAM. GPU-local (VRAM) buffers can be exported if the hardware supports peer-to-peer DMA or if the driver maintains a CPU-accessible shadow. The DMA-BUF framework is agnostic to buffer placement; placement policy is entirely the exporter's responsibility.

_See also_: PRIME, GEM, fence, implicit sync, explicit sync.

---

**DRM** (Direct Rendering Manager)
The Linux kernel subsystem located at `drivers/gpu/drm/` that provides a unified interface for GPU hardware, encompassing memory management (via GEM and TTM), display output control (KMS), GPU command submission, synchronisation primitives (`drm_syncobj`, `dma_fence`), and device multiplexing; userspace accesses DRM through `/dev/dri/cardN` (primary nodes) and `/dev/dri/renderDN` (render nodes) character devices using ioctls defined in `include/uapi/drm/drm.h` and `include/uapi/drm/drm_mode.h`. See Chapter 1 (DRM Architecture).

_See also_: KMS, GEM, TTM, render node, primary node.

---

**DRM format modifier**
A 64-bit vendor-defined token (`DRM_FORMAT_MOD_*`, defined in `include/uapi/drm/drm_fourcc.h`) that, when paired with a fourcc pixel format, completely specifies the memory layout of a buffer including tiling arrangement, hardware lossless compression state, and metadata plane locations; required for zero-copy buffer sharing between a GPU's render engine and its display engine when those engines expect different layout contracts. See Chapter 2. Drivers advertise supported modifier/format pairs via `DRM_IOCTL_MODE_ADDFB2` with `DRM_MODE_FB_MODIFIERS` set, and negotiate them with compositors via the `linux-dmabuf` Wayland protocol.

_Misconception_: A modifier is not merely a compression flag. It encodes an entire memory layout contract including tile dimensions, superblock size, and auxiliary surface offsets. Two drivers sharing a buffer must agree on the identical modifier value for the buffer to be correctly interpreted; a mismatch produces visual corruption, not a detectable error.

_See also_: framebuffer, DMA-BUF, GBM, linux-dmabuf.

---

**DRM lease**
A KMS mechanism (`DRM_IOCTL_MODE_CREATE_LEASE`, introduced in kernel 4.15, `drivers/gpu/drm/drm_lease.c`) that delegates a subset of DRM resources — specific CRTCs, connectors, and planes — to a secondary process, granting that process exclusive modesetting rights over those resources without requiring it to hold the primary node's master capability; used by VR compositors that need direct display access and by multi-seat setups. See Chapter 3 (Advanced Display Features).

_See also_: DRM master, primary node, KMS.

---

**DRM master**
The single process that holds the authenticated modesetting capability on a DRM primary node at any given time, controlled by `DRM_IOCTL_SET_MASTER` and `DRM_IOCTL_DROP_MASTER`; the compositor (Wayland compositor, X server, or VT manager) typically acquires master on VT switch and drops it when backgrounded. See Chapter 1.

_See also_: primary node, DRM lease.

---

**drm_syncobj**
A kernel object (`struct drm_syncobj`, `drivers/gpu/drm/drm_syncobj.c`) exposed to userspace as a file descriptor via `DRM_IOCTL_SYNCOBJ_CREATE` that wraps a `dma_fence` and can be signalled or waited on from both CPU-side ioctls and GPU command streams; supports both binary semantics (a fence is either pending or signalled) and timeline semantics (a monotonically increasing 64-bit sequence number, where each point on the timeline corresponds to a distinct fence). Timeline syncobjs were added in kernel 5.2 via `DRM_IOCTL_SYNCOBJ_TIMELINE_SIGNAL` and `DRM_IOCTL_SYNCOBJ_TIMELINE_WAIT`. See Chapter 4.

_Misconception_: A `drm_syncobj` is not a Linux `eventfd`. It carries GPU-fence semantics — it can represent an operation that has not yet been submitted to the GPU — and can be imported into Vulkan as a `VkSemaphore` (via `VK_KHR_external_semaphore_fd`) or `VkFence` (via `VK_KHR_external_fence_fd`).

_See also_: fence, timeline semaphore, explicit sync, binary semaphore.

---

**EDID** (Extended Display Identification Data)
A standardised data structure (VESA standard, 128-byte base block plus optional 128-byte extension blocks) stored in non-volatile memory inside a display device that describes its manufacturer identity, supported video modes, preferred timing, colour gamut, HDR capabilities, and audio formats; the GPU driver reads EDID over DDC/I2C (for HDMI/VGA) or DisplayPort AUX channel and exposes it through the `drm_connector`'s EDID property. See Chapter 2.

_See also_: connector, HPD, MST.

---

**encoder**
A DRM object (`struct drm_encoder`, `include/drm/drm_encoder.h`) that converts the digital pixel stream produced by a CRTC into the physical signal format required by a specific connector type: TMDS for HDMI, 8b/10b scrambled for DisplayPort, LVDS differential pairs, MIPI DSI, and so on; in modern drivers, encoders are increasingly treated as internal routing artifacts with minimal userspace-visible semantics. See Chapter 2.

_Misconception_: An encoder is a software abstraction layer that may map to a real hardware signal-conversion block or may be a no-op shim when the CRTC's output bus is directly compatible with the connector. The DRM kernel documentation describes encoders as "internal artifacts" that "serve no purpose in the userspace API" but are retained for ABI compatibility.

_See also_: CRTC, connector.

---

**explicit sync**
A synchronisation model in which producers and consumers of shared buffers exchange `dma_fence`-backed file descriptors (`sync_file`) explicitly between processes or GPU command queues, rather than relying on the kernel to implicitly attach fences to `dma_resv` reservation objects on DMA-BUFs; required by Vulkan's queue-submission model and adopted by Wayland via the `linux-drm-syncobj-v1` protocol (which superseded `zwp_linux_explicit_synchronization_unstable_v1`) to eliminate accidental synchronisation and improve frame pacing, particularly for NVIDIA's proprietary driver which historically lacked implicit sync support. See Chapter 4.

_See also_: implicit sync, drm_syncobj, fence, DMA-BUF, linux-dmabuf.

---

**fence** (DRM / `dma_fence`)
A lightweight, one-shot kernel signalling primitive (`struct dma_fence`, `include/linux/dma-fence.h`) representing the future completion of an asynchronous GPU or DMA operation; can be waited on from the CPU via `dma_fence_wait()`, chained into ordered sequences using `dma_fence_chain` or `dma_fence_array`, or exported to userspace as a `sync_file` file descriptor via `sync_file_create()` for use in explicit-sync pipelines. See Chapter 4.

_Misconception_: DRM `dma_fence` objects are distinct from Vulkan `VkFence` objects; they operate at different abstraction levels and are linked — not equated — via `drm_syncobj`. A `VkFence` represents CPU-observable GPU completion at the Vulkan API level, while a `dma_fence` is a kernel-internal primitive that the driver creates and signals.

_See also_: drm_syncobj, explicit sync, implicit sync, timeline semaphore.

---

**framebuffer** (DRM)
A kernel object (`struct drm_framebuffer`, `include/drm/drm_framebuffer.h`) that binds one or more GEM buffer objects to a pixel format (fourcc), a DRM format modifier, pitch, width, and height, and registers them as a scanout source; the DRM plane fetches pixel data from the framebuffer's backing GEM BOs when the framebuffer is assigned to a plane during an atomic commit. See Chapter 2.

_Misconception_: A DRM framebuffer object is not the same as the legacy fbdev framebuffer device (`/dev/fbN`). The two are separate kernel subsystems with distinct APIs and lifecycle models; fbdev is deprecated and exists for compatibility with older console infrastructure.

_See also_: GEM, DRM format modifier, CRTC, plane.

---

**FreeSync**
AMD's brand name for its implementation of VESA Adaptive-Sync over DisplayPort and HDMI VRR, which allows the display refresh period to vary in lock-step with the GPU's rendered frame delivery rate, eliminating tearing without the fixed-latency penalty of V-Sync and reducing stutter during frame rate fluctuations; the `amdgpu` driver exposes FreeSync control via the `VRR_ENABLED` KMS connector property. See Chapter 3.

_See also_: VRR, connector.

---

**GBM** (Generic Buffer Manager)
A Mesa library (`src/gbm/main/gbm.h`, installed as `libgbm`) that provides a platform-independent API for allocating GPU-scannable buffers (`gbm_bo_create()`, `gbm_bo_create_with_modifiers2()`), creating EGL window surfaces from DRM-backed GBM surfaces (`gbm_surface_create()`), and importing/exporting DMA-BUF file descriptors; it is the canonical userspace allocator for DRM-based Wayland compositors and EGL implementations on Linux. See Chapter 6 (Display Stack and Compositor Architecture).

_Misconception_: GBM is a Mesa userspace library, not a kernel interface. It wraps GEM buffer allocation and negotiates DRM format modifiers with the kernel on behalf of the application or compositor; the kernel has no object type called "GBM buffer."

_See also_: GEM, DRM format modifier, EGL, linux-dmabuf.

---

**GEM** (Graphics Execution Manager)
The DRM memory management framework (`drivers/gpu/drm/drm_gem.c`, introduced in kernel 2.6.28) that gives each GPU buffer allocation a kernel-managed handle (`uint32_t gem_handle`) with reference counting, CPU mmap support via `DRM_IOCTL_GEM_MMAP`, and cross-process/cross-device import/export through PRIME; simpler than TTM for integrated (UMA) or small-VRAM GPU designs and widely adopted by Intel, ARM, and Qualcomm drivers. See Chapter 4.

_Misconception_: GEM handles manage buffer lifecycle and CPU accessibility but do not manage GPU-side virtual address spaces. GPU VA management is handled by per-driver layers — for example, `drm_gpuvm` (introduced in kernel 6.1) provides a generic GPU VM framework built on top of GEM objects.

_See also_: TTM, DMA-BUF, PRIME, framebuffer.

---

**HDCP** (High-bandwidth Digital Content Protection)
A cryptographic authentication and stream-encryption protocol layered on HDMI and DisplayPort that restricts playback of DRM-protected content to certified downstream receivers; the DRM driver implements the HDCP 1.4 and HDCP 2.x key exchange and repeater topology validation in the KMS connector stack, exposed to userspace via the `Content Protection` and `HDCP Content Type` KMS properties. See Chapter 3.

_See also_: connector, EDID.

---

**HPD** (Hot Plug Detect)
A dedicated hardware signal line on HDMI, DisplayPort, and legacy VGA connector standards that transitions high or low when a display is physically connected or disconnected; the DRM driver registers a hardware interrupt on this line and responds by re-reading EDID, updating `drm_connector` status, and generating a `uevent` to notify userspace (compositors, display managers) of the topology change. See Chapter 2.

_See also_: connector, EDID, MST.

---

**implicit sync**
A synchronisation model in which the kernel automatically attaches in-flight GPU fences to the `dma_resv` reservation object embedded in a DMA-BUF, so that any consumer importing the buffer via `dma_buf_attach()` automatically waits for in-flight producer operations to complete, without requiring explicit fence passing between processes; necessary for X11 and legacy OpenGL compatibility but increasingly deprecated in favour of explicit sync for Wayland clients and Vulkan. See Chapter 4.

_See also_: explicit sync, fence, DMA-BUF.

---

**KMS** (Kernel Mode Setting)
The DRM subsystem component (`drivers/gpu/drm/`, KMS objects defined in `include/drm/drm_mode_object.h` and related headers) responsible for configuring and driving all display hardware from the kernel, replacing the legacy userspace modesetting that required elevated X server privileges; exposed through the atomic KMS ioctl API (`DRM_IOCTL_MODE_ATOMIC`) or the legacy per-object ioctls for backward compatibility. See Chapter 2.

_Misconception_: KMS controls only the display pipeline (scan-out, timing, colour management). GPU 3D rendering and compute are separate DRM concerns, handled through driver-specific command submission ioctls (e.g., `DRM_IOCTL_I915_GEM_EXECBUFFER2`, `DRM_IOCTL_AMDGPU_CS`) that have no interaction with the KMS display objects at the kernel level.

_See also_: CRTC, connector, plane, encoder, atomic modesetting, DRM.

---

**MST** (Multi-Stream Transport)
A DisplayPort 1.2+ feature that multiplexes up to 63 independent video streams over a single physical cable link using a time-division payload allocation scheme; devices are daisy-chained or connected through MST hubs, and the DRM MST topology manager (`drivers/gpu/drm/display/drm_dp_mst_topology.c`, `struct drm_dp_mst_topology_mgr`) discovers the hub tree via DPCD reads over AUX and allocates payload bandwidth slots per stream. See Chapter 3.

_See also_: connector, HPD, EDID.

---

**page flip**
The DRM operation that atomically replaces the framebuffer being scanned out by a CRTC at the next vertical blanking interval; initiated via `DRM_IOCTL_MODE_PAGE_FLIP` with `DRM_MODE_PAGE_FLIP_EVENT` in the legacy API, or via an atomic commit with the `DRM_MODE_ATOMIC_NONBLOCK` flag in the atomic API; when the hardware completes the flip, the kernel delivers a `drm_event` of type `DRM_EVENT_FLIP_COMPLETE` containing the vblank timestamp. See Chapter 2.

_See also_: vblank, framebuffer, CRTC, atomic modesetting.

---

**plane** (DRM)
A DRM object (`struct drm_plane`, `include/drm/drm_plane.h`) representing one layer in the display hardware's compositing stack; each plane fetches pixels from an assigned framebuffer, performs hardware-accelerated scaling and optional rotation, and blends the result with sibling planes before the combined output reaches the CRTC; planes are typed as primary (the base layer), overlay (additional content layers), or cursor (for low-latency pointer rendering). See Chapter 2.

_Misconception_: A DRM plane is a fixed hardware resource, not a software abstraction. Not all GPUs support the same number of overlay planes; hardware may impose constraints on which planes can be scaled, which pixel formats each plane accepts, and which planes can be assigned to which CRTCs. Atomic KMS validates these constraints at test-commit time.

_See also_: CRTC, framebuffer, atomic modesetting.

---

**PRIME**
The DRM buffer-sharing mechanism (`drivers/gpu/drm/drm_prime.c`) that exports a GEM buffer object as a DMA-BUF file descriptor via `DRM_IOCTL_PRIME_HANDLE_TO_FD` and imports a foreign DMA-BUF file descriptor back as a local GEM handle via `DRM_IOCTL_PRIME_FD_TO_HANDLE`, enabling zero-copy GPU-to-GPU sharing (e.g., NVIDIA render → Intel display), GPU-to-V4L2 sharing, and cross-driver compositing. See Chapter 4.

_See also_: GEM, DMA-BUF, render node.

---

**primary node**
The `/dev/dri/cardN` character device created by a DRM driver that includes `DRIVER_MODESET`; access to modesetting ioctls (KMS) requires either `CAP_SYS_ADMIN` or membership in the `video` group (or equivalent logind session ownership); compositors acquire and hold the primary node to control display output, and must negotiate DRM master to use atomic KMS. See Chapter 1.

_See also_: render node, DRM master, KMS.

---

**render node**
The `/dev/dri/renderDN` character device created for unprivileged GPU access — GEM buffer allocation, command submission, and PRIME import/export — without requiring modesetting privileges or DRM master; used by GPU-accelerated applications, Mesa drivers, media decoders (VA-API, V4L2), and Vulkan ICDs to submit work to the GPU independently of whoever holds the display. See Chapter 1.

_See also_: primary node, GEM, PRIME.

---

**TTM** (Translation Table Manager)
The older DRM memory manager (`drivers/gpu/drm/ttm/`, predating GEM, significantly redesigned in kernel 5.13) designed for discrete GPUs with separate VRAM pools; manages buffer placement across multiple memory domains (VRAM, GTT/GART, system RAM), eviction under memory pressure, CPU mapping types (write-combined, cached), and fence-based move synchronisation; used by the `amdgpu`, `radeon`, and `vmwgfx` drivers. See Chapter 4.

_See also_: GEM, VRAM, BAR.

---

**vblank** (vertical blanking interval)
The period between the end of one display frame's scan-out and the start of the next, during which the display controller is not reading pixel data from the framebuffer and hardware register updates can safely take effect; DRM drivers expose a per-CRTC vblank counter and high-resolution timestamp event (`DRM_EVENT_VBLANK`) via `DRM_IOCTL_WAIT_VBLANK` and atomic flip events, which compositors use to pace frame delivery, measure display latency, and schedule buffer releases. See Chapter 2.

_See also_: page flip, CRTC, VRR.

---

**VRR** (Variable Refresh Rate)
A display technology family — encompassing HDMI 2.1 VRR, VESA Adaptive-Sync (FreeSync), and NVIDIA G-Sync Compatible — that dynamically adjusts the display panel's refresh period within a hardware-defined min/max range to match the GPU's actual frame delivery rate; the DRM subsystem exposes VRR control via the `VRR_ENABLED` KMS connector property, and the `amdgpu` and `i915` drivers implement the protocol-specific signalling. See Chapter 3.

_See also_: FreeSync, vblank, connector.

---

## Section 2: GPU Hardware Terms

This section covers the architectural concepts, hardware block names, and register-level terms specific to GPU silicon — primarily AMD GCN/RDNA and NVIDIA Turing/Ampere/Ada families, with entries for mobile GPU architectures and platform-level memory management. Understanding these terms is essential for readers following driver internals, shader compiler register allocation, and memory subsystem design discussions in Parts II and III.

---

**BAR** (Base Address Register)
A PCIe configuration-space register that maps a region of GPU memory — VRAM or MMIO register space — into the CPU's physical address space; the GPU exposes its VRAM through a BAR so the CPU can read/write texture and buffer data; on systems without Resizable BAR (ReBAR) support, the BAR is typically limited to 256 MB regardless of installed VRAM, creating a bottleneck for large-asset streaming and requiring CPU-side staging buffers. See Chapter 5 (AMDGPU / GPU Driver Internals).

_See also_: ReBAR, VRAM, GART.

---

**falcon microcontroller**
NVIDIA's family of small embedded RISC processors (Falcon = Fast and Low-power CONtroller) instantiated multiple times inside each GPU die — in the PMU, SEC2, GSPLITE, and other firmware contexts — that run signed firmware blobs for power management, security services, and (on Turing and later) the GSP resource manager; Nouveau reverse-engineers their instruction set using envytools to load open-source or redistributable firmware. See Chapter 9 (The Nouveau Story).

_See also_: GSP-RM, nvkm.

---

**GART** (Graphics Address Remapping Table)
A GPU-side page table (analogous to a software IOMMU) that remaps discontiguous pages of system RAM into a contiguous GPU virtual address range, allowing the GPU to issue DMA transactions to scatter-gather buffers as if they were physically contiguous; implemented in hardware as part of the GPU's memory controller and managed by the kernel driver's address space management layer. See Chapter 4.

_See also_: IOMMU, SMMU, BAR.

---

**GCN** (Graphics Core Next)
AMD's GPU microarchitecture family spanning GCN 1 (Tahiti, 2012) through GCN 5 (Vega, 2017/2018) that introduced a scalar-plus-vector compute execution model, a 64-lane SIMD (wave64) as the fundamental scheduling unit, a unified shader ISA across compute and graphics, and a hierarchical memory model with LDS, L1, and L2 caches; GCN is superseded by RDNA starting with the RX 5000 series (Navi10, 2019). See Chapter 5.

_See also_: RDNA, ISA, VGPR, SGPR, wave64.

---

**GSP-RM** (GPU System Processor – Resource Manager)
NVIDIA's firmware blob that executes on the Falcon/GSP co-processor embedded in Turing and later GPUs and absorbs the hardware resource-manager functions that were previously performed entirely in the `nvidia.ko` kernel driver; by offloading initialisation, clock management, and channel allocation to signed GSP firmware, NVIDIA enables the `nvidia-open` kernel module and the Nouveau GSP firmware path to function correctly without knowledge of proprietary hardware initialisation sequences. See Chapter 10 (NVIDIA Open and Proprietary Drivers).

_See also_: falcon microcontroller, nvkm.

---

**IOMMU** (Input-Output Memory Management Unit)
A hardware unit on the CPU or platform die that translates device-issued DMA addresses to host physical memory addresses using page tables maintained by the kernel's `iommu` subsystem; provides device isolation (preventing a compromised GPU from accessing arbitrary host memory), scatter-gather remapping for DMA-BUF importers that receive buffers from foreign devices, and the foundation for GPU virtualisation via VFIO passthrough. See Chapter 4.

_See also_: SMMU, GART, DMA-BUF.

---

**ISA** (Instruction Set Architecture)
In the GPU context, the binary encoding, instruction formats, operand types, and execution semantics of the shader instructions executed by the GPU's shader cores; AMD publishes per-generation ISA documents (GCN ISA, RDNA 1/2/3/4 ISA) on GPUOpen that specify every opcode, whereas NVIDIA's ISA is proprietary and Nouveau's knowledge of it is derived from reverse engineering via the `envytools` project. See Chapter 5.

_See also_: GCN, RDNA, ACO, LLVM.

---

**nvkm**
The internal object-model layer inside the Nouveau kernel driver (`drivers/gpu/drm/nouveau/nvkm/`) that abstracts GPU hardware blocks — memory controller (`nvkm_fb`), display engine (`nvkm_disp`), FIFO command processor (`nvkm_fifo`), thermal management (`nvkm_therm`), clock controller (`nvkm_clk`), and others — behind versioned subdevice objects with a common `nvkm_object` base; this abstraction allows Nouveau to support a wide range of NVIDIA hardware generations (NV04 through Ada Lovelace) within a single driver framework. See Chapter 9.

_See also_: GSP-RM, falcon microcontroller.

---

**RDNA** (Radeon DNA)
AMD's current GPU microarchitecture family (RDNA 1, 2019; RDNA 2, 2020; RDNA 3, 2022; RDNA 4, 2024) that replaced GCN with a redesigned Workgroup Processor (dual Compute Unit cluster with shared instruction fetch and branch), a default wave32 execution model for graphics shaders (halving VGPR pressure relative to GCN wave64), a revised L0/L1/L2 cache hierarchy with larger and faster L1, and hardware ray-tracing units (RDNA 2+); the primary target of the `amdgpu` driver's GFX10+ code paths and the `radv` Vulkan driver. See Chapter 5.

_See also_: GCN, wave32, wave64, VGPR, SGPR.

---

**ReBAR** (Resizable BAR)
A PCIe capability (defined in the PCIe Base Specification as "Resizable BAR") that allows the CPU to map the GPU's entire VRAM through a single large BAR — typically the full 8 GB, 16 GB, or 24 GB of VRAM — rather than a 256 MB window, eliminating the need for the driver to remap BAR windows during large buffer uploads and significantly improving streaming bandwidth for large textures and geometry; requires UEFI firmware support, BIOS/UEFI option enablement, platform Above-4G decoding support, and driver/OS awareness. See Chapter 5.

_See also_: BAR, VRAM.

---

**SGPR** (Scalar General-Purpose Register)
A register in the GCN/RDNA shader core that holds a single value shared uniformly across all active lanes of a wave; scalar ALU operations (`s_add_u32`, `s_load_dwordx4`, etc.) read and write SGPRs and execute only once per wave rather than once per lane, saving significant power and area compared to vector execution; SGPRs are used for control flow masks, buffer descriptors, and uniform shader constants. See Chapter 5.

_See also_: VGPR, wave32, wave64, GCN, RDNA.

---

**SMMU** (System Memory Management Unit)
ARM's platform-level equivalent of an IOMMU, standardised in the ARM SMMU specification (SMMUv2, SMMUv3) and implemented on ARM SoCs to translate DMA addresses issued by system peripherals — including Mali, Adreno, and other mobile GPU IPs — into host physical memory addresses using kernel-managed page tables; the Linux kernel's `arm-smmu` and `arm-smmu-v3` drivers implement the SMMUv2/v3 interfaces. See Chapter 4.

_See also_: IOMMU, GART.

---

**TBDR** (Tile-Based Deferred Rendering)
A GPU rendering architecture used by Imagination PowerVR, ARM Mali, Apple GPU, and Qualcomm Adreno families in which the screen is partitioned into small tiles (typically 16×16 or 32×32 pixels); all draw calls affecting a tile are collected into a tile's command list during a binning pass, and then each tile is rendered completely in fast on-chip tile memory before the completed pixel data is flushed to system DRAM; this dramatically reduces external memory bandwidth compared to an immediate-mode renderer because intermediate colour, depth, and stencil data never touch DRAM. See Chapter 8 (Mobile and Embedded Drivers).

_See also_: tile memory.

---

**tile memory**
The small, fast on-chip SRAM in a TBDR GPU that holds all active colour, depth, and stencil attachment data for a single tile during its rendering phase; because tile memory is orders of magnitude faster and more energy-efficient than external DRAM, keeping intermediate render results on-chip is the principal mechanism by which mobile GPUs achieve their bandwidth efficiency; Vulkan render passes and subpass attachments with `VK_ATTACHMENT_LOAD_OP_DONT_CARE` / `VK_ATTACHMENT_STORE_OP_DONT_CARE` signal to TBDR drivers that tile memory contents need not be loaded from or stored to DRAM. See Chapter 8.

_See also_: TBDR.

---

**UMA** (Unified Memory Architecture)
A system configuration in which the CPU and GPU share the same physical RAM pool, with no separate GPU-local VRAM; common in APUs (AMD), mobile SoCs (ARM, Qualcomm), integrated Intel graphics, and Apple Silicon; eliminates VRAM as a distinct memory domain and fundamentally changes buffer placement policies — GEM's `drm_gem_shmem` helper and TTM's system-heap placement are both UMA-aware. See Chapter 4.

_See also_: BAR, GEM, TTM, GART.

---

**VGPR** (Vector General-Purpose Register)
A register in the GCN/RDNA shader core that holds an independent per-lane value across all active lanes of a wave (64 lanes for wave64, 32 lanes for wave32); vector ALU operations execute in lock-step across all lanes and produce per-lane results in VGPRs; the total VGPR pool on a Compute Unit is finite, and high per-thread VGPR usage reduces the number of waves that can simultaneously reside on the CU (occupancy), which in turn limits the GPU's ability to hide memory access latency through wavefront switching. See Chapter 5.

_Misconception_: VGPR pressure is not the sole occupancy limiter. LDS (Local Data Share) allocation, SGPR count, and hardware barrier quotas also independently cap the maximum number of concurrent waves per CU. A shader optimised for low VGPR usage may still suffer poor occupancy due to excessive LDS usage.

_See also_: SGPR, wave32, wave64, GCN, RDNA.

---

**wave32 / wave64**
The SIMD execution width of AMD shader cores: GCN executes wavefronts of 64 lanes simultaneously (wave64), meaning each VGPR allocation consumes 64 × 4 bytes of the register file; RDNA defaults to 32-lane wavefronts (wave32) for graphics and compute shaders, halving register file pressure and enabling higher occupancy for register-heavy shaders, while wave64 mode remains available for FP64 operations, image sample instructions requiring 64-lane uniformity, and explicit opt-in via `VK_AMD_shader_core_properties`. See Chapter 5.

_See also_: VGPR, SGPR, GCN, RDNA.

---

## Section 3: Mesa / Compiler Terms

Mesa's architecture spans a library loader layer, multiple OpenGL and Vulkan front-ends, a common intermediate representation (NIR), and a collection of per-GPU driver backends. The terms in this section describe the internal components and data-flow abstractions that graphics application developers encounter indirectly through driver behaviour (compile times, shader cache hits, pipeline stalls) and that driver and compiler engineers work with directly.

---

**ACO** (AMD COmpiler)
Mesa's native LLVM-free shader compiler backend for AMDGPU GPUs (`src/amd/compiler/`), introduced in Mesa 19.3 (2019) and now the default backend for the `radv` Vulkan driver; ACO takes NIR as input and directly emits GCN or RDNA ISA binary, with significantly faster compile times than the LLVM path and comparable or better generated code quality; it implements its own register allocator, instruction scheduler, and liveness analysis tuned specifically for AMD's wave-based execution model. See Chapter 16 (Mesa Shader Compilers).

_Misconception_: ACO is used by `radv` (the Vulkan driver). The OpenGL `radeonsi` driver continues to use the LLVM backend by default, though ACO support for OpenGL is in active development and partially available in recent Mesa releases. The two drivers are architecturally independent despite targeting the same hardware.

_See also_: NIR, LLVM, RDNA, GCN, ISA.

---

**disk shader cache**
A persistent filesystem cache (default location `~/.cache/mesa_shader_cache/`, configurable via `MESA_SHADER_CACHE_DIR`) that stores compiled GPU machine code keyed by a hash of the NIR intermediate representation and relevant driver state (device ID, driver version, compiler flags); when a shader program or pipeline is first compiled, the result is serialised to disk, and on subsequent runs the cached binary is loaded and submitted directly to the hardware, eliminating recompilation latency. See Chapter 17 (Mesa Caching and Performance).

_See also_: pipeline cache, NIR, ACO, LLVM.

---

**driconf**
Mesa's per-application configuration system (`/usr/share/drirc.d/*.conf` and `~/.drirc`) that applies driver workarounds, feature capability overrides, and performance tuning parameters to specific applications matched by executable name, EGL string, or GLX application name; this is the mechanism by which Mesa ships per-game compatibility fixes and performance workarounds without requiring application updates or Mesa patches. See Chapter 15 (Mesa Architecture).

_See also_: Gallium, pipe driver.

---

**Gallium**
Mesa's internal state and resource abstraction framework for OpenGL, OpenCL, and Direct3D 9 (Nine) drivers (`src/gallium/`), defining a `struct pipe_context` interface that pipe drivers implement and a `struct pipe_screen` interface for capability queries and resource allocation; the framework decouples the GL state tracker from GPU-specific code, allowing OpenGL optimisation passes to be written once and shared across all Gallium pipe drivers. See Chapter 15.

_Misconception_: Gallium is not used by Mesa's Vulkan drivers. The `radv` (AMD), `anv` (Intel), `nvk` (NVIDIA/Nouveau), `tu` (Qualcomm Turnip), and `v3dv` (Broadcom) Vulkan drivers bypass Gallium entirely and connect directly to NIR and driver-specific backends. Gallium is an OpenGL/OpenCL concern.

_See also_: pipe driver, state tracker, NIR, TGSI.

---

**GPL** (Graphics Pipeline Library)
A Vulkan extension (`VK_EXT_graphics_pipeline_library`, ratified as part of the Vulkan 1.3 extension ecosystem) that decomposes a full graphics pipeline compilation into four independently compiled library stages — vertex input, pre-rasterisation shaders, fragment shader, and fragment output interface — allowing each stage to be compiled as soon as its inputs are known rather than waiting for the full pipeline state to be specified; reduces the notorious PSO compilation hitching problem by distributing compilation work earlier in the frame or on a background thread. See Chapter 24 (Vulkan Driver Internals).

_See also_: pipeline cache, disk shader cache, SPIR-V.

---

**ICD** (Installable Client Driver)
A shared library (`.so` on Linux) that fully implements the OpenGL or Vulkan API for a specific GPU vendor; the loader (`libGL.so` / `libGLX.so` for OpenGL, `libvulkan.so.1` for Vulkan) discovers ICDs via JSON manifest files (`/usr/share/glvnd/egl_vendor.d/` for EGL, `/usr/share/vulkan/icd.d/` for Vulkan) and dispatches API calls to the appropriate ICD based on the selected device or context. See Chapter 14 (The Mesa Loader and GLVND).

_See also_: GLVND, Vulkan ICD.

---

**GLVND** (GL Vendor-Neutral Dispatch)
A Linux OpenGL dispatch architecture in which `libGL.so` and `libGLX.so` are neutral stub libraries maintained by NVIDIA and adopted by the Linux graphics ecosystem that route OpenGL/GLX calls to the correct vendor ICD at runtime based on the X11 screen or GLX context; replaces the historical model in which each vendor shipped conflicting `libGL.so` implementations that could not coexist on the same system. See Chapter 14.

_See also_: ICD, WSI.

---

**Kopper**
A Mesa compatibility layer (`src/gallium/frontends/kopper/`) that provides a `KHR_surfaceless`- and DRI3-based Vulkan WSI surface implementation for OpenGL-over-Vulkan (Zink) and other Mesa Gallium drivers that need to present rendered frames to a Wayland or X11 compositor via Vulkan swapchains without requiring a dedicated per-platform WSI code path in the pipe driver; Kopper acts as the bridge between the Gallium `pipe_context` presentation interface and the Vulkan `VK_KHR_swapchain` mechanism. See Chapter 20 (Zink and OpenGL-over-Vulkan).

_See also_: Zink, WSI, EGL.

---

**LLVM** (Low Level Virtual Machine)
The general-purpose compiler infrastructure (`llvm.org`) used by Mesa's `radeonsi` OpenGL driver, the Mesa `amd` OpenCL backend (clover/rusticl), and formerly by `radv`, to lower NIR into GCN/RDNA ISA via LLVM's AMDGPU backend (`lib/Target/AMDGPU/`); Intel's `iris` and `crocus` drivers use LLVM for offline shader pre-compilation; also used for Mesa's `llvmpipe` CPU-based software rasteriser. See Chapter 16.

_See also_: ACO, NIR, ISA.

---

**NIR** (New IR)
Mesa's primary mid-level shader intermediate representation (`src/compiler/nir/`), introduced to replace TGSI, that all Mesa front-ends — GLSL compiler, SPIR-V translator (`spirv_to_nir`), TGSI-to-NIR shim, and the `nine` D3D9 translator — lower their parsed ASTs into; NIR is then subjected to a large library of shared optimisation and lowering passes before each driver backend lowers it further to its native ISA or to LLVM IR; the convergence of all shader input formats at the NIR level is what makes cross-driver shader passes possible. See Chapter 16.

_Misconception_: NIR is not SPIR-V. SPIR-V is the Khronos portable binary format that Vulkan drivers receive from applications; Mesa's SPIR-V front-end ingests SPIR-V and translates it into NIR as the very first compilation step. NIR is Mesa-internal; SPIR-V is the inter-process API boundary.

_See also_: SPIR-V, TGSI, ACO, LLVM, Gallium.

---

**pipe driver**
A Gallium-layer shared library that implements `struct pipe_context` (draw calls, shader binding, resource management) and `struct pipe_screen` (device capabilities, resource creation) for a specific GPU family; examples include `radeonsi` (AMD GCN/RDNA, OpenGL), `iris` (Intel Gen9+, OpenGL), `r600` (AMD pre-GCN), `nouveau` (NVIDIA), `llvmpipe` (CPU software rasteriser), and `virgl` (virtual GPU in QEMU/virglrenderer); the pipe driver receives NIR shaders and compiles them to hardware ISA or LLVM IR. See Chapter 15.

_See also_: Gallium, state tracker, NIR.

---

**pipeline cache** (Vulkan / Mesa)
A `VkPipelineCache` object that serialises compiled pipeline state — including the final GPU ISA binary — to an opaque blob that can be persisted to disk (via `vkGetPipelineCacheData`) and restored in a subsequent session (via `vkCreatePipelineCache` with `pInitialData`), allowing previously compiled PSOs to be reused across frames and application launches without recompilation; Mesa implements this at the Mesa cache layer which also feeds the disk shader cache. See Chapter 24.

_See also_: disk shader cache, GPL, SPIR-V.

---

**SPIR-V** (Standard Portable Intermediate Representation – Vulkan edition)
The Khronos binary shader intermediate format (`spirv.khronos.org`) consumed by Vulkan drivers as the mandatory shader input; produced from GLSL or HLSL source by tools such as `glslangValidator` or `shaderc`, SPIR-V is a variable-length binary stream of 32-bit words encoding a typed SSA intermediate representation of the shader; Mesa's `spirv_to_nir` function (`src/compiler/spirv/spirv_to_nir.c`) translates it into NIR for further optimisation and compilation. See Chapter 23 (Vulkan Shaders and Pipelines).

_Misconception_: SPIR-V is not pre-compiled GPU machine code. It is a portable IR that requires driver-side compilation (through NIR + ACO/LLVM or equivalent) to produce actual GPU instructions. Delivering SPIR-V to a `vkCreateShaderModule` call transfers the IR to the driver; compilation to ISA happens at pipeline creation time (or earlier, with GPL).

_See also_: NIR, TGSI, pipeline cache, Vulkan ICD.

---

**state tracker**
The Gallium component that translates an OpenGL, OpenCL, or D3D9 API call stream into `pipe_context` calls on a Gallium pipe driver; the primary OpenGL state tracker is `st/mesa` (`src/mesa/state_tracker/`), which bridges `libGL`'s `gl_context` to Gallium, while `rusticl` provides the OpenCL state tracker and `nine` provides the D3D9 tracker. See Chapter 15.

_See also_: Gallium, pipe driver.

---

**TGSI** (Tungsten Graphics Shader Infrastructure)
Mesa's legacy shader intermediate representation, introduced with the original Gallium framework (circa 2008) as a RISC-like token stream that Gallium pipe drivers historically consumed directly; substantially superseded by NIR in modern Mesa, with a NIR-to-TGSI lowering pass (`src/gallium/auxiliary/tgsi/tgsi_from_nir.c`) maintaining backward compatibility for the handful of older pipe drivers that have not yet been fully converted. See Chapter 16.

_See also_: NIR, Gallium, pipe driver.

---

**WSI** (Window System Integration)
The Vulkan subsystem that connects a Vulkan device to a window system surface for presentation; defined by platform-specific Khronos extensions — `VK_KHR_wayland_surface`, `VK_KHR_xcb_surface`, `VK_KHR_xlib_surface`, `VK_KHR_display` — combined with `VK_KHR_swapchain` for swapchain management; implemented per-platform inside each Vulkan ICD, with Mesa providing a shared WSI helper library (`src/vulkan/wsi/`) that Vulkan drivers can use. See Chapter 23.

_See also_: Vulkan ICD, Kopper, Zink, GLVND.

---

**Zink**
A Mesa Gallium pipe driver (`src/gallium/drivers/zink/`) that implements OpenGL on top of Vulkan, translating Gallium pipe context calls into Vulkan API calls; allows any Vulkan driver to expose OpenGL without a dedicated GL driver backend, and is useful for CI testing (running OpenGL CTS against Vulkan drivers), hardware bring-up before a native GL driver exists, and running OpenGL applications on hardware where only a Vulkan driver is available. See Chapter 20.

_See also_: Gallium, pipe driver, Kopper, Vulkan ICD.

---

## Section 4: Wayland / Compositor Terms

Wayland is both a wire protocol and an ecosystem of stable and unstable protocol extensions that mediate between the compositor — which controls display hardware via DRM/KMS — and client applications. The terms below describe the core protocol objects, key extension protocols, and integration mechanisms that compositor developers and application developers working with Wayland surfaces need to understand. Where a term defined in another section (such as DRM lease) has specific Wayland implications, a cross-reference stub is provided.

---

**DRM lease** (Wayland context)
See Section 1 (Kernel / DRM Layer Terms) for the full definition. In the Wayland context, VR compositors use DRM leases obtained via `VK_EXT_direct_mode_display` combined with `VK_EXT_acquire_drm_display` to gain direct exclusive display access from within a Wayland session without requiring a standalone compositor.

---

**linux-dmabuf** (`zwp_linux_dmabuf_v1`)
A Wayland protocol extension (`wayland-protocols`, `unstable/linux-dmabuf/`) that allows Wayland clients to submit DMA-BUF file descriptors — accompanied by per-plane format, modifier, offset, and stride parameters — as `wl_buffer` backing storage, enabling zero-copy presentation of GPU-rendered buffers to the compositor without any CPU readback or copy; the compositor uses the buffer's DRM format modifier to import it directly onto a KMS plane via `DRM_IOCTL_MODE_ADDFB2`. See Chapter 26 (Wayland Architecture).

_Misconception_: `linux-dmabuf` does not mandate GPU-rendered content. CPU-written buffers residing in DMA-BUF-exportable memory (e.g., V4L2 capture buffers, `memfd` + `udmabuf`) are equally valid; the protocol is buffer-agnostic.

_See also_: DMA-BUF, DRM format modifier, wl_surface, wl_compositor.

---

**protocol extension** (Wayland)
An XML-defined addition to the Wayland protocol (`wayland-protocols` repository, `gitlab.freedesktop.org/wayland/wayland-protocols`) that introduces new interfaces, requests, events, and enums beyond the stable `wayland.xml` core; extensions progress through `unstable` (breaking changes permitted), `staging` (feature-complete, stabilisation in progress), and `stable` (frozen, no breaking changes) states and are versioned independently of the core protocol version. See Chapter 26.

_See also_: stable protocol, xdg-shell, linux-dmabuf, screencopy.

---

**screencopy**
An informal umbrella term for Wayland protocol extensions that allow privileged clients (screen recorders, screenshot utilities, accessibility tools) to capture compositor output as a DMA-BUF or shared-memory buffer; the wlroots-originated `wlr-screencopy-unstable-v1` and the newer `ext-image-copy-capture-v1` (proposed for promotion to stable) are the primary implementations; access is typically restricted by the compositor to prevent unauthorised screen capture. See Chapter 27 (Wayland Compositors and wlroots).

_See also_: linux-dmabuf, protocol extension, xdg-desktop-portal.

---

**stable protocol** (Wayland)
A Wayland protocol extension whose XML specification has been frozen in the `wayland-protocols` stable/ directory and will not receive breaking changes; current stable protocols include `xdg-shell`, `linux-dmabuf` (v1), `viewporter`, `fractional-scale-v1`, and `xdg-output-unstable-v1` (awaiting promotion). See Chapter 26.

_See also_: protocol extension, xdg-shell.

---

**Wayland socket**
The Unix domain socket at `$XDG_RUNTIME_DIR/wayland-0` (or `wayland-N` for nested compositors) over which the Wayland compositor and its clients exchange binary protocol messages; the compositor creates and binds the socket at startup, and clients connect to it using `wl_display_connect()`; all protocol communication — object creation, event delivery, buffer commits — flows through this socket using the Wayland wire protocol's object/opcode/argument encoding. See Chapter 26.

_See also_: wl_compositor, xdg-shell.

---

**wl_compositor**
The core Wayland global interface (typically bound at object ID 1) that clients use to create `wl_surface` objects (via `wl_compositor.create_surface`) and `wl_region` objects (via `wl_compositor.create_region`); it is the server-side singleton representing the compositor's top-level authority over surface management. See Chapter 26.

_See also_: wl_surface, Wayland socket.

---

**wl_surface**
The fundamental Wayland client-side presentation unit; a client attaches a buffer (backed by `wl_shm` shared memory or a `linux-dmabuf`-backed `wl_buffer`), optionally specifies damage regions (`wl_surface.damage_buffer`), sets buffer transform and scale, and calls `wl_surface.commit` to atomically apply all pending state changes; the compositor displays the committed buffer at the next opportunity, applying its compositing logic. See Chapter 26.

_See also_: wl_compositor, linux-dmabuf, xdg-shell.

---

**xdg-desktop-portal**
A D-Bus service framework (`org.freedesktop.portal.*`) that provides sandboxed applications with compositor-mediated access to privileged desktop resources — file open/save dialogs, screen capture, printer access, notification services, settings, and more — by proxying requests through a trusted `xdg-desktop-portal-*` backend daemon that has direct compositor access; the portal model allows Flatpak-sandboxed applications to request screenshot or screen-cast access without holding Wayland protocol privileges directly. See Chapter 28 (Desktop Integration and Portals).

_See also_: screencopy, xdg-shell.

---

**xdg-shell** (`xdg_wm_base`)
The stable Wayland protocol extension (`stable/xdg-shell/xdg-shell.xml`) that defines application window semantics for Wayland clients; provides `xdg_surface` (the extension role applied to a `wl_surface`), `xdg_toplevel` (a regular application window with title, app-ID, and resize/maximise/fullscreen state), and `xdg_popup` (transient popups with positioner constraints); the configure/ack-configure round-trip handshake allows the compositor to negotiate window size and state with the client before the client commits new buffer dimensions. See Chapter 26.

_See also_: wl_surface, stable protocol, wl_compositor.

---

## Section 5: Application / API Terms

This section covers the Vulkan API's synchronisation, resource binding, and rendering feature vocabulary that graphics application developers and browser rendering engineers encounter when using Vulkan directly or through WebGPU. Several entries (fence, pipeline cache) are duplicated from lower layers with disambiguation notes; those secondary entries appear as cross-reference stubs pointing to the primary definitions.

---

**acceleration structure**
A Vulkan ray-tracing data structure built from geometry using `vkBuildAccelerationStructuresKHR` (`VK_KHR_acceleration_structure`) that organises scene geometry into a two-level hierarchy optimised for ray–primitive intersection queries; the bottom level (BLAS) holds actual triangle or AABB geometry, and the top level (TLAS) contains instances referencing BLASes with per-instance transforms; the driver maps these structures to hardware-specific RT data layouts (NVIDIA BVH, AMD BVH4, etc.). See Chapter 31 (Ray Tracing on Linux).

_See also_: BLAS, TLAS, ray tracing.

---

**binary semaphore** (Vulkan)
A `VkSemaphore` created with `VkSemaphoreType` defaulting to `VK_SEMAPHORE_TYPE_BINARY` (Vulkan 1.0/1.1) that carries a simple unsignalled/signalled state; used to synchronise GPU queue submissions — a `vkQueueSubmit` with `pSignalSemaphores` signals it after completion, and a subsequent `vkQueueSubmit` with `pWaitSemaphores` blocks until it is signalled; the semaphore must be consumed exactly once per signal to avoid undefined behaviour. See Chapter 23 (Vulkan API Fundamentals).

_Misconception_: Binary semaphores cannot be used for CPU-to-GPU or GPU-to-CPU synchronisation; use `VkFence` for CPU-observable GPU completion. For fine-grained GPU–GPU ordering with multiple producers or consumers, prefer timeline semaphores, which avoid the strict one-signal-one-wait constraint.

_See also_: timeline semaphore, fence (Vulkan), drm_syncobj.

---

**bindless**
A Vulkan (and OpenGL `ARB_bindless_texture`) resource access model in which shaders address textures, samplers, storage buffers, and uniform buffers via 64-bit GPU virtual addresses or descriptor indices stored in descriptor buffers rather than through the traditional descriptor-set slot-binding mechanism; enabled in Vulkan via `VK_EXT_descriptor_buffer` (for GPU-managed descriptor heaps) or `VK_EXT_descriptor_indexing` (for variable-length descriptor arrays indexed at runtime); reduces API overhead and enables more flexible batching of heterogeneous draw calls. See Chapter 24 (Vulkan Driver Internals and Optimisation).

_See also_: descriptor set, push constant.

---

**BLAS** (Bottom-Level Acceleration Structure)
The leaf-node component in a Vulkan ray-tracing acceleration structure hierarchy (`VK_KHR_acceleration_structure`), created with `VkAccelerationStructureGeometryKHR` set to `VK_GEOMETRY_TYPE_TRIANGLES_KHR` or `VK_GEOMETRY_TYPE_AABBS_KHR`; contains the actual triangle mesh or bounding-box geometry and is typically built once and reused; multiple BLAS instances with different transforms are referenced by a TLAS to compose the scene. See Chapter 31.

_See also_: TLAS, acceleration structure, ray tracing.

---

**descriptor set**
A Vulkan object (`VkDescriptorSet`) allocated from a `VkDescriptorPool` using a `VkDescriptorSetLayout` that bundles a collection of resource bindings — uniform buffer views (`VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER`), storage buffer views, sampled image + sampler pairs, storage images, and others — into a unit that can be bound to a pipeline at command recording time via `vkCmdBindDescriptorSets`; shaders access the bound resources by set and binding index as declared in SPIR-V. See Chapter 23.

_See also_: bindless, push constant, render pass.

---

**fence** (Vulkan)
A `VkFence` object used to signal the CPU when a `vkQueueSubmit` batch has completed all GPU work; the CPU waits on a fence via `vkWaitForFences` or polls it with `vkGetFenceStatus`; fences may be exported to and imported from `drm_syncobj` file descriptors via `VK_KHR_external_fence_fd`. See Chapter 23.

_Misconception_: Vulkan fences are CPU-facing primitives; they cannot synchronise ordering between GPU queues. Use binary or timeline semaphores for inter-queue dependencies. See also Section 1 for the DRM `dma_fence` definition, which operates at a lower abstraction level.

_See also_: binary semaphore, timeline semaphore, drm_syncobj.

---

**mesh shader**
A programmable Vulkan and OpenGL shader stage (`VK_EXT_mesh_shader`, `GL_NV_mesh_shader`) that replaces the traditional vertex / tessellation / geometry pipeline with a two-stage GPU-driven geometry generation pipeline: the **task shader** (also called amplification shader) performs coarse culling and emits a variable number of mesh shader workgroups, and the **mesh shader** generates meshlets (indexed vertex + primitive packets) directly into the rasteriser's input stream; the model enables GPU-driven rendering of complex scenes with fine-grained per-meshlet LOD and frustum culling without CPU round-trips. See Chapter 25 (Modern GPU Techniques).

_See also_: task shader, subgroup.

---

**pipeline cache** (Vulkan)
See Section 3 (Mesa / Compiler Terms) for the primary definition. At the Vulkan API level, a `VkPipelineCache` is created via `vkCreatePipelineCache` and passed to `vkCreateGraphicsPipelines` / `vkCreateComputePipelines`; the driver populates it with compiled binary artifacts that can be extracted with `vkGetPipelineCacheData` and restored across sessions.

---

**push constant**
A small block of data (minimum guaranteed size: 128 bytes, specified in `VkPhysicalDeviceLimits::maxPushConstantsSize`) embedded directly in the command buffer via `vkCmdPushConstants` and accessible in shaders via the `push_constant` SPIR-V storage class without a descriptor or buffer binding; intended for highly dynamic per-draw data such as transformation matrices, draw IDs, or material parameters that change on every submission and would incur unacceptable descriptor update overhead if stored in descriptor sets. See Chapter 23.

_See also_: descriptor set, render pass.

---

**ray tracing** (Vulkan)
A Vulkan feature set introduced as extensions in Vulkan 1.2 and promoted to core in Vulkan 1.3 (`VK_KHR_ray_tracing_pipeline`, `VK_KHR_ray_query`, `VK_KHR_acceleration_structure`) that provides hardware-accelerated ray–scene intersection via a two-level BVH acceleration structure and dedicated shader stages: ray generation, intersection, any-hit, closest-hit, and miss; requires dedicated RT hardware units (NVIDIA Turing RT Cores, AMD RDNA 2 Ray Accelerators, Intel Xe HPG DXR units); `VK_KHR_ray_query` additionally enables inline ray queries from any shader stage without a full ray tracing pipeline. See Chapter 31.

_See also_: acceleration structure, BLAS, TLAS, mesh shader.

---

**render pass** (Vulkan)
A `VkRenderPass` object (Vulkan 1.0–1.2) or its dynamic equivalent via `VK_KHR_dynamic_rendering` (Vulkan 1.3 core) that describes the set of image attachments — colour, depth, stencil, input — their load/store operations, sample counts, and subpass dependency graph for a GPU rendering operation; drivers use render pass information to optimise tile memory usage on TBDR hardware, schedule resolve operations, and insert necessary pipeline barriers between subpasses. See Chapter 23.

_Misconception_: Render passes are not analogous to OpenGL draw calls. A render pass defines a dependency graph across potentially many draw calls; the driver may reorder or merge internal operations within the declared dependency constraints. On TBDR hardware, the distinction between a render pass that uses `VK_ATTACHMENT_STORE_OP_DONT_CARE` versus `VK_ATTACHMENT_STORE_OP_STORE` has a large bandwidth impact.

_See also_: descriptor set, TBDR, tile memory.

---

**subgroup**
The set of shader invocations that execute in lock-step on a single SIMD unit — equivalent to AMD's "wave" (wave32 or wave64) or NVIDIA's "warp" (32 threads) — exposed to Vulkan shaders via `VK_KHR_shader_subgroup_extended_types` and related extensions as a set of built-in variables (`gl_SubgroupID`, `gl_SubgroupSize`, `gl_SubgroupInvocationID`) and intrinsic operations (ballot, shuffle, reduce) for cross-lane communication without shared memory. See Chapter 25.

_See also_: wave32, wave64, mesh shader.

---

**task shader**
The optional first stage of the mesh-shader pipeline (defined in `VK_EXT_mesh_shader`); each task shader workgroup processes a coarse input unit (e.g., a mesh cluster) and emits a variable-size `gl_MeshPrimitivesEXT`-dimensioned payload along with a dispatch count of mesh shader workgroups to launch, enabling GPU-driven amplification (splitting a large mesh into many meshlets) or culling (emitting zero workgroups for off-screen clusters). See Chapter 25.

_See also_: mesh shader, subgroup.

---

**timeline semaphore**
A `VkSemaphore` created with `VkSemaphoreTypeCreateInfo::semaphoreType` set to `VK_SEMAPHORE_TYPE_TIMELINE` (introduced in Vulkan 1.2, available as `VK_KHR_timeline_semaphore` in Vulkan 1.1) whose payload is a monotonically increasing 64-bit integer; signal and wait operations specify a target payload value rather than operating on binary state, enabling multi-producer/multi-consumer synchronisation patterns, CPU-side signal (`vkSignalSemaphore`) and wait (`vkWaitSemaphores`) without a queue submission, and bidirectional interoperability with `drm_syncobj` timeline points via `VK_KHR_external_semaphore_fd`. See Chapter 23.

_Misconception_: Timeline semaphores are not drop-in replacements for binary semaphores in all contexts. Vulkan swapchain image acquisition (`vkAcquireNextImageKHR`) and presentation (`vkQueuePresentKHR`) still require binary semaphores at the WSI boundary; the swapchain extensions predate and are incompatible with timeline semaphore semantics.

_See also_: binary semaphore, fence (Vulkan), drm_syncobj, explicit sync.

---

**TLAS** (Top-Level Acceleration Structure)
The root node of a Vulkan ray-tracing acceleration structure hierarchy, built with `VkAccelerationStructureGeometryInstancesDataKHR`; contains a flat list of `VkAccelerationStructureInstanceKHR` records each specifying a transform matrix, a BLAS device address, a shader-binding-table record offset, and per-instance flags; the TLAS is traversed first during a ray cast to identify potentially intersected instances before descending into their BLASes; typically rebuilt or updated (`vkBuildAccelerationStructuresKHR` with `VK_BUILD_ACCELERATION_STRUCTURE_ALLOW_UPDATE_BIT_KHR`) each frame for animated scenes. See Chapter 31.

_See also_: BLAS, acceleration structure, ray tracing.

---

**Vulkan ICD**
The shared library that fully implements the Vulkan API for a specific GPU, discovered by the Vulkan loader (`libvulkan.so.1`) via JSON manifest files in `/usr/share/vulkan/icd.d/` or `/etc/vulkan/icd.d/`; examples include `libvulkan_radeon.so` (Mesa `radv`, AMD), `libvulkan_intel.so` (Mesa `anv`, Intel), `libvulkan_nouveau.so` (Mesa `nvk`, NVIDIA), and NVIDIA's proprietary `libGLX_nvidia.so`; each ICD exports a `vkGetInstanceProcAddr` entry point that the loader uses to build the dispatch table. See Chapter 22 (Vulkan Architecture and the Loader).

_See also_: ICD, WSI, SPIR-V, GLVND.

---

## Integrations

This glossary appendix cross-links to every major chapter in the book. The terms in Section 1 (Kernel / DRM) receive their deepest treatment in Chapters 1–5, which cover DRM architecture, the KMS display pipeline, advanced display features, GPU memory management, and the AMDGPU driver. Section 2 (GPU Hardware) terms are elaborated in Chapters 5–10, covering AMD, Intel, NVIDIA, Mali, and other GPU families. Section 3 (Mesa / Compiler) terms are treated in Chapters 14–20, which address the Mesa loader, Gallium architecture, shader compilers, caching, and Zink. Section 4 (Wayland / Compositor) terms receive full treatment in Chapters 26–28, covering the Wayland wire protocol, compositor implementation with wlroots, and desktop integration portals. Section 5 (Application / API) terms are developed in Chapters 22–25 and Chapter 31, covering Vulkan architecture, shaders and pipelines, driver internals, modern GPU techniques, and ray tracing. Browser rendering engineers should additionally consult Chapters 29–32 for how these primitives map into Chromium's GPU process, Dawn/WebGPU, and Skia's Vulkan backend.

---

## References

1. Linux kernel DRM documentation: [https://docs.kernel.org/gpu/drm-kms.html](https://docs.kernel.org/gpu/drm-kms.html)
2. Linux kernel DRM internals: [https://docs.kernel.org/gpu/drm-internals.html](https://docs.kernel.org/gpu/drm-internals.html)
3. Linux kernel DRM memory management: [https://docs.kernel.org/gpu/drm-mm.html](https://docs.kernel.org/gpu/drm-mm.html)
4. DMA-BUF kernel documentation: [https://docs.kernel.org/driver-api/dma-buf.html](https://docs.kernel.org/driver-api/dma-buf.html)
5. Linux kernel UAPI DRM headers: `include/uapi/drm/drm.h`, `include/uapi/drm/drm_mode.h`, `include/uapi/drm/drm_fourcc.h` — [https://github.com/torvalds/linux/tree/master/include/uapi/drm](https://github.com/torvalds/linux/tree/master/include/uapi/drm)
6. Mesa RADV driver documentation: [https://docs.mesa3d.org/drivers/radv.html](https://docs.mesa3d.org/drivers/radv.html)
7. Mesa source tree overview: [https://docs.mesa3d.org/sourcetree.html](https://docs.mesa3d.org/sourcetree.html)
8. Mesa environment variables (driconf, cache paths): [https://docs.mesa3d.org/envvars.html](https://docs.mesa3d.org/envvars.html)
9. ACO compiler README: `src/amd/compiler/README.md` — [https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/compiler/README.md](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/compiler/README.md)
10. AMD RDNA Architecture Whitepaper: [https://gpuopen.com/download/RDNA_Architecture_public.pdf](https://gpuopen.com/download/RDNA_Architecture_public.pdf)
11. AMD Occupancy Explained: [https://gpuopen.com/learn/occupancy-explained/](https://gpuopen.com/learn/occupancy-explained/)
12. AMD RDNA Shader ISA: [https://www.amd.com/content/dam/amd/en/documents/radeon-tech-docs/instruction-set-architectures/rdna-shader-instruction-set-architecture.pdf](https://www.amd.com/content/dam/amd/en/documents/radeon-tech-docs/instruction-set-architectures/rdna-shader-instruction-set-architecture.pdf)
13. Wayland linux-dmabuf protocol: [https://wayland.app/protocols/linux-dmabuf-v1](https://wayland.app/protocols/linux-dmabuf-v1)
14. Wayland linux explicit synchronisation protocol: [https://wayland.app/protocols/linux-explicit-synchronization-unstable-v1](https://wayland.app/protocols/linux-explicit-synchronization-unstable-v1)
15. Explicit sync blog post (Xaver Hugl): [https://zamundaaa.github.io/wayland/2024/04/05/explicit-sync.html](https://zamundaaa.github.io/wayland/2024/04/05/explicit-sync.html)
16. wayland-protocols repository: [https://gitlab.freedesktop.org/wayland/wayland-protocols](https://gitlab.freedesktop.org/wayland/wayland-protocols)
17. Vulkan Specification (Khronos): [https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html)
18. SPIR-V Specification: [https://registry.khronos.org/SPIR-V/specs/unified1/SPIRV.html](https://registry.khronos.org/SPIR-V/specs/unified1/SPIRV.html)
19. GBM header (`gbm.h`): [https://cgit.freedesktop.org/mesa/mesa/tree/src/gbm/main/gbm.h](https://cgit.freedesktop.org/mesa/mesa/tree/src/gbm/main/gbm.h)
20. RADV pipeline and shader compilation (DeepWiki analysis): [https://deepwiki.com/bminor/mesa-mesa/2.1.3-radv-pipeline-and-shader-compilation](https://deepwiki.com/bminor/mesa-mesa/2.1.3-radv-pipeline-and-shader-compilation)
21. drm_syncobj timeline support (LWN): [https://lwn.net/Articles/768998/](https://lwn.net/Articles/768998/)
22. NVIDIA open-gpu-doc: [https://github.com/NVIDIA/open-gpu-doc](https://github.com/NVIDIA/open-gpu-doc)
23. envytools (Nouveau reverse engineering): [https://github.com/envytools/envytools](https://github.com/envytools/envytools)
