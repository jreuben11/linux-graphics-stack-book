# The Linux Graphics Stack: From Kernel to Compositor
## Book Plan

### Audience
This book serves two audiences:
- **Systems and driver developers** — depth on kernel internals, DRM/Mesa architecture, and driver implementation
- **Graphics application developers** — understanding the stack beneath Vulkan, EGL, VA-API, and OpenXR

Chapters signal which perspective is emphasised where they diverge.

---

## Table of Contents

- **Part I — The Kernel Layer**
  - [Chapter 1: DRM Architecture & the Driver Model](#chapter-1-drm-architecture--the-driver-model)
  - [Chapter 2: KMS: The Display Pipeline](#chapter-2-kms-the-display-pipeline)
  - [Chapter 3: Advanced Display Features](#chapter-3-advanced-display-features)
  - [Chapter 4: GPU Memory Management](#chapter-4-gpu-memory-management)
- **Part II — GPU Drivers**
  - [Chapter 5: x86 GPU Drivers](#chapter-5-x86-gpu-drivers)
  - [Chapter 6: ARM & Embedded GPU Drivers](#chapter-6-arm--embedded-gpu-drivers)
- **Part III — The Open NVIDIA Stack**
  - [Chapter 7: Reverse Engineering NVIDIA: History and Methodology](#chapter-7-reverse-engineering-nvidia-history-and-methodology)
  - [Chapter 8: The Nouveau Kernel Driver: nvkm Architecture](#chapter-8-the-nouveau-kernel-driver-nvkm-architecture)
  - [Chapter 9: GSP-RM, Firmware, and the nvidia-open Connection](#chapter-9-gsp-rm-firmware-and-the-nvidia-open-connection)
  - [Chapter 10a: Nova — The Rust NVIDIA Kernel Driver](#nova--the-rust-nvidia-kernel-driver)
  - [Chapter 10b: NVK: Building a Vulkan Driver from Scratch](#chapter-10-nvk-building-a-vulkan-driver-from-scratch)
  - [Chapter 11: Display, Reclocking, and Power Management](#chapter-11-display-reclocking-and-power-management)
- **Part IV — Mesa Architecture**
  - [Chapter 12: The Mesa Loader and Driver Dispatch](#chapter-12-the-mesa-loader-and-driver-dispatch)
  - [Chapter 13: Gallium3D: The OpenGL State Tracker](#chapter-13-gallium3d-the-opengl-state-tracker)
  - [Chapter 14: NIR: Mesa's Shader Intermediate Representation](#chapter-14-nir-mesas-shader-intermediate-representation)
  - [Chapter 15: ACO: AMD's Optimising Compiler](#chapter-15-aco-amds-optimising-compiler)
  - [Chapter 16: Mesa's Vulkan Common Infrastructure](#chapter-16-mesas-vulkan-common-infrastructure)
  - [Chapter 17: Software Renderers](#chapter-17-software-renderers)
- **Part V — Mesa GPU Drivers**
  - [Chapter 18: Vulkan Drivers](#chapter-18-vulkan-drivers)
  - [Chapter 19: OpenGL and Compatibility Drivers](#chapter-19-opengl-and-compatibility-drivers)
- **Part VI — The Display Stack**
  - [Chapter 20: Wayland Protocol Fundamentals](#chapter-20-wayland-protocol-fundamentals)
  - [Chapter 21: Building Compositors with wlroots](#chapter-21-building-compositors-with-wlroots)
  - [Chapter 22: Production Compositors](#chapter-22-production-compositors)
  - [Chapter 23: Legacy and Sandboxed App Support](#chapter-23-legacy-and-sandboxed-app-support)
- **Part VII — Application APIs & Middleware**
  - [Chapter 24: Vulkan and EGL for Application Developers](#chapter-24-vulkan-and-egl-for-application-developers)
  - [Chapter 25: GPU Compute](#chapter-25-gpu-compute)
  - [Chapter 26: Hardware Video](#chapter-26-hardware-video)
  - [Chapter 27: VR & AR](#chapter-27-vr--ar)
  - [Chapter 38: PipeWire and the Video Session Layer](#chapter-38-pipewire-and-the-video-session-layer)
  - [Chapter 39: Qt and GTK GPU Rendering](#chapter-39-qt-and-gtk-gpu-rendering)
- **Part VIII — Gaming Layer**
  - [Chapter 28: Windows Compatibility](#chapter-28-windows-compatibility)
  - [Chapter 29: Upscaling, Effects & Overlays](#chapter-29-upscaling-effects--overlays)
- **Part IX — Tooling & Contributing**
  - [Chapter 30: Debugging & Profiling](#chapter-30-debugging--profiling)
  - [Chapter 31: Conformance & Regression Testing](#chapter-31-conformance--regression-testing)
  - [Chapter 32: Contributing to the Linux Graphics Stack](#chapter-32-contributing-to-the-linux-graphics-stack)
- **Part X — The Browser Rendering Stack**
  - [Chapter 33: Chromium's Multi-Process GPU Architecture](#chapter-33-chromiums-multi-process-gpu-architecture)
  - [Chapter 34: ANGLE — WebGL on Linux](#chapter-34-angle--webgl-on-linux)
  - [Chapter 35: Dawn and WebGPU](#chapter-35-dawn-and-webgpu)
  - [Chapter 36: The Chromium Compositor — CC and Viz](#chapter-36-the-chromium-compositor--cc-and-viz)
  - [Chapter 37: Skia and 2D Rendering](#chapter-37-skia-and-2d-rendering)
- **Part XI — Engines & Creative Tools**
  - [Chapter 40: Bevy and wgpu](#chapter-40-bevy-and-wgpu)
  - [Chapter 41: Godot 4 RenderingDevice](#chapter-41-godot-4-renderingdevice)
  - [Chapter 42: Blender GPU — Cycles and EEVEE](#chapter-42-blender-gpu--cycles-and-eevee)

---

## Part I — The Kernel Layer

### Chapter 1: DRM Architecture & the Driver Model
- What the DRM subsystem is and where it sits in the kernel
- The DRM driver model: probe, bind, component framework
- Device nodes: primary (`/dev/dri/cardN`) vs. render nodes (`/dev/dri/renderDN`)
- Privilege separation between display ownership and rendering
- DRI3: buffer passing via file descriptors, replacing shared memory
- The Present extension: synchronised frame delivery to the display
- How DRM drivers expose capabilities to userspace via ioctls
- **Integrations**: GPU drivers (Ch5, Ch6) register with DRM and implement its driver interface; libDRM (userspace) wraps the ioctls DRM exposes; GBM and Mesa sit above libDRM; KMS (Ch2) is the display half of DRM; the Chromium GPU process (Ch33) opens DRM render nodes to acquire a GPU context for ANGLE (Ch34) and Dawn (Ch35)

### Chapter 2: KMS: The Display Pipeline
- KMS objects: connectors, encoders, CRTCs, planes, framebuffers
- Legacy vs. atomic modesetting: why atomic was necessary
- Atomic commit: TEST_ONLY, non-blocking commits, rollback semantics
- Planes: primary, overlay, cursor; blending and z-order
- Page flipping and VBLANK events
- Damage tracking and partial updates
- The display pipeline from GPU scanout to physical connector
- **Integrations**: GPU drivers implement the KMS CRTC/plane callbacks; GBM allocates the framebuffers KMS scans out; Wayland compositors (Ch20–22) drive KMS via libDRM's modesetting API; advanced features like VRR and HDR (Ch3) extend the KMS atomic property set; the Chromium Viz compositor (Ch36) promotes video and canvas quads directly to KMS overlay planes, bypassing the Wayland compositor entirely in some configurations

### Chapter 3: Advanced Display Features
- VRR and adaptive sync: DRM atomic properties, FreeSync, G-Sync
- HDR: metadata, EOTF, colour volume; KMS colour management extensions
- The KMS colour pipeline: degamma, CTM, gamma; per-plane colour correction
- Explicit sync: the problem with implicit fences and NVIDIA
- `wp_linux_drm_syncobj`: DRM sync objects exposed to Wayland clients
- Content protection: HDCP in KMS
- `wp_color_representation_v1`: the companion protocol to `wp_color_management_v1`; encodes colour matrix (BT.601/BT.709/BT.2020), range (full/limited), and chroma siting metadata separately from the EOTF curve; why this split is necessary for correct HDR video playback through the compositor pipeline
- **Integrations**: Wayland compositors advertise VRR/HDR capabilities via `wp_color_management` and `wp_presentation` (Ch20); explicit sync closes the loop between Mesa drivers producing fences (Ch18, Ch19) and compositors consuming them; applications signal HDR intent via colour metadata on Vulkan swapchains (Ch24); `wp_color_representation_v1` is consumed by VA-API video surfaces (Ch26) handed to the compositor for zero-copy HDR playback

### Chapter 4: GPU Memory Management
- GEM: object lifecycle, handles, reference counting, prime handles
- DMA-BUF: the kernel buffer-sharing framework; importers, exporters, fences
- Implicit vs. explicit fencing on DMA-BUF
- PRIME: multi-GPU buffer passing and render offload
- GBM: allocating EGL-compatible buffers from userspace; the role of libgbm
- DRM format modifiers: tiling layouts, hardware compression, cross-driver negotiation
- The DRM GPU scheduler (`drm_sched`): fair queuing across multiple clients, job priority, preemption, hangcheck timeout, and GPU reset recovery; how a hung shader in one process does not freeze the entire desktop
- Multi-GPU peer-to-peer DMA: the `p2pdma` kernel framework; NVLink and AMD xGMI/Infinity Fabric interconnects; GPU NUMA topologies and NUMA-aware allocation; implications for ML multi-GPU workloads
- **Integrations**: GEM objects are the currency exchanged between GPU drivers, Mesa, and display; DMA-BUF is the zero-copy bridge to VA-API (Ch26), V4L2 (Ch26), Wayland linux-dmabuf (Ch20), and CUDA external memory (Ch25); GBM is the allocation backend EGL uses to create Wayland-presentable surfaces (Ch24); Chromium's SharedImage system (Ch36) is backed by GBM/DMA-BUF objects shared between ANGLE (Ch34), Dawn (Ch35), and Viz; `drm_sched` governs command submission fairness for every driver covered in Parts II–III and V

---

## Part II — GPU Drivers

### Chapter 5: x86 GPU Drivers
- amdgpu: GCN+ architecture, firmware blobs, DCN display engine, PSP security processor
- i915: Intel integrated and Arc discrete; GuC/HuC firmware, the Xe kernel driver split
- nouveau: reverse-engineered NVIDIA; capabilities, constraints, firmware extraction
- nvidia-open: the 2022 open kernel module; what changed, what remains closed, Wayland implications
- Common DRM driver structure: probe, modeset, GEM, GPU scheduler, power management
- AMD HMM and unified memory: Heterogeneous Memory Management, SVM (Shared Virtual Memory), GPU page faults via `amdgpu_mn` migration notifiers; APU zero-copy between CPU and GPU memory domains; implications for ROCm and Steam Deck workloads
- virtio-gpu: the DRM driver for virtualised GPUs (`drivers/gpu/drm/virtio/`); VirGL 3D acceleration over virtio; Venus (Vulkan-over-virtio-gpu); VFIO passthrough for bare-metal GPU performance in VMs; WSL2 GPU path; DRM render node access in containers and Kubernetes GPU nodes
- **Integrations**: each driver implements the DRM/KMS interfaces (Ch1, Ch2); GEM objects produced here are consumed by Mesa drivers (RADV for amdgpu, iris/ANV for i915, NVK for nouveau, Ch18/19); amdgpu's firmware model is examined more deeply in the Nouveau context (Ch9) as a contrast; AMD HMM connects to ROCm (Ch25) and APU display (Ch2); virtio-gpu implements the same DRM/GEM interfaces as real hardware drivers, making it a clean worked example of Ch1–Ch4 abstractions

### Chapter 6: ARM & Embedded GPU Drivers
- Mali GPU families: Midgard, Bifrost (Panfrost), Valhall CSF (Panthor), Mali-400/450 (Lima)
- Qualcomm Adreno and the Turnip Vulkan driver
- Apple Silicon AGX and the Asahi driver; reverse-engineering under NDA constraints
- Embedded-specific challenges: IOMMU, power domains, display subsystem integration, firmware
- The role of Device Tree and ACPI in GPU enumeration on ARM platforms
- The DRM bridge framework: `drm_bridge` and `drm_panel`; bridge chains for complex encoder topologies (SoC → DSI bridge → HDMI transmitter); USB-C Alt Mode DisplayPort; `drm_bridge_attach` and the chain traversal model; `panel-simple` and the panel driver ecosystem
- **Integrations**: these drivers feed the same DRM/GEM/KMS interfaces as x86 drivers; Panfrost/Panthor pair with Mesa's Panfrost driver; Turnip pairs with the Mesa Turnip Vulkan driver (Ch18); the display subsystem integration here is tighter than x86 — DSI panels and MIPI connectors feed directly into KMS plane/CRTC objects (Ch2); the DRM bridge framework connects ARM SoC display engines to the same KMS connector/encoder model described in Ch2

---

## Part III — The Nouveau Story

### Chapter 7: Reverse Engineering NVIDIA: History and Methodology
- Origins: why the open-source community had to reverse-engineer NVIDIA
- Envytools: register databases, decoders, assemblers for NVIDIA microcontrollers
- mmiotrace: capturing MMIO traffic from the proprietary driver
- HWDB: the hardware database driving nouveau's register knowledge
- Community structure: key contributors, the relationship with NVIDIA over the years
- Legal and ethical dimensions of driver reverse engineering
- **Integrations**: the knowledge captured in Envytools/HWDB feeds directly into the nvkm register access layer (Ch8); the same reverse-engineering discipline was applied to the display engine (Ch11) and informed the firmware extraction approach that GSP-RM later superseded (Ch9)

### Chapter 8: The Nouveau Kernel Driver: nvkm Architecture
- nvkm: NVIDIA's abstraction model within nouveau
- The object model: engines, subdevices, falcon microcontrollers
- Channel and pushbuffer management
- The GPU scheduler and fence handling
- Memory management: TTM integration, BO lifecycle
- How nvkm maps across GPU generations (NV04 through Ada)
- **Integrations**: nvkm implements the DRM driver interface (Ch1), exposing GEM/TTM objects consumed by NVK (Ch10) and the legacy GL driver; the fence/scheduler layer connects to DMA-BUF implicit sync (Ch4) and the explicit sync path required by Wayland (Ch3); GSP-RM offloading (Ch9) changes which nvkm engines are active at runtime

### Chapter 9: GSP-RM, Firmware, and the nvidia-open Connection
- What GSP-RM is: the GPU System Processor and its firmware
- How nvidia-open offloads driver logic to GSP-RM
- What the open kernel module exposes vs. what remains in firmware blobs
- Nouveau's GSP-RM support: booting nouveau with NVIDIA's own firmware
- Implications for feature parity, power management, and security
- The path toward a fully open NVIDIA stack
- **Integrations**: GSP-RM changes the interface between nvkm and the GPU hardware (Ch8); using GSP-RM firmware unlocks reclocking and power management that were previously blocked (Ch11); the DMA-BUF/explicit sync improvements enabled by nvidia-open are what unblocked proper NVIDIA Wayland support (Ch3, Ch20)

### Nova — The Rust NVIDIA Kernel Driver

- Why a new driver rather than extending nouveau: design constraints of nvkm, opportunity of GSP as a clean hardware abstraction boundary
- **nova-core**: 1st-level driver; boots the GSP, manages the command queue, abstracts hardware without register-level access; sits outside the DRM tree as a platform driver
- **nova-drm**: 2nd-level DRM driver (`drivers/gpu/drm/nova/`); implements standard DRM interfaces for userspace; consumes nova-core's GSP abstraction
- Rust implementation rationale: memory safety in a new driver with no legacy C debt; Rust-for-Linux abstractions used (GPUVM immediate-mode, HRT device driver support, Falcon firmware handling)
- GPU generation scope: Turing (RTX 20xx) and newer; pre-Turing hardware stays with nouveau
- Development status as of Linux 7.2: Turing GSP bringup, Falcon firmware hardening, large RPC support, DebugFS for GSP-RM log buffers
- Co-existence with nouveau in the kernel tree: module selection, migration path for users
- Contributor landscape: Red Hat (primary), NVIDIA engineers; GSP-RM firmware as the shared dependency with nvidia-open
- **Integrations**: nova-core is the primary consumer of GSP-RM firmware (Ch 9); nova-drm implements the DRM driver model (Ch 1) and will become the kernel backend for NVK (Ch 10b) as it matures, replacing nvkm (Ch 8) for Turing+ GPUs; the Rust-in-kernel approach echoes the ARM Tyr driver (Ch 6); contributing to nova-drm uses the Rust DRM contribution workflow described in Ch 32

### Chapter 10b: NVK: Building a Vulkan Driver from Scratch
- Motivation: starting fresh rather than layering on the legacy GL driver
- Design decisions: object model, memory heaps, descriptor set architecture
- Shader compilation pipeline: SPIR-V → NIR → nvk backend
- Implementing Vulkan synchronisation on nouveau's channel model
- Current feature coverage and Vulkan conformance status
- Lessons applicable to writing any new Mesa Vulkan driver
- **Integrations**: NVK consumes GEM/TTM objects from nvkm (Ch8) and feeds Mesa's Vulkan common infrastructure (Ch16) and NIR compiler pipeline (Ch14); it is a conformance target for dEQP-VK (Ch31) and a client of the explicit sync protocol (Ch3); DXVK and VKD3D-Proton (Ch28) run on NVK for NVIDIA gaming on Linux

### Chapter 11: Display, Reclocking, and Power Management
- The NV50 display engine: heads, DACs, SOR outputs
- HDMI/DisplayPort support and limitations on nouveau
- Reclocking: the challenge of undocumented clock tables; current state per GPU generation
- Power management: runtime PM, voltage scaling, fan control
- Thermal limits and throttling on nouveau
- Comparing nouveau's power behaviour to the proprietary driver
- **Integrations**: the display engine integrates with KMS (Ch2) via DRM connector and CRTC callbacks; reclocking directly affects the performance available to Mesa/NVK workloads (Ch10, Ch18); power management hooks into the DRM runtime PM framework and influences how the GPU scheduler (Ch8) throttles command submission

---

## Part IV — Mesa Architecture

### Chapter 12: The Mesa Loader and Driver Dispatch
- libGL and libvulkan: EGL/GLX entry points and ICD dispatch
- GLVND: vendor-neutral dispatch; how multiple OpenGL drivers coexist
- The DRI driver interface: how Mesa userspace talks to kernel drivers
- Driver selection: environment variables, PCI IDs, device capabilities
- Mesa's disk shader cache: keying strategy, invalidation, size limits
- Steam shader pre-compilation and cache warming
- Vulkan WSI architecture: `VkSurfaceKHR`, `VkSwapchainKHR`, Mesa's `src/vulkan/wsi/` common layer; `wsi_common_wayland.c` and `wsi_common_drm.c`; DRM format modifier negotiation in the swapchain; platform extensions (`VK_KHR_wayland_surface`, `VK_KHR_display`); why `currentExtent` is `UINT32_MAX` on Wayland and what it means for application code
- **Integrations**: the loader opens render nodes (Ch1) via libDRM; GLVND sits between applications and Mesa ICDs, enabling NVIDIA's proprietary driver and Mesa to coexist; the DRI interface here is what EGL (Ch24) and GBM (Ch4) use to obtain drawables; the disk cache feeds compiled shaders into all Mesa hardware drivers (Ch18, Ch19); WSI common layer is the shared infrastructure all Mesa Vulkan drivers (Ch18) use to present frames to Wayland (Ch20)

### Chapter 13: Gallium3D: The OpenGL State Tracker
- Gallium3D design goals: separating API state from hardware
- The pipe driver interface: resource creation, draw calls, transfers
- CSO (Constant State Objects): pre-compiled rasteriser, blend, depth states
- The u_blitter and u_transfer_helper utilities
- How Gallium state trackers implement OpenGL, OpenGL ES, and OpenCL
- Texture, sampler, and framebuffer object management
- **Integrations**: Gallium's pipe driver interface is what radeonsi and iris (Ch19) implement; rusticl (Ch25) uses the same pipe driver interface to expose OpenCL on any Gallium driver; NIR (Ch14) is the shader IR passed through the pipe interface; Zink (Ch19) implements the pipe interface by translating calls to Vulkan (Ch18)

### Chapter 14: NIR: Mesa's Shader Intermediate Representation
- Why Mesa needed a new IR: limitations of TGSI
- NIR structure: functions, blocks, instructions, SSA variables
- The SPIR-V → NIR front end: capability handling, decorations, types
- The GLSL → NIR front end
- Key optimisation passes: algebraic, copy propagation, dead code elimination
- Lowering passes: how platform constraints are encoded
- The NIR validator and debugging tools
- NIR as the universal interchange between all Mesa front ends and backends
- **Integrations**: every Mesa driver receives shaders as NIR — RADV/ANV/NVK pass NIR to ACO or LLVM (Ch15); radeonsi/iris pass NIR to their LLVM backends (Ch19); Lavapipe/llvmpipe (Ch17) consume NIR via LLVM; the SPIR-V front end connects Vulkan applications (Ch24) and DXVK/VKD3D shader translation (Ch28) into the Mesa pipeline; SPIR-V emitted by Tint (Ch35) and glslang/ANGLE (Ch34) enters the same NIR front end, making browser shader workloads first-class NIR consumers

### Chapter 15: ACO: AMD's Optimising Compiler
- Motivation: LLVM compile times and suboptimal code generation for RADV
- ACO's place in the RADV pipeline: NIR → ACO IR → GCN/RDNA assembly
- Instruction selection: mapping NIR ops to GCN instructions
- Register allocation: the VGPR/SGPR split; spilling strategy
- Instruction scheduling and VALU/SALU interleaving
- Wave32 vs. Wave64 on RDNA
- Benchmark impact: compile time and runtime performance vs. LLVM
- When LLVM is still used (compute, radeonsi)
- **Integrations**: ACO consumes NIR from the shared Mesa pipeline (Ch14) and produces GCN/RDNA machine code submitted to the amdgpu kernel driver (Ch5) via RADV command buffers (Ch18); faster compile times directly benefit game startup via the disk cache (Ch12) and async compilation in DXVK (Ch28)

### Chapter 16: Mesa's Vulkan Common Infrastructure
- The vk_object model: type-tagged allocation, device-child lifetime
- Render pass lowering: transforming Vulkan render passes to driver primitives
- Descriptor set infrastructure: set layout, pool, update templates
- Pipeline cache serialisation and portability
- Common Vulkan extensions implemented once for all drivers: maintenance, synchronisation2, dynamic state
- The vk_pipeline_cache and cross-driver shader key design
- **Integrations**: every Mesa Vulkan driver (RADV, ANV, NVK, Turnip, Ch18) builds on this layer; the pipeline cache connects to the Mesa disk cache (Ch12); descriptor set and render pass lowering sit between Vulkan application calls (Ch24) and NIR/ACO compilation (Ch14, Ch15); Vulkan Validation Layers (Ch30) hook in above this layer

### Chapter 17: Software Renderers
- llvmpipe: OpenGL/ES via LLVM JIT; tile-based rasterisation, threading model
- Lavapipe: CPU-based Vulkan built on llvmpipe primitives
- Zink: implementing OpenGL on top of Vulkan; architectural trade-offs, performance ceiling
- Use cases: CI without GPU hardware, CI conformance, Wayland compositor fallback, driver bringup
- Performance characteristics and known limitations of each
- **Integrations**: llvmpipe and Lavapipe consume NIR via LLVM (Ch14); they are primary targets for dEQP and piglit in headless CI (Ch31); Wayland compositors (Ch21, Ch22) fall back to llvmpipe when no GPU KMS backend is available; Zink sits above the Vulkan driver layer (Ch18) and beneath the Gallium OpenGL state tracker (Ch13)

---

## Part V — Mesa GPU Drivers

### Chapter 18: Vulkan Drivers
- RADV: AMD Vulkan in Mesa; pipeline caching, descriptor indexing, mesh shaders, ray tracing
- ANV: Intel Vulkan; bindless heap design, Xe2 architecture changes
- Turnip: Qualcomm Adreno Vulkan; tile-based rendering, sysmem vs. gmem paths
- Common bringup patterns: what every new Mesa Vulkan driver must implement first
- Conformance testing with dEQP-VK; handling failing tests in CI
- **Integrations**: these drivers consume GEM objects from amdgpu/i915/msm (Ch5, Ch6), feed ACO/NIR (Ch14, Ch15) for shader compilation, and implement Mesa's Vulkan common layer (Ch16); DXVK and VKD3D-Proton (Ch28) are their most demanding clients; gamescope (Ch22) and vkBasalt (Ch29) intercept the Vulkan API layer above them; RenderDoc (Ch30) captures frames at this boundary; ANGLE (Ch34) and Dawn (Ch35) are significant Vulkan clients whose shader and memory traffic runs through these same drivers

### Chapter 19: OpenGL and Compatibility Drivers
- radeonsi: AMD OpenGL/ES on GCN+; NIR backend, Shader DB regression tracking
- iris: Intel OpenGL/ES on Gen 8+; batch execution, hardware workarounds, Gen12 changes
- Zink as a portability layer: when to use it vs. a native GL driver
- The long tail: maintaining OpenGL 4.6 compatibility on modern hardware
- ARM and embedded OpenGL ES drivers: `panfrost` (Mali Midgard/Bifrost — `src/gallium/drivers/panfrost/`), `panthor` (Mali Valhall CSF — newer Gallium driver replacing the old Bifrost path), `lima` (Mali-400/450 — `src/gallium/drivers/lima/`), `etnaviv` (Vivante GC series — Vivante ISA reverse-engineered, Gallium driver), `freedreno` (Qualcomm Adreno OpenGL ES — `src/gallium/drivers/freedreno/`); how each implements the Gallium pipe driver interface; common challenges: UAPI differences, tile-based memory layouts, firmware interactions; these represent the majority of shipped Linux SoC graphics and are increasingly ARM-only CI targets
- **Integrations**: radeonsi and iris implement the Gallium pipe driver interface (Ch13) and consume NIR (Ch14); panfrost/lima/etnaviv/freedreno implement the same Gallium interface, feeding the same NIR pipeline from ARM kernel drivers (Ch6); rusticl's OpenCL (Ch25) runs on the same radeonsi/iris pipe drivers; Wine's OpenGL path (Ch28) calls into these drivers for games that use D3D9 via DXVK's OpenGL fallback; Glamor's 2D acceleration (used by XWayland, Ch23) calls OpenGL ES and lands here

---

## Part VI — The Display Stack

### Chapter 20: Wayland Protocol Fundamentals
- The Wayland object model, wire protocol, and event loop
- Core interfaces: wl_compositor, wl_surface, wl_output, wl_seat
- Key extension protocols: xdg-shell, linux-dmabuf, wp_presentation, wp_color_management, wp_fifo
- How GPU buffers flow from application to compositor to screen
- Wayland security model: no global input snoop, no arbitrary window access
- Protocol design principles and the extension stability guarantee
- **Integrations**: linux-dmabuf consumes DMA-BUF handles produced by GBM/Mesa (Ch4, Ch24); wp_presentation feeds timing data back to Vulkan swapchains (Ch24) and game frame-pacing systems (Ch29); wp_color_management connects to KMS colour pipelines (Ch3); the Wayland event loop is driven by the compositor's DRM/KMS backend (Ch2); Chrome's Viz compositor (Ch36) submits SharedImage-backed DMA-BUF buffers via linux-dmabuf, and consumes wp_presentation BeginFrame feedback to pace WebGPU (Ch35) and WebGL (Ch34) frames

### Chapter 21: Building Compositors with wlroots
- wlroots architecture: backend abstraction, renderer, scene graph
- Backend types: DRM/KMS, nested Wayland, X11
- The scene graph API: layers, buffers, damage tracking, output layout
- Input handling via libinput: seats, capabilities, gesture recognition
- libinput in depth: the evdev kernel event layer; libinput's device abstraction and quirk database; gesture recognition (pinch, swipe, rotate); touch, stylus, and pen tablet protocol support; the full evdev→libinput→wlr_seat→wl_pointer event chain; rate limiting and frame delivery; how compositors implement pointer constraints and relative motion (`zwp_pointer_constraints_v1`, `zwp_relative_pointer_v1`)
- Implementing output management, idle inhibit, and screencopy protocols
- Walkthrough: a minimal compositor from scratch
- **Integrations**: wlroots' DRM backend drives KMS atomic commits (Ch2) and allocates GBM buffers (Ch4); its Vulkan renderer uses Mesa Vulkan drivers (Ch18) for compositing; Sway and Hyprland (Ch22) are direct consumers of wlroots' API; XWayland (Ch23) embeds as a client of wlroots-based compositors; PipeWire screen capture (Ch26) is implemented via wlroots' screencopy protocol; the libinput event chain described here is what Chrome's Ozone/Wayland backend (Ch33) and Qt/GTK applications (Ch39) receive as `wl_pointer` events

### Chapter 22: Production Compositors
- Mutter: GNOME Shell integration, Clutter scene graph, effects pipeline, colour management
- KWin: KDE Plasma, scripting API, HDR/VRR support, tiling layouts
- Sway: i3 compatibility, wlroots-based, IPC protocol, configuration model
- Hyprland: animation system, rounded corners, dynamic tiling, plugin API
- gamescope: Valve's micro-compositor; nested sessions, FSR upscaling, Steam Deck deployment, HDR
- cosmic-comp: System76's COSMIC desktop compositor, written in Rust using `smithay`; the `smithay` Wayland server library as a wlroots-equivalent in Rust; how cosmic-comp implements the same DRM/KMS and Wayland protocols without C dependencies; the COSMIC desktop as a case study in an entirely Rust compositor stack (cosmic-comp + iced UI toolkit); implications for the Rust-in-graphics-stack theme running through Parts III and XI
- **Integrations**: all compositors drive KMS/DRM (Ch2) for display and consume Vulkan or OpenGL (Ch18, Ch19) for effects rendering; gamescope runs Proton/DXVK games (Ch28) as nested Wayland clients and applies FSR (Ch29) as a final pass before KMS scanout; PipeWire (Ch26) receives screen capture via the compositor's screencopy implementation; xdg-desktop-portal (Ch23) proxies portal requests to the active compositor; cosmic-comp's smithay-based DRM backend exercises the same KMS atomic commit and GBM allocation paths (Ch2, Ch4) as wlroots-based compositors, and is architecturally related to the Rust DRM abstractions used in nova-drm

### Chapter 23: Legacy and Sandboxed App Support
- XWayland: rootless mode, input translation, clipboard, DPI scaling, explicit sync handshake
- xdg-desktop-portal: architecture, D-Bus interfaces, screen capture, GPU surface access
- Flatpak and the portal security model
- Practical: running unmodified X11 applications on a Wayland desktop
- **Integrations**: XWayland renders X11 client content via Mesa OpenGL (Ch19) and presents it to the Wayland compositor via linux-dmabuf (Ch20); Glamor (Ch19) accelerates 2D X11 drawing inside XWayland; xdg-desktop-portal routes screen capture requests to PipeWire (Ch26) and uses DMA-BUF (Ch4) for zero-copy GPU surface sharing with sandboxed apps

---

## Part VII — Application APIs & Middleware

### Chapter 24: Vulkan and EGL for Application Developers
- Vulkan on Linux: instance extensions, physical device selection, queue families
- Memory types on AMD, Intel, and NVIDIA: device-local, host-visible, BAR
- EGL: display, context, and surface creation on Wayland and GBM
- Swapchains on Wayland: VK_KHR_wayland_surface, present modes, tearing control
- Synchronisation: fences, binary semaphores, timeline semaphores, host sync
- Integrating Vulkan Validation Layers into a development workflow
- **Integrations**: EGL surface creation uses GBM (Ch4) for KMS-backed surfaces and the Wayland linux-dmabuf protocol (Ch20) for compositor-presented surfaces; Vulkan swapchains feed frames into the Wayland compositor (Ch22) via the Present extension (Ch1); timeline semaphores are the application-visible face of the explicit sync mechanism that runs down to DRM sync objects (Ch3); ANGLE (Ch34) uses EGL for context and surface management, making the EGL path described here the same one Chrome traverses for WebGL rendering

### Chapter 25: GPU Compute
- OpenCL on Linux: rusticl (Mesa), ROCm, and the fragmented ICD landscape
- ROCm ecosystem: HIP runtime, rocBLAS, rocFFT, MIOpen; AMD-specific constraints
- Vulkan compute: compute pipelines, descriptor sets, push constants, subgroups
- **CUDA–Vulkan interop**: sharing data across the two APIs
  - Vulkan external memory (`VK_KHR_external_memory_fd`): exporting allocations as POSIX fd / DMA-BUF handles
  - CUDA external memory (`cudaExternalMemoryHandleType_OpaqueFd`): importing Vulkan allocations into CUDA
  - Vulkan external semaphores + CUDA external semaphores: synchronising across APIs without CPU round-trips
  - Practical patterns: CUDA image processing → Vulkan display; CUDA physics simulation → Vulkan visualisation
  - Pitfalls: memory coherency, queue ownership transfer, validation layer coverage of interop paths
- Intel GPU compute stack: Level Zero — Intel's low-level GPU compute API (`intel/compute-runtime`); the `ze_` API surface; how it sits below SYCL and oneAPI; `intel-compute-runtime` as the ICD serving both OpenCL 3.0 and Level Zero; the relationship to `i915`/`Xe` kernel drivers (Ch5) via DRM render nodes; `intel/metrics-discovery` for hardware performance counters; oneAPI DPC++ and SYCL: the LLVM-based compiler pipeline, how SYCL kernels compile to SPIR-V and then to GEN ISA via the IGC (Intel Graphics Compiler); comparison with ROCm/HIP in terms of portability and Linux support trajectory
- OpenCL 3.0 conformance on Linux: `rusticl`'s conformance status on radeonsi/iris (Khronos certified as of Mesa 24.x); `intel-compute-runtime` OpenCL 3.0 conformance for Arc and integrated; conformance gaps and optional feature coverage
- Interop: sharing DMA-BUF buffers between graphics and compute pipelines more broadly
- Practical: GPU-accelerated image processing with Vulkan compute
- **Integrations**: rusticl runs on Gallium pipe drivers (Ch13) — the same radeonsi/iris used for OpenGL (Ch19); ROCm uses amdgpu's compute queue exposed via DRM render nodes (Ch1); Level Zero uses i915/Xe compute queues via the same DRM render node model (Ch1, Ch5); CUDA-Vulkan interop uses the DMA-BUF/GEM export mechanism (Ch4) at the kernel level; Vulkan compute pipelines share the same NIR/ACO compilation path as graphics pipelines (Ch14, Ch15); WebGPU compute shaders (Ch35) use the same Vulkan compute queue and NIR/ACO path, making Dawn a browser-side peer to the native compute frameworks covered here; Blender's Cycles oneAPI backend (Ch42) is a direct consumer of Level Zero on Intel hardware

### Chapter 26: Hardware Video
- VA-API: decode/encode pipeline, surface formats, DMA-BUF export for zero-copy display
- VDPAU: NVIDIA's video decode path; interop with Vulkan and display
- V4L2: kernel video capture, stateless codec interfaces, request API
- libcamera: the V4L2 Media Controller Pipeline abstraction; IPA (Image Processing Algorithm) plugin API; libcamera pipeline handlers for Raspberry Pi CSI, i.MX8, and ISP devices; the `CameraManager`, `Camera`, `Request`/`FrameBuffer` lifecycle; integration with PipeWire via `libcamera-vid` and `libcamera-apps`; Android Camera HAL3 bridge; sensor configuration and RAW capture; why V4L2's `request_api` was not enough and libcamera fills the gap
- GStreamer: pipeline construction, VA-API and V4L2 elements, zero-copy buffer passing
- Practical: hardware-accelerated transcoding and playback pipelines
- **Integrations**: VA-API surfaces are DMA-BUF objects (Ch4) that can be imported directly into Vulkan (Ch24) or handed to the Wayland compositor (Ch20) for zero-copy display; V4L2 capture feeds the same DMA-BUF pipeline; libcamera's DMA-BUF output frames feed directly into PipeWire (Ch38) for session-level routing to multiple consumers; PipeWire acts as the session-level broker routing camera and screen-capture buffers (Ch21, Ch22) to consuming applications; GStreamer's VA-API elements sit atop the Mesa VA-API drivers which run on amdgpu/i915 (Ch5)

### Chapter 27: VR & AR
- OpenXR: session lifecycle, swapchain management, action system, compositor layers
- Monado: runtime architecture, driver interface, tracking systems
- Reprojection, timewarp, and latency on Linux
- Wayland and OpenXR integration; direct mode display
- SteamVR runtime on Linux: how SteamVR implements the OpenXR runtime interface alongside Monado; why users choose SteamVR (broader driver support for Valve Index, Vive) vs. Monado (fully open, no Valve dependency); the `XR_VALVE_steam_gamepad` extension and Steam Input passthrough; SteamVR's Vulkan path vs. OpenGL fallback; co-existence of multiple runtimes via `openxr_loader` active-runtime JSON
- Hand tracking and extended interaction: `XR_EXT_hand_tracking` — hardware that supports it on Linux (Quest via SteamVR Link, Ultraleap); `XR_EXT_eye_gaze_interaction` — Tobii and VIVE Pro Eye on Linux; `XR_FB_passthrough`-equivalent for camera AR on open hardware; how these extensions layer on top of the session and frame loop described in the Monado section
- **Integrations**: Monado drives the headset display via DRM direct mode (Ch2) and allocates swapchain images via GBM/Vulkan (Ch4, Ch24); reprojection shaders compile through the same NIR/Mesa pipeline (Ch14); OpenXR applications use the same Vulkan queues and synchronisation primitives as desktop apps (Ch24); tracking data from V4L2 cameras (Ch26) feeds Monado's SLAM pipeline; SteamVR on Linux uses the same Mesa Vulkan drivers (Ch18) and, for the Wayland path, the same linux-dmabuf swapchain (Ch20)

### Chapter 38: PipeWire and the Video Session Layer
- PipeWire architecture: the graph model (`pw_node`, `pw_port`, `pw_link`); the event loop and `pw_loop`; SPA (Simple Plugin API) as the low-level buffer negotiation layer; the `pw_stream` abstraction for producers and consumers
- Session management: PipeWire as a replacement for PulseAudio and JACK; `wireplumber` session manager; policy engine; device enumeration via udev/ALSA/V4L2; routing rules and metadata
- Video capture pipeline: V4L2 source node; `libcamera` → PipeWire adapter (Ch26); DMA-BUF buffer negotiation with `SPA_DATA_DmaBuf`; zero-copy path from camera ISP to consumer
- Screen capture and portal integration: `xdg-desktop-portal` PipeWire backend; `pw-capture` for compositor screencopy (Ch21, Ch22); OBS Studio as a DMA-BUF consumer; PipeWire's role in the Wayland screen share stack replacing X11 shared memory
- Remote desktop: `xdg-desktop-portal` remote-desktop interface; PipeWire streaming to RDP/VNC backends; DMA-BUF→CPU copy when the remote encoder needs raw pixels
- Buffer formats and GPU interop: `SPA_VIDEO_FORMAT_*` and DRM format mapping; DMA-BUF import into Vulkan `VkImage` (Ch24); explicit sync (`DRM_FORMAT_MOD_INVALID` path vs. explicit fence passing); GPU-accelerated encode handoff to VA-API (Ch26)
- **Integrations**: PipeWire consumes libcamera DMA-BUF frames (Ch26) and compositor screencopy buffers (Ch21, Ch22); its DMA-BUF negotiation relies on GEM/DMA-BUF infrastructure (Ch4); GPU consumers import PipeWire buffers via the same Vulkan external memory path used by ANGLE (Ch34) and Dawn (Ch35); `xdg-desktop-portal` routes screen capture from wlroots (Ch21) and Mutter (Ch22) through PipeWire to OBS and browser tab capture (Ch33)

### Chapter 39: Qt and GTK GPU Rendering
- Qt6 rendering architecture: the `QRhi` (Qt Rendering Hardware Interface) abstraction; backends — Vulkan, OpenGL, Metal, D3D12; `QSGRenderNode` and the scene graph; `QQuickRenderControl` for off-screen rendering
- Qt Wayland integration: `QPA` (Qt Platform Abstraction); `qwayland` QPA plugin; `xdg-shell` surface creation; linux-dmabuf swapchain; `zwp_linux_explicit_synchronization_v1` handshake (Ch3, Ch20)
- Qt shader pipeline: Qt Shader Tools (`qsb`); GLSL → SPIR-V via `glslang`; SPIR-V → HLSL/MSL cross-compilation; how Qt-emitted SPIR-V enters Mesa NIR (Ch14); the QSBC shader cache
- GTK4 rendering architecture: `GskRenderer` and its backends — `GskGLRenderer`, `GskVulkanRenderer`, `GskNglRenderer`; the scene graph node types; CSS rendering model
- GTK4 Wayland integration: `GDK` Wayland backend; `GdkSurface`/`GdkVulkanContext`; linux-dmabuf buffer submission; explicit sync support (Ch3, Ch20)
- GTK4 shader pipeline: GLSL shaders in GTK source; shader compilation at startup; fallback paths; GPU-accelerated CSS effects (blur, colour matrix, clip masks)
- Font and text rendering in Qt/GTK: FreeType + HarfBuzz shaping; glyph atlas management; comparison with Skia's text path (Ch37); fontconfig integration
- **Integrations**: Qt6 `QRhi` Vulkan backend and GTK4 `GskVulkanRenderer` are clients of Mesa Vulkan drivers (Ch18); their SPIR-V feeds NIR (Ch14); Wayland surface creation uses linux-dmabuf (Ch20) and the explicit sync protocol (Ch3); PipeWire (Ch38) is the capture backend for Qt Multimedia and GStreamer-GTK widgets; wlroots-based compositors (Ch21) and Mutter (Ch22) serve their Wayland surfaces; their input events arrive via the libinput event chain (Ch21)

---

## Part VIII — Gaming Layer

### Chapter 28: Windows Compatibility
- Wine: PE loader, Win32 API surface, wineserver, graphics driver shim
- DXVK: D3D9/10/11 → Vulkan translation; state caching, async pipeline compilation
- VKD3D-Proton: D3D12 → Vulkan; divergence from upstream vkd3d, DXR ray tracing
- Proton: pressure-vessel container, Steam Runtime, compatibility database
- Performance characteristics and remaining gaps
- **Integrations**: DXVK and VKD3D-Proton are heavy consumers of Mesa Vulkan drivers (Ch18) — RADV's ACO compiler (Ch15) and pipeline caching (Ch12) are tuned partly for their workloads; async shader compilation in DXVK feeds NIR compilation jobs (Ch14) off the render thread; Proton runs inside gamescope (Ch22) for Steam Deck; Wine's OpenGL path calls into Mesa's radeonsi/iris (Ch19) for games that bypass DXVK

### Chapter 29: Upscaling, Effects & Overlays
- FSR 1/2/3: spatial vs. temporal upscaling algorithms; gamescope integration; Vulkan layer mode
- vkBasalt: Vulkan layer pipeline interception; CAS, SMAA, FXAA, depth of field effects
- MangoHud: Vulkan/OpenGL overlay; frame timing graphs, GPU/CPU metrics, log export
- XeSS (Intel Xe Super Sampling): Intel's spatial/ML upscaler; the `libxess` Vulkan library; the generic Vulkan path (`XeSS_INIT_FLAG_USE_NDC_VELOCITY`) that works on any Vulkan GPU including RADV and NVK; XeSS vs. FSR tradeoffs on Intel Arc vs. AMD hardware
- DLSS on Linux: NVIDIA's Deep Learning Super Sampling via the NGX SDK; why DLSS requires proprietary NVIDIA driver support; the `wine-nvml` and DLSS-to-FSR translation layers (e.g. `LatencyFleX`/`DLSS2FSR`) for running DLSS games via Wine/Proton without NVIDIA hardware; `NvAFX` audio effects SDK as a companion NGX consumer
- LatencyFleX: frame latency reduction layer; how it hooks `VkQueue` submission timing to implement a software-side framepacing feedback loop without special driver support; comparison with NVIDIA's Reflex; integration with MangoHud for latency visualisation
- GameMode (Feral Interactive): the `gamemoded` daemon; CPU scheduler (`SCHED_BATCH`→`SCHED_OTHER` on game start), I/O scheduler, GPU performance mode (`amdgpu_power_profile_mode`, `nvidia-smi -pm 1`), kernel parameter tuning (`vm.swappiness`, split lock threshold); D-Bus API and how Proton games trigger it automatically via `LD_PRELOAD` shim
- **Integrations**: FSR runs as a Vulkan compute shader (Ch25) inside gamescope (Ch22), consuming the game's final colour buffer via DMA-BUF (Ch4); vkBasalt intercepts between the application and Mesa Vulkan drivers (Ch18) using the Vulkan layer mechanism, inserting post-process passes before the swapchain present (Ch24); MangoHud hooks the same layer interface to read GPU performance counters exposed via DRM (Ch1); XeSS on RADV/NVK exercises the same generic Vulkan compute path as FSR; GameMode's GPU performance mode interacts with amdgpu's power profile sysfs (Ch5)

---

## Part IX — Tooling & Contributing

### Chapter 30: Debugging & Profiling
- RenderDoc: frame capture workflow, resource inspector, shader debugger, replay API
- Vulkan Validation Layers: error categories, best practices layer, GPU-assisted validation, sync validation
- Mesa environment variables: MESA_DEBUG, LIBGL_DEBUG, NIR dump flags, RADV_DEBUG
- GPU hardware performance counters: AMD RadeonGPUProfiler, Intel GPA, NVIDIA Nsight
- Vendor CLI tools and lightweight profilers: `umr` (User Mode Register debugger for AMD — live MMIO register inspection, wave state dump, packet trace without GPU halt; `src/` at [https://gitlab.freedesktop.org/tomstdenis/umr](https://gitlab.freedesktop.org/tomstdenis/umr)); `intel_gpu_top` (Intel GPU engine utilisation by ring — render, blitter, video, compute; reads from `/sys/class/drm/` and perf); `gfxreconstruct` (Vulkan API trace capture and deterministic replay as an alternative to RenderDoc — suited to headless, multi-GPU, and long-capture scenarios; `VkLayer_gfxreconstruct`)
- SPIR-V toolchain for shader debugging: `spirv-val` (Khronos validator — catches spec violations before driver submission); `spirv-opt` (SPIRV-Tools optimiser — apply SPIR-V-level passes, useful for isolating whether a bug is in the SPIR-V or in the driver's NIR front end); `spirv-cross` (cross-compiler — SPIR-V → GLSL/HLSL/MSL for inspection); `spirv-dis` (disassembler — human-readable SPIR-V); how to use these in conjunction with `RADV_DEBUG=spirv` or `ANV_ENABLE_PIPELINE_CACHE=0` to capture and inspect driver-consumed SPIR-V
- GPU security and isolation: render node (`/dev/dri/renderDN`) threat model — what a compromised render-node client can and cannot do; DRM master vs. non-master privileges; the lack of IOMMU enforcement per-process in mainline and ongoing `drm_context` / `drm_client` isolation work; multi-tenant GPU scheduling fairness and hangcheck blast radius; GPU-side speculative execution side-channels (Rowhammer analogues on GPU DRAM); the Chrome GPU sandbox as a worked example of defence-in-depth (Ch33); Vulkan `VK_EXT_device_address_binding_report` and GPU-assisted validation as developer-facing security tooling
- **Integrations**: RenderDoc captures at the Vulkan API boundary (Ch24) and can replay through any Mesa Vulkan driver (Ch18); sync validation in Vulkan Validation Layers exercises the explicit sync paths that run down to DRM sync objects (Ch3); Mesa debug dumps expose NIR at each compilation stage (Ch14) and ACO disassembly (Ch15); performance counters are read from DRM render nodes (Ch1) via vendor-specific ioctls; the GPU sandbox model in Chrome (Ch33) is the most complete real-world application of the render-node isolation model described in the security section above

### Chapter 31: Conformance & Regression Testing
- dEQP structure: test modules (Vulkan CTS, GLES CTS), running locally, reading results
- piglit: test categories, running the suite, bisecting regressions, CI integration
- How conformance results gate Mesa releases and Khronos certification
- Fuzzing and sanitiser builds for driver development
- `deqp-runner`: the parallel test harness wrapping dEQP and piglit for practical CI use; how Mesa CI uses it to run hundreds of thousands of CTS tests in parallel across GPU farm nodes; `deqp-runner suite` vs. `deqp-runner run`; flake detection (automatic retry on first fail); expectation files (`.toml`) for tracking known failures; how to run a subset of the VK-GL-CTS locally using deqp-runner's filter syntax
- IGT GPU Tools (`igt-gpu-tools`): the kernel-level DRM test suite; KMS tests (`kms_plane`, `kms_atomic`, `kms_hdmi_inject`); GEM/memory tests (`gem_mmap_gtt`, `gem_exec_fence`); GPU scheduler tests; how IGT exercises the DRM uAPI directly via ioctls, distinct from Mesa-level CTS; running IGT in a VM vs. on real hardware; the `igt_runner` CI harness and result comparison
- **Integrations**: dEQP-VK tests every Mesa Vulkan driver (Ch18) and NVK (Ch10) against the Vulkan spec; GLES CTS tests radeonsi, iris, and Zink (Ch19); piglit tests run on software renderers (Ch17) in headless CI where no GPU is available; conformance failures frequently trace back to NIR optimisation pass bugs (Ch14) or ACO register allocation issues (Ch15); the WebGL CTS tests ANGLE (Ch34) and the WebGPU CTS tests Dawn (Ch35), both of which run against the same Mesa Vulkan and GL drivers, making browser conformance suites a second CI dimension over the same driver stack

### Chapter 32: Contributing to the Linux Graphics Stack
- Kernel DRM: mailing list workflow, patch formatting, `drm-misc`, review process
- Mesa: GitLab MR workflow, CI pipeline, coding style, shader-db performance tracking
- Wayland protocols: the freedesktop.org proposal process, protocol stability guarantees
- Finding good first issues across the stack; community norms and communication channels
- **Integrations**: a single feature (e.g. HDR support) touches every layer — DRM kernel properties (Ch3), Mesa colour management, Vulkan swapchain extensions (Ch24), and compositor protocol (Ch20); this chapter maps how a cross-cutting change propagates through the contributing workflow of each sub-project, and who to coordinate with at each boundary

---

## Part X — The Browser Rendering Stack

### Chapter 33: Chromium's Multi-Process GPU Architecture
- Why a dedicated GPU process: sandboxing, crash isolation, privilege separation on Linux
- Mojo IPC: pipes, interfaces, and the command buffer protocol
- The renderer process path: from JavaScript WebGL/WebGPU calls to the GPU process
- GPU feature detection and driver blocklisting on Linux; the `gpu_info` negotiation at startup
- How Chrome selects a backend on Linux: Vulkan vs. GL vs. software fallback
- OOP-D (Out-of-Process Display Compositor) and its implications for Wayland/X11
- The Ozone abstraction layer: how Chrome abstracts the display platform (Wayland, X11, GBM)
- **Integrations**: the GPU process opens DRM render nodes (Ch1) and creates Vulkan or EGL contexts via Mesa drivers (Ch18, Ch19); on Wayland, Chrome presents frames through the Ozone/Wayland backend using linux-dmabuf (Ch20); the GPU process sandbox mirrors the privilege separation between DRM render nodes and primary nodes (Ch1); the command buffer protocol is conceptually similar to Wayland's wire protocol (Ch20)

### Chapter 34: ANGLE — WebGL on Linux
- What ANGLE is: a portable OpenGL ES implementation with pluggable backends
- The Vulkan backend: ANGLE's internal Vulkan renderer architecture; RendererVk, ContextVk, vk::Framebuffer
- How WebGL draw calls translate through ANGLE to Vulkan API calls
- Shader translation: GLSL ES → SPIR-V via glslang; the ANGLE shader translator
- Surface and context management on Linux: EGL integration, Wayland EGL surface path
- ANGLE's sync object model and how it maps to Vulkan semaphores and fences
- Performance characteristics vs. native OpenGL; known overhead sources
- **Integrations**: ANGLE's Vulkan backend is a client of Mesa Vulkan drivers (Ch18) — RADV, ANV, and NVK; shader translation via glslang produces SPIR-V that enters the same Mesa NIR pipeline (Ch14) as native Vulkan apps; ANGLE uses EGL for surface creation (Ch24) and can share DMA-BUF buffers (Ch4) with the Chromium compositor (Ch36); ANGLE's conformance is tested against the WebGL CTS, complementing Mesa's dEQP-VK suite (Ch31)

### Chapter 35: Dawn and WebGPU
- Dawn architecture: the frontend API, backend abstraction layer, and wire protocol
- The Vulkan backend in depth: mapping WebGPU objects to Vulkan — pipelines, bind groups, render passes
- Tint: WGSL parsing, the Tint IR, and code generation targets (SPIR-V, MSL, HLSL)
- The DawnWire IPC protocol: command serialisation, object handle lifetime, error propagation across the process boundary
- Memory management: VMA integration for device-local allocations; shared memory regions for `mapAsync`
- WebGPU canvas presentation: swapchain integration with the Chromium compositor (Viz)
- Current WebGPU feature coverage on Linux; timeline semaphore and explicit sync support
- **Integrations**: Dawn's Vulkan backend is a Vulkan client of Mesa drivers (Ch18); SPIR-V emitted by Tint enters the Mesa NIR/ACO pipeline (Ch14, Ch15) identically to native Vulkan shader modules; canvas swapchain images are handed to Viz (Ch36) via SharedImage/DMA-BUF (Ch4); the DawnWire serialisation model is architecturally comparable to the Wayland wire protocol (Ch20); WebGPU compute shaders share the same GPU compute queue as Vulkan compute (Ch25)

### Chapter 36: The Chromium Compositor — CC and Viz
- Two-layer model: CC (content compositor, per-renderer) and Viz (display compositor, in the GPU process)
- Tile rasterisation: the raster worker pool, OOP-R (out-of-process raster), GPU raster via ANGLE/Skia
- The display compositor: surface aggregation, damage propagation, overlay candidate selection
- SharedImage: Chromium's cross-process GPU texture abstraction; mailbox handles, backing stores
- How Chrome presents to Wayland: the Ozone/Wayland backend, EGL images, linux-dmabuf buffer submission
- BeginFrame and vsync: how Viz synchronises to the display refresh; `wp_presentation` feedback
- Overlay promotion: delegating video and canvas quads directly to KMS planes
- **Integrations**: Viz drives the Wayland compositor (Ch22) or directly submits to KMS in some configurations; linux-dmabuf (Ch20) is the zero-copy path between SharedImage and the Wayland compositor; overlay promotion mirrors KMS plane assignment (Ch2) and benefits from DRM format modifiers (Ch4); BeginFrame cadence connects to `wp_presentation` (Ch20) and VRR atomic properties (Ch3); SharedImage backing stores are GBM/DMA-BUF objects (Ch4) shared with ANGLE (Ch34) and Dawn (Ch35)

### Chapter 37: Skia and 2D Rendering
- Skia's role in Chrome: text, vector paths, images, CSS filter effects, canvas 2D
- SkiaGanesh: the mature GL/Vulkan-accelerated Skia backend; GrContext, GrRenderTarget, GrTexture
- SkiaGraphite: the redesigned backend built on Dawn/WebGPU; Recorder, CommandBuffer, task graph
- The transition from Ganesh to Graphite: motivation, architecture differences, current status in Chrome
- Text rendering on Linux: FreeType rasterisation, HarfBuzz shaping, glyph atlases as GPU textures, subpixel rendering and fontconfig
- CSS filter and compositing effects: blur, drop-shadow, colour matrix — rasterised via Skia on the GPU
- **Integrations**: SkiaGanesh uses ANGLE (Ch34) as its GL backend or calls Vulkan directly via Mesa drivers (Ch18); SkiaGraphite uses Dawn (Ch35) as its GPU backend, feeding the same Tint/SPIR-V shader pipeline; glyph atlas textures are managed through the SharedImage system (Ch36); Skia rasterises content into tiles that CC hands to Viz (Ch36) for compositing; font configuration on Linux calls into fontconfig, which is independent of but feeds the same rendering pipeline as Pango/Cairo in GTK applications (Ch22)

---

## Part XI — Engines & Creative Tools

**Rationale**: Bevy, Godot, and Blender are fully open-source, Linux-first Vulkan clients whose internal rendering paths are traceable through the same stack layers covered in Parts I–V. Each chapter is scoped as *"how this engine sits on the Linux stack"*, not a general engine architecture guide. Unreal Engine and Unity are acknowledged as significant Vulkan consumers but are closed-source; their Linux-specific internals are not auditable to the standard this book requires — covered as named examples within Ch 18 (Vulkan Drivers) or Ch 41, not as standalone chapters.

### Chapter 40: Bevy and wgpu — A Rust-Native Vulkan Client

- Bevy's render architecture: the render world, extract/prepare/queue/render stages, render graph
- `wgpu` as the GPU abstraction: the Vulkan backend on Linux; how `wgpu` maps to `ash` Vulkan bindings and Mesa drivers (Ch18)
- Shader pipeline: WGSL → `naga` IR → SPIR-V; how naga-emitted SPIR-V enters Mesa NIR (Ch14) — the native-Rust peer to Dawn/Tint (Ch35)
- Buffer and texture management: `wgpu`'s memory model over Vulkan VMA; GBM/DMA-BUF surface creation for Wayland windows (Ch4, Ch24)
- Wayland window integration: `winit` → Wayland/EGL surface setup; explicit sync handshake (Ch3, Ch20)
- Compute in Bevy: compute passes via `wgpu`, WGSL compute shaders through the same naga/SPIR-V/NIR path (Ch25)
- Unreal Engine 5 and Unity: sidebar covering their Vulkan RHI on Linux as closed-source counterpoints — what is observable from public documentation vs. what is not auditable
- **Integrations**: wgpu's Vulkan path is a client of Mesa Vulkan drivers (Ch18); naga/SPIR-V enters the same NIR front end as Dawn/Tint (Ch14, Ch35); the Rust implementation echoes Nova and the DRM Rust theme; wgpu surface creation uses EGL/GBM (Ch24, Ch4); compute shaders share the Vulkan compute queue path (Ch25)

### Chapter 41: Godot 4 — RenderingDevice and the Explicit Vulkan Path

- Godot 4's rendering architecture: the `RenderingDevice` abstraction layer; Forward+ and Mobile rendering paths; Clustered deferred
- Vulkan backend in depth: how Godot maps its scene graph to Vulkan command buffers; pipeline state objects, descriptor sets, push constants
- GDShader → GLSL → SPIR-V compilation pipeline; how Godot-emitted SPIR-V traverses Mesa NIR (Ch14)
- Wayland and X11 window creation: `DisplayServerWayland` and `DisplayServerX11`; EGL context setup; linux-dmabuf swapchain (Ch20, Ch24)
- Memory management: Godot's GPU allocator over Vulkan; buffer staging, texture streaming
- Compute and GPU particles: compute shader usage in Godot 4; connection to Vulkan compute queues (Ch25)
- Godot's CI and conformance: how the project tests its Vulkan backend; interaction with Mesa CTS results (Ch31)
- **Integrations**: RenderingDevice's Vulkan backend is a client of Mesa Vulkan drivers (Ch18); SPIR-V compilation feeds NIR (Ch14); EGL/Wayland surface path uses linux-dmabuf (Ch20); GPU particles use the same Vulkan compute queue covered in Ch25; Godot's explicit sync usage connects to `wp_linux_drm_syncobj` (Ch3)

### Chapter 42: Blender — Cycles, EEVEE Next, and GPU Compute on Linux

- EEVEE Next (Blender 4.x): complete Vulkan rewrite; deferred shading, shadow maps, volumetrics; how EEVEE's render passes map to Vulkan render passes and subpasses (Ch16, Ch18)
- Cycles GPU backend: the multi-backend architecture (HIP for AMD, CUDA/OptiX for NVIDIA, oneAPI for Intel, Vulkan path); how each backend maps to kernel drivers (Ch5) and compute APIs (Ch25)
- GLSL/SPIR-V shader compilation in Blender: Cycles kernel compilation; EEVEE shader generation pipeline; SPIR-V entry into Mesa NIR (Ch14)
- OpenColorIO integration: Blender's color management pipeline; connection to KMS color properties and `wp_color_management` (Ch3, Ch20)
- GPU memory management: Blender's `BKE_blendfile` texture streaming; DMA-BUF sharing for viewport compositing (Ch4)
- Viewport rendering: EGL/Wayland surface creation for the 3D viewport; OpenGL legacy path vs. Vulkan EEVEE path (Ch19, Ch24)
- Cycles as a GPU compute workload: kernel occupancy, memory bandwidth patterns on AMD/NVIDIA/Intel; how ROCm and HIP relate to the amdgpu kernel driver (Ch5, Ch25)
- **Integrations**: EEVEE Next's Vulkan renderer is a client of Mesa Vulkan drivers (Ch18) and uses Mesa's Vulkan common infrastructure (Ch16); Cycles HIP/ROCm runs on amdgpu compute queues (Ch5, Ch25); SPIR-V from Blender's GLSL compiler enters the NIR front end (Ch14); OpenColorIO color transforms connect to KMS color pipeline (Ch3); viewport EGL context creation follows the path in Ch24

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
