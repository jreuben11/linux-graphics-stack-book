# The Linux Graphics Stack: From Kernel to Compositor, Browser, and Terminal
## Book Plan

### Audience
This book serves four audiences:
- **Systems and driver developers** — depth on kernel internals, DRM/Mesa architecture, and driver implementation
- **Graphics application developers** — understanding the stack beneath Vulkan, EGL, VA-API, and OpenXR
- **Browser and web platform engineers** — how Chromium/Chrome maps WebGPU, WebGL, and compositing onto the Linux graphics stack
- **Terminal and TUI developers** — how terminal emulators render pixel graphics (Sixel, Kitty Graphics Protocol, Ghostty) on top of the compositor stack

Chapters signal which perspective is emphasised where they diverge.

---

## Table of Contents

- **Part I — The Kernel Layer**
  - [Chapter 1: DRM Architecture & the Driver Model](#chapter-1-drm-architecture--the-driver-model)
  - [Chapter 2: KMS: The Display Pipeline](#chapter-2-kms-the-display-pipeline)
  - [Chapter 3: Advanced Display Features](#chapter-3-advanced-display-features)
  - [Chapter 4: GPU Memory Management](#chapter-4-gpu-memory-management)
  - [Chapter 51: GPU Power Management and Thermal](#chapter-51-gpu-power-management-and-thermal)
- **Part II — GPU Drivers**
  - [Chapter 5: x86 GPU Drivers](#chapter-5-x86-gpu-drivers)
  - [Chapter 6: ARM & Embedded GPU Drivers](#chapter-6-arm--embedded-gpu-drivers)
  - [Chapter 49: Multi-GPU and PRIME Render Offload](#chapter-49-multi-gpu-and-prime-render-offload)
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
  - [Chapter 46: The Evolving Wayland Protocol Ecosystem](#chapter-46-the-evolving-wayland-protocol-ecosystem)
  - [Chapter 53: Display Calibration and colord](#chapter-53-display-calibration-and-colord)
  - [Chapter 54: The Linux Input Stack](#chapter-54-the-linux-input-stack)
- **Part VII — Application APIs & Middleware**
  - [Chapter 24: Vulkan and EGL for Application Developers](#chapter-24-vulkan-and-egl-for-application-developers)
  - [Chapter 25: GPU Compute](#chapter-25-gpu-compute)
  - [Chapter 26: Hardware Video](#chapter-26-hardware-video)
  - [Chapter 27: VR & AR](#chapter-27-vr--ar)
  - [Chapter 38: PipeWire and the Video Session Layer](#chapter-38-pipewire-and-the-video-session-layer)
  - [Chapter 39: Qt and GTK GPU Rendering](#chapter-39-qt-and-gtk-gpu-rendering)
  - [Chapter 47: Font and Text Rendering Pipeline](#chapter-47-font-and-text-rendering-pipeline)
  - [Chapter 48: ROCm and Machine Learning on Linux GPUs](#chapter-48-rocm-and-machine-learning-on-linux-gpus)
  - [Chapter 50: Vulkan Video Extensions](#chapter-50-vulkan-video-extensions)
- **Part VIII — Gaming Layer**
  - [Chapter 28: Windows Compatibility](#chapter-28-windows-compatibility)
  - [Chapter 29: Upscaling, Effects & Overlays](#chapter-29-upscaling-effects--overlays)
  - [Chapter 56: Ray Tracing on Linux](#chapter-56-ray-tracing-on-linux)
- **Part IX — Tooling & Contributing**
  - [Chapter 30: Debugging & Profiling](#chapter-30-debugging--profiling)
  - [Chapter 31: Conformance & Regression Testing](#chapter-31-conformance--regression-testing)
  - [Chapter 32: Contributing to the Linux Graphics Stack](#chapter-32-contributing-to-the-linux-graphics-stack)
  - [Chapter 55: GPU Containers and Cloud Compute](#chapter-55-gpu-containers-and-cloud-compute)
- **Part X — The Browser Rendering Stack**
  - [Chapter 33: Chromium's Multi-Process GPU Architecture](#chapter-33-chromiums-multi-process-gpu-architecture)
  - [Chapter 34: ANGLE — WebGL on Linux](#chapter-34-angle--webgl-on-linux)
  - [Chapter 35: Dawn and WebGPU](#chapter-35-dawn-and-webgpu)
  - [Chapter 36: The Chromium Compositor — CC and Viz](#chapter-36-the-chromium-compositor--cc-and-viz)
  - [Chapter 37: Skia and 2D Rendering](#chapter-37-skia-and-2d-rendering)
  - [Chapter 52: Firefox and WebRender](#chapter-52-firefox-and-webrender)
- **Part XI — Engines & Creative Tools**
  - [Chapter 40: Bevy and wgpu](#chapter-40-bevy-and-wgpu)
  - [Chapter 41: Godot 4 RenderingDevice](#chapter-41-godot-4-renderingdevice)
  - [Chapter 42: Blender GPU — Cycles and EEVEE](#chapter-42-blender-gpu--cycles-and-eevee)
- **Part XII — Terminal Graphics**
  - [Chapter 43: Terminal Pixel Protocols — Sixel, Kitty, and iTerm2](#chapter-43-terminal-pixel-protocols--sixel-kitty-and-iterm2)
  - [Chapter 44: Terminal GPU Rendering Architectures](#chapter-44-terminal-gpu-rendering-architectures)
  - [Chapter 45: Terminal Integration with the Compositor Stack](#chapter-45-terminal-integration-with-the-compositor-stack)

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

### Chapter 51: GPU Power Management and Thermal
- DRM runtime PM framework: `drm_dev_enter`/`drm_dev_exit`; the `autosuspend` delay; linking `drm_device.dev` to the Linux PM core; how `drm_clflush` and KMS DPMS interact with runtime suspend
- amdgpu power management: `amdgpu_pm_sysfs`; power profiles (auto, low, high, manual, compute); `pp_power_profile_mode`; BACO (Bus Active, Chip Off); GFXOFF; SMU (System Management Unit) firmware; APU vs. dGPU power state differences; `amdgpu.ppfeaturemask`
- Intel i915/Xe power management: RC6 render C-states; `intel_rc6_enable`; GuC power management; DSSM (Dynamic Shutoff Slow Memory); `i915.enable_dc` display C-states; Xe2 power gating
- NVIDIA power management: `nvidia-smi -pm 1` persistence mode; `--power-limit` capping; TGP (Total Graphics Power) on laptops; power-on-demand vs. always-on; `nvidia-open` power state differences vs. proprietary driver
- Nouveau power management: reclocking difficulty (Ch11); safe clock tables via GSP-RM; `nouveau.pstate` module parameter; power regression history and current status
- Thermal management: `drivers/thermal/` framework; GPU thermal zones; `trip_point` throttle callback; AMD SMU thermal policy vs. Linux thermal; fan control via `thinkfan`, `nbfc`, and the AMD SMU fan curve interface
- `power-profiles-daemon` and `powerprofilesctl`: how it maps `performance`/`balanced`/`power-saver` profiles to `amdgpu` power profiles and Intel EPP (Energy Performance Preference)
- Tools and monitoring: `powertop`, `turbostat`, `rocm-smi`, `intel_gpu_top`, `nvtop`, `/sys/class/drm/*/device/power/`
- **Integrations**: amdgpu power profiles interact with GameMode (Ch29) and ROCm workloads (Ch48); thermal throttling affects conformance test reproducibility (Ch31); Nouveau reclocking (Ch11) is the GPU-specific view of the DRM PM model described here; laptop hybrid graphics (Ch49) adds a layer of power-switching above per-GPU PM; containers and cloud (Ch55) need persistent mode and power capping for predictable GPU performance

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

### Chapter 49: Multi-GPU and PRIME Render Offload
- Hybrid graphics hardware: Intel iGPU + NVIDIA/AMD dGPU laptop topology; muxed vs. muxless designs; how the two GPUs share the display pipeline
- PRIME DRM buffer sharing: `DRM_PRIME_FD_TO_HANDLE` / `DRM_PRIME_HANDLE_TO_FD`; the cross-device DMA-BUF export/import path; `drm_gem_prime_export` and `drm_gem_prime_import` driver callbacks
- `DRI_PRIME` environment variable: how Mesa's driver selection logic (Ch12) picks the rendering device; `VK_LAYER_MESA_device_select` Vulkan layer; `prime-run` wrapper script
- Reverse PRIME: rendering on dGPU, scanning out via iGPU KMS; the `xrandr --setprovideroffloadsink` blit path; the pixmap blit through the CPU vs. direct DMA-BUF import
- Explicit GPU selection in Vulkan: `VkPhysicalDeviceGroupProperties`; `VK_KHR_device_group`; multi-device submission for split-frame rendering
- AMD SmartShift: CPU + GPU TDP sharing on AMD Advantage platforms; `amdgpu` sysfs interface for SmartShift state
- `supergfxctl` (ASUS), `system76-power` (System76): vendor tools for runtime GPU switching; the `supergfxd` D-Bus daemon
- Multi-GPU rendering (not offload): SLI/CrossFire history and removal; why multi-GPU in Vulkan is application-driven, not driver-automatic; `VkDeviceGroupSubmitInfo` and `deviceMask`
- Peer-to-peer DMA: NVLink and AMD Infinity Fabric; the `p2pdma` kernel framework; GPU NUMA topology; implications for ML multi-GPU (Ch48) and ROCm collective operations
- **Integrations**: PRIME uses DMA-BUF (Ch4) as the cross-GPU transport; Wayland compositors (Ch21, Ch22) handle PRIME outputs via the DRM backend; gamescope (Ch22) supports PRIME for Steam Deck-like topologies; GPU power management (Ch51) must coordinate power states across both GPUs during offload; GPU containers (Ch55) need PRIME-aware device selection for the correct GPU to handle compute vs. display

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

### Chapter 46: The Evolving Wayland Protocol Ecosystem

This chapter covers the wave of staging protocols that reached compositor implementation in 2024–2026, directly addressing the limitation gaps catalogued in Chapter 20 §13. It is a snapshot chapter: protocols at the frontier of shipping but not yet stable. Readers should treat version references as tied to the mid-2026 snapshot (Mutter 50.x, KWin 6.7.x, wlroots 0.18.x, xdg-desktop-portal 1.22.x); the graduation trajectory is clearly signalled where known.

- **Explicit synchronisation in depth: `wp_linux_drm_syncobj_v1`**
  - Recap of the implicit-sync failure mode for NVIDIA (Ch3, Ch20 §8) — the motivation
  - DRM timeline sync objects: kernel `drm_syncobj` with timeline semantics; `DRM_IOCTL_SYNCOBJ_CREATE`, `DRM_IOCTL_SYNCOBJ_TIMELINE_SIGNAL`, `DRM_IOCTL_SYNCOBJ_TIMELINE_WAIT`; point-based vs. binary sync objects
  - Protocol mechanics in full: `wp_linux_drm_syncobj_manager_v1.get_surface` → `wp_linux_drm_syncobj_surface_v1`; `set_acquire_point(timeline, point)` — compositor waits for GPU finish before read; `set_release_point(timeline, point)` — compositor signals when done, client may reuse buffer; the surface-level granularity (per-commit, not per-device)
  - Compositor implementations: Mutter (`meta-wayland-linux-drm-syncobj.c`, landed GNOME 46); KWin (`linux_drm_syncobj_v1.cpp`, Plasma 6.0); wlroots (`wlr_linux_drm_syncobj_v1.c`, wlroots 0.17)
  - Mesa integration: how Mesa's EGL/Vulkan WSI exports acquire/release timeline points alongside the DMA-BUF in `src/egl/drivers/dri2/platform_wayland.c`; the `EGL_ANDROID_native_fence_sync` bridge on the Mesa side
  - Application impact: what changed for NVIDIA users after Plasma 6 / GNOME 46; residual issues; comparison with the old `zwp_linux_explicit_synchronization_unstable_v1` predecessor

- **HDR colour management: `wp_color_management_v1` v3**
  - Protocol history: from `wp_color_management` (2019 draft) through `xx_color_management` (breaking redesign) to the v3 staging protocol in wayland-protocols 1.45 (October 2024); why three design iterations were needed
  - Core concepts: image descriptions vs. output colour profiles; the `wp_image_description_v1` object lifecycle (create → ready/failed events → use → destroy); ICC profile upload via `wp_image_description_creator_icc_v1`; named colour primaries and transfer functions for common colour spaces (sRGB, Display P3, BT.2020, PQ, HLG)
  - Surface colour management: `wp_color_management_surface_v1.set_image_description(desc, render_intent)`; render intent enum (perceptual, relative, saturation, absolute, relative_bpc); how the compositor applies a colour transform from surface image description to display image description
  - Output feedback: `wp_color_management_output_v1` and `wp_color_management_surface_feedback_v1`; the preferred image description event; how applications use output feedback to select the optimal colour space for a given display
  - Compositor implementations: Mutter — `meta-wayland-color-management.c` (SUSE/Red Hat, GNOME 48); KWin — `colormanagement_v1.cpp` (Xaver Hugl, Plasma 6.1); the KMS colour pipeline backend (DEGAMMA_LUT, CTM, GAMMA_LUT properties from Ch3) that both compositors program in response to surface colour requests
  - HDR in practice: what `wp_color_management_v1` enables that was not possible before; the companion `wp_color_representation_v1` protocol for video colour matrix signalling; current limitations (v3 still staging; toolkit support partial as of mid-2026)

- **Cross-compositor screen capture: `ext-image-copy-capture-v1`**
  - The capture landscape before this protocol: `zwlr_screencopy_manager_v1` (wlroots-only, `wlr-protocols` repo, never in `wayland-protocols`); Mutter's `org.gnome.Shell.Screenshot` D-Bus API; KWin's `org.kde.KWin.ScreenShot2`; why fragmentation made multi-desktop screen recording libraries (OBS, PipeWire) maintain separate code paths per compositor
  - Protocol architecture: `ext_image_copy_capture_manager_v1` → `ext_image_copy_capture_session_v1` per capture source (output, toplevel); `ext_image_copy_capture_frame_v1` for each captured frame — attach a DMA-BUF or shm buffer, call `capture`, wait for `ready` or `failed` event; damage region reporting on the frame object for efficient partial capture
  - Cursor sessions: `ext_image_copy_capture_cursor_session_v1` — separate session for cursor capture; cursor position, hotspot, and bitmap delivered independently; allows overlay rendering of cursor above captured content without compositor embedding the cursor in the frame
  - Format negotiation: `ext_image_copy_capture_session_v1.get_shm_formats` and `get_dmabuf_formats` events before capture begins; client selects a supported format/modifier pair; same DRM format modifier negotiation as linux-dmabuf (Ch4, Ch20)
  - Compositor status: wlroots full implementation (`wlr_ext_image_copy_capture_v1.c`); Mutter and KWin implementations in progress as of mid-2026; the protocol is authored by Andri Yngvason and Simon Ser (@emersion, wlroots/sway ecosystem)
  - PipeWire integration path: how `xdg-desktop-portal`'s screencast backend will migrate from compositor-specific APIs to `ext-image-copy-capture-v1` once Mutter and KWin land implementations; the DMA-BUF frame → PipeWire node → consumer path (Ch38)

- **Frame scheduling: `wp_fifo_v1`**
  - Problem: clients that render faster than the display refresh rate waste GPU time and add latency; `wp_presentation` (Ch20 §7) provides feedback but no constraint; applications must self-throttle by inspecting `presented` timestamps
  - `wp_fifo_v1` design: `wp_fifo_manager_v1.get_fifo(wl_surface)` → `wp_fifo_v1`; `set_barrier()` marks a commit as a FIFO barrier — the compositor will not present the *next* commit until the barrier commit has been presented; `wait_barrier()` on a subsequent commit blocks the server-side commit processing until the barrier is consumed; the effect is backpressure — the client's render loop naturally paces to display refresh without polling `wp_presentation` timestamps
  - Authored by Valve (Derek Foreman, Collabora); designed for gaming workloads on Steam Deck where sub-frame-rate rendering and frame-pacing stability are more important than maximum throughput
  - Compositor status: Mutter and KWin both have references in the codebase (partial integration, mid-2026); wlroots implementation not yet confirmed
  - Interaction with `wp_presentation` and VRR: how FIFO constraints interact with variable-refresh-rate displays; the FIFO model on a VRR display (no fixed period to pace against); expected protocol clarifications

- **Portal evolution: GlobalShortcuts, RemoteDesktop, and InputCapture**
  - `org.freedesktop.portal.GlobalShortcuts` v2 (xdg-desktop-portal 1.20+, stable): `BindShortcuts(session, shortcuts, parent_window, activation_token)` — registers named shortcuts; `Activated`/`Deactivated` signals deliver events; `ConfigureShortcuts()` (added 1.21) opens compositor-native shortcut configuration UI; GNOME and KDE portal backends both implement it; this closes the `XGrabKey` gap for portal-aware applications
  - RemoteDesktop portal improvements (xdg-desktop-portal 1.21–1.22): clipboard support in remote sessions (1.21.1); session persistence across reconnects; the underlying Wayland path — PipeWire screencast + `virtual-keyboard-unstable-v1` + `pointer-constraints-unstable-v1` — and why this is necessarily higher-latency than X11 network forwarding
  - InputCapture portal (1.21+): synthetic pointer/keyboard input injection for accessibility tools; `CreateSession`, `GetZones`, `SetPointerBarriers`; clipboard access from captured input; how screen readers and switch-access devices use this without requiring Wayland compositor privileges directly
  - `wp_security_context_v1` (staging): compositor-side security context attachment for Flatpak connections; how KWin (Plasma 6) uses it to restrict sandbox protocol access; the `xdg-desktop-portal` coupling — the portal daemon calls `wp_security_context_v1` to tag sandboxed client connections with the Flatpak app ID, enabling per-application protocol policy without compositor-specific configuration

- **Protocols still unresolved: tray, network transparency, workspace management**
  - Status notification / system tray: no `ext-tray-v1` exists in wayland-protocols as of mid-2026; the D-Bus `org.freedesktop.StatusNotifierItem` spec (KDE SNI) is the de-facto cross-desktop mechanism; why designing a Wayland-native tray protocol is architecturally difficult (tray aggregators need compositor cooperation for positioning and rendering, not just event delivery)
  - `ext-workspace-v1` (staging, wlroots only): virtual desktop management; wlroots full implementation; Mutter and KWin absent — both have their own workspace management APIs and have not converged on the protocol
  - `ext-session-lock-v1` (staging, wlroots + River): custom lock screen surfaces; wlroots full implementation; Mutter uses `org.gnome.ScreenSaver` D-Bus, KWin uses its own lock screen protocol — desktop compositors resistant to adopting a protocol that controls the security-critical lock screen
  - Network transparency: Waypipe (`https://gitlab.freedesktop.org/mstoeckl/waypipe`) intercepts Wayland socket traffic and recompresses GPU buffers for SSH forwarding; version 0.11.0 (April 2026); minimal maintenance (one commit in 2.5 years); not production-ready for broad deployment; the architectural reason this is harder than X11 forwarding (DMA-BUF fds cannot cross a network, so all zero-copy paths must be unwound)
  - Capability discovery: `wl_registry` probing as the only mechanism; why a unified `wp_capabilities_v1` would help but has not been prioritised

- **Integrations**: `wp_linux_drm_syncobj_v1` is the Wayland surface of the DRM sync object infrastructure (Ch3, Ch4); its Mesa implementation is in `src/egl/drivers/dri2/platform_wayland.c` (Ch12); `wp_color_management_v1` programs the KMS colour pipeline (Ch3) through the compositor's KMS backend (Ch21, Ch22); `ext-image-copy-capture-v1` feeds DMA-BUF frames into PipeWire (Ch38) for screen recording and portal screen cast; `wp_fifo_v1` is a counterpart to `wp_presentation` (Ch20 §7) and interacts with VRR KMS properties (Ch3); the portal evolution closes the gaps identified in Ch20 §13 and extends Ch23's coverage of `xdg-desktop-portal`; `wp_security_context_v1` is the Wayland-layer complement to Flatpak's seccomp/namespace sandbox (Ch23)

### Chapter 53: Display Calibration and colord
- The calibration problem: why uncalibrated displays produce incorrect colours; the ICC profile as a device characterisation standard; matrix vs. LUT profile types
- `colord` daemon: architecture; `ColorDevice` and `ColorProfile` D-Bus objects (`org.freedesktop.ColorManager`); automatic profile assignment via udev device rules; ICC database at `/var/lib/colord/`; the `cd-create-profile` tool
- VCGT (Video Card Gamma Table): the `vcgt` ICC tag; how `colord-session` reads the VCGT and loads it into the KMS `GAMMA_LUT` property (Ch3) at login; the `colord-session` D-Bus service and its interaction with logind
- ArgyllCMS and DisplayCAL: colorimeter and spectrophotometer integration (`i1Display Pro`, `ColorMunki`, `Spyder X`); measurement workflow: `spotread`, `dispcal`, `targen`, `dispread`; generating a three-channel matrix+VCGT profile; the `ti3` measurement file format
- The calibration → compositor pipeline: `colord` → `wp_color_management_v1` image description (Ch46) → KMS colour pipeline (Ch3); how the D-Bus profile assignment bridges to the Wayland protocol's ICC creator interface
- GNOME Color and KDE colour management UI: `gnome-color-manager`; KDE's System Settings colour correction panel; how both wrap `colord`'s D-Bus API; per-output profile assignment with multi-monitor setups
- Night light / blue light reduction: `gammastep`, `wlsunset`, `redshift`; how they modify the KMS `GAMMA_LUT` at a layer independent of ICC VCGT; the interaction between a calibration VCGT and a night-light ramp
- HDR calibration: MaxCLL/MaxFALL metadata in ICC display profiles; the `colord` roadmap for HDR profiling; current gaps between display profiling and the `wp_color_management_v1` HDR pipeline
- **Integrations**: colord feeds the KMS GAMMA_LUT (Ch3) and is the system-level bridge to `wp_color_management_v1` (Ch46); compositors (Ch22) consume colord profiles via D-Bus; VA-API video surfaces (Ch26) carry their own colour space metadata independently of display calibration; the ICC profile format used by colord matches the `wp_image_description_creator_icc_v1` interface (Ch46)

### Chapter 54: The Linux Input Stack
- Kernel input subsystem: `/dev/input/eventN`; `struct input_event` fields (type, code, value); EV_KEY, EV_ABS, EV_REL, EV_SYN event types; `input_register_device`; the `evdev` kernel driver; udev rules for device permissions
- libinput: the device abstraction layer; `libinput_device`, `libinput_event`; device type detection algorithm; the quirks database (`/usr/share/libinput/*.quirks`); calibration matrices for touchscreens; `libinput debug-events` for diagnosis
- libwacom: graphics tablet identification; the `wacom.stylus` database; pressure curve specification; multi-ring/strip capability; how libinput uses libwacom for Wacom and non-Wacom tablet devices; `libwacom-list-devices`
- Gaming controllers: `hid-xbox`, `hid-sony`, `hid-nintendo` kernel drivers; SDL2's `GameController` API above `evdev`; udev rules for unprivileged access; `evdev` gamepad protocol vs. legacy `jsdev`; Steam Input virtual controller layer and `uinput`
- Pointer constraints on Wayland: `zwp_pointer_constraints_v1`; `zwp_locked_pointer_v1` for FPS mouse look; `zwp_confined_pointer_v1` for drag operations; `zwp_relative_pointer_v1` for raw mouse motion without acceleration; how these map to libinput's relative motion events
- Touch and gesture protocols: `wl_touch` for multi-touch; `zwp_pointer_gestures_v1` for touchpad pinch/swipe events; `zwp_input_timestamps_v1` for sub-millisecond event timestamps; the full evdev→libinput→`wlr_seat`→`wl_pointer` delivery chain
- Accessibility input: AT-SPI2 accessibility bus; switch access via `evdev` key events; eye tracking devices (Tobii, VIVE Pro Eye) as `evdev` pointing devices; the InputCapture portal (Ch46) for compositor-level input injection
- Input latency: the evdev kernel ring buffer; libinput's batching model; Wayland compositor frame-aligned input delivery; how `wp_input_timestamps_v1` enables sub-frame input latency measurement; game input latency via MangoHud (Ch29)
- **Integrations**: libinput feeds Wayland compositors (Ch21, Ch22) via `wlr_input_device` and Mutter's `MetaSeatNative`; pointer constraint protocols build on the input stack described here (Ch46); gaming controller events reach games via SDL2 which straddles evdev and Wayland; Monado (Ch27) consumes libinput tracking data; the InputCapture portal (Ch46) injects synthetic events back into this stack

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

### Chapter 47: Font and Text Rendering Pipeline
- FreeType 2: glyph rasterisation pipeline; hinting modes (autohinter, bytecode interpreter, no-hinting); subpixel rendering (LCD filtering, `FT_RENDER_MODE_LCD`); `FT_Load_Glyph` flags; the FreeType LCD filter (`FT_Library_SetLcdFilter`) and patent history; gamma correction for perceptual uniformity
- HarfBuzz: the OpenType shaping engine; `hb_buffer_t` and `hb_font_t`; GSUB (Glyph Substitution) and GPOS (Glyph Positioning) table processing; Unicode Bidirectional Algorithm (UBA) integration; complex script support (Arabic ligatures, Indic vowel signs, Hangul jamo composition); `hb_shape()` and the cluster model for cursor positioning
- fontconfig: the font matching pattern language; font families, styles, weights; alias chains (`serif` → `Liberation Serif` → actual file); per-application overrides via `~/.config/fontconfig/fonts.conf`; the binary cache (`~/.cache/fontconfig/`); `fc-list`, `fc-match`, `fc-scan` tools
- Cairo: 2D compositing library; surfaces (`cairo_image_surface_t`, `cairo_gl_surface_t`, `cairo_xcb_surface_t`); the painting model (source, mask, operator); path construction and filling; text rendering via `cairo_show_glyphs`; the GL and Skia backends
- Pango: the text layout engine; `PangoLayout` and `PangoLayoutLine`; paragraph layout and line breaking; BiDi text in mixed RTL/LTR paragraphs; font selection via `PangoFontDescription`; integration with Cairo for rendering
- Glyph atlas management: how Qt (`QFontEngine`), GTK (`PangoCairoFcFontMap`), and Skia pack glyph bitmaps into GPU texture atlases; atlas overflow and LRU eviction; distance field fonts and SDF rendering for scalable glyphs
- Subpixel rendering on Wayland: why Wayland's composited pixel pipeline broke X11 LCD subpixel rendering (buffer compositing multiplies alpha, destroying channel offsets); the grayscale fallback; per-output subpixel orientation hints via `wl_output.subpixel`; FreeType flags per display
- Variable fonts: OpenType variable font axes (wght, wdth, slnt, ital); FreeType 2.7+ variable instance support; HarfBuzz variation API; rendering performance implications
- **Integrations**: FreeType/HarfBuzz are used by Qt (Ch39), GTK (Ch39), Skia (Ch37), Pango, and terminal emulators (Ch44); fontconfig is the font resolver for all of the above; Cairo's GL backend uses Mesa OpenGL (Ch19); Wayland's `wl_output.subpixel` hint (Ch20) controls FreeType rendering mode; the compositor's HiDPI scaling (Ch22) affects font metrics and atlas density

### Chapter 48: ROCm and Machine Learning on Linux GPUs
- ROCm stack overview: the KFD (Kernel Fusion Driver) at `/dev/kfd`; `amdkfd` kernel module and its relation to `amdgpu` DRM (Ch5); the HSA (Heterogeneous System Architecture) memory model; ROCm hardware support matrix (CDNA vs. RDNA generations)
- HIP runtime: `hipMalloc` / `hipMemcpy` / `hipLaunchKernelGGL`; HIP as a CUDA-portable API; `hipcc` compiler driver; `hipify-perl` and `hipify-clang` for CUDA→HIP porting
- ROCm compilation pipeline: HIP C++ → Clang/LLVM → AMDGPU LLVM backend → GCN/CDNA ISA; `amdgcn-amd-amdhsa` target triple; `--offload-arch=gfx942` for MI300X; comparison with Mesa's ACO (Ch15) — ACO is for graphics command streams, LLVM is for compute
- ML frameworks on ROCm: PyTorch ROCm backend (`torch.version.hip`); TensorFlow ROCm via `tensorflow-rocm`; JAX via `jax[rocm]`; how `hipblaslt` replaces `cuBLAS` and `MIOpen` replaces `cuDNN`
- Math library ecosystem: rocBLAS (BLAS), rocFFT, rocRAND, MIOpen (DNN primitives), hipSPARSE; kernel autotuning via `Tensile` (convolution/GEMM auto-tuner); `rocm-smi` for device and memory monitoring
- AMD CDNA3 / MI300X for ML: 192 CUs, HBM3 memory, FP8 support, Unified Memory Architecture (CPU+GPU in one address space); implications for large-model inference; `amdgpu` XGMI (Infinity Fabric) for multi-GPU communication
- Intel oneAPI on Linux: Level Zero (Ch25); `intel-compute-runtime` as the Level Zero ICD; SYCL/DPC++ compilation via `icpx`; `intel_gpu_top` for Arc workload monitoring; oneAPI vs. ROCm portability story
- ROCm containers and cloud: Docker/Kubernetes with `--device /dev/kfd --device /dev/dri/renderDN`; ROCm Device Plugin for Kubernetes; AWS EC2 `g4ad` (RDNA2) vs. `p4d` (A100/CUDA); AMD Instinct MI300X cloud availability
- **Integrations**: ROCm uses `amdgpu`'s KFD compute queue (Ch5), distinct from the DRM render node used by Mesa; HIP kernels compile via the same AMDGPU LLVM backend as `radeonsi` (Ch19) but at a different level; ML inference via Vulkan compute (Ch25) is an alternative for portable inference; GPU containers (Ch55) wrap this stack for cloud ML deployments; multi-GPU collective ops use p2p DMA (Ch49)

### Chapter 50: Vulkan Video Extensions
- Why Vulkan Video: the fragmentation of VA-API, VDPAU, and codec-specific APIs; the argument for a unified GPU video API inside Vulkan; timeline of `VK_KHR_video_*` specification development (2021–2025)
- `VK_KHR_video_queue`: the `VkVideoSessionKHR` object; video queue families (`VK_QUEUE_VIDEO_DECODE_BIT_KHR`, `VK_QUEUE_VIDEO_ENCODE_BIT_KHR`); session parameters objects; video format query (`vkGetPhysicalDeviceVideoFormatPropertiesKHR`)
- `VK_KHR_video_decode_queue`: decode operation lifecycle; reference picture list management; DPB (Decoded Picture Buffer) image arrays; `vkCmdDecodeVideoKHR`; output picture copy vs. DPB alias modes
- Codec extensions: `VK_KHR_video_decode_h264` and `VK_KHR_video_decode_h265` — codec-specific `VkVideoDecodeH264SessionParametersCreateInfoKHR`; `VK_KHR_video_decode_av1` (ratified 2024)
- `VK_KHR_video_encode_queue` and `VK_KHR_video_encode_h264`/`h265`: encode quality levels; rate control modes; quantisation parameter ranges; `vkCmdEncodeVideoKHR`
- Mesa implementation: RADV Vulkan Video on AMD VCN hardware (`src/amd/vulkan/radv_video.c`); ANV Vulkan Video on Intel MFX/VDENC (`src/intel/vulkan/anv_video.c`); current codec and profile support matrix
- FFmpeg Vulkan hwaccel: `AVCodecContext` with `AV_PIX_FMT_VULKAN`; `AVVkFrame` and `AVVulkanFramesContext`; Vulkan frame pool management; the decoder → Vulkan image → Wayland linux-dmabuf → display zero-copy path
- Comparison with VA-API: API complexity trade-offs; zero-copy display path; hardware support matrix; migration strategy for applications
- **Integrations**: Vulkan Video uses the same `VkDevice` (Ch18) and `VkImage`/`VkDeviceMemory` (Ch24) as graphics; decoded frames are `VkImage` objects importable via linux-dmabuf (Ch20) for zero-copy display; PipeWire (Ch38) can receive Vulkan Video decoded frames as DMA-BUF; VA-API (Ch26) remains the more widely supported alternative for applications targeting older hardware

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

### Chapter 56: Ray Tracing on Linux
- Hardware ray tracing overview: BVH (Bounding Volume Hierarchy) traversal in hardware; NVIDIA RT Cores (Turing+), AMD RDNA2+ Ray Accelerators, Intel Xe DG2/Arc Ray Tracing Units; the difference between fixed-function BVH traversal and shader-based ray casting
- Vulkan ray tracing extension suite: `VK_KHR_acceleration_structure` (BLAS/TLAS build); `VK_KHR_ray_tracing_pipeline` (shader SBT model); `VK_KHR_ray_query` (inline queries in any shader stage); `VK_KHR_deferred_host_operations` (async AS build on CPU thread pool)
- Acceleration structure lifecycle: `vkBuildAccelerationStructuresKHR`; BLAS from `VkAccelerationStructureGeometryTrianglesDataKHR`; TLAS from `VkAccelerationStructureInstanceKHR` array; compaction via `VK_QUERY_TYPE_ACCELERATION_STRUCTURE_COMPACTED_SIZE_KHR`; serialisation for disk caching
- Ray tracing shader model: ray generation shaders; intersection, any-hit, closest-hit, miss shaders; the shader binding table (SBT) layout; `vkCmdTraceRaysKHR` dispatch; ray payload and callable shaders
- RADV ray tracing: how RDNA2+ `ra` (Ray Accelerator) instructions are emitted; `radv_device_supports_rt()`; the `RADV_DEBUG=rt` shader dump; RADV ray tracing conformance status in dEQP-VK
- ANV ray tracing: Intel DG2/Arc Xe-HPG BVH hardware; ANV acceleration structure layout; the `xe_ray_query` Gen12+ ISA instruction
- NVK ray tracing: Turing/Ampere RT Core mapping in NVK (Ch10b); how the GSP abstraction (Ch9) surfaces RT capabilities; timeline for NVK RT conformance
- DXR via VKD3D-Proton: D3D12 `CreateRaytracingAccelerationStructure` → VKD3D-Proton → `VK_KHR_acceleration_structure`; game DXR compatibility list; shader binding table translation overhead
- Blender Cycles and OptiX vs. Vulkan RT: the `CYCLES_DEVICE` selection; OptiX on NVIDIA via CUDA (Ch25); HIP-RT on AMD via ROCm (Ch48); the Vulkan RT path in Cycles for cross-vendor support
- **Integrations**: acceleration structures are `VkBuffer`-backed GPU memory objects (Ch4, Ch24); ray tracing shaders compile through NIR (Ch14) and ACO (Ch15) for RADV; VKD3D-Proton DXR is the gaming client (Ch28); Blender Cycles is the creative tool client (Ch42); ray queries in compute shaders (Ch25) are the most portable ray tracing path across all three GPU vendors

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

### Chapter 55: GPU Containers and Cloud Compute
- Container GPU access model: Linux namespaces do not isolate DRM devices; the `/dev/dri/renderDN` and `/dev/kfd` device mount model; the `render` and `video` udev group approach
- NVIDIA Container Toolkit: `nvidia-container-runtime` OCI hook; CDI (Container Device Interface) specification; how `/dev/nvidia*`, `/dev/dri/renderDN`, and library injection work inside containers; `nvidia-smi` and NVML inside containers; `NVIDIA_VISIBLE_DEVICES` environment variable
- ROCm in containers: AMD's official Docker images (`rocm/dev-ubuntu-*`); `--device /dev/kfd --device /dev/dri/renderDN`; the `amdgpu` and `render` group requirements; ROCm version pinning vs. kernel driver version compatibility; `rocm-smi` inside containers
- Intel GPU in containers: `--device /dev/dri/renderDN`; `vainfo` to verify VA-API inside container; Intel compute runtime ICD path; `intel-gpu-tools` container access for performance counters
- Kubernetes GPU scheduling: NVIDIA GPU Device Plugin (`nvidia.com/gpu` resource); AMD ROCm Device Plugin (`amd.com/gpu`); Intel GPU Device Plugin (`gpu.intel.com/i915`); `MIG` (Multi-Instance GPU) for NVIDIA A100/H100 partitioning; GPU sharing and time-slicing
- WSL2 GPU path: `dxcore` Linux kernel driver; `dxgkrnl` virtual GPU DRM driver (`/dev/dxg`); Mesa's `d3d12` Gallium driver for OpenGL/OpenCL via DirectX; `wsl-opengl-iosurface` IPC bridge; DirectML as ML inference path; limitations vs. native Linux (no Vulkan compute, no DMA-BUF)
- GPU virtualisation: SR-IOV for Intel GVT-g/Xe (time-sliced GPU VM access); AMD MxGPU SR-IOV (spatial partitioning); VFIO GPU passthrough with `vfio-pci` for near-bare-metal VM performance; `mediated device` (`mdev`) framework in the kernel
- Cloud GPU instances: AWS `g4dn` (NVIDIA T4), `p4d` (A100), `g5g` (Graviton+T4G); GCP A100 and L4 instances; kernel driver version pinning on cloud AMIs; EFA (Elastic Fabric Adapter) and GPU RDMA via P2P DMA for distributed training
- **Integrations**: container PRIME device selection uses DRI_PRIME (Ch49) to route to the correct GPU; P2P DMA (Ch4) enables direct GPU-to-GPU communication across ROCm (Ch48) multi-GPU training; virtio-gpu (Appendix F) is the alternative VM display path; WSL2's d3d12 Gallium driver is an OpenGL-on-D3D12 layer analogous to Zink (Ch17); GPU power management (Ch51) needs `nvidia-smi -pm 1` persistence mode inside containers for predictable performance

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

### Chapter 52: Firefox and WebRender
- Firefox's rendering architecture vs. Chromium: retained-mode GPU display list vs. Chromium's tile-raster model; why Gecko moved GPU compositing into WebRender rather than a separate compositor process
- WebRender: `gfx/wr/` directory structure; the `BuiltDisplayList` wire format; `DisplayItem` types (text, image, border, box-shadow, clip); how CSS properties map to display items; the `RenderBackend` thread model
- WebRender's render graph: `RenderTask` tree; `PictureTask` for effects; `AlphaTask` for transparency; the `TextureCache` for glyph bitmaps, images, and rendered picture subtrees; `GpuCache` for per-draw-call uniforms
- Picture caching (`PictureCache`): caching entire CSS stacking contexts as GPU textures; invalidation on property change; tile-based partial invalidation (WR tiles, not CC tiles); how this reduces per-frame GPU work for largely static pages
- WebRender backends on Linux: the GL backend (`webrender_gl`, using `gleam` GL bindings); Vulkan backend (experimental, `wr_gfx_vk`); software fallback (`swgl` — software WebRender GL, a SIMD-accelerated software rasteriser); backend selection logic
- WebRender and Wayland: `RenderCompositorNativeLayer` abstraction; `NativeLayerWayland` using `wl_subsurface` for delegated compositing without readback; linux-dmabuf surface submission; the `wp_presentation` feedback path in WebRender's frame timing
- Gecko's WebGPU implementation: `wgpu-core` (the `wgpu` Rust library) as Gecko's WebGPU backend; how it diverges from Chrome's Dawn (Ch35); `wgpu`'s Vulkan backend on Linux vs. Dawn's Vulkan backend; WGSL compilation via `naga` in wgpu vs. Tint in Dawn
- Stylo: the Servo-derived parallel CSS layout engine in Firefox; how style values feed `DisplayItem` generation; the thread-parallel style computation model
- **Integrations**: WebRender uses Mesa OpenGL (Ch19) or Vulkan (Ch18) backends; font rendering uses FreeType/HarfBuzz (Ch47); Wayland linux-dmabuf integration (Ch20) is the zero-copy surface path; Gecko's wgpu WebGPU backend is architecturally parallel to Chrome's Dawn (Ch35); wgpu also underlies Bevy (Ch40), making it a shared Rust GPU abstraction across browser and game engine; picture caching is architecturally analogous to Viz's surface aggregation (Ch36)

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

## Part XII — Terminal Graphics

**Rationale**: Modern GPU-accelerated terminals (kitty, Ghostty, WezTerm) are fully-fledged Wayland/EGL clients that upload pixel graphics as GPU textures and composite them with glyph atlases in the same render pass. The terminal graphics story is therefore not a separate topic — it is the same Linux graphics stack (DRM, GBM, Mesa, linux-dmabuf, KMS) applied to a niche client type with three competing pixel-data protocols layered on top. These chapters map those protocols and rendering architectures onto the infrastructure already established in Parts I–VI.

**Target audience**: Terminal and TUI developers; systems developers who want to understand how a seemingly simple application type drives the full stack from Sixel decode to KMS scanout.

### Chapter 43: Terminal Pixel Protocols — Sixel, Kitty, and iTerm2

- **Sixel: DEC heritage and modern survival**
  - Origins: VT240/VT340 (1986); the six-pixel vertical band (sixel unit); original 16-color palette; modern 256-register extension in xterm, foot, WezTerm, and mlterm
  - Encoding mechanics: character range `?`–`~` (6 bits); band structure; the `"` raster-attribute command (DECGRA): `Pa`, `Ph`, `Pv` — pan/pad positioning and pixel dimensions; DCS initialiser `ESC P q [P1;P2;P3]` — aspect ratio, background mode, grid size
  - Color handling: `#` register selector; palette entry definition (`#n;2;r;g;b` RGB, `#n;1;h;l;s` HLS); up to 256 registers in extended mode; color quantization algorithms (Wu's, median-cut, neural-network); Floyd-Steinberg error diffusion vs. ordered (Bayer 4×4) dithering
  - Level 1 vs. Level 2: raster attributes enable pre-allocation of exact dimensions; Level 1 requires dynamic growth during decode
  - Limitations: 256-color cap causing visible banding on photographic content; no transparency layer; synchronisation with scrollback (image anchored at cursor row, misaligns on scroll without explicit compositor-level tracking); bandwidth verbosity (~3–5× JPEG for photographic content)
  - libsixel: streaming decode API vs. batch encode API; quantization hooks; integration patterns used by foot, xterm, VTE

- **Kitty Graphics Protocol: stateful, chunked, GPU-ready**
  - Design philosophy: eliminate repeated pixel transmission via server-side image storage; map cleanly to GPU texture management
  - APC escape framing: `ESC _ G <ctrl> ; <payload> ESC \\` — Application Program Command vs. OSC framing; 7-bit-safe base64 payload; comma-separated key=value control data
  - Pixel formats (`f` key): 24-bit RGB, 32-bit RGBA, PNG (`f=100`) — PNG allows self-describing dimensions; compression (`o=z`) via RFC 1950 ZLIB deflate
  - Transmission media (`t` key): direct (inline payload), regular file path, temporary file (auto-delete), POSIX shared memory object — the `t=s` path enables true zero-copy for same-machine clients
  - Chunking (`m` key): 4 KB chunks, `m=1` for all-but-last, `m=0` to finalise; base64 alignment requirement (multiples of 4 bytes); decompression on final chunk
  - Image persistence and IDs: 32-bit image ID; multiple placements referencing the same stored image; `a=d` deletion actions (by ID, by pixel coordinates, by z-index range, or all)
  - Placement system: cell-anchored positioning (`c`×`r` cell rectangle); pixel offsets within cell (`X`, `Y`); source-rectangle cropping (`x`, `y`, `w`, `h`); placement IDs for independent management; z-index layering — negative z renders beneath text, positive above
  - Unicode placeholder mechanism (`U+10EEEE`): diacritic-encoded row/column/image ID metadata; images move with text when lines are inserted/deleted; enables tmux and pager proxying without decoding the protocol
  - Relative placements: parent–child positioning with `P`/`Q` keys; dynamic repositioning when parent moves
  - Animation: frame sequences via `a=T`/`a=f`; per-frame background color, compositing mode, frame gap (ms); terminal-driven playback at display refresh rate
  - Feature detection: `a=q` query with terminal response; `q=1`/`q=2` response suppression for bulk operations
  - GPU mapping: IDs and placement table map directly to GPU texture handles and draw-call rectangles; z-index maps to render-layer ordering

- **iTerm2 Inline Images Protocol: simplicity and portability**
  - Escape sequence: `ESC ] 1337 ; File=[params] : <base64> BEL` (OSC 1337); atomic, stateless, no image IDs
  - Parameters: `name`, `size`, `width`/`height` (cells / px / %/ auto), `inline`, `preserveAspectRatio`, WezTerm-extension `doNotMoveCursor`
  - Payload: complete image file (PNG, JPEG, etc.) base64-encoded; relies on image format's own compression — no protocol-level compression
  - Multipart transmission (iTerm2 3.5+): `MultipartFile`/`FilePart`/`FileEnd` sequence for SSH/tmux compatibility; chunk limits 256 B (legacy tmux) to 1 MiB (modern)
  - Adoption: iTerm2 (macOS, reference), WezTerm (full), Hyper, Konsole (partial); simpler than Kitty but no animation, no z-index, no cropping at protocol level
  - Trade-offs vs. Sixel/Kitty: stateless simplicity wins for remote/tmux contexts; bandwidth inefficiency (no chunked compression, no deduplication) is the cost

- **Protocol comparison and selection guidance**
  - Bandwidth: Kitty (compressed, deduplicated) ≈ original JPEG; Sixel ≈ 3–5× JPEG for photos; iTerm2 ≈ original JPEG (no extra compression)
  - Feature matrix: animation (Kitty only), z-indexing (Kitty), transparency (Kitty/iTerm2, not Sixel), true-color (Kitty/iTerm2, not Sixel)
  - Portability: Sixel widest hardware/firmware support (embedded, DEC terminals); Kitty widest GPU-terminal support; iTerm2 best for macOS ecosystem and tmux-over-SSH
  - Detection: `\033[c` secondary DA query; terminal-specific OSC queries; libsixel and wezterm-imgcat implement auto-detection stacks

- **Integrations**: The pixel data that all three protocols carry ultimately becomes a GPU texture (Ch44); how that texture reaches the display is entirely determined by the terminal's Wayland/EGL/KMS integration (Ch45); color quantization in Sixel is the terminal-layer analogue of the KMS color pipeline (Ch3) but far lower fidelity; Kitty's shared-memory transmission mode (`t=s`) uses POSIX shm and is conceptually related to wl_shm (Ch45) and DMA-BUF (Ch4)

### Chapter 44: Terminal GPU Rendering Architectures

- **Glyph atlas fundamentals**
  - FreeType rasterization + HarfBuzz text shaping pipeline: ligatures, emoji, bidirectional text; cache miss paths
  - 2D texture atlas vs. 3D texture array: slice-per-font-size; dynamic atlas expansion without glyph reshuffling; GPU memory layout for fast per-cell lookup
  - Per-cell quad rendering: (row, col) → screen rect; fragment shader samples glyph + applies foreground/background color
  - Subpixel rendering: LCD filter geometry (R-G-B vs. V-RGB) vs. greyscale antialiasing; gamma-correct blending; fontconfig's `hintstyle`/`antialias` surface to the atlas rasterizer

- **kitty: mature OpenGL renderer**
  - OpenGL (not Vulkan): predates widespread Vulkan adoption; no plans to port
  - 3D texture array glyph atlas: each layer is a 2D grid of cells; `glTexSubImage3D` for incremental updates
  - Dedicated image texture pool: Kitty-protocol images stored in separate GPU textures; texture handle tracked per image ID
  - Single-pass compositing: one draw call renders glyph quads + image rectangles; z-index map determines fragment output order; alpha blending with premultiplied alpha
  - GPU image decode path: image data decoded CPU-side → uploaded via `glTexImage2D`; no GPU-side decode (no astc/etc2 paths)
  - Performance: 60–120 FPS typical; bottleneck is texture upload on large images or high-DPI glyph atlases

- **Alacritty: minimal OpenGL, latency-optimised**
  - OpenGL + GLFW abstraction; similar atlas approach; deliberately excludes image protocol support
  - Focus: sub-millisecond input→visual latency; first GPU terminal to validate the atlas-based approach as practical
  - Architecture lessons: separation of input polling, terminal state, and render loop; double-buffered atlas to avoid mid-frame updates

- **WezTerm: wgpu multi-backend**
  - Rust + wgpu abstraction layer: Vulkan (Linux preferred), Metal (macOS), DirectX 12 (Windows), WebGPU (browser), OpenGL ES fallback
  - Glyph atlas and image compositing: all three protocols (Sixel, Kitty, iTerm2) decoded CPU-side → GPU texture
  - wgpu on Linux resolves to the Vulkan backend via Mesa drivers (Ch18); shader compilation via naga → SPIR-V → NIR (Ch14)
  - Known colour issue: HLS colour space handling in Sixel differs from xterm (RGB normalisation mismatch); this is a quantisation artefact, not a Mesa driver bug
  - Platform-agnostic performance: ~60–90 FPS on Linux Vulkan; slightly behind Ghostty's native approach

- **Ghostty and libghostty: Zig-native, platform-optimised**
  - Zig implementation: 78.9% Zig, 11.3% Swift (macOS UI layer), 4.7% C; memory safety and SIMD-optimised VT parser
  - Multi-threaded architecture: separate read, write, and render threads; avoids jank from large VT stream bursts (e.g., `cat` of large files)
  - Platform-native rendering backends:
    - macOS: Metal (highest performance, ~120 FPS, ~45 MB RSS)
    - Linux: OpenGL (current mainline) with Vulkan path in development
    - Windows: DirectX 12
  - GPU rendering: per-platform glyph atlas (3D texture arrays); image compositing (Kitty and iTerm2 protocols) done GPU-side in terminal render pass
  - libghostty: factored-out VT parsing and terminal state machine as a cross-platform C/Zig library; provides the full emulation core without the rendering layer; enables embedding in other applications; API documented at `libghostty.tip.ghostty.org`; WebAssembly target for browser embedding; not yet a stable versioned release (2026)
  - Performance: benchmarked as fastest terminal on several metrics (0.7 s to `cat` 100 K lines); SIMD parser contribution is the dominant factor over rendering

- **foot: CPU-side software rendering**
  - Design rationale: Wayland-first, zero GPU dependency — suitable for remote SSH sessions, minimal-RAM embedded deployments
  - Server-daemon architecture: single `footd` process hosts multiple terminal windows; shared FreeType font cache and glyph library; lower per-window memory footprint (~30–50 MB vs. 60–100 MB for GPU terminals)
  - Rendering pipeline: FreeType rasterize → 32-bit RGBA software framebuffer → damage-region tracking → `wl_shm` upload to Wayland
  - Sixel integration: `sixel.c` decoder; decoded image rasterized into software framebuffer at correct cell offset; composited below text (z-order: image then text)
  - Fractional scaling: `wp_fractional_scale_v1` for HiDPI; DPI-correct glyph sizing without GPU involvement
  - Frame rate: 50–60 FPS on modern CPUs; drops during bulk scroll redraws; input latency comparable to GPU terminals (5–10 ms) via Wayland frame-clock

- **VTE (GNOME terminal library): GTK4 transition**
  - Role: GTK widget providing terminal emulation for GNOME Terminal, Tilix, and others; not a standalone terminal
  - GTK3 path: Cairo CPU rasterization → ~20–30 FPS; full-texture upload each frame
  - GTK4 path (VTE ≥ 0.76, 2024): GSK (GTK Scene Graph) GPU backend — Vulkan or OpenGL; 60 FPS via display frame-clock integration; lz4 scrollback compression replacing zlib
  - Sixel support: `vte_terminal_set_enable_sixel()` API; build-time `-Dsixel=true` Meson flag; decoded via libsixel or internal decoder; uploaded to GPU as GSK texture node
  - GNOME Terminal enablement: not on by default in most distributions; must be explicitly built or configured

- **Compositing pipeline: text + pixel graphics**
  - Z-ordering model: sort all draw elements (glyph quads, image rectangles) by z-index; back-to-front alpha blend; premultiplied alpha avoids fringe artefacts
  - Image vs. text layering: negative Kitty z-index embeds image beneath text — useful for background images, sparkline overlays in TUI dashboards (btop++, lazygit)
  - Single-pass vs. multi-pass rendering: Kitty uses a single render pass; VTE with GTK4/GSK may use multiple GSK render node types (texture node for images, text node for glyphs)
  - Memory management: GPU texture eviction policy for large image pools (Kitty: LRU eviction when total GPU texture memory exceeds threshold)

- **Integrations**: glyph atlas shaders are plain GLSL/SPIR-V that pass through Mesa NIR (Ch14) and ACO/LLVM backends (Ch15) for AMD/NVIDIA/Intel; WezTerm's wgpu SPIR-V path is structurally identical to Bevy/wgpu (Ch40) and Dawn (Ch35); the libghostty VT parser is architecturally related to the VTE widget and could be embedded in a Wayland compositor's built-in terminal; GTK4 VTE rendering uses the same GSK Vulkan backend as other GTK4 widgets (Ch39); font shaping via HarfBuzz is shared infrastructure with Skia text rendering in Chrome (Ch37) and Pango/Cairo in GTK3 apps

### Chapter 45: Terminal Integration with the Compositor Stack

- **The terminal as a Wayland client**
  - A GPU-accelerated terminal is indistinguishable from any other Wayland client at the protocol level: it creates a `wl_surface`, acquires a GPU context via EGL, renders its framebuffer, and submits via linux-dmabuf
  - `xdg_toplevel` surface role: window management, title, app-id; `xdg_surface.configure` → resize → reallocate GBM buffers
  - Multi-window terminals (kitty, foot server mode): one Wayland connection, multiple `wl_surface` objects; shared EGL display

- **EGL context and GBM buffer allocation**
  - EGL on Wayland: `eglGetPlatformDisplayEXT(EGL_PLATFORM_WAYLAND_KHR, ...)` → `eglCreateWindowSurface` wrapping a `wl_egl_window`; Mesa's EGL WSI creates GBM-backed buffers internally
  - GBM buffer allocation: `gbm_device_create` from a DRM render node fd; `gbm_surface_create_with_modifiers2` — DRM format modifier negotiation ensures tiling layout is compatible with KMS scanout
  - Format modifier negotiation: compositor advertises supported modifiers via `zwp_linux_dmabuf_v1.modifier` events; terminal's EGL/GBM selects an intersection — critical for enabling zero-copy scanout
  - Double-buffering: two GBM buffers in flight; `eglSwapBuffers` queues a buffer release event; terminal waits for `wl_buffer.release` before reusing

- **DMA-BUF export and linux-dmabuf submission**
  - After `eglSwapBuffers`, Mesa's WSI exports the GBM backing store as a DMA-BUF fd via `gbm_bo_get_fd`
  - `zwp_linux_dmabuf_v1.create_params` + `add` (plane, fd, offset, stride, modifier) + `create_immed` → `wl_buffer` handle
  - `wl_surface.attach(wl_buffer, 0, 0)` + `wl_surface.damage_buffer(...)` + `wl_surface.commit()`
  - Compositor receives the commit: imports the DMA-BUF into a GPU texture (Mesa `EGLImage` or Vulkan external image); adds to composition list

- **Explicit sync: `wp_linux_drm_syncobj`**
  - Why implicit fences fail for NVIDIA: GSP-RM does not expose fence timeline objects; the compositor cannot safely read the buffer without an explicit signal
  - `wp_linux_drm_syncobj_surface_v1.set_acquire_point(timeline, point)` — compositor waits for this point before reading the buffer
  - `set_release_point(timeline, point)` — compositor signals this point when it has finished reading, allowing the terminal to reuse the buffer
  - Timeline sync objects are DRM sync objects (Ch3); the same mechanism governs all Wayland clients — the terminal is not special; this chapter grounds the Ch3 theory in a concrete application

- **CPU-path terminals (foot): wl_shm**
  - Foot allocates anonymous shared memory via `memfd_create` + `mmap`
  - `wl_shm.create_pool` + `wl_shm_pool.create_buffer` → `wl_buffer` backed by CPU-accessible memory
  - Compositor imports the shm buffer as a CPU-side texture upload (no DMA-BUF, no GPU-side import)
  - Performance implication: compositor must do a CPU→GPU upload each frame; zero-copy scanout is impossible; adds ~1–2 ms compositor overhead per frame on large terminals

- **Compositor-side processing**
  - Damage accumulation: compositor merges damage regions from all client commits; repaints only damaged areas
  - Buffer import: DMA-BUF-backed buffers imported as `EGLImage` → `GL_TEXTURE_EXTERNAL_OES` or Vulkan `VkImage` with external memory; format modifier preserved
  - Plane promotion (zero-copy scanout): if the terminal's buffer has the correct DRM format modifier for a KMS plane, the compositor may assign it directly to a KMS overlay plane — no re-render; the terminal pixel data is scanned out directly by the display engine; requires: single terminal window covering exactly one output; no compositor effects (blur, rounding) on the surface; compositor support (wlroots' DRM backend does this; Mutter: partial)
  - Fallback path: compositor renders all surfaces into its own output framebuffer; terminal buffer is sampled as a texture

- **KMS atomic commit: from compositor to display**
  - Compositor's output framebuffer (or promoted client buffer) assigned to KMS primary/overlay plane
  - Atomic commit: `DRM_IOCTL_MODE_ATOMIC` — plane `FB_ID`, `CRTC_ID`, `SRC_*`, `CRTC_*`; fence fd for non-blocking commit
  - VBLANK: display engine reads framebuffer at next VBLANK; pixel pipeline → connector → physical display
  - `wp_presentation` feedback: compositor sends `presented` event with VBLANK timestamp and refresh duration; terminal uses this for frame pacing (avoids over-rendering)

- **Colour space and HDR considerations**
  - All three pixel protocols assume sRGB (no metadata to signal otherwise)
  - If the KMS pipeline is configured for HDR (`wp_color_management_v1`, Ch3), the compositor will tone-map terminal pixel content from sRGB to the display's colour volume — usually invisibly for text, but can shift image colours
  - Future extension opportunity: a Kitty protocol extension carrying HDR metadata (`o=hdr` flag + colour volume descriptor) would allow the compositor to handle HDR terminal graphics correctly, mirroring `wp_color_representation_v1` used for VA-API video surfaces (Ch3, Ch26); this remains speculative as of 2026

- **Security model**
  - Terminal emulators open a DRM render node (`/dev/dri/renderDN`) — no DRM master privilege, no display ownership; threat model is the same as any other GPU client (Ch30 security section)
  - Terminal pixel graphics do not expand the attack surface: DMA-BUF import into the compositor is mediated by the kernel's DMA-BUF framework; a malformed Sixel/Kitty payload can corrupt terminal state but cannot escape the compositor's process boundary
  - Sandboxed terminals (Flatpak): must be granted `--device=dri` or use a portal; portal screen-capture for terminal content uses PipeWire (Ch38)

- **Integrations**: this chapter is the concrete application of DRM render nodes (Ch1), GBM/DMA-BUF (Ch4), linux-dmabuf and `wp_presentation` (Ch20), explicit sync / `wp_linux_drm_syncobj` (Ch3), KMS atomic commit (Ch2), and compositor plane promotion (Ch21, Ch22); the wl_shm path used by foot is the same CPU-texture-upload path used by Wayland clients that lack GPU acceleration; the explicit-sync narrative closes the loop opened by Ch3's discussion of NVIDIA's fence limitations; PipeWire screen capture of terminal windows (Ch38) and sandboxed terminal access (Ch23) are the security-adjacent topics this chapter connects to

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
