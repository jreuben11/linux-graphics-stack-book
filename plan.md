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
  - [Chapter 10a: Nova — The Rust NVIDIA Kernel Driver](#chapter-10a-nova--the-rust-nvidia-kernel-driver)
  - [Chapter 10b: NVK: Building a Vulkan Driver from Scratch](#chapter-10b-nvk-building-a-vulkan-driver-from-scratch)
  - [Chapter 11: Display, Reclocking, and Power Management](#chapter-11-display-reclocking-and-power-management)
- **Part IV — Mesa Architecture**
  - [Chapter 12: The Mesa Loader and Driver Dispatch](#chapter-12-the-mesa-loader-and-driver-dispatch)
  - [Chapter 13: Gallium3D: The OpenGL State Tracker](#chapter-13-gallium3d-the-opengl-state-tracker)
  - [Chapter 14: NIR: Mesa's Shader Intermediate Representation](#chapter-14-nir-mesas-shader-intermediate-representation)
  - [Chapter 15: ACO: AMD's Optimising Compiler](#chapter-15-aco-amds-optimising-compiler)
  - [Chapter 16: Mesa's Vulkan Common Infrastructure](#chapter-16-mesas-vulkan-common-infrastructure)
  - [Chapter 17: Software Renderers](#chapter-17-software-renderers)
  - [Chapter 159: The Vulkan–Mesa–DRM Stack: A Full Vertical Slice](#chapter-159-the-vulkanmesadrm-stack-a-full-vertical-slice)
- **Part V — Mesa GPU Drivers**
  - [Chapter 18: Vulkan Drivers](#chapter-18-vulkan-drivers)
  - [Chapter 19: OpenGL and Compatibility Drivers](#chapter-19-opengl-and-compatibility-drivers)
- **Part VI-A — Wayland Protocol and Compositor Architecture**
  - [Chapter 20: Wayland Protocol Fundamentals](#chapter-20-wayland-protocol-fundamentals)
  - [Chapter 21: Building Compositors with wlroots](#chapter-21-building-compositors-with-wlroots)
  - [Chapter 22: Production Compositors](#chapter-22-production-compositors)
  - [Chapter 23: Legacy and Sandboxed App Support](#chapter-23-legacy-and-sandboxed-app-support)
  - [Chapter 46: The Evolving Wayland Protocol Ecosystem](#chapter-46-the-evolving-wayland-protocol-ecosystem)
- **Part VI-B — Display Services, Input, and Color**
  - [Chapter 53: Display Calibration and colord](#chapter-53-display-calibration-and-colord)
  - [Chapter 54: The Linux Input Stack](#chapter-54-the-linux-input-stack)
  - [Chapter 105: Font Rendering — FreeType2, HarfBuzz, and the Text Pipeline](#chapter-105-font-rendering--freetype2-harfbuzz-and-the-text-pipeline)
  - [Chapter 112: Variable Refresh Rate — FreeSync, G-Sync, and Frame Pacing](#chapter-112-variable-refresh-rate--freesync-g-sync-and-frame-pacing)
  - [Chapter 194: Cross-Stack Integration — Protocols, Synchronisation, and the Coordination Layer](#chapter-194-cross-stack-integration--protocols-synchronisation-and-the-coordination-layer)
- **Part VII-A — GPU APIs and Extended Reality**
  - [Chapter 24: Vulkan and EGL for Application Developers](#chapter-24-vulkan-and-egl-for-application-developers)
  - [Chapter 25: GPU Compute](#chapter-25-gpu-compute)
  - [Chapter 26: Hardware Video](#chapter-26-hardware-video)
  - [Chapter 27: VR & AR](#chapter-27-vr--ar)
- **Part VII-B — Multimedia Frameworks and Desktop Integration**
  - [Chapter 38: PipeWire and the Video Session Layer](#chapter-38-pipewire-and-the-video-session-layer)
  - [Chapter 38b: ALSA — The Linux Audio Subsystem](#chapter-38b-alsa--the-linux-audio-subsystem)
  - [Chapter 39: Qt and GTK GPU Rendering](#chapter-39-qt-and-gtk-gpu-rendering)
  - [Chapter 47: Font and Text Rendering Pipeline](#chapter-47-font-and-text-rendering-pipeline)
  - [Chapter 50: Vulkan Video Extensions](#chapter-50-vulkan-video-extensions)
  - [Chapter 111: Flatpak Graphics — GPU Access in Sandboxed Applications](#chapter-111-flatpak-graphics--gpu-access-in-sandboxed-applications)
  - [Chapter 114: OpenCV and GPU-Accelerated Computer Vision on Linux](#chapter-114-opencv-and-gpu-accelerated-computer-vision-on-linux)
- **Part VIII — Gaming Layer**
  - [Chapter 28: Windows Compatibility](#chapter-28-windows-compatibility)
  - [Chapter 29: Upscaling, Effects & Overlays](#chapter-29-upscaling-effects--overlays)
  - [Chapter 56: Ray Tracing on Linux](#chapter-56-ray-tracing-on-linux)
  - [Chapter 104: DXVK and VKD3D-Proton — D3D-to-Vulkan Translation](#chapter-104-dxvk-and-vkd3d-proton--d3d-to-vulkan-translation)
- **Part IX — Tooling & Contributing**
  - [Chapter 30: Debugging & Profiling](#chapter-30-debugging--profiling)
  - [Chapter 31: Conformance & Regression Testing](#chapter-31-conformance--regression-testing)
  - [Chapter 32: Contributing to the Linux Graphics Stack](#chapter-32-contributing-to-the-linux-graphics-stack)
  - [Chapter 55: GPU Containers and Cloud Compute](#chapter-55-gpu-containers-and-cloud-compute)
  - [Chapter 107: Headless Rendering — Offscreen, CI, and Server Workloads](#chapter-107-headless-rendering--offscreen-ci-and-server-workloads)
  - [Chapter 109: Mesa Testing — piglit, dEQP, and Continuous Integration](#chapter-109-mesa-testing--piglit-deqp-and-continuous-integration)
- **Part X — The Browser Rendering Stack**
  - [Chapter 33: Chromium's Multi-Process GPU Architecture](#chapter-33-chromiums-multi-process-gpu-architecture)
  - [Chapter 34: ANGLE — WebGL on Linux](#chapter-34-angle--webgl-on-linux)
  - [Chapter 35: Dawn and WebGPU](#chapter-35-dawn-and-webgpu)
  - [Chapter 36: The Chromium Compositor — CC and Viz](#chapter-36-the-chromium-compositor--cc-and-viz)
  - [Chapter 37: Skia and 2D Rendering](#chapter-37-skia-and-2d-rendering)
  - [Chapter 52: Firefox and WebRender](#chapter-52-firefox-and-webrender)
  - [Chapter 203: WebXR — Browser-Based Immersive Experiences on Linux](#chapter-203-webxr--browser-based-immersive-experiences-on-linux)
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
  - [Chapter 110: SPIR-V Tooling — spirv-tools, SPIRV-Cross, and the Shader Ecosystem](#chapter-110-spir-v-tooling--spirv-tools-spirv-cross-and-the-shader-ecosystem)
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
  - [Chapter 74: HDR and Wide Color Gamut on Linux](#chapter-74-hdr-and-wide-color-gamut-on-linux) *(Part VI-B — Display Services)*
  - [Chapter 75: Explicit GPU Synchronisation](#chapter-75-explicit-gpu-synchronisation) *(Part VI-B — Display Services)*
  - [Chapter 76: Modern Vulkan Extensions](#chapter-76-modern-vulkan-extensions) *(Part VII-A — GPU APIs)*
  - [Chapter 77: Shader Source-to-ISA: The Complete Compilation Toolchain](#chapter-77-shader-source-to-isa-the-complete-compilation-toolchain) *(Part IV)*
  - [Chapter 78: Gamescope and the Steam Deck: A Complete Gaming Graphics Stack](#chapter-78-gamescope-and-the-steam-deck-a-complete-gaming-graphics-stack) *(Part VIII)*
  - [Chapter 79: Remote Display, Screen Casting, and GPU-Accelerated Game Streaming](#chapter-79-remote-display-screen-casting-and-gpu-accelerated-game-streaming) *(Part IX)*
  - [Chapter 80: GPU Security: Isolation, Content Protection, and Confidential Computing](#chapter-80-gpu-security-isolation-content-protection-and-confidential-computing) *(Part IX)*
- **Part XVIII — Rendering Abstraction Libraries**
  - [Chapter 81: SDL3 GPU API: A Portable High-Level GPU Abstraction](#chapter-81-sdl3-gpu-api-a-portable-high-level-gpu-abstraction)
  - [Chapter 82: Vulkan Ecosystem Toolkit: VMA, volk, vk-bootstrap, and Friends](#chapter-82-vulkan-ecosystem-toolkit-vma-volk-vk-bootstrap-and-friends)
  - [Chapter 83: Filament: Google's Physically Based Rendering Engine on Linux](#chapter-83-filament-googles-physically-based-rendering-engine-on-linux)
  - [Chapter 84: bgfx, Cross-Platform Rendering Abstractions, and the Frame Graph Pattern](#chapter-84-bgfx-cross-platform-rendering-abstractions-and-the-frame-graph-pattern)
  - [Chapter 113: CGAL and Computational Geometry on the Linux Graphics Stack](#chapter-113-cgal-and-computational-geometry-on-the-linux-graphics-stack)
- **Part XIX — Android Graphics**
  - [Chapter 85: Android Compositor: SurfaceFlinger, HardwareBuffer, and the Buffer Pipeline](#chapter-85-android-compositor-surfaceflinger-hardwarebuffer-and-the-buffer-pipeline)
  - [Chapter 86: Vulkan on Android: Drivers, ANGLE, and Mobile GPU Performance](#chapter-86-vulkan-on-android-drivers-angle-and-mobile-gpu-performance)
  - [Chapter 87: Android AR: ARCore Architecture, Camera HAL Integration, and the Android XR Platform](#chapter-87-android-ar-arcore-architecture-camera-hal-integration-and-the-android-xr-platform)
  - [Chapter 161: Android Game Development Kit (AGDK): Native Game Architecture, Input, Audio, and Frame Pacing](#chapter-161-android-game-development-kit-agdk-native-game-architecture-input-audio-and-frame-pacing)
  - [Chapter 164: Android Runtime and Native Interop: ART, JNI, and the NDK](#chapter-164-android-runtime-and-native-interop-art-jni-and-the-ndk)
- **Part XX — AI/ML Inference on Linux**
  - [Chapter 48: ROCm and Machine Learning on Linux GPUs](#chapter-48-rocm-and-machine-learning-on-linux-gpus)
  - [Chapter 88: NPU and AI Accelerator Integration on Linux](#chapter-88-npu-and-ai-accelerator-integration-on-linux)
  - [Chapter 94: ComfyUI and ComfyScript: Node-Graph AI Image Generation on Linux GPUs](#chapter-94-comfyui-and-comfyscript-node-graph-ai-image-generation-on-linux-gpus)
  - [Chapter 108: ROCm and HIP — AMD's GPU Compute Stack](#chapter-108-rocm-and-hip--amds-gpu-compute-stack)
  - [Chapter 115: NeRFStudio, Neural Radiance Fields, and 3D Gaussian Splatting on Linux](#chapter-115-nerfstudio-neural-radiance-fields-and-3d-gaussian-splatting-on-linux)
  - [Chapter 124: Local LLM Inference on Linux GPUs](#chapter-124-local-llm-inference-on-linux-gpus)
- **Part XX additions to existing parts**
  - [Chapter 89: GPU Virtualization in Depth](#chapter-89-gpu-virtualization-in-depth) *(Part IX)*
  - [Chapter 90: Open ARM GPU Drivers — Lima, Panfrost, and Panthor](#chapter-90-open-arm-gpu-drivers--lima-panfrost-and-panthor) *(Part II)*
  - [Chapter 91: MLIR and the Emerging GPU Compiler Infrastructure](#chapter-91-mlir-and-the-emerging-gpu-compiler-infrastructure) *(Part IV)*
  - [Chapter 92: The Raspberry Pi GPU Stack — VideoCore and V3D](#chapter-92-the-raspberry-pi-gpu-stack--videocore-and-v3d) *(Part II)*
  - [Chapter 93: GPU Performance Analysis Methodology](#chapter-93-gpu-performance-analysis-methodology) *(Part IX)*
- **Part XXI — Platform, Legacy, and History**
  - [Chapter 95: X11/Xorg Architecture and the DRI Legacy Stack](#chapter-95-x11xorg-architecture-and-the-dri-legacy-stack)
  - [Chapter 103: The Linux Graphics Stack: History and Design Philosophy](#chapter-103-the-linux-graphics-stack-history-and-design-philosophy)
  - [Chapter 197: The Linux Graphics Stack in Context — Comparison with Windows and macOS](#chapter-197-the-linux-graphics-stack-in-context--comparison-with-windows-and-macos)
- **Part XXI additions to existing parts**
  - [Chapter 96: libcamera and the Linux Camera Stack](#chapter-96-libcamera-and-the-linux-camera-stack) *(Part VII-B — Multimedia Frameworks)*
  - [Chapter 97: Unreal Engine 5 on Linux](#chapter-97-unreal-engine-5-on-linux) *(Part XI)*
  - [Chapter 98: WebAssembly and WebGPU as a Deployment Target](#chapter-98-webassembly-and-webgpu-as-a-deployment-target) *(Part X)*
  - [Chapter 99: Automotive and Embedded Linux Graphics](#chapter-99-automotive-and-embedded-linux-graphics) *(Part II)*
  - [Chapter 100: etnaviv: The Vivante GPU Open Driver](#chapter-100-etnaviv-the-vivante-gpu-open-driver) *(Part II)*
  - [Chapter 101: Color Science and the ICC Profile Pipeline](#chapter-101-color-science-and-the-icc-profile-pipeline) *(Part VI-B — Display Services)*
  - [Chapter 102: The DRM GPU Scheduler and Multi-Process Fairness](#chapter-102-the-drm-gpu-scheduler-and-multi-process-fairness) *(Part I)*
- **Part XXII — Additional Chapters**
  - [Chapter 116: RISC-V GPU Drivers](#chapter-116-risc-v-gpu-drivers) *(Part II)*
  - [Chapter 117: Slang — Differentiable and Modular Shading Language](#chapter-117-slang--differentiable-and-modular-shading-language) *(Part XV)*
  - [Chapter 118: NAK — The Nouveau/NVK Rust Shader Compiler](#chapter-118-nak--the-nouveaunk-rust-shader-compiler) *(Part III)*
  - [Chapter 119: Zink — OpenGL on Vulkan](#chapter-119-zink--opengl-on-vulkan) *(Part IV)*
  - [Chapter 120: GPU Memory Management Internals — TTM, GEM, and BAR](#chapter-120-gpu-memory-management-internals--ttm-gem-and-bar) *(Part I)*
  - [Chapter 121: DRM Lease and VR Direct Display](#chapter-121-drm-lease-and-vr-direct-display) *(Part I)*
  - [Chapter 122: DKMS and Out-of-Tree GPU Kernel Modules](#chapter-122-dkms-and-out-of-tree-gpu-kernel-modules) *(Part IX)*
  - [Chapter 123: Screen Capture and Remote Desktop on Linux](#chapter-123-screen-capture-and-remote-desktop-on-linux) *(Part VI-B — Display Services)*
  - [Chapter 134: OpenCL on Linux](#chapter-134-opencl-on-linux) *(Part XIV)*
  - [Chapter 125: RenderDoc on Linux](#chapter-125-renderdoc-on-linux) *(Part IX)*
  - [Chapter 126: Hybrid Graphics and Laptop Power Management](#chapter-126-hybrid-graphics-and-laptop-power-management) *(Part II)*
  - [Chapter 127: Mesh Shaders and Variable Rate Shading](#chapter-127-mesh-shaders-and-variable-rate-shading) *(Part VII-A — GPU APIs)*
  - [Chapter 128: DisplayPort MST and Multi-Monitor Topology](#chapter-128-displayport-mst-and-multi-monitor-topology) *(Part VI-B — Display Services)*
  - [Chapter 129: GPU Firmware Deep Dive](#chapter-129-gpu-firmware-deep-dive) *(Part I)*
  - [Chapter 130: Wayland Protocol Extension Development](#chapter-130-wayland-protocol-extension-development) *(Part VI-A — Wayland Compositor)*
  - [Chapter 131: Touch, Stylus, and Tablet Input on Wayland](#chapter-131-touch-stylus-and-tablet-input-on-wayland) *(Part VI-B — Display Services)*
  - [Chapter 132: Wayland Security](#chapter-132-wayland-security) *(Part VI-A — Wayland Compositor)*
  - [Chapter 133: Vulkan Compute Queues and Task Graphs](#chapter-133-vulkan-compute-queues-and-task-graphs) *(Part VII-A — GPU APIs)*
- **Part XXIII — Additional Chapters (set 2)**
  - [Chapter 135: Vulkan Ray Tracing on Linux](#chapter-135-vulkan-ray-tracing-on-linux) *(Part VII-A — GPU APIs)*
  - [Chapter 136: WSL2 Linux Graphics — dxgkrnl and Mesa D3D12](#chapter-136-wsl2-linux-graphics--dxgkrnl-and-mesa-d3d12) *(Part IX)*
  - [Chapter 137: GPU Performance Profiling — RGP, GPA, and VK_EXT_performance_query](#chapter-137-gpu-performance-profiling--rgp-gpa-and-vk_ext_performance_query) *(Part IX)*
  - [Chapter 138: Wayland Fractional Scaling and HiDPI](#chapter-138-wayland-fractional-scaling-and-hidpi) *(Part VI-A — Wayland Compositor)*
  - [Chapter 139: DRM Hardware Overlay Planes and Composition Bypass](#chapter-139-drm-hardware-overlay-planes-and-composition-bypass) *(Part I)*
  - [Chapter 140: HDMI and DisplayPort Audio on Linux](#chapter-140-hdmi-and-displayport-audio-on-linux) *(Part VI-B — Display Services)*
  - [Chapter 141: Vulkan Cooperative Matrices and GPU ML Acceleration](#chapter-141-vulkan-cooperative-matrices-and-gpu-ml-acceleration) *(Part VII-A — GPU APIs)*
- **Part XXIV — Critical Gap Chapters**
  - [Chapter 142: V4L2 and the Linux Media Subsystem](#chapter-142-v4l2-and-the-linux-media-subsystem) *(Part XIII)*
  - [Chapter 143: RADV Internals: The Mesa AMD Vulkan Driver](#chapter-143-radv-internals-the-mesa-amd-vulkan-driver) *(Part XVII)*
  - [Chapter 144: Boot Graphics Pipeline: From Firmware to KMS Handoff](#chapter-144-boot-graphics-pipeline-from-firmware-to-kms-handoff) *(Part I)*
  - [Chapter 145: XWayland: Architecture and the X11-to-Wayland Bridge](#chapter-145-xwayland-architecture-and-the-x11-to-wayland-bridge) *(Part VI-A — Wayland Compositor)*
  - [Chapter 146: WebCodecs and Browser Hardware Acceleration](#chapter-146-webcodecs-and-browser-hardware-acceleration) *(Part X)*
  - [Chapter 147: Chrome and Firefox Hardware Video Decode via VA-API](#chapter-147-chrome-and-firefox-hardware-video-decode-via-va-api) *(Part X)*
  - [Chapter 148: Vulkan Synchronisation: A Complete Developer Reference](#chapter-148-vulkan-synchronisation-a-complete-developer-reference) *(Part VII-A — GPU APIs)*
- **Part XXV — Advanced Topics and Gap-Fill**
  - [Chapter 106: The Vulkan Memory Model — Formal Execution and Memory Ordering](#chapter-106-the-vulkan-memory-model--formal-execution-and-memory-ordering) *(Part VII-A — GPU APIs)*
  - [Chapter 149: GPU Hang Detection and Recovery — TDR, Scheduling Timeouts, and Reset Sequences](#chapter-149-gpu-hang-detection-and-recovery--tdr-scheduling-timeouts-and-reset-sequences) *(Part I)*
  - [Chapter 150: EGL Architecture and DMA-BUF Integration](#chapter-150-egl-architecture-and-dmabuf-integration) *(Part VII-A — GPU APIs)*
  - [Chapter 151: Wayland Text Input and Input Method Editors](#chapter-151-wayland-text-input-and-input-method-editors) *(Part VI-A — Wayland Compositor)*
  - [Chapter 152: The Rust GPU Ecosystem: ash, wgpu, naga, and Bevy](#chapter-152-the-rust-gpu-ecosystem-ash-wgpu-naga-and-bevy) *(Part VII-A — GPU APIs)*
  - [Chapter 153: OBS Studio GPU Pipeline: Capture, Encode, and Stream](#chapter-153-obs-studio-gpu-pipeline-capture-encode-and-stream) *(Part IX)*
  - [Chapter 154: GPU-Driven Rendering: Indirect Draw, Culling, and Mesh Shaders](#chapter-154-gpu-driven-rendering-indirect-draw-culling-and-mesh-shaders) *(Part VII-A — GPU APIs)*
  - [Chapter 192: GPU-Generated Commands — VK\_EXT\_device\_generated\_commands and Work Graphs](#chapter-192-gpu-generated-commands--vk_ext_device_generated_commands-and-work-graphs) *(Part VII-A — GPU APIs)*
  - [Chapter 155: USB DisplayLink and the evdi Virtual DRM Driver](#chapter-155-usb-displaylink-and-the-evdi-virtual-drm-driver) *(Part II)*
  - [Chapter 156: Mesa Nine: The Direct3D 9 State Tracker for Gallium](#chapter-156-mesa-nine-the-direct3d-9-state-tracker-for-gallium) *(Part IV)*
  - [Chapter 157: Vulkan Descriptor Binding: Sets, Push Descriptors, and Descriptor Buffers](#chapter-157-vulkan-descriptor-binding-sets-push-descriptors-and-descriptor-buffers) *(Part VII-A — GPU APIs)*
  - [Chapter 158: HDR and Display Color Management on Linux](#chapter-158-hdr-and-display-color-management-on-linux) *(Part VI-B — Display Services)*
  - [Chapter 160: Freedreno, Turnip, and the Qualcomm Adreno Driver](#chapter-160-freedreno-turnip-and-the-qualcomm-adreno-driver) *(Part II)*
  - [Chapter 162: Framebuffer Compression: AFBC, DCC, CCS, and UBWC](#chapter-162-framebuffer-compression-afbc-dcc-ccs-and-ubwc) *(Part I)*
  - [Chapter 163: VKMS and Virtual Display Drivers for Testing](#chapter-163-vkms-and-virtual-display-drivers-for-testing) *(Part I)*
  - [Chapter 165: Vulkan Video: Hardware Decode and Encode via the Vulkan API](#chapter-165-vulkan-video-hardware-decode-and-encode-via-the-vulkan-api) *(Part VII-A — GPU APIs)*
- **Part XXVI — New Chapters: Gap Analysis Fill**
  - [Chapter 167: NTSYNC — NT Synchronization Primitives in the Linux Kernel](#chapter-167-ntsync--nt-synchronization-primitives-in-the-linux-kernel) *(Part VIII)*
  - [Chapter 168: WebNN — The Web Neural Network API](#chapter-168-webnn--the-web-neural-network-api) *(Part X)*
  - [Chapter 169: Snapdragon X Elite on Linux — Adreno X1-85, freedreno, and the Arm Laptop Era](#chapter-169-snapdragon-x-elite-on-linux--adreno-x1-85-freedreno-and-the-arm-laptop-era) *(Part II)*
  - [Chapter 170: AMDVLK vs. RADV — AMD's Two Open Vulkan Drivers](#chapter-170-amdvlk-vs-radv--amds-two-open-vulkan-drivers) *(Part XVII)*
  - [Chapter 171: Linux Gaming Anti-Cheat — EasyAntiCheat, BattlEye, and the Ring-0 Problem](#chapter-171-linux-gaming-anti-cheat--easyanticheat-battleye-and-the-ring-0-problem) *(Part VIII)*
  - [Chapter 172: eGPU on Linux — Thunderbolt, USB4, and PCIe Hot-Plug](#chapter-172-egpu-on-linux--thunderbolt-usb4-and-pcie-hot-plug) *(Part II)*
  - [Chapter 173: VK\_EXT\_shader\_object — Pipeline-Free Shader Binding in Vulkan](#chapter-173-vk_ext_shader_object--pipeline-free-shader-binding-in-vulkan) *(Part VII-A — GPU APIs)*
  - [Chapter 200: Vulkan Memory Allocation and Resource Management](#chapter-200-vulkan-memory-allocation-and-resource-management) *(Part VII-A — GPU APIs)*
  - [Chapter 201: Vulkan Debugging, Validation, and Profiling](#chapter-201-vulkan-debugging-validation-and-profiling) *(Part VII-A — GPU APIs)*
  - [Chapter 202: Vulkan WSI Deep Dive](#chapter-202-vulkan-wsi-deep-dive) *(Part VII-A — GPU APIs)*
  - [Chapter 174: WezTerm and Alacritty — GPU Terminal Rendering Architectures](#chapter-174-wezterm-and-alacritty--gpu-terminal-rendering-architectures) *(Part XII)*
  - [Chapter 175: Linux Compositor Accessibility — AT-SPI2, Screen Readers, and the Wayland Gap](#chapter-175-linux-compositor-accessibility--at-spi2-screen-readers-and-the-wayland-gap) *(Part VI-A — Wayland Compositor)*
  - [Chapter 176: OpenCASCADE Technology — The BRep Kernel and 3D Visualization Stack](#chapter-176-opencascade-technology--the-brep-kernel-and-3d-visualization-stack) *(Part XI)*
  - [Chapter 177: NVK — NVIDIA Vulkan in Mesa: Architecture, WSI, and Conformance](#chapter-177-nvk--nvidia-vulkan-in-mesa-architecture-wsi-and-conformance) *(Part V)*
  - [Chapter 178: The PTY/TTY Kernel Layer and Line Disciplines](#chapter-178-the-ptytty-kernel-layer-and-line-disciplines) *(Part XII)*
  - [Chapter 179: The Linux `accel` Subsystem — NPU and AI Accelerator Drivers](#chapter-179-the-linux-accel-subsystem--npu-and-ai-accelerator-drivers) *(Part II)*
  - [Chapter 180: GPU Reverse Engineering — Tools, Methodology, and Case Studies](#chapter-180-gpu-reverse-engineering--tools-methodology-and-case-studies) *(Part IX)*
- **Part XXVII — Display Hardware, Connectors, and Signal Standards**
  - [Chapter 181: Modern Display Interface Standards](#chapter-181-modern-display-interface-standards)
  - [Chapter 182: Digital Display Connectors and the Physical Layer](#chapter-182-digital-display-connectors-and-the-physical-layer)
  - [Chapter 183: EDID and DisplayID — How Linux Discovers Display Capabilities](#chapter-183-edid-and-displayid--how-linux-discovers-display-capabilities)
  - [Chapter 184: Embedded DisplayPort (eDP) and Laptop Panel Management](#chapter-184-embedded-displayport-edp-and-laptop-panel-management)
  - [Chapter 185: Wireless Display Technologies on Linux](#chapter-185-wireless-display-technologies-on-linux)
  - [Chapter 186: Video Pixel Formats and Display Signal Encoding](#chapter-186-video-pixel-formats-and-display-signal-encoding)
  - [Chapter 187: HDMI CEC and the Linux CEC Subsystem](#chapter-187-hdmi-cec-and-the-linux-cec-subsystem)
  - [Chapter 188: Display Power States — DPMS, Panel Self-Refresh, and Display Idle Management](#chapter-188-display-power-states--dpms-panel-self-refresh-and-display-idle-management)
- **Part XXVII additions to existing parts**
  - [Chapter 189: VLC Media Player — Architecture, GPU Acceleration, and the Linux Graphics Stack](#chapter-189-vlc-media-player--architecture-gpu-acceleration-and-the-linux-graphics-stack) *(Part XIII)*
  - [Chapter 190: VTK — Scientific Visualization on the Linux Graphics Stack](#chapter-190-vtk--scientific-visualization-on-the-linux-graphics-stack) *(Part XI)*

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

### Chapter 196: The GPU as Embedded Computer — Firmware-as-OS Architecture Across Vendors *(Part II)*
- The GPU-as-embedded-computer paradigm: modern GPUs contain dedicated management processors (Falcon, RISC-V, GuC IA cores, ARM cores) running their own firmware OSes; the host driver is increasingly an RPC/CTB client
- NVIDIA GSP-RM: the most complete firmware-OS model; Falcon-to-RISC-V evolution; full Resource Manager on GPU; CPU driver as thin RPC client; hardware-signed firmware; SR-IOV virtualisation model
- AMD's tiered firmware architecture: PSP (root of trust), SMU (power/thermal), DMCUB (display DSP), GFX ME/MEC (command processor firmware), MES (hardware queue scheduler), VCN; contrast — amdgpu still does direct register programming for GPU engine init
- Intel GuC/HuC/GSC: GuC handles workload scheduling (CTB ring-less submission) and HuC authentication; GSC handles PXP DRM attestation on Xe; xe.ko/i915.ko still perform hardware init and memory management; lighter firmware dependency than NVIDIA
- Qualcomm Adreno: minimal firmware model; freedreno does direct register programming; Zap security firmware for mobile SoCs; CP microcode partially open; the "old model" surviving through vendor cooperation
- Other vendors: Broadcom V3D (minimal firmware, mostly in-kernel); ARM Mali CSF on Valhall (job scheduling moves into firmware, changing submission model — Panthor implements this); Apple AGX (adversarial, full firmware RE by Asahi)
- Vendor spectrum table: register-programming ↔ firmware-assisted ↔ firmware-as-OS across all vendors
- Security and trust model comparison: IOMMU firmware DMA, hardware signature verification, vGPU/SR-IOV isolation, attestation (NVIDIA vGPU licensing, Intel PXP), threat surface differences
- Open-source driver feasibility: can you write a correct open-source driver without firmware source? Verdict by vendor; NVIDIA nova/NVK as existence proof; ARM Mali CSF as the hardest non-adversarial case
- Convergence trajectory: evidence the industry is moving toward NVIDIA's model (AMD MES expansion, Intel GSC scope growth, ARM Mali CSF); counter-evidence (Qualcomm staying open, RISC-V GPU community)
- Strategic outlook: implications for kernel driver developers, security researchers, distribution maintainers, and the open-source GPU ecosystem
- **Integrations**: x86 GPU drivers in detail (Ch5); NVIDIA GSP-RM deep dive (Ch9); Asahi AGX adversarial RE (Ch73); ARM Mali CSF/Panthor (Ch90); firmware loading mechanics (Ch129); Qualcomm minimal-firmware model (Ch160)

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
- **Integrations**: nvkm implements the DRM driver interface (Ch1), exposing GEM/TTM objects consumed by NVK (Ch10b) and the legacy GL driver; the fence/scheduler layer connects to DMA-BUF implicit sync (Ch4) and the explicit sync path required by Wayland (Ch3); GSP-RM offloading (Ch9) changes which nvkm engines are active at runtime

### Chapter 9: GSP-RM, Firmware, and the nvidia-open Connection
- What GSP-RM is: the GPU System Processor and its firmware
- How nvidia-open offloads driver logic to GSP-RM
- What the open kernel module exposes vs. what remains in firmware blobs
- Nouveau's GSP-RM support: booting nouveau with NVIDIA's own firmware
- Implications for feature parity, power management, and security
- The path toward a fully open NVIDIA stack
- **Integrations**: GSP-RM changes the interface between nvkm and the GPU hardware (Ch8); using GSP-RM firmware unlocks reclocking and power management that were previously blocked (Ch11); the DMA-BUF/explicit sync improvements enabled by nvidia-open are what unblocked proper NVIDIA Wayland support (Ch3, Ch20)

### Chapter 10a: Nova — The Rust NVIDIA Kernel Driver

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
- **Integrations**: the display engine integrates with KMS (Ch2) via DRM connector and CRTC callbacks; reclocking directly affects the performance available to Mesa/NVK workloads (Ch10b, Ch18); power management hooks into the DRM runtime PM framework and influences how the GPU scheduler (Ch8) throttles command submission

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

### Chapter 192: GPU-Generated Commands — VK_EXT_device_generated_commands and Work Graphs *(Part VII-A — GPU APIs)*
- History: from `VK_NV_device_generated_commands` (NVIDIA-only) to the multi-vendor EXT; design goals (remove last CPU bottleneck in GPU-driven scenes)
- `VkIndirectCommandsLayoutEXT`: token sequence design, token types, `indirectStride`, dispatch-token constraint
- `VkIndirectExecutionSetEXT`: pipeline-mode and shader-object-mode IES; GPU-side pipeline/shader selection by index; update functions
- All `VK_INDIRECT_COMMANDS_TOKEN_TYPE_*` values: EXECUTION_SET, PUSH_CONSTANT, INDEX_BUFFER, VERTEX_BUFFER, DRAW_INDEXED, DRAW_MESH_TASKS, DISPATCH, TRACE_RAYS2
- `VkIndirectCommandsInputModeFlagBitsEXT`: VULKAN vs DXGI index buffer format (VKD3D-Proton binary-compat path)
- Scratch memory: `vkGetGeneratedCommandsMemoryRequirementsEXT`, `VK_BUFFER_USAGE_2_PREPROCESS_BIT_EXT`, 32-bit address constraint on AMD
- Two-phase model: inline execution vs explicit preprocess; `stateCommandBuffer` argument; async compute overlap; `COMMAND_PREPROCESS_BIT_EXT` barrier stage
- Complete worked example: 64,000-instance GPU-driven multi-material scene, GPU culling compute → DGC sequence records → `vkCmdExecuteGeneratedCommandsEXT`
- RADV implementation (`src/amd/vulkan/radv_dgc.c`): JIT NIR compute shader `meta_dgc_prepare`; scratch buffer sub-region layout; IB chaining for >65536 sequences; ACE parallel stream for mesh/task
- VKD3D-Proton: `ExecuteIndirect` batching (four DGC modes, v3.0.1 coalescing); Halo Infinite/Starfield/Crimson Desert gains; Work Graphs level-by-level emulation without `VK_AMDX_shader_enqueue`
- `VK_AMDX_shader_enqueue`: GPU-native work graphs, `OpEnqueueNodePayloadsAMDX`, `vkCreateExecutionGraphPipelinesAMDX`; comparison with DGC; `VK_KHR_work_graphs` standardisation status
- Performance guide: when DGC wins (deep material diversity, variable-count draws, preprocessing overlap) and when it doesn't (few pipelines, stable draw lists, compute-only)
- **Integrations**: Ch154 (GPU-driven rendering — the motivating pattern), Ch173 (`VK_EXT_shader_object` + IES shader mode), Ch127 (mesh shaders + DRAW_MESH_TASKS token), Ch157 (descriptor heap co-design), Ch104 (VKD3D-Proton ExecuteIndirect), Ch143 (RADV internals — `radv_dgc.c`), Ch135 (ray tracing — TRACE_RAYS2 token), Ch28 (D3D12 Work Graphs status), Ch31 (dEQP-VK.device_generated_commands.*)

### Chapter 159: The Vulkan–Mesa–DRM Stack: A Full Vertical Slice
- Vulkan loader: ICD discovery via JSON manifests, dispatch table construction, implicit/explicit layers
- Mesa Vulkan common runtime: `vk_device`/`vk_instance`/`vk_physical_device` hierarchy, driver registration
- Shader compilation end-to-end: `spirv_to_nir()` → NIR optimisation passes → ACO/BRW/NAK ISA backends; when `vkCreateGraphicsPipelines()` fires compilation
- Command buffer recording: lifecycle (begin/record/end/submit), `VK_KHR_dynamic_rendering` vs render passes, pool and secondary buffer model
- Descriptor sets and resource binding: layout/pool/set creation, GPU-memory descriptor implementation, push constants and push descriptors
- Pipeline barriers, image layouts, and hazard tracking: execution/memory dependencies, layout transition hardware semantics, barrier implementation
- GPU rasterisation pipeline and cache hierarchy: fixed-function stages, fragment quad invocation, framebuffer compression, L1/L2 cache topology
- DRM kernel rendezvous: render vs primary nodes, GEM buffer lifecycle, command submission ioctls, `drm_gpu_scheduler`
- GBM and DMA-BUF: allocation, format modifier negotiation, PRIME cross-device buffer sharing
- Explicit GPU synchronisation: `drm_syncobj` timeline, death of implicit fences, Vulkan timeline semaphores, `linux-drm-syncobj-v1` Wayland protocol
- NVIDIA three-stack coexistence: proprietary closed, nvidia-open (open kernel + closed userspace), nouveau+NVK fully open; GSP firmware role
- Present path: `vkQueuePresentKHR` → Mesa WSI swapchain → `linux-dmabuf-v1` compositor import → KMS atomic commit → scanout; triple-buffer timing
- Supporting subsystems: Resizable BAR / Smart Access Memory, IOMMU DMA safety, GPU microcode firmware
- Memory types, heaps, and `VkMemoryAllocateInfo`: heap topology, allocation strategy, staging buffer upload patterns
- Debugging the full stack: ioctl tracing, GPU hang debugging, NIR dumps, ACO/NAK/BRW disassembly
- Mesa–Wayland bidirectional relationship: Mesa WSI as Wayland client, compositor as Mesa consumer, what Wayland core doesn't know about Mesa
- Common performance patterns: persistent mapped UBOs, indirect draw / GPU-driven rendering
- Full stack diagram (Mermaid): loader → Mesa common → driver → DRM → GPU → KMS
- **Integrations**: Ch01 (DRM/GEM lifecycle), Ch04 (drm_gpu_scheduler), Ch12 (Mesa loader), Ch14 (NIR/spirv_to_nir), Ch15 (ACO backend), Ch16 (Vulkan common layer), Ch18 (RADV/ANV/NVK driver implementations), Ch21/Ch22 (Wayland compositor as Mesa consumer), Ch75 (explicit GPU sync / syncobj), Ch177 (NVK — NVIDIA Vulkan in Mesa)

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

## Part VI-A — Wayland Protocol and Compositor Architecture

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

---

## Part VI-B — Display Services, Input, and Color

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

### Chapter 105: Font Rendering — FreeType2, HarfBuzz, and the Text Pipeline *(Part VI-B)*

- FreeType2 rendering pipeline: glyph rasterisation at the pixel level; hinting modes (autohinter, bytecode interpreter, no-hinting, light); subpixel LCD rendering (`FT_RENDER_MODE_LCD`); the patent history and `FT_Library_SetLcdFilter`; gamma-correct linear blending; FreeType 2.13 variable font support; `FT_Face`, `FT_GlyphSlot`, `FT_Bitmap` API lifecycle; source at https://freetype.org/
- HarfBuzz text shaping: the OpenType shaping engine; `hb_buffer_t` Unicode codepoint input; `hb_font_t` backed by FreeType or native; `hb_shape()` — GSUB ligature substitution, GPOS mark positioning, kerning; complex script support: Arabic contextual forms, Devanagari vowel signs, Hangul jamo composition; cluster model for cursor and selection; BiDi text via `hb_unicode_funcs_t`; source at https://harfbuzz.github.io/
- fontconfig: font discovery and pattern matching; `FcPattern`, `FcFontList`, `FcFontMatch`; alias chains (serif → Liberation Serif → actual path); per-user `fonts.conf` and system `/etc/fonts/conf.d/`; binary cache at `~/.cache/fontconfig/`; integration with FreeType for hinting overrides; `fc-list`, `fc-match`, `fc-scan`
- Glyph atlas management: packing glyph bitmaps into GPU texture atlases; 2D atlas vs 3D texture array strategies; LRU eviction; SDF (signed distance field) rendering for resolution-independent scalable glyphs; distance field atlas generation with `msdfgen`; how Qt (`QFontEngine`), GTK4 (`PangoCairoFcFontMap`), Skia, and terminal emulators all maintain separate atlas implementations
- Wayland subpixel rendering considerations: why composited pipelines break X11 LCD subpixel rendering (alpha multiplication destroys channel offsets); `wl_output.subpixel` hint for FreeType mode selection; grayscale fallback on Wayland; per-output rendering adjustment
- Variable fonts and colour emoji: OpenType variable axes (wght, wdth, ital, slnt); FreeType 2.7+ variable instance rendering; HarfBuzz `hb_variation_t`; colour emoji via OpenType CBDT/CBLC (bitmap) and COLRv1 (vector); `FT_Load_Glyph` with `FT_LOAD_COLOR` flag
- **Integrations**: FreeType/HarfBuzz are consumed by Qt (Ch39), GTK4 (Ch39), Skia (Ch37), terminal emulators (Ch44, Ch47); `wl_output.subpixel` comes from the Wayland compositor output model (Ch20); KMS colour pipeline (Ch3) and HDR (Ch74) affect how gamma-corrected glyphs appear on HDR displays; Ch47 covers Pango/Cairo as higher-level text layout consumers of this chapter's primitives

### Chapter 112: Variable Refresh Rate — FreeSync, G-Sync, and Frame Pacing *(Part VI-B)*

- VRR fundamentals: why fixed-refresh-rate displays cause frame pacing artefacts; the display panel's variable-blanking-interval mechanism; VESA Adaptive-Sync (DisplayPort 1.2a) and HDMI 2.1 VRR as the hardware standards; FreeSync (AMD open spec) vs G-Sync Compatible (NVIDIA validation) vs G-Sync module (proprietary scaler) — what differs at the panel hardware level
- DRM/KMS VRR support: `VRR_ENABLED` atomic property on CRTC; `DRM_CAP_ADDFB2_MODIFIERS`; drm connector `VRR_CAPABLE` property; `vrr_min_hz` and `vrr_max_hz` from EDID; kernel amdgpu VRR implementation (`amdgpu_dm_crtc_vrr_active()`); i915/Xe VRR for Intel Arc panels; nouveau VRR status
- Compositor VRR integration: Mutter (`meta-kms-crtc.c` VRR toggle, GNOME 46); KWin (`compositor/drm/drm_crtc.cpp`, Plasma 6.0 VRR per-output); wlroots `wlr_output_state_set_adaptive_sync`; the `wp_presentation` timing feedback interaction on VRR displays (undefined refresh period, dynamic `presentedAt` timestamps)
- Application-side frame pacing: the frame loop model on VRR — no fixed target; `wp_fifo_v1` backpressure (Ch46) as the Wayland-native pacing mechanism for VRR; gamescope VRR mode for Steam Deck (Ch78); MangoHud frame pacing graph (Ch29) and VRR impact visualisation
- NVIDIA G-Sync on Linux: nouveau VRR (limited); NVIDIA proprietary driver VRR via `nvidia-drm`; `nvidia-settings --assign CurrentMetaMode` for G-Sync toggle; interaction with the explicit sync path (Ch75) required for NVIDIA Wayland
- Low Framerate Compensation (LFC): behaviour below VRR minimum refresh rate — double-frame or fixed-rate fallback; per-driver LFC minimum; interaction with `wp_fifo_v1` at sub-minimum rates
- **Integrations**: Ch2 (KMS atomic — VRR_ENABLED property on CRTC), Ch3 (advanced display features — VRR alongside HDR and explicit sync), Ch22 (Mutter/KWin VRR implementation), Ch29 (MangoHud frame pacing), Ch46 (wp_fifo_v1 pacing on VRR displays), Ch78 (Gamescope VRR on Steam Deck), Ch121 (DRM Lease — VRR timing in direct-to-display VR)

---

## Part VII-A — GPU APIs and Extended Reality

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

---

## Part VII-B — Multimedia Frameworks and Desktop Integration

### Chapter 38: PipeWire and the Video Session Layer
- PipeWire architecture: the graph model (`pw_node`, `pw_port`, `pw_link`); the event loop and `pw_loop`; SPA (Simple Plugin API) as the low-level buffer negotiation layer; the `pw_stream` abstraction for producers and consumers
- Session management: PipeWire as a replacement for PulseAudio and JACK; `wireplumber` session manager; policy engine; device enumeration via udev/ALSA/V4L2; routing rules and metadata
- Video capture pipeline: V4L2 source node; `libcamera` → PipeWire adapter (Ch26); DMA-BUF buffer negotiation with `SPA_DATA_DmaBuf`; zero-copy path from camera ISP to consumer
- Screen capture and portal integration: `xdg-desktop-portal` PipeWire backend; `pw-capture` for compositor screencopy (Ch21, Ch22); OBS Studio as a DMA-BUF consumer; PipeWire's role in the Wayland screen share stack replacing X11 shared memory
- Remote desktop: `xdg-desktop-portal` remote-desktop interface; PipeWire streaming to RDP/VNC backends; DMA-BUF→CPU copy when the remote encoder needs raw pixels
- Buffer formats and GPU interop: `SPA_VIDEO_FORMAT_*` and DRM format mapping; DMA-BUF import into Vulkan `VkImage` (Ch24); explicit sync (`DRM_FORMAT_MOD_INVALID` path vs. explicit fence passing); GPU-accelerated encode handoff to VA-API (Ch26)
- **Integrations**: PipeWire consumes libcamera DMA-BUF frames (Ch26) and compositor screencopy buffers (Ch21, Ch22); its DMA-BUF negotiation relies on GEM/DMA-BUF infrastructure (Ch4); GPU consumers import PipeWire buffers via the same Vulkan external memory path used by ANGLE (Ch34) and Dawn (Ch35); `xdg-desktop-portal` routes screen capture from wlroots (Ch21) and Mutter (Ch22) through PipeWire to OBS and browser tab capture (Ch33)

### Chapter 38b: ALSA — The Linux Audio Subsystem *(Part VII-B)*
- ALSA kernel architecture: `snd_card`, `snd_pcm_ops` vtable, the `sound/core/`, `sound/pci/`, `sound/soc/` source tree; character device layout (`/dev/snd/pcmC0D0p`, `/dev/snd/controlC0`, `/dev/snd/seq`); replaced OSS in Linux 2.6
- `libasound` PCM API: `snd_pcm_open()`, `snd_pcm_hw_params` negotiation (rate, format, channels, period size, buffer size); the interrupt-driven ring buffer model; xrun recovery (`snd_pcm_recover()`); full compilable playback example
- Hardware params negotiation: `snd_pcm_hw_params_set_rate_near()`, `snd_pcm_hw_params_set_period_size_near()`, `snd_pcm_hw_params_set_buffer_size_near()`; the period/buffer size relationship; latency calculation; MMAP mode vs. read/write mode
- ALSA mixer API: `snd_mixer_open()`, `snd_mixer_attach()`, `snd_mixer_selem_*` for volume/mute control; the simple element interface vs. raw control interface; `amixer`/`alsamixer` as reference implementations
- `asound.conf` plugin system: the PCM plugin chain (`plug`, `dmix`, `dsnoop`, `softvol`, `rate`, `asym`); `/etc/asound.conf` vs. `~/.asoundrc`; how PipeWire installs its virtual PCM plugin to intercept `libasound` calls (cross-ref Ch38)
- UCM2 (Use Case Manager): `snd_use_case_mgr_open()`, the `HiFi.conf` verb/device/modifier hierarchy; how systemd/PipeWire/PulseAudio use UCM2 for routing on embedded platforms; `alsaucm` debugging tool
- ASoC (ALSA System-on-Chip): the machine/codec/platform triad; `snd_soc_card`, `snd_soc_dai_link`, `snd_soc_component_driver`; DAPM (Dynamic Audio Power Management) — widgets, routes, and the graph-based power algorithm; registering a machine driver; practical I²S + codec example
- HDA (High Definition Audio): the Intel HDA controller (`snd_hda_intel`); CORB (Command Output Ring Buffer) and RIRB (Response Input Ring Buffer); HDA verb round-trip; codec patching with `snd_hda_patch_t`; `hdajackretask` for pin reconfiguration; the DisplayPort/HDMI audio widget chain (cross-ref Ch140)
- ALSA sequencer and MIDI: `snd_seq_open()`, `snd_seq_port_subscribe_t`; the sequencer graph and MIDI routing; `aseqdump`, `aplaymidi`; the relationship between ALSA MIDI and PipeWire MIDI (Ch38)
- Debugging and tuning: `cat /proc/asound/cards`, `aplay -l`, `alsactl`, `alsa-info.sh`; underrun/overrun diagnosis; `latency_test`; `ALSA_DEBUG=1`; PCM probing with `aplay --dump-hw-params`
- Post-2021 deployment: PipeWire as the ALSA owner on modern desktops; direct `libasound` use on embedded/headless; when to use ALSA directly vs. PipeWire API; the `pw-jack` JACK compatibility shim (Ch38)
- **Integrations**: Ch38 (PipeWire — owns ALSA devices on modern desktops; ALSA PCM plugin routes libasound calls into PipeWire graph), Ch140 (HDMI/DisplayPort Audio — HDA codec widget chain for HDMI audio), Ch39 (Qt Multimedia and GStreamer-GTK audio capture use ALSA or PipeWire-ALSA bridge), Ch57/Ch58 (FFmpeg/GStreamer ALSA source/sink elements), Ch26 (VA-API video decode outputs PCM via ALSA or PipeWire)

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

### Chapter 111: Flatpak Graphics — GPU Access in Sandboxed Applications *(Part VII-B)*

- Flatpak sandbox model: Linux namespaces (user, mount, network, PID); seccomp filter; D-Bus policy; the `/run/host` and `/run/flatpak` bind mounts; how Flatpak differs from a container runtime (no kernel isolation of GPU, only file system and syscall restrictions)
- GPU device access in Flatpak: the `--device=dri` finish-arg granting access to `/dev/dri/renderDN` and `/dev/dri/cardN`; why DRM render nodes do not require DRM master privilege; GPU driver library injection via `LD_LIBRARY_PATH` and the Mesa ICD discovery path; `org.freedesktop.platform` runtime providing Mesa shared libraries
- xdg-desktop-portal as the privileged broker: `org.freedesktop.portal.OpenURI`, `org.freedesktop.portal.FileChooser`, `org.freedesktop.portal.ScreenCast`; how Flatpak apps access display and camera without direct Wayland compositor access; `wp_security_context_v1` (Ch46) tagging Flatpak connections with app IDs at the compositor level
- GPU compute in Flatpak: OpenCL (rusticl/Mesa) and Vulkan compute from inside the sandbox; `--device=dri` sufficient for render node compute; CUDA and ROCm require `/dev/nvidia*` and `/dev/kfd` which need explicit grants or CDI (Container Device Interface) via NVIDIA Container Toolkit (Ch55); OpenVINO NPU access (`/dev/accel/accel0`) similarly requires explicit device grants
- Wayland surface creation from Flatpak: EGL via `libEGL.so.1` from the runtime; `wl_display` connection via the socket forwarded into the sandbox (`$WAYLAND_DISPLAY`); linux-dmabuf buffer negotiation proceeds normally; `xdg_toplevel` surface roles; explicit sync handshake (Ch75) transparent to the app
- Application GPU performance inside Flatpak: shader cache (`~/.cache/mesa_shader_cache` inside the sandbox; `XDG_CACHE_HOME` respects the portal); disk shader cache invalidation on Mesa version update; `LIBGL_DEBUG=verbose` from inside the sandbox; `flatpak run --env=MESA_DEBUG=1`
- **Integrations**: Ch1 (DRM render nodes — the access mechanism), Ch23 (xdg-desktop-portal — screen capture and camera portals), Ch46 (wp_security_context_v1 — compositor-side Flatpak tagging), Ch55 (GPU containers — contrast with container GPU access), Ch25 (Vulkan/OpenCL compute from sandbox), Ch88 (NPU accel node access from sandbox)

### Chapter 114: OpenCV and GPU-Accelerated Computer Vision on Linux *(Part VII-B)*

- OpenCV architecture on Linux: `cv::Mat` and `cv::UMat` (unified transparent API); build configuration: `cmake -DWITH_OPENCL=ON -DWITH_CUDA=ON -DWITH_VA=ON -DWITH_VA_INTEL=ON`; module structure (`core`, `imgproc`, `dnn`, `videoio`, `calib3d`, `features2d`); source at https://github.com/opencv/opencv
- OpenCL T-API (Transparent API): `cv::UMat` keeps data on GPU via OpenCL; `cv::ocl::Context` initialisation; automatic fallback to CPU when no OpenCL device; `OPENCV_OPENCL_DEVICE` environment variable; rusticl (Ch25) as the Mesa OpenCL backend for AMD/Intel on Linux
- CUDA backend: `cv::cuda::GpuMat`; `cv::cuda::cvtColor`, `cv::cuda::resize`, `cv::cuda::threshold`; stream-based async dispatch with `cv::cuda::Stream`; CUDA modules (`cuda_imgproc`, `cuda_features2d`, `cuda_stereo`); OpenCV CUDA on Linux: CUDA toolkit requirement, `CUDA_ARCH_BIN` build flag
- OpenCV DNN module and GPU inference: `cv::dnn::readNetFromONNX` / `readNetFromTensorflow`; `net.setPreferableBackend(cv::dnn::DNN_BACKEND_CUDA)` and `DNN_TARGET_CUDA_FP16`; OpenVINO backend (`DNN_BACKEND_INFERENCE_ENGINE`) for Intel NPU/GPU; ONNX model deployment workflow from training to `cv::dnn` inference
- VA-API integration: `cv::VideoCapture` with VA-API decode (`CAP_PROP_HW_ACCELERATION`, `VIDEO_ACCELERATION_VAAPI`); `cv::VideoWriter` VA-API encode; `cv::UMat` ↔ `VASurface` interop via the `va_intel` module; zero-copy DMA-BUF path from decode to processing to encode
- Practical pipeline: camera capture (V4L2 / libcamera via `cv::VideoCapture`) → GPU resize/colour-convert → DNN inference → annotate → VA-API encode → GStreamer sink; latency analysis
- **Integrations**: Ch25 (OpenCL compute — T-API uses rusticl/ROCm OpenCL), Ch26 (VA-API — hardware decode/encode interop), Ch59 (NVIDIA DeepStream — contrast with OpenCV DNN for GPU inference pipelines), Ch88 (NPU — OpenVINO backend for Intel NPU), Ch96 (libcamera — camera source feeding OpenCV pipeline)

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

### Chapter 104: DXVK and VKD3D-Proton — D3D-to-Vulkan Translation *(Part VIII)*

- DXVK architecture in depth: D3D9/10/11 state machine mapped to Vulkan; `HudRenderer`, `D3D9Context`, `D3D11Context`; the `DxvkDevice` and `DxvkContext` core objects; async pipeline compilation (`DXVK_ASYNC`); the `dxvk.conf` tunable configuration system; source at https://github.com/doitsujin/dxvk
- VKD3D-Proton divergence: the fork from upstream vkd3d focused on game performance; `d3d12_device`, `d3d12_command_list`, `d3d12_pipeline_state`; bindless resource heap (`D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV`); DXR (DirectX Raytracing) via `VK_KHR_ray_tracing_pipeline` and `VK_KHR_acceleration_structure`; source at https://github.com/HansKristian-Work/vkd3d-proton
- Shader translation pipeline: DXBC/DXIL → SPIR-V via vkd3d-shader; how translated SPIR-V enters Mesa NIR (Ch14) and then ACO or LLVM for codegen; the shader cache and pipeline library path
- Proton integration: how Steam's Proton runtime selects and bundles DXVK + VKD3D-Proton; `pressure-vessel` container; Steam Runtime SDK; shader pre-caching with `PROTON_LOG` and `steam shader pre-cache`
- Game compatibility and performance tuning: `DXVK_HUD` overlay; `RADV_PERFTEST`, `VKD3D_CONFIG` knobs; async compute queues for VKD3D-Proton; frame pacing with gamescope (Ch78)
- **Integrations**: Ch14 (NIR — DXBC/DXIL SPIR-V enters via spirv_to_nir), Ch15 (ACO — primary codegen backend for DXVK/VKD3D on AMD), Ch18 (Mesa Vulkan drivers — RADV/ANV/NVK as the Vulkan backend), Ch28 (Ch28 covers Wine and Proton; this chapter deepens the DXVK/VKD3D internals), Ch56 (DXR ray tracing path via VKD3D-Proton), Ch78 (Gamescope — Steam Deck deployment)

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
- **Integrations**: dEQP-VK tests every Mesa Vulkan driver (Ch18) and NVK (Ch10b) against the Vulkan spec; GLES CTS tests radeonsi, iris, and Zink (Ch19); piglit tests run on software renderers (Ch17) in headless CI where no GPU is available; conformance failures frequently trace back to NIR optimisation pass bugs (Ch14) or ACO register allocation issues (Ch15); the WebGL CTS tests ANGLE (Ch34) and the WebGPU CTS tests Dawn (Ch35), both of which run against the same Mesa Vulkan and GL drivers, making browser conformance suites a second CI dimension over the same driver stack

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

### Chapter 107: Headless Rendering — Offscreen, CI, and Server Workloads *(Part IX)*

- Headless rendering modes: EGL `EGL_PLATFORM_SURFACELESS_MESA` (surfaceless, no window system required); `EGL_PLATFORM_GBM_KHR` with a GBM device from a DRM render node; DRM render-only nodes (`/dev/dri/renderDN`) with no display output; VKMS (Virtual KMS, `drivers/gpu/drm/vkms/`) as a software KMS target for CI without physical display hardware
- Vulkan headless rendering: `VK_EXT_headless_surface` — `vkCreateHeadlessSurfaceEXT` with `VkSwapchainKHR` for offscreen present; alternatively, render to `VkImage` with `VK_IMAGE_USAGE_TRANSFER_SRC_BIT` and readback via `vkCmdCopyImageToBuffer`; `VkPhysicalDeviceType` selection for CI (prefer `VK_PHYSICAL_DEVICE_TYPE_VIRTUAL_GPU` or `INTEGRATED_GPU` in VM environments)
- Software renderers for headless CI: llvmpipe (Ch17) via `LIBGL_ALWAYS_SOFTWARE=1`; Lavapipe via `VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/lvp_icd.x86_64.json`; VKMS + `weston --backend=drm --drm-device=vkms`; performance expectations — llvmpipe benchmarked at ~10% of hardware GPU throughput for rasterisation tasks
- CI screenshot and image comparison: `glReadPixels` / `vkCmdCopyImageToBuffer` → PNG via `stb_image_write`; perceptual hash (`phash`) comparison for GPU-output regression testing; `imagemagick compare -metric PSNR` for quantitative differences; piglit (Ch31) uses this pattern extensively for per-test golden image comparison
- Mesa CI infrastructure: FDO (`gitlab.freedesktop.org/mesa/mesa`) GitLab CI pipeline; `deqp-runner` (Ch31) for parallel CTS across GPU farm; LAVA (Linaro Automation and Validation Architecture) for board-based CI; Docker images with Mesa built from source; `MESA_LOADER_DRIVER_OVERRIDE` for driver selection; CI for llvmpipe/lavapipe on x86 nodes, hardware GPU tests on dedicated runners
- Server-side GPU rendering: compositing server (no display, render to EGL images, serve via WebRTC or VA-API encode); cloud game streaming (render on GPU server → H.264/AV1 encode via VA-API → WebRTC delivery, Ch60b); NVIDIA Container Toolkit (Ch55) for headless cloud GPU; `EGL_DEVICE_EXT` enumeration for selecting the correct GPU in multi-GPU server environments
- **Integrations**: Ch1 (DRM render nodes — headless access model), Ch17 (llvmpipe/Lavapipe — software renderers for CI), Ch26 (VA-API encode — server-side streaming from rendered frames), Ch31 (piglit/dEQP — headless conformance testing), Ch55 (GPU containers — headless cloud compute), Ch60b (adaptive streaming — WebRTC delivery of server-rendered frames)

### Chapter 109: Mesa Testing — piglit, dEQP, and Continuous Integration *(Part IX)*

- Mesa CI architecture: GitLab CI on `gitlab.freedesktop.org/mesa/mesa`; stages (build → test → performance); Docker-in-Docker for Mesa build containers; GPU farm node configuration; `.gitlab-ci.yml` structure and per-driver job matrices; `ci-scripts/` and `src/ci/` helper scripts
- piglit test suite: `piglit run -j8 gpu/mesa`; test types: GLSL conformance, fixed-function GL, extension coverage, performance microbenchmarks; `piglit summary html`; `piglit compare` for regression bisection; `PIGLIT_PLATFORM=gbm` for headless DRM; source at https://gitlab.freedesktop.org/mesa/piglit
- dEQP (drawElements Quality Program): `deqp-vk`, `deqp-gles2`, `deqp-gles3`, `deqp-egl`; test case hierarchy (`dEQP-VK.pipeline.*, dEQP-GLES3.functional.*`); `--deqp-caselist-file` for subset runs; the `deqp-runner` harness (Ch31) for CI parallelism; Khronos Vulkan CTS and OpenGL ES CTS as authoritative subsets
- Shader regression testing: `shader-db` (Mesa's GLSL shader performance database); `mesa-shader-db` CI job compares instruction counts before/after MR; ACO-specific `aco_stats` pass counting; `glsl-to-tgsi`, `nir-to-rdna` compile-time benchmarks; preventing accidental regressions in ACO/NIR optimisation passes (Ch14, Ch15)
- Continuous Integration workflow: MR pipeline triggers; `force-rebuild` labels; `needs:` dependency graph in `.gitlab-ci.yml`; pipeline artifacts (test logs, performance CSVs); Marge-Bot for automatic merge after CI green; how a NIR patch author should run the CI locally with `meson test -C build/`
- GPU hardware test runners: LAVA board farm for ARM (Panfrost, V3D, etnaviv); x86 GPU runners (RADV/ANV/NVK); bare-metal vs containerised runners; `deqp-runner suite` expectation `.toml` files; flake rate tracking and automatic retry logic; known-failure lists and how to update them
- **Integrations**: Ch14 (NIR — shader-db tracks NIR pass regressions), Ch15 (ACO — aco_stats CI tracking), Ch17 (llvmpipe/Lavapipe — CI targets for software renderer conformance), Ch31 (piglit and dEQP — this chapter deepens the CI infrastructure behind Ch31's test suite overview), Ch107 (Headless Rendering — CI uses headless EGL/DRM)

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

### Chapter 195: Browser Image Formats — Decode Pipelines, Compression Mechanisms, and HDR

- The `<img>` decode pipeline in Chrome: resource loader, codec selection via `SkCodec`, incremental decode, `SkBitmap` production, colour space conversion via skcms
- GPU upload path: `SkBitmap` → `gpu::SharedImage` → `VkImage` staging buffer upload → `viz::TextureDrawQuad`
- Format negotiation: HTTP `Accept` header and `<picture>`/`srcset`
- JPEG: DCT, quantisation, progressive decode, libjpeg-turbo SIMD acceleration
- PNG: DEFLATE/LZ77, filter prediction, libpng, 16-bit per channel
- WebP: VP8 (lossy), VP8L (lossless), extended container (alpha, animation, ICC), libwebp
- AVIF: AV1 intra-frame coding, block partitioning, 10/12-bit HDR, CLLI/MDCV HDR metadata, libavif + dav1d
- JPEG XL: VarDCT and Modular modes, lossless JPEG recompression, libjxl, Chrome/Safari support
- GIF: LZW, palette animation, migration to animated WebP/AVIF
- SVG: XML/DOM rasterisation via Skia
- ICC profiles and colour management: skcms, NCLX codes, AVIF HDR → `wp_color_management_v1` → KMS `drm_hdr_output_metadata`
- `ImageDecoder` API (WebCodecs): frame-level access, `VideoFrame` output, GPU import via `importExternalTexture`
- Format comparison table and decision guide
- **Integrations**: AVIF/JXL HDR metadata connects to the KMS HDR output pipeline (Ch74) and `wp_color_management_v1` (Ch194 §5); `ImageDecoder` output as `VideoFrame` is the same type WebCodecs produces (Ch146); all raster decode uses `SkCodec` inside Skia (Ch37); decoded textures become `viz::TextureDrawQuad` references in the Viz compositor (Ch36); AVIF hardware decode would use the same VA-API / Vulkan Video path as Ch147

### Chapter 203: WebXR — Browser-Based Immersive Experiences on Linux

- WebXR session types: `inline` (Magic Window), `immersive-vr`, `immersive-ar`; feature negotiation via `requiredFeatures`/`optionalFeatures` mapping onto OpenXR extension availability
- The JS frame loop: `XRSession.requestAnimationFrame` at device native rate; `XRFrame`, `XRViewerPose`, `XRView` (projectionMatrix, transform, eye); frame object lifetime constraints
- Reference spaces: `viewer`, `local`, `local-floor`, `bounded-floor`, `unbounded` and their `XrReferenceSpaceType` OpenXR equivalents
- Input sources: `XRInputSource` (targetRayMode, gripSpace, Gamepad), interaction profile strings → OpenXR profile paths (`GetOpenXrInputProfilesMap`); `XRHand` — 25 named `XRJointSpace` entries, `fillJointRadii`, `fillPoses` batch APIs
- Rendering: `XRWebGLLayer` (UA-managed framebuffer, `framebufferScaleFactor`, `gl.makeXRCompatible()`); WebXR Layers API — `XRWebGLBinding`, `XRProjectionLayer`, `getViewSubImage`, `texture-array` for single-pass stereo
- Chromium multi-process model: renderer → browser → `isolated_xr_device` service; Mojo interfaces `VRService`, `XRDevice`, `XRFrameDataProvider`, `XRPresentationProvider`; `XRFrameData` struct carrying pose, views, input, hand joints, planes, anchors, depth
- OpenXR backend class hierarchy: `OpenXrDevice` → `OpenXrRenderLoop` → `OpenXrApiWrapper`; `OpenXrExtensionHelper`; `OpenXrInputHelper` with action sets and interaction profiles; `OpenXrHandTracker` for `XR_EXT_hand_tracking`
- High-level WebXR frameworks: **A-Frame** (declarative ECS on Three.js; `<a-scene>`, `<a-entity>`, component `tick()`; `laser-controls`/`hand-controls` wrapping `XRInputSource`; rendering via `THREE.WebGLRenderer.xr`); **Babylon.js** (`WebXRExperienceHelper`, `WebXRDefaultExperience`, `WebXRCamera`; dual `WebGLEngine`/`WebGPUEngine` backends; `WebXRHandTracking` feature module); **Three.js VRButton/ARButton** (`WebXRManager`, `renderer.xr.setSession()`); call chain on Linux: A-Frame → Three.js → ANGLE Vulkan → Mesa → DRM; framework comparison table (API style, WebGPU support, hand tracking, bundle size); Linux performance profiling points (DevTools, `chrome://tracing`, Mesa overlay, Monado `XRT_PRINT_PERFORMANCE`)
- Linux status: officially unsupported (only D3D11 and OpenGLES graphics bindings exist); community `webxr-linux` patch adds `OpenXrGraphicsBindingVulkan` (`XR_KHR_vulkan_enable2`) sharing swapchain images via `VkExternalMemoryImageCreateInfo`; feature compatibility table with Monado and SteamVR
- Firefox WebXR via `wgpu`: Vulkan backend creates `XrGraphicsBindingVulkanKHR` naturally; `immersive-vr` works on Linux with SteamVR/Monado; `dom.webxr.hands.enabled` flag
- WebXR vs. native OpenXR tradeoffs: Mojo IPC latency overhead (~1–2 ms/frame), no raw camera access, restricted extension surface; `XRProjectionLayer` reduces copy overhead; forthcoming WebGPU `GPULayer` via Dawn
- **Integrations**: OpenXR runtime (Ch27) — `XrSession` in Monado/SteamVR is the common substrate; ANGLE (Ch34) is the WebGL backend for `XRWebGLLayer`; Dawn (Ch35) will back the forthcoming `XRWebGPUBinding`; Chromium compositor (Ch36) shares `gpu::SharedImage` mailboxes with the XR swapchain; Firefox/WebRender (Ch52) shares `wgpu` Vulkan device with the XR frame loop; Android ARCore WebXR (Ch87); WebNN (Ch168) for in-browser ML inference augmenting XR tracking

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

### Chapter 60c: WebRTC Server Infrastructure on Linux

- **Why servers are needed**: NAT traversal (TURN relay), media routing (SFU vs MCU vs cascaded SFU), recording, signalling at scale — peer-to-peer WebRTC topology fails beyond 4–6 participants; SFU topology overview
- **coturn (STUN/TURN server)**: `turnserver.conf` — realm, listening ports (3478 UDP/TCP, 5349 TLS), relay IP, fingerprint, `lt-cred-mech`; SQLite/PostgreSQL/Redis auth backends; RFC 5766 TURN allocation lifecycle (Allocate → CreatePermission → ChannelBind → Send/Data); systemd unit; iptables relay port range; capacity planning (RAM per allocation, relay bandwidth); TLS with Let's Encrypt; Prometheus metrics via native `/metrics` endpoint (port 9641, `prom`+`microhttpd` compile-time option); `net.core.rmem_max`/`wmem_max` tuning
- **Janus WebRTC Gateway**: C plugin architecture; `janus_plugin_t` vtable (`init`, `destroy`, `handle_message`, `setup_media`, `incoming_rtp(janus_plugin_session*, janus_plugin_rtp*)`, `incoming_rtcp`, `incoming_data`); VideoRoom SFU plugin (publisher/subscriber model, 3 concurrent publishers default, `janus_videoroom_publisher` struct); EchoTest, AudioBridge, RecordPlay plugins; transport plugins (WebSocket, HTTP REST); VideoRoom signalling flow (JSON-over-WebSocket: `create` → `join` publisher → `configure` JSEP offer → `subscribe`); meson build, libnice ≥ 0.1.18 dependency
- **mediasoup**: Node.js/TypeScript API + C++ Worker subprocess (Unix pipe, FlatBuffers protocol); Router → Transport → Producer → Consumer object model; `WebRtcTransport` (ICE+DTLS), `PlainTransport` (RTP/SRTP, GStreamer/FFmpeg ingest), `PipeTransport` (Worker cascade); simulcast `setPreferredLayers()` / Consumer score / `layerschange`; Node.js code: `createWorker()`, `createRouter()`, `createWebRtcTransport()`, `produce()`, `consume()`; GStreamer PlainTransport ingest pipeline; FFmpeg RTP recording
- **LiveKit**: Pion-based Go SFU; Room → Participant → Track model; `livekit.yaml` configuration (ports, Redis, `rtc.port_range`, `rtc.use_ice_lite`, `rtc.nat_1to1_ips`); `livekit-ingress` service for WHIP ingest (`ingress.yaml`); LiveKit Egress — `RoomCompositeEgressRequest` spawns Chromium headless + GStreamer pipeline → MP4/WebM; Docker Compose deployment
- **Pion (Go WebRTC library)**: `pion/webrtc`, `pion/interceptor`, `pion/rtp`, `pion/rtcp` packages; SFU forwarding loop in Go (PeerConnection, `AddTrack`, `OnTrack`, `ReadRTP`, goroutine-per-subscriber write); NACK and TWCC via `interceptor.Registry`; `sync.Pool` for RTP packet buffer reuse; goroutine-per-connection model and low-GC-pause design
- **WHIP/WHEP signalling** (RFC 9725 / draft): WHIP HTTP POST `/whip` with SDP offer body → 201 Created + SDP answer + `Location` header; trickle ICE via HTTP PATCH (`application/trickle-ice-sdpfrag`); DELETE to terminate; WHEP `draft-ietf-wish-whep` egress mirror; GStreamer `whipclientsink` (gst-plugins-rs, replaces deprecated `whipsink`); `whepsrc`; OBS ≥ 30 native WHIP output; OvenMediaEngine WHIP/WHEP endpoint
- **SFU internals**: simulcast (3 SSRCs: low/medium/high spatial layers, SFU selects by subscriber bandwidth); VP9/AV1 SVC temporal+spatial layer dropping; Transport-CC (`draft-holmer-rmcat-transport-wide-cc-extensions-01`) — per-packet arrival timestamp collection → RTCP feedback → sender GoogCC BWE; PLI/FIR keyframe requests when new subscriber joins or loss threshold exceeded; layer selection hysteresis to avoid oscillation
- **Recording pipelines**: Janus RecordPlay `.mjr` binary format (RTP dump + timestamps); `janus-pp-rec` → WebM/MP4 post-session conversion; mediasoup `PlainTransport` → GStreamer `udpsrc → rtph264depay → mp4mux → filesink`; FFmpeg `ffmpeg -i rtp://127.0.0.1:5004 -c copy recording.mp4`; LiveKit Egress composite recording (Chromium headless + GStreamer); PCAP fallback (`tcpdump -w capture.pcap udp port 10000:60000`)
- **Linux deployment**: systemd units for coturn/Janus/LiveKit with `PrivateNetwork=no`, `AmbientCapabilities=CAP_NET_BIND_SERVICE`; Prometheus scrape config; capacity planning (TURN: N participants × bitrate relay throughput; SFU: N×M forwarded streams, ~100 streams/core at 1 Mbps); `net.core.rmem_max=26214400`, `nf_conntrack_max` increase; ICE `nat1to1` public IP override; ICE-TCP port 443 as firewall fallback; health check endpoints; TLS architecture (coturn terminates TURN TLS; DTLS per-media-connection, cannot be nginx-terminated)
- **Integrations**: Ch60b (WebRTC protocol stack — ICE, DTLS-SRTP, RTP/RTCP, GoogCC — not repeated here); Ch57 (FFmpeg RTP recording from PlainTransport); Ch58 (GStreamer `webrtcbin`, `whipclientsink`, `whepsrc`, `udpsrc` RTP ingest); Ch38 (PipeWire screen capture → GStreamer → WHIP publish path); Ch59 (DeepStream RTSP camera ingest → mediasoup PlainTransport); Ch123 (desktop screen capture streaming via WHIP to SFU)

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

### Chapter 110: SPIR-V Tooling — spirv-tools, SPIRV-Cross, and the Shader Ecosystem *(Part XIV)*

- SPIRV-Tools toolkit deep dive: `spirv-as` (text assembler); `spirv-dis` (binary disassembler, human-readable output); `spirv-val` (spec-compliant validator — cites specification section numbers in errors); `spirv-opt` optimisation passes in detail (`--eliminate-dead-code-aggressive`, `--inline-entry-points-exhaustive`, `--loop-unroll`, `--convert-local-multi-store`, `--reduce-load-size`); `spirv-link` (linking separately compiled modules using the `Linkage` capability); source at https://github.com/KhronosGroup/SPIRV-Tools
- `spirv-opt` pass analysis and practical guidance: which passes help vs. hurt in GPU shader pipelines; interaction with Mesa NIR lowering (Ch14) — passes that are redundant because NIR does equivalent work; benchmarking `spirv-opt` preprocessing time against total ACO compile time (Ch15); `--target-env vulkan1.3` for environment-specific optimisation; debug info preservation (`--preserve-bindings`, `--preserve-spec-constants`)
- SPIRV-Cross in depth: SPIR-V → GLSL, HLSL, MSL, JSON cross-compilation; the reflection API (`spirv_cross::ShaderResources`) for descriptor binding introspection; `build_combined_image_samplers()` for GLSL combined-image-sampler lowering; usage in ANGLE (Ch34), DXVK (Ch28), MoltenVK, and bgfx (Ch84); `--msl-version 20300` for Metal 3 target; source at https://github.com/KhronosGroup/SPIRV-Cross
- SPIRV-Reflect: `SpvReflectShaderModule`; automatic `VkDescriptorSetLayout` and `VkPipelineLayout` generation from SPIR-V reflection; integration with vk-bootstrap (Ch82) for zero-boilerplate descriptor layout; input/output variable reflection for vertex attribute validation
- glslang and shaderc: `glslang` as the reference GLSL → SPIR-V front end; `shaderc` wrapping glslang with a simpler C API; `shaderc_compile_options_set_target_spirv` and `shaderc_compile_options_add_macro_definition`; runtime GLSL compilation vs. offline SPIR-V; integration in Qt (Ch39), Bevy (Ch40), and bgfx (Ch84)
- SPIR-V debugging extensions: `NonSemantic.Shader.DebugInfo.100` (`DebugSource`, `DebugLine`, `DebugLocalVariable`, `DebugScope`); `VK_EXT_debug_utils` + `OpLine` for source-level shader debugging in RenderDoc (Ch30); `--generate-debug-info` glslang flag; `spirv-val` error taxonomy (ID out-of-bounds, unsupported capability, missing decoration)
- **Integrations**: Ch14 (NIR — SPIR-V enters Mesa via `spirv_to_nir`), Ch15 (ACO — SPIRV-Tools `spirv-opt` as optional preprocessing before ACO), Ch28 (DXVK — SPIRV-Cross cross-compiles DXBC shaders), Ch30 (RenderDoc — SPIR-V debug info for source-level debugging), Ch34 (ANGLE — shaderc for GLSL ES → SPIR-V), Ch61 (Ch61 covers SPIR-V module structure and front ends; this chapter focuses on the toolchain for authoring, optimisation, and cross-compilation), Ch117 (Slang — produces SPIR-V consumed by the toolchain described here)

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

### Chapter 113: CGAL and Computational Geometry on the Linux Graphics Stack *(Part XVIII)*

- CGAL overview: the Computational Geometry Algorithms Library; C++17 template-heavy design; `CGAL::Surface_mesh`, `CGAL::Polyhedron_3`, `CGAL::Nef_polyhedron_3`; half-edge data structure; CMake integration; https://www.cgal.org/; LGPL/GPL dual-licensed
- Mesh processing algorithms: `CGAL::Polygon_mesh_processing` — isotropic remeshing, hole filling, Boolean operations on meshes (`corefine_and_compute_union`), self-intersection removal, smoothing; `CGAL::Surface_mesh_simplification` — Garland-Heckbert QEM decimation; practical use in Blender mesh cleanup pipelines (Ch42) and glTF preprocessing (Ch64)
- Computational geometry foundations relevant to graphics: convex hull (`CGAL::convex_hull_3`); Delaunay triangulation (`CGAL::Delaunay_triangulation_3`); Voronoi diagrams; alpha shapes for point cloud surface reconstruction; AABB tree (`CGAL::AABB_tree`) for BVH-style ray intersection and closest-point queries — the CPU counterpart to GPU BVH (Ch56)
- Implicit surfaces and SDF: `CGAL::Implicit_surface_3`; signed distance functions as inputs; marching cubes (`CGAL::make_mesh_3` with implicit domain); NeRF/Gaussian Splatting mesh extraction (Ch115) as a practical consumer of this pipeline
- GPU acceleration of geometry algorithms: CGAL is CPU-only; practical GPU counterparts — thrust/CUB for parallel geometry queries on CUDA (Ch66); `VK_EXT_mesh_shader` for procedural geometry generation in Vulkan (Ch76); the workflow of preprocessing heavy CGAL computation offline and passing results to GPU render pipelines
- Integration into the graphics pipeline: CGAL mesh → `VkBuffer` vertex data upload (Ch24); mesh simplification for LOD (level of detail) matching Nanite cluster hierarchy (Ch97); collision mesh generation for physics (PhysX 5 Ch69 §12); Boolean CSG operations for CAD applications using Filament (Ch83) or bgfx (Ch84)
- **Integrations**: Ch24 (Vulkan buffer upload of CGAL mesh output), Ch42 (Blender — mesh import/export and preprocessing), Ch56 (AABB tree — CPU counterpart to GPU BVH for ray tracing), Ch64 (glTF — CGAL-processed meshes exported as glTF assets), Ch83 (Filament — consumer of CGAL-preprocessed geometry), Ch115 (NeRFStudio — mesh extraction from Gaussian Splat uses marching cubes/CGAL)

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

### Chapter 87: Android AR: ARCore Architecture, Camera HAL Integration, and the Android XR Platform

- ARCore architecture: application-layer SDK (Play Services `com.google.ar.core`) above Camera HAL; three pillars: motion tracking (VIO), environment understanding (planes/depth), light estimation; minimum API 24 (Android 7.0); AR Required vs AR Optional manifest flags; ARCore vs ARKit comparison
- Android Camera Pipeline and ARCore: Camera HAL3 (`camera_device3_ops_t`, `camera3_stream_t`, `process_capture_request`); Camera2 `CameraDevice.createCaptureSession()`, `CaptureRequest.Builder`, `TEMPLATE_RECORD`; IMU via `SensorEventListener` (`TYPE_ACCELEROMETER`, `TYPE_GYROSCOPE`); Visual-Inertial Odometry (VIO) factor graph / EKF; `ArCamera_getPose()` vs `ArCamera_getDisplayOrientedPose()`
- Session lifecycle: C API `ArSession_create/resume/pause/destroy`; `ArSession_update()` → `ArFrame`; `ArFrame_acquireCamera()`; `ArPose` 7-element float (quaternion + translation); `ArSession_setCameraTextureName()` for `GL_TEXTURE_EXTERNAL_OES`; `ArConfig` — focus mode, update mode, depth mode, plane finding mode, light estimation mode
- AR rendering pipeline: camera image as `AHardwareBuffer` → `GL_TEXTURE_EXTERNAL_OES` via `EGLImage` (`EGL_KHR_image_base` + `GL_OES_EGL_image_external`); background full-screen quad with `samplerExternalOES`; view/projection from `ArCamera_getProjectionMatrix/getViewMatrix`; depth occlusion in fragment shader; Vulkan path: `VK_ANDROID_external_memory_android_hardware_buffer`, `vkGetAndroidHardwareBufferPropertiesANDROID`, `VkSamplerYcbcrConversion`
- Plane detection: `ArSession_getAllTrackables(AR_TRACKABLE_PLANE)`; `AR_PLANE_HORIZONTAL_UPWARD/DOWNWARD_FACING`, `AR_PLANE_VERTICAL`; `ArPlane_getPolygon()`; subsume: `ArPlane_acquireSubsumedBy()`; point clouds: `ArFrame_acquirePointCloud()`, `ArPointCloud_getData()` float[4] per point; Scene Semantics API (Android 12+): `ArSemanticMode_ENABLED`, `ArFrame_acquireSemanticImage()`; Instant Placement
- Depth API: three sources — stereo camera (Pixel dToF), smooth depth (temporal filter), MotionStereo (parallax on single camera); `ArDepthMode` enum; `ArFrame_acquireDepthImage16Bits()` and `acquireRawDepthImage16Bits()` (16-bit millimeter); confidence image `acquireRawDepthConfidenceImage()`; typical resolution 160×120–240×180; occlusion GLSL pattern
- Anchors and Cloud Anchors: `ArSession_acquireNewAnchor()`; hit-testing `ArFrame_hitTest()` → `ArHitResultList`; anchor tracking states; Persistent/Geospatial anchors via VPS (Visual Positioning System); `ArEarth_acquireNewAnchor()`; Cloud Anchors: `ArSession_hostCloudAnchorAsync()`, `resolveCloudAnchorAsync()`; `ArFeatureMapQuality`; Cloud Anchor TTL
- Light estimation: `ArLightEstimateMode_AMBIENT_INTENSITY` (lux + color correction) vs `ENVIRONMENTAL_HDR` (full spherical HDR cubemap); `ArLightEstimate_getEnvironmentalHdrMainLightDirection/Intensity()`; `ArLightEstimate_acquireEnvironmentalHdrCubemap()` → `ArImageCubemap` (6 `ArImage*`); IBL split-sum PBR integration
- OpenXR on Android and Android XR: ARCore OpenXR loader; extensions `XR_EXT_hand_tracking`, `XR_KHR_android_surface_swapchain`, `XR_ANDROID_trackables`; Android XR platform (Samsung Galaxy headset, 2025); Jetpack XR (`androidx.xr.*`): `Session.create()`, `Entity`, `PanelEntity`, `GltfModelEntity`, `SpatialCapabilities`; Project Moohan reference hardware
- Recording and Playback: `ArRecordingConfig`, `ArSession_startRecording()` to MP4 with custom tracks; `ArSession_startPlayback()`; `ArDataset` for offline pipelines; deterministic CI/test replay
- Performance and power: VIO tracking one dedicated CPU core; camera preview 30 fps 640×480 or 1280×720; thermal throttling effect on `ArSession_update()` latency; `Choreographer`/`FrameRateCompatibility` for VSYNC alignment; AR tracking thread decoupled from render thread; AHardwareBuffer zero-copy from HAL → gralloc → ARCore → app
- **Integrations**: Ch85 (SurfaceFlinger buffer pipeline for AR overlays), Ch86 (Vulkan AHardwareBuffer import, YCbCr conversion), Ch27 (OpenXR on Linux/Monado — compare to ARCore OpenXR), Ch26 (Camera HAL / V4L2 — Linux-side equivalent), Ch6 (ARM Mali/Adreno driver powering ARCore rendering), Ch24 (Vulkan EGL interop analogous to AHardwareBuffer Vulkan path)

### Chapter 161: Android Game Development Kit (AGDK): Native Game Architecture, Input, Audio, and Frame Pacing

- AGDK component overview: `game-activity`, `game-text-input`, `games-controller`/Paddleboat, `games-frame-pacing`/Swappy, `oboe`, `games-performance`/Android Performance Tuner, `games-memory-advice`; Gradle Prefab AAR distribution; CMake `find_package()` integration
- NativeActivity and android_native_app_glue: `android_app` struct layout; background pthread model; `ALooper` pipe (LOOPER_ID_MAIN/INPUT/USER); `ANativeActivityCallbacks` 16 fields; `AInputQueue` event loop; NativeActivity limitations (broken text input, no SurfaceView, coarse batching)
- GameActivity: Kotlin subclass requirement; AGDK `android_app` struct differences (`GameActivity*`, input arrays replacing `onInputEvent`); CMake integration and linker entry-point change; `APP_CMD_INIT_WINDOW` as canonical Vulkan swapchain creation hook
- Input: `android_app_swap_input_buffers()` double-buffered model; `GameActivityMotionEvent` historical samples; per-pointer axis access via `GameActivityPointerAxes_getAxisValue()`; event filters and axis enable; `GameTextInput` IME bridge (`GameActivity_setImeEditorInfo`, `GameActivity_getTextInputState`)
- Paddleboat: controller enumeration and mapping; `Paddleboat_init/update`; `processGameActivityMotionInputEvent` filter pattern; `Paddleboat_Controller_Data` struct (buttons, sticks, triggers); vibration (`Paddleboat_Vibration_Data`); connect/disconnect callbacks; `getActiveAxisMask()` for axis registration
- Swappy frame pacing: double-present jank and buffer-stuffing problems; `SwappyVk_determineDeviceExtensions` → `initAndGetRefreshCycleDuration` → `setWindow` → `setSwapIntervalNS` → `queuePresent` → `destroySwapchain` sequence; `VK_GOOGLE_display_timing` injection; Choreographer vsync calibration; pipeline/non-pipeline/auto modes; multi-refresh-rate auto-selection; `SwappyGL_swap` for GLES apps
- ANativeWindow: producer end of BufferQueue / Surface vtable; `acquire/release` reference counting; `setBuffersGeometry` and format constants; Vulkan `vkCreateAndroidSurfaceKHR` integration; EGL `eglCreateWindowSurface`; CPU `lock/unlockAndPost` software path
- Oboe audio: OpenSL ES vs AAudio latency history; `AudioStreamBuilder` with `PerformanceMode::LowLatency` + `SharingMode::Exclusive`; MMAP path on AAudio; `AudioStreamCallback::onAudioReady` callback model; input (microphone) stream; stream restart on disconnect; `AudioFormat::Float`
- Android Performance Tuner: `TuningFork_init/frameTick/flush`; instrument IDs (PACED_FRAME_TIME, CPU_TIME, GPU_TIME, SWAPPY_WAIT_TIME); fidelity parameters and annotation protobufs; Google Play backend integration
- Memory Advice API: `MemoryAdvice_init/registerWatcher/getMemoryState`; `APPROACHING_LIMIT` / `CRITICAL` states; `getPercentageAvailableCapacity()`; `/proc/meminfo` + `ActivityManager.MemoryInfo` sampling model
- Android GPU Inspector: frame capture (Vulkan API trace + GPU counter per draw call); system profiling (perfetto GPU counter tracks); supported GPU families (Adreno, Mali, PowerVR); Vulkan validation layer integration; `TRACE_EVENT` perfetto annotation from app code; CLI capture for CI
- **Integrations**: Ch85 (SurfaceFlinger/BufferQueue as ANativeWindow consumer), Ch86 (Vulkan swapchain creation, @FastNative JNI boundary), Ch87 (ARCore / OpenXR loop analogous to android_main), Ch75 (Swappy fence backpressure on dma_fence/sync_file), Ch26 (AMediaCodec NDK video decode parallel to VA-API)

### Chapter 164: Android Runtime and Native Interop: ART, JNI, and the NDK

- ART vs HotSpot JVM: register-based DEX bytecode (register vs stack machine); `d8`/`r8` toolchain; DEX shared string/type/proto pool
- dex2oat compilation tier ladder: `verify`, `quicken`, `speed-profile`, `speed`, `everything`; `.oat` ELF format; JIT compiler (`libart-compiler.so`) + JIT profile recording
- Profile-Guided Compilation (PGC) and Cloud Profiles: JIT hot-method recording; `dexopt` background recompilation; Google Play aggregated profile delivery; Baseline Profiles API (`androidx.profileinstaller`)
- Concurrent Copying (CC) GC and Baker read barriers: per-reference-load barrier; from/to-space flip; `kRunnable` safepoint model; `kNative` passive suspension; interaction with `@CriticalNative`
- userfaultfd GC (Android 13+, A-GC): `mremap()` compaction; page-fault-driven copy; zero per-reference overhead in steady state; kernel 5.10+ requirement
- Zygote fork model and `.art` boot images: pre-initialised heap serialisation; CoW page sharing across processes; OTA image regeneration via `dexopt`
- nterp interpreter (Android 12+): opcode-indexed table dispatch; DEX-direct operation; JIT-compatible frame layout for on-stack replacement
- Non-SDK interface restrictions (API 28+): `@hide`/`@UnsupportedAppUsage` greylist/blacklist; reflection blocking; JNI `GetMethodID` null-return; NDK symbol versioning
- ART intrinsics: `java.lang.Math`, `String`, `System.arraycopy`, bit manipulation, VarHandle CAS; NEON SIMD codegen; comparison with HotSpot intrinsics
- ART deoptimisation model: inline cache overflow; CHA invalidation; OSR exit; dex register map; comparison with HotSpot uncommon traps
- ART vs HotSpot feature matrix (summary table across 12 dimensions)
- The Android GPU call stack: Kotlin/Java → JNI boundary → ART → framework native C++ → vendor Vulkan ICD → kernel DRM → GPU hardware
- `@FastNative` and `@CriticalNative`: `kAccFastNative = 0x00080000`; `kAccCriticalNative = 0x00200000`; ABI change (no `JNIEnv*`); static-only restriction; 2× / 3–5× speedup on AArch64
- ART JNI trampoline internals: 8-step standard stub; HandleScope push/pop; `kRunnable → kNative` `StoreRelease` barrier; GC safepoint check; `@FastNative` omits HandleScope; `@CriticalNative` omits both barrier and HandleScope
- GC interaction: `kNative` passive suspension vs `kRunnable` blocking; `SuspendAll()` and `@CriticalNative` deadlock risk; practical guidance (nanosecond-range methods only)
- Java licensing: `dalvik.*` namespace outside Java SE IP; Oracle v. Google SCOTUS 2021 fair-use ruling; JNI as open specification
- Why Project Panama FFM does not apply to Android (HotSpot-internal; ART lacks JVM TI, Graal stubs)
- Is JNI still used? Framework boundary; zero-per-frame NDK-direct pattern; Rust Binder services; Flutter/Unity/Unreal shim model
- NativeActivity and `android_native_app_glue`: background pthread; `android_app` struct; `ANativeActivityCallbacks` 16 fields; `AInputQueue` event loop; 3 limitations
- GameActivity (AGDK): Kotlin subclass; double-buffered input arrays; `android_app_swap_input_buffers`; `GameTextInput`; Paddleboat integration
- `ANativeWindow` in depth: producer end of `BufferQueue`; acquire/release refcount; `setBuffersGeometry`; `vkCreateAndroidSurfaceKHR`; `eglCreateWindowSurface`; CPU `lock/unlockAndPost`
- Swappy frame pacing (cross-reference with Ch161): `VK_GOOGLE_display_timing`; `SwappyVk` 5-step sequence; Choreographer calibration; pipeline/auto modes
- Vulkan swapchain lifecycle ordering: `APP_CMD_INIT_WINDOW` vs `APP_CMD_RESUME`; authoritative `android_app::window` signal
- Managed GPU access: `android.graphics.vulkan` wrappers (Android 14+); ANGLE as default GLES (Android 17+); LiteRT Vulkan delegate
- NDK trajectory: C/C++ fully supported; Rust as co-equal; `ndk`/`ndk-sys` Rust crates; `ash` for Vulkan; `cargo-ndk`; `android-activity` entry point; APEX ABI stability
- What will not replace the NDK: Project Panama FFM, WASM/WebGPU, Kotlin/Native (targets ART on Android), Flutter Dart FFI
- NDK evolution table (near/medium/long term)
- **Integrations**: Ch85 (SurfaceFlinger/ANativeWindow consumer side), Ch86 (Vulkan-only chapter now; runtime context here), Ch87 (Quest platform runs on Android 14 ART), Ch161 (AGDK components in depth), Ch4 (GPU memory topology), Ch34 (ANGLE below JNI boundary), Ch26 (Binder for Zygote ServiceManager; non-SDK enforcement)

### Chapter 166: Android AR: ARCore Architecture, Camera HAL Integration, and Android XR (expanded)

- Expanded companion to Chapter 87 covering the Android XR era
- Android XR platform: Samsung Galaxy XR / Project Moohan; Jetpack XR SDK (`androidx.xr`): `Session.create()`, `Entity`, `PanelEntity`, `GltfModelEntity`, `SpatialCapabilities`
- OpenXR on Android: `XR_KHR_android_create_instance`, `XrSwapchainImageAndroidKHR`; ARCore's embedded OpenXR loader; how ARCore implements the OpenXR runtime contract on Android phones vs. headsets
- Qualcomm Snapdragon Spaces XDK: Snapdragon Spaces OpenXR profile, hand tracking extensions, passthrough API
- Monado OpenXR runtime as forward reference: the open-source alternative for Linux AR/VR development
- Deeper Vulkan camera import: `VK_ANDROID_external_memory_android_hardware_buffer`, `VkSamplerYcbcrConversion`, `VkExternalFormatANDROID` for camera YUV
- `VK_EXT_plane_detection` for spatial plane query via Vulkan
- Environmental HDR spherical harmonic light estimation API deep dive
- **Integrations**: Ch87, Ch27 (OpenXR session model on Linux/Monado), Ch86 (Vulkan Android extensions), Ch85 (SurfaceFlinger swapchain)

### Chapter 191: LiteRT and MediaPipe — On-Device ML Inference on the Android Graphics Stack

- LiteRT (formerly TensorFlow Lite, renamed 2024): on-device inference runtime for `.tflite` model files; `Interpreter` / `InterpreterApi`; `AllocateTensors()` → `Invoke()`
- Delegate model: NNAPI delegate (Android 8.1+, routes to DSP/NPU via `ANeuralNetworksModel`), GPU delegate (OpenGL ES 3.1 compute shaders or Vulkan compute), Edge TPU delegate (Coral hardware); delegate selection heuristics
- `AHardwareBuffer` tensor interop: GPU delegate supports `AHardwareBuffer`-backed tensors for zero-copy from camera → inference → display; `TfLiteGpuDelegateOptionsV2`; `GlBufferHandle` input/output binding
- LiteRT model formats: `.tflite` flatbuffer; quantised models (INT8, INT4, dynamic range quant); `SignatureDef`-based multi-signature models; LiteRT's relationship to ONNX Runtime and OpenVINO mobile backends
- MediaPipe framework: `CalculatorGraph` as a directed compute graph; `Packet` data flow; `GlCalculatorHelper` for GPU-accelerated calculator nodes; `GpuBuffer` ↔ `GlTexture` ↔ `AHardwareBuffer` interop
- MediaPipe Tasks API (2022+): high-level `ObjectDetector`, `PoseLandmarker`, `HandLandmarker`, `FaceLandmarker`, `ImageClassifier`; `BaseOptions` for model loading; `RunningMode` (IMAGE, VIDEO, LIVE_STREAM)
- Camera2 → MediaPipe pipeline: `SurfaceTexture` from Camera2 → `GL_TEXTURE_EXTERNAL_OES` → MediaPipe `GlTextureBuffer` input; zero-copy camera-to-inference path
- ARCore + MediaPipe composition: using ARCore for world tracking + MediaPipe for person segmentation or pose overlays; shared `ArFrame` timestamp for synchronisation
- On-device ML performance: latency vs. accuracy tradeoffs; INT8 quantisation impact; batch size 1 constraints on mobile; thermal throttling; profiling via Android GPU Inspector
- **Integrations**: Ch85 (AHardwareBuffer, SurfaceFlinger), Ch86 (GPU delegate via GLES/Vulkan compute), Ch87 (ARCore + MediaPipe composition), Ch88 (NPU/NNAPI — NNAPI delegate backend), Ch108 (ROCm comparison — desktop ML inference contrast)

---

## Part XX — AI/ML Inference on Linux

> **Note:** Chapter 48 (ROCm and Machine Learning on Linux GPUs) was originally part of Part VII-A (GPU APIs & Extended Reality) and has been moved here as it is more naturally an AI/ML infrastructure chapter than a general middleware chapter.

### Chapter 199: Jupyter Internals — Architecture, Python Runtime, Multi-Kernel Support, and GPU Computing

- Architecture: `jupyter_server` Tornado HTTP+WebSocket server; JupyterLab SPA vs Classic Notebook vs nteract; kernel/frontend decoupling via ZMQ (→ ch198 §18); JupyterHub multi-user model (Proxy + Hub + Spawner + KernelManager + kernel subprocess); REST API (`/api/kernels`, `/api/sessions`) and WebSocket `channels` endpoint multiplexing all five ZMQ sockets
- ZMQ→WebSocket bridge: how `jupyter_server` proxies ZMQ multi-part messages over a single browser WebSocket; `@jupyterlab/services` kernel client demultiplexing by `channel` field
- IPython/Python internals: `ZMQInteractiveShell` execution pipeline (input transformers → AST transform → compile → exec); `sys.displayhook` → `PlainTextFormatter`/`HTMLFormatter`; `sys.stdout` → `OutStream` writing iopub `stream` messages; `_repr_html_`/`_repr_png_`/`_repr_mimebundle_` rich display dispatch; line/cell magic dispatch (`MagicsMixin`); `IPCompleter`+jedi tab completion; Traitlets reactive config system; DAP debugger via `debugpy` on control socket
- Multi-kernel support: `KernelManager`/`MultiKernelManager`; kernelspec (`kernel.json` schema: argv, env, metadata); xeus C++ kernel framework (`xinterpreter` virtual interface; xeus-python, xeus-cling CUDA C++, xeus-lua); kernel provisioners (local, Docker, Kubernetes via Enterprise Gateway); SSH-tunnelled remote kernels
- Notebook file format: `.ipynb` JSON schema (nbformat, cells, outputs, MIME data dict); `nbconvert` (HTMLExporter, LatexExporter, ScriptExporter, Jinja2 templates); `papermill` parameterised execution; `jupytext` notebook↔Python/Markdown sync; `nbmake`/`nbval` pytest testing
- JupyterLab rendering pipeline: Lumino widget framework (`Widget`, `DockPanel`, `CommandRegistry`, plugin DI system); MIME renderer registry (`IRenderMimeRegistry`, `IRenderMime.IRenderer`); cell `OutputArea`; renderers for vega, plotly, GeoJSON; Voilà dashboard server; `anywidget` ESM custom widget authoring
- Interactive widgets: `comm_open`/`comm_msg` protocol; `ipywidgets` `HasTraits` sync; slider→GPU update loop example; `Output` widget; `anywidget` WebGL canvas widget
- GPU computing: CuPy `cupy.ndarray` (single CUDA context per kernel process, shared across all cells); RAPIDS cuDF/cuML; PyTorch/JAX CUDA context lifecycle; GPU memory accumulation across cells; `torch.cuda.empty_cache()`; multi-GPU with `CUDA_VISIBLE_DEVICES`; ROCm/HIP notebooks
- GPU visualisation: `ipyvolume` WebGL volume rendering (Three.js, CuPy→numpy→base64→WebGL path); `plotly` WebGL traces (`Scattergl`, `Scatter3d`); `vispy`/`moderngl` PNG export limitation; `pydeck` deck.gl GPU tiles; emerging `wgpu-py`+`anywidget` WebGPU path
- Real-time collaboration: Yjs CRDT (`Y.Doc`, `Y.Map`, `Y.Text`); `@jupyter/ydoc` (`YNotebook`, `YCell`); `jupyter-collaboration` y-websocket endpoint; awareness protocol for cursor presence
- JupyterLite: static-site Jupyter; Pyodide (CPython→Wasm via Emscripten, `micropip`); service worker REST/WebSocket intercept; `SharedArrayBuffer` in-process kernel messaging; no CUDA/no Unix sockets; `jupyter lite build`
- Linux-specific: GPU device access (`/dev/nvidia*`, `/dev/dri/renderD128`); `nvidia-container-toolkit` CDI; JupyterHub `DockerSpawner`/`KubeSpawner` GPU resource limits; cgroup v2 `MemoryMax`/`CPUQuota` via `SystemdSpawner`; MIG partitioning for shared GPU notebooks; `jupyter-server-proxy` for TensorBoard/Dask dashboard
- Roadmap: JupyterLab 4/5 (virtual output scrolling, notebook v5 format, binary ZMQ frame protocol); Jupyter AI (`%%ai` magic, Ollama local LLM backend); WebGPU widgets; JupyterLite WebGPU; DAP GPU debugger integration
- **Integrations**: ch198 §18 (ZMQ kernel protocol detail); ch34 (ANGLE/WebGL for ipyvolume/plotly); ch98 (WebAssembly/Pyodide JupyterLite); ch48 (ROCm, KFD, RAPIDS); ch55 (GPU containers, nvidia-container-toolkit); ch88 (NPU/accel device access); ch124 (LLM inference, Ollama as jupyter_ai backend)

### Chapter 48: ROCm and Machine Learning on Linux GPUs

*(Moved from Part VII-A to Part XX — AI/ML infrastructure chapter)*
- ROCm stack overview: the KFD (Kernel Fusion Driver) at `/dev/kfd`; `amdkfd` kernel module and its relation to `amdgpu` DRM (Ch5); the HSA (Heterogeneous System Architecture) memory model; ROCm hardware support matrix (CDNA vs. RDNA generations)
- HIP runtime: `hipMalloc` / `hipMemcpy` / `hipLaunchKernelGGL`; HIP as a CUDA-portable API; `hipcc` compiler driver; `hipify-perl` and `hipify-clang` for CUDA→HIP porting
- ROCm compilation pipeline: HIP C++ → Clang/LLVM → AMDGPU LLVM backend → GCN/CDNA ISA; `amdgcn-amd-amdhsa` target triple; `--offload-arch=gfx942` for MI300X; comparison with Mesa's ACO (Ch15) — ACO is for graphics command streams, LLVM is for compute
- ML frameworks on ROCm: PyTorch ROCm backend (`torch.version.hip`); TensorFlow ROCm via `tensorflow-rocm`; JAX via `jax[rocm]`; how `hipblaslt` replaces `cuBLAS` and `MIOpen` replaces `cuDNN`
- Math library ecosystem: rocBLAS (BLAS), rocFFT, rocRAND, MIOpen (DNN primitives), hipSPARSE; kernel autotuning via `Tensile` (convolution/GEMM auto-tuner); `rocm-smi` for device and memory monitoring
- AMD CDNA3 / MI300X for ML: 192 CUs, HBM3 memory, FP8 support, Unified Memory Architecture (CPU+GPU in one address space); implications for large-model inference; `amdgpu` XGMI (Infinity Fabric) for multi-GPU communication
- Intel oneAPI on Linux: Level Zero (Ch25); `intel-compute-runtime` as the Level Zero ICD; SYCL/DPC++ compilation via `icpx`; `intel_gpu_top` for Arc workload monitoring; oneAPI vs. ROCm portability story
- ROCm containers and cloud: Docker/Kubernetes with `--device /dev/kfd --device /dev/dri/renderDN`; ROCm Device Plugin for Kubernetes; AWS EC2 `g4ad` (RDNA2) vs. `p4d` (A100/CUDA); AMD Instinct MI300X cloud availability
- **Integrations**: ROCm uses `amdgpu`'s KFD compute queue (Ch5), distinct from the DRM render node used by Mesa; HIP kernels compile via the same AMDGPU LLVM backend as `radeonsi` (Ch19) but at a different level; ML inference via Vulkan compute (Ch25) is an alternative for portable inference; GPU containers (Ch55) wrap this stack for cloud ML deployments; multi-GPU collective ops use p2p DMA (Ch49); compare with CUDA/NVML-based inference (Ch88, Ch124)

### Chapter 124: Local LLM Inference on Linux GPUs

- GGML architecture: tensor type system, block-quantised formats (Q4_K_M, Q8_0, F16), runtime backend selection; reference https://github.com/ggerganov/llama.cpp
- `ggml-vulkan.cpp`: `ggml_vk_context`, `vk_pipeline`; compute shaders for matrix-vector multiply; SPIR-V compiled via glslang at init
- llama.cpp Vulkan path: `llama_backend_init`, `ggml_backend_vk_init`; device enumeration; KV cache via `ggml_backend_vk_buffer_type`; compute graph (Q/K/V split → RoPE → GQA → FFN); `--n-gpu-layers` tensor parallelism
- GGUF file format: magic, header, tensor metadata, data blocks; `mmap` of weights; page-fault behaviour on first pass; BAR-resizable DMA-BUF zero-copy path
- Ollama: Go server + llama.cpp library; `gpu.go` NVML/ROCm/sysfs detection; REST API (`/api/generate`, `/api/chat`); parallel request handling; reference https://github.com/ollama/ollama
- ONNX Runtime GPU EPs: CUDA EP → ROCm EP → OpenVINO EP → CPU fallback; `OrtCUDAProviderOptions`; cuBLAS GEMM dispatch; model quantisation
- OpenVINO EP on Intel Arc: `OrtOpenVINOProviderOptions`; IE plugin → Level Zero; HETERO:NPU,GPU,CPU heterogeneous execution
- ROCm MIOpen for inference: `rocblas_gemm_ex`; HIP unified memory on APUs; `vllm` on ROCm; `ROCR_VISIBLE_DEVICES`
- PagedAttention / vLLM KV cache: page table, block manager, prefix caching; llama.cpp sliding-window context; CPU eviction under pressure
- Performance benchmarking: `llama-bench` (prompt tokens/s, generation tokens/s); quantisation impact table; bandwidth-bound vs compute-bound analysis; `nvtop`/`radeontop`/`intel_gpu_top` during inference
- **Integrations**: Ch3, Ch6 (Panfrost for edge inference), Ch14, Ch18 (RADV for ROCm), Ch24, Ch25, Ch48, Ch55, Ch61, Ch71, Ch88

### Chapter 88: NPU and AI Accelerator Integration on Linux

- AI PC landscape 2026: Intel MTL/LNL/ARL NPU, AMD XDNA (Ryzen AI 300), Qualcomm Snapdragon X Elite NPU; power/latency case for NPU vs GPU
- Intel NPU kernel driver (`drivers/accel/ivpu/`): `DRM_IOCTL_IVPU_*`; accel DRM subsystem (`/dev/accel/accel0`); firmware loading `ivpu_fw_load`; `ivpu_bo` GEM objects; job submission ioctl
- OpenVINO for Intel NPU: `ov::Core` enumeration (NPU/GPU/CPU); `ov::CompiledModel` NPU compilation; HETERO plugin; tensor layout requirements; power modes
- AMD XDNA (`amdxdna` driver, Linux 6.11): AIE2 array (RISC-V scalar + SIMD vector cores); `AMDXDNA_BO_*` buffer types; Vitis AI and ONNX Runtime XDNA EP; reference https://github.com/amd/xdna-driver
- Qualcomm Hexagon DSP/NPU: FastRPC IPC; SNPE/QNN SDK; ONNX Runtime QNN EP; Linux mainline status
- DRM `accel` subsystem (Linux 6.2): `drm_accel_device`; `/dev/accel/accel*` nodes; Gaudi2, ivpu, AMDXDNA as current users; char-device vs accel comparison
- Heterogeneous dispatch: OpenVINO HETERO plugin; ORT EP priority chain; DMA-BUF between render node and accel node for zero-copy; `dma_buf_map` across devices
- Framework integration: PyTorch `torch.compile` → Inductor → NPU; HuggingFace `device="npu"`; GGML backend registration API (`ggml_backend_reg`)
- Power and thermal: ACPI D0/D3 transitions in `ivpu_pm.c`; approximately 10W NPU vs 200W GPU for sustained 7B inference
- Debugging: OpenVINO `benchmark_app`; `xdna-smi`; `trace_ivpu_*` tracepoints; `/sys/kernel/debug/dri/accel0/`
- **Integrations**: Ch1, Ch3, Ch6, Ch24, Ch25, Ch55, Ch71, Ch124

---

### Chapter 94: ComfyUI and ComfyScript: Node-Graph AI Image Generation on Linux GPUs

- ComfyUI architecture: aiohttp web server + DAG execution engine + React frontend; not a GUI toolkit; GPU backends: CUDA (NVIDIA), ROCm (AMD HIP), MPS (Apple), CPU fallback; supported model families: SD 1.5, SDXL, SD3, FLUX.1, PixArt-Sigma, AuraFlow
- Node graph execution model: node class anatomy (`INPUT_TYPES`, `RETURN_TYPES`, `FUNCTION`, `CATEGORY`, `OUTPUT_NODE`); prompt JSON format with link refs `["node_id", output_index]`; `PromptExecutor.execute()` topological sort; output caching keyed by (node_id, inputs_hash); `comfy.model_management` GPU memory coordinator
- Core built-in nodes (`nodes.py`): `CheckpointLoaderSimple` (MODEL+CLIP+VAE); `CLIPTextEncode` / `CLIPTextEncodeSDXL` (dual CLIP); `EmptyLatentImage` (4-ch or 16-ch latent); `KSampler` / `KSamplerAdvanced`; `VAEDecode` / `VAEEncode` / tiled variants; `SaveImage` / `PreviewImage`; `LoraLoader`; `ControlNetLoader` + `ControlNetApplyAdvanced`; `ImageScale`; `ConditioningCombine`
- Sampler and scheduler system: k-diffusion wrappers in `comfy/samplers.py`; KSAMPLER_NAMES including euler, euler_ancestral, dpmpp_2m, dpmpp_sde, dpmpp_3m_sde, ddim, uni_pc, lcm, etc.; scheduler names: normal, karras, exponential, sgm_uniform, beta, kl_optimal; CFG formula `eps = uncond + cfg * (cond - uncond)`; `ModelPatcher.patch_model()` / `unpatch_model()`; FlowMatch for FLUX/SD3 (rectified flow); DDIM inversion for img2img
- Memory management and GPU selection: `model_management.py` `current_loaded_models` list; `LoadedModel` wrapper; CLI flags: `--gpu-only`, `--highvram`, `--normalvram`, `--lowvram`, `--novram`, `--cpu`; `get_torch_device()` CUDA/ROCm/MPS detection; `CUDA_VISIBLE_DEVICES` / `--cuda-device N`; `soft_empty_cache()` = `torch.cuda.empty_cache()` + gc; `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True`; precision flags `--fp8_e4m3fn-unet`, `--force-fp16`, `--bf16-unet`
- REST API and WebSocket: `POST /prompt` → `{"prompt_id": ...}`; `GET /queue`, `/history`, `/view`, `/object_info`, `/system_stats`; `POST /upload/image`; WebSocket at `/ws?client_id=<uuid>` — message types: `status`, `progress`, `executing`, `executed`, `execution_error`; Python aiohttp client example
- ComfyScript (https://github.com/Chaoses-Ib/ComfyScript): `pip install comfy-script[default]`; stub generation from `/object_info`; `Workflow(wait=True)` context manager; typed node return wrappers; real-time mode; mypy/pyright compatible; SDXL two-pass + LoRA workflow example
- Custom node development: `NODE_CLASS_MAPPINGS` / `NODE_DISPLAY_NAME_MAPPINGS`; tensor conventions (IMAGE `[B,H,W,C]` float32, MASK `[B,H,W]`, LATENT dict with `"samples"` key, MODEL/CLIP/VAE as opaque objects); `IS_CHANGED()` classmethod for cache invalidation; web UI JS extensions; ComfyUI-Manager; community packs: Impact-Pack, Advanced-ControlNet, WAS-Node-Suite, GGUF
- FLUX.1 and DiT support: `UNETLoader`, `DualCLIPLoader`, `VAELoader`, `CLIPTextEncodeFlux`, `FluxGuidance` nodes; 16-channel VAE latent; dual-stream + single-stream transformer; FLUX-dev vs schnell; fp8 quantisation (`--fp8_e4m3fn-unet`); GGUF loading via ComfyUI-GGUF; memory requirements by precision
- Performance and Linux optimisations: xFormers / PyTorch SDPA for Flash Attention; `torch.compile` warm-up and throughput gains; preview methods (TAESD, latent2rgb); batch_size > 1 efficiency; nsys profiling; ROCm-specific: `HSA_OVERRIDE_GFX_VERSION`, `rocm-smi`, Triton attention; `nvidia-smi dmon` monitoring
- Production deployment: Docker with NVIDIA Container Toolkit (`--gpus all`); model volume mounts; Kubernetes `nvidia.com/gpu: 1` resource; nginx reverse proxy; WebSocket automation; n8n / Airflow integration via REST API; queue management
- **Integrations**: Ch25 (CUDA kernel dispatch), Ch48 (ROCm backend), Ch55 (GPU containers), Ch66 (CUDA graphs, NVRTC for torch.compile), Ch42 (iterative GPU algorithm scheduling), Ch69 (Omniverse AI pipeline comparison)

### Chapter 108: ROCm and HIP — AMD's GPU Compute Stack *(Part XX)*

- ROCm architecture overview: the KFD (Kernel Fusion Driver) and its relationship to `amdgpu` DRM; `/dev/kfd` and `/dev/dri/renderDN` dual access model; ROCm hardware support matrix — CDNA (Instinct MI) vs. RDNA (Radeon) generations; HSA (Heterogeneous System Architecture) memory model; source at https://github.com/ROCm/ROCm
- HIP programming model: `hipMalloc`, `hipMemcpy`, `hipLaunchKernelGGL`; HIP as a CUDA-portable API layer; `hipcc` compiler driver; `hipify-perl` and `hipify-clang` for CUDA→HIP source translation; HIP kernel syntax and thread indexing model; `hipDeviceProperties` and capability queries
- ROCm compilation pipeline: HIP C++ → Clang/LLVM → AMDGPU LLVM backend → GCN/CDNA ISA; `amdgcn-amd-amdhsa` target triple; `--offload-arch=gfx942` for MI300X; LLVM IR → AMDGPU ISA as compared to Mesa ACO (Ch15) — ACO for graphics command streams, LLVM for compute
- ML frameworks on ROCm: PyTorch ROCm backend (`torch.version.hip`); TensorFlow-ROCm; JAX via `jax[rocm]`; `hipblaslt` replacing `cuBLAS`; `MIOpen` replacing `cuDNN`; `Tensile` auto-tuner for GEMM/convolution kernels; `rocm-smi` monitoring
- AMD CDNA3 / MI300X for LLM inference: 192 CUs, HBM3 memory, FP8 support, Unified Memory Architecture (UMA CPU+GPU); `vllm` on ROCm; `ollama` ROCm backend; `ROCR_VISIBLE_DEVICES`; memory bandwidth characteristics vs. NVIDIA H100; multi-GPU via XGMI Infinity Fabric
- ROCm containers and cloud: Docker with `--device /dev/kfd --device /dev/dri/renderDN`; AMD ROCm Device Plugin for Kubernetes (`amd.com/gpu`); ROCm official Docker images (`rocm/dev-ubuntu-*`); ROCm version pinning vs. kernel driver compatibility; AWS `g4ad` (RDNA2) cloud availability
- **Integrations**: Ch5 (amdgpu — KFD compute queue distinct from DRM render node), Ch15 (ACO — graphics codegen contrast with LLVM compute path), Ch25 (Vulkan compute — alternative portable path), Ch48 (Ch48 covers ROCm in the ML context; this chapter deepens the HIP runtime and ROCm compiler stack), Ch55 (GPU containers — ROCm container device access), Ch124 (Local LLM Inference — ROCm backend for llama.cpp/vllm)

### Chapter 115: NeRFStudio, Neural Radiance Fields, and 3D Gaussian Splatting on Linux *(Part XX)*

- Neural Radiance Field (NeRF) fundamentals: the volume rendering integral; MLP mapping (x,y,z,θ,φ) → (colour, density); positional encoding; stratified + hierarchical sampling; differentiable rendering and gradient flow; `nerf-pytorch` reference implementation; training on NVIDIA A100/H100 via CUDA (Ch66) and on AMD MI300X via ROCm (Ch108)
- NeRFStudio: modular NeRF framework; `ns-train nerfacto --data ...`; pipeline: `DataParser` → `Model` → `Renderer`; Nerfacto model — hash encoding, appearance embedding, distortion loss; `ns-viewer` web viewer via Three.js + WebSocket; source at https://github.com/nerfstudio-project/nerfstudio; PyTorch + CUDA dependency; ROCm compatibility status
- 3D Gaussian Splatting (3DGS): differentiable rasteriser — 3D Gaussians with learned position/covariance/opacity/colour; tile-based sorted back-to-front splat rasteriser; `gsplat` library (Python/CUDA, MIT); `gaussian-splatting` reference (CUDA); training: Adam optimiser, densification strategy, pruning; rendering: sorted splat tiles → alpha blend; inference throughput 100–200+ FPS on RTX 4090; scene capture pipeline: COLMAP SfM → camera poses → Gaussian training
- CUDA acceleration: `gsplat` CUDA kernels for forward (rasterise) and backward (gradient) passes; `torch.cuda.amp` mixed precision; `torch.compile` for kernel fusion; multi-GPU training via `torch.nn.parallel.DistributedDataParallel`; `nvidia-smi dmon` and `nvtop` monitoring during training
- Mesh extraction from Gaussians: marching cubes on SDF estimated from Gaussians (`sugar` method); CGAL mesh post-processing (Ch113); export to glTF (Ch64) for real-time rendering pipeline integration; Blender import via `gaussian-splatting-blender-addon`; limitations of the mesh extraction (density artefacts, view-dependent colour loss)
- OptiX Cooperative Vectors for neural rendering (Ch67): embedding MLP inference inside OptiX any-hit shaders for real-time NeRF rendering; `3DGUT` (NVIDIA, April 2025) — GPU-accelerated Gaussian unstructured tiles; inference at 30+ FPS on RTX 4090; comparison with rasterisation-based Gaussian Splatting
- **Integrations**: Ch42 (Blender — Gaussian Splat import and preview), Ch66 (CUDA — gsplat CUDA rasteriser), Ch67 (OptiX Cooperative Vectors — neural material shaders), Ch68 (DLSS 4 — Ray Reconstruction + Gaussian Splatting hybrid rendering), Ch108 (ROCm — AMD GPU training alternative), Ch113 (CGAL — mesh extraction from NeRF/Gaussian SDF), Ch117 (Slang — neural shader language for NeRF material shaders)

---

### Chapter 89: GPU Virtualization in Depth *(Part IX)*

- Virtualisation approaches: emulation (virtio-gpu), paravirtualization (VirGL/Venus), passthrough (VFIO); trade-offs and use cases
- IOMMU and VFIO foundations: Intel VT-d / AMD-Vi DMA mapping; IOMMU groups; `vfio-pci` ops; BAR mmap via `/dev/vfio/group_N`; PCIe FLR requirements
- VFIO GPU passthrough: QEMU `-device vfio-pci`; OVMF; ACS override quirks; Looking Glass for framebuffer capture; reference https://looking-glass.io/
- Intel SR-IOV: GVT-g (`kvmgt`, mdev, shadow GTT) → Xe SR-IOV (`xe_gt_sriov_pf_*`, `sriov_numvfs`); performance tiers
- AMD MxGPU SR-IOV: spatial partitioning; `amdgpu_virt_*`; `amdgpu_sriov_vf()` guard; world switch overhead
- NVIDIA vGPU and MIG: time-sliced vGPU types; `nvidia-smi mig --create-gpu-instance`; cgroups for container MIG; `NVIDIA_VISIBLE_DEVICES=MIG-GPU-...`
- virtio-gpu protocol: `virtio_gpu_cmd_*`; CAPSET_VENUS; blob resources (`VIRTIO_GPU_CMD_RESOURCE_CREATE_BLOB`) for zero-copy host↔guest
- Venus (Vulkan-over-virtio): `vn_ring` command stream; `virtio_gpu_blob` for `VkDeviceMemory`; vkrenderer daemon; reference Mesa `src/virtio/`
- WSL2: `dxgkrnl` DRM driver; Mesa `d3d12` Gallium via `libdxcore.so`; DirectML for ONNX Runtime; WSLg (Weston + RDP); limitations vs native Linux
- GPU containers: NVIDIA Container Toolkit; CDI spec; AMD ROCm Docker; Kubernetes device plugins; device cgroup v2 (eBPF)
- Performance comparison table: VFIO passthrough / SR-IOV / MIG / Venus / VirGL / virtio-gpu (emulated)
- **Integrations**: Ch1, Ch3, Ch4, Ch18, Ch19, Ch24, Ch55, Ch80, Ch124, Ch88

### Chapter 90: Open ARM GPU Drivers — Lima, Panfrost, and Panthor *(Part II)*

- Mali GPU family history: Utgard / Midgard / Bifrost / Valhall; the closed-driver problem; open driver projects; deployment: PinePhone, Orange Pi 5 (RK3588), i.MX8
- Lima kernel driver (`drivers/gpu/drm/lima/`): `lima_bo`, PP/GP register-level programming; `lima_sched` (drm_gpu_scheduler); Mesa Lima Gallium (`src/gallium/drivers/lima/`)
- Panfrost kernel driver (`drivers/gpu/drm/panfrost/`): `panfrost_bo`; AS (Address Space) MMU model; `panfrost_mmu_*`; Midgard VLIW ISA; Bifrost clause-based ISA
- Mesa Panfrost Gallium (`src/gallium/drivers/panfrost/`): `pan_context`; Bifrost compiler (`src/panfrost/compiler/`): `bi_*` IR; NIR → Bifrost lowering; clause scheduling; register allocation
- Panthor kernel driver (mainlined Linux 6.8): Valhall CSF (Command Stream Frontend) firmware-based submission; `panthor_fw_*`; `panthor_heap_*`; `DRM_IOCTL_PANTHOR_*`
- Mesa Panfrost Vulkan for Valhall (`src/panfrost/vulkan/`): `pvk_*` structs; Valhall descriptor model (buffer descriptors in memory); tiler heap allocator
- Real-world platforms: PinePhone Pro (RK3399/Mali-T860), Librem 5 (i.MX8MQ/etnaviv), Orange Pi 5 (RK3588/Mali-G610), Pinebook Pro
- Mesa CI with Lava: `panfrost-lava-*` CI jobs; `deqp-runner` on ARM boards; `panfrost-fails.txt`
- Comparison with Asahi AGX: reverse-engineering methodology; firmware dependency; Bifrost ISA attribution (Lyude Paul, Alyssa Rosenzweig); cross-pollination
- Contributing: dri-devel mailing list; LAVA test lab; `PANFROST_DEBUG=msgs,dump,perf`
- **Integrations**: Ch1, Ch2, Ch5, Ch14, Ch16, Ch21, Ch73, Ch92

### Chapter 91: MLIR and the Emerging GPU Compiler Infrastructure *(Part IV)*

- GPU compiler stack layers: application → ML framework → MLIR dialects → LLVM → PTX/AMDGCN/SPIR-V → Mesa → ISA; MLIR's role at the middle tier
- MLIR dialects for GPU: `gpu`, `vector`, `linalg`, `arith`, `memref`, `spirv`, `nvgpu`, `rocdl`; progressive lowering (linalg → vector → llvm); `mlir-translate --mlir-to-spirv`; reference https://mlir.llvm.org/docs/Dialects/GPU/
- GPU dialect: `gpu.func`, `gpu.launch`, `gpu.thread_id`, `gpu.barrier`; lowering to NVVM / ROCDL / SPIR-V; `GpuToSpirvPass`
- OpenAI Triton: `@triton.jit`; Triton IR → TritonGPU IR → LLVM → PTX/AMDGCN; `tt-opt` pipeline; `@triton.autotune`; Triton-ROCm; Triton Vulkan (experimental); reference https://github.com/openai/triton
- FlashAttention-2 in Triton: tile loop, on-chip KV accumulation, `tl.dot` for QK^T, online numerically-stable softmax; usage in llama.cpp and vLLM
- IREE: StableHLO → Stream → HAL → SPIR-V; `iree-compile --iree-hal-target-backends=vulkan-spirv`; `iree_hal_device_t`; Vulkan HAL submits offline SPIR-V; reference https://iree.dev/
- XLA: HLO → LLVM/PTX; `jax.jit`; producer-consumer fusion; XLA on ROCm (`jax[rocm]`); XLA Vulkan (via IREE, experimental)
- SPIR-V as lingua franca: binary format; `OpTypeCooperativeMatrixKHR`; `spirv-opt` passes; `spirv-cross` back-translation; `mlir-translate` → `vkCreateShaderModule` → Mesa NIR
- Cooperative matrix: `VK_KHR_cooperative_matrix`; `OpCooperativeMatrixMulAddKHR`; RADV → RDNA WMMA; ANV → Intel XMX; Triton `tl.dot` lowering
- Mesa connection: MLIR SPIRV → SPIR-V binary → `spirv_to_nir` → NIR → ACO/LLVM; `RADV_DEBUG=spirv,nir,aco`
- **Integrations**: Ch14, Ch15, Ch16, Ch24, Ch25, Ch61, Ch77, Ch124, Ch88

### Chapter 92: The Raspberry Pi GPU Stack — VideoCore and V3D *(Part II)*

- RPi hardware generations: BCM2835 (Pi 1, VC4), BCM2711 (Pi 4, V3D), BCM2712 (Pi 5, V3D v7.1); GPU openness history (Herman Hermitage ISA reverse engineering, Eric Anholt vc4/v3d drivers)
- VideoCore IV / vc4: QPU (4-way SIMD, 64-bit ISA); tile binning + tile rendering; `vc4` kernel driver (`drivers/gpu/drm/vc4/`): `vc4_bo`, `vc4_cl`, `vc4_job`; Mesa vc4 Gallium; `DRM_IOCTL_VC4_SUBMIT_CL`
- V3D (BCM2711): 4 QPUs per slice; `v3d` kernel driver (`drivers/gpu/drm/v3d/`): `v3d_bo`, render/TFU/CSD job types; `drm_gpu_scheduler`; Mesa v3d Gallium: `v3d_nir_to_vir`; VIR (V3D IR): SSA, scheduling, register allocation
- V3D shader compiler: `v3d_compile_vs/fs/cs`; `v3d_nir_lower_*`; VIR instruction set (FADD, FMUL, LDTMU); QPU binary format; 2-/4-thread fragment shaders
- KMS and display: `vc4_hdmi`, `vc4_crtc`; HVS (Hardware Video Scaler); atomic modeset; BCM2712 RP1 south bridge; 4K@60 on Pi 5
- Multimedia and camera: MMAL over VCHI firmware IPC; libcamera `PipelineHandlerRPi`; Pi ISP pipeline (Unicam CSI-2 → debayer → AWB → DMA)
- Hardware video decode: V4L2 M2M (`/dev/video10`-12, `bcm2835-codec`); FFmpeg `v4l2m2m`; MPV `--hwdec=v4l2m2m`; zero-copy NV12→DMA-BUF→texture
- Raspberry Pi OS stack: Wayfire (Pi 5/Bookworm); wlroots + vc4 DRM; GLES 3.1 on Pi 4; `vulkan-broadcom` (Mesa V3DV); V3DV Vulkan 1.0 conformance
- V3DV Vulkan driver (`src/broadcom/vulkan/`): `v3dv_device`, `v3dv_cmd_buffer`, `v3dv_pipeline`; CL generation; VC QOS registers
- Production use: `cage` single-app compositor; kiosk signage; ROS 2 + GPU perception; Pi CM4/CM5 embedded; thermal management
- **Integrations**: Ch1, Ch2, Ch5, Ch13, Ch14, Ch21, Ch42, Ch57, Ch90

### Chapter 93: GPU Performance Analysis Methodology *(Part IX)*

- Performance analysis vs debugging; measurement hierarchy: system → API → hardware counter; common fallacies (VRAM size vs bandwidth, GPU-bound as the goal)
- Frame time decomposition: `vkCmdWriteTimestamp2` with TOP/BOTTOM_OF_PIPE; `vkGetQueryPoolResults`; `timestampPeriod`; GPU-bound vs CPU-bound determination
- `VK_KHR_performance_query`: counter enumeration; types (UINT64, FLOAT32, percentage); `VkQueryPoolPerformanceCreateInfoKHR`; multi-pass collection; `vkAcquireProfilingLockKHR`
- GPU-bound vs CPU-bound: `vkWaitForFences` blocking; `vkQueueSubmit` cost; descriptor set allocation rate; MangoHUD CPU/GPU bars explained
- Occupancy and wave analysis: active warps / max warps per SM; VGPR pressure on RDNA; RDNA wave64 vs wave32; Nsight `sm__warps_active`; RGP occupancy lane view; `RADV_DEBUG=shader`
- Memory bandwidth profiling: DRAM bandwidth measurement via perf query (l2_cache_miss_rate, DRAM r/w bytes); NVIDIA L2 counters; AMD TCC hit rate; bandwidth-bound identification; BC7/ASTC mitigation
- Pipeline stall taxonomy: TMU latency, FP32 latency, LDS bank conflict, export stall (RDNA); RGP stall breakdown; Nsight `sm__pipe_fma_cycles_active`; Intel EU stalls
- AMD: RGP (`RADV_THREAD_TRACE=1`); `radeon_gpu_analyzer` for ISA; `umr` for raw PERF_SEL; reference https://gpuopen.com/rgp/
- NVIDIA: `ncu --set full`; key metrics (`sm__throughput`, `dram__throughput`); roofline model; `nsys profile --trace=vulkan,nvtx`; `VK_NV_device_diagnostic_checkpoints`
- Intel: `intel_gpu_top`; EU active/stall/idle; VTune GPU analysis; `VK_INTEL_performance_query`; Xe Slice/Subslice utilisation
- Worked case study: path-traced scene optimisation — frame time decomposition → bandwidth-bound diagnosis → BVH ray sort → L2 hit rate improvement → async BVH refit
- **Integrations**: Ch15, Ch24, Ch28, Ch29, Ch30, Ch56, Ch61, Ch67, Ch124

- **Integrations**: Ch1 (DRM kernel driver model), Ch2 (KMS on Apple display), Ch14 (Mesa driver architecture), Ch24 (Vulkan via Honeykrisp), Ch77 (SPIR-V → NIR → AGX ISA), Ch119 (Zink path for OpenGL)

### Chapter 135: Vulkan Ray Tracing on Linux *(Part VII-A — GPU APIs)*

- Ray tracing hardware: RDNA2+ BVH traversal (`image_bvh_intersect_ray`); Intel Xe-HPG RT unit; NVIDIA RTX Tensor/RT Cores; architectural comparison
- Extensions: `VK_KHR_ray_tracing_pipeline`, `VK_KHR_acceleration_structure`, `VK_KHR_ray_query`, `VK_KHR_deferred_host_operations`; feature query pattern
- Five shader stages: rgen, rint, rahit, rchit, rmiss; GLSL `traceRayEXT()`; payload `rayPayloadEXT`; built-ins: `gl_HitTEXT`, `gl_PrimitiveID`, `gl_WorldRayOriginEXT`
- BLAS construction: `VkAccelerationStructureGeometryKHR`; `vkGetAccelerationStructureBuildSizesKHR`; `vkCmdBuildAccelerationStructuresKHR`; 64-byte `VkAccelerationStructureInstanceKHR` layout
- TLAS: instance transform, SBT offset, cull mask; build flags: `PREFER_FAST_TRACE`, `ALLOW_UPDATE`, `ALLOW_COMPACTION`; compaction query + copy (30–60% size reduction)
- BVH algorithms: SAH, LBVH Morton code, PLOC; RADV GPU builder (`src/amd/vulkan/bvh/`): radix sort → LBVH → encode; ANV GRL (`src/intel/vulkan/grl/`): PLOC+SAH pipeline
- Shader Binding Table: record layout, `vkGetRayTracingShaderGroupHandlesKHR`; hit group indexing formula: `instanceSBTOffset + geometryIndex × stride + traceRayOffset`
- RADV RT pipeline: single compute mega-kernel; `radv_nir_lower_rt_io`; ACO emit `image_bvh_intersect_ray`; `RADV_PERFTEST=emulate_rt`
- ANV RT pipeline: GRL BVH build; Xe-HPG dedicated traversal unit; `genX(cmd_buffer_trace_rays)`; `INTEL_DEBUG=rt`
- `VK_KHR_ray_query`: inline queries from any stage; `rayQueryEXT`; AO example; `VK_KHR_ray_tracing_maintenance1`; software BVH fallback; Intel OIDN denoising
- **Integrations**: Ch18 (RADV), Ch19 (ANV), Ch24 (Vulkan API), Ch77 (SPIR-V RT opcodes), Ch110 (spirv-opt RT), Ch127 (hybrid mesh+RT), Ch133 (RT on async compute), Ch141 (coop matrix denoising)

### Chapter 136: WSL2 Linux Graphics — dxgkrnl and Mesa D3D12 *(Part IX)*

- WSL2 architecture: lightweight VM (Hyper-V); `dxgkrnl.sys` host kernel driver; `dxg` Linux kernel module (`/dev/dxg`); GPU-P (GPU Paravirtualization); PCIe-less GPU access
- GPU-P: virtual GPU via VMBUS; `VkDeviceMemory` → host DX12 heap; indirect rendering; `dxgkrnl` ioctls: `LX_DXOPENADAPTERFROMLUID`, `LX_DXSUBMITCOMMAND`, `LX_DXSIGNALSYNCOBJECT`
- Mesa D3D12 Gallium (`dzn`): `nir_to_dxil` compiler translating Mesa NIR to DXIL (DX12 shader IR); `mesa-vulkan-drivers` `dzn` Vulkan driver; OpenGL via Zink over `dzn`
- WSLg: Wayland + RDP compositing; `weston` as XWayland host + RDP server; `FreeRDP` client on Windows; `mstsc.exe` rendering; `DISPLAY` and `WAYLAND_DISPLAY` auto-set
- GPU selection: `MESA_D3D12_DEFAULT_ADAPTER_NAME`; GPU-P across NVIDIA/AMD/Intel/Apple M-series; performance headroom vs native Linux (15–30% overhead)
- Troubleshooting: `dxdiag`; WDDM 2.9+ driver requirement; `wsl --update`; `vulkaninfo --json`; GPU driver update via Windows Update
- Limitations: no CUDA in WSL1/WSL2 Mesa path; `nvidia-smi` in WSL2 CUDA path; no display hotplug; Wayland limitations vs native
- **Integrations**: Ch14 (Mesa Gallium — D3D12 is a Gallium driver), Ch24 (Vulkan dzn), Ch20 (WSLg Wayland), Ch28 (DXVK in WSL2), Ch77 (SPIR-V → DXIL — `nir_to_dxil` is a cross-IR compiler)

### Chapter 137: GPU Performance Profiling — RGP, GPA, and VK_EXT_performance_query *(Part IX)*

- Vulkan timestamps: `vkCmdWriteTimestamp2`; `VkQueryPool (TIMESTAMP)`; `calibratedTimestamp` (`VK_EXT_calibrated_timestamps`); converting ticks to nanoseconds
- `VK_EXT_performance_query`: `VkPerformanceCounterKHR`; `vkEnumeratePhysicalDeviceQueueFamilyPerformanceQueryCountersKHR`; AMD GPU clocks/occupancy; Intel EU utilisation; NVIDIA (via NVK or proprietary)
- AMD RGP (Radeon GPU Profiler): `VK_AMD_gpa_interface` + `VK_AMD_shader_core_properties`; SQTT (Shader Queue Thread Trace); `.rdc` frame capture; barrier bottleneck identification; wave occupancy heatmap; RGP download at GPUOpen
- AMD Radeon Developer Panel (RDP): live profiling; frame time graph; radeon_top; `RADV_THREAD_TRACE=1`
- Intel GPA (Graphics Performance Analyzers): `VK_INTEL_performance_query`; EU active/stall/idle; VTune GPU Hotspots; `intel_gpu_top`; Xe metrics via Linux perf (`i915 PMU`)
- GALLIUM_HUD: built-in Mesa overlay; `GALLIUM_HUD=fps,GPU-load,cpu`; available counters; SVG output; CPU/GPU correlation
- NVIDIA NSight (proprietary): `VK_NV_device_diagnostic_checkpoints`; shader occupancy; L2 cache hit; Nsight Systems timeline; Linux CLI: `nsys profile --capture-range=cudaProfilerApi`
- Frame time budget: 16.7 ms at 60 Hz; GPU hang vs CPU stall; `vkWaitForFences` CPU-wait debugging; back-pressure analysis
- **Integrations**: Ch24 (Vulkan timestamps), Ch93 (GPU performance methodology), Ch133 (async compute queue profiling), Ch137 self-referential: profiling the entire stack

### Chapter 138: Wayland Fractional Scaling and HiDPI *(Part VI-A — Wayland Compositor)*

- HiDPI problem: PPI formula; 4K/15" ≈ 282 PPI; device pixel ratio (DPR); integer 2× wastes logical space; need for 1.25–1.75× fractional scales
- `wl_output.scale`: integer-only (Wayland 1.9); `wl_surface.set_buffer_scale(2)`; `GDK_SCALE=2`; limitations — too coarse for 14-inch QHD panels
- `wp_fractional_scale_v1` (wayland-protocols staging, 2023): `wp_fractional_scale_manager_v1`; `get_fractional_scale(surface)`; `preferred_scale(scale)` event; `scale = desired_scale × 120` fixed-point encoding
- Client protocol: `set_buffer_scale` stays 1; `wp_viewport.set_destination(logical_w, logical_h)`; render at `ceil(logical × scale)` buffer; full commit sequence; why `ceil` not `round`
- Compositor support: GNOME Mutter (GNOME 47, 2024 stable; GNOME 43 experimental); `gsettings scale-monitor-framebuffer`; KWin Plasma 6; wlroots 0.17 (`wlr_fractional_scale_v1_notify_scale`); Hyprland `monitor = eDP-1,scale,1.5`
- Toolkit integration: GTK4 `GdkWaylandSurface` auto-binds `wp_fractional_scale_v1`; `cairo_surface_set_device_scale`; Qt6 `QWaylandWindow::setPreferredScale`; Electron/Chromium `--ozone-platform=wayland`; SDL3 `SDL_GetWindowPixelDensity`; Firefox `MOZ_ENABLE_WAYLAND=1`
- Mixed-DPI multi-monitor: `wl_surface.enter/leave`; compositor re-sends `preferred_scale`; `wlr-randr --scale`; `kscreen-doctor`; cursor scaling; `hyprcursor`; "blurry on move" transient artefact
- XWayland: integer-scale only; `Xwayland -dpi 144` for 1.5× equivalent; X11 apps blur at fractional scale; `MOZ_ENABLE_WAYLAND=1` escapes XWayland
- Subpixel rendering: `FT_LOAD_TARGET_LCD`; Pango fractional device units; `hintstyle=hintslight`; fontconfig LCD filter; `gsettings antialiasing rgba`
- **Integrations**: Ch20 (wl_output.scale), Ch21 (wlroots fractional scale), Ch22 (Mutter/KWin scale implementation), Ch39 (GTK/Qt toolkit DPR), Ch105 (font rendering HiDPI), Ch130 (staging protocol lifecycle)

### Chapter 139: DRM Hardware Overlay Planes and Composition Bypass *(Part I)*

- Display engine architecture: plane → CRTC → Encoder → Connector; Intel ICL+ up to 7 planes/pipe; AMD DCN 3.x 4 planes/display; Qualcomm DSS 6 planes; `DRM_PLANE_TYPE_PRIMARY/OVERLAY/CURSOR`
- KMS plane objects: `drm_plane` struct; `drmSetClientCap(DRM_CLIENT_CAP_UNIVERSAL_PLANES, 1)`; `drmModeGetPlaneResources`; plane properties: `FB_ID`, `CRTC_ID`, `SRC_X/Y/W/H` (16.16 fixed-point), `CRTC_X/Y/W/H`, `ZPOS`, `ALPHA` (0–0xFFFF), `PIXEL_BLEND_MODE`, `ROTATION`
- Atomic commit: `drmModeAtomicAddProperty`; `DRM_MODE_ATOMIC_TEST_ONLY` for hardware capability test; `DRM_MODE_ATOMIC_NONBLOCK + DRM_MODE_PAGE_FLIP_EVENT`; disable: `CRTC_ID = 0`
- DRM fourcc formats: `DRM_FORMAT_ARGB8888`, `DRM_FORMAT_NV12`, `DRM_FORMAT_P010`, `DRM_FORMAT_ABGR2101010`; modifiers: `DRM_FORMAT_MOD_LINEAR`, `I915_FORMAT_MOD_Y_TILED`, `AMD_FMT_MOD_DCC`; `IN_FORMATS` blob; `drmModeAddFB2WithModifiers`
- Composition bypass / direct scanout: prerequisites (format+modifier match, no scaling, full coverage, fence); `DRM_MODE_ATOMIC_TEST_ONLY` scan-out test; wlroots `wlr_drm_connector_test_primary_fb`; power savings: ~2.5 W GPU (direct) vs ~4.5 W (composited) for full-screen video
- Z-order and alpha: `ZPOS` layering; `ALPHA` plane-wide multiplier; `PIXEL_BLEND_MODE`: `None/Pre-multiplied/Coverage`; cursor: `drmModeMoveCursor` without atomic commit
- Video overlay: `COLOR_ENCODING` (BT.601/BT.709/BT.2020); `COLOR_RANGE` (limited/full); zero-copy VA-API NV12 DMA-BUF → KMS plane; hardware scaling (`CRTC_W/H ≠ SRC_W/H`); `mpv --vo=drm`
- Compositor allocation: greedy Z-order algorithm; `ATOMIC_TEST_ONLY` per surface; wlroots `wlr_drm_plane_alloc`; Mutter `MetaKmsUpdate` integration; `-EINVAL` fallback to GPU composition
- Debugging: `cat /sys/kernel/debug/dri/0/state`; Intel `i915_display_info`; AMD `amdgpu_dm_visual_confirm`; `modetest -P plane_id@crtc_id:WxH+X+Y:format`; `WLR_DRM_DEBUG=1`
- **Integrations**: Ch2 (KMS — planes are first-class KMS objects), Ch3 (HDR planes, P010 format), Ch4 (DMA-BUF — plane surfaces), Ch20 (compositor direct scanout), Ch21 (wlroots plane allocation), Ch22 (Mutter/KWin plane allocation), Ch38 (VA-API NV12 to plane), Ch74 (HDR10 video overlay), Ch123 (writeback connector captures all planes)

### Chapter 140: HDMI and DisplayPort Audio on Linux *(Part VI-B — Display Services)*

- HDMI audio protocol: Audio Sample Packets in TMDS blanking; LPCM (8ch/192kHz/24-bit), AC-3, DTS, DTS-HD, Dolby TrueHD/Atmos, DTS:X; ACR: `f_audio = 128 × f_TMDS × (N/CTS)`; HDMI 2.1 eARC (full TrueHD/DTS-HD pass-through, 37 Mbps)
- EDID CEA-861: Short Audio Descriptors (SAD, 3 bytes each): format code, max channels, sample rates, bit depths/bitrate; ELD (84 bytes): `monitor_name_length`, `sad_count`, `conn_type`, SAD array; `drm_edid_to_eld(connector, drm_edid)`; `cat /proc/asound/card*/eld*`
- `drm_audio_component`: `drm_audio_component_ops` vtable: `get_power`, `put_power`, `codec_wake_override`, `get_cdclk_freq`, `sync_audio_rate`, `get_eld`; Linux component framework `component_add/bind`; Intel `i915_audio_component` in `intel_audio.c`; AMD `amdgpu_dm_audio.c`; `intel_audio_codec_enable` → `pin_eld_notify`
- ALSA HDA Intel driver: `snd_hda_intel` + `snd_hda_codec_hdmi` (`patch_hdmi.c`); one PCM stream per port; HDA verbs: `AC_VERB_SET_STREAM_FORMAT`, `AC_VERB_SET_CHANNEL_STREAMID`; `aplay -l`; AMD `snd_hda_codec_atihdmi`; NVIDIA `patch_nvhdmi.c`
- PipeWire/WirePlumber: ALSA monitor discovers HDMI PCM devices; `wpctl status`; WirePlumber udev watch → ELD re-read; IEC 61937 passthrough; `mpv --audio-spdif=ac3,dts,truehd`
- Hotplug lifecycle: HPD interrupt → DRM hotplug → `drm_edid_to_eld` → `pin_eld_notify` → ALSA PCM create/destroy → udev → WirePlumber; suspend/resume audio loss; `hda-verb GET_PIN_SENSE`; `hdajacksensetest`
- DisplayPort audio: Main Link idle Audio Stream Packets; synchronous clock (no ACR); DPCD `DP_AUDIO_SINK_CAPS`; 8ch/192kHz LPCM; MST per-port ELD; ARC vs eARC bandwidth table; `DRM_ELD_CONN_TYPE_DP`
- CEC: single-wire bidirectional on HDMI pin 13; `drivers/media/cec/`; `/dev/cec0`; `cec-ctl`; `CEC_MSG_ACTIVE_SOURCE`, `STANDBY`, `SET_STREAM_PATH`; `drm_dp_cec.c` for DP CEC-over-AUX; `libcec`; Kodi CEC integration
- **Integrations**: Ch2 (KMS connector/EDID), Ch3 (ARC/eARC for HDR audio), Ch38 (PipeWire HDMI routing), Ch128 (DP MST per-port audio), Ch139 (plane mode change triggers audio ACR recalc)

### Chapter 141: Vulkan Cooperative Matrices and GPU ML Acceleration *(Part VII-A — GPU APIs)*

- MMA hardware: NVIDIA Tensor Cores (Volta+); AMD Matrix Cores (CDNA MFMA, RDNA3 WMMA); Intel XMX/DPAS (Xe-HPG+); peak ML TFLOPS; `C[M×N] += A[M×K] × B[K×N]` tile decomposition
- `VK_KHR_cooperative_matrix`: `VkCooperativeMatrixPropertiesKHR` (M/N/K/AType/BType/CType/ResultType/scope); `vkGetPhysicalDeviceCooperativeMatrixPropertiesKHR`; `VK_SCOPE_SUBGROUP_KHR`; subgroup-cooperative tile fragments
- GLSL: `#extension GL_KHR_cooperative_matrix`; `coopmat<float16_t, gl_ScopeSubgroup, 16, 16, gl_MatrixUseA>`; `coopMatLoad/Store/MulAdd/Construct`; `gl_CooperativeMatrixLayoutRowMajor`; SPIR-V: `OpCooperativeMatrixMulAddKHR`
- Hardware ISA: NVIDIA PTX `mma.sync.aligned.m16n8k16`; AMD `v_wmma_f32_16x16x16_f16` / `v_wmma_i32_16x16x16_iu8` (RDNA3 wave32); Intel `dpas.8x8.f32.hf.hf` systolic array (Xe-HPG)
- RADV (RDNA3+ gfx11+): `radv_physical_device_get_cooperative_matrix_properties`; `nir_lower_cooperative_matrix` → ACO `v_wmma_*` emission; `RADV_DEBUG=shaderinfo` VGPR count
- ANV (DG2/Arc+): `anv_get_cooperative_matrix_props`; `nir_lower_cooperative_matrix` → `intel_nir_lower_dpas.c`; Xe-HPG INT8 274 TOPS / FP16 137 TFLOPS peak
- Quantisation: INT8 GEMM accumulate INT32; `saturatingAccumulation`; FP8 E4M3/E5M2 (RDNA3 RADV mid-2026 target); BF16 `v_wmma_f32_16x16x16_bf16`; GGUF block quantisation dequant shader
- Practical usage: `llama.cpp` Vulkan backend (`GGML_VULKAN=1`): 45–70 tok/s Llama 3.1 8B Q4_K_M on RX 7900 XTX; `stable-diffusion.cpp`; VkFFT; tinygrad Vulkan
- Tuning: VGPR pressure vs occupancy; L1 cache tiling (2 KB tile fits 32 KB L1); `subgroup_size_control = FULL` for wave32; peak TFLOPS formula: `CUs × (8192/4) × clk_GHz`
- **Integrations**: Ch24 (Vulkan API), Ch25 (GPU compute), Ch108 (ROCm rocBLAS — same WMMA/MFMA HW), Ch124 (OpenCL cl_khr_cooperative_matrix), Ch127 (mesh+ML hybrid), Ch133 (async compute queue for GEMM), Ch135 (coop matrix denoising after RT)

---

### Section additions to existing chapters (cross-chapter content)

**Chapter 28 §9 — RTX Remix** (added): open-source game remastering toolkit (MIT, RTX Remix 1.0, March 2025); `dxvk-remix` fork of DXVK intercepting D3D8/9 → Vulkan + OptiX path tracing injection; USD asset overlay via `over` composition arc; DLSS 4 MFG and NRC in RTX Remix 1.0; Linux via Proton/Wine (`PROTON_NO_DXVK=1`, driver ≥ 535); Portal RTX and Half-Life 2 RTX as reference titles; connection to Ch28 DXVK (§3), dxvk-nvapi (§6), Ch67 (OptiX), Ch68 (DLSS), Ch70 (NRC)

**Chapter 30 §6.3 expansion — NvPerf SDK** (added): programmatic GPU counter collection via `libnvperf_host.so` + `libnvperf_target.so`; Linux permissions — `perf_event_paranoid ≤ 0` or `CAP_SYS_ADMIN`; `NvperfVulkanLoadDriver`/`NvperfVulkanInitDevice`; `RangeProfilerVulkan` with `PushRange`/`PopRange` wrapping `VkCommandBuffer` submissions; key counter groups (sm, l1tex, lts, dram); `ncu --metrics` CLI path; `nsys profile --trace=cuda,vulkan,nvtx` system timeline capture

**Chapter 57 §7.4 — NVIDIA Video Codec SDK direct API** (added): SDK as header-only (`nvEncodeAPI.h`, `cuviddec.h`) linking against driver-shipped `libnvidia-encode.so`/`libnvcuvid.so`; NVENC runtime-load pattern (`dlopen("libnvidia-encode.so.1")`, `NvEncodeAPICreateInstance`); `NV_ENC_OPEN_ENCODE_SESSION_EX_PARAMS` with `NV_ENC_DEVICE_TYPE_CUDA`; `NV_ENC_INITIALIZE_PARAMS` with preset `NV_ENC_PRESET_P4_GUID`; lookahead (`enableLookahead`, `lookaheadDepth=32`), adaptive quantisation (`enableAQ`), async mode; NVDEC `cuvidCreateDecoder` for zero-copy CUDA-resident decode; when to use SDK vs. FFmpeg (broadcast, sub-10ms latency, zero-copy NVDEC→CUDA→NVENC)

**Chapter 68 §11 — TensorRT** (added): build-then-deploy model — ONNX → `trtexec` engine plan (GPU-arch-specific, driver-version-sensitive); `trtexec --onnx --saveEngine --fp16 --timingCacheFile`; C++ API: `IBuilder`, `INetworkDefinition` (`kSTRONGLY_TYPED` flag), `NvOnnxParser::createParser`, `IOptimizationProfile` for dynamic resolution shapes, `BuilderFlag::kFP16`; runtime: `ICudaEngine::deserialize`, `IExecutionContext::setTensorAddress`, `enqueueV3(cudaStream)`; Vulkan interop via `VK_KHR_external_memory` (Ch25); INT8 calibration with `IInt8EntropyCalibrator2`; when to use (custom neural denoisers, NeRF decoders, RTXNTC-equivalent custom models)

**Chapter 69 §12 — PhysX 5 and NVIDIA Warp** (added): PhysX 5.6.1 (Apache 2.0, github.com/NVIDIA-Omniverse/PhysX); simulation types — rigid body (`PxRigidDynamic`), soft body (`PxSoftBody` FEM), PBD cloth/fluids (`PxPBDParticleSystem`), articulations (`PxArticulationReducedCoordinate`); GPU pipeline — `eENABLE_GPU_DYNAMICS`, `PxBroadPhaseType::eGPU`, `PxCudaContextManager`; PhysX → USD bridge via `UsdPhysicsRigidBodyAPI`, `PhysxSchema`; USDRT Fabric zero-copy geometry update path; NVIDIA Warp (BSD-3, pip install `warp-lang`) — Python-like GPU kernels JIT-compiled to PTX; `@wp.kernel` with `wp.atomic_add`, `wp.tid()`; differentiable kernels via `wp.Tape`; Warp ↔ USD Fabric zero-copy via DLPack (`omni.warp.from_fabric`); Newton physics engine (GTC 2025) built on Warp

---

## Part XXI — Platform, Legacy, and History

### Chapter 95: X11/Xorg Architecture and the DRI Legacy Stack

- Why X11 matters: still the default on many enterprise Linux systems; XWayland bridges Wayland compositors to X11 clients; understanding the legacy explains Wayland's design choices
- X11 architecture: client-server model; display string (`DISPLAY=:0`); X protocol over Unix socket; request/reply/event model; atoms and properties; window hierarchy; XCB vs Xlib
- XServer extension model: core protocol + GLX + Composite + Damage + Fixes + Present + RANDR + RENDER; how extensions are negotiated and loaded
- GLX: binding OpenGL to X11 windows; `glXChooseVisual`, `glXCreateContext`, `glXMakeCurrent`, `glXSwapBuffers`; GLX 1.3 FBConfigs; GLX extensions (GLX_ARB_create_context, GLX_EXT_swap_control)
- AIGLX (Accelerated Indirect GLX): how DRI-based hardware acceleration was brought to compositing managers; `__DRI_DRIVER` env var; `glXGetProcAddressARB`; Compiz as the canonical AIGLX consumer
- DRI1 → DRI2 → DRI3 evolution: DRI1 (shared memory kernel buffer, `drmMap`); DRI2 (buffer passing by name, races with Compiz); DRI3 (buffer passing by fd, `DRI3Open`, `DRI3PixmapFromBuffer`); why each transition was necessary
- The Composite extension: `CompositeRedirectSubwindows`; off-screen backing store per window; how compositors (Compiz, Mutter-X11, KWin-X11) redirect and composite windows with OpenGL
- XRANDR: display configuration; `RRSetCrtcConfig`, `RRCreateMode`; multi-monitor without restart; connection to KMS via `xf86-video-modesetting` driver
- The Present extension: `XPresentPixmap`; synchronised buffer swap with VBLANK; MSC (media stream counter); how XWayland uses Present to deliver frames to the Wayland compositor
- XWayland bridge: rootless mode; `_XWAYLAND_MAY_GRAB_KEYBOARD`; X11 clipboard ↔ Wayland clipboard; HiDPI scaling; how DRI3 buffers flow from Mesa through XWayland to a `wl_buffer` and into the compositor; explicit sync handshake (`linux-drm-syncobj-v1`)
- Security comparison: X11 (any client can read any window's contents, keylog globally); Wayland (per-surface isolation, no global input capture without portal)
- **Integrations**: Ch1 (DRM — DRI3 uses DRM render nodes), Ch4 (GBM — DRI3 buffers are GBM BOs), Ch20 (Wayland protocol — XWayland is a Wayland client), Ch23 (Ch23 covers XWayland at the application level; this chapter covers the X11 side), Ch30 (debugging — `LIBGL_DEBUG`, `LIBGLX_DEBUG`, `xlsatoms`, `xdpyinfo`)

### Chapter 99: Automotive and Embedded Linux Graphics *(Part II)*

- Automotive Linux landscape: AGL (Automotive Grade Linux), GENIVI Alliance history, Android Automotive vs Linux IVI; regulatory requirements (functional safety ISO 26262, ASPICE)
- SoC platforms: NXP i.MX8 (etnaviv/Vivante GPU), Renesas R-Car H3/V3H (SGX544), Qualcomm SA8155P (Adreno 640), TI TDA4VM (PowerVR), Texas Instruments J721E; the DRM/KMS driver landscape per SoC
- AGL UCB (Unified Code Base): Yocto-based; `meta-agl`, `meta-agl-demo`; AFB (Application Framework Binder); WindowManager based on wlroots; `agl-compositor` (wlroots-based); IVI shell protocol (`ivi-application`, `ivi-layout`)
- Weston on embedded: minimal Wayland compositor; `weston.ini`; DRM backend with single-CRTC output; `weston-terminal`; Panel plugin; headless backend for CI; EGL/GLES2 rendering
- cage: single-application Wayland compositor (`cage -s -- my_app`); use case for digital instrument clusters and HMI; source at https://github.com/cage-kiosk/cage
- Qt for Automotive: Qt IVI (In-Vehicle Infotainment) module; `QtApplicationManager`; multi-process (Wayland) vs single-process model; `QML` cluster rendering at 60 FPS; Qt Safe Renderer for ASIL-B functional safety displays; integration with `agl-compositor`
- Yocto/Buildroot for embedded graphics: `meta-openembedded/meta-oe` OpenGL/Wayland layers; `IMAGE_INSTALL += weston mesa wayland`; GPU firmware packaging; sysroot cross-compilation; `devtool` for driver iteration
- GPU driver bring-up on new SoCs: device tree nodes (`drm` compatible string, `reg`, `interrupts`, `clocks`, `power-domains`); `simple-framebuffer` as fallback; DRM driver registration (`drm_dev_register`); IOMMU integration for secure GPU memory; bring-up debug via `drm_info`
- Display bring-up: LVDS, MIPI DSI, HDMI; `drm_panel` for display panels; `panel-simple` driver; EDID via DDC I²C; DSI bridge chips (TC358764, SN65DSI86)
- Remote rendering: cluster GPU rendering frame forwarded over SOME/IP or DoIP to instrument cluster SoC; `GStreamer` pipeline for H.264 over Ethernet in automotive cabin networks
- **Integrations**: Ch1 (DRM — platform drivers), Ch2 (KMS atomic — display bring-up), Ch21 (wlroots — agl-compositor), Ch39 (Qt/GTK — Qt IVI), Ch57 (FFmpeg — camera + rear-view video), Ch90 (Panfrost/Lima — ARM Mali in i.MX8), Ch100 (etnaviv — Vivante in i.MX6/8), Ch92 (Raspberry Pi — comparable embedded bring-up)

### Chapter 197: The Linux Graphics Stack in Context — Comparison with Windows and macOS *(Part XXI)*

- Structural philosophy: bazaar (Linux), cathedral (Windows), monastery (macOS) — how governance models drive technical outcomes
- Innovations where Linux leads: Mesa NIR (open universal shader IR), DMA-BUF (cross-process zero-copy), explicit sync as Wayland protocol primitive, Rust GPU drivers (Nova, Asahi), community reverse engineering quality, DXVK/VKD3D-Proton translation layer performance
- Windows leads — and Linux's response: WDDM TDR GPU hang recovery vs. drm_gpuvm isolation work; DirectX 12 Ultimate feature set vs. Vulkan extension parity table; DirectStorage vs. io_uring + P2P DMA infrastructure; DirectX Agility SDK vs. Steam Runtime + Flatpak Mesa delivery; PIX tooling vs. RenderDoc + RGP + Nsight; gaming ecosystem vs. Steam Play/Proton; DirectML/DLSS vs. ROCm + FSR + llama.cpp Vulkan
- macOS leads — and Linux's response: Apple unified memory vs. HMM on AMD APU / Intel iGPU / ARM SoCs; Metal ergonomics vs. VK_KHR_dynamic_rendering + wgpu; HDR 2016 vs. KWin 6 + wp-color-management-v1 (2024); ProRes hardware vs. software FFmpeg; ANE on-device ML vs. /dev/accel/ XDNA + Qualcomm HTP + llama.cpp Vulkan; compositor stability vs. KWin 6 + explicit sync
- Windows and macOS catching up to Linux: Vulkan on Windows; WSL2 GPU passthrough (dxgkrnl + Mesa D3D12); MoltenVK; Game Porting Toolkit; Asahi architectural leverage
- Velocity comparison: Mesa quarterly cadence vs. D3D12 Agility SDK vs. Apple hardware-launch day; distribution lag vs. Steam Runtime
- Platform comparison tables: overall stack characteristics; feature parity matrix (ray tracing, mesh shaders, VRS, HDR, VRR, DirectStorage, hardware ML, AV1 encode)
- Strategic convergence: SPIR-V + WGSL as lingua franca; shader path across all three platforms from a single WGSL source; competitive differentiation moving to hardware ISA compiler quality
- **Integrations**: Ch103 (history — context for structural choices), Ch28 (DXVK/VKD3D-Proton), Ch119 (Zink), Ch52 (Firefox/naga), Ch34 (ANGLE), Ch118 (NAK), Ch196 (GPU firmware), Ch88 (NPU/accel subsystem)

### Chapter 103: The Linux Graphics Stack: History and Design Philosophy

- Origins: XFree86 in the early 1990s; the DRI project (1998) — Precision Insight, VA Linux, SGI; Jens Owen and the original `dri.h`; Brian Paul's Mesa (1993, public domain from day one)
- The DRI architecture debate: why client-side rendering won over server-side (latency, complexity, OpenGL state machine mismatch with X11 protocol)
- Mesa milestones: OpenGL 1.0 software renderer (1993) → Glide (3dfx Voodoo, 1997) → DRI hardware acceleration (1998) → Gallium3D (Keith Whitwell, Zack Rusin, 2008) → NIR (Connor Abbott, 2014) → ACO (valve-funded, 2019)
- The kernel DRM story: agpgart (1999) → early `drm.h` (2000) → KMS (Jesse Barnes, Keith Packard, 2008) → GEM (Intel, 2008) → TTM (Tungsten Graphics, 2007) → DMA-BUF (Sumit Semwal, 2012) → atomic modesetting (2015)
- Nouveau: Ian Romanick's initial attempt (2005), Martin Peres, Ben Skeggs; the GSP-RM transition (2022); NVK (Faith Ekstrand, 2022) as the vindication of a 17-year reverse engineering effort
- The Wayland story: Kristian Høgsberg's frustration with X11 compositor bugs; `wayland-0.1` (2008); Weston reference compositor; the long adoption: Ubuntu (2017), Fedora (2017), GNOME (2017), KDE (2021); Mir's death; why Wayland's security model won
- AMD's open source pivot: catalyst → fglrx → AMDGPU open (2016); RADV (Bas Nieuwenhuizen, Dave Airlie, 2016); ACO (Valve, 2019); RADV beating AMD's own driver in benchmarks by 2020
- NVIDIA's open source journey: decades of "never", nvidia-open (2022) — GSP-only, not truly open; NVK (truly open, Mesa, 2022); the ongoing tension between open and proprietary
- ARM GPU openness: reverse engineering Mali (Panfrost team); Apple Silicon (Asahi team); the constraint of no public ISA documentation; how reverse engineering produced production-quality drivers
- Design philosophy recurring themes: separation of display and render (DRM primary vs render nodes); explicit synchronisation (from implicit fences to DRM syncobj); buffer sharing (from shared memory to DMA-BUF); layering (kernel → libDRM → Mesa → app, with no layer skipping)
- The road ahead: Rust in the kernel (`rust-gpu`, `nova`); AI/ML workloads and the accel subsystem; display colour pipelines; the next 10 years
- **Integrations**: all chapters — this is the narrative thread connecting the entire book

---

### Chapter 96: libcamera and the Linux Camera Stack *(Part VII-B — Multimedia Frameworks)*

- libcamera's place: the unified camera abstraction above V4L2 and platform-specific ISPs; https://libcamera.org/; replaces the fragmented vendor HAL approach
- V4L2 media controller framework: `media_device`, media graph (entities, pads, links); `media-ctl --print-topology`; `MEDIA_IOC_SETUP_LINK`; pipeline configuration; sensor → CSI-2 receiver → ISP → memory entities
- libcamera architecture: `CameraManager` → `Camera` → `CameraConfiguration` → `Request` → `FrameBuffer` → `FrameBufferAllocator`; the event-driven `Request::completed` signal
- PipelineHandler: per-platform subclass (`RkISP1`, `IPU3`, `RPi`, `Simple`, `VIMC`); pipeline configuration, buffer allocation, request queuing; https://libcamera.org/api-html/classlibcamera_1_1PipelineHandler.html
- ISP algorithms: `IPAModule` (Image Processing Algorithms); AEC/AGC (auto exposure/gain), AWB (auto white balance), LSC (lens shading correction), CCM (colour correction matrix), gamma; IPA runs as a separate process for security
- DMA-BUF integration: `FrameBuffer` backed by DMA-BUF fds; zero-copy camera→display via `linux-dmabuf-unstable-v1`; `EGLImage` from DMA-BUF for GL texture import; V4L2 M2M encode via DMA-BUF
- PipeWire camera integration: `libcamera-apps` via PipeWire; `pw_stream` for camera frames; Wayland `xdg-desktop-portal` camera access; `OBS-Studio` via PipeWire camera source
- GStreamer integration: `libcamerasrc` element; YUV format handling; `videorate`, `videoconvert`; full pipeline: `libcamerasrc ! video/x-raw,format=NV12 ! videoconvert ! autovideosink`
- Platform coverage: Raspberry Pi (RkISP1 + Unicam), Intel IPU3/IPU6, RockChip RKISP1, simple pipeline for USB cameras (UVC), Qualcomm Camss; hardware-specific quirks
- **Integrations**: Ch26 (VA-API — hardware encode of camera output), Ch38 (PipeWire), Ch57 (FFmpeg — libcamerasrc feeds FFmpeg), Ch85 (Android Camera HAL comparison), Ch92 (Raspberry Pi — PipelineHandlerRPi)

### Chapter 176: OpenCASCADE Technology — The BRep Kernel and 3D Visualization Stack *(Part XI)*

- Why OCCT: exact BRep geometry (B-spline/NURBS, not triangles), topology/geometry separation, STEP/IGES data exchange; powers FreeCAD, Salome, Code_Aster; OCCT 8.0.0p1 (June 2026), C++17 mandatory
- Architecture: six modules — FoundationClasses (`TKernel`, `TKMath`, `gp`, `BVH`, `NCollection`), ModelingData (`TKBRep`, `TKG3d`, `TKG2d`), ModelingAlgorithms (`TKBO`, `TKFillet`, `TKPrim`, `TKOffset`, `TKMesh`), Visualization (`TKOpenGl`, `TKV3d`, `TKService`), DataExchange (`TKDESTEP`, `TKDEIGES`, `TKDEGLTF`, `TKDEOBJ`, `TKDESTL`), ApplicationFramework (`TKCAF`, `TKXCAF`)
- BRep model: topology (`TopoDS_Shape`, `TopoDS_Face/Edge/Wire/Solid`, `TopLoc_Location`, `TopAbs_Orientation`) vs geometry (`Geom_Surface`, `Geom_Curve`, `Geom2d_Curve`); `BRep_TFace`/`BRep_TEdge` bridging; tolerance invariant `Tol(Vertex) >= Tol(Edge) >= Tol(Face)`; `TopoDS_Iterator`, `TopExp_Explorer`
- `BRep_Builder` low-level: `MakeFace(surface, tol)`, `UpdateEdge(curve, tol)`, `MakeVertex(pt, tol)`, `MakeFace(triangulation)`; `BRepBuilderAPI_*` high-level wrappers
- Modeling algorithms: Boolean ops (`BRepAlgoAPI_Fuse/Cut/Common`, multi-shape modern API, `SetRunParallel`, `BOPAlgo_Options::SetParallelMode`); primitives (`BRepPrimAPI_MakeBox/Cylinder/Sphere/Cone`, `MakePrism`, `MakeRevol`); fillets (`BRepFilletAPI_MakeFillet`, constant/variable radius, `Law_Function`); offsets (`BRepOffsetAPI_MakeThickSolid`, `MakePipe`, `ThruSections`); mesh (`BRepMesh_IncrementalMesh`, linear + angular deflection, `Poly_Triangulation`)
- Visualization stack: `AIS_InteractiveContext` → `V3d_Viewer/View` → `Graphic3d_GraphicDriver` → `OpenGl_GraphicDriver` → GLX (X11) or EGL (Wayland/headless); `AIS_Shape`, `AIS_ColoredShape`, sub-shape selection modes (0=whole, 1=vertex, 2=edge, 4=face); `OpenGl_GraphicDriver::InitEglContext(EGLDisplay, EGLContext, EGLConfig)` for Wayland; shader-based rendering (Blinn-Phong + PBR since 7.5.0); Z-layers
- BVH picking: `Select3D`, `SelectMgr_ViewerSelector`, `BVH_Tree<Real,3>`, `BVH_BinnedBuilder`, `BVH_LinearBuilder`; `AIS_InteractiveContext::MoveTo` → BVH traversal → closest selectable entity
- Vulkan status: NO shipped Vulkan backend as of 8.0.0p1; prototype tracked as issue #30631 (unmerged); all rendering via `TKOpenGl` (OpenGL 3.2+ core minimum)
- Data exchange: STEP (`STEPControl_Reader/Writer`, `STEPCAFControl_*` for assemblies + colors; 75% faster in 8.0.0); IGES (`IGESControl_*`); STL (`StlAPI_Writer/Reader`); glTF 2.0 (`RWGltf_CafWriter/Reader`, since 7.5.0, Draco since 7.7.0, Y-up/metres via `RWMesh_CoordinateSystemConverter`); OBJ (`RWObj_CafReader`, since 7.4.0); PLY (`TKDEPLY`)
- XDE (`TKXCAF`): `TDocStd_Document`, `XCAFDoc_ShapeTool` (assembly tree), `XCAFDoc_ColorTool`, `XCAFDoc_LayerTool`; assembly references via `TNaming_NamedShape` + `TopLoc_Location`
- OCAF: `TDF_Label` tree, `TNaming_Builder`, undo/redo (`doc->NewCommand()/CommitCommand()/Undo()/Redo()`), binary persistence (`TKBin`, `.cbf`), XML persistence (`TKXml`, `.xbf`)
- FreeCAD integration: `Part::TopoShape` wrapping `TopoDS_Shape`; `Part::Feature` parametric DAG; Python `Part.makeBox/fuse/cut` → OCCT calls; FreeCAD's own document system (not OCAF); Coin3D viewer (not OCCT AIS)
- Build and packaging: cmake flags (`USE_OPENGL`, `USE_OPENMP`, `USE_VTK=OFF` by default in 8.0), Ubuntu split packages (`libocct-foundation-dev`, `libocct-modeling-data-dev`, etc.), Fedora `opencascade-devel`; linking toolkit names (`libTKernel`, `libTKBRep`, `libTKOpenGl`, `libTKDESTEP`, …)
- **Integrations**: Ch12 (Mesa OpenGL ICD — `OpenGl_GraphicDriver` dispatches through Mesa), Ch20 (Wayland — EGL + `wl_egl_window` for `InitEglContext`), Ch42 (Blender — separate geometry kernel; FreeCAD/Blender STL exchange), Ch64 (glTF — `RWGltf_CafWriter` produces glTF 2.0 with coordinate conversion), Ch77 (GLSL shaders — OCCT PBR shaders compiled at runtime), Ch107 (headless rendering — EGL `EGL_EXT_platform_device` for CI), Ch113 (CGAL — complementary geometry: CGAL for mesh repair before OCCT BRep reconstruction), Ch150 (EGL and DMA-BUF — OCCT EGL surface creation)

---

### Chapter 97: Unreal Engine 5 on Linux *(Part XI)*

- UE5 Linux support: EpicGames' commitment to Linux as a first-class target; Steam Deck as the key platform; Proton vs native builds; supported GPU tiers
- RHI (Rendering Hardware Interface): `FRHICommandList`, `FRHICommandListImmediate`; the abstraction over Vulkan, D3D12, Metal; `FVulkanCommandListContext`; `IRHICommandContext` virtual interface; render thread vs RHI thread vs GPU thread
- Vulkan RHI backend: `FVulkanDevice`, `FVulkanQueue`; descriptor set layout caching (`FVulkanDescriptorSetLayout`); VMA integration for GPU memory; pipeline state object cache; shader compilation (`FVulkanShader`, SPIR-V via DXC); `RHICreateVertexBuffer`, `RHILockBuffer`
- Nanite virtualized geometry: streaming triangle clusters (128 triangles/cluster); BVH over clusters; software rasterizer for small triangles (Visibility Buffer pass); mesh shader path for large triangles; persistent cull + draw on GPU; `Nanite::FRenderer`; cluster LOD selection; streaming from disk
- Lumen global illumination: screen-space probe capture; radiance cache (world-space probes); surface cache (albedo + normal per mesh); ray tracing fallback for Lumen reflections; Lumen on Linux: hardware RT on Vulkan (RTX/RDNA2+), software Lumen on all hardware
- Niagara particle system: GPU simulation via compute shaders; `FNiagaraGPUSystemTick`; particle data as structured buffers; interface to Physics, Audio; Vulkan async compute for particle simulation
- Shader compilation on Linux: UE5 shader compiler (`ShaderCompileWorker`); HLSL → SPIR-V via DXC (`DxcCreateInstance`); offline shader DDC (Derived Data Cache); Shader Model 6 features on Vulkan via SPIRV-Cross; `r.Vulkan.EnableShaderDebugInfo`
- Linux packaging: `UnrealPak`, `BuildCookRun`; Steam depot upload; flatpak wrapping for distribution; missing Steam Deck gotchas (Proton vs native, controller input, screen resolution)
- **Integrations**: Ch16 (Mesa Vulkan common — UE5 submits to RADV/ANV/NVK), Ch18 (RADV — primary GPU driver), Ch24 (Vulkan — RHI submits to VkQueue), Ch56 (ray tracing — Lumen RT), Ch61 (modern Vulkan extensions — mesh shaders, dynamic rendering), Ch77 (shader toolchain — DXC for HLSL→SPIR-V)

### Chapter 98: WebAssembly and WebGPU as a Deployment Target *(Part X)*

- The portability promise: a single codebase targeting Vulkan (native Linux), WebGPU (browser), and Metal (macOS) via `wgpu` (Rust) or Emscripten (C++)
- WebAssembly fundamentals: linear memory model; WASM binary format (sections, types, functions, exports, imports); WASI for filesystem/networking; SIMD extensions (`v128`); threading via SharedArrayBuffer + Atomics
- Emscripten → WebGPU: `emcc -sUSE_WEBGPU=1`; `webgpu.h` C API in Emscripten; `emscripten_webgpu_get_device()`; calling sequence: JS `navigator.gpu.requestAdapter()` → C `wgpuAdapterRequestDevice()`; framebuffer via `wgpuSwapChainGetCurrentTextureView()`; shared vertex/fragment WGSL shaders between native and WASM targets
- wgpu (Rust): `wgpu::Instance`, `wgpu::Adapter`, `wgpu::Device`, `wgpu::RenderPipeline`; backend selection (`Backends::VULKAN` native, `Backends::BROWSER_WEBGPU` in WASM); `wasm-bindgen` + `web-sys` for JS interop; `winit` integration; `cargo build --target wasm32-unknown-unknown --features webgpu`
- WGSL shaders: syntax; built-in functions; binding model (`@binding`, `@group`); compute shaders in WGSL; how Tint (the WGSL compiler in Dawn) validates and translates to SPIR-V for Vulkan, MSL for Metal, HLSL for D3D12
- How browser WebGPU maps to Linux Vulkan: Dawn's Vulkan backend (`src/dawn/native/vulkan/`); `VulkanBackend::CreateAdapterFromDevice`; command buffer translation; swapchain via `VkSwapchainKHR`; the performance gap: native Vulkan vs browser WebGPU (typically 5–15% overhead from Dawn + V8 JIT overhead)
- WASM SIMD for CPU-side work: `wasm_f32x4_add`, `wasm_i32x4_mul`; use alongside WebGPU for mixed CPU/GPU workloads (physics on WASM SIMD, rendering on WebGPU)
- Use cases: portable GPU tools (shader toy-style live coding, GPU-accelerated image editors), game engines targeting web + desktop (Bevy's WASM backend, Godot Web export), ML inference (ONNX Runtime Web with WebGPU EP)
- **Integrations**: Ch35 (Dawn/WebGPU — WASM uses the same Dawn backend), Ch40 (Bevy/wgpu — wgpu targets both native Vulkan and WASM WebGPU), Ch41 (Godot — Web export uses WebGPU), Ch61 (SPIR-V/WGSL relation), Ch77 (shader toolchain — Tint is part of the toolchain), Ch91 (MLIR — IREE targets WebAssembly as well as Vulkan)

### Chapter 100: etnaviv: The Vivante GPU Open Driver *(Part II)*

- Vivante GC-series GPU family: GC400 (2D only), GC2000 (OpenGL ES 2.0, shaders), GC7000 (OpenGL ES 3.x, Vulkan-capable); deployment in NXP i.MX6, i.MX8, Marvel ARMADA, Amlogic S905; https://github.com/etnaviv
- Reverse engineering history: Christian Gmeiner, Lucas Stach (Pengutronix); the `envytools`-inspired approach; early FOSS drivers (2013); mainline kernel merge (2015); https://github.com/etnaviv/etna_viv
- etnaviv kernel driver (`drivers/gpu/drm/etnaviv/`): `etnaviv_gem` GEM objects (shmem-backed); `etnaviv_cmdbuf` command buffer; `etnaviv_gpu` per-core struct; `etnaviv_sched` (drm_gpu_scheduler); `ETNAVIV_PARAM_*` capabilities query; `DRM_IOCTL_ETNAVIV_GEM_SUBMIT`
- Vivante ISA: unified shader architecture; vec4 operations; texture sampling; branch/loop; the `ISA_*` instruction encoding; etnaviv compiler translates NIR → Vivante ISA (`etnaviv_compiler.c`)
- Mesa etnaviv Gallium driver (`src/gallium/drivers/etnaviv/`): `etna_screen`, `etna_context`; TGSI/NIR compiler pipeline; `etna_resource` (tiled vs linear, supertiled); `etna_draw_vbo` → command buffer emit; state tracker state objects (blend, rasterizer, depth-stencil)
- GC7000 and Vulkan: the `etnaviv` Vulkan driver effort (`src/etnaviv/vulkan/`); GC7000UL in i.MX8MP; conformance status; comparison to Panfrost Vulkan; https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/?label_name=etnaviv
- i.MX8 deployment: Librem 5 (GC7000Lite, OpenGL ES 3.0 via etnaviv); NXP i.MX8M Plus EVK; Variscite DART-MX8M-PLUS; device tree: `gpu: gpu@38000000 { compatible = "vivante,gc" }`
- Performance and limitations: no geometry shader support on GC2000; limited MSAA; single-sample shading only; GC7000 performance competitive with Mali-G52 on ES3 workloads
- Contributing: `dri-devel@lists.freedesktop.org`; Lucas Stach at Pengutronix as main maintainer; CI via Lava with i.MX6Q board; `ETNA_MESA_DEBUG=msgs,dump,compiler`
- **Integrations**: Ch1 (DRM), Ch5 (Mesa Gallium3D), Ch14 (NIR — etnaviv compiles via NIR), Ch90 (Panfrost/Lima — similar open embedded driver story), Ch99 (Automotive/Embedded — i.MX8 is the reference automotive SoC)

### Chapter 101: Color Science and the ICC Profile Pipeline *(Part VI-B — Display Services)*

- Why color science matters: the journey from `(r,g,b)` values in a shader to photons of a specific spectral power distribution leaving a display; each device encodes color differently
- CIE color spaces: XYZ (device-independent); Lab (perceptually uniform); xy chromaticity diagram; the sRGB primaries and D65 white point; the difference between sRGB and linear sRGB (gamma 2.2 encoding)
- ICC profiles: structure (128-byte header + tag table + tag data); v2 vs v4 profiles; input/display/output profiles; LUT-based vs matrix-based; `A2B0`/`B2A0` tags; `rXYZ`/`gXYZ`/`bXYZ` + `rTRC`/`gTRC`/`bTRC` for matrix profiles; ICC profile inspection with `iccDump`
- LittleCMS (`lcms2`): `cmsOpenProfileFromFile`, `cmsCreateTransform`; the colour transform pipeline (input profile → PCS → output profile); rendering intents (perceptual, relative colorimetric, absolute colorimetric, saturation); thread safety; used by GIMP, Krita, Darktable, Inkscape, libvips
- colord: D-Bus color management daemon; `CdClient`, `CdDevice`, `CdProfile`; device→profile mapping persistence; `cd-sensor` for colorimeter access; automatic ICC profile installation; `colormgr` CLI; https://github.com/hughsie/colord
- ArgyllCMS and DisplayCAL calibration workflow: `dispcal` for display characterisation; `targen` + `dispread` for measurement target; `colprof` for ICC profile generation; colorimeter vs spectrophotometer (i1Display Pro, ColorMunki, X-Rite i1Pro); `DisplayCAL` GUI wrapping ArgyllCMS; output: `vcgt` tag (Video Card Gamma Table) loaded via `xcalib` or `dispwin`; https://www.argyllcms.com/
- vcgt and KMS CTM: two mechanisms for applying color correction at the display level; `vcgt` (gamma table, CLUT loaded into GPU RAMDAC); KMS CTM (3×3 matrix applied by the display controller, Ch74); colord's `cd-sensor` can write vcgt via `xcalib` on X11 or via KMS `CTM` property on Wayland
- Application-level color management: GIMP's color proof pipeline; Krita's `KoColorSpace`; Darktable's `dt_iop_colorout` module; Firefox (Ch52) color managing images via LittleCMS; Chrome's `--force-color-profile`
- Soft-proofing: simulating how a print will look on screen; CMYK→RGB gamut mapping; out-of-gamut warning; rendering intent comparison; used in prepress workflows
- HDR color management (connection to Ch74): the BT.2020 primaries and PQ/HLG transfer functions as "ICC-like" profile data; `VK_EXT_swapchain_colorspace`; the Wayland `color-management-v1` protocol's profile concept as ICC-adjacent
- **Integrations**: Ch2 (KMS CTM property), Ch22 (Mutter/KWin applying ICC via KMS), Ch37 (Skia color management), Ch52 (Firefox LittleCMS), Ch53 (Ch53: Display Calibration and colord — this chapter provides the underlying science), Ch74 (HDR/WCG — spectral extension of ICC concepts)

### Chapter 102: The DRM GPU Scheduler and Multi-Process Fairness *(Part I)*

- Why a GPU scheduler: multiple processes (compositor + apps + hardware video decode) submit GPU work simultaneously; without scheduling, one process can starve others or cause hangs
- `drm_gpu_scheduler` (`drivers/gpu/drm/scheduler/`): `drm_sched_job`, `drm_sched_entity`, `drm_gpu_scheduler`; FIFO queue per entity; entity → scheduler mapping; `drm_sched_job_init`, `drm_sched_entity_push_job`, `drm_sched_main` kthread
- Priority classes: `DRM_SCHED_PRIORITY_MIN`, `NORMAL`, `HIGH`, `KERNEL`; how compositors request `HIGH` priority for their render queue; per-entity preemption (where hardware supports it); priority inheritance for fence dependencies
- The timeline fence model: `drm_syncobj` as GPU-visible fence containers; `drm_syncobj_timeline` for multi-point timelines; how the scheduler waits on input fences and signals output fences; the `drm_sched_fence` → `dma_fence` → `drm_syncobj` chain
- Multi-queue scheduling: render + async compute + DMA (blit) queues per GPU; `drm_sched_entity` per queue; how AMDGPU exposes compute queues (`amdgpu_ctx_add_fence`); Intel Xe's parallel submission model
- Timeout and hang detection: `drm_sched_job` timeout (`DRM_SCHED_STOP_TIMEOUT`); `drm_sched_backend_ops::timedout_job` callback; GPU reset flow; `amdgpu_device_gpu_recover`; how the scheduler fences are signalled after reset to unblock waiting userspace
- Per-process GPU time accounting: `drm_file` statistics; `/proc/pid/fd/` → DRM file; `amdgpu_read_mm_stats` sysfs; GPU time slicing across processes; the compositor's GPU budget vs application GPU budget
- Connection to Wayland: the compositor's `wl_surface.commit` triggers a Wayland render job; that job's GPU fence controls when the KMS atomic commit can fire; `linux-drm-syncobj-v1` exposes the timeline fence to Wayland clients so client rendering and compositor rendering are properly sequenced
- Real-world priority inversion: example of an application holding a fence the compositor waits on, blocking scanout; how explicit sync (Ch75) and the scheduler's priority system together prevent this
- **Integrations**: Ch1 (DRM architecture — scheduler is a core DRM component), Ch3 (DRM syncobj — the fence mechanism the scheduler uses), Ch4 (GPU memory — scheduler manages job lifetimes tied to BO lifetimes), Ch21 (wlroots — compositor queue priority), Ch75 (explicit GPU sync — scheduler timeline fences underpin syncobj)

---

## Part XXII — Additional Chapters

### Chapter 123: Screen Capture and Remote Desktop on Linux *(Part VI-B — Display Services)*

- X11 screen capture: `XGetImage`, `XShmGetImage`, XComposite `NameWindowPixmap`, the security problem (any client can capture any window)
- KMS writeback connectors: `DRM_MODE_CONNECTOR_WRITEBACK`, `drm_writeback_job`; supported by Mali-DP, Arm Komeda, VC4 TXP, AMDGPU, R-Car DU, Qualcomm DPU, VKMS
- wlr-screencopy protocol: `zwlr_screencopy_manager_v1`, `zwlr_screencopy_frame_v1`; wl_shm and DMA-BUF buffer types; grim, slurp, wf-recorder
- ext-image-copy-capture: standardised wayland-protocols successor; session-based API; cursor capture; GNOME/KDE support
- PipeWire screencast: `pw_stream` with `SPA_MEDIA_TYPE_VIDEO`; DMA-BUF zero-copy path; monitor/window/region source types
- xdg-desktop-portal ScreenCast: `org.freedesktop.portal.ScreenCast` D-Bus interface; portal implementations (gnome, kde, wlr)
- OBS Studio on Wayland: pipewire-capture source; DMA-BUF EGL image import; VA-API/NVENC output; V4L2 loopback virtual camera
- WebRTC screen sharing: `getDisplayMedia()` → xdg-portal → PipeWire; Chromium PipeWire-based capturer; Firefox via GtkScreenCast
- Remote desktop: gnome-remote-desktop (FreeRDP, H.264 via VA-API); xrdp; Sunshine/Moonlight (GameStream, NVENC/VAAPI)
- **Integrations**: Ch2 (KMS writeback), Ch20 (Wayland ext-image-copy-capture), Ch21 (wlr-screencopy), Ch38 (PipeWire), Ch50 (Firefox), Ch57 (remote desktop encoding), Ch111 (Flatpak portal), Ch112 (VRR)

### Chapter 116: RISC-V GPU Drivers *(Part II)*

- RISC-V GPU ecosystem: open-source GPU IP, commercial SoCs (StarFive JH7110 / Imagination BXT M-21), RVV extensions relevant to GPU offload
- Imagination PowerVR BXT M-21 on JH7110: `pvr` Mesa Vulkan driver; kernel DRM `drivers/gpu/drm/imagination/`; Mesa 24.x and Linux 6.8+ status
- Open-source GPU IP: LlamaGPU, RVGPU (remote OpenGL/Vulkan via Wayland); virtio-gpu as pragmatic fallback on QEMU/Spike
- Mesa software renderers on RISC-V: llvmpipe with RVV vectorisation; lavapipe; LLVM RISC-V target
- RISC-V display stack: KMS/DRM on JH7110 (`drivers/gpu/drm/verisilicon/`); Wayland on RISC-V (wlroots, sway)
- **Integrations**: Ch1 (DRM driver model), Ch4 (GEM, virtio-gpu), Ch17 (software renderers), Ch89 (GPU virtualisation), Ch122 (DKMS for BSP GPU drivers)

### Chapter 117: Slang — Differentiable and Modular Shading Language *(Part XV)*

- Overview: open-source shading language (NVIDIA Research → https://github.com/shader-slang/slang); generics, interfaces, automatic differentiation, modules
- Automatic differentiation: `[Differentiable]` attribute; `fwd_diff`/`bwd_diff` operators; neural rendering and differentiable rendering use cases
- Language features: generics and interfaces; `import` modules; `extension` blocks; `[Require]` capability constraints
- Compilation targets: SPIR-V (Vulkan), DXIL (D3D12), CUDA PTX, CPU fallback; `slangc` compiler; C API via `SlangCompileRequest`
- Integration: Falcor renderer; Vulkan SPIR-V output to RADV/ANV/NVK; PySlang Python bindings; ROCm/HIP via SPIR-V
- **Integrations**: Ch14 (NIR — Slang SPIR-V → spirv_to_nir), Ch24 (Vulkan), Ch66 (CUDA PTX target), Ch110 (SPIR-V toolchain), Ch115 (NeRFStudio neural rendering workloads)

### Chapter 118: NAK — The Nouveau/NVK Rust Shader Compiler *(Part III)*

- NAK: Rust-language shader compiler backend for Nouveau/NVK replacing C-based nvir/nvc0; lives in Mesa `src/nouveau/compiler/`
- Architecture: NIR → NAK IR → register allocation → SASS binary; `nak_compile_shader()` entry; `nak::ir` crate; `nak::opt` passes
- NAK IR: SSA form, `Instr`/`Op`/`Src`/`Dst` types; `RegFile` (GPR, predicate, uniform); SASS instruction modelling
- Register allocation: `nak_ra` crate; live range analysis; spilling to local memory; predicated execution model
- SASS ISA coverage: Ampere (sm_86), Ada Lovelace (sm_89), Hopper (sm_90); `nak_encode`; inline assembly via `nak_asm`
- Optimisation passes: copy propagation, DCE, constant folding, memory coalescing, barrier optimisation
- **Integrations**: Ch8 (Nouveau kernel driver), Ch10b (NVK Vulkan driver), Ch14 (NIR input IR), Ch20 (Nova Rust driver), Ch110 (SPIR-V → NIR → NAK path)

### Chapter 119: Zink — OpenGL on Vulkan *(Part IV)*

- Purpose and history: Mesa Gallium driver implementing OpenGL via Vulkan; enables GL on Vulkan-only drivers (NVK, v3dv, Turnip, PowerVR)
- Architecture: `zink_screen` (VkDevice), `zink_context`, `zink_batch_state`, `zink_resource`; position alongside radeonsi/iris/etnaviv
- State translation: FBOs → VkRenderPass, textures → VkImage, UBOs → VkBuffer, transform feedback via `VK_EXT_transform_feedback`
- Shader translation: GLSL → NIR → `nir_to_spirv()`; Zink NIR lowering passes; shader variant keys; `VK_EXT_shader_object` path
- Descriptor management: lazy / db (`VK_EXT_descriptor_buffer`) / auto modes; `zink_descriptors_update()` hot path
- Pipeline caching: dynamic state extensions (EDS1/2/3); GPL; async compilation; on-disk `VkPipelineCache`
- Compatibility profile: `GL_QUADS` emulation GS, display lists, accumulation buffer, fixed-function lighting; GL 4.6 core CTS conformance
- Kopper WSI: platform-agnostic display for Zink; NVK default (Mesa 25.1), v3dv (RPi5), Turnip (Adreno), PowerVR (Mesa 26.1)
- Debugging: `ZINK_DEBUG` flags, `ZINK_DESCRIPTORS` mode, render pass fragmentation analysis
- **Integrations**: Ch13 (Gallium3D), Ch14 (NIR), Ch16 (Mesa Vulkan common), Ch18 (RADV), Ch20 (NVK), Ch24 (Vulkan), Ch110 (SPIR-V), Ch120 (GPU memory)

### Chapter 120: GPU Memory Management Internals — TTM, GEM, and BAR *(Part I)*

- GPU memory topology: VRAM (GDDR6X/HBM3), GTT (system RAM via IOMMU/GART), PCIe BAR, GPU MMU
- GEM internals: `drm_gem_shmem_object`, `drm_gem_dma_object`; driver-specific embedding (`amdgpu_bo`, `nouveau_bo`, `i915_gem_object`)
- TTM in depth: `ttm_device`, `ttm_resource_manager`, `ttm_buffer_object`, `ttm_resource`; memory types; `ttm_bo_validate`; ww_mutex reservation
- drm_buddy: buddy allocator for VRAM; `gpu_buddy_alloc_blocks`; AMDGPU `amdgpu_vram_mgr`
- drm_gpuvm + drm_exec: `drm_gpuvm_bo`, `drm_gpuva`; VM_BIND state machine; `drm_exec_until_all_locked`; Xe usage
- Resizable BAR / SAM: `pci_resize_resource()`; `amdgpu_device_resize_fb_bar()`; write-combining; 5–15% gain in bandwidth-limited workloads
- VRAM eviction: `ttm_lru_walk_for_evict`, `ttm_bo_swapout`; fence-wait gate; GPU TLB invalidation; OOM cascade
- DMA-BUF internals: `dma_buf_ops` vtable; `dma_resv`; VA-API → Vulkan → KMS scanout cross-driver example
- Debugging: fdinfo `drm-total-*/drm-shared-*/drm-active-*`; per-driver debugfs; `drm_info`
- **Integrations**: Ch1 (DRM), Ch2 (KMS), Ch4 (GEM/DMA-BUF overview — differentiated scope), Ch5 (amdgpu/i915/Xe), Ch8 (Nouveau), Ch89 (GPU virtualisation), Ch102 (DRM GPU scheduler)

### Chapter 121: DRM Lease and VR Direct Display *(Part I)*

- VR latency problem: motion-to-photon <20 ms; direct display eliminates compositor overhead; ATW (Asynchronous Time Warp)
- DRM lease API (kernel 4.15): `DRM_IOCTL_MODE_CREATE_LEASE`, `DRM_IOCTL_MODE_LIST_LESSEES`, `DRM_IOCTL_MODE_GET_LEASE`, `DRM_IOCTL_MODE_REVOKE_LEASE`
- HMD connector discovery: EDID vendor ID (VLV, HVR, OVR); `non-desktop` quirk in `drm_edid.c`; `VK_EXT_acquire_drm_display`; `vkGetDrmDisplayEXT`
- Monado OpenXR: `comp_window_direct_drm.c`; DRM lease + `VK_EXT_acquire_drm_display`; `u_timing_frame` predictor
- wp_drm_lease_device_v1: compositor-mediated lease; `create_lease_request`, `request_connector`, `submit`, `lease_fd`; wlroots/KWin/Mutter support
- Direct-to-display Vulkan: `VK_KHR_display`, `VK_EXT_acquire_drm_display`; `vkCreateDisplayPlaneSurfaceKHR`; wsi_drm.c
- VR timing: `DRM_EVENT_FLIP_COMPLETE`; `drmWaitVBlank`; `VK_EXT_present_timing` (Vulkan 1.4.335, November 2025)
- Practical setup: Valve Index, Meta Quest (ALVR), HTC Vive; libsurvive; Monado-gui
- **Integrations**: Ch2 (KMS atomic), Ch20 (Wayland wp_drm_lease_device_v1), Ch21 (wlroots), Ch24 (Vulkan), Ch25 (OpenXR), Ch75 (explicit sync), Ch112 (VRR)

### Chapter 122: DKMS and Out-of-Tree GPU Kernel Modules *(Part IX)*

- Kernel module ABI problem: no stable ABI; vermagic; `CONFIG_MODVERSIONS` CRC; `EXPORT_SYMBOL_GPL` restrictions
- DKMS mechanics: `dkms.conf` directives; `dkms add/build/install/status/autoinstall`; `/var/lib/dkms/`; systemd hook
- NVIDIA proprietary modules: `nvidia.ko`, `nvidia-drm.ko`, `nvidia-modeset.ko`, `nvidia-uvm.ko`; binary stub + glue; `MODULE_LICENSE("NVIDIA")`
- NVIDIA open modules (2022, MIT/GPL): Turing+ requirement; GSP-RM firmware; Blackwell open-only; Fedora default
- AMD upstream-first: all of `drivers/gpu/drm/amd/` in mainline; no DKMS needed; zero version matrix overhead
- Intel firmwares: GuC/HuC/DMC in `linux-firmware`; version matching; initrd inclusion
- Out-of-tree lifecycle: development → RFC → staging → mainline; `MODULE_DEVICE_TABLE`; out-of-tree Makefile pattern
- Secure Boot + MOK: `sign-file`; `mokutil --import`; DKMS auto-signing with enrolled key
- Distribution packaging: Ubuntu (`nvidia-dkms-*`), Fedora (RPMFusion `akmod-nvidia`), Arch, NixOS (`hardware.nvidia.open`), Gentoo
- **Integrations**: Ch1 (DRM), Ch5 (amdgpu upstream), Ch8 (Nouveau), Ch10a (Nova Rust driver), Ch76 (contributing), Ch102 (DRM GPU scheduler)

### Chapter 134: OpenCL on Linux *(Part XIV)*

- ICD loader (`libOpenCL.so`); `ocl-icd`; `/etc/OpenCL/vendors/*.icd`; `clinfo`; `cl_platform_id` / `cl_device_id` / `cl_context` / `cl_command_queue` / `cl_program` / `cl_kernel`
- Mesa rusticl: OpenCL 3.0 via Gallium; `src/gallium/frontends/rusticl/`; OpenCL C → Clang/LLVM → NIR → Gallium backend; `RUSTICL_ENABLE`; conformance on Intel iris
- Intel NEO / Compute Runtime: `intel-compute-runtime`; IGC (Intel Graphics Compiler); `cl_intel_subgroups`; `cl_intel_unified_shared_memory` (USM); Gen9–Xe2 support
- AMD ROCm OpenCL: `ROCm/clr`; OpenCL 2.0; AMDGPU target via clang; `cl_amd_device_attribute_query`; vs HIP for portability
- pocl (Portable Computing Language): LLVM CPU backend; x86_64/AArch64/RISC-V; `pocl-cuda`; OpenCL 3.0 conformance
- OpenCL programming model: `cl_mem`; global/local/constant/private memory; `clEnqueueNDRangeKernel`; SVM; USM; `clCreateProgramWithIL` for SPIR-V
- OpenCL–OpenGL/Vulkan interop: `cl_khr_gl_sharing`; `cl_khr_external_memory` / `cl_khr_external_memory_dma_buf`; zero-copy VA-API → OpenCL pipeline
- Real applications: Darktable rawspeed; hashcat; BOINC; vkFFT; ffmpeg OpenCL filters; Blender Cycles (historical: removed in 3.0)
- Debugging: `clGetEventProfilingInfo`; `oclgrind`; `clpeak`; `RUSTICL_DEBUG=1`; `AMD_OCL_WAIT_COMMAND=1`
- **Integrations**: Ch14 (NIR — rusticl uses NIR), Ch15 (ACO — rusticl on AMD), Ch17 (llvmpipe as pocl platform), Ch25 (GPU Compute), Ch108 (ROCm/HIP), Ch110 (SPIR-V ingestion)

### Chapter 125: RenderDoc on Linux *(Part IX)*

- Architecture: `librenderdoc.so` injected into target process; `qrenderdoc` Qt GUI; `.rdc` capture file format; in-process vs out-of-process replay
- Installation: build from source (CMake + Qt6); `VK_LAYER_RENDERDOC_Capture` implicit layer JSON in `implicit_layer.d/`; `ENABLE_RENDERDOC=1`
- Vulkan capture: dispatch table replacement; F12 trigger; resource tracking (`vkMapMemory`); descriptor tracking per draw; multi-frame capture
- OpenGL capture: PLT hook + EGL export interception; `glXGetProcAddress` / `eglGetProcAddress` replacement; `GL_KHR_debug` markers
- UI: EventBrowser; Texture Viewer (mip/array/HDR); Mesh Output; Pipeline State inspector; Resource Inspector; Performance Counters via `VK_EXT_performance_query`
- Shader debugging: pixel-level selection; SPIR-V disassembly; `OpLine` source-level; vertex/compute thread debugging; software SPIR-V emulation
- Programmatic capture API: `RENDERDOC_API_1_6_0`; `dlopen("librenderdoc.so")`; `StartFrameCapture` / `EndFrameCapture`; `SetCaptureFilePathTemplate`; CI use
- Mesa driver development: `renderdoccmd replay` against dev Mesa build; shader hot-patching; `gfxreconstruct` alternative; `VK_LAYER_LUNARG_api_dump`
- **Integrations**: Ch18 (RADV), Ch19 (ANV), Ch24 (Vulkan), Ch30 (Debugging), Ch75 (Explicit sync), Ch104 (DXVK), Ch109 (Mesa CI), Ch110 (SPIR-V)

### Chapter 126: Hybrid Graphics and Laptop Power Management *(Part II)*

- Hardware model: iGPU (display-connected) + dGPU (PCIe endpoint); MUX switch vs MUX-less designs; ACPI `_PS0`/`_PS3` power methods; `lspci -v`
- `vga_switcheroo`: `drivers/gpu/vga/vga_switcheroo.c`; `vga_switcheroo_register_client`; `/sys/kernel/debug/vga_switcheroo/switch`; IGD/DIS immediate vs DIGD/DDIS delayed
- `bbswitch`: ACPI `_DSM` power toggle; `/proc/acpi/bbswitch`; Bumblebee project (`optirun`/`primusrun`); superseded by PRIME but still useful fallback
- PRIME render offload: `DRI_PRIME=1`; `DRI_PRIME=pci-0000:01:00.0`; NVIDIA `__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia`; `MESA_VK_DEVICE_SELECT`; `xrandr --setprovideroffloadsink`
- NVIDIA Optimus: `nvidia-prime`; `prime-select`; `NVreg_DynamicPowerManagement=0x02` for D3cold; `/sys/bus/pci/*/power/control=auto`; on-demand mode
- envycontrol: `integrated`/`hybrid`/`nvidia` modes; udev rules for power gating; `--reset`; GDM/SDDM/LightDM compatibility
- TLP and power-profiles-daemon: `RUNTIME_PM_ON_BAT=auto`; `powerprofilesctl`; `performance` disables PM; battery threshold control
- AMD SmartShift: APU + discrete RDNA; `amdgpu.runpm=1`; `amd_pstate` interaction; `rocm-smi --showpower`
- Diagnostics: `powertop --auto-tune`; `turbostat`; `/sys/bus/pci/*/power/runtime_status`; D3cold verification; `upower -i`
- **Integrations**: Ch5 (amdgpu), Ch49 (PRIME), Ch51 (GPU Power Management), Ch106 (Multi-GPU PRIME), Ch122 (DKMS — NVIDIA module install)

### Chapter 127: Mesh Shaders and Variable Rate Shading *(Part VII-A — GPU APIs)*

- Traditional pipeline limits: IA bottleneck; GS limitations; indirect draw (`vkCmdDrawIndexedIndirect`) limits; why Nanite needed mesh shaders
- `VK_EXT_mesh_shader`: Task shader (`EmitMeshTasksEXT`) → Mesh shader (`SetMeshOutputsEXT`) → Rasterizer; `perprimitiveEXT`; `taskPayloadSharedEXT`; 256v/256p per workgroup
- Mesh shader GLSL: `#extension GL_EXT_mesh_shader`; meshlets via `meshoptimizer` (`meshopt_buildMeshlets`); frustum/backface culling in task shader; `VkPhysicalDeviceMeshShaderPropertiesEXT`
- Hardware support: NVIDIA Ampere+; RADV RDNA2+ (Mesa 23.1, RDNA3 attribute ring); ANV DG2/Xe-HPG; NVK mid-2026; `RADV_DEBUG=nomeshshader`
- `VK_KHR_fragment_shading_rate`: per-pipeline / per-primitive / per-tile VRS; `(width,height)` invocation rate; combiner ops; `VkFragmentShadingRateAttachmentInfoKHR`; `R8_UINT` encoding `(log2(w)<<2)|log2(h)`
- VRS API: `vkCmdSetFragmentShadingRateKHR`; `gl_PrimitiveShadingRateEXT`; tile-based VRS image; compute shader VRS generation; foveated rendering
- VRS hardware: RADV RDNA2+ per-pipeline/primitive; RDNA3 attachment; ANV Gen12+; foveated VR use case; `vulkaninfo | grep -i shading_rate`
- Combined mesh + VRS: per-primitive rate via `perprimitiveEXT gl_PrimitiveShadingRateEXT`; cluster LOD + VRS rate selection; Nanite-style Vulkan
- **Integrations**: Ch18 (RADV), Ch19 (ANV), Ch24 (Vulkan), Ch77 (Shader compilation), Ch97 (UE5/Nanite), Ch110 (SPIR-V opcodes), Ch112 (VRR + VRS for VR)

### Chapter 128: DisplayPort MST and Multi-Monitor Topology *(Part VI-B — Display Services)*

- DP physical/link layer: HBR/HBR2/HBR3/UHBR10/UHBR20; AUX channel; DPCD; link training; DSC; VESA DP 2.1
- SST vs MST: VC payload slots (64 per link); time slot allocation; sideband messages (`LINK_ADDRESS`, `ALLOCATE_PAYLOAD`, `ENUM_PATH_RESOURCES`); branch/sink topology
- `drm_dp_mst_topology`: `drm_dp_mst_topology_mgr`; `drm_dp_mst_branch`; `drm_dp_mst_port`; ACT handshake; `drm_dp_add_payload_part1/part2`; `drm_dp_mst_atomic_check`; RAD/LCT/LCR addressing
- USB-C DP Alt Mode: lane assignment; `typec_altmode_driver`; Thunderbolt 3/4 dock; `boltctl`; USB4; DP 2.1 over USB4; `DRM_MODE_CONNECTOR_USB`
- Docking station support: Synaptics VMM / Parade PS176/PS186; hotplug via `drm_kms_helper_hotplug_event`; `card0-DP-1-1` / `card0-DP-1-2` connector naming
- Bandwidth and DSC: `ENUM_PATH_RESOURCES`; `drm_dp_mst_find_vcpi_slots`; `drm_dp_dsc_sink_caps_init`; `drm_dp_compute_dsc_config`; `-ENOSPC` on overflow
- Kernel debugging: `drm_dp_mst_dump_topology`; `drm.dp_mst=1`; `amdgpu_mst_topology` debugfs; `i915_dp_mst_info`; AUX timeout diagnosis
- Compositor: KMS connector enumeration; EDID via `REMOTE_I2C_READ`; Mutter/KWin multi-monitor profiles; `wlr_output` per connector; `xrandr --listproviders`
- **Integrations**: Ch2 (KMS), Ch3 (Advanced Display), Ch74 (HDR over MST), Ch75 (Explicit sync — vblank), Ch101 (ICC profiles per display), Ch4 (DMA-BUF scanout)

### Chapter 129: GPU Firmware Deep Dive *(Part I)*

- `request_firmware` API: 6 kernel variants; search path `/lib/firmware/`; initramfs inclusion; `CONFIG_EXTRA_FIRMWARE`; suspend/resume caching; firmware BO allocation
- AMD firmware: GFX ME/MEC/RLC/PFP; VCN; DMCUB; SMU; PSP SOS/ASD (RSA signature chain); MES; `amdgpu_ucode_init_single_fw`; PSP authentication ring; `psp_cmd_submit_buf`
- Intel GuC/HuC/DMC/GSC: `intel_guc_fw_upload`; GuC submission (Gen12+); HuC→GuC auth chain; DMC DC5/DC6 power gating; Xe GSC CPD format; `i915.enable_guc=3`
- NVIDIA Falcon/GSP-RM: Falcon HS/LS/NS security hierarchy; SEC2 → Booter → GSP-RM trust chain; RISC-V GSP-RM architecture; 256 KB RPC queues; Nouveau `extract-firmware-nouveau.py`; FUC/envytools history
- linux-firmware: date-based versions; `WHENCE` license file; distribution packages; fwupd/LVFS update path; `modinfo amdgpu | grep firmware`; initramfs rebuild
- Security: AMD PSP RSA validation; Intel PAVP; NVIDIA GSP signature; GPU confidential computing (SEV-ES, MIG); CVE-2023-20579; `fwupdmgr security` attestation
- Debugging: `dmesg | grep firmware`; `amdgpu_firmware_info` debugfs; `i915_guc_info`; `i915.enable_guc=0` fallback; `amdgpu.fw_load_type=2`; version mismatch handling
- **Integrations**: Ch1 (DRM probe), Ch2 (DMC display gating), Ch5 (AMDGPU firmware init), Ch6 (i915/Xe GuC), Ch8 (Nouveau Falcon), Ch10 (Nova/GSP), Ch26 (VCN video decode), Ch51 (power states), Ch122 (DKMS firmware dependency)

### Chapter 130: Wayland Protocol Extension Development *(Part VI-A — Wayland Compositor)*

- Wire protocol: 32-bit object IDs; message layout `[object_id][size_opcode][args]`; type system (uint/int/fixed/string/object/new_id/array/fd); SCM_RIGHTS FD passing; version negotiation via `wl_registry.bind`
- XML grammar: `<protocol>`/`<interface version="N">`/`<request type="destructor">`/`<event>`/`<arg type="...">`; `since`; `allow-null`; `enum`; `description`; `ext_notification_manager_v1` worked example
- wayland-scanner: `client-header` / `server-header` / `private-code`; generated vtable structs and send functions; Meson `wayland.scan_xml()` and manual `custom_target`
- Server implementation: `wl_global_create`; bind handler; `wl_resource_create`; `wl_resource_set_implementation`; version-gated events; `eventfd` + `wl_event_loop_add_fd` thread safety
- Client implementation: `wl_registry_listener`; `wl_registry_bind` version negotiation; `prepare_read` / `read_events` / `dispatch_pending` multi-threaded pattern
- Governance: `wayland-protocols` staged (`staging/`) → stable (`stable/`); `wlr-protocols` (`zwlr_*`); namespace: `ext_` vs `zwlr_` vs `wp_`; 2-ACK staging / 3-ACK stable; triangle of implementations
- Design patterns: factory; capability advertisement; double-buffer commit; version-gated features; anti-patterns: missing destructor, string vs FD, synchronous ping-pong, version mutation
- Testing: `WAYLAND_DEBUG=1`; `wayland-debug`; `wayland-info`; `wlcs` conformance; Weston headless; version negotiation tests
- **Integrations**: Ch20 (core Wayland protocol), Ch21 (wlroots zwlr_*), Ch22 (compositors), Ch46 (protocol governance), Ch75 (linux-drm-syncobj-v1), Ch121 (wp_drm_lease_device_v1), Ch132 (security in protocol design)

### Chapter 131: Touch, Stylus, and Tablet Input on Wayland *(Part VI-B — Display Services)*

- Kernel HID drivers: `wacom.ko` (`wacom_wac.c` + `wacom_sys.c`); `hid-uclogic.ko` (Huion/XP-Pen); `EV_ABS`; `ABS_MT_*` multi-touch type B; `BTN_TOOL_PEN`; `ABS_PRESSURE`/`ABS_TILT_X`/`ABS_TILT_Y`; `evtest`; `ID_INPUT_TABLET`
- libinput tablet: `LIBINPUT_EVENT_TABLET_TOOL_*`; `get_pressure()` / `get_tilt_x()` / `get_tilt_y()`; `LIBINPUT_TABLET_TOOL_TYPE_*`; pad events (button/ring/strip); touch: `TOUCH_DOWN/UP/MOTION/FRAME/CANCEL`
- libwacom: device database; `libwacom_new_from_path`; `libwacom_get_num_buttons`; `libwacom_get_integration_flags`; stylus axes; GNOME Wacom panel integration
- `wl_touch`: `wl_seat_get_touch`; events: `down/up/motion/frame/cancel/shape/orientation`; surface-local coordinates; `frame()` batching; `shape` (v6) major/minor axes
- `zwp_tablet_manager_v2`: `zwp_tablet_seat_v2`; `zwp_tablet_v2`; `zwp_tablet_tool_v2` (`proximity_in/out`, `tip_down/up`, `motion`, `pressure`, `tilt`, `rotation`, `slider`, `wheel`, `button`, `frame`); `zwp_tablet_pad_v2`
- wlroots: `wlr_tablet_manager_v2_create`; `wlr_tablet_tool_notify_axis`; pad mode groups; `wlr_virtual_keyboard_manager` for tablet mode; protocol path `unstable/tablet/tablet-unstable-v2.xml`
- GTK/Qt integration: GTK3 `GdkDeviceManager` (GIMP 3.0) vs GTK4 `GdkSeat`; `QTabletEvent::pressure()` / `xTilt()` / `yTilt()`; Krita `KisTabletEvent`; `value120` wheel convention
- Calibration/mapping: tablet area → screen area; multi-monitor assignment; `gnome-control-center wacom`; `input-remapper`; `iio-sensor-proxy` auto-rotate; HiDPI `GDK_SCALE`
- **Integrations**: Ch54 (Linux input stack), Ch20 (wl_touch in wl_seat), Ch21 (wlroots zwp_tablet), Ch22 (compositors), Ch39 (Qt/GTK), Ch130 (protocol development — tablet as case study)

### Chapter 132: Wayland Security *(Part VI-A — Wayland Compositor)*

- X11 security problems: `XGetImage`; `XSendEvent`; `XGrabKeyboard`; `XQueryKeymap` keylogger; `xauth` weaknesses; CVE-2024/2025 Xwayland vulns
- Wayland isolation: per-client `wl_resource` scoping; `wl_keyboard` focus routing; `SO_PEERCRED` verification; `$XDG_RUNTIME_DIR` mode 0700; no shared framebuffer
- Attack surfaces: `zwlr_screencopy_manager_v1` no-ACL; XWayland as X11 security island; `zwp_virtual_keyboard_manager_v1` injection; DMA-BUF FD leakage; clipboard manager access
- xdg-desktop-portal: `org.freedesktop.portal.ScreenCast`; `persist_mode` (0/1/2); `restore_token`; permission store (`~/.local/share/xdg-permission-store/`); AppID via `SO_PEERCRED`→cgroup; TOCTOU mitigation
- Flatpak sandboxing: bubblewrap namespaces; `--socket=wayland`; `wp_security_context_v1` (staging, wayland-protocols 1.48); `--device=dri` vs `--device=all`; zypak
- Virtual keyboard security: `zwp_virtual_keyboard_manager_v1` threat model; compositor authorization requirement; RemoteDesktop portal as safe injection path
- DRM render node ACL: card vs render node split; `udev uaccess` + logind dynamic ACLs; container `--device=/dev/dri/renderD128`; `O_CLOEXEC` hygiene
- Clipboard: `wl_data_device` focus requirement; `ext-data-control-v1` privileged access; DnD routing; clipboard manager trust model
- Hardening: screencopy restriction code; `--unshare=ipc`; portal D-Bus over direct `zwlr_`; `wp_security_context_v1` adoption roadmap
- **Integrations**: Ch20 (Wayland protocol isolation), Ch21 (wlroots protocol exposure), Ch22 (Mutter/KWin ACLs), Ch38 (PipeWire screencast), Ch111 (Flatpak), Ch123 (Screen Capture), Ch130 (protocol security by design)

### Chapter 133: Vulkan Compute Queues and Task Graphs *(Part VII-A — GPU APIs)*

- GPU hardware queues: AMD RDNA3/4 (GFX/ACE/SDMA); Intel Xe2 (render/compute/copy/video); NVIDIA Ampere (unified + copy engines); hardware preemption at kernel level
- Queue families: `vkGetPhysicalDeviceQueueFamilyProperties`; `VkQueueFamilyProperties` flags; RADV (3 families)/ANV (1+video)/NVK layouts; `VkDeviceQueueCreateInfo`; priority floats
- Timeline semaphores (Vulkan 1.2): `VK_SEMAPHORE_TYPE_TIMELINE`; `VkTimelineSemaphoreSubmitInfo`; `vkWaitSemaphores`; `vkSignalSemaphore`; `vkGetSemaphoreCounterValue`; WSI binary semaphore exception
- Pipeline barriers: `vkCmdPipelineBarrier2` (Vulkan 1.3); `VkDependencyInfo`; stage/access mask table; image layout transitions; queue family ownership transfer; `CONCURRENT` vs `EXCLUSIVE`
- Async compute: GFX + ACE overlap; Z-prepass + particle simulation example; `VkSubmitInfo2` with timeline waits; candidate workloads (SSAO/TAA/culling/physics); common pitfalls
- DMA queue streaming: staging buffer pattern; timeline-semaphore texture streaming; PCIe vs VRAM bandwidth asymmetry; `VK_EXT_external_memory_host`; sparse texture upload
- Frame graphs / task graphs: Frostbite DAG model; compile phases (topo sort, culling, aliasing, queue assignment, barrier insertion); Granite; UE5 RDG; azhirnov/FrameGraph; `VK_EXT_attachment_feedback_loop_layout`
- Practical multi-queue frame: complete timeline table (Z-prepass/particles/lighting/culling/shading/TAA/present); `vkQueueSubmit2` per stage; GPU timestamp profiling; binary semaphore bridge for WSI
- Vulkan video queues: `VK_QUEUE_VIDEO_DECODE_BIT_KHR`; `VkVideoSessionKHR`; `vkCmdDecodeVideoKHR`; H.264/H.265/AV1 profiles; RADV AV1 encode (mid-2025); video→graphics sync
- **Integrations**: Ch24 (Vulkan API), Ch25 (GPU Compute), Ch26 (Hardware Video — video queues), Ch50 (timeline semaphores detail), Ch75 (explicit GPU sync), Ch84 (bgfx frame graph), Ch97 (UE5 RDG), Ch102 (DRM GPU scheduler), Ch127 (mesh shaders on compute queues)

---

## Part XXIV — Critical Gap Chapters

### Chapter 142: V4L2 and the Linux Media Subsystem *(Part XIII)*

- V4L2 architecture: device nodes (/dev/videoN, /dev/mediaM, /dev/subdevX); media controller framework; entity/pad/link graph; `MEDIA_IOC_DEVICE_INFO`, `MEDIA_IOC_ENUM_ENTITIES`
- Device types: capture, output, mem-to-mem (M2M transcoding), overlay; `VIDIOC_QUERYCAP`; `V4L2_CAP_VIDEO_CAPTURE`, `V4L2_CAP_STREAMING`
- Pixel format taxonomy: FourCC table (NV12, YUV420M, MJPEG, H264, VP9, AV1); `VIDIOC_ENUM_FMT`, `VIDIOC_S_FMT`, `v4l2_pix_format_mplane`
- Memory models: MMAP, USERPTR, DMABUF; buffer lifecycle state machine; `select()`/`poll()` streaming loop
- V4L2 subdev API: `MEDIA_IOC_SETUP_LINK`; pad-level format negotiation; sensor driver (imx219); V4L2 controls
- libcamera: pipeline handlers; IPA modules; `libcamera::Stream`, `libcamera::Request`; V4L2Compat; `cam` tool
- DMA-BUF bridge to GPU: EGL, Vulkan, OpenCL import paths; zero-copy camera→inference→compositor
- Stateful vs stateless codecs: request API (`media_request_fd`); `MEDIA_REQUEST_IOC_QUEUE`; H.264 stateful decode
- GStreamer V4L2: `v4l2src`, `v4l2video0convert`, `v4l2h264enc`; DMA-BUF zero-copy pipeline
- Raspberry Pi CSI: RP1 ISP, unicam, pisp; cam/libcamera-still; DMA-BUF sensor→ISP→display
- **Integrations**: Ch26 (VA-API DMABUF interop), Ch37 (PipeWire pw-v4l2 compat), Ch38 (ffmpeg V4L2 M2M), Ch99 (automotive camera)

### Chapter 143: RADV Internals: The Mesa AMD Vulkan Driver *(Part XVII)*

- RADV overview: open-source Vulkan 1.4 in Mesa `src/amd/vulkan/`; history from 2016; conformance; gaming workloads
- AMD hardware: GCN→RDNA1/2/3/4; Shader Engines, CUs, SIMDs, LDS; VGPR/SGPR; wave32/wave64; NGG
- radv_physical_device, radv_device: WSI; feature caps; `amdgpu_gpu_info`; memory management (VRAM/GTT, suballoc, DMA-BUF)
- Command buffer: `radv_cmd_buffer`; PM4 emission; dynamic rendering migration; pipeline barriers
- Shader pipeline: SPIR-V→NIR→RADV NIR passes→ACO (Ch15); ACO vs LLVM; RADV-specific lowering
- Descriptors: inline UBOs; push descriptors; buffer device address; `VK_EXT_descriptor_buffer`
- Ray tracing: `radv_bvh.c`; CPU/GPU hybrid BVH build (RDNA2+); `VK_KHR_ray_tracing_pipeline`
- Mesh shaders (RDNA3), VRS, conditional rendering, transform feedback, pipeline statistics
- Pipeline cache: `~/.cache/mesa_shader_cache_db/`; Steam Shader Pre-Caching
- Debugging: `RADV_DEBUG`, `RADV_PERFTEST`; `umr`; RGP/SQTT capture; GPU hang analysis
- **Integrations**: Ch12 (amdgpu kernel), Ch15 (ACO), Ch18 (Vulkan driver interface), Ch48 (ROCm), Ch104 (DXVK runs on RADV), Ch135 (ray tracing)

### Chapter 144: Boot Graphics Pipeline: From Firmware to KMS Handoff *(Part I)*

- Boot timeline: UEFI GOP → efifb/simplefb → simpledrm → Plymouth → KMS native driver probe
- UEFI GOP: GOP framebuffer; EFI memory map; GOP mode passed via EFI stub; efifb `/proc/fb`
- simplefb/simpledrm: `compatible = "simple-framebuffer"`; `CONFIG_DRM_SIMPLEDRM`; handoff mechanism
- DRM driver probe race: `pci_register_driver` ordering; deferred probe; native driver claims PCI; simpledrm unbind
- KMS at first bind: `drm_atomic_helper_resume`; initial mode; EDID timing; display subsystem restore
- Plymouth: renderer plugins (DRM, GLES, FB); `drmModeSetCrtc`; VT switch to compositor (`DRM_IOCTL_SET_MASTER`)
- Boot→compositor handoff: `DRM_IOCTL_SET_MASTER`; atomic plane restore; LVDS/eDP power sequencing
- Secure Boot: SBAT; shim+GRUB; nvidia-open signing; efistub
- Embedded: U-Boot video; panel-simple; pwm-backlight; Raspberry Pi config.txt
- Debugging: `drm.debug=0x4`; `plymouth --debug`; `/sys/kernel/debug/dri/0/state`
- **Integrations**: Ch1 (DRM driver model), Ch2 (KMS), Ch5 (x86 GPU drivers), Ch92 (Raspberry Pi VC4/simpledrm)

### Chapter 145: XWayland: Architecture and the X11-to-Wayland Bridge *(Part VI-A — Wayland Compositor)*

- XWayland role: X server as Wayland client; rootless (default) vs full-screen; compositor-launched
- Wayland client side: `wl_surface`/`xdg_toplevel` per X11 window; `xwayland-shell-v1`
- DRI3 buffer sharing: DMA-BUF between X11 clients and compositor; PRESENT for vsync
- Input: Wayland→X11 event translation; XKB sync; pointer grab conflicts
- Clipboard/selections: X11 PRIMARY/CLIPBOARD ↔ `wl_data_device`; XDND ↔ `wl_data_offer`
- GLX on XWayland: DRI3 path; GBM buffer allocation; `glXSwapBuffers` → DMA-BUF → compositor
- Explicit sync: `linux-drm-syncobj-v1`; XWayland 23.1+; eliminating tearing
- Rootless internals: WM_CLASS, `_NET_WM_PID`; `_XWAYLAND_MAY_GRAB_KEYBOARD`; decoration/tiling
- Common issues: HiDPI (`_XWAYLAND_GLOBAL_OUTPUT_SCALE`); anti-cheat (EAC/BattleEye); cursor scaling
- Debugging: `WAYLAND_DEBUG=1`; `LIBGL_DEBUG=verbose`; `xwininfo`/`xprop`
- **Integrations**: Ch4 (compositor DMA-BUF import), Ch20 (linux-dmabuf), Ch22 (Wayland input), Ch95 (X11/Xorg legacy)

### Chapter 146: WebCodecs and Browser Hardware Acceleration *(Part X)*

- WebCodecs: W3C API (Chrome 94+); `VideoDecoder`/`VideoEncoder`/`AudioDecoder`/`AudioEncoder`; low-latency vs MSE/WebRTC
- `VideoDecoder`: `isConfigSupported`; `EncodedVideoChunk`; `VideoFrame`; flush/reset; key vs delta
- `VideoEncoder`: bitrate, framerate, `latencyMode`; forced keyframes; bitrate modes
- Linux hardware path: Chrome GPU process; `VaapiVideoDecoderPipeline`; `V4L2VideoDecodeAccelerator`; WebCodecs→Blink→GPU→VA-API→libva→kernel
- `VideoFrame` and DMA-BUF: `importExternalTexture()`; SharedImage backed by VASurface; zero-copy
- WebCodecs + WebGL/WebGPU: `texImage2D(video)`; `device.importExternalTexture(videoFrame)`; `GPUExternalTexture`
- Codec support matrix: H.264/VP8/VP9/AV1/HEVC on Intel/AMD/NVIDIA; AV1 encode (iHD, RDNA3)
- `MediaCapabilities`: `decodingInfo()`/`encodingInfo()`; `smooth + powerEfficient`; VA-API probe
- WebRTC encoded transform: `RTCEncodedVideoFrame`; custom codec insertion
- Debugging: `chrome://media-internals`; `LIBVA_MESSAGING_LEVEL=1`; `chrome://gpu`
- **Integrations**: Ch26 (VA-API), Ch33 (Chrome GPU process), Ch35 (WebGPU importExternalTexture), Ch147 (same VA-API backend)

### Chapter 147: Chrome and Firefox Hardware Video Decode via VA-API *(Part X)*

- Why hardware decode matters: CPU/battery/thermal; 2020–2024 VA-API timeline in Chrome and Firefox
- Chrome VA-API: `--enable-features=VaapiVideoDecoder`; OOP-VD (Chrome 113+); `VaapiWrapper`; `VADisplay` from DRM render node
- Chrome zero-copy: `VASurface`→DMA-BUF→GBM→`EGLImage`→GL texture; NV12 via `EGL_EXT_image_dma_buf_import`; SharedImage
- OOP-VD sandbox: dedicated video decode process; `/dev/dri/renderD128` policy; mojo IPC
- Wayland: `--ozone-platform=wayland`; `zwp_linux_dmabuf_v1` NV12 import into compositor
- Firefox: enabled FF97+; GMP; FFmpegDataDecoder; `media.hardware-video-decoding.enabled`; `SurfaceDescriptorDMABuf`; WebRender DMA-BUF TextureHost
- AV1/VP9: Intel `VAProfileAV1Profile0` (Gen12+); AMD RADV; HEVC patent issues; `chrome://gpu` per-codec
- NVIDIA: NVDEC + proprietary; no open VA-API; Vulkan Video path; fallback
- Practical: `LIBVA_DRIVER_NAME`; Flatpak `--device=dri`; `vainfo`; `LIBVA_TRACE`
- **Integrations**: Ch26 (VA-API), Ch33 (Chrome GPU process), Ch36 (WebRTC), Ch37 (PipeWire DMA-BUF), Ch146 (WebCodecs same backend)

### Chapter 148: Vulkan Synchronisation: A Complete Developer Reference *(Part VII-A — GPU APIs)*

- Why hard: no implicit hazard tracking; explicit CPU-GPU and GPU-GPU ordering required; wrong sync = UB
- Three primitives: Fence (CPU waits GPU), Semaphore (GPU→GPU across queues), Event (within queue / CPU→GPU)
- Fences: `VkFence`; `VK_FENCE_CREATE_SIGNALED_BIT`; `vkQueueSubmit2`; `vkWaitForFences`; N-buffering
- Binary semaphores: acquire→render→present; swapchain semaphore pattern; ordering guarantees
- Timeline semaphores (`VkSemaphoreTypeTimeline`): counter; CPU-side `vkSignalSemaphore`/`vkWaitSemaphores`; Vulkan 1.2 core
- Pipeline barriers (`vkCmdPipelineBarrier2`): `VkDependencyInfo`; stage/access masks; `VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT`
- Image layout transitions: state machine (UNDEFINED→COLOR_ATTACHMENT→SHADER_READ_ONLY→PRESENT); ownership transfer
- Render pass implicit sync: `VkSubpassDependency2`; `VK_SUBPASS_EXTERNAL`; when deps replace explicit barriers
- Events: `vkCmdSetEvent2`/`vkCmdWaitEvents2`; split-barrier; fine-grained GPU pipelining
- Queue ownership transfer: EXCLUSIVE vs CONCURRENT; release/acquire pair; async compute
- External sync: `VK_KHR_external_semaphore_fd`; DRM syncobj; `wp_linux_drm_syncobj_v1`
- Sync validation: VK_LAYER_KHRONOS_validation; `--synchronization-validation`; RenderDoc; common errors
- Real-world patterns: shadow map barrier; async texture upload; multi-GPU timeline semaphores
- **Integrations**: Ch16 (Vulkan core), Ch24 (EGL present path), Ch20 (linux_drm_syncobj), Ch133 (async compute queues)

---

## Part XXV — Advanced Topics and Gap-Fill

### Chapter 106: The Vulkan Memory Model — Formal Execution and Memory Ordering *(Part VII-A — GPU APIs)*
- The problem: GPU parallelism with no global sequential consistency; why undefined behaviour is real
- Vulkan Memory Model spec (`VK_KHR_vulkan_memory_model`, core in 1.2): execution scope, storage class, memory domain
- Execution scopes: `Invocation`, `Subgroup`, `Workgroup`, `QueueFamily`, `Device`; which barrier stages correspond to which scope
- Storage class semantics: `StorageBuffer`, `Uniform`, `Image`, `PhysicalStorageBuffer`; visibility rules per class
- Memory operations: `Load`, `Store`, `Atomic`; `MakeAvailable`, `MakeVisible`, `NonPrivate` decorations in SPIR-V
- Acquire/release semantics: `Acquire`, `Release`, `AcquireRelease`; happens-before graph construction
- Availability and visibility chains: why a `Release` in one invocation isn't automatically visible to another without explicit `Acquire`
- Subgroup operations: `subgroupBarrier`, `subgroupMemoryBarrier`; scope-limited vs. full barriers; divergence and non-uniform control flow
- Atomic operations: `AtomicAdd`, `AtomicExchange`, `AtomicCompareExchange`; sequentially consistent vs. relaxed atomics
- SPIR-V decorations in practice: `NonWritable`, `NonReadable`, `Coherent`; `MemorySemanticsMask`
- Shader compiler treatment: ACO/NIR memory model lowering; how barriers become hardware wait states
- Common bugs: missing `NonPrivate` on cross-invocation loads; release without corresponding acquire; wrong scope for subgroup reductions
- Validation: `VK_KHR_vulkan_memory_model` in `VkPhysicalDeviceVulkanMemoryModelFeatures`; synchronisation validation layer; spirv-val
- **Integrations**: Ch14 (NIR barriers), Ch15 (ACO hardware wait-state emission), Ch133 (compute queues and task graphs), Ch148 (sync primitives provide the execution ordering that the memory model operates within), Ch154 (GPU culling shaders rely on correct memory ordering for atomic draw counts)

### Chapter 149: GPU Hang Detection and Recovery — TDR, Scheduling Timeouts, and Reset Sequences *(Part I)*
- Why GPU hangs happen: infinite shader loops, command buffer errors, firmware faults, bus errors, power events
- Linux GPU hang taxonomy: soft hang (scheduler timeout), hard hang (GPU unresponsive), firmware crash, IOMMU fault
- `drm_gpu_scheduler`: heartbeat mechanism (`drm_sched_heartbeat_timer`); `timeout_ms`; `drm_sched_job_timedout`; per-engine vs. per-VM hangs
- amdgpu hang detection: `amdgpu_job_timedout`; `amdgpu_device_gpu_recover`; ring reset vs. full GPU reset; RAS (Reliability, Availability, Serviceability) for ECC errors
- amdgpu reset sequence: `amdgpu_device_pre_asic_reset` → `amdgpu_asic_reset` → IP block re-init → fence signalling for orphaned jobs; BACO vs. mode1 reset
- Intel i915/xe: `intel_gt_reset`; engine reset (`i915_gem_reset_engine`); GuC engine reset communication; hangcheck timer
- NVIDIA open kernel module: `nv_gpu_reset`; GSP-RM reset protocol; watchdog firmware timeout
- Nouveau: limited reset capability; context kill on hang; `nouveau_channel_kill`
- Kernel error reporting: `/sys/kernel/debug/dri/0/amdgpu_gpu_recover`; `/sys/kernel/debug/dri/0/i915_error_state`; GPU crash dump format
- Userspace recovery: `VK_ERROR_DEVICE_LOST` handling; `VkDeviceFaultInfoEXT` (`VK_EXT_device_fault`); graceful teardown vs. restart
- IOMMU faults: page fault on GPU IOVA access; `amdgpu_iommu_fault_handler`; ARM SMMU fault on Adreno/Mali
- Tools: `dmesg | grep -i "gpu\|hang\|reset"`; `amdgpu_top`; `intel_gpu_top`; RenderDoc GPU crash capture
- CI and testing: VKMS hang-free guarantee; IGT `gem_exec_hang_*` test suite; reproducing hangs deterministically
- **Integrations**: Ch01 (DRM driver lifecycle), Ch04 (drm_gpu_scheduler), Ch164 (power management — power loss triggers hang), Ch163 (VKMS hang-free by design), Ch159/Ch160 (Panfrost/Turnip hang detection)

---

## Part XXVII — New Appendices and Chapter Additions (June 2026)

### Appendix N: Vulkan on Linux — Platform Extensions Reference
The Linux/Wayland-specific Vulkan extensions absent from Windows/macOS: `VK_KHR_wayland_surface`, `VK_EXT_image_drm_format_modifier`, `VK_KHR_external_memory_fd`, `VK_EXT_external_memory_dma_buf`, `VK_KHR_external_semaphore_fd`, `VK_KHR_external_fence_fd`, `VK_EXT_acquire_drm_display`, `VK_KHR_display`/`VK_KHR_display_swapchain`, `VK_EXT_headless_surface`. Each extension: purpose, key structs/functions with C code examples, Mesa driver support status. Cross-links: Ch4 (DMA-BUF), Ch20 (linux-dmabuf), Ch24 (Vulkan app devs), Ch75 (explicit sync), Ch121 (DRM lease).

### Appendix O: SPIR-V Binary Format Reference
Compact reference card: 5-word module header (magic `0x07230203`, version, generator, bound, schema), instruction encoding (opcode in bits [15:0], word count in bits [31:16]), logical module section order, ~60-entry opcodes table, capabilities table, storage classes, decorations (with BuiltIn sub-table), execution models and modes, `spirv-dis`/`spirv-val`/`spirv-opt`/`spirv-cross`/`glslangValidator`/`dxc`/`tint` command recipes, annotated `spirv-dis` listing. Cross-links: Ch14 (NIR), Ch16 (Mesa Vulkan common), Ch61 (SPIR-V ecosystem), Ch77 (shader toolchain), Ch110 (SPIR-V tooling).

### Appendix P: WGSL Language Reference
Compact reference card for the WebGPU Shading Language (W3C Candidate Recommendation, June 2026). Covers: scalar/vector/matrix/array/struct/atomic/pointer types with short-form aliases; texture and sampler types (sampled, depth, storage); address spaces (`function`, `private`, `workgroup`, `uniform`, `storage`, `handle`); all attributes (`@group`/`@binding`, `@location`, `@builtin`, `@interpolate`, `@align`/`@size`, `@invariant`, `@workgroup_size`); built-in values by stage (vertex/fragment/compute); all built-in functions (math, bit manipulation, type conversion, texture sampling, pack/unpack, atomics, synchronisation, derivatives, subgroup operations); control flow; entry-point structure templates for vertex/fragment/compute; key differences from GLSL with common pitfall callouts. Cross-links: Ch35 (Dawn/Tint), Ch40 (Bevy/wgpu/naga), Ch61 (SPIR-V front ends), Ch98 (WebGPU/WASM), Appendix L (shader toolchain matrix), Appendix O (SPIR-V output).

### Appendix Q: Shader Language Comparison Reference
Side-by-side reference for GLSL 4.60, HLSL SM 6.6, WGSL (CR June 2026), MSL 3.1, and OpenCL C 3.0. Covers: scalar types, vector and matrix types (with short-form aliases and storage-order differences), storage qualifiers/address spaces, entry-point syntax (vertex/fragment/compute templates in all five languages), built-in variables by stage, resource binding syntax, texture sampling functions, workgroup size declarations, shared/workgroup memory, barrier types, control-flow differences (ternary, discard, recursion, implicit conversion, switch fallthrough), preprocessor/module systems, and compilation target + toolchain table (compiler, output format, Linux path to GPU). Includes `spirv-cross` cross-compilation matrix. Cross-links: Ch35, Ch61, Ch77, Ch98, Ch110, Appendix L, O, P.

### Appendix R: EGL Platform Reference
EGL 1.5 + KHR/EXT/MESA extensions quick reference (Mesa 25.1). Covers: initialisation sequence; platform extension matrix (Wayland, GBM, X11, XCB, Surfaceless, EGLDevice) with native display/window types; `eglChooseConfig` attribute reference (colour buffer, depth/stencil, MSAA, `EGL_RENDERABLE_TYPE`, `EGL_SURFACE_TYPE`, config caveat); surface types and creation (window, pbuffer, surfaceless, DMA-BUF import via `EGL_EXT_image_dma_buf_import`); context creation attributes (major/minor version, profile, debug, robust access, reset notification); key Linux EGL extensions (platform, buffer sharing, sync/fence, swap damage); EGL error codes with common causes; common patterns (GBM compositor loop, headless DMA-BUF export). Cross-links: Ch4, Ch20, Ch21, Ch24, Appendix G, N.

### Appendix S: DRM/KMS ioctl Quick Reference
Linux kernel 6.12 / libdrm 2.4.123 ioctl reference. Covers: device/authentication ioctls and `DRM_CAP_*`/`DRM_CLIENT_CAP_*` tables; KMS object discovery (resources, connectors, CRTCs, planes, object properties); atomic commit API (`drmModeAtomicReq`, `drmModeAtomicAddProperty`, commit flags, mode blob creation, flip event handling); legacy KMS ioctls; framebuffer management (`drmModeAddFB2WithModifiers`); GEM buffer management (close, map, flink, open); PRIME DMA-BUF import/export (`drmPrimeHandleToFD`, `drmPrimeFDToHandle`); dumb buffers (software rendering path); syncobj API (create/destroy, export/import sync_file, wait, reset, signal, timeline variants); vblank/timestamp; KMS object property tables (CRTC, connector, plane properties with types and descriptions). Cross-links: Ch1, Ch3, Ch4, Ch75, Appendix G, H, N.

### Appendix T: Terminal Graphics Protocol Reference
Side-by-side reference for the three terminal image protocols (baseline: xterm 389, Kitty 1.5.0, iTerm2 3.0, June 2026). Covers: protocol comparison matrix (escape type, colour depth, alpha, chunking, shared memory, placement, animation, query); Sixel protocol (DCS wrapper, parameter reference, colour introduction syntax, sixel band encoding, control characters, libsixel C API and CLI); Kitty Graphics Protocol (APC wrapper, full key-value reference table for action/transmission type/format/dimensions/placement/z-index/cursor-movement/animation/delete, chunked transmission, shared-memory transfer, virtual placement, query, Rust sketch for chunked RGBA); iTerm2 Inline Image Protocol (OSC 1337 wrapper, `File=` argument reference, other OSC 1337 commands); terminal support matrix; detection/capability query (Kitty query, DA1 Sixel bit, `$TERM_PROGRAM`); common client patterns (Python detection, tmux pass-through, protocol selection priority). Cross-links: Ch67–70, Appendix J.

### Appendix U: WebGPU API Quick Reference
W3C WebGPU Candidate Recommendation (June 2026) API reference covering the JavaScript/WebIDL surface with `wgpu` Rust equivalents noted. Covers: adapter/device initialisation; `GPUBuffer` (usage flags, `mapAsync`/`getMappedRange`/`unmap`/`writeBuffer`); `GPUTexture` and `GPUTextureView` (usage flags, common `GPUTextureFormat` table, `writeTexture`); `GPUSampler` (address modes, filter modes, compare); `GPUShaderModule` and `getCompilationInfo`; bind group layouts and bind groups (buffer/texture/sampler/storage-texture binding types, `GPUShaderStage` flags); `GPURenderPipeline` (vertex buffer layout, vertex formats table, fragment targets and blend, primitive state, depth-stencil, MSAA); `GPUComputePipeline` (async creation); `GPUCommandEncoder` render pass (all `GPURenderPassEncoder` methods: draw, drawIndexed, indirect, viewport, scissor, blend constant, occlusion queries) and compute pass (`dispatchWorkgroups`, indirect); copy commands and debug groups; `GPUQueue` (`submit`, `writeBuffer`, `writeTexture`, `onSubmittedWorkDone`); canvas and swap chain (browser `GPUCanvasContext` + wgpu `Surface`); key enum tables (topology, cull mode, blend factors/operations, stencil ops, load/store ops, map mode, query type, texture dimension/aspect, address/filter modes, selected `GPUFeatureName`); error handling (error scope stack, `uncapturederror`). Cross-links: Ch35, Ch40, Ch98, Appendix P, Q.

### Section additions to existing chapters (cross-chapter content)

**Ch93 §13: End-to-End Frame Delivery Latency** *(Part IX — added June 2026)*: per-hop latency budget table (CPU → vkQueueSubmit → DRM scheduler → GPU render → wl_surface.commit → compositor → KMS → vblank → panel); `wp_presentation_feedback` for per-frame scanout timestamps; MangoHUD `present_timing`; `ftrace drm:drm_vblank_event` for vblank jitter; motion-to-photon budget at 144 Hz; optimisation per hop (VRR, direct scanout, MPRT).

### Section additions to existing chapters (cross-chapter content — Part XXV)

---

## Chapter Body Entries — Previously Unregistered Chapters (July 2026)

The following chapters existed as `.md` files but had no body entry in plan.md.
Entries are generated from the chapter content and sorted by chapter number within each Part.

### Part I — The Kernel Layer

### Chapter 162: Framebuffer Compression: AFBC, DCC, CCS, and UBWC *(Part I — Kernel Layer)*

- Bandwidth motivation: 4K@60 XRGB8888 consumes ~1920 MB/s uncompressed; hardware compression targets 3-4x reduction to fit within LPDDR5 budgets, also improving cache efficiency via denser cache-line packing
- AFBC (ARM Frame Buffer Compression): 16x16 superblock layout with 16-byte header + variable body; exploits solid-colour sub-blocks and intra-frame prediction; modifiers include AFBC_FORMAT_MOD_BLOCK_SIZE_16x16, YTR, SPARSE, TILED; supported on Mali Bifrost/Valhall and ARM DPU (RK3399, RK3588)
- AMD DCC (Delta Colour Compression): per-256-byte micro-tile delta encoding with DCC key (0x00=constant, 0x55=compressible, 0xFF=uncompressed); AMD_FMT_MOD_DCC, DCC_RETILE, DCC_PIPE_ALIGN modifiers; direct scanout via AMD Display Core decompressor; radv_use_dcc_for_image_late() in radv_image.c
- Intel CCS (Colour Control Surface): 2-bit tag per 8x8 tile (00=uncompressed, 01=clear-colour, 10=partial, 11=unresolvable); modifiers I915_FORMAT_MOD_Y_TILED_CCS through I915_FORMAT_MOD_4_TILED_BMG_CCS spanning Gen9 to Xe2 (Lunar Lake/Battlemage)
- ANV CCS integration: anv_image planes array with aux_usage = ISL_AUX_USAGE_CCS_E; isl_format_supports_ccs_e() gating; ISL (Intel Surface Layout) abstracts tiling and CCS placement
- Qualcomm UBWC (Universal Bandwidth Compression): 4x4 micro-tiles with 1-byte metadata per tile (fast-clear, compression levels, uncompressed); DRM_FORMAT_MOD_QCOM_COMPRESSED and QCOM_TILED modifiers; enabled on all A5xx+ in Turnip and Freedreno
- DRM format modifier structure: 64-bit value with upper 8 bits for vendor (ARM=0x08, AMD=0x02, INTEL=0x03, QCOM=0x05); DRM_FORMAT_MOD_LINEAR=0, DRM_FORMAT_MOD_INVALID=-1; drm_format_modifier_blob struct and IN_FORMATS plane property for advertising supported modifier/format pairs
- IN_FORMATS blob API: drmModeObjectGetProperties + drmModeGetPropertyBlob to retrieve drm_format_modifier_blob; iterate count_formats and count_modifiers to enumerate plane-supported compressed formats
- EGL modifier negotiation: eglQueryDmaBufModifiersEXT queries per-format supported modifiers with external_only flag; eglCreateImageKHR with modifier attributes for compressed buffer import via EGL_LINUX_DMA_BUF_EXT
- zwp_linux_dmabuf_feedback_v1 Wayland protocol: compositor sends format+modifier table via dmabuf_feedback_format_table and dmabuf_feedback_tranche_formats callbacks; enables zero-copy scanout by having clients allocate in compositor-preferred compressed format
- Debugging tools: WAYLAND_DEBUG=1, drm-info IN_FORMATS, AMD_DEBUG=nodcc, INTEL_DEBUG=noccs, apitrace for modifier inspection; GALLIUM_HUD=dcc-compressed,dcc-uncompressed for AMD compression ratio counters; /sys/kernel/debug/dri/0/state for ARM AFBC plane modifiers
- Roadmap near-term: AMD DCC for multi-plane/video on RDNA4 (Mesa 25.1), RADV DCC fast-clears on RDNA3 (8bpp/16bpp), Intel CCS restoration for Xe2 via SR-IOV in Linux 6.18, UBWC gralloc detection fixes in Turnip/Freedreno, AFBC for Rockchip RK3588 and MediaTek DPU
- Roadmap long-term: cross-vendor compressed buffer import/export, lossy compression tiers (ARM Fixed-Rate Compression) via modifier quality field, transparent compression below DRM at SMMU/IOMMU level, standardised dma_buf sidecar for compression metadata
- **Integrations**: Ch4 (DRM Framebuffers: drmModeAddFB2WithModifiers), Ch6 (KMS Atomic: IN_FORMATS plane property), Ch11 (DMA-BUF: modifier as metadata with EGL_LINUX_DMA_BUF_EXT and zwp_linux_dmabuf_v1), Ch150 (EGL/DMA-BUF: eglQueryDmaBufModifiersEXT and eglCreateImageKHR), Ch22 (RADV: DCC management and fast-clear elimination), Ch23 (ANV: CCS via ISL Intel Surface Layout), Ch159 (Panfrost: AFBC on Mali hardware for RK3588 zero-copy scanout)


### Chapter 163: VKMS and Virtual Display Drivers for Testing *(Part I — Kernel Layer)*

- VKMS overview (Linux 4.14+): software DRM driver in drivers/gpu/drm/vkms/ implementing full atomic KMS (CRTC, encoder, connector, planes) without hardware; module params enable_cursor, enable_overlay, enable_writeback, enable_plane_pipeline
- drm_driver flags: DRIVER_MODESET | DRIVER_ATOMIC | DRIVER_GEM with DRM_GEM_SHMEM_DRIVER_OPS for system-RAM framebuffer allocation via drm_gem_shmem_helper
- hrtimer-based vblank simulation: vkms_vblank_simulate() fires at vrefresh period (~16.67ms at 60Hz), calls drm_crtc_handle_vblank() and enqueues composition work via alloc_ordered_workqueue
- Software composition engine: line-by-line scanline compositing using pixel_argb_u16 (4x16-bit channels), premultiplied-alpha blend via pre_mul_alpha_blend(), zpos-sorted plane stack; per-format pixel_read_line_t / pixel_write_line_t callbacks in vkms_formats.c
- DRM Color Pipeline API (vkms_colorop.c): VKMS is the in-tree reference driver for DRM_COLOROP_1D_CURVE (sRGB/PQ EOTF), DRM_COLOROP_CTM_3X4, DRM_COLOROP_MULTIPLIER, DRM_COLOROP_3D_LUT (17^3 tetrahedral LUT); apply_colorop() runs post-composition before writeback/CRC
- EDID emulation: drm_edid_build_fake() exposes 1920x1080@60Hz preferred mode; configfs runtime EDID injection and HPD trigger via /config/vkms/<dev>/connectors/conn0/edid and status attributes
- VKMS writeback connector: DRM_MODE_CONNECTOR_WRITEBACK backed by drm_writeback_connector / drm_writeback_job; WRITEBACK_FB_ID + WRITEBACK_OUT_FENCE_PTR (DMA-fence sync_file) allow pixel-exact frame capture; drm_writeback_signal_completion() signals fence after last scanline
- CRC testing infrastructure: crc32_le() over pixel_argb_u16 output buffer per frame; drm_crtc_add_crc_entry() feeds /sys/kernel/debug/dri/0/crtc-0/crc/data; IGT igt_pipe_crc_t API; kms_cursor_crc and kms_plane_multiple tests via IGT_FORCE_DRIVER=vkms
- KUnit tests (drivers/gpu/drm/vkms/tests/): get_lut_index, lerp_u16, format round-trip tests; CONFIG_DRM_VKMS_KUNIT_TEST; run via kunit.py without hardware on x86-64/arm64/RISC-V
- VKMS + Mesa CI stack: VKMS + LLVMpipe/lavapipe + sway/weston for headless Wayland CI; WLR_BACKENDS=drm, LIBGL_ALWAYS_SOFTWARE=1; deqp-runner parallelises VK-GL-CTS against lavapipe; Weston uses virtme (QEMU wrapper) for DRM-backend testing
- Configfs runtime reconfiguration (Linux 6.13): /config/vkms/<name>/{planes,crtcs,encoders,connectors} directories with symlink-based pipeline wiring; create_default_dev=0 enables multi-instance management; hot-plug simulation via status attribute
- Headless backend comparison: VKMS (full DRM/KMS+CRC, no GPU), Xvfb (X11 only, no DRM), GBM offscreen (rendering only, no display), virtio-gpu/virgl (VM-based, approximate GPU), Xvnc (network VNC)
- DRM lease API for VR direct mode: drmModeCreateLease() delegates CRTC/connector/plane subset to VR runtime fd; drmModeRevokeLease(); Monado OpenXR uses leases for direct-to-HMD rendering bypassing compositor
- GPU testing with LLVMpipe/lavapipe: LIBGL_ALWAYS_SOFTWARE=1 for OpenGL 4.5; VK_ICD_FILENAMES=lvp_icd.x86_64.json for Vulkan 1.3 CTS conformance testing without GPU
- **Integrations**: Ch01 (DRM Architecture), Ch02 (KMS Display Pipeline), Ch34 (Wayland), Ch121 (DRM Lease/VR Direct Display), Ch125 (RenderDoc), Ch139 (DRM Hardware Planes), Ch147 (VA-API Video Decode), Ch155 (evdi/DisplayLink)


### Part II — GPU Drivers

### Chapter 73: Asahi Linux and the Apple Silicon AGX Driver *(Part II — GPU Drivers)*

- AGX GPU family (G13/G14 generations): TBDR architecture; SoC mapping T8103 (M1) through T6022 (M2 Ultra); gpu_generation/num_clusters exposed via DRM_IOCTL_ASAHI_GET_PARAMS returning struct drm_asahi_params_global
- TBDR pipeline: VDM (Vertex Data Master) → PPP (Primitive Processing Pipeline) → TVB (Tiled Vertex Buffer) for TA pass; ISP (Image Synthesis Processor) + on-chip tilebuffer + BG/EoT programs for fragment pass; partial render handling
- Reverse engineering methodology: m1n1 bare-metal hypervisor for MMIO peripherals; DYLD_INSERT_LIBRARIES / IOKit interception for GPU command buffers; wrap library and agxdecode tool in Mesa; G13G ISA documented by iterative shader binary diffing
- drm/asahi Rust kernel driver: first Rust driver in DRM subsystem; 70+ firmware ABI packed struct types in fw/ directory; module structure: driver.rs, gpu.rs, mmu.rs, gem.rs, channel.rs, microseq.rs, crashdump.rs, hw/t8103.rs etc.
- UAT (Unified Address Translator) GPU MMU: ARM64-compatible page tables with 16 KiB pages; 40-bit output address space; 64 VM contexts (TTBAT); CPU-firmware page table handoff via atomic flush state flags
- ASC co-processor communication: RTKit channel ring buffers in channel.rs; firmware microsequence VM instructions in microseq.rs; no hardware register writes for work submission; crash recovery via crashdump.rs
- Asahi UAPI (merged into mainline without driver): DRM_IOCTL_ASAHI_VM_CREATE, GEM_CREATE (WRITEBACK flag), VM_BIND (BIND_SINGLE_PAGE for sparse), QUEUE_CREATE (usc_exec_base), SUBMIT with explicit DRM syncobj; struct drm_asahi_cmd_render carries vdm_ctrl_stream_base, bg/eot tile programs, tile dimensions, drm_asahi_timestamps
- Mesa Asahi Gallium driver (src/gallium/drivers/asahi/): agx_context / agx_draw_vbo state tracker; NIR compiler backend with varying lowering to st_var/ldcf/iter, uniform register file (256 entries), image layout selection (strided/GPU-tiled/twiddled); geometry shaders and TF lowered to compute dispatches
- OpenGL 4.6 conformance on AGX: buffer robustness via unsigned-minimum clamping; image robustness via forced out-of-bounds texture coordinates; clip control via vertex shader epilogue remapping clip-space Z
- Honeykrisp Vulkan driver (src/asahi/vulkan/): forked from NVK (Mesa NVIDIA Vulkan driver); Vulkan render pass load/store ops mapped to synthesised BG/EoT USC programs in struct drm_asahi_bg_eot; prolog/epilog shader split for dynamic state; VK_EXT_image_drm_format_modifier for zero-copy WSI; Vulkan 1.3 conformance June 2024, Vulkan 1.4 late 2024; sparse binding in Mesa 25.1 enabling VKD3D-Proton DX12_0
- DCP (Display Control Processor): embedded ARM64 ASC core running RTKit RTOS; layered protocol: apple-mailbox (96-bit messages) → apple-rtkit → EPIC service calls; drm/apple KMS driver implements atomic DRM/KMS on top; VRR requires full modeset; USB-C external display via fairydust branch (DCP + DPXBAR + ATCPHY + ACE)
- UMA implications: no VRAM/GTT split; all GEM BOs in shared LPDDR5/LPDDR5X; cache-coherent CPU-GPU without drm_clflush_sg; DMA-BUF zero-copy via drm_gem_prime_import; DRM_ASAHI_BIND_SINGLE_PAGE + DRM_ASAHI_FEATURE_SOFT_FAULTS for Vulkan sparse binding
- Conformance and status: OpenGL 4.6, OpenGL ES 3.2, OpenCL 3.0, Vulkan 1.4 certified; UAPI in mainline; Gallium and Honeykrisp fully upstream in Mesa 24.3+; drm/asahi kernel driver (~21,000 lines) out-of-tree; M3/M4 (G15/G16) GPU acceleration awaits reverse engineering of hardware ray tracing and Dynamic Caching
- **Integrations**: Ch1 (DRM Architecture), Ch4 (GPU Memory Management / GEM), Ch6 (ARM & Embedded GPU Drivers), Ch7 (Reverse Engineering / Nouveau), Ch10 (NVK — Honeykrisp forked from NVK), Ch14 (NIR — AGX compiler backend), Ch16 (Mesa Vulkan Common Infrastructure), Ch18 (Vulkan Drivers landscape), Ch75 (Explicit GPU Sync)


### Chapter 155: USB DisplayLink and the evdi Kernel Module *(Part II — GPU Drivers)*

- DisplayLink architecture: software-defined display pipeline where host CPU encodes frames via DL3 codec and sends over USB bulk transfers to a decoder-only DisplayLink chip (DL-3xxx/5xxx/6xxx/7400)
- evdi (Extensible Virtual Display Interface): out-of-tree DKMS DRM driver creating virtual /dev/dri/cardN devices with DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC; custom ioctl table (EVDI_CONNECT, EVDI_REQUEST_UPDATE, EVDI_GRABPIX, EVDI_DDCCI_RESPONSE, EVDI_ENABLE_CURSOR_EVENTS)
- evdi kernel module structure: evdi_drm_drv.c, evdi_painter.c, evdi_gem.c, evdi_fb.c, evdi_modeset.c, evdi_cursor.c, evdi_sysfs.c, evdi_i2c.c; sysfs-triggered device lifecycle via /sys/bus/platform/driver/evdi/add
- evdi GEM memory model: evdi_gem_object wraps drm_gem_object with shmem-backed pages; lazy page allocation; drm_clflush_pages() for cache coherency; VM_MIXEDMAP mmap flag to enable struct-page access for CLFLUSH on x86
- evdi painter damage tracking: evdi_painter struct with MAX_DIRTS=16 dirty_rects ring; evdi_painter_mark_dirty() merges/collapses overflow via bounding-box; drm_atomic_helper_dirtyfb + FB_DAMAGE_CLIPS plane property; DRM_EVDI_EVENT_UPDATE_READY notification via poll()/read()
- EVDI_GRABPIX pixel delivery: struct drm_evdi_grabpix carries dirty rects and client buffer pointer; copy_primary_pixels() performs CPU memcpy from GEM pages to DLM buffer; fundamental bottleneck: ~8 MB copy for 1080p at 1-5 ms latency
- Cursor emulation: default software cursor compositing via evdi_cursor_compose_and_copy() with per-pixel alpha blend; opt-in cursor event mode (EVDI_ENABLE_CURSOR_EVENTS) delivers DRM_EVDI_EVENT_CURSOR_SET/MOVE instead of compositing into framebuffer
- libevdi userspace API: evdi_open(), evdi_connect(), evdi_register_buffer(), evdi_handle_events() with callback-based event model; evdi_connect2() adds pixel_area_limit and pixel_per_second_limit bandwidth constraints; request-notify-grab render loop
- DisplayLink Manager (DLM): proprietary daemon with plugin pipeline (source/transform/sink); udev rules (74-displaylink.rules, VID 17e9); systemd displaylink-driver.service; DKMS-bundled installer; closed-source due to DL3 codec IP, HDCP 2.0 DCP LLC licensing barrier, and cross-platform code reuse
- DRM integration and atomic modesetting: compositor commits via drmModeAtomicCommit() with FB_DAMAGE_CLIPS; double CPU copy path (GPU DMA-BUF -> evdi GEM shmem -> DLM buffer); single primary plane only; NVIDIA proprietary driver incompatibility
- USB4 and DL-7400: quad 4K@120Hz / dual 8K@60Hz HDR10; DisplayPort 2.1 Alt Mode; USB4 Gen 3x2 (40 Gbps); distinction between DisplayLink-over-USB4 (requires evdi+DLM) vs. native USB4 DP tunneling (kernel thunderbolt/typec subsystem, no daemon needed)
- Performance limitations: 10-35 ms end-to-end latency; 25-40% CPU for 1080p video; gaming impractical due to variable DL3 encode time and no hardware overlays; Wayland xdg-desktop-portal screen capture broken for evdi virtual displays
- Open-source ecosystem: udl (mainline, USB 2.0 DL-1xx only); udlfb (legacy fbdev with fb_defio, considered deprecated); Vino (experimental Rust DRM driver for DL-3xxx via usbmon reverse engineering, 2026); pyevdi, displaylink-debian, displaylink-rpm community packaging
- Upstreaming blocker: DRM maintainers require open userspace before accepting drivers defining new DRM uAPI; closed DLM prevents evdi mainlining; open DLM community request exceeds 2,000 votes on DisplayLink forum with no substantive response
- **Integrations**: Ch01 (DRM Architecture), Ch04 (Framebuffer and Planes), Ch06 (KMS Atomic Modesetting), Ch11 (DMA-BUF), Ch49 (Multi-GPU), Ch139 (DRM Hardware Planes), Ch147 (VA-API and Video Decode), Ch172 (eGPU / Thunderbolt / USB4)


### Chapter 160: Freedreno and Turnip: Open-Source Qualcomm Adreno Driver *(Part II — GPU Drivers)*

- Adreno GPU families (A3xx–A8xx): SIMT warp-based architecture with SP, UCHE L2, TP, RB, CP, VSC hardware blocks; wave128 on A6xx, wave64 on A7xx
- TBDR binning pass and VSC (Visibility Stream Compressor): vertex-only binning emits per-tile primitive bitmasks into 16 (A6xx) or 32 (A7xx) VSC pipes; tu6_emit_binning_pass in tu_cmd_buffer.c
- LRZ (Low Resolution Z): 1/8th-resolution hierarchical Z-buffer for early fragment rejection; tu_lrz.c tracks per-draw enable/disable conditions; A7xx adds early-LRZ-late-depth mode
- GMU (Graphics Management Unit): ARM Cortex-M3 coprocessor handling GPU power/clock; ACTIVE/IFPC/SLUMBER states; a6xx_gmu_set_oob() OOB handshake; HFI shared-memory circular queues (H2F/F2H)
- Freedreno Gallium3D driver: fd6_gmem.c GMEM/binning pass; autotune algorithm selects GMEM vs SYSMEM mode; TU_AUTOTUNE_ALGO env var
- Turnip Vulkan driver (A6xx+): Vulkan 1.3, tu_renderpass.c GMEM-aware render pass, tu_pick_gmem_layout(), VK_EXT_physical_device_drm for DRM node correlation
- UBWC (Universal Bandwidth Compression): lossless render target compression with 4-byte metadata per 256-byte tile; DRM_FORMAT_MOD_QCOM_COMPRESSED; VK_EXT_image_drm_format_modifier for zero-copy compositor integration
- AFBC on Adreno: import-only (DPU scan-out of codec/ISP AFBC buffers); GPU shader units do not write AFBC; DRM_FORMAT_MOD_ARM_AFBC_* modifiers via zwp_linux_dmabuf_v1
- A7xx/A8xx new features: concurrent binning (BR/BV split), 2 MB GMEM, 32 VSC pipes, ray tracing (VK_KHR_ray_query merged Mesa 25.0 for A740+), VK_EXT_mesh_shader; A8xx VRS pending
- ir3 compiler pipeline: GLSL/SPIR-V -> NIR -> ir3 IR -> register allocation -> a6xx binary; 64-bit RISC ISA with (sy)/(ss) sync markers; fp16 at 2x throughput; IR3_SHADER_DEBUG flags
- msm_drm kernel driver: a6xx_gpu.c, a6xx_gmu.c, a6xx_ringbuffer.c (CP_INDIRECT_BUFFER_PFE ring submission); msm_gem.c GEM BO management; SMMU/IOMMU IOVA mapping
- Debugging and profiling tools: fdperf (ncurses hardware counter monitor), cffdump/.rd command stream capture/decode, rddecompiler, replay, TU_BREADCRUMBS UDP hang debugging, Perfetto GPU counter tracing
- Freedreno vs Turnip vs Zink: Freedreno for legacy A3xx-A5xx; Turnip as primary for A6xx+ Vulkan; Zink-on-Turnip for OpenGL 4.6 (used by Steam Frame VR); driver selection matrix by workload
- Snapdragon X Elite Linux status (kernel 6.19, Mesa 26.0): Adreno 830 Vulkan 1.3 working; iris V4L2 video decoder (Linux 6.15); suspend/resume in progress; A8xx Gen 8 merged Mesa 26.0
- **Integrations**: Ch1 (DRM Architecture), Ch11 (DMA-BUF), Ch14 (Gallium3D), Ch15 (NIR), Ch19 (Vulkan), Ch20/21 (Wayland/wlroots), Ch159 (Panfrost/Mali), Ch169 (Snapdragon X Elite Linux)


### Chapter 169: Snapdragon X Elite on Linux: Adreno X1-85, freedreno, and the Arm Laptop Era *(Part II — GPU Drivers)*

- Adreno X1-85 hardware (X185 / ADRENO_7XX_GEN2): 1,536 FP32 ALUs, 3 MB GMEM, 256 GB GPU VA, 4.6 TFLOPS peak; LPDDR5X unified memory shared with Oryon CPU
- DRM/MSM kernel driver (drivers/gpu/drm/msm/): X185 catalog entry landed targeting Linux 6.11; gmxc.lvl power rail, gen70500_sqe.fw / gen70500_gmu.bin / gen70500_zap.mbn firmware; GPU node disabled by default pending firmware extraction
- ACPI + Device Tree overlay boot model: UEFI-based X Elite uses DTB via EFI stub alongside ACPI; per-OEM per-laptop .dts files in arch/arm64/boot/dts/qcom/ for ThinkPad T14s Gen 6, Dell XPS 13 9345, ASUS Vivobook S 15, HP OmniBook X 14, etc.
- GMU ring submission (a6xx_gpu.c): CP_INDIRECT_BUFFER_PFE PM4 packets, OUT_PKT7/OUT_RING macros, msm_fence signalling; IFPC (Inter-Frame Power Collapse) and speedbin support added in kernel 6.17
- Mesa Turnip Vulkan driver (src/freedreno/vulkan/): SPIR-V → NIR → IR3 → CP_LOAD_STATE7; Vulkan 1.3 on X1-85; VK_KHR_ray_query merged Mesa 25.0; Turnip added to default AArch64 Mesa build in Mesa 25.1
- freedreno Gallium3D (src/gallium/drivers/freedreno/): A7xx OpenGL paths in Mesa 24.3; Zink (OpenGL→Vulkan translation via MESA_LOADER_DRIVER_OVERRIDE=zink) as fallback path on Turnip; Adreno X2-85 (Gen 8) support in Mesa 26.0
- msm_dpu display pipeline (dpu_9_2_x1e80100.h): 4 VIG + 6 DMA planes, 6 layer mixers, 4 DSPP, 4 DSC encoders, max_line_width 5120, has_src_split/has_3d_merge/has_idle_pc; eDP Panel Self-Refresh (PSR) for battery life
- UBWC (Universal Bandwidth Compression): Qualcomm proprietary tiled memory layout; DMA-BUF modifier negotiation with Wayland compositor via zwp_linux_dmabuf_v1; gbm_bo_create_with_modifiers(); zero-copy import from V4L2/camera; UBWC config consolidated in kernel 6.17
- IRIS V4L2 video decoder (drivers/media/platform/qcom/iris/): stateful V4L2 M2M decoder, H.264 decode upstreamed in Linux 6.15; H.265/AV1/VP9 decode in development; no upstream VA-API backend for IRIS; GStreamer v4l2h264dec / FFmpeg v4l2m2m as direct V4L2 paths
- Hexagon NPU / CDSP (remoteproc/qcom_*.c): no open upstream compute driver; accel/qda RFC patch series (2026) unmerged; SNPE/QNN not available on Linux; Qualcomm declined to open-source DSP headers; distinct from qaic (Cloud AI 100 PCIe card)
- Firmware distribution: gen70500 blobs (sqe.fw, gmu.bin, zap.mbn) via linux-firmware-qcom package or Windows-partition extraction using qcom-firmware-extract; Lenovo ThinkPad T14s Gen 6 only model with blobs in linux-firmware.git
- Kernel version timeline: 6.8/6.9 core SoC infra; 6.11 GPU+DPU targeted; 6.14 functional GPU bring-up (with Mesa 25.0); 6.15 IRIS H.264; 6.17 IFPC speedbin UBWC cleanup; recommended stack: kernel 6.17 + Mesa 25.1 + Ubuntu 25.04
- **Integrations**: Ch6 (ARM/Embedded GPU Drivers — msm DRM, remoteproc, SMMU, DT vs. ACPI), Ch88 (NPU and AI Accelerators — Hexagon gap vs. Intel IVPU/qaic), Ch160 (freedreno/Turnip/Adreno — A6xx/A7xx architecture, IR3 compiler, UBWC), Ch168 (WebNN — Hexagon NPU backend blocked; fallback to WebGPU/Turnip or XNNPACK)


### Chapter 172: eGPU on Linux — Thunderbolt, USB4, and PCIe Hot-Plug *(Part II — GPU Drivers)*

- Thunderbolt/USB4 PCIe tunneling architecture: bandwidth comparison across TB3/TB4/USB4 Gen2x2/USB4 v2 (TB5); security levels (none/user/secure/dponly/usbonly/nopcie) via /sys/bus/thunderbolt/devices/domainX/security
- bolt daemon (boltd): freedesktop.org D-Bus service for Thunderbolt device enrollment; boltctl commands (list, authorize, enroll, forget, monitor); sysfs authorized attribute and udev rule auto-authorization
- Linux kernel Thunderbolt/USB4 subsystem (drivers/thunderbolt/): CONFIG_USB4; key files nhi.c, tb.c, switch.c, tunnel.c, usb4.c; Firmware vs Software connection manager models
- PCIe tunnel creation via tb_tunnel_alloc_pci() in tunnel.c: bandwidth allocation, adapter register config, upstream/downstream PCIe path activation triggering pciehp enumeration
- DRM PCIe hot-plug lifecycle: drm_dev_register() on probe creating /dev/dri/cardN nodes; drm_dev_unplug() for surprise removal returning ENODEV; drm_dev_enter()/drm_dev_exit() guards for hardware access serialization
- amdgpu hot-unplug support (Linux 5.14+): drm_dev_unplug() in amdgpu_pci_remove(); drm_gpu_scheduler job draining; TTM BO eviction on backing store disappearance; Runtime PM D3cold over TB link
- PRIME reverse offload: DRM_IOCTL_PRIME_HANDLE_TO_FD / PRIME_FD_TO_HANDLE for eGPU-to-iGPU DMA-BUF sharing; bandwidth math (~16 Gbps for 4K@60Hz consuming ~70% of TB3 PCIe budget)
- Wayland eGPU integration: DRI_PRIME env var (index or pci-XXXX path); wlroots udev-based GPU hot-plug (PR #2423); all-ways-egpu script (WLR_DRM_DEVICES, MUTTER_DEBUG_FORCE_KMS_MODE, KWIN_DRM_DEVICES)
- NVIDIA eGPU: AllowExternalGpus Xorg option; proprietary driver lifecycle vs drm_dev_unplug; RTX 5080 TB5 CUDA hard-lock bug (#979); Nova Rust driver (DRM-next, kernel 6.15 era) unverified for eGPU
- Comparative external GPU technologies: OCuLink (raw PCIe ×4, no encapsulation, 0.5m); M.2 PCIe adapters (ADT-Link R43SG); NVLink/Infinity Fabric (GPU-to-GPU, not eGPU); InfiniBand GPU-Direct RDMA (HPC)
- USB4 v2 / Thunderbolt 5: 80 Gbps (120 Gbps Bandwidth Boost asymmetric); ~64 Gbps PCIe equiv to PCIe 4.0 ×4; kernel USB4 v2 support added in ~6.4-6.5 via Westerberg 20-patch series; usb4.c extended encapsulation
- Thunderclap DMA attack surface (NDSS 2019): pre-authorization device is no DMA master; mitigations via iommu_dma_protection sysfs, bolt secure mode cryptographic challenge, per-device iommu policy
- Practical setup workflow: lspci/lsmod Thunderbolt verification; boltctl enroll; PCIe device visibility; kernel boot params (pci=assign-busses,hpmmioprefsize=16G pcie_ports=native); lstbt topology tool from thunderbolt-utils
- **Integrations**: Ch4 (GPU Memory Management / DMA-BUF PRIME), Ch5 (x86 GPU Drivers / amdgpu & xe probe/remove), Ch49 (Multi-GPU and PRIME Render Offload), Ch51 (GPU Power Management and Thermal), Ch80 (GPU Security)


### Chapter 179: The Linux `accel` Subsystem: NPU and AI Accelerator Drivers *(Part II — GPU Drivers)*

- Motivation and Linux 6.2 origin: separate `/dev/accel/` subsystem created to avoid mis-fitting NPU/AI accelerator drivers into DRM or misc char devices; merged February 2023 via drm-next
- DRM foundation and DRIVER_COMPUTE_ACCEL flag: accel reuses struct drm_device, GEM, DRM scheduler, DMA-BUF, debugfs; DRIVER_COMPUTE_ACCEL is mutually exclusive with DRIVER_RENDER and DRIVER_MODESET
- Core machinery: ACCEL_MAJOR=261, ACCEL_MAX_MINORS=256, accel_open(), DEFINE_DRM_ACCEL_FOPS macro, accel_minors_xa xarray, /dev/accel/accelN device nodes, accel_class sysfs
- Intel ivpu driver (Meteor/Arrow/Lunar/Panther Lake): struct ivpu_device embedding drm_device, ivpu_fw_info firmware loading via request_firmware(), ivpu_cmdq/ivpu_job submission model with doorbell registers, per-context MMU SSID isolation, DRM_IOCTL_IVPU_SUBMIT
- HabanaLabs Gaudi driver: hl_device, DRM_IOCTL_HL_CS with hl_cs_chunk/hl_cs_in, staged submission flags (HL_CS_FLAGS_STAGED_SUBMISSION), TPC/MME/NIC engines, hlthunk userspace library, PyTorch Habana integration
- Qualcomm Cloud AI 100 qaic driver: MHI (Modem Host Interface) control plane + DMA Bridge Channels (DBC) data plane, struct qaic_device/qaic_drm_device, DRM_IOCTL_QAIC_CREATE_BO/ATTACH_SLICE_BO/EXECUTE_BO/WAIT_BO
- AMD XDNA driver (Linux 6.10+): AIE2 tile-mesh architecture, struct amdxdna_dev with IOMMU SVA via iommu_sva_bind_device(), SR-IOV for multi-tenant cloud, XRT (Xilinx Runtime) userspace
- ARM Ethos-U (ethosu) and Rockchip rocket drivers: ethosu targets Cortex-M embedded NPUs without GEM/PCI; rocket targets RK3588 RKNPU, uses DRM_GEM_SHMEM_HELPER + DRM_SCHED, Mesa3D rocket Gallium userspace
- Writing an accel driver: devm_drm_dev_alloc() + drm_dev_register(), per-fd context management with IOMMU SSID, GEM BO IOCTLs (create/mmap/info/wait), job submission with drm_sched_job, firmware loading via request_firmware()
- GEM memory management without display: struct ivpu_bo extending drm_gem_shmem_object with vpu_addr; struct qaic_bo with sg_table and DBC association; DMA-BUF (PRIME) for zero-copy camera-to-NPU and GPU-to-NPU interop
- DRM GPU scheduler integration: drm_sched_job/drm_sched_entity for per-context job queuing; run_job callback rings hardware doorbell; dma_fence signals completion; used by ivpu and rocket
- Security model: /dev/accel/accel0 mode 0600 with udev TAG+=uaccess; DRM_RENDER_ALLOW for IOCTLs; per-context IOMMU domain isolation (ivpu SSID, amdxdna PASID, qaic per-DBC locking); cgroup v2 device controller for container deployments
- Device node selection: /dev/dri/renderD* for GPU compute (Vulkan/OpenCL/CUDA); /dev/kfd for AMD ROCm/HIP; /dev/accel/accelN for dedicated NPU inference (OpenVINO, hlthunk, qaicrt, XRT); comparison table vs CUDA and ROCm paths
- Vendor landscape: AMD amdxdna correctly uses accel; NVIDIA absent (GPUs use DRIVER_RENDER; proprietary /dev/nvidia* for CUDA; NVDLA driver rejected); Arm split between mainlined ethosu and out-of-tree Ethos-N; 2022 LKML controversy over firmware-ABI standards resolved at Linux Plumbers Conference
- **Integrations**: Ch5 (x86 GPU Drivers / Intel i915/Xe and ivpu on Meteor Lake), Ch25 (GPU Compute / DRM render nodes, GEM, DMA-BUF, DRM scheduler), Ch48 (ROCm and ML on Linux GPUs / amdkfd vs amdxdna), Ch55 (GPU Containers and Cloud Compute / cgroup v2, CDI, Kubernetes device plugins for /dev/accel/), Ch88 (NPU and AI Accelerator Integration on Linux / OpenVINO, ONNX Runtime EP selection, userspace stack above accel), Ch108 (ROCm and HIP: AMD GPU Compute Stack / amdgpu vs amdxdna on same APU), Ch124 (Local LLM Inference on Linux GPUs / llama.cpp, Ollama, OpenVINO NPU backend device selection)


### Part IV — Mesa Architecture

### Chapter 77: Shader Source-to-ISA: The Complete Compilation Toolchain *(Part IV — Mesa Architecture)*

- Two-stage portable model: SPIR-V as the application/driver boundary IR; front ends (glslang, DXC, Slang, Tint) vs. vendor back ends (ACO, BRW, NAK); naga/wgpu as a Rust-native WGSL/SPIR-V compiler
- glslang GLSL→SPIR-V: TShader/TProgram/TIntermediate AST; GlslangToSpv() C++ API; glslangValidator CLI; -V/--target-env flags; Shader Model 5 HLSL limit vs. DXC
- DXC HLSL→SPIR-V: LLVM/Clang fork; -spirv flag; IDxcCompiler3 COM API; -fvk-use-scalar-layout/-fvk-b-shift flags; DXVK and vkd3d-Proton usage; vkd3d-shader DXIL→SPIR-V converter
- SPIRV-Tools validation and optimization: spvValidateBinary() C API; spirv-val SSA dominance/capability checks; spirv-opt passes (ADCE, CCP, loop-unroll, loop-fission, loop-fusion, --inline-entry-points-exhaustive); MESA_DEBUG=spirv_val; MESA_SPIRV_DUMP_PATH
- spirv-cross cross-compilation and reflection: CompilerGLSL/CompilerHLSL/CompilerMSL class hierarchy; ShaderResources API (uniform_buffers, push_constant_buffers, sampled_images); MoltenVK MSL target; ANGLE integration in src/compiler/translator/spirv/
- spirv_to_nir() Mesa SPIR-V→NIR translation: spirv_to_nir_options capability bits; OpDecorate→nir_variable properties; StorageClass→nir_var_shader_in/out/nir_var_mem_ssbo; OpLoad/OpStore→nir_intrinsic_load_deref/store_deref; vtn_variables.c built-in lookup table; vtn_glsl450.c OpExtInst handling
- ACO AMD compiler (RADV): 12-stage pipeline — instruction selection with divergence analysis, value numbering (CSE), optimization (VFMA3 combining), live-variable analysis, spilling, scheduling, linear-scan register allocation (SGPR/VGPR), HW lowering, s_waitcnt insertion, NOP hazard resolution, assembly emission; targets GCN/RDNA ISA
- BRW Intel EU compiler (ANV/iris): brw_compile_vs/fs/cs/gs() entry points; brw_from_nir() NIR→fs_inst/vec4_instruction IR; predicated execution BRW_OPCODE_IF/ENDIF; SEND messages for memory/texture; register regioning; vec4 backend removed in Mesa 24.x
- NAK NVIDIA Rust compiler (NVK): first Rust GPU compiler in Mesa (merged 24.0); targets Turing SM75+ (Ampere, Ada, Blackwell); emits SASS directly bypassing PTX; reverse-engineered encodings via Envytools; cbindgen C header integration; ir.rs SSA-based IR
- Mesa disk_cache API: disk_cache_create/put/get/compute_key(); SHA-1 cache_key = SHA-1(SPIR-V XOR driver UUID); XDG_CACHE_HOME/mesa_shader_cache/ default; MESA_SHADER_CACHE_DIR/MESA_SHADER_CACHE_DISABLE env vars; single-file mesa_shader_cache_db backend (Mesa 24.x)
- VkPipelineCache and Fossilize: vkCreateGraphicsPipeline() disk cache integration; Fossilize .foz archive format serializing VkShaderModule/VkDescriptorSetLayout/VkPipelineLayout/VkRenderPass/VkSampler/VkPipeline; fossilize-replay offline pre-compilation; VK_LAYER_fossilize capture layer
- shader-db regression testing: real-world GLSL/SPIR-V shader collection; instruction count/VGPR/SGPR/code-size metrics; si-report.py (RDNA/GCN) and anv-report-fossil.py (ANV); Mesa GitLab CI jobs shader-db:radv and shader-db:anv on bare-metal hardware
- Slang differentiable shading language: HLSL superset with module system (.slang import); pre-checked generics and interfaces (ILight pattern); [ForwardDifferentiable]/[BackwardDifferentiable] auto-differentiation; slangc targets (SPIR-V, DXIL, HLSL, GLSL, CUDA, Metal); used in RTX Kit (NRC, NeuralVDB), Falcor, slangtorch
- **Integrations**: Ch14 (NIR IR), Ch15 (ACO compiler), Ch16 (Mesa Vulkan common), Ch17 (llvmpipe/lavapipe), Ch18 (Vulkan drivers: RADV/ANV/NVK), Ch28 (DXVK/vkd3d-Proton), Ch61 (SPIR-V ecosystem), Ch70 (RTX Kit/neural rendering), Ch71 (Intel graphics stack), Ch76 (modern Vulkan extensions)


### Chapter 156: Mesa Nine: Direct3D 9 State Tracker *(Part IV — Mesa Architecture)*

- Mesa Nine (Gallium Nine) overview: Direct3D 9 state tracker in Mesa 10.x–25.1 (src/gallium/frontends/nine/); ~25 KLOC COM-based Gallium frontend removed in Mesa 25.2
- WineD3D vs Nine translation chains: D3D9→OpenGL→Gallium (WineD3D) vs D3D9→Gallium directly (Nine); eliminates OpenGL layer, double validation, and GLSL shader compilation pass
- NineDevice9 struct: pipe_context, pipe_context (worker thread), pipe_screen, nine_state, cso_context, nine_queue_pool ring buffer; IDirect3DDevice9 COM implementation
- NINE_STATE_* dirty bitmask system: 20 coarse state categories (FB, RASTERIZER, BLEND, DSA, VS/PS, TEXTURE, SAMPLER, VDECL, VS_CONST, PS_CONST, etc.); nine_update_state() dispatch before each draw
- D3D9→Gallium type mapping: nine_pipe.c d3d9_to_pipe_format_checked(); D3DFMT_A8R8G8B8→PIPE_FORMAT_B8G8R8A8_UNORM; BGRA ordering; rasterizer, blend, DSA state translation via cso_set_* calls
- IDirect3DStateBlock9 (stateblock9.c): Capture()/Apply() with mask-based selective copy; nine_range linked lists tracking dirty float4 register spans [bgn,end) for VS/PS constant upload
- nine_context.c command queue: worker thread + lock-free ring buffer (nine_queue_pool: 32 nine_cmdbuf slots, ~131 KB each); csmt_cmd_* structs encoded by app thread, decoded by worker calling pipe->draw_vbo()
- Shader translation (nine_shader.c): D3D9 SM1.1–SM3.0 bytecode→TGSI/NIR via ureg_program; static opcode dispatch table (_OPI macros); nine_translate_shader() entry point; vertex declaration D3DDECLUSAGE→TGSI_SEMANTIC mapping
- Constant buffer performance: nine_range dirty-range lists for SetVertexShaderConstantF; nine_buffer_upload.c sub-allocated slab pool with PIPE_MAP_PERSISTENT|PIPE_MAP_COHERENT; avoids re-uploading unchanged float4 registers
- Fixed-function pipeline emulation (nine_ff.c ~2000 lines): nine_ff_build_vs() generates TGSI VS on-the-fly for T&L (world/view/projection matrix, up to 8 D3D9 lights, texgen); nine_ff_ps_key encodes D3DTSS_* texture stage cascade (up to 8 stages, ~20 D3DTOP_* ops)
- SwapChain9 and Present(): flush command queue → MSAA resolve via pipe_context::blit() → XPresentPixmap via DRI3 → rotate backbuffers; triple-buffering via PresentWaitMSC for frame pacing
- DRI3 and DMA-BUF texture path: DRI3PixmapFromBuffers shares GEM buffer between Gallium and X server (zero-copy); DRI2 legacy fallback; XWayland adds compositing hop (+1 frame latency); render node access via DRI3Open
- Performance and deprecation: Nine 30–80% faster than WineD3D (CPU-bound D3D9); competitive with DXVK; removed Mesa 25.2 due to DXVK dominance, X11-only DRI3/Present restriction, and maintenance burden; wine-nine-standalone archived
- **Integrations**: Ch14 (Gallium3D pipe_context/pipe_screen interface), Ch15 (TGSI and NIR shader IR), Ch16 (RadeonSI Gallium driver), Ch17 (Iris Gallium driver), Ch66 (Wine/Proton D3D, WineD3D, DXVK), Ch152 (Rust GPU / wgpu)


### Part V — Mesa GPU Drivers

### Chapter 177: NVK — NVIDIA Vulkan in Mesa *(Part V — Mesa GPU Drivers)*

- Driver lineage and milestones: NVK clean-slate Mesa Vulkan driver (not a Gallium wrapper); merged Mesa 23.3; stable Vulkan 1.3 in 24.1; Vulkan 1.4 conformant in 25.0; Maxwell/Pascal/Volta enabled in 25.1; Kepler/Blackwell in 25.2
- NVKMD abstraction layer: nvkmd_ops vtable in nvkmd/nvkmd.h decouples NVK from DRM ioctls; covers bo_alloc/bo_free/bo_map, vm_bind/vm_unbind, queue_submit; one live backend (nouveau/), Nova backend reserved
- Source tree layout: src/nouveau/{compiler/nak, headers/, vulkan/, winsys/}; nvkmd/ inside vulkan/; class headers in headers/ (cl9097.h, clc597.h per generation)
- Memory architecture: GEM/TTM relationship; VRAM (DEVICE_LOCAL), BAR1/ReBAR (DEVICE_LOCAL|HOST_VISIBLE), GART coherent/cached; nvk_heap suballocator (nvk_heap.c) using util_vma_heap for driver-internal allocations; no libdrm_amdgpu equivalent
- Push buffer encoding: NVIDIA channel/subchannel model; method/data pairs; open-gpu-doc class headers (NV9097, NVC597, NVC3C0, NVC5B5); nv_push API (nv_push.h) with P_MTHD/P_IMMD macros; nv_push_dump disassembler tool
- NAK integration (NIR to SASS): nak_compile_shader() C-callable Rust FFI; nak_compiler per SM version (SM75/80/86/89/90/100/120); returns nak_shader_bin with SASS binary, num_gprs, nak_qmd; disk cache at $XDG_CACHE_HOME/mesa_shader_cache/
- GSP-RM firmware interaction: Turing+ only; NVK->NVKMD->DRM ioctls->nouveau kernel->GSP-RM RPC->GPU; pre-Turing hardware stuck at boot clocks; reclocking, display, and NVDEC video decode require GSP-RM
- WSI and Kopper: Mesa shared WSI layer (src/vulkan/wsi/); nvk_wsi.c device callbacks; DMA-BUF export via DRM_IOCTL_PRIME_HANDLE_TO_FD; zwp_linux_dmabuf_feedback_v1 format/modifier negotiation; Kopper bridges Zink/OpenGL to NVK swapchain since Mesa 25.1
- Wayland explicit sync: wp_linux_drm_syncobj_v1 protocol (Mesa 24.1); NVK exports drm_syncobj timeline point per frame; DRM_IOCTL_SYNCOBJ_EXPORT_SYNC_FILE to compositor KMS path
- Timeline semaphores and DRM_IOCTL_NOUVEAU_EXEC: drm_nouveau_exec_push[] (va+va_len), drm_nouveau_sync with DRM_NOUVEAU_SYNC_TIMELINE_SYNCOBJ; maps VkFence/binary VkSemaphore/timeline VkSemaphore directly to syncobj handles; pipeline barriers use NV9097_INVALIDATE_TEXTURE_DATA_CACHE
- Conformance table: Turing/Ampere/Ada/Blackwell/Volta/Pascal/Maxwell at Vulkan 1.4; Kepler capped at Vulkan 1.2 (lacks vulkanMemoryModel); missing VK_KHR_ray_tracing_pipeline, VK_KHR_acceleration_structure, stable VK_KHR_video_decode_queue; DXVK/VKD3D-Proton compatible from Mesa 24.1
- Building and debugging: meson -Dvulkan-drivers=nouveau; Linux 6.6+ required; NAK MSRV Rust 1.82.0; NVK_DEBUG flags (push, push_sync, zero_memory, gart); NAK_DEBUG flags (print, serial, spill); NVK_I_WANT_A_BROKEN_VULKAN_DRIVER for experimental hardware
- dEQP-VK testing: deqp-runner with per-generation caselist/skips/baseline in src/nouveau/ci/; VK_ICD_FILENAMES to select local build; freedesktop.org CI runs full suite on real hardware per merge request
- Roadmap: ray tracing stabilisation (RT Core command stream reverse engineering); Vulkan video decode promotion; Nova NVKMD backend once DRM uAPIs stable; pre-GSP reclocking via PMU firmware; Rusticl via NVK compute
- **Integrations**: Ch7 (reverse engineering NVIDIA/envytools), Ch8 (nouveau kernel driver/nvkm, DRM_IOCTL_NOUVEAU_EXEC), Ch9 (GSP-RM firmware), Ch10a (Nova Rust kernel driver), Ch10b (NVK: Building a Vulkan Driver from Scratch), Ch11 (display, reclocking, power management), Ch16 (Mesa Vulkan common infrastructure), Ch18 (Vulkan drivers: RADV, ANV comparison), Ch118 (NAK Rust shader compiler)


### Part VI-A / VI-B — Display Stack Additional Chapters

### Chapter 74: HDR and Wide Color Gamut on Linux *(Part VI-B — Display Services)*

- HDR fundamentals: OETF/EOTF/OOTF transfer functions; PQ (SMPTE ST 2084) absolute-luminance curve 0–10,000 nits; HLG (ITU-R BT.2100) scene-referred backward-compatible broadcast HDR; BT.2020/DCI-P3/sRGB wide-color-gamut primaries and colour volume
- HDR metadata standards: SMPTE ST 2086 / CTA-861-H static metadata (MaxCLL, MaxFALL, mastering primaries); HDR10+ dynamic metadata (SMPTE ST 2094-40); HDR_OUTPUT_METADATA DRM connector property blob; drm_hdmi_infoframe_set_hdr_metadata() in drm_hdmi_state_helper.c
- KMS classic color pipeline: DEGAMMA_LUT / CTM / GAMMA_LUT three-stage CRTC model; drm_crtc_enable_color_mgmt(); drm_color_lut struct; drm_color_ctm / drm_color_ctm_3x4 with S31.32 sign-magnitude fixed-point; BT.2020→BT.709 gamut conversion via drmModeAtomicCommit()
- DRM Color Pipeline API (Linux 6.19+): drm_colorop object model for per-plane pre-blending hardware transforms; COLOR_PIPELINE enumeration property on drm_plane; AMD DCN 3.x eight-stage pipeline (degamma, multiplier, 3x4 matrix, blend gamma, 3D LUT 17³, shaper LUT, inverse gamma, post-blending GAMMA_LUT); NVIDIA preview support
- Mutter HDR pipeline: GNOME 46–48 incremental delivery; hdr_output_metadata UAPI struct; libdisplay-info / di_info_get_hdr_static_metadata(); linear BT.2020 intermediate framebuffer; 203-nit SDR reference white (ITU-R BT.2408); gdctl --color-mode bt2100
- KWin HDR pipeline: Plasma 6.0–6.6 series; ICtCp-domain brightness-remapping tone mapping; per-DRM-plane COLOR_PIPELINE colorop support (Plasma 6.6, MR !6600); night-light chromatic adaptation in ICtCp; VkColorSpaceKHR / VkHdrMetadataEXT integration; DRM property table for HDR10 (HDR_OUTPUT_METADATA, Colorspace, CTM, GAMMA_LUT)
- wp_color_management_v1 Wayland protocol: merged wayland-protocols 1.41 (Feb 2025); wp_color_manager_v1 singleton; wp_image_description_v1; wp_image_description_creator_icc_v1 / wp_image_description_creator_params_v1; named TFs (st2084_pq, hlg, srgb, ext_srgb, bt1886); named primaries (bt2020, dcip3, displayp3); xx_color_manager_v4 experimental predecessor; GTK 4 GdkColorState; adoption by KWin/Mutter/SDL3/Qt6/wlroots 0.18
- color-representation-v1 Wayland protocol (wayland-protocols 1.44): wp_color_representation_surface_v1 for YCbCr-to-RGB conversion parameters on hardware-decoded video buffers; coefficients (bt2020, bt709, ictcp) and range (limited/full); complement to color-management-v1 for VA-API P010 planes
- Vulkan and HDR: VK_EXT_swapchain_colorspace — VK_COLOR_SPACE_HDR10_ST2084_EXT, VK_COLOR_SPACE_HDR10_HLG_EXT, VK_COLOR_SPACE_EXTENDED_SRGB_LINEAR_EXT; VK_EXT_hdr_metadata / VkHdrMetadataEXT; swapchain format selection VK_FORMAT_A2B10G10R10_UNORM_PACK32 / VK_FORMAT_R16G16B16A16_SFLOAT; VK_EXT_surface_maintenance1 present-mode compatibility
- Tone mapping: Reinhard / Hable filmic / ACES operators; KWin ICtCp-domain per-channel brightness remapping; Mutter linear BT.2020 SDR luminance adaptation; Gamescope --hdr-itm-enable inverse tone mapping, 3D LUT looks, mura compensation (Steam Deck OLED); scRGB / EGL_EXT_gl_colorspace_scrgb_linear intermediate representation
- HDR video passthrough: VA-API VAProfileHEVCMain10, VA_FOURCC_P010, VAHdrMetaDataHDR10; dma-buf import to DRM overlay plane (DRM_FORMAT_P010); per-plane colorop chain (degamma PQ → matrix → HDR_OUTPUT_METADATA); DisplayPort 1.4 DSC and HDMI 2.1 VRR HDR transport; Vulkan Video VK_KHR_video_decode_h265 / VK_KHR_video_decode_av1; HDMI AVI InfoFrame and DP MSA/HDRIF colour-space signalling
- **Integrations**: Ch2 (KMS Display Pipeline), Ch3 (Advanced Display Features), Ch20 (Wayland Protocol Fundamentals), Ch22 (Production Compositors), Ch26 (Hardware Video Acceleration), Ch46 (Wayland Protocol Ecosystem), Ch53 (Display Calibration), Ch63 (KTX2 and Texture Compression), Ch75 (Explicit GPU Sync), Ch101 (Color Science Theory), Ch158 (HDR Signaling Metadata)


### Chapter 75: Explicit GPU Synchronization *(Part VI-B — Display Services)*

- DMA-BUF implicit fences (dma_fence, dma_resv): struct dma_fence (one-shot pending/signalled), struct dma_resv with ww_mutex wound-wait deadlock prevention, DMA_BUF_IOCTL_EXPORT/IMPORT_SYNC_FILE conversion shim (Linux 5.19)
- drm_syncobj kernel primitive: binary drm_syncobj (DRM_IOCTL_SYNCOBJ_CREATE/WAIT/RESET/SIGNAL/DESTROY) vs. timeline drm_syncobj (64-bit seqno, DRM_IOCTL_SYNCOBJ_TIMELINE_WAIT/SIGNAL/TRANSFER/QUERY, wait-before-signal semantics)
- drm_syncobj cross-process sharing: DRM_IOCTL_SYNCOBJ_HANDLE_TO_FD / FD_TO_HANDLE for fd export/import, DRM_IOCTL_SYNCOBJ_EVENTFD (Linux 6.6) for non-blocking event-loop fence notification
- drm_syncobj internal architecture: drm_syncobj_add_point(), drm_syncobj_replace_fence(), wait_queue_head_t state transitions; sparse (timeline_point → dma_fence) mapping array
- Vulkan timeline semaphores: VkSemaphoreTypeCreateInfo / VK_SEMAPHORE_TYPE_TIMELINE (Vulkan 1.2 core), VkTimelineSemaphoreSubmitInfo, vkWaitSemaphores, vkSignalSemaphore; RADV/ANV/NVK all use shared Mesa vk_drm_syncobj.c backend
- Vulkan external semaphore export: vkGetSemaphoreFdKHR with OPAQUE_FD produces drm_syncobj fd for compositor import; SYNC_FD only valid for binary semaphores
- linux-drm-syncobj-v1 Wayland protocol (wayland-protocols 1.34, 2024): wp_linux_drm_syncobj_manager_v1, wp_linux_drm_syncobj_timeline_v1, wp_linux_drm_syncobj_surface_v1; set_acquire_point / set_release_point semantics
- Compositor explicit sync adoption timeline: Mutter (GNOME 46.1, March 2024), KWin (KDE Plasma 6.1 + NVIDIA 555.58, June 2024), wlroots 0.18 (July 2024), Sway and Hyprland; XWayland DRI3/Present XSyncFence path (April 2024)
- EGL sync objects: EGL_KHR_fence_sync (eglCreateSyncKHR, eglClientWaitSyncKHR), EGL_ANDROID_native_fence_sync (eglDupNativeFenceFDANDROID); Mesa platform_wayland.c uses both for implicit-path shim and explicit-path acquire point
- Implicit-to-explicit migration: NVIDIA flickering root cause (no implicit fences on DMA-BUFs), Linux 5.19 conversion shim, chronological 2019–2026 migration timeline, NVIDIA 555.58 end-to-end explicit sync fix
- KMS explicit sync: IN_FENCE_FD / OUT_FENCE_PTR atomic properties for GPU-side fence waits; DRM_MODE_ATOMIC_NONBLOCK with out_fence_ptr lower-latency than DRM_IOCTL_WAIT_VBLANK
- Performance: GPU-side vs. CPU-side fence waits (vkWaitSemaphores vs. pWaitSemaphores); pitfalls — premature release point signalling, unnecessary CPU flushes, multi-source fence merge via DRM_IOCTL_SYNCOBJ_TRANSFER; RCU-protected dma_resv_list GC
- io_uring unified event loop: sync_file as pollable fence (sync_file_poll, EPOLLIN on dma_fence signal); IORING_OP_POLL_ADD on sync_file; DRM_IOCTL_SYNCOBJ_EVENTFD + io_uring for non-blocking acquire-point waiting; limitations — DRM uring_cmd not yet implemented
- **Integrations**: Ch1 (DRM Architecture), Ch2 (KMS Display Pipeline), Ch4 (GPU Memory Management), Ch20 (Wayland Protocol Fundamentals), Ch21 (Building Compositors with wlroots), Ch22 (Production Compositors), Ch24 (Vulkan and EGL for Application Developers), Ch46 (Wayland Protocol Ecosystem), Ch50 (Vulkan Video), Ch74 (HDR Display), Appendix G (Synchronisation Primitives Reference)


### Chapter 151: Wayland Text Input and IME Protocols *(Part VI-A — Wayland Compositor)*

- zwp_text_input_v3 (application-side protocol): enable/disable, set_cursor_rectangle, set_surrounding_text, commit requests; preedit_string, commit_string, delete_surrounding_text, done(serial) events; serial/done double-buffering to prevent focus-race conditions
- zwp_input_method_v2 (IME-side protocol): activate/deactivate, surrounding_text, content_type, done events from compositor; commit_string, set_preedit_string, delete_surrounding_text, commit requests from IME; get_input_popup_surface and grab_keyboard requests
- zwp_virtual_keyboard_v1 (on-screen keyboards): keymap (fd), key, modifiers requests; privileged protocol requiring compositor permission to prevent keylogger abuse
- zwp_input_method_keyboard_grab_v2: IME intercepts hardware key events before the focused app; keymap, key, modifiers, repeat_info events; unconsumed keys forwarded back via zwp_virtual_keyboard_v1
- Compositor implementations: wlroots wlr_text_input_v3 / wlr_input_method_v2 structs (Sway, Hyprland, labwc); GNOME Mutter routes via D-Bus to IBus rather than zwp_input_method_v2; KDE KWin supports both fcitx5 (v2 protocol) and IBus (D-Bus)
- GTK4 text input: GdkWaylandInputContext / GdkWaylandSeat wraps zwp_text_input_v3; Pango attribute lists for preedit underline; GTK_IM_MODULE ignored on Wayland — text-input-v3 path is built into GDK directly
- Qt6 text input: QWaylandTextInputV3 in qtbase wayland plugin; QInputMethod abstract class; QWaylandTextInputMethod updates cursor rectangle, surrounding text, content type; QInputMethodEvent for preedit and commit delivery; QT_IM_MODULES ordered fallback (Qt 6.7+)
- fcitx5 Wayland architecture: WaylandIMServerV2 / WaylandIMInputContextV2 in src/frontend/waylandim/; D-Bus org.fcitx.Fcitx5; grab_keyboard on activate; evdev key+8 XKB offset; updatePreeditImpl / commitStringImpl; candidate window via zwp_input_popup_surface_v2, xdg_popup, or XWayland fallback
- IBus Wayland support: versions 1.5.26 (GNOME D-Bus only) → 1.5.29 (v1) → 1.5.32+ (v2); GNOME D-Bus bridge architecture (Mutter ↔ ibus-daemon); ibus-wayland direct v2 mode; known gaps: preedit styling, GTK4 GTK_IM_MODULE conflict, Qt6 <6.7 limitations
- XWayland XIM bridge: X11 app XIM client → XWayland built-in XIM server → zwp_text_input_v3 → compositor → IME; XMODIFIERS=@im=fcitx5/_XIM_SERVERS root property; XFilterEvent loop; XOpenIM / XCreateIC / Xutf8LookupString; four conditions required for XIM to work under XWayland
- zwp_input_popup_surface_v2 positioning model: get_input_popup_surface() assigns popup role to wl_surface; compositor delivers text_input_rectangle in global coordinates; IME renders with Cairo/Pango into wl_shm buffer; contrast with xdg_popup / xdg_positioner for client-side input panel
- Emoji and Unicode via commit_string: multi-codepoint ZWJ sequences delivered as single atomic UTF-8 string; skin-tone modifiers, flag regional-indicator pairs, keycap variation-selector sequences; HarfBuzz grapheme-cluster iteration required in text widgets; GNOME Characters uses zwp_virtual_keyboard_v1 paste simulation
- Protocol roadmap: ext_text_input_v1 stabilisation (blocking: cursor-rectangle semantics); preedit style hints (MR !234); text-input-unstable-v4 Qt proposal; GNOME Mutter alignment on zwp_input_method_v2; XIM deprecation as Xorg-free distributions ship (Ubuntu 26.04 LTS)
- **Integrations**: Ch34 (Wayland Core), Ch35 (Compositors: Mutter/KWin), Ch36 (wlroots), Ch39 (Wayland Input), Ch123 (XWayland)


### Chapter 158: HDR Display Hardware and Signaling on Linux *(Part VI-B — Display Services)*

- HDR fundamentals and standards: PQ (SMPTE ST 2084), HLG (ITU-R BT.2100), HDR10 (SMPTE ST 2086 / CTA-861-H), Dolby Vision; luminance encoding unit mismatches across hdr_metadata_infoframe, VkHdrMetadataEXT, VAHdrMetaDataHDR10
- HDMI HDR InfoFrames: AVI InfoFrame (type 0x82, BT.2020 colourimetry), DRM InfoFrame (type 0x87, SMPTE ST 2086 mastering metadata); hdmi_drm_infoframe struct; hdmi_eotf enum; drm_hdmi_infoframe_set_hdr_metadata() helper
- HDMI 2.0 SCDC and scrambling: Status and Control Data Channel over DDC pins; drm_scdc_set_scrambling(), drm_scdc_set_high_tmds_clock_ratio(); prerequisite for 4K/60Hz HDR10 via high TMDS clock
- DisplayPort VSC SDP HDR signaling: DP_SDP_VSC; DB4 pixel encoding/colorimetry, DB16 EOTF, DB17 static metadata ID, DB18-DB27 HDR primaries/luminance; amdgpu set_vsc_sdp_colorimetry(); DPCD EDP_GENERAL_CAP3 for eDP backlight HDR
- libdisplay-info EDID parsing: di_info_create_from_edid(), di_info_get_hdr_static_metadata(), di_hdr_static_metadata struct (pq/hlg/type1 flags, desired_content_max_luminance); low-level CTA extension traversal via di_edid_cta_get_data_blocks()
- CTA-861 HDR Static Metadata Data Block (Extended Tag 0x06): EOTF support bitmap (bits 0-3 = SDR/HDR/PQ/HLG), desired content max luminance encoding (50 × 2^(code/32)); kernel parse_hdr_metadata_common() in drm_edid.c populating drm_connector.hdr_sink_metadata
- OLED and MiniLED local dimming: ABM (Adaptive Backlight Modulation) DRM connector property; Intel DMC Panel Replay with Local Dimming on Meteor Lake/Lunar Lake; intel_dp_aux_backlight.c INTEL_EDP_HDR_CONTENT_LUMINANCE DPCD register; /sys/class/drm/card0-eDP-1/panel_power_savings
- drm_hdr_output_metadata UAPI structure: hdr_output_metadata and hdr_metadata_infoframe in include/uapi/drm/drm_mode.h; HDMI_EOTF_SMPTE_ST2084; CIE xy primaries scaled ×50000; min_display_mastering_luminance in 0.0001 cd/m² units; drm_connector_attach_hdr_output_metadata_property()
- Compositor-to-KMS HDR metadata flow: HDR capability discovery via libdisplay-info, constructing hdr_output_metadata blob via drmModeCreatePropertyBlob/drmModeAtomicAddProperty, atomic commit coordination with DEGAMMA_LUT/CTM/GAMMA_LUT/COLOR_PIPELINE, wp_color_management_v1 MaxCLL/MaxFALL population
- VESA DisplayHDR certification tiers (v1.2): DisplayHDR 400/500/600/1000/1400 and True Black 400/500/600; peak luminance, black level, local dimming (1D/2D/OLED pixel) and DCI-P3 gamut requirements; drmdb and edid-query tooling for compliance verification
- HDR roadmap: HDMI 2.1 FRL (Linux 7.2), DRM Color Pipeline CSC colorop (Linux 6.19+), NVIDIA per-plane Color Pipeline API, VK_EXT_hdr_metadata on Wayland (NVIDIA 580.94.11 driver), HDR10+ SMPTE ST 2094-40 dynamic metadata HF-VSIF (not yet upstream), DisplayID 2.1 capability reporting
- **Integrations**: Ch2 (KMS Display Pipeline), Ch3 (Advanced Display Features), Ch20 (Wayland Protocol Fundamentals), Ch22 (Production Compositors), Ch26 (Hardware Video Acceleration), Ch36 (Gamescope), Ch53 (Display Calibration), Ch74 (HDR and Wide Color Gamut), Ch75 (Explicit GPU Sync), Ch101 (Color Science)


### Chapter 175: Linux Compositor Accessibility — AT-SPI2, Screen Readers, and the Wayland Accessibility Gap *(Part VI-A — Wayland Compositor)*

- AT-SPI2 D-Bus architecture: separate accessibility bus via at-spi-bus-launcher; org.a11y.Bus.GetAddress; at-spi-registryd claims org.a11y.atspi.Registry; Embed/RegisterEvent flow
- Core AT-SPI2 D-Bus interfaces: org.a11y.atspi.Accessible (Name, Role, State, GetChildren), Cache (GetItems bulk transfer, AddAccessible/RemoveAccessible signals), Text, Action, Component, Table, EditableText
- GTK4 accessibility stack: GTK4 removes ATK intermediary; GtkATContext/GtkAtSpiContext (gtk/a11y/gtkatspicontext.c); GTK_ACCESSIBLE_ROLE_*, GTK_ACCESSIBLE_STATE_*, GTK_ACCESSIBLE_RELATION_LABELLED_BY; GtkAccessibleText added in GTK 4.14; AccessKit backend in GTK 4.18
- Qt6 AT-SPI2 bridge: QSpiAccessibleBridge plugin in qtbase src/gui/accessible/linux/qspiaccessiblebridge.cpp; QAccessibleInterface subclassing; QAccessible::installFactory pattern
- X11 vs Wayland accessibility gap: XGrabKey global grabs and XQueryTree cross-window inspection removed in Wayland; keyboard events only to focused surface; keyboard-shortcuts-inhibit-unstable-v1 insufficient for screen readers
- GNOME 48 / KDE 6.4 keyboard interception fix: compositor-specific D-Bus interface for Orca to register global shortcuts; hardcoded service-name authorization; not a general Wayland protocol
- xdg-desktop-portal AT.Shortcuts proposal: org.freedesktop.portal.AT.Shortcuts (issue #1046); one-time consent prompt; {sa(sa{sv})} D-Bus structure for bulk shortcut maps; wayland-protocols issue #65 for native protocol
- Orca screen reader architecture: pyatspi2/libatspi -> Orca core (Python) -> Speech Dispatcher -> eSpeak-NG/Festival/Piper; BrlAPI/BRLTTY for braille; ROLE_HEADING tree navigation via getRole()/queryText()
- GNOME Shell magnifier: magnifier.js inside compositor process; Clutter scene-graph actor cloning with affine scale transform; org.gnome.desktop.a11y.magnifier GSettings; prevents GPU power gating
- Terminal accessibility: VTE exposes screen buffer via org.a11y.atspi.Text (flat review, TextChanged signals, caret tracking); GPU-rendered terminals (Ghostty, Alacritty, WezTerm) lack widget trees; TUI semantic gap — raw text vs structured widget model
- Newton project: three-layer architecture — Wayland protocol (app pushes tree to Mutter, frame-synchronised), D-Bus protocol (Mutter to Orca, with key interception), AccessKit cross-platform abstraction; funded by GNOME STF 2024
- COSMIC compositor cosmic-atspi-unstable-v1: push-based Wayland accessibility protocol extension; AccessKit Rust AT-SPI2 adapter for non-GTK Rust apps (Alacritty, Ghostty); Newton long-term goal to replace AT-SPI2 on Wayland
- Developer testing tools: Accerciser AT-SPI2 inspector (Interface Viewer, Event Monitor plugins); dbus-monitor via org.a11y.Bus.GetAddress address; orca --debug verbose logging; pyatspi2/dogtail for automated accessible-tree testing
- **Integrations**: Ch20 (Wayland Protocol Fundamentals — security model creating accessibility gap, keyboard-shortcuts-inhibit-unstable-v1), Ch21 (wlroots compositors — no built-in screen-reader support, zwlr_input_inhibit_manager_v1), Ch22 (Mutter and KWin — GNOME 48 / KDE 6.4 D-Bus keyboard interception interfaces), Ch39 (Qt and GTK GPU Rendering — GtkATContext and QSpiAccessibleBridge within toolkit rendering pipeline), Ch132 (Wayland Security — cross-client pixel access and global keyboard grab restrictions)


### Chapter 194: Cross-Stack Integration — Protocols, Synchronisation, and the Coordination Layer *(Part VI-B — Display Services)*

- Fragmentation costs: five concrete deficiencies — pipeline latency, redundant memcpy copies, color-space blindness, implicit sync hazards, and debuggability gaps across six independent stack layers
- Wayland protocol extensions as coordination bus: wayland-protocols repository stable/staging/unstable tiers; protocol ratification as multi-party contract mechanism for buffer, sync, color, and timing contracts
- xdg-desktop-portal: D-Bus three-party coordination (app/portal/compositor) for screen capture, file picker, camera; hides compositor-specific backends (xdg-desktop-portal-gnome, xdg-desktop-portal-wlr) behind stable API
- DMA-BUF zero-copy buffer transport (Linux 3.3): gbm_bo_get_fd / drmPrimeHandleToFD / eglCreateImageKHR(EGL_LINUX_DMA_BUF_EXT); fd-based reference counting across process and subsystem boundaries
- DRM format modifiers (Linux 4.10): drm_fourcc.h DRM_FORMAT_MOD_* constants; AMD_FMT_MOD_DCC, I915_FORMAT_MOD_Yf_TILED_CCS, AFBC_FORMAT_MOD_BLOCK_SIZE_16x16; negotiation via zwp_linux_dmabuf_v1 / VK_EXT_drm_format_modifier
- GBM (libgbm): gbm_bo_create_with_modifiers2() allocates DRM GEM buffers in compositor-negotiated tiled formats; bridges format modifier negotiation to buffer allocation
- Explicit GPU sync — wp_linux_drm_syncobj_v1: DRM_IOCTL_SYNCOBJ_CREATE timeline syncobj; Mesa EGL/Vulkan WSI exports fence; protocol carries acquire/release points; landed Mesa 24.1, KWin 6.1, Mutter 46
- Color management end-to-end — wp_color_management_v1 (ratified 2025): surface-level color description (primaries, transfer function, ICC profile); compositor tonemap; KMS drm_hdr_output_metadata / DRM_PROP_DEGAMMA / DRM_PROP_CTM for display EOTF
- Frame pacing — wp_fifo_v1: set_barrier / wait_barrier primitives enforce FIFO ordered frame delivery; prevents compositor frame-skipping for video players and smooth playback applications
- Tearing control — wp_tearing_control_v1: PRESENTATION_HINT_ASYNC opt-in for minimal-latency mid-frame scanout; wp_presentation feedback_presented event with ZERO_COPY / HW_CLOCK flags and KMS vblank timestamps
- Overlay plane promotion and zero-copy scanout: KMS DRM_PLANE_TYPE_OVERLAY; GskSubsurfaceNode (GTK4) promotes wl_subsurface to KMS atomic plane via TEST_ONLY commit; ext_image_copy_capture_v1 unifies screen capture
- Mesa NIR (introduced Mesa 10.5): single SSA IR shared across all drivers; ~80 shared optimisation passes (nir_opt_algebraic, nir_opt_dead_cf, nir_lower_vars_to_ssa); backends ACO/BRW/NAK/Panfrost/Turnip all start from same NIR
- SPIR-V + spirv_to_nir() as universal entry point: GLSL/Vulkan/OpenCL/DXVK/VKD3D all funnel through SPIR-V into NIR; enables DXVK, VKD3D-Proton, and zink without per-driver porting
- Cross-stack debugging: WAYLAND_DEBUG, MESA_DEBUG, drm debugfs state; RenderDoc via VK_LAYER_RENDERDOC_Capture; drm_monitor pageflip timestamps; trace-cmd with amdgpu/i915 ftrace tracepoints for GPU scheduler correlation
- **Integrations**: Ch2 (KMS/DRM atomic API and overlay planes), Ch14 (NIR shader IR), Ch16 (Mesa Vulkan common / WSI explicit sync), Ch20 (Wayland protocol fundamentals), Ch39 (Qt/GTK GPU rendering, GskSubsurfaceNode, GdkFrameClock), Ch45 (Terminal compositor integration, DMA-BUF subsurfaces), Ch46 (Wayland protocol ecosystem and ratification process), Ch74 (HDR and wide color gamut), Ch75 (Explicit GPU synchronisation), Ch104 (DXVK and VKD3D-Proton, spirv_to_nir)


### Chapter 198: D-Bus, dbus-broker, and Modern Linux IPC: Varlink, zbus, hyprwire, and BUS1 *(Part VI-B — Display Services)*

- Linux IPC primitives: pipes, Unix domain sockets, POSIX shared memory, socketpair, memfd+splice, io_uring — throughput and latency benchmarks for each, plus which is used by D-Bus, Wayland, PipeWire, and Binder
- D-Bus wire protocol: three buses (system/session/activation), object model (bus name, object path, interface, member), message types (METHOD_CALL, METHOD_RETURN, ERROR, SIGNAL), binary type system with a{sv} dict-of-variants, XML introspection via org.freedesktop.DBus.Introspectable, UID+SO_PEERCRED security model
- dbus-daemon vs dbus-broker: dbus-broker 37 two-process architecture (broker + launcher), per-peer resource accounting (match rule limits, reply quotas, memory quotas), Linux audit integration, distribution adoption; busctl commands for list/introspect/call/monitor/capture
- sd-bus (C): sd_bus_open_system/user, sd_bus_call_method, sd_bus_match_signal, sd_bus_add_object_vtable; worked examples of org.freedesktop.login1.Session.TakeDevice and PauseDevice/ResumeDevice signal subscription
- GLib/GIO gdbus-codegen: GDBusConnection, GDBusProxy, GDBusInterfaceSkeleton; code generation from D-Bus introspection XML; async method call pattern with g_dbus_proxy_call; colord signal subscription; gdbus and busctl CLI tools
- Python D-Bus: dbus-python classic binding vs dasbus (class-based, PyGObject-backed, annotation-driven) vs python-sdbus (sd-bus FFI with asyncio); comparison table of maintenance status and API style
- zbus (Rust): pure-Rust, no libdbus dependency, async-native, runtime-agnostic; #[proxy] and #[interface] proc-macros; zvariant/serde wire format; full TakeDevice/PauseDevice example with OwnedFd and SignalStream; zbus vs the older dbus FFI crate
- Qt QDBusInterface, QDBusReply, QDBusAbstractAdaptor, qdbusxml2cpp code generation; KDE usage patterns: KWin D-Bus API, KScreen, Plasma Wayland session services
- Varlink: point-to-point JSON-over-Unix-socket IPC, NUL framing, IDL syntax; io.systemd.* production interfaces (20+ in systemd 261); varlinkctl tool; zlink Rust crate; C direct-socket implementation; architectural limitation: no broadcast signals; comparison to D-Bus for compositor seat management
- Android Binder: kernel-mediated IPC via /dev/binder, /dev/hwbinder, /dev/vndbinder; AIDL and Stable AIDL (Android 11+); ISurfaceComposer, IGraphicBufferProducer, IAllocator; libbinder_ndk C++ API; Rust binder crate; binder-trace and dumpsys SurfaceFlinger
- hyprwire and hyprtavern: Vaxry's D-Bus critique; hyprwire binary protocol (strict, Wayland-inspired); hyprtavern session bus with built-in capability permission model; hyprctl Unix socket protocol (.socket.sock command socket, .socket2.sock event socket); hyprland-rs crate
- BUS1 in-kernel IPC: KDBUS history, 2026 Rust revival by David Rheinsberg; capability-based kernel-mediated message passing without userspace daemon; potential graphics relevance for seat management, colour management, DMA-BUF fd sharing
- PipeWire native protocol: spa_pod binary type-tagged messages over Unix socket; pw_core, pw_registry, pw_node, pw_stream objects; SPA_DATA_DmaBuf zero-copy screen capture path via eglCreateImageKHR; D-Bus role limited to activation and org.freedesktop.ReserveDevice1; wireplumber Lua session policy
- eBPF and io_uring for IPC acceleration: bpftrace D-Bus tracing, BPF LSM socket-level policy enforcement, uprobe+BPF map for Varlink method counting; IORING_OP_READ_MULTISHOT for Wayland/D-Bus sockets, IORING_OP_SEND_ZC, IORING_OP_SPLICE for PipeWire DMA-BUF handoff, SQPOLL zero-syscall compositor loop; D-Bus vs Wayland protocol decision matrix; libzmq and Jupyter kernel protocol with five ZMQ sockets (shell/iopub/stdin/control/hb)
- **Integrations**: Ch21 (wlroots seat management / TakeDevice), Ch23 (xdg-desktop-portal ScreenCast/RemoteDesktop), Ch38 (PipeWire activation and ReserveDevice1), Ch53 (colord ColorManager), Ch78 (GameMode), Ch87 (ARCore Android Camera HAL / ICameraProvider), Ch111 (Flatpak graphics / portal sandbox), Ch132 (Wayland security / uaccess), Ch164 (Android NDK / libbinder_ndk)


### Part VII-A — GPU APIs and Extended Reality (Additional Chapters)

### Chapter 76: Modern Vulkan Extensions *(Part VII-A — GPU APIs)*

- Extension model taxonomy: VK_KHR_ (Khronos-ratified), VK_EXT_ (multi-vendor), VK_NV_/VK_AMD_ (single-vendor); promotion path from vendor → EXT → KHR → Vulkan core; vkEnumerateDeviceExtensionProperties + VkPhysicalDeviceFeatures2 pNext-chain query/enable pattern
- VK_KHR_dynamic_rendering (Vulkan 1.3): abolishes VkRenderPass/VkFramebuffer; VkRenderingInfo + VkRenderingAttachmentInfo; explicit vkCmdPipelineBarrier2 layout transitions; VkPipelineRenderingCreateInfo in pNext; frame-loop diagram from acquire to present
- TBDR tile-memory implications: ARM Mali, Qualcomm Adreno, Imagination PowerVR require VK_KHR_dynamic_rendering_local_read (Vulkan 1.4) for intra-pass read-back; vkCmdSetRenderingAttachmentLocationsKHR + vkCmdSetRenderingInputAttachmentIndicesKHR; VK_IMAGE_LAYOUT_RENDERING_LOCAL_READ_KHR
- Descriptor indexing / bindless (VK_EXT_descriptor_indexing, Vulkan 1.2 core): VkPhysicalDeviceDescriptorIndexingFeatures; runtimeDescriptorArray, descriptorBindingPartiallyBound, descriptorBindingUpdateAfterBind; VK_DESCRIPTOR_POOL_CREATE_UPDATE_AFTER_BIND_BIT; GL_EXT_nonuniform_qualifier + nonuniformEXT GLSL qualifier; adoption in Doom Eternal, UE5 Nanite/Lumen, Godot 4
- Mesh shaders (VK_EXT_mesh_shader): replaces vertex/tessellation/geometry pipeline; task shaders via vkCmdDrawMeshTasksEXT + EmitMeshTasksEXT; mesh shaders via SetMeshOutputsEXT; taskPayloadSharedEXT broadcast channel; two-phase Hi-Z occlusion culling with imageAtomicMin depth pyramid; indirect dispatch via VkDrawMeshTasksIndirectCommandEXT; RDNA2/3 wave32 compute mapping; NVIDIA Turing SM tensor/primitive-export
- Cooperative matrices (VK_KHR_cooperative_matrix): SPIR-V OpCooperativeMatrixMulAddKHR mapping to NVIDIA tensor cores, Intel Xe matrix engines, AMD RDNA4 AI accelerators; vkGetPhysicalDeviceCooperativeMatrixPropertiesKHR + VkCooperativeMatrixPropertiesKHR; coopMatMulAdd GLSL; RADV v_wmma_* GFX11 ISA (Mesa 23.3+); NVK NAK Rust backend (Mesa 24.0); VKD3D-Proton 3.0 FSR4 FP8 via VK_EXT_shader_float8; VK_NV_cooperative_matrix2 per-element and reduce-redistribute ops
- Shader objects (VK_EXT_shader_object): abolishes VkPipeline for graphics; VkShaderEXT compiled via vkCreateShadersEXT with VkShaderCreateInfoEXT; VK_SHADER_CODE_TYPE_SPIRV_EXT and VK_SHADER_CODE_TYPE_BINARY_EXT; vkCmdBindShadersEXT at draw time; RADV fast-link ISA caching (default Mesa 24.1); ANV deferred linking for Intel EU ISA (Mesa 25.x)
- Graphics pipeline libraries (VK_EXT_graphics_pipeline_library / GPL): four segments — Vertex Input Interface, Pre-rasterisation Shaders, Fragment Shader, Fragment Output Interface; VkPipelineLibraryCreateInfoKHR linking; graphicsPipelineLibraryFastLinking property; VK_PIPELINE_CREATE_LINK_TIME_OPTIMIZATION_BIT_EXT background LTO; adoption in DXVK 2.0 and VKD3D-Proton for shader-stutter elimination
- Extended dynamic state 3 (VK_EXT_extended_dynamic_state3 / EDS3): ~33 additional dynamic states beyond Vulkan 1.3 core; vkCmdSetPolygonModeEXT, vkCmdSetRasterizationSamplesEXT, vkCmdSetColorBlendEnableEXT/EquationEXT/WriteMaskEXT, conservative rasterisation, NVIDIA-specific states; RADV radv_dynamic_state tracking; ANV Xe-HPG 3D-state packets; implicitly required by VK_EXT_shader_object
- Driver adoption matrix (Mesa 25.x–26.x): RADV, ANV, NVK, NVIDIA proprietary, AMDVLK coverage per extension; Vulkan CTS conformance notes; AMDGPU gang-submit kernel requirement for mesh-shader task-payload correctness; Mesa 25.0 Vulkan 1.4 conformance; Mesa 26.0 VK_KHR_maintenance10 and ray-tracing improvements
- Real-world engine adoption: DXVK (GPL + descriptor indexing + shader objects); VKD3D-Proton 3.0 (cooperative matrix FSR4, mesh shaders for Alan Wake 2); Bevy 0.14 (meshlet rendering via draw_indirect, no VK_EXT_mesh_shader); Godot 4 (dynamic rendering + batched descriptor sets); Unreal Engine 5.3+ (bindless resource tables for Nanite/Lumen; mesh shaders in UE5.4+)
- Roadmap: Vulkan Roadmap 2026 Milestone (VK_KHR_fragment_shading_rate, VK_EXT_host_image_copy); VK_EXT_descriptor_heap flat CPU/GPU heap (Vulkan 1.4.340); Mesa 26.1 NIR/URB mesh-shader unification; VK_EXT_mesh_shader and VK_KHR_cooperative_matrix promotion consideration for Vulkan 1.5; VK_NV_cooperative_matrix2 FP8/INT4 type extension
- **Integrations**: Ch16 (Mesa Vulkan Common Infrastructure), Ch18 (Vulkan GPU Drivers: RADV, ANV, NVK), Ch24 (Vulkan and EGL for Application Developers), Ch25 (GPU Compute), Ch28 (Windows Compatibility: DXVK, VKD3D-Proton), Ch56 (Ray Tracing on Linux), Ch61 (SPIR-V and Shader IR), Ch70 (RTX Neural Shading), Ch71 (Intel ANV Vulkan Driver), Ch75 (Explicit GPU Synchronisation), Ch77 (Shader Toolchain: glslang, DXC, Tint, SPIRV-Cross)


### Chapter 150: EGL Architecture and DMA-BUF Integration *(Part VII-A — GPU APIs)*

- EGL core objects: EGLDisplay, EGLSurface, EGLContext; eglGetPlatformDisplay, eglChooseConfig, eglCreateWindowSurface, eglMakeCurrent initialisation sequence
- EGL platform extension model: EGL_EXT_platform_x11, EGL_EXT_platform_wayland, EGL_EXT_platform_gbm, EGL_KHR_surfaceless_context; eglGetPlatformDisplay dispatch
- GBM/DRM platform: gbm_create_device, gbm_surface_create, gbm_surface_lock_front_buffer, drmModeAddFB, drmModeSetCrtc for compositor rendering directly to KMS
- Wayland platform: wl_egl_window, eglCreatePlatformWindowSurface; eglSwapBuffers transparently posts wl_buffer via zwp_linux_dmabuf_v1
- EGLImage zero-copy sharing (EGL_KHR_image_base): eglCreateImageKHR(EGL_LINUX_DMA_BUF_EXT); glEGLImageTargetTexture2DOES; GL_TEXTURE_EXTERNAL_OES for YUV hardware conversion (NV12/YUV420)
- EGL_MESA_image_dma_buf_export: eglExportDMABUFImageQueryMESA / eglExportDMABUFImageMESA for texture-to-DMA-BUF export to KMS or V4L2 encoders
- DMA-BUF import extension chain: EGL_EXT_image_dma_buf_import (basic) + EGL_EXT_image_dma_buf_import_modifiers; eglQueryDmaBufFormatsEXT / eglQueryDmaBufModifiersEXT for format/modifier negotiation
- zwp_linux_dmabuf_v1 vs legacy wl_drm: wl_drm's insecure GEM name sharing replaced by DMA-BUF FD path; cross-GPU PRIME transfer via drm_prime_handle_to_fd/fd_to_handle
- EGL sync objects: EGL_KHR_fence_sync (eglCreateSync, eglClientWaitSync); EGL_ANDROID_native_fence_sync producing DRM sync_file FDs for explicit synchronisation; EGL_KHR_wait_sync for server-side GPU waits
- Mesa EGL internals: src/egl/{main,drivers/dri2}; dri2_initialize_wayland, dri2_swap_buffers call chain through platform_wayland.c; dri2_add_config for EGLConfig-to-format/modifier mapping
- Five WSI pipeline paths: Vulkan+Wayland WSI, OpenGL+EGL/GBM, OpenGL+GLX via XWayland, EGL headless/offscreen (EGL_EXT_platform_device), Vulkan direct KMS (VK_EXT_acquire_drm_display)
- NVIDIA EGL external platform architecture: eglexternalplatform plugin ABI (loadEGLExternalPlatform); egl-wayland (EGLStream, legacy), egl-wayland2 (DMA-BUF, current), egl-gbm, egl-x11; JSON discovery in /usr/share/egl/egl_external_platform.d/
- Practical pitfalls: EGL_NO_DISPLAY for client extension queries, multi-plane DMA-BUF (NV12/P010) attribute layout, EGL_BAD_MATCH causes, implicit sync stalls and avoidance via explicit fence FDs
- **Integrations**: Ch4 (GPU Memory Management), Ch12 (Mesa Loader), Ch20 (Wayland), Ch24 (Vulkan/EGL), Ch26 (Hardware Video), Ch39 (Qt/GTK), Ch75 (Explicit Sync), Ch139 (Hardware Planes)


### Chapter 152: Rust GPU Ecosystem: wgpu, ash, and naga *(Part VII-A — GPU APIs)*

- ash: raw zero-cost Vulkan bindings in Rust; runtime function loading via vkGetInstanceProcAddr/vkGetDeviceProcAddr; strongly typed extension loader pattern (khr::surface, khr::wayland_surface, khr::swapchain)
- gpu-allocator: TLSF sub-allocation crate pairing with ash; MemoryLocation::GpuOnly/CpuToGpu/GpuToCpu; analogous to VMA in C++
- wgpu architecture: safe WebGPU-standard Rust API over wgpu-hal backend trait (Vulkan, GLES, Metal, DX12, WebGPU/WASM); Device, Queue, Buffer, Texture, RenderPipeline abstractions
- wgpu-hal: unsafe backend abstraction trait (Api, Device, Queue, CommandEncoder); Vulkan backend in wgpu-hal/src/vulkan/ using ash; key files: instance.rs, adapter.rs, device.rs, command.rs, conv.rs
- naga: shader compiler with WGSL/GLSL/SPIR-V front-ends and typed SSA IR; back-ends emit SPIR-V, GLSL, HLSL, MSL; Arena<T> handle-indexed IR allocator; merged into wgpu monorepo 2023
- WGSL: WebGPU Shading Language; @vertex/@fragment/@compute entry points; @group/@binding resource declarations; @workgroup_size; std140/std430 alignment rules
- cuTile-rs: NVIDIA Rust library for Tensor Core tile programming; Tile<T,ROWS,COLS> generic type; #[kernel] macro; maps to PTX wgmma/wmma cooperative matrix instructions; TileGym benchmark suite
- cudarc: safe Rust wrappers for CUDA Driver API, cuBLAS, cuDNN, NVRTC; CudaContext/CudaStream/CudaSlice<T> types; load_module/load_function PTX loading; LaunchConfig for kernel dispatch
- vulkano: compile-time safe Vulkan via Rust type system; vulkano_shaders::shader! macro compiles GLSL at build time; Subbuffer<T>; task-graph RecordingCommandBuffer replacing AutoCommandBufferBuilder
- rust-gpu: compiles Rust to SPIR-V via rustc_codegen_spirv; spirv-builder in build.rs targets spirv-unknown-vulkan1.2; spirv-std provides Image2d, Sampler, arch:: intrinsics; no dyn Trait, no recursion, no heap alloc
- blade: minimal GPU library collapsing wgpu-hal indirection; single blade-graphics crate; compile-time backend selection via #[cfg]; hard resource limits (8 resources/bind group, 256 bytes plain data)
- vello: compute-based 2D vector renderer using WGSL compute shaders on wgpu; four GPU stages: coarse rasterisation, path encoding (Euler-spiral flatten), fine rasterisation, composite; Scene CPU encoding model
- burn: Rust ML framework with pluggable Backend trait; burn-wgpu (WGSL/naga), burn-cuda (cudarc/PTX), burn-ndarray (CPU); CubeCL GPU DSL compiles to WGSL or PTX; #[derive(Module)] for parameter serialisation
- encase + bytemuck: GPU buffer layout crates; encase ShaderType derive enforces std140/std430 padding; bytemuck Pod/Zeroable enables zero-copy cast_slice for #[repr(C)] vertex data; naga_oil Composer adds #import/#define_import_path and virtual function override to WGSL
- **Integrations**: Ch19 (Vulkan Architecture), Ch20 (SPIR-V), Ch22 (RADV), Ch23 (ANV), Ch25 (GPU Compute), Ch35 (Dawn/WebGPU), Ch40 (Bevy/wgpu), Ch57 (WebGPU in Chromium), Ch134 (Asahi/Apple GPU), Ch141 (Cooperative Matrices), Ch15 (ACO — AMD Shader Compiler), Ch177 (NVK — NVIDIA Vulkan)


### Chapter 154: GPU-Driven Rendering: Indirect Draw, Meshlets, and GPU Culling *(Part VII-A — GPU APIs)*

- CPU-GPU bottleneck: per-draw CPU overhead (vkCmdDrawIndexed, descriptor sets, push constants) replaced by one compute dispatch + vkCmdDrawIndexedIndirectCount per frame
- Indirect draw commands: VkDrawIndexedIndirectCommand struct layout; VkDrawIndexedIndirect and VK_KHR_draw_indirect_count (Vulkan 1.2 core drawIndirectCount feature)
- GPU frustum culling compute shader: ObjectData/DrawCommand structs; cull.comp with frustum_cull() sphere test, atomicAdd draw_count, VkMemoryBarrier2 synchronisation between cull and draw
- Two-phase occlusion culling: depth pyramid (Hi-Z) used in cull.comp to test projected bounding sphere against conservative mip-level depth sample from previous frame
- Meshlets: Meshlet struct with vertex_offset, triangle_offset, bounds_sphere, cone fields; meshoptimizer API (meshopt_buildMeshlets, meshopt_buildMeshletsBound) for cluster generation
- Task and mesh shader pipeline (VK_EXT_mesh_shader): task shader per-meshlet culling with taskPayloadSharedEXT + EmitMeshTasksEXT; mesh shader primitive output via SetMeshOutputsEXT + gl_PrimitiveTriangleIndicesEXT; vkCmdDrawMeshTasksIndirectCountEXT
- Persistent GPU scene representation: GPUScene buffer (ObjectData, MeshData, MaterialData arrays); streaming updates via staging buffer + vkCmdCopyBuffer; single shared vertex/index buffers avoiding binding changes
- Bindless resources: VK_EXT_descriptor_indexing (runtimeDescriptorArray, nonUniformIndexing, partiallyBound, updateAfterBind); VK_KHR_buffer_device_address 64-bit GPU pointers; gl_DrawID for per-object data fetch
- Hierarchical Z-Buffer (Hi-Z) pyramid: min-reduction compute shader (hiz_reduce.comp) with non-power-of-two edge handling; per-mip VkImageMemoryBarrier2; reversed-Z depth considerations
- Visibility buffer / deferred texturing: Pass 1 writes uvec2(gl_DrawID, gl_PrimitiveID) to R32G32_UINT target; Pass 2 full-screen shading with gl_BaryCoordEXT (VK_KHR_fragment_shader_barycentric) for UV interpolation via BDA scene buffers
- Cone culling in mesh shaders: meshopt_computeMeshletBounds() cone_apex/cone_axis/cone_cutoff; combined frustum + backface cone cull in task shader (task_cull.task.glsl)
- Cluster LOD hierarchy (Nanite-style DAG): self_error/parent_error per cluster; meshopt_simplify() for iterative reduction; GPU LOD selection in task shader with projected_error() screen-space threshold test
- VK_AMDX_shader_enqueue (Vulkan Workgraphs): VkExecutionGraphPipelineCreateInfoAMDX; nodePayloadAMDX/EnqueueNodePayloadsAMDX; two-node cull->draw graph replacing dispatch+barrier+indirect-draw; vkCmdDispatchGraphAMDX; RDNA3 + RADV only
- Pipeline statistics queries: VK_QUERY_TYPE_PIPELINE_STATISTICS with clipping invocations/primitives and fragment invocations; VkQueryPool creation, vkCmdBeginQuery/EndQuery, vkGetQueryPoolResults; cull efficiency metric
- **Integrations**: Ch19 (Vulkan Architecture), Ch20 (SPIR-V), Ch22 (RADV), Ch23 (ANV), Ch97 (UE5 Nanite), Ch127 (Mesh Shaders/VRS), Ch133 (Vulkan Compute Queues), Ch135 (Ray Tracing TLAS), Ch145 (GPU Profiling), Ch152 (Rust GPU), Ch157 (Descriptor Binding/Bindless/BDA)


### Chapter 157: Vulkan Descriptor Binding in Depth *(Part VII-A — GPU APIs)*

- Descriptor fundamentals: hardware records mapping UNIFORM_BUFFER, STORAGE_BUFFER, SAMPLED_IMAGE, STORAGE_IMAGE, SAMPLER, COMBINED_IMAGE_SAMPLER, ACCELERATION_STRUCTURE_KHR to GPU VAs; AMD 32-byte T# / Intel 32-byte surface state
- VkDescriptorSetLayout and VkDescriptorPool: binding slot declaration, pool sizing, VK_ERROR_OUT_OF_POOL_MEMORY, FREE_DESCRIPTOR_SET_BIT vs vkResetDescriptorPool
- Descriptor updates: vkUpdateDescriptorSets with VkWriteDescriptorSet; VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT for updating descriptors while GPU is in-use
- Push descriptors (VK_KHR_push_descriptor): vkCmdPushDescriptorSetKHR embeds descriptors directly into command buffer, eliminating pool/set lifecycle
- VK_EXT_descriptor_indexing (bindless): runtimeDescriptorArray, PARTIALLY_BOUND, VARIABLE_DESCRIPTOR_COUNT; nonuniformEXT maps to SPIR-V NonUniform decoration; ACO waterfall loop
- VK_EXT_descriptor_buffer (GPU-side): vkGetDescriptorEXT writes raw bytes to application-managed VkBuffer; vkCmdBindDescriptorBuffersEXT + vkCmdSetDescriptorBufferOffsetsEXT
- Push constants vs descriptor sets: push constants live in SGPRs (best for <=128 bytes); comparison table of all binding strategies by cost and use-case
- RADV and ANV driver implementation: RADV 32-byte T# image / 16-byte V# buffer descriptors in BO; ANV surface state heap; NonUniform handled via ACO waterfall loop
- VkDescriptorUpdateTemplate (Vulkan 1.1 core): pre-baked offset/stride mapping; vkUpdateDescriptorSetWithTemplate used by DXVK for all per-draw descriptor updates
- VK_EXT_mutable_descriptor_type: VkMutableDescriptorTypeListEXT; one slot holds SRV/UAV/CBV; used by DXVK and VKD3D-Proton to emulate D3D12 volatile descriptor heaps
- VK_EXT_inline_uniform_block (Vulkan 1.3 core): descriptorCount equals byte size; VkWriteDescriptorSetInlineUniformBlock embeds uniform data directly in pool BO
- Frame-in-flight descriptor management: N pool+set+fence slots; vkResetDescriptorPool O(1) vs FREE_DESCRIPTOR_SET_BIT free-list overhead; timeline semaphore alternative
- Null descriptors (VK_EXT_robustness2): nullDescriptor feature enables VK_NULL_HANDLE in slots with zero-return semantics; used by DXVK, VKD3D-Proton, and sparse texture streaming
- NVK descriptor implementation: nvkmd_mem GPU-visible slab + util_vma_heap sub-allocator; nvk_descriptor_writer dirty-range sync; TLAS and AS descriptors stored as 64-bit GPU VA
- **Integrations**: Ch19 (Vulkan Architecture), Ch20 (SPIR-V), Ch22 (RADV), Ch23 (ANV), Ch104 (DXVK/VKD3D-Proton), Ch16 (Mesa Vulkan Common), Ch154 (GPU-Driven Rendering), Ch135 (Vulkan Ray Tracing)


### Chapter 165: Vulkan Video Encode *(Part VII-A — GPU APIs)*

- VK_KHR_video_encode_queue architecture: encode vs decode differences — reversed input/output (VkImage src, VkBuffer dst), encoder-owned DPB/reference management, quality levels, VK_QUERY_TYPE_VIDEO_ENCODE_FEEDBACK_KHR, vkCmdEncodeVideoKHR
- Encode session setup: VkVideoEncodeCapabilitiesKHR, encodeInputPictureGranularity alignment, VK_IMAGE_USAGE_VIDEO_ENCODE_DPB_BIT_KHR / VIDEO_ENCODE_SRC_BIT_KHR, VkVideoSessionCreateInfoKHR with encode profile
- H.264 encode (VK_KHR_video_encode_h264, Dec 2023): StdVideoH264SequenceParameterSet/PictureParameterSet in VkVideoSessionParametersKHR; VkVideoEncodeH264PictureInfoKHR; IDR insertion via IdrPicFlag; B-frame and slice control
- H.265/HEVC encode (VK_KHR_video_encode_h265, Dec 2023): VPS/SPS/PPS three-parameter-set model; CTU structure up to 64x64; tile encoding with maxTiles and VK_VIDEO_ENCODE_H265_CAPABILITY_MULTIPLE_TILES_PER_SLICE_BIT_KHR; half-bitrate vs H.264
- AV1 encode (VK_KHR_video_encode_av1, Nov 2024, Vulkan 1.3.302): StdVideoAV1SequenceHeader; VkVideoEncodeAV1PredictionModeKHR (INTRA_ONLY, SINGLE_REFERENCE, UNIDIRECTIONAL/BIDIRECTIONAL_COMPOUND); referenceNameSlotIndices; OBU output; superblock sizes 64/128; per-tile quantisation
- Rate control (VkVideoEncodeRateControlInfoKHR): four modes — DEFAULT, DISABLED/CQP, CBR (virtualBufferSizeInMs VBV), VBR; per-layer temporal scalability (SVC-T) with layerCount/pLayers; AMD VCN firmware PID controller; Intel VDENC BRC PAK statistics feedback
- Mesa RADV encode: src/amd/vulkan/radv_video.c + ac_vcn_enc.c; AMDGPU_HW_IP_VCN_ENC ring; IB commands RENCODE_IB_OP_INITIALIZE/ENCODE; AV1 OBU construction in ac_vcn_enc_av1.c; H.264/H.265 default on VCN 2.x/3.x since Mesa 25.0; AV1 encode merged Mesa 25.2 (VCN 4.x)
- Mesa ANV encode: src/intel/vulkan/anv_video.c; VDENC_PIPE_MODE_SELECT, MFX_AVC_IMG_STATE, VDENC_WALKER_STATE, MFX_PAK_OBJECT; temporarily disabled in Mesa 25.3.5 pending correctness fix
- FFmpeg Vulkan encode (FFmpeg 7.1): h264_vulkan / hevc_vulkan / av1_vulkan encoders; AVVulkanDeviceContext / AVHWFramesContext; AV_PIX_FMT_VULKAN zero-copy decode-to-encode transcode; VK_QUERY_TYPE_VIDEO_ENCODE_FEEDBACK_KHR for byte-count readback
- GStreamer Vulkan encode (gst-plugins-bad): vulkah264enc (1.24+), vulkanh265enc (1.26+), vulkav1enc (1.28+); GstVulkanEncoder base class managing session lifecycle, DPB pool, command buffer submission, encode feedback; zero-copy vulkah264dec→vulkah265enc transcode
- Latency-optimised streaming encode: zero B-frames, CBR with small VBV (50–200 ms), application-triggered IDR on scene cut, VK_IMAGE_LAYOUT_PRESENT_SRC_KHR→VIDEO_ENCODE_SRC layout transition for OBS Vulkan capture
- ALVR/WiVRn wireless VR streaming: Vulkan WSI layer intercepts vkQueuePresentKHR; encode on same VkDevice as VR renderer; H.264/H.265, zero B-frames, CBR ~100–200 Mbit/s Wi-Fi 6, forced IDR every 1–2 s; 20–40 ms total encode-network-decode budget
- **Integrations**: Ch50 (Vulkan Video Decode), Ch26 (VA-API Encode), Ch57 (FFmpeg), Ch58 (GStreamer Encode Pipeline), Ch79 (Remote Display and Streaming), Ch153 (OBS Studio GPU Pipeline)


### Chapter 173: VK_EXT_shader_object: Pipeline-Free Shader Binding in Vulkan *(Part VII-A — GPU APIs)*

- VkPipeline compilation problem: combinatorial explosion of pipeline variants causes per-frame stutter; prior mitigations (VkPipelineCache, deferred creation, VK_EXT_graphics_pipeline_library) only partially address root cause
- VK_EXT_shader_object overview: introduced in Vulkan 1.3.246 (March 2023); VkShaderEXT opaque handle replaces VkPipeline; per-stage compilation collapses N*M*K variant space to N+M+K
- VkShaderCreateInfoEXT and vkCreateShadersEXT: batch creation API; VK_SHADER_CODE_TYPE_SPIRV_EXT and VK_SHADER_CODE_TYPE_BINARY_EXT; VK_SHADER_CREATE_LINK_STAGE_BIT_EXT for linked shaders
- vkCmdBindShadersEXT and binary round-trip: vkGetShaderBinaryDataEXT retrieves device-specific compiled binary; VK_INCOMPATIBLE_SHADER_BINARY_EXT success code for graceful fallback; binary keyed by pipelineCacheUUID + driver version
- Dynamic state contract: when shaderObject feature enabled, all pipeline state becomes dynamic; VK_EXT_extended_dynamic_state{1,2,3} and VK_EXT_vertex_input_dynamic_state commands available implicitly; vkCmdSetCullMode, vkCmdSetPolygonModeEXT, vkCmdSetColorBlendEquationEXT etc.
- VK_KHR_dynamic_rendering dependency: shader objects require vkCmdBeginRendering (not vkCmdBeginRenderPass); VkRenderingInfo replaces VkRenderPassBeginInfo/VkFramebuffer; VkRenderingAttachmentInfo specifies attachments inline
- Linked vs. unlinked shaders: linked shaders (VK_SHADER_CREATE_LINK_STAGE_BIT_EXT) created together enabling cross-stage NIR I/O optimisation; unlinked shaders compile fully independently for maximum mix-and-match flexibility
- RADV implementation: src/amd/vulkan/radv_shader_object.c; enabled by default in Mesa 24.1; ACO compiler backend compiles per-stage NIR; binary blob contains AMD GFX ISA (RDNA/GCN) + driver metadata
- ANV (Intel) implementation: landed in Mesa 25.3-devel; SPIR-V -> NIR -> Intel EU/Xe ISA via src/intel/compiler/; last of three major Mesa drivers to reach parity
- NVK (NVIDIA) implementation: February 2024; NAK Rust-written compiler (src/nouveau/compiler/) handles per-stage NIR->NVIDIA ISA; contributed common Mesa Vulkan runtime framework in src/vulkan/runtime/ benefiting all Mesa Vulkan drivers
- VK_LAYER_KHRONOS_shader_object software fallback: translates shader object API calls into pipeline create/bind internally; requires VK_KHR_dynamic_rendering + VK_KHR_maintenance2; auto-disables when driver natively exposes extension
- Performance guarantees: draw with shader objects <= 150% CPU time of static pipeline draws; <= 120% of maximally-dynamic pipeline draws; binary creation <= 150% of equivalent data copy cost; GPU-side delta-tracking moves from driver to application
- Migration guide: VkPhysicalDeviceShaderObjectFeaturesEXT feature detection via pNext chain; VkPipelineLayout still required via pSetLayouts in VkShaderCreateInfoEXT; ray-tracing stages explicitly excluded; mixing pipelines and shader objects per-draw permitted
- **Integrations**: Ch24 (Vulkan/EGL for Application Developers), Ch148 (Vulkan Synchronisation), Ch154 (GPU-Driven Rendering), Ch157 (Vulkan Descriptor Binding), Ch143 (RADV Internals/ACO compiler)


### Chapter 200: Vulkan Memory Allocation and Resource Management *(Part VII-A — GPU APIs)*

- `VkPhysicalDeviceMemoryProperties`: memory types (device-local, host-visible, host-coherent, host-cached, lazily-allocated) and heaps; `vkGetPhysicalDeviceMemoryProperties2`; `VK_EXT_memory_budget` heap budget/usage queries for adaptive pressure response
- `VkDeviceMemory` allocation: `vkAllocateMemory`, `VkMemoryAllocateInfo`, `VkMemoryDedicatedAllocateInfo` (image/buffer dedicated allocation avoids aliasing limitations); `maxMemoryAllocationCount` per-device limit and practical suballocation requirement
- Memory binding: `vkBindBufferMemory2` / `vkBindImageMemory2` with `VkBindBufferMemoryInfo` array; alignment requirements from `VkMemoryRequirements2`; `VkBindImagePlaneMemoryInfo` for multi-planar YCbCr images
- **VMA (Vulkan Memory Allocator)**: `VmaAllocator` creation and `VmaAllocationCreateInfo`; allocation strategies (best-fit, first-fit, SEQUENTIAL pool, TLSF pool); `vmaCreateBuffer`/`vmaCreateImage` helper APIs; `VmaAllocationInfo` for offset into `VkDeviceMemory`; defragmentation pass (`vmaBeginDefragmentation`, `VkCopyBufferToBuffer2`)
- Buffer device address (`VK_KHR_buffer_device_address`, core in Vulkan 1.2): `vkGetBufferDeviceAddress` returning a `VkDeviceAddress` GPU VA; 64-bit pointer traversal in shaders via `PhysicalStorageBuffer` storage class; capture/replay (`VK_BUFFER_CREATE_DEVICE_ADDRESS_CAPTURE_REPLAY_BIT`) for pipeline caches; intersection with DGC and descriptor-less binding
- Sparse resources: `VkSparseImageMemoryRequirements`, `VkSparseImageFormatProperties`; mip-tail packing; `vkQueueBindSparse` timeline; virtual texture streaming and megatexture use cases on RADV/ANV
- Host-image copy (`VK_EXT_host_image_copy`, promoted to Vulkan 1.4 core): `vkCopyMemoryToImageEXT` / `vkCopyImageToMemoryEXT`; `VK_IMAGE_USAGE_HOST_TRANSFER_BIT_EXT`; eliminates staging buffer round-trip for asset upload; driver layout conversion overhead
- Memory aliasing: multiple `VkBuffer`/`VkImage` objects bound to the same `VkDeviceMemory` range with non-overlapping lifetimes; `VK_IMAGE_CREATE_ALIAS_BIT`; transient attachment aliasing for tile-based GPU memory savings; hazard requirements
- Residency and device loss: `VK_ERROR_OUT_OF_DEVICE_MEMORY` vs `VK_ERROR_OUT_OF_HOST_MEMORY`; heap eviction under pressure; `VK_KHR_present_wait` for signalling idle frames; `VK_EXT_device_fault` correlation
- RADV/ANV/NVK allocator internals: how each driver partitions its heap (RADV `radv_alloc_memory`, ANV `anv_device_alloc_bo`); GEM handle to `VkDeviceMemory` mapping; BAR window constraints on discrete GPUs
- **Integrations**: Ch24 (Vulkan/EGL — memory allocation basics), Ch106 (Vulkan Memory Model — visibility/ordering), Ch154 (GPU-driven rendering — large indirect buffer management), Ch157 (descriptor buffers — GPU-visible heap allocation), Ch82 (VMA in the Vulkan ecosystem toolkit)


### Chapter 201: Vulkan Debugging, Validation, and Profiling *(Part VII-A — GPU APIs)*

- `VK_LAYER_KHRONOS_validation`: activation via `VK_INSTANCE_LAYERS`; `VkLayerSettingsCreateInfoEXT` for fine-grained sub-feature control; sub-layers: core validation (object lifetime, handle validity), thread safety, stateless parameter, best practices, synchronization validation, GPU-Assisted Validation, debug printf
- Best Practices sub-layer: `VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT` warnings for pipeline misuse (creating pipelines inside render passes, redundant clears, unnecessary layout transitions, non-optimal image layouts); vendor-specific guidance surfaced per RADV/ANV/NVK driver
- Synchronization Validation (`VK_VALIDATION_FEATURE_ENABLE_SYNCHRONIZATION_VALIDATION_EXT`): detects read-after-write and write-after-write hazards at command buffer level; reports missing `VkImageMemoryBarrier2` and incorrect `srcStageMask`/`dstStageMask`; complements but does not replace Ch148's reference
- GPU-Assisted Validation: instruments SPIR-V at vkCreateShaderModule time with bounds-check opcodes; requires `VK_KHR_buffer_device_address`; catches out-of-bounds descriptor indexing, uninitialized descriptor reads; overhead ~5–30% GPU time
- **`VK_EXT_debug_utils`**: `vkSetDebugUtilsObjectNameEXT` names every `VkImage`/`VkBuffer`/`VkPipeline` for readable validation output and profiler labels; `vkCmdBeginDebugUtilsLabelEXT`/`vkCmdEndDebugUtilsLabelEXT` for GPU breadcrumbs; `vkCreateDebugUtilsMessengerEXT` callback with `messageSeverity` × `messageType` filter matrix
- **`VK_EXT_device_fault`**: `vkGetDeviceFaultInfoEXT` post-hang diagnostic; `VkDeviceFaultVendorInfoEXT` array with fault type, index, and VA; GPU crash breadcrumbs via debug labels; AMD-specific UVM fault register decoding; use with `VK_EXT_device_address_binding_report` for VA identification
- **RenderDoc** (open-source frame capture): overlay injection vs explicit `vkStartFrameCapture`/`vkEndFrameCapture` API; event list with every draw/dispatch/copy; resource inspector (live texture preview, buffer hex, descriptor set contents); pipeline state inspector (vertex input, VS/PS source, blend state); GPU counter integration (ARM, AMD RGP bridge, NVIDIA Nsight SDK); remote capture via `renderdoccmd remoteserver`; source at `renderdoc/driver/vulkan/vk_core.cpp`
- **AMD Radeon GPU Profiler (RGP)**: captures GPU timeline via SQTT (Shader Queue Thread Trace); wave occupancy and SIMD utilization per pipeline stage; barrier stall visualization; `VK_AMD_shader_info` counter queries; `.rgp` file format for offline analysis
- **Intel Graphics Performance Analyzer (GPA)**: Xe EU thread occupancy, L3 cache hit rate, sampler throughput; frame analyzer event-level GPU metrics; Xe2 PMU counter access
- **NVIDIA Nsight Graphics**: GLSL/SPIRV shader debugger with source-line stepping on RTX hardware; range profiler with NV-specific counter sets; memory access pattern heat maps; Shader Profiler with per-ISA instruction throughput; `VK_NV_device_diagnostics_config` for GPU crash dump
- `VK_KHR_performance_query`: hardware PMU counter enumeration via `vkEnumeratePhysicalDeviceQueueFamilyPerformanceQueryCountersKHR`; performance query pools; per-vendor counter semantics (AMD, Intel, ARM); interaction with RenderDoc's counter capture
- `vkconfig` (Vulkan Configurator): GUI for enabling layers, toggling validation features, loading override layer JSON; part of Vulkan SDK; `VkConfig` layer settings file persisted to `~/.local/share/vulkan/settings.d/`
- Debugging workflow: layer activation → messenger callback → RenderDoc frame capture → vendor GPU profiler → `VK_EXT_device_fault` on hang; common false-positive patterns in synchronization validation
- **Integrations**: Ch24 (Vulkan application foundation — initial device setup and layer activation), Ch148 (Vulkan Synchronisation — what sync validation catches), Ch157 (descriptor binding — GPU-Assisted Validation catches descriptor indexing bugs), Ch173 (shader objects debugging workflow), Ch125 (RenderDoc on Linux in depth), Ch137 (GPU performance profiling — RGP, GPA, Nsight overview), Ch30 (general debugging tooling)


### Chapter 202: Vulkan WSI Deep Dive *(Part VII-A — GPU APIs)*

- WSI abstraction overview: `VK_KHR_surface` as the platform-agnostic surface type; `vkGetPhysicalDeviceSurfaceCapabilitiesKHR`, `vkGetPhysicalDeviceSurfaceFormatsKHR`, `vkGetPhysicalDeviceSurfacePresentModesKHR`; surface querying before swapchain creation
- **Wayland WSI path**: `VK_KHR_wayland_surface`; `VkWaylandSurfaceCreateInfoKHR` wrapping `wl_display`/`wl_surface`; Mesa `wsi_common_wayland.c` implements the swapchain; linux-dmabuf modifier negotiation via `zwp_linux_dmabuf_v1` feedback; DMA-BUF export → compositor import → `drmModeAtomicCommit`; explicit sync via `wp_linux_drm_syncobj_v1` on acquire/present fences
- **X11/XCB WSI path**: `VK_KHR_xcb_surface` and `VK_KHR_xlib_surface`; DRI3 Present extension (`xcb_dri3_pixmap_from_buffer`, `xcb_present_pixmap`); X11 redirect vs direct present mode; Glamor interaction; HiDPI and scale factor under XWayland
- **Present modes in depth**: `VK_PRESENT_MODE_FIFO_KHR` (vsync; queued; no tearing), `VK_PRESENT_MODE_FIFO_RELAXED_KHR` (allows late tear once), `VK_PRESENT_MODE_MAILBOX_KHR` (replaces queued image; low-latency triple-buffer), `VK_PRESENT_MODE_IMMEDIATE_KHR` (no sync; tears); per-driver support matrix on RADV, ANV, NVK; latency vs tearing vs power trade-offs
- Swapchain creation: `VkSwapchainCreateInfoKHR` — `minImageCount`, `imageFormat`, `imageColorSpace`, `imageExtent`, `compositeAlpha` (`VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR` vs pre-multiplied), `preTransform`, `presentMode`; `VK_SWAPCHAIN_CREATE_MUTABLE_FORMAT_BIT_KHR` for format view reinterpretation; old swapchain recycling
- Image acquisition and presentation: `vkAcquireNextImage2KHR` with `VkAcquireNextImageInfoKHR` (device group `deviceMask`); `vkQueuePresentKHR` with `VkPresentInfoKHR`; `VkSwapchainPresentFenceInfoEXT` per-image acquire fence; present id tracking with `VK_KHR_present_id` / `VK_KHR_present_wait`
- **`VK_EXT_swapchain_maintenance1`**: `VkSwapchainPresentModeInfoEXT` per-present mode switching without recreation; `VkSwapchainPresentScalingCreateInfoEXT`; deferred swapchain destruction via `VkSwapchainPresentFenceInfoEXT` + `VkReleaseSwapchainImagesInfoEXT`; safe recreation without `vkDeviceWaitIdle`
- **`VK_EXT_present_timing`** (merged Mesa 26.1 for RADV/ANV/NVK/PanVK/Turnip): `VkPresentTimingInfoGOOGLE` desired present time; presentation timestamp feedback via `vkGetSwapchainTimedPropertiesEXT`; sub-frame latency targeting; per-WSI backend implementation in Mesa's `wsi_common_present_timing.c`
- **Direct display**: `VK_KHR_display` — `vkGetPhysicalDeviceDisplayPropertiesKHR`, `vkCreateDisplayPlaneSurfaceKHR`; kiosk/signage use cases; `VK_EXT_acquire_drm_display` — `vkAcquireDrmDisplayEXT` with DRM fd or `DRM_IOCTL_MODE_CREATE_LEASE`; Mesa `wsi_common_drm.c`
- **Headless WSI**: `VK_EXT_headless_surface`; `VkHeadlessSurfaceCreateInfoEXT`; Mesa headless WSI implementation for offscreen rendering, CI, and server workloads; relationship to `EGL_EXT_platform_device` and GBM headless
- HDR swapchain: `VkSwapchainCreateInfoKHR` with `VK_COLOR_SPACE_HDR10_ST2084_EXT`, `VK_COLOR_SPACE_EXTENDED_SRGB_LINEAR_EXT` (scRGB), `VK_COLOR_SPACE_BT2020_LINEAR_EXT`; wide-gamut surface workflow; compositor-side `wp_color_management_v1` surface image description coordination
- Mesa WSI common layer architecture: `src/vulkan/wsi/wsi_common.c` shared infrastructure; per-backend files `wsi_common_wayland.c`, `wsi_common_x11.c`, `wsi_common_drm.c`; swapchain image allocation via GBM (`gbm_bo_create_with_modifiers2`); modifier selection negotiation with compositor or display engine
- **Integrations**: Ch24 (Vulkan/EGL — WSI basics and linux-dmabuf introduction), Ch150 (EGL Architecture and DMA-BUF Interop — the EGL side of the same presentation path), Ch20 (Wayland protocol fundamentals — `wl_surface` and `zwp_linux_dmabuf_v1`), Ch74 (HDR and WCG — HDR swapchain color space), Ch75 (Explicit GPU Sync — `wp_linux_drm_syncobj_v1` integration with WSI acquire/present fences), Ch112 (VRR and frame pacing — `VK_EXT_present_timing` and FIFO strategies)


### Part VIII — Gaming Layer

### Chapter 78: Gamescope and the Steam Deck: A Complete Gaming Graphics Stack *(Part VIII — Gaming Layer)*

- Van Gogh APU (AMD Custom GPU 0405): TSMC 7nm, 4x Zen 2 cores + 8 RDNA 2 CUs, DCN 3.0 display engine, 16 GB LPDDR5 unified memory; zero-copy DMA-BUF framebuffer path via TTM
- SteamOS 3 architecture: immutable Arch Linux root, A/B OTA partitions, Desktop Mode (KDE Plasma/KWin on Wayland) vs Game Mode (Gamescope as session compositor with Steam Big Picture)
- Gamescope session architecture: wlserver_init(), embedded XWayland, custom Wayland extensions (gamescope_action_binding, wp_linux_drm_syncobj_v1, gamescope_input_method), DRM/SDL/Wayland/headless backend abstraction
- steamcompmgr and libliftoff plane assignment: per-vblank decision to direct-scanout via drmModeAtomicCommit() or Vulkan compute composite; DRM_MODE_ATOMIC_TEST_ONLY hardware plane tests
- DCN 3.0 hardware planes: DPP (Display Pipe and Plane) per-plane CSC/tone-mapping, MPC (Multiple Pipe/Plane Combined) alpha blending, VSTARTUP/VUPDATE/VREADY atomic sync signals
- FSR upscaling pipeline: FSR 1 two-pass EASU/RCAS spatial compute shaders, FSR 2 temporal with motion vectors, NIS (NVIDIA Image Scaling) spatial fallback, integer scaling for pixel art
- VRR and frame pacing: VRR_ENABLED CRTC property, CVBlankTimer with timerfd_settime() rolling draw-time prediction, 40 fps/40 Hz battery sweet spot, DRM_MODE_PAGE_FLIP_ASYNC immediate flips
- HDR pipeline: VK_COLOR_SPACE_HDR10_ST2084_EXT Vulkan swapchain, HDR_OUTPUT_METADATA KMS property, inverse tone mapping (ITM) for SDR content, frog_color_management_v1 Wayland protocol for HDR games via Proton
- Mura compensation (SteamOS 3.6): per-pixel 2D LUT in VkImage applied as Vulkan compute pass after upscaling; factory-calibrated per-panel luminance non-uniformity correction for OLED
- MangoHUD Vulkan implicit layer: intercepts vkQueuePresentKHR and vkGetDeviceProcAddr; AMDGPU sysfs metrics (gpu_busy_percent, mem_info_vram_used, hwmon temps, AMD RAPL power); LD_PRELOAD for OpenGL via glXSwapBuffers/eglSwapBuffers
- mangoapp architecture: separate Wayland client rendering overlay at native panel resolution as wl_surface; metrics delivered from MangoHUD in-process via shared memory/socket to avoid upscaling of overlay text
- Input-to-photon latency pipeline: USB/BT HID -> evdev/libinput -> SDL2 -> XWayland -> Gamescope -> KMS flip -> OLED; SCHED_FIFO real-time scheduling via CAP_SYS_NICE; Wayland latency marker extension for flip timing
- Docking and external display: DisplayPort Alt Mode over USB-C, drmModeGetResources() hotplug re-enumeration, EDID-driven mode negotiation, HDR_OUTPUT_METADATA on external HDR monitors, PipeWire for HDMI/DP audio switching
- **Integrations**: Ch2 (KMS Display Pipeline), Ch3 (Advanced Display Features: VRR/HDR), Ch5 (AMDGPU driver: TTM, sysfs power metrics), Ch21 (wlroots compositor patterns), Ch22 (Production Compositors), Ch24 (Vulkan Swapchains and WSI), Ch25 (Vulkan Compute Shaders), Ch28 (Windows Compatibility: Proton/DXVK), Ch29 (Upscaling/Effects/Overlays), Ch30 (Debugging and RenderDoc), Ch72 (AMD FidelityFX/FSR), Ch74 (HDR on Linux), Ch75 (Explicit GPU Synchronisation)


### Chapter 167: NTSYNC: NT Synchronization Primitives in the Linux Kernel *(Part VIII — Gaming Layer)*

- NT synchronization problem: NtWaitForMultipleObjects wait-all atomicity — vectored, multi-typed, state-mutating atomic wait with no POSIX equivalent; wineserver RPC bottleneck for multi-threaded D3D12 games
- esync (eventfd-based sync): EFD_SEMAPHORE eventfd per object, epoll for multi-wait, WINEESYNC=1; fd exhaustion requiring DefaultLimitNOFILE=1048576; no true atomic wait-all semantics
- fsync (futex-based sync): shared-memory futexes, out-of-tree FUTEX_WAIT_MULTIPLE; futex_waitv() merged Linux 5.16 enabling wait-any without custom kernel; still no atomic wait-all
- Out-of-tree patchwork ecosystem: wine-staging, Frogging-Family wine-tkg-git, GE-Proton, winesync DKMS module (kernels 6.6–6.13 vs upstream ntsync API)
- ntsync kernel module (Linux 6.14): /dev/ntsync misc char device; per-open-fd instance isolation; ntsync_device with wait_all_lock, ntsync_obj with per-object spinlock and all_hint atomic
- UAPI object types: NTSYNC_TYPE_SEM (count/max), NTSYNC_TYPE_MUTEX (owner TID/recursion count, EOWNERDEAD abandoned detection), NTSYNC_TYPE_EVENT (signaled/manual flags for auto-reset vs manual-reset)
- UAPI header include/uapi/linux/ntsync.h: ntsync_sem_args, ntsync_mutex_args, ntsync_event_args, ntsync_wait_args; ioctls NTSYNC_IOC_CREATE_SEM/MUTEX/EVENT, NTSYNC_IOC_WAIT_ANY/ALL, NTSYNC_IOC_PULSE_EVENT, NTSYNC_IOC_KILL_OWNER; NTSYNC_MAX_WAIT_COUNT=64
- NTSYNC_IOC_WAIT_ALL atomicity: device-wide wait_all_lock mutex; try_wake_all() acquires all per-object spinlocks under device mutex; atomic_try_cmpxchg(); all_hint counter optimizes wait-any fast path (per-object spinlock only when all_hint==0)
- NtPulseEvent correctness: walks waiter list under per-object spinlock at signal time, wakes only already-sleeping threads, resets state atomically — impossible in esync/fsync userspace
- Wine integration: inproc_sync struct with per-object fd; dlls/ntdll/unix/sync.c linux_wait_objs(); Wine 11.0 (Jan 2026) auto-detects /dev/ntsync; server-bound objects handed fd via wineserver request; WINEESYNC/WINEFSYNC env vars for legacy paths
- Proton integration: GE-Proton 10-9 first shipped ntsync, GE-Proton 10-10 enables by default; PROTON_USE_NTSYNC=1 launch option; Steam Deck SteamOS kernel lag behind mainline; ls /dev/ntsync to verify availability
- Performance benchmarks (ntsync v6 cover letter): Dirt 3 +678% FPS vs vanilla Wine wineserver; Resident Evil 2 +196%; fsync-to-ntsync gains typically 10–40% FPS with greater frame-time consistency; eliminates stutter from wait-all retry-loop races
- CONFIG_NTSYNC Kconfig (drivers/misc): tristate module or built-in; udev rule MODE=0666 for non-root access; distro support: Fedora 42, Arch linux/linux-zen, Ubuntu 25.04, Debian Forky/Sid; kernel selftests tools/testing/selftests/drivers/ntsync/
- Security model: cross-instance fd yields EINVAL; POSIX fd lifetime/refcount cleanup; Flatpak sandbox needs explicit /dev/ntsync device access; trust model analogous to memfd_create, userfaultfd, eventfd — software-only state, no hardware/DMA access
- **Integrations**: Ch28 (Wine and Proton: wineserver architecture, NT emulation, DXVK, Proton stack), Ch104 (DXVK and VKD3D-Proton: D3D12 fence/ID3D12Fence to Vulkan timeline semaphore translation, command queue CPU-side NT event signaling), Ch171 (Linux Anti-Cheat: fd enumeration of /dev/ntsync fds, anti-cheat compatibility with ntsync-dispatched synchronization)


### Chapter 171: Linux Gaming Anti-Cheat — EasyAntiCheat, BattlEye, and the Ring-0 Problem *(Part VIII — Gaming Layer)*

- Anti-cheat ecosystem overview: EAC (Epic/EOS), BattlEye, Riot Vanguard, XIGNCODE3, nProtect GameGuard, and VAC — their Linux support status as of mid-2026
- Windows kernel-mode driver model: KMDF/WDFDriver at ring 0, ObRegisterCallbacks for handle stripping, PsSetLoadImageNotifyRoutine, PatchGuard, EV certificate signing, TPM 2.0 measured-boot attestation
- Why ring-0 anti-cheat cannot work on Wine: no ntoskrnl.exe, no ring-0 kernel object manager, Linux ptrace/proc/PID/mem openness vs. Windows PPL protected-process model, absence of CONFIG_MODULE_SIG_FORCE enforcement
- EasyAntiCheat Linux architecture: native ELF easyanticheat_x64.so from EOS SDK, two generations (Kamu v1 standalone vs. EOS v2), EOS Developer Portal opt-in, depot deployment steps
- EAC compatibility case studies: Apex Legends EOS migration breaking Linux (November 2024), EA 30% cheat reduction claim, Fortnite active block, Halo/Dead by Daylight working
- BattlEye Linux support: native BattlEye.so, PROTON_BATTLEYE_RUNTIME env var, email-based server-side opt-in with no SDK/depot changes required; GTA V case (BattlEye added September 2024, Linux not enabled)
- Steam Linux Runtime containers: pressure-vessel with scout/soldier/sniper rootfs generations; Proton EasyAntiCheat Runtime and Proton BattlEye Runtime as separate Steam tools (App IDs 1826330 and 1161040); PROTON_EAC_RUNTIME / PROTON_BATTLEYE_RUNTIME env vars
- Proton version and anti-cheat compatibility: EAC bootstrapper v1.6.0 regression under Proton 8.x (Proton issue #6740); ntsync (Linux 6.14) anti-cheat testing
- Riot Vanguard: vgk.sys kernel driver, boot-state attestation via UEFI Secure Boot + TPM, kernel module verification failure on Linux; XIGNCODE3 and nProtect GameGuard as rootkit-style ring-0 systems incompatible with Wine
- VAC as user-space counter-example: in-process memory scanning + server-side VACnet ML behavioural analysis; fully Linux-compatible without modification
- Remaining user-space gap: process_vm_readv/proc/PID/mem interception impossible without ObRegisterCallbacks; PR_SET_DUMPABLE as limited mitigation; /proc/modules hideable by kernel-level cheats
- Linux IMA and hypothetical kernel anti-cheat: IMA TPM PCR extension, CONFIG_MODULE_SIG_FORCE, UEFI Secure Boot chain — theoretically viable but practically incompatible with consumer Linux and DKMS GPU drivers
- Server-side mitigation as the viable Linux defence: movement/damage validation, VACnet-style ML behavioural telemetry — platform-agnostic and the recommended architecture for new Linux-friendly titles
- Developer guidance: EOS portal Linux toggle, libeasyanticheat.so rename/deploy procedure, PROTON_LOG=1 / WINEDEBUG=+loaddll debugging, EAC log paths in Wine prefix, Steam Deck Verified/Playable criteria
- **Integrations**: Ch28 (Wine/Proton compatibility layer), Ch78 (Gamescope/Steam Deck compositor), Ch80 (GPU security and Linux process model), Ch104 (DXVK and VKD3D-Proton), Ch167 (NTSYNC/ntsync kernel driver)


### Part IX — Tooling & Contributing

### Chapter 79: Remote Display, Screen Casting, and GPU-Accelerated Game Streaming *(Part IX — Tooling and Contributing)*

- Remote graphics landscape: three categories — screen casting (PipeWire + DMA-BUF), remote desktop (RDP/VNC, VA-API encode), and game streaming (Sunshine/Moonlight, sub-20 ms latency, RTSP+RTP)
- PipeWire pw_stream API: struct spa_video_info_raw, SPA_PARAM_EnumFormat/Buffers negotiation, SPA_DATA_DmaBuf PROCESS callback, pw_stream_connect with PW_STREAM_FLAG_MAP_BUFFERS
- Zero-copy DMA-BUF pipeline: GBM alloc → DMA-BUF fd → PipeWire shared-memory metadata → EGL_LINUX_DMA_BUF_EXT / VkImportMemoryFdInfoKHR / vaCreateSurfaces VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2
- xdg-desktop-portal ScreenCast: org.freedesktop.portal.ScreenCast D-Bus interface (CreateSession, SelectSources, Start, OpenPipeWireRemote), portal backends (xdg-desktop-portal-gnome/wlr/kde), pw_context_connect_fd()
- FreeRDP and GNOME Remote Desktop: libfreerdp-server3, RDPGFX AVC444/AVC420, VA-API H.264 encode, libei for Wayland input emulation replacing XTest, Mutter Remote Desktop D-Bus API
- OBS Studio PipeWire plugin: plugins/linux-pipewire/ DMA-BUF EGL import, VA-API/NVENC/AMF encode, OBS 31 explicit sync via SPA_META_SyncTimeline + drm_syncobj_wait() eliminating glFinish() stalls
- Sunshine game streaming server: capture backend priority (KMS DRM_IOCTL_MODE_GETFB2 → wlr-screencopy-unstable-v1 → portal → NvFBC → XShm), encode via VA-API VAEntrypointEncSliceLP, CUDA+NVENC, or Vulkan Video (h264/hevc/av1_vulkan)
- Sunshine session handshake: four phases — AES-GCM pairing, RTSP/JSON serverinfo capability negotiation, RTSP/SDP stream setup, UDP streaming (video 47998, control 47999, audio 48000) with Reference Frame Invalidation (RFI)
- NvFBC (NVIDIA Framebuffer Capture): NvFBCCreateInstance() function-pointer table, NVFBC_CAPTURE_TO_SYS (~1.8 ms) vs NVFBC_CAPTURE_SHARED_CUDA (~50 µs CUdeviceptr), requires NVIDIA driver ≥ 515.57, no Wayland support
- Moonlight client: FFmpegVideoDecoder with avcodec_send_packet/avcodec_receive_frame, Pacer subsystem, VA-API (Intel/AMD) / NVDEC (NVIDIA Wayland) / VDPAU (NVIDIA X11) renderer hierarchy, RTCP-driven adaptive bitrate
- Virtual displays and headless GPU: VKMS (CONFIG_DRM_VKMS, kernel v4.19, Configfs), nvidia-drm modeset=1 + EDID injection (drm.edid_firmware), virtio-gpu paravirtualized KMS, VFIO GPU passthrough, Sway/Cage headless Wayland
- Latency analysis: full frame chain GPU render → KMS capture (1–3 ms) → RGB→NV12 color convert (0.1–0.5 ms) → NVENC/VA-API encode (1–3 ms) → UDP → decode (1–3 ms) → present; 8–25 ms GPU-accelerated vs 25–60 ms software; measurement via pw-top, perf+eBPF on drm_vblank_event/vaBeginPicture/vaEndPicture
- **Integrations**: Ch2 (KMS Display Pipeline), Ch20 (Wayland Protocol Fundamentals), Ch22 (Production Compositors), Ch23 (Legacy and Sandboxed App Support), Ch26 (Hardware Video Acceleration VA-API/VDPAU), Ch38 (PipeWire), Ch55 (GPU Containers and Cloud), Ch57 (FFmpeg), Ch58 (GStreamer), Ch60b (Streaming Protocols), Ch66 (CUDA), Ch74 (HDR on Linux)


### Chapter 80: GPU Security: Isolation, Content Protection, and Confidential Computing *(Part IX — Tooling and Contributing)*

- GPU attack surface overview: process isolation failures, DMA attacks, content protection bypass, firmware supply-chain trust, side-channel attacks, confidential computing, and driver hardening
- GPU process isolation via DRM_GPUVM framework: per-process GPUVA spaces, drm_gem_object gpuva list, context teardown sequence, and VRAM scrubbing on process exit
- AMDGPU cleaner shader (CDNA2/GFX9.4.2+, GFX11.5): clears LDS, VGPRs, SGPRs after each job; sysfs enforce_isolation and run_cleaner_shader interfaces
- IOMMU and DMA attack mitigation: Intel VT-d, AMD-Vi, ARM SMMU; iommu_paging_domain_alloc_flags(), IOMMU groups, ACS (Access Control Services), VFIO for GPU passthrough
- HDCP 2.2 content protection: AKE/LC/SKE three-phase protocol; HDCP_STREAM_TYPE0/TYPE1; drm_connector_attach_content_protection_property(); DRM_MODE_CONTENT_PROTECTION_DESIRED/ENABLED; Intel GSC/MEI implementation; CVE-2024-53050
- Secure display path: Intel PXP (Protected Xe Path) on Gen12+ for encrypted GPU-to-panel pipeline; Wayland zwp_linux_content_type_manager_v1 / zwp_content_type_v1 protocol extension
- GPU firmware signing and secure boot: NVIDIA nvkm_firmware_get() with PKC signatures, ACR/Falcon HS/LS/WPR model; Intel GuC/HuC via intel_uc_fw_fetch() with CSS headers; AMD PSP (ARMv7 Cortex-A5) with ARK->ASK->CEK signing chain; CONFIG_MODULE_SIG_FORCE and MOK enrollment
- AMD SEV-SNP and GPU confidential computing: encrypted VM DMA buffers (C=0 pages), HSMP and SMI Transfer interfaces, ongoing AMD GPU trust-domain extension work
- NVIDIA H100 confidential computing: CC-Off/CC-On/CC-DevTools modes; CPR (Compute Protected Region) in HBM2e; AES-GCM encrypted bounce buffers; SPDM attestation with ECC-384 device key; NRAS and nvtrust open-source tooling
- GPU side-channel attacks: cache Prime+Probe/Flush+Reload timing attacks; GPUHammer Rowhammer on GDDR6 (USENIX Security 2025, NVIDIA A6000, AI model degradation); power side channels via nvmlDeviceGetPowerUsage() / rsmi_dev_power_ave_get(); mitigations: perf_event_paranoid, amdgpu_perfcnt_ioctl, OA counters, ECC on A100/H100/MI300
- DRM ioctl permission model: DRM_AUTH, DRM_MASTER, DRM_ROOT_ONLY, DRM_RENDER_ALLOW flags; render nodes (/dev/dri/renderD*) vs primary nodes (/dev/dri/card*); drm_ioctl() dispatch in drm_ioctl.c; syzkaller fuzzing; amdgpu CVE history in VCN and KFD paths
- eBPF for GPU access control: BPF LSM lsm/file_open hook (kernel 5.7+) for cgroup-based render-node access control; seccomp-BPF to allowlist DRM ioctl numbers and block modesetting range (0xA0+); DMA-BUF export tracing via dma_buf:dma_buf_fd tracepoint; kprobe-based crypto-jacking detection on amdgpu_cs_ioctl; audit trail via kprobes on drm_mode_setcrtc and drm_mode_addfb2
- **Integrations**: Ch1 (DRM Architecture), Ch4 (GPU Memory Management), Ch5 (x86 GPU Drivers / i915/Xe), Ch9 (GSP-RM Firmware), Ch22 (Production Compositors), Ch25 (GPU Compute), Ch30 (Debugging and Profiling), Ch32 (Contributing), Ch55 (GPU Containers and Cloud), Ch66 (CUDA), Ch71 (Intel Xe), Ch72 (AMD Radeon Tools)


### Chapter 153: OBS Studio GPU Pipeline on Linux *(Part IX — Tooling and Contributing)*

- libobs framework: modular C framework abstracting Sources, Filters, Transitions, Outputs, Encoders, Services; OpenGL (EGL/GLX) and experimental Vulkan rendering backends
- PipeWire DMA-BUF screen capture: xdg-desktop-portal ScreenCast → PipeWire stream → pw_stream_dequeue_buffer with SPA_DATA_DmaBuf; zero-copy path via gs_texture_create_from_dmabuf
- EGL DMA-BUF import: eglCreateImageKHR(EGL_LINUX_DMA_BUF_EXT) + glEGLImageTargetTexture2DOES for zero-copy GPU texture from PipeWire frame
- obs-vkcapture Vulkan game capture: VK_LAYER_OBS_vkcapture injects into game, intercepts vkQueuePresentKHR, exports swapchain image as DMA-BUF to OBS bypassing compositor
- OBS scene graph composition: GPU textures composited via OpenGL FBO with glBlendFunc; HLSL-like .effect shaders compiled to GLSL via gs_effect_create and Mesa GLSL compiler
- Hardware video encoding comparison: VA-API (Intel/AMD, DRM Prime DMA-BUF zero-copy), NVENC (NVIDIA, CUDA GL interop via NvEncRegisterResource/cuGraphicsGLRegisterImage), AMF (AMD, Vulkan interop via AMFSurface::InteropInitFrom(VkImage))
- Multi-GPU routing: DRI_PRIME=1, MESA_VK_DEVICE_SELECT for Mesa; __NV_PRIME_RENDER_OFFLOAD for NVIDIA; cross-GPU DMA-BUF via EGL_EXT_image_dma_buf_import with PCIe transfer; P2P on RDNA3/Ampere+
- v4l2loopback virtual camera: OBS OpenGL FBO → glReadPixels (CPU) → YUV convert → /dev/video10; V4L2 webcam source via VIDIOC_DQBUF with MJPEG/H.264 VA-API decode
- Browser source via CEF: Chromium Embedded Framework renders offscreen with Dawn/WebGPU or ANGLE/WebGL, shares framebuffer with OBS via shared memory or DMA-BUF
- libobs plugin API: obs_source_info with video_render callback; obs_encoder_info for custom hardware encoders; obs_enter_graphics/obs_leave_graphics for shared OpenGL context access
- Performance diagnostics: OBS Stats dock (render lag, frame time, skipped/dropped frames); intel_gpu_top/radeontop/nvidia-smi dmon; LIBVA_MESSAGING_LEVEL for VA-API debug
- Roadmap: Vulkan Video encode (VK_KHR_video_encode_queue) in RADV/ANV to replace VA-API; explicit sync via linux-drm-syncobj-v1 (landed OBS 31.1); AV1 NVENC via nvEncodeAPI for Ada Lovelace+
- **Integrations**: Ch38 (PipeWire), Ch123 (Screen Capture / xdg-desktop-portal), Ch26 (Hardware Video / VA-API), Ch142 (EGL and DMA-BUF), Ch49 (Multi-GPU PRIME), Ch55 (GPU Containers), Ch80 (GPU Security), Ch135 (Vulkan Ray Tracing / obs-vkcapture)


### Chapter 180: GPU Reverse Engineering: Tools, Methodology, and Case Studies *(Part IX — Tooling and Contributing)*

- RE-derived driver landscape: Nouveau (NVIDIA), Panfrost/Panthor (ARM Mali), etnaviv (Vivante GC), freedreno/turnip (Adreno), agx/Asahi (Apple AGX) — all built via command-stream and firmware tracing
- GPU RE layers: MMIO register space (PCI BAR0), command-stream format (pushbuffer/IB/command list), shader ISA (proprietary VLIW/RISC), and firmware protocol (ARM Cortex-M or Falcon shared-memory mailboxes)
- envytools nva tools: nvalist, nvapeek, nvapoke, nvafuzz, nvawatch, nvascan, nvagetbios, nvafakebios, nvadownload/nvaupload, nvafucstart — direct BAR0 MMIO access via nva.ko character device
- rnndb rules-ng-ng XML register database: headergen2 generates C headers consumed by Nouveau (nvkm_rd32/nvkm_wr32); demmt decodes valgrind-mmt traces against rnndb; demmio decodes mmiotrace output
- envydis/envyas ISA targets: nv50, gf100, gk110, gm107, falcon — multi-target disassembler/assembler for NVIDIA shader ISAs and Falcon microcontroller firmware
- valgrind-mmt + demmt pipeline: mmt tool intercepts mmap() pushbuffer writes and ioctl() calls to /dev/nvidiaX; demmt decodes method-value pairs, correlates DMA addresses, calls envydis inline
- MMIO probing methodology: /sys/bus/pci/devices/BDF/resource for BAR0 base; baseline nvapeek snapshot; trigger known GPU op; nvapoke candidate values; cross-reference VBIOS via nvbios; commit rnndb XML entry
- Command-stream capture per driver: Nouveau mmiotrace+valgrind-mmt; freedreno libwrap.so LD_PRELOAD + cffdump + .rd binary format + /sys/kernel/debug/dri/0/rd; etnaviv viv_interpose.so + libvivhook feature-bit overrides + dump_cmdstream.py; Mesa u_trace generic GPU timestamp tracing
- ISA RE: nvdisasm (CUDA SDK, SM50–SM100); llvm-objdump AMDGPU target; spirv-dis (SPIRV-Tools); intel_aubdump GEM execbuffer2 → AUB files; freedreno qrisc-disasm for SQE firmware
- Firmware RE: Ghidra for ARM Cortex-M (Mali CSF, Adreno GMU, NVIDIA PMU) and Falcon; radiff2/bindiff differential firmware analysis between consecutive versions
- Case studies: Nouveau/envytools G80 six-phase pipeline; Panfrost panwrap/panloader Mali job descriptor and Bifrost ISA; Asahi m1n1 hypervisor tracing of Apple AGX ASC with ~1000-field init structure; etnaviv viv_interpose + libvivhook + rnndb-format XML for Vivante GC
- Legal framework: DMCA 1201(f) interoperability exception; EU Directive 2009/24/EC Art. 6; clean-room methodology; GPL compatibility of RE-derived drivers; Asahi clean-specification procedural separation model
- **Integrations**: Ch7 (NVIDIA RE history and Nouveau), Ch8 (Nouveau kernel driver, nvhw/ headers from rnndb), Ch73 (Asahi Linux and Apple Silicon AGX), Ch90 (Panfrost, Panthor, and Lima), Ch160 (freedreno and turnip: Adreno on Linux)


### Part X — The Browser Rendering Stack

### Chapter 168: WebNN — The Web Neural Network API *(Part X — The Browser Rendering Stack)*

- WebNN API surface: navigator.ml, MLContextOptions (accelerated, powerPreference), createContext() overloads; deviceType removed from web-facing API in favour of accelerated/powerPreference pair
- MLGraphBuilder DAG model: MLOperand composition, builder.build() async compilation, immutable MLGraph; 95 operators across matmul, conv2d, relu, softmax, layerNormalization, gru, lstm, and more
- MLTensor buffer management: context.createTensor(), writeTensor(), readTensor(), destroy(); opaque device-side allocations replacing per-inference ArrayBuffer copies
- dispatch() execution timeline: replaces compute(); fire-and-forget enqueuing on context GPU timeline; pipelined multi-dispatch; MLContext.lost Promise for context-loss events
- WebGPU interoperability: createContext(GPUDevice) binding; exportableToGPU MLTensor; exportToGPU() zero-copy GPUBuffer sharing; gpu::SharedImage mailbox path in Chromium
- Chromium process layout: Blink third_party/blink/renderer/modules/ml/ (JS bindings), services/webnn/ (GPU-process backend), services/webnn/public/mojom/ (15 Mojo interface definitions)
- Chromium backend selection: webnn_context_provider_impl.cc; Windows→ORT/DirectML, macOS→CoreML, Android→NNAPI, Linux/ChromeOS→TFLite/XNNPACK CPU; kFallbackToInProcess sandboxed renderer path
- TFLite/XNNPACK Linux CPU backend: graph_builder_tflite.cc (quantised op fusion, FlatBuffer compilation), graph_impl_tflite.cc (Interpreter.Invoke), XNNPACK AVX2/AVX-512 SIMD kernels; no GPU delegate wired on Linux desktop
- Mojo IPC path: renderer→WebNNContextProviderImpl→WebNNContextImpl→WebNNGraphBuilderImpl→TFLite FlatBuffer→WebNNGraphImpl→XNNPACK inference; Mojo Data Pipe vs BigBuffer tensor transfer
- Linux GPU inference gap: accelerated:true maps to XNNPACK CPU regardless of powerPreference; no DirectML/CoreML/TensorRT-RTX equivalent; NVIDIA WebNN path Windows-only via TensorRT-RTX + Windows ML KB5096142
- Firefox WebNN: no production Gecko implementation; rustnn Rust project (3-layer: graph→ONNX/CoreML converters→executors) covering all 95 operators; content-process security gap not yet resolved; wgpu/Vulkan path envisioned
- NPU backend and Linux: Chromium internal mojom::Device::kNpu maps low-power preference; Intel intel_vpu + OpenVINO, AMD amdxdna + Peano LLVM, Qualcomm SNPE all lack services/webnn/ Linux backend; all fall back to CPU
- WebNN vs WebGPU for ML: operator fusion and SIMD CPU advantage for CNN/transformers; WebGPU preferred for large LLMs (FlashAttention-style, KV-cache); combined pipeline pattern: WebGPU preprocess → WebNN inference → WebGPU postprocess
- Developer workflow: navigator.ml feature detection, context.opSupportLimits() for operator/dtype capabilities, ONNX Runtime Web WebNN EP (ort-wasm-simd-threaded.jsep.wasm), Transformers.js v3+ device:'webnn', @webnn/polyfill, chrome://flags #web-machine-learning-neural-network, origin trial Chrome 146
- **Integrations**: Ch33 (Chromium GPU Architecture — WebNN runs in same GPU process as WebGPU/command-buffer/Viz, shares Mojo sandbox model), Ch35 (Dawn and WebGPU — createContext(GPUDevice) shares Dawn VkDevice; no Dawn WebNN backend), Ch36 (Chromium Compositor and Viz — gpu::SharedImage mailbox backing for interop MLTensor), Ch52 (Firefox and WebRender — wgpu GPUDevice interop path for future Firefox WebNN), Ch88 (NPU and AI Accelerator Integration — intel_vpu/amdxdna kernel drivers and OpenVINO/XDNA user-space runtimes a future Linux WebNN NPU backend would use), Ch98 (WebAssembly and WebGPU as Deployment Target — WASM+WebGPU vs WebNN execution provider comparison in ONNX Runtime Web)


### Chapter 193: Tauri — Rust-Native Desktop Applications via WebKitGTK *(Part X — Browser Rendering Stack)*

- Tauri vs. Electron architecture: system WebView (WebKitGTK, ~0 MB bundled engine) vs. bundled Chromium (~120 MB); GTK3 window management vs. Chrome GPU process; OpenGL ES vs. ANGLE/Vulkan rendering path
- Process model: Application Process (GTK3 + Tao EventLoop + Wry) → WebKit Web Content Process (WebCore + JavaScriptCore + GL compositor) → UI-side compositor (wl_surface::commit); DMA-BUF GPU buffer sharing between processes
- Tao crate: winit fork with GTK3 Linux backend; ApplicationWindow + gtk_init(); required deps: libgtk-3-dev, libwebkit2gtk-4.1-dev; GTK3 (not GTK4) — no GSK scene graph
- Wry crate: cross-platform WebView abstraction; WebViewBuilder, WebViewBuilderExtUnix, WebContext; custom URI scheme via webkit_web_context_register_uri_scheme(); tauri://localhost asset serving without a network socket
- Backend substitution: tauri-runtime-wry (WebKitGTK, production) vs. tauri-runtime-verso (Verso/Servo, experimental, NLnet-funded); compile-time Cargo.toml choice; WPE embedded backend planned; CEF discussed; no runtime engine swap
- WebKitGTK rendering: WebKitWebView GObject widget; WebKitSettings hardware acceleration policy; webkit_web_view_get_user_content_manager(); EGL context on DRM render node (/dev/dri/renderD128); OpenGL ES 3.0 via Mesa GLES state tracker
- Linux rendering path: WebCore layout → WebKit threaded compositor → OpenGL ES 3.0 → Mesa NIR → DRM GEM → DMA-BUF fd → zwp_linux_dmabuf_v1 → wl_surface::commit → KMS atomic commit
- IPC system: #[tauri::command] proc macro + serde_json; window.webkit.messageHandlers.__TAURI_IPC__.postMessage(); WebKit Unix socket IPC crossing; evaluate_javascript() response; Channel<T> API for streaming via custom URI scheme
- Security: Tauri 2.0 capabilities system (JSON capability files, default-deny, compile-time permission table); CSP with SHA-256 hash injection at build time via tauri-build; Isolation Pattern (sandboxed iframe + AES-GCM key) for untrusted third-party content
- Plugin system: tauri-plugin-fs (path glob scopes), tauri-plugin-dialog (GtkFileChooserDialog), tauri-plugin-notification (zbus → org.freedesktop.Notifications D-Bus), tauri-plugin-updater (AppImage/deb + ed25519 signature verification)
- Distribution: AppImage (FUSE-mountable, 70-100 MB with bundled WebKitGTK); .deb (2-10 MB binary + system libwebkit2gtk-4.1-37 dependency); .rpm (via Rust rpm crate, webkit2gtk4.1 dep on Fedora/RHEL)
- Platform strategy: WebKitGTK required because it is the only production system WebView on Linux; GTK required because WebKitWebView is a GtkWidget subclass; WPE escape hatch for embedded/kiosk (GBM/DMA-BUF, no GTK); Servo/Verso not yet production-ready
- Roadmap: GTK4/webkitgtk-6.0 migration (GSK Vulkan renderer for window chrome, tracking issues #12561/#12563); WebKit WebGPU via wgpu (Vulkan path for web content); Servo WebRender→wgpu migration for tauri-runtime-verso Vulkan compositor; native DMA-BUF wl_subsurface zero-copy path
- **Integrations**: Ch4 (GBM and DMA-BUF), Ch20 (Wayland Protocol Fundamentals), Ch34 (ANGLE and WebGL), Ch39 (GTK4 and GNOME Application Stack), Ch45 (Terminal Integration with the Compositor Stack), Ch52 (Firefox and WebRender), Ch192 (Ratatui TUI Framework)


### Part XI — Engines & Creative Tools

### Chapter 190: VTK — Scientific Visualization on the Linux Graphics Stack *(Part XI — Engine and Creative Tools)*

- VTK pipeline architecture: demand-driven vtkAlgorithm/vtkDataObject model; vtkRenderWindow + vtkRenderer + vtkActor scene graph; vtkPolyDataMapper for GPU upload; Python >> operator (VTK 9.4+)
- VTK data model: vtkPolyData, vtkUnstructuredGrid, vtkImageData, vtkHyperTreeGrid, vtkMultiBlockDataSet; vtkCellArray with 64-bit offsets; vtkImplicitArray zero-copy virtual arrays; vtkCellGrid for discontinuous Galerkin FEM
- Rendering backends on Linux: vtkXOpenGLRenderWindow (GLX), vtkEGLRenderWindow (GPU headless via EGL_NO_SURFACE), vtkOSOpenGLRenderWindow (libOSMesa software); runtime glad symbol loading (VTK 9.4+); Wayland via VTK_USE_WAYLAND_OPENGL+EGL
- GPU volume rendering: vtkGPUVolumeRayCastMapper ray casting via GL_TEXTURE_3D, entry/exit FBOs, vtkVolumeShaderComposer dynamic GLSL; vtkSmartVolumeMapper auto-selects GPU/CPU/OSPRay; vtkColorTransferFunction + vtkPiecewiseFunction; vtkMultiVolume for co-registered CT/PET
- WebGPU backend (vtkRenderingWebGPU): Dawn → Vulkan on Linux desktop; compute shaders for frustum culling and billion-point point cloud rendering; experimental in VTK 9.3-9.6; no volume rendering yet
- ANARI backend (vtkRenderingANARI, VTK 9.4+): Khronos ANARI 1.0 portable rendering API; vtkAnariPass; backends: Intel OSPRay (CPU/GPU), NVIDIA VisRTX (OptiX RTX), AMD RadeonProRender; ANARI_LIBRARY env var selects implementation
- VTK-m GPU-accelerated filters: ArrayHandle control/execution split; WorkletMapField/MapTopology/ReduceByKey; device adapters: Serial, OpenMP, TBB, CUDA (nvcc), HIP (hipcc), Kokkos; Contour (Marching Cubes), Gradient, Threshold, Streamline (RK4), Triangulate; VTK_ENABLE_VTKM_OVERRIDES intercepts vtkContourFilter
- Scientific I/O formats: legacy .vtk, VTK XML (.vtp/.vtu/.vti/.pvtu), VTKHDF (HDF5 + MPI-IO, VTK 9.1+), ADIOS2 BP4/BP5 via vtkADIOS2VTXReader, DICOM via vtkDICOMImageReader/vtk-dicom, Exodus II (FEM/SEACAS), NetCDF CF-convention, EnSight Gold binary
- ParaView client-server: pvserver (MPI + EGL/OSMesa), paraview Qt GUI, remote render with JPEG/LZ4 image compression; IceT binary-swap sort-last depth compositing for parallel rendering; pvpython/pvbatch for headless/Slurm batch
- Catalyst 2.0 in-situ: libcatalyst.so stub + dlopen catalyst-paraview.so implementation; catalyst_execute(conduit_node_t*) with Conduit Mesh Blueprint; decouples simulation from visualization backend at deploy time; ADIOS2 SST streaming
- Headless and container deployments: OSMesa (no GPU), EGL (GPU without display), VTK_DEFAULT_EGL_DEVICE_INDEX for multi-GPU; kitware/vtk Docker images; NVIDIA Container Toolkit + EGL ICD injection for Kubernetes GPU pods
- Scientific ecosystem integrations: numpy_to_vtk/vtk_to_numpy zero-copy; dataset_adapter NumPy-style field access; itkwidgets for Jupyter; 3D Slicer MRML scene graph + VTK-ITK bridge; trame Python web framework for remote/local rendering via vtk.js; vtk.js WebGL/WebGPU browser client
- **Integrations**: Ch12 (Mesa Loader), Ch17 (Software Renderers/OSMesa), Ch24 (Vulkan and EGL), Ch25 (GPU Compute), Ch42 (Blender GPU/Cycles), Ch48 (ROCm), Ch55 (GPU Containers), Ch107 (Headless Rendering), Ch176 (OpenCASCADE)


### Part XII — Terminal Graphics

### Chapter 174: WezTerm and Alacritty — GPU Terminal Rendering Architectures *(Part XII — Terminal Graphics)*

- WezTerm crate structure: 35-crate Cargo workspace; key crates wezterm-gui (RenderContext, GlyphCache), wezterm-font (HarfBuzz shaping), termwiz (standalone terminal surface library), mux (Window/Tab/Pane hierarchy)
- WezTerm dual-renderer: FrontEndSelection enum (OpenGL via glium default, WebGpu opt-in, Software); RenderContext enum dispatches allocate_texture_atlas / vertex buffers to Glium or WebGpu backends
- WezTerm wgpu initialisation: WebGpuState::new_impl() — wgpu Instance → Surface (wl_surface via VK_KHR_wayland_surface) → Adapter (RADV/ANV/NVK) → Device; PresentMode::Fifo; CompositeAlphaMode::PostMultiplied
- WezTerm glyph atlas: GlyphCache with guillotiere rectangle packing; CachedGlyph carries Sprite UV refs; LfuCache for pixel-graphics image data; atlas-full triggers full GlyphCache rebuild; custom box-drawing via tiny_skia PolyCommand paths
- WezTerm font shaping: HarfBuzz via FFI in wezterm-font/src/shaper/harfbuzz.rs; GlyphInfo carries hb_position_t units (1/64 px); GSUB/GPOS for ligatures and emoji ZWJ clusters; FreeType rasterisation with FT_LOAD_TARGET_LCD/LIGHT/MONO
- WezTerm rendering pipeline: TripleLayerQuadAllocator (background/text/cursor layers); ShaderUniform (foreground HSB, milliseconds, projection matrix); WGSL shaders compiled by naga → SPIR-V → vkCreateShaderModule → Mesa NIR → ACO/LLVM-IR; single RenderPass draw_indexed()
- WezTerm Wayland/IME: winit Wayland backend; wp_fractional_scale_v1 (Issue #5149); zwp_text_input_v3 IME (Issue #1772); DMA-BUF submission handled transparently by Mesa Vulkan WSI; zwp_linux_drm_syncobj_v1 explicit-sync roadmap
- WezTerm built-in multiplexer: Mux singleton; Domain abstraction (LocalDomain, RemoteSshDomain, ClientDomain, ExecDomain); PDU protocol with LEB128 framing and bincode; dirty-tracking via pane.get_changed_since(seqno)
- WezTerm pixel graphics protocols: Sixel (experimental, termwiz ImageData), Kitty Graphics Protocol (enable_kitty_graphics, cell-mapping caveat Issue #986), iTerm2; image_cache LfuCache keyed by 32-byte hash; LFU eviction; animated frame pacing via min_frame_duration
- Alacritty crate structure: 4-crate workspace (alacritty, alacritty_terminal, alacritty_config, alacritty_config_derive); alacritty_terminal is reusable VT library used by zellij; no HarfBuzz, no ligatures, no pixel graphics protocols by design
- Alacritty OpenGL context: Display owns glutin + winit state; Gles2Renderer (ES 2.0) vs Glsl3Renderer (GL 3.3 GLSL 330); EGL path — eglGetPlatformDisplay(EGL_PLATFORM_WAYLAND_KHR) → Mesa EGL → DRI render node /dev/dri/renderD128; eglSwapBuffers → zwp_linux_dmabuf_v1 via Mesa EGL
- Alacritty instanced rendering: glDrawElementsInstanced with BATCH_MAX=65536; static index buffer for unit quad; vbo_instance (STREAM_DRAW) re-uploaded per frame; dual-source blending gl::SRC1_COLOR / gl::ONE_MINUS_SRC1_COLOR for subpixel AA; DamageTracker skips frames with no changes
- Alacritty glyph cache and atlas: GlyphCache HashMap with ahash; Atlas uses hand-written shelf bin-packing (ATLAS_SIZE=1024); appends new Atlas on full (vs WezTerm's full-cache rebuild); crossfont wraps FreeType on Linux; BitmapBuffer RGB/RGBA; no HarfBuzz shaping (per-codepoint only)
- Linux graphics stack paths: WezTerm wgpu → ash → libvulkan.so → RADV/ANV/NVK → Mesa NIR → ACO/LLVM-IR → kernel DRM; Alacritty glutin → EGL → Mesa GL state tracker → Gallium3D (radeonsi/iris/nouveau) → kernel DRM
- **Integrations**: Ch43 (Terminal Pixel Protocols), Ch44 (Kitty and Ghostty GPU Rendering), Ch45 (Terminal Integration with Compositor Stack), Ch14 (Mesa Shader Compiler: NIR/ACO/LLVM), Ch18 (Mesa Vulkan Drivers: RADV/ANV/NVK), Ch19 (Mesa OpenGL), Ch40 (Bevy and wgpu), Ch152 (Rust GPU Ecosystem: ash/wgpu/naga)


### Chapter 178: The PTY/TTY Kernel Layer and Line Disciplines *(Part XII — Terminal Graphics)*

- TTY subsystem architecture: layered stack (tty_io.c core → line discipline → TTY driver → hardware/virtual device); device nodes /dev/tty, /dev/console, /dev/ttyS*, /dev/ptmx, /dev/pts/N
- TTY device types: UART (8250/PL011), USB serial (ftdi_sio/pl2303/cp210x), CDC-ACM (cdc_acm), JTAG dual-channel probe, Bluetooth HCI bridge (N_HCI line discipline via hci_uart.c), pseudo-terminal (pty.c)
- tty_struct, tty_driver, tty_operations: key kernel structs including tty->link for PTY master/slave pairing; tty_alloc_driver(), tty_register_driver(), mandatory open/write callbacks
- PTY architecture (pty.c): UNIX 98 /dev/ptmx + /dev/pts/N vs BSD legacy; pty_write() via tty->link and tty_insert_flip_string_and_push_tail(); ptm_unix98_ops vs pty_unix98_ops
- devpts filesystem and PTY lifecycle: posix_openpt(), grantpt(), unlockpt(), ptsname(); ptmx_open() allocates tty_struct pair; TTY_PTY_LOCK/TIOCSPTLCK; grantpt() no-op since glibc 2.33
- N_TTY line discipline internals: struct n_tty_data, N_TTY_BUF_SIZE=4096, flip buffer (tty_bufhead/tty_buffer), canonical mode (ICANON, line editing, canon_head), echo processing (echo_buf, ECHO/ECHOE/ECHOCTL), raw mode (VMIN/VTIME), signal delivery via isig() and kill_pgrp()
- termios interface: struct ktermios/termios with c_iflag/c_oflag/c_cflag/c_lflag/c_cc[]; tcgetattr()/tcsetattr() with TCSANOW/TCSADRAIN/TCSAFLUSH; cfmakeraw() clears all processing flags
- TIOCSWINSZ and SIGWINCH chain: struct winsize (ws_row/col/xpixel/ypixel); compositor → emulator → TIOCSWINSZ → kernel stores winsize → kill_pgrp(SIGWINCH) → shell COLUMNS/LINES update; pixel dimension contract for Kitty protocol
- Terminal emulator I/O architecture: poll()+read() read thread with 64 KiB buffer; VT DFA state machine (Ground/Escape/CSI/OSC/DCS states); Ghostty SIMD AVX2/NEON parser sustaining >100 MB/s; libghostty-vt standalone library
- PTY performance: N_TTY_BUF_SIZE 4 KiB bottleneck causing write blocking; PTY vs pipe throughput (10–50 MB/s vs hundreds MB/s); large master read buffers, O_NONBLOCK, render budget (max bytes-per-frame) for GPU terminal vsync pacing
- Sessions, process groups, controlling terminal: setsid() creates session leader; slave opened without O_NOCTTY acquires controlling terminal; tcsetpgrp() sets foreground process group in tty->pgrp; ^C → byte 0x03 → VINTR → isig() → SIGINT to pgrp (not emulator)
- openpty()/forkpty() lifecycle: <pty.h>; forkpty() = openpty() + fork() + setsid() + slave as stdin/stdout/stderr; portable-pty crate (WezTerm), nix crate; security: TIOCSTI disabled for non-privileged callers (Linux 6.2), devpts newinstance isolation
- Terminal multiplexers PTY chaining: tmux client-server daemon (libevent, struct grid/screen VT state machine, allow-passthrough DCS envelope, TIOCSWINSZ cascade per pane); zellij (Rust/tokio, Wasmtime WASM plugins, KDL layouts, APC passthrough without opt-in)
- io_uring PTY I/O (Linux 5.1+): SQ/CQ ring buffers; io_uring_prep_read() with registered fds; IORING_OP_READ_MULTISHOT with provided buffer rings (Linux 6.1+); IORING_OP_SPLICE for zero-copy bridging; Ghostty uses io_uring with epoll fallback; Wayland compositor-native multiplexing via foot --server vs tmux PTY chaining
- **Integrations**: Ch43 (Terminal Pixel Protocols — Sixel/Kitty/iTerm2), Ch44 (Terminal GPU Rendering Architectures), Ch45 (Terminal Integration with Compositor Stack), Ch174 (WezTerm and Alacritty Architecture), Ch20 (Wayland EGL clients), Ch22 (Wayland window management)


### Part XIII — Video Streaming on Linux

### Chapter 189: VLC Media Player — Architecture, GPU Acceleration, and the Linux Graphics Stack *(Part XIII — Video Streaming on Linux)*

- VLC plugin-based architecture: vlc_object_t hierarchy, module bank with dlopen/vlc_entry, score-based capability selection (vlc_module_begin/end macros, set_capability score), input_thread_t / vout_thread_t / aout_instance_t
- libVLC embedding API: libvlc_instance_t, libvlc_media_t, libvlc_media_player_t; autotools/Meson build; libvlccore/libvlc packaging split
- Demux and input layer: access modules (file/http/rtp/v4l2), vlc_tick_t clock, es_out_t interface (es_out_Add/Send/SetPCR), block_t struct (i_pts/i_dts/i_flags), adaptive ABR demux with bandwidth estimator
- VA-API hardware decode path: modules/hw/vaapi/ + modules/codec/avcodec/vaapi.c; vaCreateConfig/Context/BeginPicture/RenderPicture/EndPicture; VASurface pool recycling; VADRMPRIMESurfaceDescriptor zero-copy DMA-BUF export via VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2
- Embedded hardware decode: V4L2 M2M stateless (rkvdec/hantro, V4L2_CID_STATELESS_H264_SPS/PPS/SLICE_PARAMS) vs stateful (bcm2835-codec/Venus); MMAL path for Raspberry Pi (modules/hw/mmal/); NVDEC path (modules/hw/nvdec/); per-platform coverage table
- Wayland zero-copy video output: modules/video_output/wayland/; zwp_linux_dmabuf_v1 import of DMA-BUF fds as wl_buffer; wp_viewporter for GPU-side scaling; xdg_toplevel window; linux-drm-syncobj-v1 explicit sync (VLC 4.0 master)
- Vulkan renderer (VLC 4.0 master): modules/video_output/vulkan/; VK_KHR_wayland_surface swapchain; VASurface import via VK_EXT_external_memory_dma_buf + VK_EXT_image_drm_format_modifier; libplacebo HDR tone mapping, debanding, upscaling, AV1 film grain on GPU
- OpenGL/EGL renderer (VLC 3.x stable): modules/video_output/opengl/; interop_vaapi.c EGLImage import via EGL_EXT_image_dma_buf_import; GLSL YUV-to-RGB fragment shader (NV12/I420/P010); GBM/KMS direct rendering via egl_display_gbm.c + gbm_surface + drmModePageFlip
- Audio output stack: ALSA (snd_pcm_open/writei, IEC 958 SPDIF passthrough); PipeWire preferred on GNOME/KDE Wayland (pw_stream_new_simple, spa_audio_info_raw); PulseAudio fallback; AV sync via audio clock master and PCR from TS demux
- Stream output transcoding (--sout): chain syntax #transcode{vcodec,venc}:sink; VA-API hardware H.264/HEVC encode; zero-copy decode-to-encode sharing VADisplay; HTTP streaming server; mosaic bridge filter
- VLC 4.0 roadmap: Qt6 native Wayland UI; Vulkan/libplacebo as default renderer replacing OpenGL; libplacebo Dolby Vision Profile 5 (IPT-PQ) SoC 2026; linux-drm-syncobj-v1 eliminating vaSyncSurface CPU stalls; Vulkan Video (VK_KHR_video_decode_queue) future native decode path
- Comparison table — VLC vs GStreamer vs FFmpeg: plugin auto-selection vs explicit graph wiring vs static codec registry; Wayland zero-copy paths; VA-API and Vulkan renderer status; audio session management
- **Integrations**: Ch20 (Wayland: zwp_linux_dmabuf_v1, xdg_toplevel, wp_viewporter, linux-drm-syncobj-v1), Ch24 (Vulkan: VK_EXT_external_memory_dma_buf, VK_EXT_image_drm_format_modifier, VK_KHR_wayland_surface), Ch26 (VA-API hardware decode), Ch38 (PipeWire audio), Ch57 (FFmpeg avcodec/avformat), Ch58 (GStreamer pipeline comparison), Ch140 (HDMI audio passthrough via ALSA/PipeWire), Ch142 (V4L2 media subsystem), Ch158 (HDR display, libplacebo PQ-to-SDR), Ch186 (NV12/P010 pixel formats and DRM format modifiers)


### Part XVII — AMD Developer Ecosystem

### Chapter 170: AMDVLK vs. RADV — AMD's Two Open Vulkan Drivers *(Part XVII — AMD Developer Ecosystem)*

- AMDVLK architecture: five repos (xgl, pal, llpc, gpurt, llvm-fork); XGL Vulkan front-end translates vkCreate* calls to PAL API calls; no shader compilation in XGL
- PAL (Platform Abstraction Layer): cross-platform GPU abstraction shared with Windows DX12 and ROCm; submits via raw DRM_IOCTL_AMDGPU_CS without libdrm; PM4 command buffer encoding, VRAM heap selection, PAL ELF ABI
- LLPC pipeline compiler: SPIR-V -> LLVM IR (lgc.call.* metadata) -> LGC middle-end (BuilderReplayer, LS-HS/NGG shader merging, SGPR/VGPR setup) -> LLVM AMDGPU backend -> PAL ABI ELF; amdllpc offline tool
- GPURT ray tracing library: BVH construction/traversal for VkAccelerationStructureKHR; HLSL traversal shader via DirectXShaderCompiler; shared with DX12 path; archived September 2025
- ICD discovery: amdvlk64.so + /etc/vulkan/icd.d/amd_icd64.json; VK_ICD_FILENAMES and VK_DRIVER_FILES for ICD selection; AMD_VULKAN_ICD=AMDVLK|RADV Arch convention
- RADV architecture: Mesa src/amd/vulkan/; NIR-centric via spirv_to_nir() and radv_optimize_nir(); kernel submission via libdrm_amdgpu through radeon_winsys abstraction
- ACO custom compiler backend (Mesa 20.1): NIR -> ACO IR via aco::select_program(); linear-scan register allocator for GCN/RDNA banked register file; direct ISA binary emission; RADV_DEBUG=llvm selects LLVM fallback
- Compiler pipeline comparison: AMDVLK LLPC uses full LLVM opt pipeline (high quality, slow compile); RADV ACO purpose-built for latency (fast compile, competitive quality); both support Wave32/Wave64
- Pipeline cache and VK_EXT_graphics_pipeline_library: RADV stores ACO blobs in mesa_shader_cache_sf (MESA_SHADER_CACHE_DIR); LLPC uses partial pipeline hash split; RADV GPL enabled by default Mesa 23.1
- Extension history: VK_AMD_rasterization_order, VK_AMD_gcn_shader, VK_AMD_shader_ballot, VK_AMD_shader_info, VK_AMD_anti_lag (1.3.291), VK_AMDX_shader_enqueue (work graphs experimental); Vulkan 1.4 conformance both drivers
- Distribution defaults: RADV default on all major distros (mesa-vulkan-drivers / vulkan-radeon); Steam Deck ships RADV exclusively; Radeon Software 25.20 dropped AMDVLK in favour of Mesa RADV
- Performance: RADV wins majority of gaming benchmarks (ACO compile speed, VK_EXT_graphics_pipeline_library, Valve tuning); Mesa 26.0 Wave32 ray tracing closes RT gap 50-75%; RADV ahead on llama.cpp Vulkan compute (1097 vs 777 tokens/s on Strix Halo)
- AMDVLK discontinuation: final release v-2025.Q2.1 (April 2025); AMD archival announcement September 15 2025; XGL/PAL/GPURT repos archived; RDNA4 (GFX12) was last supported generation; LLPC survives as ROCm amdllpc tool
- VkPhysicalDeviceDriverProperties detection: VK_DRIVER_ID_MESA_RADV vs VK_DRIVER_ID_AMD_OPEN_SOURCE; driverName 'radv' vs 'AMD open-source driver'; RADV_DEBUG flags: errors, shaders, spirv, nir, aco, nodcc, nomemorycache
- **Integrations**: Ch9 (Mesa Architecture), Ch15 (ACO Compiler), Ch31/Ch5 (amdgpu kernel driver), Ch35 (VA-API on AMD), Ch72 (AMD FidelityFX SDK), Ch143 (RADV Internals)


### Part XXVII — Display Hardware, Connectors, and Signal Standards

### Chapter 181: Modern Display Interface Standards *(Part XXVII — Display Hardware and Connectors)*

- DisplayPort evolution (DP 1.1–2.1): 8b/10b vs 128b/132b encoding, link rate table (RBR through UHBR20), DPCD AUX channel, MST (DP 1.2), DSC/FEC/PSR2 (DP 1.4), 77.37 Gbps ceiling at UHBR20
- Panel Replay (DP 2.1): replaces PSR2, VSC SDP revision 0x07, DP_PANEL_REPLAY_CAP DPCD register, TRANS_DP2_CTL on Meteor Lake, intel_psr.c; 99% bandwidth reduction during static frames
- Forward Error Correction: Reed-Solomon RS(254,250), 2.4% overhead, drm_dp_get_fec_ready_flag() in drm_dp_helper.c, mandatory prerequisite for DSC activation
- HDMI evolution (1.0–2.1b): TMDS 8b/10b → Fixed Rate Link (FRL) 16b/18b, seven FRL levels (FRL0–FRL6) from TMDS to 48 Gbps raw, dedicated link-training protocol replacing clock lane
- HDMI 2.1 feature suite: eARC (37 Mbps return channel for lossless audio), ALLM (Game Mode signalling), HDMI Forum VRR (VS-IF negotiation), QFT (Quick Frame Transport), QMS (seamless refresh-rate switching)
- HDMI 2.1a/2.1b: Source-Based Tone Mapping (SBTM) via extended Dynamic HDR InfoFrame (2.1a); explicit 4K/144Hz profiling for VRR/ALLM/QFT (2.1b)
- USB4 display tunnelling: v1 (40 Gbps) vs v2 (80 Gbps symmetric, 120 Gbps asymmetric Bandwidth Boost); DP tunnel carries Main Link + AUX + HPD as logical sub-channels via DP Bandwidth Allocation (DPB) protocol
- Thunderbolt 3/4/5: TBT3 = 40 Gbps dual DP 1.2; TBT4 = mandatory DP 2.0, two 4K/60 displays, IOMMU DMA protection; TBT5 = 120 Gbps Bandwidth Boost, DP 2.1 UHBR20, 240W USB-PD, dual 8K/60
- HDBaseT 2.0: 5PLAY feature set over Cat-6 at 100m, 18 Gbps matching HDMI 2.0, no upstream KMS driver (transparent HDMI/DP bridge); HDBaseT 3.0 targeting 100 Gbps for commercial AV
- VESA AdaptiveSync Display 1.1a: 144Hz minimum, 60Hz VRR floor, 9x9 GtG test matrix, Dual-Mode certification; MediaSync for 24/48Hz film content; DPCD DP_EDP_CONFIGURATION_SET register
- DisplayHDR certification tiers (v1.2): LCD tiers HDR400–HDR1400 (local dimming requirements per tier); OLED/MicroLED True Black 400/500/600; kernel only signals hdr_output_metadata, does not drive panel algorithms
- Linux kernel UHBR support (i915): intel_dp_is_uhbr(), intel_dp_set_dpcd_sink_rates(), intel_dp_128b132b_intra_hop(); MTL=UHBR20, BMG=UHBR13.5; CONFIG_DRM_I915, CONFIG_THUNDERBOLT, CONFIG_USB4
- HDMI 2.1 FRL in amdgpu: patch series targeting Linux 7.2, disabled by default (amdgpu.dc_feature_mask=0x400 to enable), DCN 3.x integration, DSC over HDMI FRL; TB_TUNNEL_DP abstraction for USB4 DP tunnel bandwidth arbitration
- Bandwidth arithmetic worked examples: 4K/144Hz needs DSC on DP 1.4 (28.66 Gbps > 25.92 Gbps effective) but fits UHBR10 uncompressed; 8K/60 10-bit 4:4:4 only fits DP 2.1 UHBR20 (77.37 Gbps) uncompressed; DSC 3:1 on UHBR20 reaches 232 Gbps equivalent headroom; drm_dsc_compute_rc_parameters() in drm_dsc_helper.c
- **Integrations**: Ch2 (KMS Display Pipeline: drm_connector/drm_encoder/drm_crtc, hdr_output_metadata property), Ch128 (DisplayPort MST and Multi-Monitor Topology), Ch140 (HDMI and DisplayPort Audio: eARC, ACR, ALSA/ASoC), Ch172 (eGPU on Linux: Thunderbolt/USB4/PCIe Hot-Plug bandwidth partitioning), Ch182 (Connectors and Physical Layer: USB-C, DP80/HDMI Ultra-High-Speed cable certification), Ch183 (EDID and DisplayID: capability reporting for DSC/FRL/HDR), Ch184 (eDP: laptop-internal DP variant, eDP 1.5 UHBR, Panel Replay), Ch186 (Pixel Formats and Signal Encoding: per-format bandwidth costs)


### Chapter 182: Digital Display Connectors and the Physical Layer *(Part XXVII — Display Hardware and Connectors)*

- USB-C DisplayPort Alt Mode entry sequence: PD VDM Discover Identity/SVIDs/Modes/Enter_Mode over CC line at 300 kbps, SID 0xFF01, DP_Configure; kernel drivers/usb/typec/altmodes/displayport.c
- USB-C DP pin assignments: typec_dp_pin_assignment enum (C/D/E active); assignment D = 2-lane DP + USB3 SS multi-function mode; SBU1/SBU2 carry AUX CH differential pair
- TCPM Linux kernel state machine: drivers/usb/typec/tcpm/tcpm.c; FOREACH_STATE macro; pd_mode_data struct; tcpm_cc_is_sink()/tcpm_cc_is_source() CC line voltage helpers
- Thunderbolt 3/4 and USB4: ICM firmware security levels (none/user/secure/dponly) via sysfs /sys/bus/thunderbolt; drivers/thunderbolt/ (tb.c, icm.c, switch.c, dp_tunnel.c); struct tb_switch, struct tb_port with cap_usb4
- HDMI connector variants: Type A (19-pin, 13.9mm), Mini-C, Micro-D, Automotive-E; HDMI 2.1 Fixed Rate Link (FRL) replacing TMDS with 16b/18b encoding, 3-4 lanes up to 12 Gbps/lane (FRL6=48 Gbps raw)
- DVI variants and TMDS compatibility: DVI-D Single/Dual Link, DVI-I (analogue+digital), DVI-A; passive DVI-D to HDMI valid for video (same TMDS); dual-link DVI to HDMI requires active converter
- HPD electrical signalling: per-connector voltage levels (HDMI pin19 5V, DP pin18 3.3V); DP short/long pulse multiplexing (>2ms connect, 0.5-1ms MST topology/IRQ_HPD); drm_kms_helper_hotplug_event() firing uevents
- DDC I2C bus: EDID at address 0x50, extension segments at 0x30; drm_get_edid() in drm_edid.c; DP uses I2C-over-AUX via struct drm_dp_aux with drm_dp_aux_register(); DDC/CI monitor control at address 0x37 using VCP codes via ddcutil
- DRM connector state machine: enum drm_connector_status (connected/disconnected/unknown); drm_connector_list_iter safe iteration; DRM_CONNECTOR_POLL_CONNECT/DISCONNECT/HPD flags; sysfs at /sys/class/drm/card0-HDMI-A-1/{status,modes,edid}
- Signal integrity at HBR3: DP_TRAIN_VOLTAGE_SWING_LEVEL_0-3 (200-800mV) and DP_TRAIN_PRE_EMPH_LEVEL_0-3 (0-9.5dB) in drm_dp.h; drm_dp_link_train_clock_recovery_delay() and channel_eq_delay() helpers; spread-spectrum clocking via DPCD DP_DOWNSPREAD_CTRL 0x0107
- Active vs passive adapters: DP++ dual-mode for passive DP-to-HDMI 1.4; active DP-to-HDMI 2.0 contains TMDS encoder chip; DP-to-HDMI 2.1 adapters use UHBR10 input with FRL output; DPCD Branch Device OUI at 0x0500-0x0505 for adapter identification
- Standard DP connector variants: full-size DP (16.1mm, 20-pin), Mini DisplayPort (7.5mm), Micro DisplayPort (4.6mm); all share same electrical specs; TVS ESD diode capacitance must be under 0.5pF at HBR3
- **Integrations**: Ch1 (DRM Driver Architecture), Ch2 (KMS Display Pipeline), Ch128 (DisplayPort MST), Ch140 (HDMI and DisplayPort Audio), Ch172 (eGPU over Thunderbolt), Ch181 (Display Interface Standards), Ch183 (EDID and Display Capability Negotiation)


### Chapter 183: EDID and DisplayID: How Linux Discovers Display Capabilities *(Part 27 — Display Hardware and Connectors)*

- EDID base block (128-byte structure): manufacturer PNP ID, DTD timings, chromaticity coordinates, established/standard timings; E-EDID 1.4 adds bit-depth field and CVT support
- CEA-861/CTA-861 extension block: Data Block Collection format (tag+length headers), Video Data Block with VIC codes, Audio Data Block with Short Audio Descriptors (SADs), HDMI VSDB (OUI 0x000C03) with deep-color flags and CEC SPA
- HDR and WCG signalling: HDR Static Metadata Block (extended tag 0x06) with EOTF bitmap and MaxCLL/MaxFALL; Colorimetry Data Block (extended tag 0x05) for BT.2020RGB/DCI-P3; Video Capability Data Block (extended tag 0x00) for quantization range
- DisplayID 2.0 block architecture: variable-length typed blocks (Product ID 0x20, Display Parameters 0x21, Type VII Timing 0x22, Tiled Display Topology 0x28, Container ID 0x29); iterator API displayid_iter_edid_begin/next/end in drm_edid.c
- Linux EDID parsing (drm_edid.c): drm_edid_read(), drm_edid_read_ddc(), drm_edid_read_dp(); opaque struct drm_edid wrapper (since 5.19); checksum validation via edid_block_valid(); drm_edid_connector_update() triggering drm_parse_cea_ext() pipeline
- drm_display_info struct: accumulates bpc, is_hdmi, has_audio, color_formats, hdmi_colorimetry, hdr_sink_metadata from parsed EDID/CEA data; drm_edid_connector_add_modes() builds drm_display_mode list from DTDs, SVDs, standard timings, DMT modes
- EDID quirks database (edid_quirk_list[]): enum drm_edid_internal_quirk flags including EDID_QUIRK_PREFER_LARGE_60, EDID_QUIRK_FORCE_8BPC, EDID_QUIRK_NON_DESKTOP (VR headsets); EDID_QUIRK macro keyed by 3-letter PNP code and product ID
- EDID override and firmware injection: drm.edid_firmware kernel parameter; /lib/firmware/edid/ pre-built binaries; drm_load_edid_firmware() in drm_edid_load.c; drm_connector_override_edid() for runtime override; use cases: headless servers, broken monitors, IGT CI testing
- DDC/CI monitor control with ddcutil: VCP code space (MCCS); I2C address 0x37 for HDMI/DVI vs /dev/drm_dp_auxN for DisplayPort; VCP codes for brightness (0x10), contrast (0x12), input select (0x60), power mode (0xD6); libddcutil C API
- HDCP sink capability discovery: HDCP 1.4 capability via Bksv read from DDC 0x74/DPCD 0x68000 (not in EDID); HDCP 2.3 signalled in DPCD 0x006A or HDMI Forum VSDB; DRM HDCP framework (drm_hdcp.c): Content Protection property with UNDESIRED/DESIRED/ENABLED states
- Practical debugging: edid-decode tool for parsing EDID blobs from /sys/class/drm/card0-HDMI-A-1/edid; drm.debug=0x7f for kernel DRM debug log; common failures (wrong resolution, missing audio, no HDR, wrong DPI) mapped to EDID causes and remedies
- **Integrations**: Ch2 (KMS Connector Modes), Ch128 (DisplayPort MST), Ch140 (HDMI/DP Audio ELD), Ch158 (HDR on Linux), Ch181 (Display Standards), Ch182 (DDC I2C Channel), Ch184 (Embedded DisplayPort eDP), Ch187 (CEC Consumer Electronics Control)


### Chapter 184: Embedded DisplayPort (eDP) and Laptop Panel Management *(Part 27 — Display Hardware and Connectors)*

- eDP vs. external DisplayPort: reduced 400 mV PHY swing, ZIF connector, no hotplug (HPD repurposed for PSR IRQ), eDP version history 1.0–1.5 feature table
- DPCD register space and AUX channel protocol: 1 MB flat address map, key register table (0x000–0x1B0), half-duplex 1 Mbps AUX transactions, drm_dp_dpcd_read/write helpers, intel_dp_aux_xfer()
- PSR1 (Panel Self-Refresh, eDP 1.2): TCON local frame buffer, link standby vs. main-link off, intel_psr.c functions (intel_psr_enable/disable/invalidate/flush), frontbuffer tracking, AMD amdgpu_dm_psr.c
- PSR2 Selective Update (eDP 1.4): 64-px × 4-line SU granularity, VSC SDP dirty region encoding, PSR2 Selective Fetch (Gen12+), known HW quirks (Wa_16014451276, ADL-P SU region bug, DC6 exit instability)
- Panel Replay (eDP 1.5 / DP 2.1): CRC-gated activation via VSC SDP, DP_PANEL_REPLAY_CAP registers (0x1B0–0x1B2), fast-wake link re-establishment (<1 ms), Linux 6.3 support in intel_psr.c
- DRRS Dynamic Refresh Rate Switching (eDP 1.3): seamless vs. non-seamless modes, DPCD link-rate table (0x010–0x01F), intel_drrs.c delayed_work mechanism, pipeconf register vs. M/N divisor methods, VRR mutual exclusion
- Backlight control methods: PWM via pwm_backlight/pwm_bl.c, I2C controller ICs (TI LP8557, LM3630A), eDP AUX backlight (DPCD 0x720–0x723, intel_dp_aux_backlight.c, i915.enable_dpcd_backlight), ACPI video _BCM/_BCL fallback, IIO ALS integration
- DSC on eDP (eDP 1.4b+): bandwidth math for 4K@120 Hz, slice/bpp/chunk-size parameters, PPS negotiation via DSC SDP, intel_dsc.c (intel_dsc_compute_params), AMD dc_dsc_compute_config(), DSC-in-compressed-domain CRC for Panel Replay
- Panel power sequencing T1–T12: VDD-rise to backlight-on timing table, 500 ms mandatory T12 cycle, intel_pps.c (intel_pps_init/enable/disable/wait_panel_power_cycle), BIOS PP_ON_DELAYS/PP_OFF_DELAYS registers, Device Tree panel-edp.yaml bindings
- drm_panel subsystem: drm_panel_funcs (prepare/enable/disable/unprepare/get_modes), panel-simple.c for DT-described panels, drm_panel_bridge mapping bridge callbacks to panel lifecycle in atomic commits
- Bridge chips: TI SN65DSI86 DSI-to-eDP (ti-sn65dsi86.c, drm_bridge_funcs, EDID proxying via AUX master), Toshiba TC358775 DSI-to-LVDS; touch controllers via I2C HID (i2c-hid, ACPI _HID)
- Debugging tools: i915_edp_psr_status debugfs (PSR entry/exit counts, deepest sleep), intel_dp_dpcd (intel-gpu-tools), /sys/class/backlight/ brightness sysfs, drm.debug=0x1f kernel parameter, DPCD Panel Replay capability detection
- **Integrations**: Ch2 (KMS Atomic Modesetting), Ch3 (PSR2/Panel Replay as KMS features), Ch51 (GPU Power Management / RC6 / DC states), Ch126 (Hybrid Graphics Laptop Power Management), Ch181 (DisplayPort 2.1 Link Rates / UHBR), Ch183 (EDID over I2C-over-AUX), Ch188 (Display Power States — DC State Hierarchy)


### Chapter 185: Wireless Display Technologies on Linux *(Part 27 — Display Hardware and Connectors)*

- Wireless display paradigms overview: Miracast/Wi-Fi Direct (50-100ms, ~25Mbps), WiGig 60GHz (< 5ms, 7Gbps), and network streaming (VNC/RDP/WebRTC/Chromecast) — latency/bandwidth/range trade-offs table
- Miracast/WFD standard: Wi-Fi Direct P2P Group Owner negotiation, GO Intent values (0-15), WPS PBC/PIN provisioning, DHCP on 192.168.49.0/24
- RTSP M1-M16 control plane: capability negotiation (wfd_video_formats bitmask, wfd_audio_codecs, wfd_client_rtp_ports), SETUP/PLAY/UIBC back-channel messages on TCP port 7236
- Data plane: H.264/MPEG-TS/RTP over UDP; 188-byte MPEG-TS packets, RFC 2250 payload type 33, mtu=1316 (7xMPEG-TS per RTP), CEA resolution table up to 1920x1080p60
- MiracleCast architecture: miracle-wifid, miracle-sinkctl, miracle-gst, miracle-userctl daemons; wpa_supplicant P2P D-Bus API (fi.w1.wpa_supplicant1.Interface.P2PDevice), exclusive interface ownership constraint
- wpa_supplicant P2P depth: GO Negotiation Action frames, 802.11 P2P IE (OUI 50:6F:9A), persistent groups with Invitation Request/Response, wpa_ctrl C library for programmatic control
- WiGig IEEE 802.11ad/802.11ay: 60GHz mmWave (57-71GHz), GCMP cipher, mandatory SLS beamforming, 4x2.16GHz channels; wil6210 kernel driver at drivers/net/wireless/ath/wil6210/ for Qualcomm Sparrow/Talyn chipsets
- WiGig Display Extension (WDE) status: absent from Linux ecosystem as of 2026; wil6210 uses cfg80211 (not mac80211) with WMI firmware messages; requires wil6210.brd and wil6210.fw blobs
- VNC/RDP network display: wayvnc via wlr-export-dmabuf-v1/wlr-screencopy-v1; Weston RDP backend using FreeRDP; xrdp; GNOME Remote Desktop; KRdp for KDE Plasma 6; RDP RemoteFX AVC444 (H.264)
- Chromecast: CASTV2 protobuf protocol over TLS (port 8009), mDNS _googlecast._tcp discovery via Avahi, CastMessage namespaces, App ID 0F5096E8; mkchromecast with ffmpeg x11grab or PipeWire capture
- XDG ScreenCast portal: org.freedesktop.portal.ScreenCast D-Bus API, CreateSession/SelectSources/Start/OpenPipeWireRemote call sequence, pipewire-serial (v6) for stable stream identification, pipewiresrc GStreamer element
- GStreamer webrtcbin pipeline: ICE/DTLS-SRTP/RTCP-NACK, rtph264pay config-interval=1, vaapih264enc/vaav1enc, webrtcsink with built-in WHIP/WHEP signalling server on ports 8080/8443
- VA-API zero-copy DMA-BUF pipeline: DRM_IOCTL_PRIME_HANDLE_TO_FD export, vaCreateSurfaces with VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2/VADRMPRIMESurfaceDescriptor, vaapih264enc key properties (rate-control=cbr, max-bframes=0, quality-level)
- Latency budget analysis: VA-API HW encode 5-10ms vs x264 ultrafast 30-80ms; total ~36-46ms (VA-API) vs ~61-116ms (software); x264 tune=zerolatency disables lookahead and enables slice-level encoding
- **Integrations**: Ch22 (Wayland Compositors), Ch26 (VA-API Hardware Video Encode), Ch38 (PipeWire Video Pipeline), Ch57 (FFmpeg), Ch58 (GStreamer), Ch79 (Remote Display and Screen Casting), Ch123 (Screen Capture and Remote Desktop), Ch146 (WebCodecs and Browser Hardware Acceleration)


### Chapter 186: Video Pixel Formats and Display Signal Encoding *(Part XXVII — Display Hardware and Connectors)*

- Color space fundamentals — RGB vs YCbCr: BT.601/BT.709/BT.2020 luma coefficients, transfer functions (sRGB gamma, SMPTE ST 2084 PQ, ARIB STD-B67 HLG), four-component color space specification (primaries, white point, EOTF, YCbCr matrix)
- Chroma subsampling — 4:4:4 / 4:2:2 / 4:2:0 / 4:1:1 notation; NV12 semi-planar memory layout for 1920x1080; chroma siting conventions (MPEG-2 cosited vs JPEG interstitial) and DRM_FORMAT_MOD_CHROMASITING_COSITED
- Bit depth — 8-bit sRGB (DRM_FORMAT_XRGB8888, NV12), 10-bit HDR consumer via P010 (MSB-aligned 16-bit words), 12-bit Dolby Vision via DRM_FORMAT_P012/P016, True 10-bit vs 8-bit+FRC detection via edid-decode and DRM 'max bpc' connector property
- Quantization range — full (0-255) vs limited/studio swing (luma 16-235, chroma 16-240); mismatch symptoms (washed-out or crushed image); i915 'Broadcast RGB' connector property; amdgpu hdmi_avi_infoframe_set_quantization_range() with hardware quantization scaler
- DRM fourcc pixel format taxonomy — fourcc_code() macro in drm_fourcc.h; RGB formats (XRGB8888, ARGB8888, XRGB2101010, XBGR2101010); YUV packed (YUYV, UYVY, YVYU); planar/semi-planar (YUV420, NV12, NV21, P010, P016); drmModeGetPlane() enumeration and IN_FORMATS plane property blob
- V4L2 pixel formats and multiplanar API — v4l2_pix_format_mplane struct with colorspace/ycbcr_enc/quantization/xfer_func fields; v4l2_plane_pix_format per-plane stride/size; VIDIOC_ENUM_FMT / VIDIOC_TRY_FMT / VIDIOC_S_FMT three-step negotiation idiom for V4L2_PIX_FMT_NV12 and V4L2_PIX_FMT_P010
- HDR metadata — SMPTE ST 2086 static metadata fields (display_primaries, white_point, max_cll, max_fall); HDMI Dynamic Range and Mastering InfoFrame (type 0x87); DP HDR Metadata SDP (type 0x04); DRM hdr_output_metadata / hdr_metadata_infoframe UAPI structs; drmModeCreatePropertyBlob + drmModeAtomicCommit to set HDR connector property
- AVI InfoFrame color encoding — CTA-861-G type 0x82 v2 13-byte structure; Y1:Y0 colorspace bits (RGB/YUV422/YUV444/YUV420); C1:C0 colorimetry and EC2:EC0 extended colorimetry (BT.2020=100); Q1:Q0 RGB quantization; VIC codes; struct hdmi_avi_infoframe in include/linux/hdmi.h; hdmi_avi_infoframe_pack()
- Format negotiation pipeline — VA-API/V4L2 decoder DMA-BUF export → zwp_linux_dmabuf_v1 wl_buffer import → compositor IN_FORMATS overlay plane check → DRM_MODE_ATOMIC_TEST_ONLY commit → real scanout or GPU blit fallback; zwp_linux_dmabuf_feedback_v1 (protocol v4) (format,modifier) advertisement
- Display engine CSC and EOTF — amdgpu DPP unit: AMD_FMT_MOD deswizzle, BT.2020 YCbCr-to-RGB CSC matrix, inverse PQ EOTF in OGAM/DEGAM block, linear-light blending, output re-encoding; GPU blit fallback path for failed overlay atomic commits
- Worked example — 4K HDR10 HEVC decode: VA-API VCN/Quick Sync P010+AMD_FMT_MOD export → zwp_linux_buffer_params_v1_add() two-plane DMA-BUF import → atomic commit with hdr_output_metadata blob + DRM_MODE_COLORIMETRY_BT2020_YCC → amdgpu AVI InfoFrame (VIC 97, YUV420, BT.2020 ext colorimetry) + HDR InfoFrame 0x87 → DPP CSC/OGAM processing → panel tone mapping
- **Integrations**: Ch2 (KMS Planes), Ch3 (HDR KMS Properties), Ch26 (VA-API), Ch60 (Codec Compression), Ch74 (HDR Wide Color Gamut), Ch101 (Color Science / ICC Profiles), Ch142 (V4L2 Format Negotiation API), Ch158 (HDR Display Color Management), Ch162 (AFBC/DCC/CCS/UBWC Modifiers)


### Chapter 187: HDMI CEC and the Linux CEC Subsystem *(Part 27 — Display Hardware and Connectors)*

- CEC Protocol Fundamentals: single-wire open-drain serial bus on HDMI pin 13, 400 baud, pulse-width-modulated bit encoding (0.6 ms / 1.5 ms low pulses), 16-byte max frame with header, EOM, and ACK bits
- Physical and Logical Addresses: physical address extracted from EDID HDMI VSDB (OUI 0x000C03, dotted A.B.C.D notation); logical addresses 0–15 dynamically claimed by polling; 0xFFFF = invalid/unknown
- CEC Message Categories: One Touch Play (0x04 Image View On, 0x82 Active Source), Standby (0x36), Routing Control (0x80, 0x85, 0x86), Remote Control Pass-Through (0x44/0x45 User Control), System Audio (0x70–0x72), OSD and Device Discovery (0x46, 0x64, 0x83, 0x84, 0x8F/0x90)
- Linux Kernel CEC Subsystem: drivers/media/cec/core/ (cec-adap.c, cec-api.c, cec-core.c, cec-notifier.c, cec-pin.c); struct cec_adapter with transmit_queue/wait_queue; struct cec_adap_ops with adap_enable, adap_transmit, adap_log_addr, received callbacks; mainlined in Linux 4.8
- Core Driver API: cec_allocate_adapter(), cec_register_adapter(), cec_received_msg(), cec_transmit_done(), cec_s_phys_addr(), cec_s_phys_addr_from_edid(); CEC_CAP_* flags (PHYS_ADDR, LOG_ADDRS, TRANSMIT, MONITOR_ALL, RC)
- Hardware Driver Implementations: VC4 (Raspberry Pi 4, vc4_hdmi_cec.c, Linux 5.13); Amlogic Meson AO-CEC in always-on power domain (ao-cec.c, wake-from-standby); Rockchip RK3288 with Designware HDMI IP; common probe pattern with devm_request_threaded_irq and cec_notifier_cec_adap_register
- CEC Notifier Mechanism: cec-notifier.c bridges DRM subsystem (EDID owner) and media subsystem (CEC adapter); cec_notifier_conn_register() / cec_notifier_cec_adap_register() loosely couple drivers; cec_notifier_set_phys_addr_from_edid() parses CTA-861 HDMI VSDB; address replayed on late-probe
- Userspace API /dev/cecN: struct cec_msg (tx_ts, rx_ts, msg[16], reply, tx_status, tx_arb_lost_cnt); ioctls CEC_ADAP_G_CAPS, CEC_ADAP_S_LOG_ADDRS, CEC_TRANSMIT, CEC_RECEIVE, CEC_DQEVENT, CEC_S_MODE; mode flags INITIATOR, FOLLOWER, MONITOR, MONITOR_ALL
- cec-ctl and v4l-utils Tooling: cec-ctl --playback/--tv/--monitor-all/--monitor-pin; cec-compliance for spec conformance testing; cec-follower for device simulation; systemd unit for One Touch Play on system resume
- libcec Userspace Abstraction: Pulse-Eight cross-platform C++ ICECAdapter API over /dev/cecN, VideoCore firmware, USB dongles, I2C controllers; Kodi CEC peripheral plugin (PeripheralCecAdapter.cpp) sends Image View On, Active Source, Standby, translates 0x44 to Kodi remote events
- CEC Pin Framework and GPIO: cec-pin.c software bit-bang via hrtimer at ~0.6/1.5/3.7 ms boundaries; struct cec_pin_ops (read, low, high, enable_irq); cec-gpio platform driver wraps GPIO descriptor; DRM connector property CEC_PIN_IS_HIGH for register-readable CEC pins
- DisplayPort CEC Tunnelling: drm_dp_cec.c implements VESA DP CEC-Tunneling-over-AUX; DPCD registers 0x3000–0x301F (DP_CEC_TUNNELING_CONTROL, TX/RX_MESSAGE_BUFFER, LOGICAL_ADDRESS_MASK); drm_dp_cec_set_edid() / drm_dp_cec_unset_edid(); used by i915, Nouveau, AMDGPU; single-HDMI-device limitation
- Practical Automation and Troubleshooting: Home Assistant pycec integration via /dev/cecN; bus arbitration loss tracking (tx_arb_lost_cnt) with automatic retry; 0xFFFF diagnosis via edid-decode HDMI VSDB check and dmesg CEC notifier messages; HDMI switch EDID-rewriting requirements; CEC vs ARC/eARC pin coexistence (pin 13 vs pin 37)
- **Integrations**: Ch2 (KMS/DRM Connector Model — CEC_PIN_IS_HIGH property, EDID hotplug triggers notifier), Ch140 (HDMI ARC/eARC — System Audio Mode opcodes 0x70/0x72 coordination), Ch142 (V4L2 and Media Subsystem — /dev/cecN device node conventions, cec-compliance/v4l-utils shared infrastructure), Ch182 (HDMI Connector Physical Layer — pin 13 electrical spec, HPD pin 19 triggering EDID reads), Ch183 (EDID and DisplayID — CEC physical address in HDMI VSDB OUI 0x000C03, cec_get_edid_phys_addr())


### Chapter 188: Display Power States — DPMS, Panel Self-Refresh, and Display Idle Management *(Part XXVII — Display Hardware and Connectors)*

- Display power problem space: component power budget (backlight 2–4 W, controller 0.3–1 W, DRAM reads 0.1–0.5 W); five-level power reduction hierarchy from brightness dimming to CRTC disable
- VESA DPMS legacy: DRM_MODE_DPMS_ON/STANDBY/SUSPEND/OFF constants in drm_mode.h; drm_connector_funcs.dpms() callback; obsolete for digital panels, replaced by atomic CRTC active property
- KMS CRTC active boolean: drm_crtc_state.active; drmModeAtomicAddProperty() blank/unblank pattern; wlroots DRM backend drm_connector_set_dpms() in backend/drm/drm.c
- Intel Display C-states: DC_STATE_EN_DC3CO / DC5 / DC6 / DC9 constants in intel_display_power.c; DMC firmware; intel_display_power_get/put() power well reference counting; i915.enable_dc module parameter
- Intel DC5/DC6 preconditions and debugging: is_dc5_dc6_blocked(); PSR required for DC state entry; debugfs paths i915_display_info, i915_power_domain_info, i915_edp_psr_status; intel_gpu_top
- AMD DCN display power: DCCG/DISPCLK/DCFCLK/SOCCLK clock hierarchy; Display Mode Library (DML) in dc/dml/; SMU voltage rail coordination via amdgpu_dm.c; amdgpu.pg_mask / amdgpu.dcfeaturemask module parameters
- AMD MALL and SubVP: Memory Attached Last Level cache (32–64 MB); SubVP pre-fetches scanout into MALL enabling DRAM self-refresh; dc_debug_options flags force_disable_subvp / force_subvp_mclk_switch; DMUB firmware trace groups
- PSR as DC-state enabler: intel_psr_notify_dc5_dc6() / is_dc5_dc6_blocked() in intel_psr.c; PSR1 enables DC6; PSR2 Selective Update requires DRAM access so limits to DC5; Panel Replay CRC-gated activation and ALPM pairing
- Backlight and ALS: /sys/class/backlight/ sysfs interface; ACPI video _BCL/_BCM/_BQC vs. native GPU backlight; iio-sensor-proxy net.hadess.SensorProxy D-Bus interface; GNOME mutter backlight control (GNOME 49); OLED burn-in pixel shift
- power-profiles-daemon display integration: net.hadess.PowerProfiles D-Bus; platform_profile sysfs; power-saver triggers AMDGPU panel power savings and DPM clock reduction; DRRS/PSR idle timeout per profile; TLP conflict
- Compositor idle management: ext-idle-notify-v1 Wayland protocol; wlr_idle_notifier_v1 / wlr_idle_notification_v1_create(); staged dim→blank sequence; zwp_idle_inhibit_manager_v1 and systemd inhibitor locks; swayidle, GNOME IdleMonitor, KDE PowerDevil
- ACPI S0ix Modern Standby: s2idle vs. deep (S3) via /sys/power/mem_sleep; display shutdown prerequisite for S0ix entry; DC9 (Intel) / DMUB suspend (AMD) in GPU .runtime_suspend() path; S0ix resume sequence; pmc_core/slp_s0_residency_usec debugging
- **Integrations**: Ch2 (KMS/DRM Architecture), Ch3 (Atomic Modesetting), Ch21 (wlroots Compositor Internals), Ch22 (KDE Plasma and GNOME Shell), Ch51 (GPU Power Management and Runtime PM), Ch126 (Hybrid Graphics and Optimus), Ch184 (Embedded DisplayPort and Laptop Panel Management)

