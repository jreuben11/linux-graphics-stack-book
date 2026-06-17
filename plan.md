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
- **Part XIII — Video Streaming on Linux**
  - [Chapter 57: FFmpeg Architecture and Programming](#chapter-57-ffmpeg-architecture-and-programming)
  - [Chapter 58: GStreamer: Pipeline-Based Multimedia](#chapter-58-gstreamer-pipeline-based-multimedia)
  - [Chapter 59: NVIDIA DeepStream SDK](#chapter-59-nvidia-deepstream-sdk)
  - [Chapter 60: Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs](#chapter-60-video-compression-algorithms-dct-motion-estimation-and-modern-codecs)
  - [Chapter 60b: Video Streaming Protocols and Adaptive Bitrate Delivery](#chapter-60b-video-streaming-protocols-and-adaptive-bitrate-delivery)
- **Part XIV — Khronos Extended Ecosystem**
  - [Chapter 61: SPIR-V Ecosystem in Depth](#chapter-61-spir-v-ecosystem-in-depth)
  - [Chapter 62: SYCL 2020 and Portable Heterogeneous Compute](#chapter-62-sycl-2020-and-portable-heterogeneous-compute)
  - [Chapter 63: KTX2, Basis Universal, and GPU Texture Compression](#chapter-63-ktx2-basis-universal-and-gpu-texture-compression)
  - [Chapter 64: glTF 2.0 — The 3D Asset Pipeline Standard](#chapter-64-gltf-20--the-3d-asset-pipeline-standard)
  - [Chapter 65: Vulkan Safety Critical and OpenVX](#chapter-65-vulkan-safety-critical-and-openvx)
- **Part XV — NVIDIA Proprietary Graphics Stack**
  - [Chapter 66: CUDA Runtime, Streams, and NVRTC](#chapter-66-cuda-runtime-streams-and-nvrtc)
  - [Chapter 67: OptiX 9 — NVIDIA's Ray Tracing Framework](#chapter-67-optix-9--nvidias-ray-tracing-framework)
  - [Chapter 68: DLSS 4, Neural Rendering, and Frame Generation](#chapter-68-dlss-4-neural-rendering-and-frame-generation)
  - [Chapter 69: NVIDIA Omniverse, OpenUSD, and the RTX Renderer](#chapter-69-nvidia-omniverse-openusd-and-the-rtx-renderer)
  - [Chapter 70: RTX Kit — RTXDI, RTXGI, NRD, RTXNS, and RTXNTC](#chapter-70-rtx-kit--rtxdi-rtxgi-nrd-rtxns-and-rtxntc)
- **Part XVI — Intel Open Graphics Stack**
  - [Chapter 71: Intel Xe Kernel Driver, Arc GPU Architecture, and the Intel Open Stack](#chapter-71-intel-xe-kernel-driver-arc-gpu-architecture-and-the-intel-open-stack)
- **Part XVII — AMD Developer Ecosystem**
  - [Chapter 72: AMD FidelityFX SDK and Radeon Developer Tools](#chapter-72-amd-fidelityfx-sdk-and-radeon-developer-tools)
- **Part XVI/XVII additions to existing parts**
  - [Chapter 73: Asahi Linux and the Apple Silicon AGX Driver](#chapter-73-asahi-linux-and-the-apple-silicon-agx-driver) *(Part II)*
  - [Chapter 74: HDR and Wide Color Gamut on Linux](#chapter-74-hdr-and-wide-color-gamut-on-linux) *(Part VI)*
  - [Chapter 75: Explicit GPU Synchronisation](#chapter-75-explicit-gpu-synchronisation) *(Part VI)*
  - [Chapter 76: Modern Vulkan Extensions](#chapter-76-modern-vulkan-extensions) *(Part VII)*
  - [Chapter 77: Shader Source-to-ISA: The Complete Compilation Toolchain](#chapter-77-shader-source-to-isa-the-complete-compilation-toolchain) *(Part IV)*
  - [Chapter 78: Gamescope and the Steam Deck: A Complete Gaming Graphics Stack](#chapter-78-gamescope-and-the-steam-deck-a-complete-gaming-graphics-stack) *(Part VIII)*
  - [Chapter 79: Remote Display, Screen Casting, and GPU-Accelerated Game Streaming](#chapter-79-remote-display-screen-casting-and-gpu-accelerated-game-streaming) *(Part IX)*
  - [Chapter 80: GPU Security: Isolation, Content Protection, and Confidential Computing](#chapter-80-gpu-security-isolation-content-protection-and-confidential-computing) *(Part IX)*
- **Part XVIII — Rendering Abstraction Libraries**
  - [Chapter 81: SDL3 GPU API: A Portable High-Level GPU Abstraction](#chapter-81-sdl3-gpu-api-a-portable-high-level-gpu-abstraction)
  - [Chapter 82: Vulkan Ecosystem Toolkit: VMA, volk, vk-bootstrap, and Friends](#chapter-82-vulkan-ecosystem-toolkit-vma-volk-vk-bootstrap-and-friends)
  - [Chapter 83: Filament: Google's Physically Based Rendering Engine on Linux](#chapter-83-filament-googles-physically-based-rendering-engine-on-linux)
  - [Chapter 84: bgfx, Cross-Platform Rendering Abstractions, and the Frame Graph Pattern](#chapter-84-bgfx-cross-platform-rendering-abstractions-and-the-frame-graph-pattern)
- **Part XIX — Android Graphics**
  - [Chapter 85: Android Compositor: SurfaceFlinger, HardwareBuffer, and the Buffer Pipeline](#chapter-85-android-compositor-surfaceflinger-hardwarebuffer-and-the-buffer-pipeline)
  - [Chapter 86: Vulkan on Android: Drivers, ANGLE, and Mobile GPU Performance](#chapter-86-vulkan-on-android-drivers-angle-and-mobile-gpu-performance)

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

## Part XIII — Video Streaming on Linux

While Chapter 26 surveys hardware video APIs (VA-API, V4L2, GStreamer pipeline construction) and Chapter 50 covers Vulkan Video extensions, this part goes deeper: FFmpeg's internal architecture and C programming API, GStreamer plugin development and DMA-BUF buffer passing, NVIDIA's DeepStream SDK for GPU-accelerated video analytics, and the foundational codec algorithms (DCT, motion estimation, AV1, H.265/HEVC) that underlie every hardware decode path in the stack.

**Target audience**: Graphics application developers and systems developers integrating video pipelines; backend engineers building GPU-accelerated transcoding, streaming, or analytics systems.

### Chapter 57: FFmpeg Architecture and Programming

- FFmpeg component overview: libavformat, libavcodec, libavfilter, libavdevice, libswscale, libswresample; the `AVFormatContext`, `AVStream`, `AVCodecContext`, `AVPacket`, `AVFrame` lifecycle
- Demuxing and decoding: `avformat_open_input` → `avformat_find_stream_info` → `av_find_best_stream` → `avcodec_open2` → `av_read_frame` → `avcodec_send_packet` / `avcodec_receive_frame`; reference counting via `av_frame_ref` and `av_packet_unref`
- Hardware acceleration: `AVHWDeviceContext` for VAAPI, VDPAU, Vulkan; `AVHWFramesContext`; zero-copy decode with DMA-BUF export from `AVVkFrame`; `AV_PIX_FMT_VAAPI`, `AV_PIX_FMT_VULKAN` pixel format chains
- The lavfi filter graph: `avfilter_graph_alloc`, `AVFilterInOut`, `avfilter_graph_parse2`, `avfilter_graph_config`; GPU filter chains (`scale_cuda`, `overlay`, `drawtext`) vs. CPU filters; sink/source pad negotiation
- Encoding pipeline: `AVCodecContext` encode parameters, rate control modes (CBR, VBR, CRF, CQP), GOP structure, B-frame distance; hardware encoders (`vaapi_encode`, `nvenc_h264`, `amf_h264`)
- FFmpeg Vulkan hwaccel in depth (cross-ref Ch50): `AVVkFrame`, `AVVulkanFramesContext`, `AVVulkanDeviceContext`; zero-copy Vulkan image pipeline from decode to display
- The ffmpeg CLI architecture: the lavf/lavc dispatch model; stream mapping (`-map`), codec copy mode (`-c copy`), complex filtergraph (`-filter_complex`); threading model (`-threads`, `-filter_threads`); the segment muxer for HLS
- Streaming protocols: RTSP, RTMP, SRT, HLS, DASH; latency modes; SRS and mediamtx as open RTMP/SRT servers on Linux; low-latency HLS (`-hls_flags delete_segments`)
- Custom `AVCodec` and `AVFilter` plugins: the `AVCodec.decode` callback contract; `AVFilter.query_formats` / `filter_frame` callbacks; `plugin_init` symbol export; Meson integration in an out-of-tree plugin
- **Integrations**: FFmpeg Vulkan hwaccel uses the same `AVVkFrame` objects that Vulkan Video (Ch50) produces; VA-API decode (Ch26) feeds FFmpeg via `AVHWDeviceContext`; GStreamer wraps FFmpeg codecs via the `gst-libav` plugin (Ch58); NVIDIA's NVENC hardware encoder is the FFmpeg path into the CUDA/NVENC stack (Ch66); DeepStream (Ch59) exposes a GStreamer+FFmpeg API surface for inference pipelines; codec algorithms (Ch60) are the mathematical foundation of every `libavcodec` software decoder used here

### Chapter 58: GStreamer: Pipeline-Based Multimedia

- GStreamer object model: `GstElement`, `GstPad`, `GstCaps`, `GstBuffer`, `GstPipeline`; the GLib type system and `GObject` property model; `gst_element_factory_make` vs. `gst_parse_launch`
- Pipeline construction patterns: programmatic element linking with `gst_element_link_filtered`; dynamic pads — `decodebin` / `uridecodebin` `pad-added` signal; `parsebin` for format detection; `capsfilter` for capability pinning
- The `GstBuffer` memory model: `GstMemory`, `GstMapInfo`; `GstDmaBufAllocator` for DMA-BUF–backed buffers; zero-copy buffer passing between GPU elements; `GstVideoMeta` plane stride and offset annotations for multi-planar NV12/P010 formats
- VA-API elements: `vaapi{decode,encode,postproc}` in `gstreamer-vaapi`; `vah264dec`, `vah265dec`, `vaav1dec` in `gst-plugins-bad` (Mesa VA-API driver bridge); DMA-BUF output caps negotiation with downstream Vulkan consumers
- V4L2 elements: `v4l2src`, `v4l2h264dec`, `v4l2hevcenc`; the stateless codec path via V4L2 request API; `v4l2convert` for format negotiation; `libcamera` source via `libcamerasrc`
- Clock and synchronisation: `GstClock`, buffer `pts`/`dts`/`duration`; pipeline clock election algorithm; live sources and `min-latency`; `gst_base_src_set_live`; `queue2` for buffering; leaky queues for real-time pipelines
- Inter-process pipelines: `GstNet` for network clock synchronisation; `fdsrc` / `fdsink`; `ipcpipelinesrc` / `ipcpipelinesink` for multi-process decoupling; PipeWire `pipewiresrc` / `pipewiresink` (Ch38)
- Writing GStreamer plugins: `GstBaseSrc`, `GstBaseTransform`, `GstBaseParser` base classes; the `plugin_init` export; GObject boilerplate with `G_DEFINE_TYPE`; caps renegotiation and `gst_base_transform_reconfigure_src`; Meson `gst-plugin` build helper
- Debugging and profiling: `GST_DEBUG` category/level; `GST_DEBUG_DUMP_DOT_DIR` for pipeline graph export to Graphviz; `gst-inspect-1.0` element capability listing; `gst-shark` for trace-level profiling; `gst-launch-1.0 -v` verbose negotiation output
- **Integrations**: `gst-libav` wraps FFmpeg codecs (Ch57); `gstreamer-vaapi` and `gst-plugins-bad` VA-API elements use Mesa VA-API drivers on amdgpu/i915 (Ch5, Ch26); PipeWire (Ch38) uses SPA node concepts analogous to `GstElement` and can act as `pipewiresrc`/`pipewiresink`; DeepStream (Ch59) is a NVIDIA-proprietary GStreamer plugin set that extends the infrastructure described here; V4L2 elements exercise the same V4L2 kernel path as Ch26; libcamera (Ch26) feeds GStreamer via `libcamerasrc`

### Chapter 59: NVIDIA DeepStream SDK

- DeepStream architecture: the GStreamer meta-plugin stack — `Gst-NvInfer`, `Gst-NvStreammux`, `Gst-NvDsPostProcess`, `Gst-NvVideoConvert`; `NvBufSurface` as the CUDA-managed DMA-BUF–compatible buffer type shared across all elements
- NvInfer: TensorRT engine inference on batched video streams; `nvinfer` element configuration (`config-file-path`, `batch-size`, `interval`); primary vs. secondary inference cascades; output tensor → `NvDsObjectMeta` / `NvDsFrameMeta` attachment; confidence thresholding and NMS
- Tracker elements: `NvDCF` (discriminative correlation filter), `NvSORT` (SORT with Kalman filter), `ByteTrack` (via `Gst-NvTracker`); bounding-box tracking across frames without re-inference; multi-stream object ID consistency; tracker configuration via YAML
- `nvstreammux` and demux: multiplexing up to 32 concurrent video streams into batched tensor inputs; resolution alignment; timestamp synchronisation across streams with different frame rates; `nvstreamdemux` to route per-stream metadata back downstream
- Message broker: `Gst-NvMsgBroker`; structured analytics output via Kafka, Azure IoT Hub, AMQP/MQTT; `NvDsEventMsgMeta` payload schema; `nvmsgconv` for schema transformation
- TensorRT engine creation: ONNX → `trtexec` → serialised engine plan; INT8 calibration for throughput vs. accuracy trade-off; engine caching; precision-GPU architecture matrix (Ampere FP8, Ada INT8 throughput)
- CUDA interop: `NvBufSurface` as both a DMA-BUF fd (importable into Vulkan via `cudaExternalMemory`, Ch25) and a CUDA device pointer; `NvBufSurfaceCopy` for GPU-to-CPU readback; writing a custom GStreamer element that operates on `NvBufSurface` with a CUDA kernel
- DeepStream vs. open alternatives: MediaPipe (CPU/NPU inference) on Linux; OpenCV DNN on VA-API (`cv::dnn::readNetFromONNX` + `cv::dnn::DNN_TARGET_OPENCL`); Triton Inference Server with GStreamer pipelines; when the open stack is appropriate vs. DeepStream's integrated optimised path
- **Integrations**: DeepStream sits above GStreamer (Ch58), TensorRT/CUDA (Ch66), and NVML (Ch66); NvBufSurface bridges the CUDA DMA-BUF model (Ch25 CUDA–Vulkan interop) and GStreamer's `GstDmaBufAllocator` (Ch58); VA-API (Ch26) and V4L2 (Ch26) are the equivalent open alternatives for hardware decode; PipeWire (Ch38) can feed camera streams into DeepStream via `pipewiresrc`; the tracker algorithms connect to the motion estimation concepts in Ch60

### Chapter 60: Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs

- The compression duality: intra-frame (spatial) and inter-frame (temporal) redundancy removal; the rate–distortion trade-off; PSNR and SSIM as quality metrics; VMAF as a perceptual metric
- Discrete Cosine Transform (DCT): the 2D separable 8×8 transform; basis functions and energy compaction; quantisation matrices (luma vs. chroma); zig-zag scan ordering; Huffman coding in MJPEG; CABAC entropy coding in H.264/H.265
- Motion estimation and compensation: block matching (full search, diamond search, EPZS hexagonal); half-pel and quarter-pel sub-pixel interpolation via 6-tap filter; the reference picture buffer (DPB); bidirectional B-frame prediction and decoding order vs. presentation order
- H.264/AVC: NAL unit types (IDR, P, B, SEI); SPS and PPS parameter sets; slice header; CAVLC vs. CABAC entropy coding; in-loop deblocking filter; profile/level constraints and their hardware capability implications
- H.265/HEVC: CTU (64×64 Coding Tree Unit) quadtree hierarchy; CU, PU, TU split decisions; 35-direction intra prediction; HEVC entropy coding (CABAC with larger context model than H.264); deblocking and SAO (Sample Adaptive Offset) loop filters; how CTB-level parallelism maps to GPU wavefront execution for hardware decode
- AV1: superblock structure (128×128); compound and warped motion models; recursive intra prediction and palette mode; constrained directional enhancement filter (CDEF); Loop Restoration filter; arithmetic coder (ANS); the `libaom` reference encoder and `rav1e` Rust encoder; why AV1 is computationally heavier but royalty-free
- VVC/H.266: affine motion compensation; subblock transform; Geometric Partition Mode (GPM); the `VVdeC` open decoder on Linux; codec complexity vs. HEVC and AV1 compression efficiency
- GPU acceleration of codec decode: wavefront parallel processing (WPP) in HEVC enabling row-level GPU parallelism; AV1 tile-based parallel decode; the AV1 film grain synthesis GPU kernel; how VA-API (Ch26) and Vulkan Video (Ch50) map these parallel decode paths to GPU compute queues
- Rate control algorithms: CBR, VBR, CRF, CQP; the x264/x265 rate control loop (`ABR` → lookahead → QP assignment); two-pass encoding for VoD; hardware rate control in AMD AMF and Intel MSDK
- **Integrations**: these algorithms are the mathematical foundation of VA-API (Ch26) and Vulkan Video (Ch50) hardware decode; FFmpeg (Ch57) exposes software codec implementations via libavcodec; GStreamer (Ch58) wraps them via `gst-libav`; Vulkan AV1 decode structures (`VkVideoDecodeAV1PictureInfoKHR`) map directly to the frame header described here; NVIDIA NVENC and AMD AMF implement rate control and motion estimation in fixed-function silicon, referenced in Ch66 and Ch68

---

### Chapter 60b: Video Streaming Protocols and Adaptive Bitrate Delivery

- HLS (HTTP Live Streaming, RFC 8216): master playlist and media playlist grammar; `#EXT-X-TARGETDURATION`, `#EXT-X-MEDIA-SEQUENCE`, `#EXT-X-KEY` (AES-128, SAMPLE-AES); segment containers — MPEG-2 TS vs fMP4/CMAF (`#EXT-X-MAP`); Low-Latency HLS — parts (`#EXT-X-PART`), preload hints (`#EXT-X-PRELOAD-HINT`), can-block-reload (`_HLS_msn` long-poll), delta playlists (`_HLS_skip`); FFmpeg `hls` muxer options (`-hls_segment_type fmp4`, `-hls_flags low_latency`)
- MPEG-DASH (ISO/IEC 23009-1): MPD XML hierarchy (`MPD` → `Period` → `AdaptationSet` → `Representation`); `type=dynamic` vs `type=static`; segment addressing — `$Number$`-based, `$Time$`-based with `SegmentTimeline`, `SegmentList`; CMAF (ISO 23000-19) as shared fMP4 segment format for HLS+DASH; Low-Latency DASH via HTTP `Transfer-Encoding: chunked` with `availabilityTimeOffset` and `availabilityTimeComplete=false`; DASH-IF IOP live profile; Common Encryption (CENC) multi-DRM
- WebRTC (RFC 8825): SDP offer/answer exchange; ICE candidate gathering — host, server-reflexive (STUN, RFC 8489), relayed (TURN, RFC 8656); DTLS-SRTP key derivation (RFC 5764); RTP fixed header — sequence number, timestamp clock rates (90 kHz video, 48 kHz audio), M-bit, SSRC; H.264 packetisation (RFC 6184): Single NAL, STAP-A, FU-A; RTCP feedback: PLI, NACK, REMB, Transport-CC (RFC 9143) and GoogCC bandwidth estimation; GStreamer `webrtcbin` → `vaapih264dec` zero-copy path; SFU topology vs. mesh; WHIP/WHEP signalling (RFC 9725)
- SRT (Secure Reliable Transport): UDT-based; four-way HSREQ/HSRSP + KMREQ/KMRSP handshake; Caller/Listener/Rendezvous/Group (bonding) modes; ARQ with configurable latency budget (`SRTO_LATENCY`); `SRTO_PASSPHRASE` AES-128/256 encryption; latency sizing formula (4× RTT minimum); libsrt Linux integration; FFmpeg `srt://` protocol; GStreamer `srtsrc`/`srtsink`
- QUIC-based media: WebTransport (W3C) — QUIC streams and datagrams for browser push delivery; MOQT (Media Over QUIC Transport, `draft-ietf-moq-transport`) — Track/Group/Object data model, SUBSCRIBE control, CDN relay caching, QUIC datagrams for real-time audio; latency positioning between WebRTC (< 500 ms) and HLS (> 5 s)
- RTSP (RFC 7826) and RTMP: RTSP as camera/NVR control protocol (ONVIF mandates RTSP/RTP); RTMP as ingest-only protocol (OBS → YouTube/Twitch); RTMPS; `gst-rtsp-server`; nginx-rtmp
- Adaptive Bitrate algorithms: throughput-based (EWMA, `α=0.8`, safety factor 0.8–0.9) and its oscillation failure mode; Buffer-Based Rate Adaptation (BBA, SIGCOMM 2014) — `buf_lo`/`buf_hi` thresholds, throughput-agnostic; BOLA (IEEE/ACM ToN 2020) — Lyapunov drift-plus-penalty control, `Δ(m) = (V·v_m − buffer) / size_m`, provably near-optimal, implemented in dash.js and Shaka Player; MPC (SIGCOMM 2015) — finite-horizon optimisation over `K=5` segments, harmonic-mean predictor, penalty terms for quality, switching, and rebuffering; Pensieve (SIGCOMM 2017) — deep reinforcement learning policy network (3-layer FC, ~30K params), state vector: last 8 throughputs + download times + buffer level + current bitrate + next segment sizes; sidecar deployment on Linux
- CMAF segment packaging: `moof`+`mdat` structure; `tfdt` absolute decode times for CDN chunking; HTTP `Transfer-Encoding: chunked` for LL-DASH; `nginx proxy_buffering off` and `X-Accel-Buffering: no`; byte-range `sidx` seeks for VOD
- CDN strategies: immutable segment caching (`Cache-Control: max-age=31536000, immutable`); manifest short TTL (`s-maxage=1`); origin shielding; manifest rewriting for multi-CDN failover; HTTP/3 (QUIC) at the edge; byte-range keyframe seeks
- Linux server landscape: SRS (Simple Realtime Server) — C++, co-routine model, RTMP+HLS+DASH+WebRTC+SRT; MediaMTX — Go, zero-config, all protocols; nginx-rtmp — production-stable C nginx module; OvenMediaEngine — WHIP/WHEP SFU, sub-second latency; WHIP/WHEP (RFC 9725) as the standard HTTP signaling for WebRTC ingest/egress
- **Integrations**: Ch57 (FFmpeg) covers the library-level muxer/demuxer API for all protocols here; Ch58 (GStreamer) implements `hlssink2`, `dashsink`, `webrtcbin`, `srtsrc`/`srtsink`, `whipsink`; Ch59 (DeepStream) ingests RTSP/RTMP camera streams; Ch60 (codecs) — GOP structure, keyframe intervals, and rate control modes determine segment sizes that ABR algorithms reason about; Ch26 (VA-API) and Ch50 (Vulkan Video) provide the GPU decode endpoint for delivered streams; Ch38 (PipeWire) feeds screen-capture streams into GStreamer WebRTC pipelines

---

## Part XIV — Khronos Extended Ecosystem

Parts IV–VII covered the core Khronos APIs (Vulkan, EGL, OpenGL ES, OpenCL, SPIR-V, OpenXR). This part addresses the broader Khronos standards ecosystem: the SPIR-V toolchain consolidated from scattered chapters into a single authoritative reference; portable heterogeneous compute via SYCL 2020; GPU texture compression and the KTX2 container format; the glTF 2.0 runtime 3D asset standard; and the Vulkan Safety Critical profile for automotive and avionics deployments alongside the OpenVX vision graph API.

**Target audience**: Graphics application developers building asset pipelines and portable compute; embedded and safety-critical systems engineers; advanced readers who want to understand the full Khronos landscape beyond core Vulkan.

### Chapter 61: SPIR-V Ecosystem in Depth

- SPIR-V module structure: file magic (`0x07230203`), version word, generator magic, bound; instruction word encoding (opcode + word count); type system (scalar, vector, matrix, image, sampler, pointer, struct); ID namespace and forward references
- Capabilities and extension mechanism: `OpCapability` declarations; `OpExtension` strings; `OpExtInstImport` for extended instruction sets; the GLSL.std.450 set (dot product, cross, normalize, pow, etc.); the NonSemantic extended instruction sets (NonSemantic.Shader.DebugInfo.100, NonSemantic.DebugPrintf)
- SPIR-V front ends: GLSL → SPIR-V via `glslang`; HLSL → SPIR-V via `DXC` (DirectXShaderCompiler); WGSL → SPIR-V via `Tint` (Ch35) and `naga` (Ch40); OpenCL C → SPIR-V via `clang` + the OpenCL SPIR-V environment; SPIR-V → NIR in Mesa (Ch14): capability mapping and lowering strategy
- SPIRV-Tools toolkit in depth: `spirv-as` (text assembly → binary); `spirv-dis` (binary → text disassembly, human-readable); `spirv-val` (specification validator — cross-references SPIR-V spec section numbers in errors); `spirv-opt` (multi-pass optimiser pipeline — `--eliminate-dead-code-aggressive`, `--inline-entry-points-exhaustive`, `--loop-unroll`, `--convert-local-multi-store`); `spirv-link` (linking separately compiled modules, SPIR-V `Linkage` capability)
- SPIRV-Cross: SPIR-V → GLSL/HLSL/MSL/JSON cross-compilation; the reflection API for descriptor binding introspection; usage in ANGLE (Ch34), DXVK (Ch28), MoltenVK; the `spirv_cross::Compiler::build_combined_image_samplers` for GLSL combined-image-sampler lowering
- `spirv-opt` pass analysis: what each pass does, when it helps vs. hurts (inlining vs. compile-time explosion); interaction with Mesa NIR lowering (Ch14) — passes that are redundant because NIR does equivalent work; benchmarking `spirv-opt` preprocessing vs. ACO compile time (Ch15)
- SPIR-V debugging: `NonSemantic.Shader.DebugInfo.100` (`DebugSource`, `DebugLine`, `DebugLocalVariable`); `VK_EXT_debug_utils` + `OpLine` for source correlation in RenderDoc (Ch30); the `--generate-debug-info` glslang flag; `spirv-val` error taxonomy (ID out-of-bound, unsupported capability, missing decoration)
- Shader compilation pipeline performance: offline SPIR-V compilation (glslang/DXC at build time) vs. online (runtime GLSL); `VkPipelineCache` + SPIR-V module caching interplay (Ch12, Ch16); Vulkan pipeline library (`VK_EXT_graphics_pipeline_library`) for partial SPIR-V reuse; compile-time vs. runtime quality trade-offs
- **Integrations**: SPIR-V is the lingua franca between all front ends (GLSL/HLSL/WGSL/OpenCL C, Chs 14, 35, 40) and all Mesa driver back ends (ACO, LLVM, Chs 14, 15); SPIRV-Cross is the cross-compilation backbone for ANGLE (Ch34) and DXVK (Ch28); `spirv-val` is the first diagnostic step before CTS runs (Ch31); NonSemantic debug extensions enable source-level shader debugging in RenderDoc (Ch30); `spirv-opt` is used by Dawn (Ch35) and ANGLE (Ch34) as a preprocessing step before submission to Mesa NIR

### Chapter 62: SYCL 2020 and Portable Heterogeneous Compute

- SYCL 2020 programming model: `sycl::queue`, `sycl::buffer`, `sycl::accessor`, `sycl::event`; the host–device memory ownership model; task graph construction via command groups; in-order vs. out-of-order queues
- Kernel dispatch: `queue.parallel_for<class K>(sycl::range, handler)` and `queue.submit` + command group; `sycl::nd_range` and work-group dimensions; local memory via `sycl::local_accessor`; `sycl::group_barrier` and group algorithms (`sycl::joint_reduce`, `sycl::group_broadcast`)
- Unified Shared Memory (USM): `sycl::malloc_device`, `sycl::malloc_host`, `sycl::malloc_shared`; explicit vs. implicit data migration; USM vs. buffer/accessor ownership semantics; page migration overhead on discrete GPUs
- Intel oneAPI DPC++: LLVM-based SYCL 2020 compiler (`icpx`); targets Intel GPUs via Level Zero (Ch25), NVIDIA CUDA and AMD HIP via offline compiler plugins; how DPC++ SYCL kernels traverse the IGC (Intel Graphics Compiler) to GEN ISA on Xe/Arc (Ch5)
- AdaptiveCpp (formerly hipSYCL): single-pass compiler architecture; generic LLVM JIT as the default backend-agnostic flow; targets AMD HIP (AMDGPU LLVM backend, Ch48), NVIDIA CUDA, Level Zero, and OpenCL; native interoperability with HIP/CUDA intrinsics via `__acpp_backend_builtin`; performance vs. oneAPI DPC++ on AMD and NVIDIA hardware
- SYCL 2020 and Vulkan/OpenCL interop: mapping `sycl::buffer` to `VkBuffer` via `cl_khr_external_memory_opaque_fd`; latency cost of cross-API synchronisation; the `cl_khr_external_semaphore` bridge; when to prefer native Vulkan compute (Ch25) vs. SYCL portability
- Specialisation constants and backend-specific paths: `sycl::specialization_id`; `if constexpr` + `__SYCL_DEVICE_ONLY__` for backend dispatch; `sycl::kernel_bundle` for ahead-of-time kernel compilation; SYCL-Bench 2020 as the portable benchmark suite
- Migration from CUDA to SYCL: Intel `dpct` (CUDA → SYCL migration tool); semantic gaps — warp shuffle (`__shfl_sync` → `sycl::sub_group::shuffle`), cooperative groups, CUDA streams → SYCL queues; productivity cost vs. portability gain; when migration is worthwhile
- **Integrations**: SYCL compiles through the LLVM AMDGPU backend for AMD (same as ROCm/HIP, Ch48) and Intel IGC for Intel (same as Level Zero, Ch25); AdaptiveCpp on AMD targets the amdgpu compute queue (Ch5); Blender Cycles' oneAPI backend (Ch42) is a SYCL consumer of Level Zero; DMA-BUF interop between SYCL buffers and Vulkan images follows the same kernel mechanism as CUDA–Vulkan interop (Ch25); SYCL 2020 portability extends to the ARM Mali compute path via OpenCL backend, connecting to Ch6 drivers

### Chapter 63: KTX2, Basis Universal, and GPU Texture Compression

- Why GPU texture compression: memory bandwidth reduction (4:1 for BC1, 8:1 for BC7 on RGBA); storage footprint; fixed-rate decode in hardware (no variable-length codes); the energy cost of texture fetch vs. texture-compression-unit decode
- Block compression taxonomy on Linux — hardware support matrix:
  - BC1–BC7 (S3TC/BPTC family): `VK_FORMAT_BC*`; support on AMD RDNA (radeonsi/RADV, Ch18/19), Intel Xe (iris/ANV), NVIDIA NVK; BC7 6-mode analysis
  - ASTC LDR/HDR: `VK_FORMAT_ASTC_*`; ARM Mali target (Panfrost/Panthor, Chs 6, 19); 2D/3D/HDR profiles; the ASTC encoder (`astcenc`, fast/medium/thorough quality levels)
  - ETC1/ETC2: legacy mobile path; `VK_FORMAT_ETC2_*`; supported on Mali and many Android GPUs; `etc2comp` encoder
  - Querying support: `vkGetPhysicalDeviceFormatProperties`, `VK_FORMAT_FEATURE_SAMPLED_IMAGE_BIT`; fallback transcoding at runtime
- Basis Universal supercompression: ETC1S encoding pipeline (palettized block colours, global codebook); UASTC (universal ASTC-compatible block encoding at higher quality); the `basisu` CLI encoder (`-uastc`, `-uastc_level 0–4`); runtime transcoding via `basist::basisu_transcoder` C++ API; decode throughput comparison CPU vs. GPU transcoder
- KTX2 container format: file magic (`0xAB4B5458`); header fields (`vkFormat`, `typeSize`, `pixelWidth/Height/Depth`, `levelCount`); DFD (Data Format Descriptor) for colour space and channel layout; `KTXsupercompressionScheme` values (Zstandard, Basis-LZ, UASTC); level index table; `ktxTexture2` C API — `ktxTexture2_CreateFromNamedFile`, `ktxTexture2_TranscodeBasis`, `ktxTexture2_VkUploadEx`
- Vulkan upload path: `ktxTexture2_VkUploadEx` helper walking the mip pyramid; `VkImage` creation from KTX2 metadata; staging buffer, `vkCmdCopyBufferToImage` per mip level; DRM format modifier negotiation for tiled layouts (Ch4); optimal tiling vs. linear tiling selection
- glTF 2.0 + KTX2 integration: the `KHR_texture_basisu` extension; how tinygltf and cgltf discover KTX2 texture URIs; the loader image hook for transcoding before Vulkan upload; full pipeline: `.glb` parse → KTX2 extract → `TranscodeBasis` → Vulkan staging → `VkImage`
- Compression quality trade-offs: BC7 vs. UASTC on photographic content (PSNR comparison); BC1 vs. ETC1S on colour-sparse UI textures; ASTC 4×4 vs. 8×8 for normal maps; the ETC1S global codebook size effect on atlas quality; toktx tool for KTX2 authoring from PNG/EXR
- **Integrations**: KTX2 textures upload via the same Vulkan `VkImage` path described in Ch24; format support queries map to DRM format capability queries (Ch4); ASTC is the target format on ARM Mali drivers (Chs 6, 19); glTF+KTX2 assets feed Bevy (Ch40), Godot (Ch41), and Blender import (Ch42); the `KHR_texture_basisu` extension connects this chapter to Ch64 (glTF)

### Chapter 64: glTF 2.0 — The 3D Asset Pipeline Standard

- glTF 2.0 structure (ISO/IEC 12113:2022): the JSON manifest and binary `.glb` envelope; asset, scene, nodes, meshes, cameras, skins, animations, materials, textures, images, samplers, buffers, bufferViews, accessors — the JSON→GPU memory hierarchy
- Mesh data layout: accessor `componentType` (UNSIGNED_BYTE, UNSIGNED_SHORT, FLOAT) and `type` (SCALAR, VEC2, VEC3, VEC4, MAT4); interleaved vs. separate-attribute buffer layout; accessor `byteStride` and how it maps to `VkVertexInputBindingDescription.stride`; index buffer accessor and `VK_INDEX_TYPE_*`
- PBR Metallic-Roughness material model: base colour factor + texture; metallic-roughness texture (G = roughness, B = metallic); normal map with scale; occlusion map; emissive map with factor; alpha mode (OPAQUE, MASK with `alphaCutoff`, BLEND); `doubleSided` flag; how these map to Vulkan descriptor sets and fragment shader uniforms
- Official Khronos extensions: `KHR_draco_mesh_compression` (Draco geometry compression — decode before upload); `KHR_mesh_quantization` (16-bit/8-bit vertex compression, reduces buffer size 50–70%); `KHR_texture_transform` (UV scale/rotation/offset for atlas animation); `KHR_materials_transmission` (glass/liquid energy-conserving BSDF); `KHR_materials_volume` (extinction coefficient, thickness); `KHR_materials_unlit` (flat shading for UI, baked lighting); `EXT_meshopt_compression` (meshoptimizer vertex cache / overdraw optimisation)
- Loading with tinygltf: `TinyGLTF::LoadASCIIFromFile` / `LoadBinaryFromFile`; `Model`, `Scene`, `Node`, `Mesh`, `Primitive`, `Accessor` structs; accessor data extraction via `model.buffers[view.buffer].data[view.byteOffset]`; image loading hook for custom KTX2 decoder (Ch63)
- Loading with cgltf: single-header C99 (`cgltf.h`); `cgltf_parse`, `cgltf_load_buffers`; zero-copy pointer-into-mapped-buffer accessor access; `cgltf_accessor_read_float`; comparison with tinygltf API ergonomics for embedded/real-time use cases
- glTF on the Vulkan upload path: vertex buffer `VkBuffer` from accessor binary; index buffer; staging upload via `VkCommandBuffer`; descriptor set binding for PBR maps; the `VkPipeline` specialisation strategy for opaque vs. alpha-blend materials
- glTF in engines: Bevy's `GltfAssetPlugin` and `SceneLoader` (Ch40); Godot's built-in glTF importer and `GLTFDocument` API (Ch41); Blender's glTF import/export addon and KHR_materials mapping (Ch42); `gltf-transform` CLI for pre-processing (deduplicate, draco compress, resize textures)
- glTF 3.0 roadmap: procedural materials via MaterialX; physics and audio metadata extensions; spatial audio; status of draft extensions as of 2026
- **Integrations**: glTF vertex/index buffers upload via Vulkan buffer paths (Ch24); PBR textures use KTX2/Basis Universal (Ch63); Bevy (Ch40) and Godot (Ch41) use glTF as their primary 3D asset format; Blender (Ch42) exports to glTF for runtime delivery; Draco geometry codec connects to the block compression and DCT concepts of Ch60

### Chapter 65: Vulkan Safety Critical and OpenVX

- Vulkan SC 1.0 design rationale: the functional-safety imperative in automotive (ISO 26262 ASIL D), avionics (DO-178C Level A), and medical devices; why standard Vulkan's runtime flexibility (implicit memory allocation, on-the-fly pipeline compilation, undefined behaviour on misuse) is incompatible with safety certification; the `VK_SC_10_*` feature set and the `vksc_core` symbol naming convention
- Key Vulkan SC restrictions: mandatory offline pipeline compilation — no `vkCreateComputePipelines` / `vkCreateGraphicsPipelines` at runtime; `VkDeviceObjectReservationCreateInfo` for pre-declared resource counts; static memory pools (no growth); removed features (sparse memory, descriptor template updates, shader module caching, `VK_NULL_HANDLE` aliasing); the offline pipeline cache format as the build artefact
- The offline toolchain: `spirv-val` + `spirv-opt` (Ch61) for offline SPIR-V validation; `vksc-offline-compiler` (vendor tool) for pipeline plan generation; `vkPipelineCacheCreateInfo` with pre-built data embedded at device creation; how offline compilation shifts certification burden to the build system rather than the driver
- Fault handling and robustness: `VK_EXT_device_fault` (Vulkan SC mandatory); `pfnDeviceFaultVendorBinaryHandler` for structured GPU crash reports; `VK_EXT_robustness2` for out-of-bounds memory access guarantees; `VK_EXT_pipeline_robustness` for per-pipeline bounds checks; comparison with standard Vulkan's `VK_ERROR_DEVICE_LOST`
- Vulkan SC on Linux: current driver and SoC vendor landscape (NXP i.MX 9, TI Jacinto, Qualcomm SA8195P); the Vulkan SC Conformance Test Suite (VkSC-CTS); open-source toolchain support via SPIRV-Tools and khronos_vulkan_sc validation layers; deployment in automotive Head Unit and ADAS display ECUs
- OpenVX 1.3 graph API: `vxCreateContext`, `vxCreateGraph`, `vxCreateVirtualImage`; node types — convolution, dilate/erode, Gaussian pyramid, optical flow (Lucas-Kanade), Harris corners, HOG, Canny; `vxVerifyGraph` for offline graph optimisation; the conformance test suite (`OpenVX-sample-impl` passing tests on Linux)
- OpenVX on Linux: the Khronos sample implementation; vendor optimised implementations on embedded GPUs (ARM Mali); OpenVX as a higher-level alternative to writing individual OpenCL/Vulkan compute dispatches for classical vision tasks; `PyVX` Python bindings for prototyping
- NNEF 1.0 and ANARI 1.0 brief coverage: NNEF as a portable neural network serialisation format (operations, weight encoding) complementary to ONNX; current Linux implementations; ANARI 1.0 (August 2023) as a scientific rendering API with OSPRay, VisRTX, Blender Cycles, and VTK/ParaView backends; `KhronosGroup/ANARI-SDK` on Linux
- **Integrations**: Vulkan SC's offline pipeline compilation uses SPIRV-Tools (Ch61); Vulkan SC conformance testing parallels standard Vulkan CTS (Ch31); OpenVX nodes on ARM SoCs interact with Mali GPU drivers (Ch6); NNEF inference can run via rusticl on Gallium pipe drivers (Ch25); ANARI's Blender Cycles backend (Ch42) and VisRTX backend (Ch67) are the main Linux integration points; Vulkan SC's fault-handling model connects to the GPU security and isolation discussion in Ch30

---

## Part XV — NVIDIA Proprietary Graphics Stack

Parts II–III covered the open NVIDIA kernel driver ecosystem (Nouveau, Nova, NVK). This part covers the proprietary NVIDIA SDK stack as a coherent system: the CUDA runtime and its streaming execution model, the OptiX 9 ray tracing framework, DLSS 4's transformer-based neural rendering architecture, and the Omniverse platform's USD-centric rendering pipeline. These are closed-source technologies but architecturally important on the Linux professional workstation, HPC, and AI-rendering stack; each exposes a well-defined API and is documentable from public SDK headers, technical blog posts, and academic publications. Content is scoped to what is verifiable from public sources — proprietary internals are not speculated upon.

**Target audience**: Graphics application developers and ML engineers who work on NVIDIA proprietary driver deployments on Linux; systems developers integrating CUDA, OptiX, or USD pipelines.

### Chapter 66: CUDA Runtime, Streams, and NVRTC

- CUDA driver vs. runtime API: `libcuda.so` (driver, versioned against kernel module, `cuInit` / `cuCtxCreate`) vs. `libcudart.so` (runtime, versioned against toolkit, `cudaInitDevice`); why mismatched versions cause `CUDA_ERROR_INVALID_PTX`; the driver API path for loading PTX/cubin without a full toolkit dependency via `cuModuleLoadData` + `cuKernelGetFunction`
- The CUDA execution model: CUDA streams as ordered GPU work queues; `cudaStreamCreate`, `cudaStreamSynchronize`, `cudaStreamWaitEvent`; default stream 0 and its implicit cross-stream serialisation; `cudaStreamCreateWithFlags(cudaStreamNonBlocking)` to break that barrier; stream priority via `cudaStreamCreateWithPriority`
- CUDA events and asynchronous timing: `cudaEventCreate`, `cudaEventRecord(event, stream)`, `cudaEventSynchronize`, `cudaEventElapsedTime`; events as GPU-side timestamps; `cudaEventCreateWithFlags(cudaEventDisableTiming)` for lightweight synchronisation without profiling overhead; pinned memory and `cudaMemcpyAsync` for overlap with computation
- Memory model: device memory (`cudaMalloc`/`cudaFree`), host-pinned (`cudaMallocHost`), managed/unified (`cudaMallocManaged`); explicit prefetch via `cudaMemPrefetchAsync(ptr, count, device, stream)`; `cudaMemAdvise` hints (`cudaMemAdviseSetPreferredLocation`, `cudaMemAdviseSetReadMostly`); the relationship between CUDA unified memory and NVIDIA's HMM (Heterogeneous Memory Management) kernel infrastructure (Ch5)
- NVRTC runtime compilation (NVRTC 13.3, CUDA 13.x): `nvrtcCreateProgram`, `nvrtcAddNameExpression`, `nvrtcCompileProgram`; C++23 support; `nvrtcGetPTX` → `cuModuleLoadData`; header-only deployment without a full toolkit (curated CUDA C++ headers); use in OptiX 9 Cooperative Vectors (Ch67) for inline neural network shaders
- CUDA graphs: `cudaGraphCreate`, `cudaGraphAddKernelNode`, `cudaGraphAddMemcpyNode`; capturing a stream: `cudaStreamBeginCapture` → work → `cudaStreamEndCapture`; `cudaGraphInstantiate` + `cudaGraphLaunch`; graph update without reinstantiation via `cudaGraphExecKernelNodeSetParams`; when graphs reduce CPU dispatch overhead for repetitive frame-loop workloads
- CUDA Multi-Process Service (MPS): sharing a CUDA GPU across multiple processes; `nvidia-cuda-mps-server`; time-sliced vs. Ampere MIG (Multi-Instance GPU) spatial partitioning; MPS in Kubernetes GPU node deployments (Ch55); limitations with interactive graphics workflows
- NVML monitoring API: `nvmlDeviceGetUtilizationRates` (GPU, memory bus %), `nvmlDeviceGetMemoryInfo` (used/free/total), `nvmlDeviceGetTemperature`, `nvmlDeviceGetClockInfo`, `nvmlDeviceGetPowerUsage`; integrating NVML into monitoring daemons; the underlying library for `nvidia-smi`; `nvml.h` header and `libnvidia-ml.so`
- CUDA on Linux: `dkms` and nvidia-open kernel module interaction (Ch9); CUDA stream priority as the user-space analogue of the DRM GPU scheduler priority (Ch4); `/proc/driver/nvidia/` and `/sys/kernel/debug/nvidia/` diagnostic interfaces
- **Integrations**: CUDA streams are the compute analogue of Vulkan queues (Ch25); CUDA–Vulkan interop (Ch25) uses the memory and synchronisation primitives described here; ROCm/HIP (Ch48) provides a portable alternative at the programming model level; CUDA unified memory uses the HMM kernel infrastructure shared with amdgpu SVM (Ch5); NVIDIA Container Toolkit (Ch55) controls which CUDA device files are exposed per container; OptiX (Ch67) and TensorRT/NGX (Ch68) are built on top of the CUDA runtime described here

### Chapter 67: OptiX 9 — NVIDIA's Ray Tracing Framework

- OptiX programming model: `OptixDeviceContext` (wraps a CUDA context); acceleration structures — BLAS (`OptixBuildInput` from triangle meshes or custom primitives) and IAS (instance acceleration structure); the shader binding table (SBT) and call-site–shader pairing; `optixLaunch` to dispatch a ray generation shader; OptiX 9.1 (January 2026) as current release requiring R590 driver+
- OptiX shader types: ray generation shader (launch entry point, one per pixel); intersection shader (custom primitive geometry test); any-hit shader (transparency, alpha test); closest-hit shader (shading at first surface hit); miss shader (background / environment map); callable shader (reusable subroutine callable from other shader types); mapping to Vulkan KHR ray tracing (Ch56) — same pipeline stages, different API
- Acceleration structure build: `OptixBuildInput` variants — triangle meshes (`OptixBuildInputTriangleArray`, vertex/index buffers as `CUdeviceptr`), custom primitives (AABB list), curves (hair/fur), instance arrays (IAS); `optixAccelComputeMemoryUsage`; `OPTIX_BUILD_FLAG_ALLOW_COMPACTION` + `optixAccelCompact` for 50–70% memory savings; `OPTIX_BUILD_OPERATION_UPDATE` for animated mesh BVH refitting
- NVRTC integration: shader programs as inline C++ strings → `nvrtcCreateProgram` (Ch66) → PTX → `optixModuleCreate`; `OptixPipelineCompileOptions`, `OptixProgramGroupDesc`; `optixPipelineLinkOptions.maxTraceDepth`; the shader compilation: `.cu` source → NVRTC → PTX → `OptixModule` → linked `OptixPipeline`
- OptiX 9 Cooperative Vectors: embedding small neural networks inside any OptiX shader; `optixCoopVecMatMul` for weight-matrix multiplication; inference of multi-layer perceptrons inside any-hit or closest-hit shaders; use in real-time neural radiance field rendering and material NeRF; the `OPTIX_COMPILE_OPTIMIZATION_LEVEL` impact on neural shader compilation
- The Clusters (Megageometry) API (OptiX 9.0): building BVHs over massive dynamic meshes (tens of millions of triangles); cluster hierarchy build; connection to Blackwell hardware-accelerated linear curve primitives
- The OptiX denoiser: `OptixDenoiser` API; AI model weights shipped with the OptiX SDK; AOV inputs (albedo guide, normal guide, motion vectors) for guided denoising; temporal stability across frames; comparison with DLSS 4 ray reconstruction (Ch68) — the OptiX denoiser is lighter-weight and SDK-bundled; no proprietary driver dependency for the denoiser weights
- Vulkan interop with OptiX: exporting CUDA memory to Vulkan via `cudaExternalMemoryHandleType_OpaqueWin32` (Windows) / `cudaExternalMemoryHandleType_OpaqueFd` (Linux) + `VK_KHR_external_memory_fd` (Ch25); sharing Vulkan timeline semaphores with CUDA events; the full hybrid pipeline — Vulkan G-buffer rasterisation → CUDA/OptiX path tracing → Vulkan tonemap + present
- Blender Cycles and OptiX: `CYCLES_DEVICE=OPTIX`; OptiX kernel compilation from Cycles CUDA source at render startup; the OptiX denoiser in Blender's viewport and render output (Ch42); comparison with HIP-RT on AMD (Ch48)
- **Integrations**: OptiX BVH semantics are directly comparable to `VK_KHR_acceleration_structure` (Ch56); OptiX shaders compile via NVRTC (Ch66), the same path as Ch56's NVK ray tracing; Blender Cycles (Ch42) uses OptiX as its NVIDIA GPU path; Cooperative Vectors use the same transformer inference concept as DLSS 4 (Ch68); GPU memory is shared with Vulkan via the CUDA external memory mechanism (Ch25); RTX Remix (DXVK-based, open source) is the gaming equivalent of OptiX-powered scene remastering and connects to Ch28

### Chapter 68: DLSS 4, Neural Rendering, and Frame Generation

- DLSS architecture evolution: from spatial CNN super-resolution (DLSS 1.0, 2018) through temporal (DLSS 2, Turing, 2020; DLSS 3, Ada, 2022) to transformer-based (DLSS 4, Blackwell/Ada Lovelace, CES 2025); why vision transformers outperform CNNs for high-frequency detail reconstruction across full-image attention fields; DLSS 4.5 (June 2026) second-generation transformer model
- DLSS 4 super resolution internals: the vision transformer model; self-attention over the temporal history buffer (accumulation of N previous rendered frames); motion vector integration for temporal alignment; the loss function balancing PSNR and perceptual sharpness; ghosting and aliasing artefact suppression via per-pixel confidence estimation; input buffers (`color`, `depth`, `motionVectors`, `exposure`)
- Multi Frame Generation (MFG): generating 1–4 additional synthetic frames per rendered frame (`DLSS 4.5` dynamic MFG: 1–6×); the optical flow estimator (dense per-pixel motion field from consecutive rendered frames); the inpainting network for disoccluded region synthesis; the compositing pass blending generated and rendered frames; frame stability constraints (why >4× is unstable for fast motion)
- Ray Reconstruction: AI-powered denoiser replacing hand-tuned spatiotemporal denoisers; input: noisy radiance (typically 1–4 spp) + albedo + normals + motion vectors; output: clean radiance; trained on 5× more diverse ray-tracing scenarios than DLSS 3; integration with DXR via `VKD3D-Proton` (Ch28) and with VKD3D-Proton's `VK_NVX_raytracing_validation`
- The NGX SDK API: `NVSDK_NGX_VULKAN_Init(appId, appDir, vkInstance, vkPhysicalDevice, vkDevice)`; `NVSDK_NGX_VULKAN_CreateFeature1` for DLSS SR, frame generation, and ray reconstruction feature handles; `NVSDK_NGX_VULKAN_EvaluateFeature` dispatch; `NVSDK_NGX_Parameter` for quality preset, resolution, and jitter; required Vulkan extensions: `VK_NV_optical_flow`, `VK_NV_low_latency2`, `VK_KHR_timeline_semaphore`
- NVIDIA Reflex and LatencyFleX: Reflex's `NvAPI_D3D_SetSleepMode` + `NvAPI_D3D_Sleep` for input-to-photon latency reduction by throttling CPU render-ahead; the Linux open alternative LatencyFleX (Ch29) implementing the same present-queue feedback loop without NvAPI; MangoHud latency visualisation (Ch29)
- 3D Gaussian Splatting and neural scene representations: Gaussian Splatting as a differentiable rasteriser — 3D Gaussians with learned position/covariance/opacity/colour → sorted back-to-front → per-Gaussian tile-based splatting; CUDA-accelerated tile rasteriser (`gsplat` library, integrated with NVIDIA `3DGUT` April 2025); training via Adam optimiser on CUDA; inference (rendering) at 100–200+ FPS on RTX hardware; comparison with path tracing for high-fidelity novel view synthesis; the `3DGS → real-time game engine` integration challenge (no standard G-buffer, no occlusion, no shadow casting)
- OpenUSD and MaterialX in neural rendering: how USD scene description is consumed by DLSS/RTX-enabled renderers; MaterialX 1.39 as a procedural material description language embedded in USD; the `mdl_material` USD schema for NVIDIA MDL shaders; AOUSD Core Specification 1.0 (December 17, 2025, Alliance for OpenUSD, Linux Foundation)
- DLSS on Linux: proprietary driver or nvidia-open (R550+) requirement; NGX feature availability via `NVSDK_NGX_Result_FAIL_FeatureNotSupported` diagnostic; DLSS-to-FSR translation shims (DLSS2FSR, LatencyFleX) for AMD/Intel users on Wine; game compatibility via Proton-GE + ngxrun wrapper
- **Integrations**: DLSS 4 requires the nvidia-open or proprietary driver (Part III); Ray Reconstruction feeds the same DXR/VKD3D-Proton pipeline as standard Vulkan ray tracing (Chs 28, 56); the NGX SDK uses Vulkan timeline semaphores and external memory (Ch24, Ch25); LatencyFleX is the Linux open alternative (Ch29); Gaussian Splatting training runs on the CUDA compute model (Ch66); OptiX 9 Cooperative Vectors (Ch67) implement per-shader neural network inference at the same transformer-model conceptual level; the AOUSD/USD scene description is consumed by the RTX Renderer (Ch69)

### Chapter 69: NVIDIA Omniverse, OpenUSD, and the RTX Renderer

- OpenUSD architecture: the composition arc system (layer stacks, references, inherits, specialises, payload arcs, variantSets); `UsdStage::Open`, `UsdPrim`, `UsdAttribute`, `UsdRelationship`; the `SdfLayer` authoring model and sublayer composition; schema classes — `UsdGeomMesh`, `UsdGeomCamera`, `UsdLuxSphereLight`, `UsdShadeMaterial`, `UsdPhysicsRigidBodyAPI`; writing USD in C++ with the USD SDK 25.02
- AOUSD Core Specification 1.0 (December 17, 2025): formal standardisation under the Alliance for OpenUSD (Linux Foundation); schema versioning guarantees; conformance test implications; what changes from Pixar's USD 24.x to AOUSD 1.0 aligned tooling
- Hydra rendering delegate architecture: `HdRenderDelegate` as the abstraction between USD scene description and rendering backends; `HdStorm` (OpenGL, production-stable); `HdPrman` (Pixar RenderMan); the RTX Renderer delegate (`omni.rtx.renderer`); implementing a custom `HdRenderDelegate` for Vulkan ray tracing; `HdEngine::Execute` → render delegate → GPU command submission
- Omniverse RTX Renderer modes: RTX Real-Time 2.0 (path tracing + DLSS neural features for interactive speed); RTX Interactive (progressive path tracing without DLSS for final-quality offline); GPU architecture support matrix — Blackwell (full DLSS 4.5 + MFG), Ada (DLSS SR + RR), Ampere (DLSS SR + OptiX denoiser), Turing (OptiX denoiser only); Project Zorah case study — full-stack RTX path tracing showcase (GTC 2023+): MDL OmniHair (Marschner R/TT/TRT) for strand hair, OmniSurface for armour and skin (SSS, iridescent coating), PhysX 5 cloth simulation via USDRT Fabric Scene Delegate, DLSS 3 Frame Generation yielding effective 4K/60fps; Blackwell LSS curves for hair; headless per-frame cloud rendering via Kit `--no-window`
- Slang shader language: NVIDIA's extension of HLSL with generics, interfaces, automatic differentiation (`[Differentiable]` attribute for gradient-based rendering), and cooperative vector support; how Slang shaders compile to PTX via `slangc`; the Falcor rendering framework (built on Slang) as a research baseline; comparison with GLSL/HLSL in the standard Mesa pipeline
- Multi-GPU rendering in Omniverse: up to 16 GPUs via NCCL-style work partitioning; NVLink for high-bandwidth inter-GPU buffer transfers (analogous to the p2p DMA described in Ch4, Ch49); frame subdivision vs. spectral subdivision strategies; `omni.gpu_foundation` multi-device context management
- Kit SDK architecture: extension-based Python+C++ application framework; `omni.kit.app` event loop; `omni.rtx.renderer` extension wiring RTX delegate into Kit's `UsdContext`; Python USD API (`pxr.Usd`, `pxr.UsdGeom`, `pxr.UsdShade`); `omniverse-launcher` and headless Kit for server-side rendering on Linux
- On-Linux deployment: Ubuntu 22.04/24.04 support; driver requirements (Ada Lovelace+ for full feature set); headless RTX rendering with `Kit` + virtual display (`Xvfb` or DRM render-only node); container support (NVIDIA Container Toolkit, Ch55); profiling with Nsight Graphics via remote capture
- USD for Linux graphics pipelines beyond Omniverse: Blender 4.x USD import/export addon (materials via BSDF → UsdPreviewSurface mapping); Godot USD import (in development); `usdview` CLI for asset inspection; `usd-cli` (Remedy Entertainment's open tool for USD pipeline automation)
- **Integrations**: the RTX Renderer uses DLSS 4 (Ch68) and OptiX denoiser (Ch67); multi-GPU transfers use p2p DMA (Chs 4, 49); the Slang shader compilation pipeline produces PTX via the same NVRTC infrastructure (Ch66); Blender (Ch42) and Godot (Ch41) consume USD assets; the Hydra delegate abstraction provides a Vulkan-based open alternative via HdStorm (Ch18, Ch24); NCCL collective communication over NVLink parallels the ROCm RCCL path (Ch48); headless container deployment uses the same NVIDIA Container Toolkit (Ch55)

### Chapter 70: RTX Kit — RTXDI, RTXGI, NRD, RTXNS, and RTXNTC

- RTX Kit overview: five open-source MIT-licensed Vulkan/Linux SDKs (github.com/NVIDIA-RTX) bridging raw `VK_KHR_ray_tracing` (Ch56) and closed NGX/DLSS (Ch68); pipeline ordering — RTXGI → RTXDI → RTXNS/RTXNTC → NRD → DLSS
- **RTXDI v3.0** (ReSTIR DI + PT): reservoir data structure `R = (y, W, M)`; Weighted Reservoir Sampling (WRS); three-stage pipeline — initial RIS candidates (light BVH), temporal reuse with motion vectors, spatial reuse with MIS-weighted visibility; ReSTIR PT path-suffix reuse with reconnection shift mapping and Jacobian correction; RTXDI API — `RtxdiContext`, `RTXDI_SampleLightsForSurface`, `RTXDI_TemporalResampling`, `RTXDI_SpatialResampling` — plus HLSL/Slang companion includes; production references (Cyberpunk 2077 path tracing, Alan Wake 2)
- **RTXGI 2.0** (SHaRC + NRC): SHaRC (Spatially Hashed Radiance Cache) — GPU hash map with `uint64_t` spatial keys, `SharcGetCachedRadiance`/`SharcUpdateCache` shader API, ~1–3 ms/frame at 1080p; NRC (Neural Radiance Cache, Müller et al. SIGGRAPH 2021) — multi-resolution hash encoding + 4-layer MLP trained online each frame, ~4–8 ms; choosing between SHaRC (open world, fast convergence) and NRC (enclosed, high-quality indirect); Omniverse `omni.rtx.neuralradiancecache` extension
- **NRD v4.17** (NVIDIA Real-time Denoisers): three denoiser algorithms — REBLUR (adaptive-radius spatial blur with temporal accumulation), RELAX (recommended for ReSTIR-sampled inputs; separate diffuse/specular history buffers; robust to correlated samples), SIGMA (shadow denoising, 5×5 cross-bilateral); required G-buffer inputs: `IN_MV` (motion vectors), `IN_NORMAL_ROUGHNESS` (Oct-packed), `IN_VIEWZ` (linear depth), `IN_DIFF/SPEC_RADIANCE_HITDIST` (radiance + hit distance); pre-divided albedo requirement; `NrdIntegration` helper: `BeginFrame`, `SetCommonSettings`, `SetDenoiserSettings`, `Denoise`; Vulkan backend: 5-pass dispatch (pre, history-fix, blur, post-blur, split); ~1.5–3 ms/frame at 1080p
- **RTXNS v1.3** (Neural Shaders SDK): `VK_NV_cooperative_vector` Vulkan extension (`coopVecMatMulNV`, `coopVecActivation` GLSL ops mapping to tensor core MMA instructions); weight formats — FP16, E4M3 (FP8 for Ada+), INT8; `rtxns::NetworkLayout` for GPU weight buffer setup; Slang `NeuralNetwork<inputW, hiddenW, outputW, layers, Activation>` template → SPIR-V CoopVec; Linux driver ≥ 572.16 + Ada Lovelace+ requirement; `vulkaninfo | grep VK_NV_cooperative_vector` verification
- **RTXNTC v0.5** (Neural Texture Compression): MLP-encoded textures (4-layer × 32-neuron, ELU, FP8 weights) replacing BC7 at up to 7× VRAM savings; multi-resolution UV hash encoding input; offline encoding — `rtxntc-encode --onnx --quality --format E4M3`; runtime inference via `rtxntc::SampleTexture` (compiles to CoopVec SPIR-V); fallback scalar path for pre-Ada hardware; `.ntc` file format (weights + topology header)
- Full-frame pipeline integration: RTXGI indirect → RTXDI direct → RTXNS neural materials → NRD denoising → DLSS SR/MFG; combined RTX Kit overhead ~6 ms at 1080p on RTX 4080; build instructions on Linux (CMake + GCC/Clang + CUDA 12.x + Vulkan SDK 1.3.x)
- **Integrations**: Ch56 (Vulkan ray tracing) — RTXDI calls `vkCmdTraceRaysKHR` over the same `VkAccelerationStructureKHR`; Ch67 (OptiX) — OptiX denoiser is lighter NRD alternative; NRC shares multi-resolution hash encoding with Instant-NGP; Ch68 (DLSS) — NRD feeds DLSS SR; Ray Reconstruction replaces both; Ch69 (Omniverse) — RTX Interactive uses NRD + RTXDI; Project Zorah uses RTXGI NRC; Ch66 (CUDA) — NRC training + RTXNTC encoding are CUDA workloads; Ch61 (SPIR-V) — `cooperative_vector` capability not yet in spirv-val 1.6; use Vulkan layered validation instead

---

## Part XVI — Intel Open Graphics Stack

### Chapter 71: Intel Xe Kernel Driver, Arc GPU Architecture, and the Intel Open Stack

- Intel GPU history: Gen → Iris → Arc; the transition from i915 to xe.ko for discrete GPUs
- Xe kernel driver architecture: `xe_device`, `xe_gt`, `xe_tile`; LMEM vs GGTT; GMD (Graphics Micro-Driver) model; source `drivers/gpu/drm/xe/`
- GuC and HuC firmware: ring-less command submission via GuC; context scheduling; HuC media decode assistance; firmware loading vs i915
- Xe2/Battlemage architecture: Xe2-HPG GPU microarchitecture; Render Slice; L2$ per-DSS; XMX (Xe Matrix Extensions / tensor cores); hardware ray tracing; AV1 hardware encode (VDEnc)
- Intel ANV Vulkan driver: `src/intel/vulkan/`; BVH builder for ray tracing; sparse binding; ANV on xe vs i915
- Level Zero and oneAPI: `ze_driver_handle_t`, `ze_context_handle_t`, `ze_command_list_handle_t`; relationship to OpenCL; oneAPI DPC++ compiler (intel/llvm fork)
- Intel Media Driver: `libva-intel-media-driver` (iHD); MDF/CM kernel architecture; VPP pipeline; AV1 encode path; VA-API interface
- XeSS: Intel Xe Super Sampling; temporal upscaling using XMX on Arc; DP4a fallback; `xess_context_handle_t`, quality modes
- **Integrations**: Ch1, Ch4, Ch5, Ch14 (BRW compiler), Ch16, Ch18, Ch24, Ch26, Ch75 (explicit sync), Ch76, Ch77

---

## Part XVII — AMD Developer Ecosystem

### Chapter 72: AMD FidelityFX SDK and Radeon Developer Tools

- AMD developer ecosystem: FidelityFX SDK, AMF, and Radeon tools on GPUOpen (gpuopen.com); all open source
- FidelityFX SDK architecture: backend abstraction (Vulkan, DX12, native); `ffx_interface_t`; `src/components/` structure; https://github.com/GPUOpen-LibrariesAndSDKs/FidelityFX-SDK
- FSR 4: neural upscaling using XMX on RDNA 4; motion vector + depth inputs; temporal accumulation; RCAS sharpening; `FfxFsr3ContextDescription`, `ffxFsr3ContextCreate`, `ffxFsr3ContextDispatch`
- Other FidelityFX effects: CAS (Contrast Adaptive Sharpening), SPD (Single Pass Downsampler), SSSR (Stochastic Screen Space Reflections), Brixelizer GI (sparse voxel GI)
- AMF — Advanced Media Framework: `AMFFactory`, `AMFComponent` for VCE/VCN; `amf_surface_t`; Linux VA-API backend; https://github.com/GPUOpen-LibrariesAndSDKs/AMF
- Radeon GPU Profiler (RGP): frame-level GPU timeline; SQTT (Shader Queue Thread Trace); instruction-level timing; `VK_AMD_gpa_interface`; `.rgp` file format
- Radeon Memory Visualizer (RMV): heap allocation timeline; VRAM fragmentation; resource lifetime tracking; AMD Radeon Developer Service
- RenderDoc: AMD-led cross-vendor open source frame debugger; in-process capture layer; serialised RDC format; Python API; https://github.com/baldurk/renderdoc
- **Integrations**: Ch5, Ch15, Ch18, Ch25, Ch26, Ch29, Ch30, Ch48, Ch56, Ch76

---

## Part XVIII — Rendering Abstraction Libraries

### Chapter 81: SDL3 GPU API: A Portable High-Level GPU Abstraction

- Why SDL3 added SDL_GPU: gap between raw Vulkan and full engines; explicit command buffers without pipeline state object verbosity; released 2024
- Device creation: `SDL_CreateGPUDevice` with `SDL_GPUShaderFormat` flags; `SDL_ClaimWindowForGPUDevice`; Vulkan always selected on Linux
- Shaders and pipeline state: `SDL_CreateGPUShader` from SPIR-V; `SDL_GPUGraphicsPipelineCreateInfo`; vertex attributes, rasterizer, depth/stencil, blend
- Textures and samplers: `SDL_CreateGPUTexture`; usage flags; `SDL_CreateGPUSampler`; upload via `SDL_GPUCopyPass`
- Buffers: `SDL_CreateGPUBuffer`; `SDL_GPUTransferBuffer` staging pattern
- Command buffers and render passes: `SDL_AcquireGPUCommandBuffer`, `SDL_BeginGPURenderPass`, `SDL_BindGPUGraphicsPipeline`, `SDL_DrawGPUPrimitives`, `SDL_SubmitGPUCommandBuffer`
- Compute passes: `SDL_BeginGPUComputePass`, `SDL_DispatchGPUCompute`
- Swapchain and HDR: `SDL_AcquireGPUSwapchainTexture`; `SDL_GPUSwapchainComposition` (SDR, HDR10_ST2084); present modes
- SDL_GPU vs Vulkan vs WebGPU: comparison table; sweet spot for indie games and tools
- **Integrations**: Ch16, Ch18, Ch24, Ch40, Ch61, Ch76, Ch82, Ch84

### Chapter 82: Vulkan Ecosystem Toolkit: VMA, volk, vk-bootstrap, and Friends

- The bootstrapping problem: hundreds of lines before a triangle; ecosystem of helpers that eliminate boilerplate without hiding the GPU model
- VMA (Vulkan Memory Allocator): `VmaAllocator`; `VMA_MEMORY_USAGE_AUTO`; `vmaCreateBuffer`/`vmaCreateImage`; suballocation (256 MB default blocks); staging pattern; defragmentation; `VmaPool`; https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator
- VMA internals: memory type selection; `VmaVirtualBlock`; budget tracking; `VMA_MEMORY_PROPERTY_LAZILY_ALLOCATED_BIT` for mobile TBDR
- volk: runtime Vulkan function pointer loading; `volkInitialize()`, `volkLoadDevice()`; eliminates loader trampoline overhead; https://github.com/zeux/volk
- vk-bootstrap: `vkb::InstanceBuilder`, `vkb::PhysicalDeviceSelector`, `vkb::DeviceBuilder`, `vkb::SwapchainBuilder`; 30-line device setup; https://github.com/charles-lunarg/vk-bootstrap
- SPIRV-Reflect: `SpvReflectShaderModule`; automatic `VkDescriptorSetLayout` generation from SPIR-V; https://github.com/KhronosGroup/SPIRV-Reflect
- Dear ImGui Vulkan backend: `ImGui_ImplVulkan_Init`, `ImGui_ImplVulkan_RenderDrawData`; descriptor pool management
- Shader hot-reload: inotify + glslang/DXC at runtime; pipeline recreation dance; shaderc https://github.com/google/shaderc
- Tracy GPU profiler: `TracyVkContext`, GPU timestamp zones, `VkPhysicalDeviceLimits::timestampPeriod`; zero-overhead in release; https://github.com/wolfpld/tracy
- Minimal Vulkan application stack: vk-bootstrap + VMA + volk + SPIRV-Reflect + imgui combined skeleton (~200 lines vs 2000 raw)
- **Integrations**: Ch16, Ch18, Ch24, Ch28, Ch40, Ch61, Ch70, Ch76, Ch77, Ch81

### Chapter 83: Filament: Google's Physically Based Rendering Engine on Linux

- Filament's place: Google's open-source PBR library (not a game engine); production use in Android, Chrome OS, Google Maps 3D, ARCore; C++ with Vulkan backend on Linux; https://github.com/google/filament
- Three-layer architecture: Engine (resource management, backend), Scene (ECS — Entity/TransformManager/LightManager/RenderableManager), Renderer (frame submission); `filament::backend::Driver` abstraction
- Engine init and Vulkan backend: `Engine::create(Backend::VULKAN)`; `SwapChain` from native window; `VulkanPlatform` in `src/backend/vulkan/`
- ECS scene setup: `utils::Entity`; `TransformManager::setTransform`; `LightManager::Builder`; `RenderableManager::Builder` with geometry + material
- Vertex and index buffers: `VertexBuffer::Builder` attribute declarations; `BufferDescriptor`; `IndexBuffer` with UINT16/UINT32
- FILAMAT material system: `.mat` file compiled by `matc`; `shadingModel`, `blending`, parameters; PBR model (baseColor, metallic, roughness, reflectance, emissive, normal, AO); `MaterialInstance` for per-object override
- PBR implementation: Cook-Torrance BRDF, GGX NDF, Smith G, Schlick Fresnel; IBL from KTX2 cubemap; directional/point/spot lights; VSM and DPCF shadow maps
- FrameGraph: `src/fg/`; resource declaration (`FrameGraphTexture`), pass setup+execute lambdas; automatic aliasing, barrier insertion, pass culling
- Post-processing: tone mapping (ACES, Filmic), bloom (dual kawase), DoF, FXAA/TAA — all FrameGraph passes; `View::setBloomOptions`
- Headless rendering: `SwapChain::CONFIG_HEADLESS`; `Renderer::readPixels`; server-side asset rendering and CI screenshot testing
- matdbg and RenderDoc integration: WebSocket live material editor; RenderDoc capturable app registration
- **Integrations**: Ch16, Ch18, Ch24, Ch37, Ch40, Ch41, Ch42, Ch61, Ch63, Ch64, Ch76, Ch82

### Chapter 84: bgfx, Cross-Platform Rendering Abstractions, and the Frame Graph Pattern

- Abstraction spectrum: raw Vulkan → VMA+helpers → bgfx → SDL3 GPU → Filament → Bevy/Godot; when each level is appropriate
- bgfx architecture: `bgfx::init()` selects backend (Vulkan on Linux); frame model (draw calls between `bgfx::frame()` calls); `bgfx::Encoder` for multi-threaded submission; https://github.com/bkaradzic/bgfx
- bgfx resources: `VertexLayout` declarations; typed opaque handles (`VertexBufferHandle`, `TextureHandle`); `createDynamicVertexBuffer`; format codes (BGFX_TEXTURE_FORMAT_*)
- bgfx shader system: bgfx superset of GLSL compiled by `shaderc` tool; `bgfx::createUniform`; SPIR-V path for Vulkan backend
- View and render target system: `bgfx::setViewFrameBuffer`; view ordering (shadow=0, geometry=1, post=2); `bgfx::setViewMode`
- Multi-threaded encoding: `bgfx::Encoder`; per-thread recording, merged during `bgfx::frame()`; compare to Vulkan secondary command buffers
- Frame graph pattern: declarative frame description with explicit resource dependencies; automatic resource lifetime, memory aliasing, barrier insertion, topological sort; implementations in Filament, Bevy, WebRender, Unreal RDG
- Frame graph implementation: builder pattern with resource virtualisation; aliasing between non-overlapping passes shares `VkDeviceMemory` via VMA; 100-line simplified implementation outline
- Transient resource allocators: ring buffer of pre-allocated `VkImage`/`VkBuffer`; `VMA_POOL_CREATE_LINEAR_ALGORITHM_BIT`; `VmaVirtualBlock` for offset-only management
- Other notable abstractions: Diligent Engine (`IRenderDevice`, `IDeviceContext`); The Forge (cross-platform, shipped games); LLGL (thin C++ abstraction); Magnum (scientific visualisation)
- Decision guide: raw Vulkan vs VMA vs bgfx vs SDL3 GPU vs Filament vs Bevy/Godot
- **Integrations**: Ch16, Ch18, Ch24, Ch28, Ch40, Ch41, Ch52, Ch61, Ch77, Ch81, Ch82, Ch83

---

## Part XIX — Android Graphics

### Chapter 85: Android Compositor: SurfaceFlinger, HardwareBuffer, and the Buffer Pipeline

- Android graphics architecture: Linux kernel (DRM/KMS, DMA-BUF) → HAL layer (Gralloc, HWComposer, EGL) → framework (SurfaceFlinger, BufferQueue, HWUI) → application (Canvas/Skia, Vulkan, GLES)
- Gralloc: Android's GPU memory allocator HAL; Gralloc4 (AIDL-based, Android 11+); `native_handle_t`; usage flags (HW_RENDER, HW_TEXTURE, HW_COMPOSER); ION/DMA-BUF backing; compare to GBM
- AHardwareBuffer (NDK, API 26+): `AHardwareBuffer_allocate` with `AHardwareBuffer_Desc`; `AHardwareBuffer_lock`/`unlock`; cross-process sharing via Unix socket; EGL import via `eglGetNativeClientBufferANDROID`; Vulkan import via `VkAndroidHardwareBufferPropertiesANDROID`
- BufferQueue: `IGraphicBufferProducer`/`IGraphicBufferConsumer`; slot model (up to 64 slots); dequeue→render→queue→acquire→release cycle; double/triple buffering; `frameworks/native/libs/gui/BufferQueue.cpp`
- SurfaceFlinger: Android's Wayland compositor equivalent; Layer model (BufferLayer, ColorLayer, etc.); composition loop: acquireBuffer → HWComposer::prepare → GPU fallback or direct HW composition → present fence
- HWComposer and direct scanout: HWC2 API; `setLayerBuffer`, `validateDisplay`, `presentDisplay`; layer composition decision; compare to DRM atomic commit
- ASurfaceControl (API 29+): `ASurfaceTransaction_create`; `ASurfaceTransaction_setBuffer`, `setPosition`, `setAlpha`, `setZOrder`; `ASurfaceTransaction_apply`; compare to `wl_surface.commit`
- HWUI: Android's UI rendering engine; RenderThread with Skia/Vulkan pipeline; DisplayList recording; `SkiaOpenGLPipeline` vs `SkiaPipelineVulkan`; RenderNode tree mirrors View hierarchy
- Sync fences in Android: `android::Fence` wraps DMA-BUF sync file; acquire/release fences per BufferQueue slot; `SyncFileInfo`; compare to `linux-drm-syncobj-v1`
- Frame end-to-end: Canvas.drawRect → HWUI → Vulkan/GLES commands → GraphicBuffer → SurfaceFlinger → HWC → DRM atomic → scanout
- Color management and HDR: wide color gamut (P3, Android 8+); HDR support (Android 10+); `AHARDWAREBUFFER_FORMAT_R16G16B16A16_FLOAT`; per-layer dataspace; SurfaceFlinger tone mapping
- **Integrations**: Ch1, Ch2, Ch4, Ch20, Ch21, Ch22, Ch34, Ch37, Ch38, Ch74, Ch75, Ch86

### Chapter 86: Vulkan on Android: Drivers, ANGLE, and Mobile GPU Performance

- Android Vulkan requirements: Vulkan 1.0 required Android 7.0; hardware requirement Android 10; Vulkan 1.1 target Android 12+; Android loader `/system/lib64/libvulkan.so`; ICD at `/vendor/lib64/hw/vulkan.*.so`
- Android Vulkan loader: differs from Khronos desktop loader; layer loading from `/data/local/debug/vulkan/`; `android.hardware.vulkan` HIDL/AIDL HAL; `vkCreateInstance` ICD enumeration
- GPU vendor drivers: Qualcomm Adreno (Vulkan 1.3, Snapdragon 8 Gen 3); ARM Mali (Vulkan 1.1/1.2); Imagination PowerVR; no open source, shipped in `/vendor` partition
- Mesa on Android — Turnip and freedreno: open-source Adreno drivers shipping on Qualcomm reference platforms; `Android.bp` build rules; Turnip Vulkan 1.3 conformance on Adreno 6xx/7xx; `VK_DRIVER_ID_MESA_TURNIP`
- ANGLE on Android: default OpenGL ES on Pixel devices (Android 12+) via updatable APEX module; opt-in via developer options; `EGL_ANDROID_GLES_layers`; ANGLE-on-Vulkan performance vs vendor GLES
- AHardwareBuffer × Vulkan interop: `VK_ANDROID_external_memory_android_hardware_buffer`; `vkGetAndroidHardwareBufferPropertiesANDROID`; `VkImportAndroidHardwareBufferInfoANDROID`; camera YUV via `VkExternalFormatANDROID`; reverse export path
- Android-specific Vulkan extensions: `VK_KHR_android_surface` (`vkCreateAndroidSurfaceKHR`); `VK_ANDROID_external_format_resolve`; `VK_GOOGLE_display_timing`; `VK_KHR_present_wait`
- Shader compilation: SPIR-V required (no GLSL at driver); shaderc AAR; `shaderc_compile_into_spv`; Android Baseline Profile (ABP) on Android 14+; Vulkan validation layer from Google Play
- Memory management on mobile: unified memory (DEVICE_LOCAL | HOST_VISIBLE); `VK_MEMORY_PROPERTY_LAZILY_ALLOCATED_BIT` for MSAA on TBDR; VMA lazy allocation handling; optimal MSAA resolve pattern
- TBDR architecture: tile-based deferred rendering on Adreno/Mali/PowerVR vs desktop IMR; tile memory; `loadOp=CLEAR`/`storeOp=DONT_CARE` effectively free; `VK_QCOM_render_pass_transform`; why render passes still matter on mobile
- Android GPU performance tools: Snapdragon GPU Profiler; ARM Streamline/Performance Studio; Android GPU Inspector (AGI, open source); perfetto GPU counter track (Android 12+)
- Chrome on Android: ANGLE as Chrome GLES backend since Android 8; Dawn Vulkan backend for WebGPU; GPU process with AHardwareBuffer for cross-process textures; Viz compositor with ASurfaceControl
- **Integrations**: Ch4, Ch6, Ch16, Ch18, Ch24, Ch33, Ch34, Ch35, Ch50, Ch61, Ch76, Ch82, Ch85

---

### Section additions to existing chapters (cross-chapter content)

**Chapter 28 §9 — RTX Remix** (added): open-source game remastering toolkit (MIT, RTX Remix 1.0, March 2025); `dxvk-remix` fork of DXVK intercepting D3D8/9 → Vulkan + OptiX path tracing injection; USD asset overlay via `over` composition arc; DLSS 4 MFG and NRC in RTX Remix 1.0; Linux via Proton/Wine (`PROTON_NO_DXVK=1`, driver ≥ 535); Portal RTX and Half-Life 2 RTX as reference titles; connection to Ch28 DXVK (§3), dxvk-nvapi (§6), Ch67 (OptiX), Ch68 (DLSS), Ch70 (NRC)

**Chapter 30 §6.3 expansion — NvPerf SDK** (added): programmatic GPU counter collection via `libnvperf_host.so` + `libnvperf_target.so`; Linux permissions — `perf_event_paranoid ≤ 0` or `CAP_SYS_ADMIN`; `NvperfVulkanLoadDriver`/`NvperfVulkanInitDevice`; `RangeProfilerVulkan` with `PushRange`/`PopRange` wrapping `VkCommandBuffer` submissions; key counter groups (sm, l1tex, lts, dram); `ncu --metrics` CLI path; `nsys profile --trace=cuda,vulkan,nvtx` system timeline capture

**Chapter 57 §7.4 — NVIDIA Video Codec SDK direct API** (added): SDK as header-only (`nvEncodeAPI.h`, `cuviddec.h`) linking against driver-shipped `libnvidia-encode.so`/`libnvcuvid.so`; NVENC runtime-load pattern (`dlopen("libnvidia-encode.so.1")`, `NvEncodeAPICreateInstance`); `NV_ENC_OPEN_ENCODE_SESSION_EX_PARAMS` with `NV_ENC_DEVICE_TYPE_CUDA`; `NV_ENC_INITIALIZE_PARAMS` with preset `NV_ENC_PRESET_P4_GUID`; lookahead (`enableLookahead`, `lookaheadDepth=32`), adaptive quantisation (`enableAQ`), async mode; NVDEC `cuvidCreateDecoder` for zero-copy CUDA-resident decode; when to use SDK vs. FFmpeg (broadcast, sub-10ms latency, zero-copy NVDEC→CUDA→NVENC)

**Chapter 68 §11 — TensorRT** (added): build-then-deploy model — ONNX → `trtexec` engine plan (GPU-arch-specific, driver-version-sensitive); `trtexec --onnx --saveEngine --fp16 --timingCacheFile`; C++ API: `IBuilder`, `INetworkDefinition` (`kSTRONGLY_TYPED` flag), `NvOnnxParser::createParser`, `IOptimizationProfile` for dynamic resolution shapes, `BuilderFlag::kFP16`; runtime: `ICudaEngine::deserialize`, `IExecutionContext::setTensorAddress`, `enqueueV3(cudaStream)`; Vulkan interop via `VK_KHR_external_memory` (Ch25); INT8 calibration with `IInt8EntropyCalibrator2`; when to use (custom neural denoisers, NeRF decoders, RTXNTC-equivalent custom models)

**Chapter 69 §12 — PhysX 5 and NVIDIA Warp** (added): PhysX 5.6.1 (Apache 2.0, github.com/NVIDIA-Omniverse/PhysX); simulation types — rigid body (`PxRigidDynamic`), soft body (`PxSoftBody` FEM), PBD cloth/fluids (`PxPBDParticleSystem`), articulations (`PxArticulationReducedCoordinate`); GPU pipeline — `eENABLE_GPU_DYNAMICS`, `PxBroadPhaseType::eGPU`, `PxCudaContextManager`; PhysX → USD bridge via `UsdPhysicsRigidBodyAPI`, `PhysxSchema`; USDRT Fabric zero-copy geometry update path; NVIDIA Warp (BSD-3, pip install `warp-lang`) — Python-like GPU kernels JIT-compiled to PTX; `@wp.kernel` with `wp.atomic_add`, `wp.tid()`; differentiable kernels via `wp.Tape`; Warp ↔ USD Fabric zero-copy via DLPack (`omni.warp.from_fabric`); Newton physics engine (GTC 2025) built on Warp

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
