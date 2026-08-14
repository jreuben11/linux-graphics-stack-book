# Index — The Linux Graphics Stack

A complete navigational table of contents for every part, chapter, and appendix in the book. For the project description, audiences, and contribution guide, see [README.md](README.md).

- [Introduction](intro.md)
- [Conclusion](conclusion.md)
- [Master outline (plan.md)](plan.md)

**268 chapters + 21 appendices across 27 parts.**

## Contents

- [Part I — The Kernel Layer](#part-i--the-kernel-layer)
- [Part II — GPU Drivers](#part-ii--gpu-drivers)
- [Part III — The Open NVIDIA Stack](#part-iii--the-open-nvidia-stack)
- [Part IV — Mesa Architecture](#part-iv--mesa-architecture)
- [Part V — Mesa GPU Drivers](#part-v--mesa-gpu-drivers)
- [Part VI-A — Wayland Protocol and Compositor Architecture](#part-vi-a--wayland-protocol-and-compositor-architecture)
- [Part VI-B — Display Services, Input, and Color](#part-vi-b--display-services-input-and-color)
- [Part VII-A — GPU APIs and Extended Reality](#part-vii-a--gpu-apis-and-extended-reality)
- [Part VII-B — Multimedia Frameworks and Desktop Integration](#part-vii-b--multimedia-frameworks-and-desktop-integration)
- [Part VII-C — Desktop Frameworks](#part-vii-c--desktop-frameworks)
- [Part VIII — The Gaming Layer](#part-viii--the-gaming-layer)
- [Part IX — Tooling & Contributing](#part-ix--tooling--contributing)
- [Part X — The Browser Rendering Stack](#part-x--the-browser-rendering-stack)
- [Part XI — Engines & Creative Tools](#part-xi--engines--creative-tools)
- [Part XII — Terminal Graphics](#part-xii--terminal-graphics)
- [Part XIII — Video Streaming on Linux](#part-xiii--video-streaming-on-linux)
- [Part XIV — The Khronos Extended Ecosystem](#part-xiv--the-khronos-extended-ecosystem)
- [Part XV — The NVIDIA Proprietary Graphics Stack](#part-xv--the-nvidia-proprietary-graphics-stack)
- [Part XVI — The Intel Open Graphics Stack](#part-xvi--the-intel-open-graphics-stack)
- [Part XVII — The AMD Developer Ecosystem](#part-xvii--the-amd-developer-ecosystem)
- [Part XVIII — Rendering Abstraction Libraries](#part-xviii--rendering-abstraction-libraries)
- [Part XIX — Android Graphics](#part-xix--android-graphics)
- [Part XX — AI/ML Inference on Linux](#part-xx--aiml-inference-on-linux)
- [Part XXI — Platform, Legacy, and History](#part-xxi--platform-legacy-and-history)
- [Part XXVII — Display Hardware, Connectors, and Signal Standards](#part-xxvii--display-hardware-connectors-and-signal-standards)
- [Part XXVIII — Linux Multimedia: Audio, Communication, and Music Production](#part-xxviii--linux-multimedia-audio-communication-and-music-production)
- [Part XXIX — Graphics Algorithms](#part-xxix--graphics-algorithms)
- [Appendices](#appendices)

---

### Part I — The Kernel Layer

- [Part Overview](chapters/part-01-kernel-layer/part-intro.md)
- [Ch 1: DRM Architecture & the Driver Model](chapters/part-01-kernel-layer/ch01-drm-architecture.md)
- [Ch 2: KMS: The Display Pipeline](chapters/part-01-kernel-layer/ch02-kms-display-pipeline.md)
- [Ch 3: Advanced Display Features](chapters/part-01-kernel-layer/ch03-advanced-display-features.md)
- [Ch 4: GPU Memory Management](chapters/part-01-kernel-layer/ch04-gpu-memory-management.md)
- [Ch 51: GPU Power Management and Thermal](chapters/part-01-kernel-layer/ch51-gpu-power-management.md)
- [Ch 102: The DRM GPU Scheduler and Multi-Process Fairness](chapters/part-01-kernel-layer/ch102-drm-gpu-scheduler.md)
- [Ch 120: GPU Memory Management Internals — TTM, GEM, and BAR](chapters/part-01-kernel-layer/ch120-gpu-memory-management.md)
- [Ch 121: DRM Lease and VR Direct Display](chapters/part-01-kernel-layer/ch121-drm-lease-vr-direct-display.md)
- [Ch 129: GPU Firmware Deep Dive](chapters/part-01-kernel-layer/ch129-gpu-firmware.md)
- [Ch 139: DRM Hardware Overlay Planes and Composition Bypass](chapters/part-01-kernel-layer/ch139-drm-hardware-planes.md)
- [Ch 144: Boot Graphics Pipeline: From Firmware to KMS Handoff](chapters/part-01-kernel-layer/ch144-boot-graphics-pipeline.md)
- [Ch 149: GPU Hang Detection and Recovery](chapters/part-01-kernel-layer/ch149-gpu-hang-detection-recovery.md)
- [Ch 162: Framebuffer Compression: AFBC, DCC, CCS, and UBWC](chapters/part-01-kernel-layer/ch162-framebuffer-compression.md)
- [Ch 163: VKMS and Virtual Display Drivers for Testing](chapters/part-01-kernel-layer/ch163-vkms-virtual-display.md)

[↑ Contents](#contents)

---

### Part II — GPU Drivers

- [Part Overview](chapters/part-02-gpu-drivers/part-intro.md)
- [Ch 5: x86 GPU Drivers](chapters/part-02-gpu-drivers/ch05-x86-gpu-drivers.md)
- [Ch 6: ARM & Embedded GPU Drivers](chapters/part-02-gpu-drivers/ch06-arm-embedded-gpu-drivers.md)
- [Ch 49: Multi-GPU and PRIME Render Offload](chapters/part-02-gpu-drivers/ch49-multi-gpu-prime.md)
- [Ch 73: Asahi Linux and the Apple Silicon AGX Driver](chapters/part-02-gpu-drivers/ch73-asahi-apple-silicon.md)
- [Ch 90: Open ARM GPU Drivers — Lima, Panfrost, and Panthor](chapters/part-02-gpu-drivers/ch90-panfrost-panthor-lima.md)
- [Ch 92: The Raspberry Pi GPU Stack — VideoCore and V3D](chapters/part-02-gpu-drivers/ch92-raspberry-pi-videocore.md)
- [Ch 99: Automotive and Embedded Linux Graphics](chapters/part-02-gpu-drivers/ch99-automotive-embedded-graphics.md)
- [Ch 100: etnaviv: The Vivante GPU Open Driver](chapters/part-02-gpu-drivers/ch100-etnaviv-vivante.md)
- [Ch 116: RISC-V GPU Drivers](chapters/part-02-gpu-drivers/ch116-riscv-gpu-drivers.md)
- [Ch 126: Hybrid Graphics and Laptop Power Management](chapters/part-02-gpu-drivers/ch126-hybrid-graphics-laptop.md)
- [Ch 155: USB DisplayLink and the evdi Virtual DRM Driver](chapters/part-02-gpu-drivers/ch155-usb-displaylink-evdi.md)
- [Ch 160: Freedreno, Turnip, and the Qualcomm Adreno Driver](chapters/part-02-gpu-drivers/ch160-freedreno-turnip-adreno.md)
- [Ch 169: Snapdragon X Elite on Linux — Adreno X1-85, freedreno, and the Arm Laptop Era](chapters/part-02-gpu-drivers/ch169-snapdragon-x-elite-linux.md)
- [Ch 172: eGPU on Linux — Thunderbolt, USB4, and PCIe Hot-Plug](chapters/part-02-gpu-drivers/ch172-egpu-thunderbolt-usb4.md)
- [Ch 179: The Linux `accel` Subsystem: NPU and AI Accelerator Drivers](chapters/part-02-gpu-drivers/ch179-linux-accel-subsystem.md)
- [Ch 196: The GPU as Embedded Computer — Firmware-as-OS Architecture Across Vendors](chapters/part-02-gpu-drivers/ch196-gpu-firmware-as-os.md)

[↑ Contents](#contents)

---

### Part III — The Open NVIDIA Stack

- [Part Overview](chapters/part-03-nouveau-story/part-intro.md)
- [Ch 7: Reverse Engineering NVIDIA: History and Methodology](chapters/part-03-nouveau-story/ch07-reverse-engineering-nvidia.md)
- [Ch 8: The Nouveau Kernel Driver: nvkm Architecture](chapters/part-03-nouveau-story/ch08-nouveau-kernel-driver.md)
- [Ch 9: GSP-RM, Firmware, and the nvidia-open Connection](chapters/part-03-nouveau-story/ch09-gsp-rm-firmware.md)
- [Ch 10a: Nova — The Rust NVIDIA Kernel Driver](chapters/part-03-nouveau-story/ch10a-nova-rust-nvidia-driver.md)
- [Ch 10b: NVK: Building a Vulkan Driver from Scratch](chapters/part-03-nouveau-story/ch10b-nvk-vulkan-driver.md)
- [Ch 11: Display, Reclocking, and Power Management](chapters/part-03-nouveau-story/ch11-display-reclocking-power.md)
- [Ch 118: NAK — The Nouveau/NVK Rust Shader Compiler](chapters/part-03-nouveau-story/ch118-nak-rust-shader-compiler.md)

[↑ Contents](#contents)

---

### Part IV — Mesa Architecture

- [Part Overview](chapters/part-04-mesa-architecture/part-intro.md)
- [Ch 12: The Mesa Loader and Driver Dispatch](chapters/part-04-mesa-architecture/ch12-mesa-loader-dispatch.md)
- [Ch 13: Gallium3D: The OpenGL State Tracker](chapters/part-04-mesa-architecture/ch13-gallium3d.md)
- [Ch 14: NIR: Mesa's Shader Intermediate Representation](chapters/part-04-mesa-architecture/ch14-nir-shader-ir.md)
- [Ch 15: ACO: AMD's Optimising Compiler](chapters/part-04-mesa-architecture/ch15-aco-compiler.md)
- [Ch 16: Mesa's Vulkan Common Infrastructure](chapters/part-04-mesa-architecture/ch16-mesa-vulkan-common.md)
- [Ch 17: Software Renderers](chapters/part-04-mesa-architecture/ch17-software-renderers.md)
- [Ch 77: Shader Source-to-ISA: The Complete Compilation Toolchain](chapters/part-04-mesa-architecture/ch77-shader-toolchain.md)
- [Ch 91: MLIR and the Emerging GPU Compiler Infrastructure](chapters/part-04-mesa-architecture/ch91-mlir-gpu-compilation.md)
- [Ch 119: Zink — OpenGL on Vulkan](chapters/part-04-mesa-architecture/ch119-zink-opengl-on-vulkan.md)
- [Ch 156: Mesa Nine: The Direct3D 9 State Tracker for Gallium](chapters/part-04-mesa-architecture/ch156-mesa-nine-direct3d9.md)
- [Ch 159: The Vulkan–Mesa–DRM Stack: A Full Vertical Slice](chapters/part-04-mesa-architecture/ch159-vulkan-mesa-drm-nvidia-stack.md)

[↑ Contents](#contents)

---

### Part V — Mesa GPU Drivers

- [Part Overview](chapters/part-05-mesa-gpu-drivers/part-intro.md)
- [Ch 18: Vulkan Drivers](chapters/part-05-mesa-gpu-drivers/ch18-vulkan-drivers.md)
- [Ch 19: OpenGL and Compatibility Drivers](chapters/part-05-mesa-gpu-drivers/ch19-opengl-compatibility-drivers.md)
- [Ch 177: NVK — NVIDIA Vulkan in Mesa](chapters/part-05-mesa-gpu-drivers/ch177-nvk-nvidia-vulkan.md)

[↑ Contents](#contents)

---

### Part VI-A — Wayland Protocol and Compositor Architecture

- [Part Overview](chapters/part-06a-wayland-compositor/part-intro.md)
- [Ch 20: Wayland Protocol Fundamentals](chapters/part-06a-wayland-compositor/ch20-wayland-protocol-fundamentals.md)
- [Ch 21: Building Compositors with wlroots](chapters/part-06a-wayland-compositor/ch21-building-compositors-wlroots.md)
- [Ch 22: Production Compositors](chapters/part-06a-wayland-compositor/ch22-production-compositors.md)
- [Ch 23: Legacy and Sandboxed App Support](chapters/part-06a-wayland-compositor/ch23-legacy-sandboxed-app-support.md)
- [Ch 46: The Evolving Wayland Protocol Ecosystem](chapters/part-06a-wayland-compositor/ch46-wayland-protocol-ecosystem.md)
- [Ch 130: Wayland Protocol Extension Development](chapters/part-06a-wayland-compositor/ch130-wayland-protocol-dev.md)
- [Ch 132: Wayland Security](chapters/part-06a-wayland-compositor/ch132-wayland-security.md)
- [Ch 138: Wayland Fractional Scaling and HiDPI](chapters/part-06a-wayland-compositor/ch138-wayland-fractional-scaling.md)
- [Ch 145: XWayland: Architecture and the X11-to-Wayland Bridge](chapters/part-06a-wayland-compositor/ch145-xwayland-architecture.md)
- [Ch 151: Wayland Text Input and Input Method Editors](chapters/part-06a-wayland-compositor/ch151-wayland-text-input-ime.md)
- [Ch 175: Linux Compositor Accessibility: AT-SPI2, Screen Readers, and the Wayland Gap](chapters/part-06a-wayland-compositor/ch175-atspie2-compositor-accessibility.md)
- [Ch 207: xdg-desktop-portal — Sandboxed Desktop Integration](chapters/part-06a-wayland-compositor/ch207-xdg-desktop-portal.md)

[↑ Contents](#contents)

---

### Part VI-B — Display Services, Input, and Color

- [Part Overview](chapters/part-06b-display-services/part-intro.md)
- [Ch 53: Display Calibration and colord](chapters/part-06b-display-services/ch53-display-calibration-colord.md)
- [Ch 54: The Linux Input Stack](chapters/part-06b-display-services/ch54-linux-input-stack.md)
- [Ch 74: HDR and Wide Color Gamut on Linux](chapters/part-06b-display-services/ch74-hdr-wide-color-gamut.md)
- [Ch 75: Explicit GPU Synchronisation](chapters/part-06b-display-services/ch75-explicit-gpu-sync.md)
- [Ch 101: Color Science and the ICC Profile Pipeline](chapters/part-06b-display-services/ch101-color-science-icc.md)
- [Ch 105: Font Rendering — FreeType2, HarfBuzz, and the Text Pipeline](chapters/part-06b-display-services/ch105-font-rendering.md)
- [Ch 112: Variable Refresh Rate — FreeSync, G-Sync, and Frame Pacing](chapters/part-06b-display-services/ch112-vrr-freesync-frame-pacing.md)
- [Ch 123: Screen Capture and Remote Desktop on Linux](chapters/part-06b-display-services/ch123-screen-capture-remote-desktop.md)
- [Ch 128: DisplayPort MST and Multi-Monitor Topology](chapters/part-06b-display-services/ch128-displayport-mst.md)
- [Ch 131: Touch, Stylus, and Tablet Input on Wayland](chapters/part-06b-display-services/ch131-touch-tablet-wayland.md)
- [Ch 140: HDMI and DisplayPort Audio on Linux](chapters/part-06b-display-services/ch140-hdmi-dp-audio.md)
- [Ch 158: HDR and Display Color Management on Linux](chapters/part-06b-display-services/ch158-hdr-linux-display.md)
- [Ch 194: Cross-Stack Integration — Protocols, Synchronisation, and the Coordination Layer](chapters/part-06b-display-services/ch194-cross-stack-integration.md)
- [Ch 198: D-Bus, dbus-broker, and Modern Linux IPC](chapters/part-06b-display-services/ch198-dbus-modern-linux-ipc.md)

[↑ Contents](#contents)

---

### Part VII-A — GPU APIs and Extended Reality

- [Part Overview](chapters/part-07a-gpu-apis/part-intro.md)
- [Ch 24: Vulkan and EGL for Application Developers](chapters/part-07a-gpu-apis/ch24-vulkan-egl-application-developers.md)
- [Ch 25: GPU Compute](chapters/part-07a-gpu-apis/ch25-gpu-compute.md)
- [Ch 26: Hardware Video](chapters/part-07a-gpu-apis/ch26-hardware-video.md)
- [Ch 27: VR & AR](chapters/part-07a-gpu-apis/ch27-vr-ar.md)
- [Ch 76: Modern Vulkan Extensions](chapters/part-07a-gpu-apis/ch76-modern-vulkan-extensions.md)
- [Ch 106: The Vulkan Memory Model — Formal Execution and Memory Ordering](chapters/part-07a-gpu-apis/ch106-vulkan-memory-model.md)
- [Ch 127: Mesh Shaders and Variable Rate Shading](chapters/part-07a-gpu-apis/ch127-mesh-shaders-vrs.md)
- [Ch 133: Vulkan Compute Queues and Task Graphs](chapters/part-07a-gpu-apis/ch133-vulkan-compute-queues.md)
- [Ch 135: Vulkan Ray Tracing on Linux](chapters/part-07a-gpu-apis/ch135-vulkan-ray-tracing.md)
- [Ch 141: Vulkan Cooperative Matrices and GPU ML Acceleration](chapters/part-07a-gpu-apis/ch141-vulkan-cooperative-matrices.md)
- [Ch 148: Vulkan Synchronisation: A Complete Developer Reference](chapters/part-07a-gpu-apis/ch148-vulkan-synchronisation.md)
- [Ch 150: EGL Architecture and DMA-BUF Integration](chapters/part-07a-gpu-apis/ch150-egl-architecture-dmabuf.md)
- [Ch 152: The Rust GPU Ecosystem: ash, wgpu, naga, and Bevy](chapters/part-07a-gpu-apis/ch152-rust-gpu-ecosystem.md)
- [Ch 154: GPU-Driven Rendering: Indirect Draw, Culling, and Mesh Shaders](chapters/part-07a-gpu-apis/ch154-gpu-driven-rendering.md)
- [Ch 157: Vulkan Descriptor Binding: Sets, Push Descriptors, and Descriptor Buffers](chapters/part-07a-gpu-apis/ch157-vulkan-descriptor-binding.md)
- [Ch 165: Vulkan Video: Hardware Decode and Encode via the Vulkan API](chapters/part-07a-gpu-apis/ch165-vulkan-video.md)
- [Ch 173: VK_EXT_shader_object — Pipeline-Free Shader Binding in Vulkan](chapters/part-07a-gpu-apis/ch173-vk-ext-shader-object.md)
- [Ch 192: GPU-Generated Commands — VK_EXT_device_generated_commands and Work Graphs](chapters/part-07a-gpu-apis/ch192-vk-ext-device-generated-commands.md)

[↑ Contents](#contents)

---

### Part VII-B — Multimedia Frameworks and Desktop Integration

- [Part Overview](chapters/part-07b-multimedia-frameworks/part-intro.md)
- [Ch 38: PipeWire and the Video Session Layer](chapters/part-07b-multimedia-frameworks/ch38-pipewire.md)
- [Ch 38b: ALSA — The Linux Audio Subsystem](chapters/part-07b-multimedia-frameworks/ch38b-alsa-linux-audio-subsystem.md)
- *(Ch 39: Qt and GTK GPU Rendering has been absorbed into Part VII-C — see Ch39a–Ch39i under Desktop Frameworks below)*
- [Ch 47: Font and Text Rendering Pipeline](chapters/part-07b-multimedia-frameworks/ch47-font-text-rendering.md)
- [Ch 50: Vulkan Video Extensions](chapters/part-07b-multimedia-frameworks/ch50-vulkan-video.md)
- [Ch 96: libcamera and the Linux Camera Stack](chapters/part-07b-multimedia-frameworks/ch96-libcamera.md)
- [Ch 111: Flatpak Graphics — GPU Access in Sandboxed Applications](chapters/part-07b-multimedia-frameworks/ch111-flatpak-graphics.md)
- [Ch 114: OpenCV and GPU-Accelerated Computer Vision on Linux](chapters/part-07b-multimedia-frameworks/ch114-opencv-gpu-vision.md)
- [Ch 206: SDL3 — Cross-Platform Multimedia Integration on Linux](chapters/part-07b-multimedia-frameworks/ch206-sdl-multimedia.md)
- [Ch 209: OpenSLAM — Classical and Graph-Based SLAM on the Linux Stack](chapters/part-07b-multimedia-frameworks/ch209-openslam.md)
- [Ch 210: SLAM Theory and State of the Art](chapters/part-07b-multimedia-frameworks/ch210-slam-theory-sota.md)
- [Ch 211: ROS 2 Multimodal Sensor and Perception Pipeline](chapters/part-07b-multimedia-frameworks/ch211-ros2-sensor-perception-pipeline.md)
- [Ch 211b: NVIDIA Isaac Sim, Isaac Lab, and the GR00T Foundation-Model Family](chapters/part-07b-multimedia-frameworks/ch211b-isaac-sim-isaac-lab-groot.md)

[↑ Contents](#contents)

---

### Part VII-C — Desktop Frameworks

- [Part Overview](chapters/part-07c-desktop-frameworks/part-intro.md)
- [Ch 39a: Qt6 — Scene Graph, QRhi, and Wayland Integration](chapters/part-07c-desktop-frameworks/ch39a-qt6.md)
- [Ch 39b: KDE Frameworks 6 and KWin](chapters/part-07c-desktop-frameworks/ch39b-kde.md)
- [Ch 39c: GTK4 — GskRenderer, libadwaita, and WebKitGTK](chapters/part-07c-desktop-frameworks/ch39c-gtk4.md)
- [Ch 39d: GNOME Shell and Mutter](chapters/part-07c-desktop-frameworks/ch39d-gnome.md)
- [Ch 39e: iced — Rust-Native GUI with wgpu](chapters/part-07c-desktop-frameworks/ch39e-iced.md)
- [Ch 39f: libcosmic and the COSMIC Desktop](chapters/part-07c-desktop-frameworks/ch39f-libcosmic.md)
- [Ch 39g: Flutter on Linux — Impeller and the GTK Embedder](chapters/part-07c-desktop-frameworks/ch39g-flutter.md)
- [Ch 39h: Dear ImGui — Immediate-Mode GUI for Linux Applications](chapters/part-07c-desktop-frameworks/ch39h-dear-imgui.md)
- [Ch 39i: Desktop Framework Comparisons](chapters/part-07c-desktop-frameworks/ch39i-desktop-framework-comparisons.md)

[↑ Contents](#contents)

---

### Part VIII — The Gaming Layer

- [Part Overview](chapters/part-08-gaming-layer/part-intro.md)
- [Ch 28: Windows Compatibility](chapters/part-08-gaming-layer/ch28-windows-compatibility.md)
- [Ch 29: Upscaling, Effects & Overlays](chapters/part-08-gaming-layer/ch29-upscaling-effects-overlays.md)
- [Ch 56: Ray Tracing on Linux](chapters/part-08-gaming-layer/ch56-ray-tracing-linux.md)
- [Ch 78: Gamescope and the Steam Deck: A Complete Gaming Graphics Stack](chapters/part-08-gaming-layer/ch78-gamescope-steam-deck.md)
- [Ch 104: DXVK and VKD3D-Proton — D3D-to-Vulkan Translation](chapters/part-08-gaming-layer/ch104-dxvk-vkd3d-proton.md)
- [Ch 167: NTSYNC — NT Synchronization Primitives in the Linux Kernel](chapters/part-08-gaming-layer/ch167-ntsync-nt-synchronization-linux.md)
- [Ch 171: Linux Gaming Anti-Cheat — EasyAntiCheat, BattlEye, and the Ring-0 Problem](chapters/part-08-gaming-layer/ch171-linux-gaming-anti-cheat.md)

[↑ Contents](#contents)

---

### Part IX — Tooling & Contributing

- [Part Overview](chapters/part-09-tooling-contributing/part-intro.md)
- [Ch 30: Debugging & Profiling](chapters/part-09-tooling-contributing/ch30-debugging-profiling.md)
- [Ch 31: Conformance & Regression Testing](chapters/part-09-tooling-contributing/ch31-conformance-regression-testing.md)
- [Ch 32: Contributing to the Linux Graphics Stack](chapters/part-09-tooling-contributing/ch32-contributing.md)
- [Ch 55: GPU Containers and Cloud Compute](chapters/part-09-tooling-contributing/ch55-gpu-containers-cloud.md)
- [Ch 79: Remote Display, Screen Casting, and GPU-Accelerated Game Streaming](chapters/part-09-tooling-contributing/ch79-remote-display-streaming.md)
- [Ch 80: GPU Security: Isolation, Content Protection, and Confidential Computing](chapters/part-09-tooling-contributing/ch80-gpu-security.md)
- [Ch 89: GPU Virtualization in Depth](chapters/part-09-tooling-contributing/ch89-gpu-virtualization.md)
- [Ch 93: GPU Performance Analysis Methodology](chapters/part-09-tooling-contributing/ch93-gpu-performance-analysis.md)
- [Ch 107: Headless Rendering — Offscreen, CI, and Server Workloads](chapters/part-09-tooling-contributing/ch107-headless-rendering.md)
- [Ch 109: Mesa Testing — piglit, dEQP, and Continuous Integration](chapters/part-09-tooling-contributing/ch109-mesa-testing-ci.md)
- [Ch 122: DKMS and Out-of-Tree GPU Kernel Modules](chapters/part-09-tooling-contributing/ch122-dkms-out-of-tree-modules.md)
- [Ch 125: RenderDoc on Linux](chapters/part-09-tooling-contributing/ch125-renderdoc-linux.md)
- [Ch 136: WSL2 Linux Graphics — dxgkrnl and Mesa D3D12](chapters/part-09-tooling-contributing/ch136-wsl2-linux-graphics.md)
- [Ch 137: GPU Performance Profiling — RGP, GPA, and VK_EXT_performance_query](chapters/part-09-tooling-contributing/ch137-gpu-performance-profiling.md)
- [Ch 153: OBS Studio GPU Pipeline: Capture, Encode, and Stream](chapters/part-09-tooling-contributing/ch153-obs-studio-gpu-pipeline.md)
- [Ch 180: GPU Reverse Engineering — Tools, Methodology, and Case Studies](chapters/part-09-tooling-contributing/ch180-gpu-reverse-engineering.md)

[↑ Contents](#contents)

---

### Part X — The Browser Rendering Stack

- [Part Overview](chapters/part-10-browser-rendering-stack/part-intro.md)
- [Ch 33: Chromium's Multi-Process GPU Architecture](chapters/part-10-browser-rendering-stack/ch33-chromium-gpu-architecture.md)
- [Ch 34: ANGLE — WebGL on Linux](chapters/part-10-browser-rendering-stack/ch34-angle-webgl.md)
- [Ch 35: Dawn and WebGPU](chapters/part-10-browser-rendering-stack/ch35-dawn-webgpu.md)
- [Ch 36: The Chromium Compositor — CC and Viz](chapters/part-10-browser-rendering-stack/ch36-chromium-compositor.md)
- [Ch 37: Skia and 2D Rendering](chapters/part-10-browser-rendering-stack/ch37-skia-2d-rendering.md)
- [Ch 52: Firefox and WebRender](chapters/part-10-browser-rendering-stack/ch52-firefox-webrender.md)
- [Ch 98: WebAssembly and WebGPU as a Deployment Target](chapters/part-10-browser-rendering-stack/ch98-webassembly-webgpu.md)
- [Ch 146: WebCodecs and Browser Hardware Acceleration](chapters/part-10-browser-rendering-stack/ch146-webcodecs-browser-hardware-acceleration.md)
- [Ch 147: Chrome and Firefox Hardware Video Decode via VA-API](chapters/part-10-browser-rendering-stack/ch147-chrome-firefox-vaapi-video-decode.md)
- [Ch 168: WebNN — The Web Neural Network API](chapters/part-10-browser-rendering-stack/ch168-webnn-web-neural-network-api.md)
- [Ch 193: Tauri — Rust-Native Desktop Applications via WebKitGTK](chapters/part-10-browser-rendering-stack/ch193-tauri-webkitgtk-desktop.md)
- [Ch 195: Browser Image Formats — Decode Pipelines, Compression Mechanisms, and HDR](chapters/part-10-browser-rendering-stack/ch195-browser-image-formats.md)
- [Ch 203: WebXR — Browser-Based Immersive Experiences on Linux](chapters/part-10-browser-rendering-stack/ch203-webxr.md)

[↑ Contents](#contents)

---

### Part XI — Engines & Creative Tools

- [Part Overview](chapters/part-11-engine-creative-tools/part-intro.md)
- [Ch 40: Bevy and wgpu](chapters/part-11-engine-creative-tools/ch40-bevy-wgpu.md)
- [Ch 41: Godot 4 RenderingDevice](chapters/part-11-engine-creative-tools/ch41-godot4-rendering-device.md)
- [Ch 42: Blender GPU — Cycles and EEVEE](chapters/part-11-engine-creative-tools/ch42-blender-gpu.md)
- [Ch 97: Unreal Engine 5 on Linux](chapters/part-11-engine-creative-tools/ch97-unreal-engine-5.md)
- [Ch 176: OpenCASCADE Technology — The BRep Kernel and 3D Visualization Stack](chapters/part-11-engine-creative-tools/ch176-opencascade-cad-kernel.md)
- [Ch 176a: State-of-the-Art CAD AI — Generative Models, Reverse Engineering, and Agentic Design](chapters/part-11-engine-creative-tools/ch176a-sota-cad-ai.md)
- [Ch 190: VTK — Scientific Visualization on the Linux Graphics Stack](chapters/part-11-engine-creative-tools/ch190-vtk-scientific-visualization.md)
- [Ch 205: AI-Driven 3D Creation — Blender MCP, Claude Code, and Generative Tools](chapters/part-11-engine-creative-tools/ch205-blender-ai-mcp.md)
- [Ch 205a: Programmable Games and Competitive-Code Sandboxes](chapters/part-11-engine-creative-tools/ch205a-programmable-games.md)
- [Ch 205b: AI Agents in Games — RL Environments and LLM-Driven NPCs](chapters/part-11-engine-creative-tools/ch205b-ai-agents-in-games.md)
- [Ch 205c: Open-Source 2D Simulation-Game Engines](chapters/part-11-engine-creative-tools/ch205c-2d-simulation-engines.md)
- [Ch 205d: Modding Architectures — Scripting, Sandboxing, and Hot-Reload on Linux](chapters/part-11-engine-creative-tools/ch205d-modding-architectures.md)
- [Ch 205e: FOSS Simulation-Game Frameworks — Data Models, Rules Engines, and Modding as SDKs](chapters/part-11-engine-creative-tools/ch205e-simulation-game-frameworks.md)
- [Ch 205f: Artificial Life on the GPU — Cellular Automata, Lenia, and Digital-Organism Simulators](chapters/part-11-engine-creative-tools/ch205f-artificial-life-gpu.md)
- [Ch 241: GIMP, Krita, and darktable — GPU-Accelerated Raster and RAW Photo Editing on Linux](chapters/part-11-engine-creative-tools/ch241-gimp-krita-darktable-photo-editing.md)
- [Ch 242: DaVinci Resolve — Professional Color Grading and the Linux GPU Pipeline](chapters/part-11-engine-creative-tools/ch242-davinci-resolve-color-grading.md)

[↑ Contents](#contents)

---

### Part XII — Terminal Graphics

- [Part Overview](chapters/part-12-terminal-graphics/part-intro.md)
- [Ch 43: Terminal Pixel Protocols — Sixel, Kitty, and iTerm2](chapters/part-12-terminal-graphics/ch43-terminal-pixel-protocols.md)
- [Ch 44: Terminal GPU Rendering Architectures](chapters/part-12-terminal-graphics/ch44-terminal-gpu-rendering.md)
- [Ch 45: Terminal Integration with the Compositor Stack](chapters/part-12-terminal-graphics/ch45-terminal-compositor-integration.md)
- [Ch 174: WezTerm and Alacritty — GPU Terminal Rendering Architectures](chapters/part-12-terminal-graphics/ch174-wezterm-alacritty-architecture.md)
- [Ch 178: The PTY/TTY Kernel Layer and Line Disciplines](chapters/part-12-terminal-graphics/ch178-pty-tty-kernel-layer.md)
- [Ch 192: Ratatui — The Rust TUI Application Framework](chapters/part-12-terminal-graphics/ch192-ratatui-tui-framework.md)

[↑ Contents](#contents)

---

### Part XIII — Video Streaming on Linux

- [Part Overview](chapters/part-13-video-streaming/part-intro.md)
- [Ch 57: FFmpeg Architecture and Programming](chapters/part-13-video-streaming/ch57-ffmpeg.md)
- [Ch 58: GStreamer: Pipeline-Based Multimedia](chapters/part-13-video-streaming/ch58-gstreamer.md)
- [Ch 59: NVIDIA DeepStream SDK](chapters/part-13-video-streaming/ch59-deepstream.md)
- [Ch 60: Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs](chapters/part-13-video-streaming/ch60-video-codecs.md)
- [Ch 60b: Video Streaming Protocols and Adaptive Bitrate Delivery](chapters/part-13-video-streaming/ch60b-video-streaming-protocols.md)
- [Ch 60c: WebRTC Server Infrastructure on Linux](chapters/part-13-video-streaming/ch60c-webrtc-server-infrastructure.md)
- [Ch 60d: BitTorrent Adaptive Streaming on Linux — libtorrent, WebTorrent, and the GPU Decode Pipeline](chapters/part-13-video-streaming/ch60d-bittorrent-streaming.md)
- [Ch 60e: Media Source Extensions and Encrypted Media Extensions](chapters/part-13-video-streaming/ch60e-mse-eme-browser-drm.md)
- [Ch 142: V4L2 and the Linux Media Subsystem](chapters/part-13-video-streaming/ch142-v4l2-media-subsystem.md)
- [Ch 189: VLC Media Player — Architecture, GPU Acceleration, and the Linux Graphics Stack](chapters/part-13-video-streaming/ch189-vlc-architecture-gpu-linux.md)

[↑ Contents](#contents)

---

### Part XIV — The Khronos Extended Ecosystem

- [Part Overview](chapters/part-14-khronos-ecosystem/part-intro.md)
- [Ch 61: SPIR-V Ecosystem in Depth](chapters/part-14-khronos-ecosystem/ch61-spirv-ecosystem.md)
- [Ch 62: SYCL 2020 and Portable Heterogeneous Compute](chapters/part-14-khronos-ecosystem/ch62-sycl.md)
- [Ch 63: KTX2, Basis Universal, and GPU Texture Compression](chapters/part-14-khronos-ecosystem/ch63-ktx2-texture-compression.md)
- [Ch 64: glTF 2.0 — The 3D Asset Pipeline Standard](chapters/part-14-khronos-ecosystem/ch64-gltf.md)
- [Ch 65: Vulkan Safety Critical and OpenVX](chapters/part-14-khronos-ecosystem/ch65-vulkan-sc-openvx.md)
- [Ch 110: SPIR-V Tooling — spirv-tools, SPIRV-Cross, and the Shader Ecosystem](chapters/part-14-khronos-ecosystem/ch110-spirv-tooling.md)
- [Ch 134: OpenCL on Linux](chapters/part-14-khronos-ecosystem/ch134-opencl-linux.md)

[↑ Contents](#contents)

---

### Part XV — The NVIDIA Proprietary Graphics Stack

- [Part Overview](chapters/part-15-nvidia-stack/part-intro.md)
- [Ch 66: CUDA Runtime, Streams, and NVRTC](chapters/part-15-nvidia-stack/ch66-cuda-runtime.md)
- [Ch 67: OptiX 9 — NVIDIA's Ray Tracing Framework](chapters/part-15-nvidia-stack/ch67-optix.md)
- [Ch 68: DLSS 4, Neural Rendering, and Frame Generation](chapters/part-15-nvidia-stack/ch68-dlss-neural-rendering.md)
- [Ch 69: NVIDIA Omniverse, OpenUSD, and the RTX Renderer](chapters/part-15-nvidia-stack/ch69-omniverse-usd.md)
- [Ch 70: RTX Kit — RTXDI, RTXGI, NRD, RTXNS, and RTXNTC](chapters/part-15-nvidia-stack/ch70-rtx-kit.md)
- [Ch 117: Slang — Differentiable and Modular Shading Language](chapters/part-15-nvidia-stack/ch117-slang-differentiable-shading.md)
- [Ch 240: NVIDIA Cosmos, OSMO, and Omniverse Farm — Orchestrating Physical AI at Scale](chapters/part-15-nvidia-stack/ch240-cosmos-osmo-omniverse-farm.md)

[↑ Contents](#contents)

---

### Part XVI — The Intel Open Graphics Stack

- [Part Overview](chapters/part-16-intel-stack/part-intro.md)
- [Ch 71: Intel Xe Kernel Driver, Arc GPU Architecture, and the Intel Open Stack](chapters/part-16-intel-stack/ch71-intel-xe-arc.md)

[↑ Contents](#contents)

---

### Part XVII — The AMD Developer Ecosystem

- [Part Overview](chapters/part-17-amd-ecosystem/part-intro.md)
- [Ch 72: AMD FidelityFX SDK and Radeon Developer Tools](chapters/part-17-amd-ecosystem/ch72-amd-fidelityfx-tools.md)
- [Ch 143: RADV Internals: The Mesa AMD Vulkan Driver](chapters/part-17-amd-ecosystem/ch143-radv-internals.md)
- [Ch 170: AMDVLK vs. RADV — AMD's Two Open Vulkan Drivers](chapters/part-17-amd-ecosystem/ch170-amdvlk-vs-radv.md)

[↑ Contents](#contents)

---

### Part XVIII — Rendering Abstraction Libraries

- [Part Overview](chapters/part-18-rendering-abstractions/part-intro.md)
- [Ch 81: SDL3 GPU API: A Portable High-Level GPU Abstraction](chapters/part-18-rendering-abstractions/ch81-sdl3-gpu.md)
- [Ch 82: Vulkan Ecosystem Toolkit: VMA, volk, vk-bootstrap, and Friends](chapters/part-18-rendering-abstractions/ch82-vma-vulkan-helpers.md)
- [Ch 83: Filament: Google's Physically Based Rendering Engine on Linux](chapters/part-18-rendering-abstractions/ch83-filament.md)
- [Ch 84: bgfx, Cross-Platform Rendering Abstractions, and the Frame Graph Pattern](chapters/part-18-rendering-abstractions/ch84-bgfx-render-graph.md)
- [Ch 113: CGAL and Computational Geometry on the Linux Graphics Stack](chapters/part-18-rendering-abstractions/ch113-cgal-computational-geometry.md)

[↑ Contents](#contents)

---

### Part XIX — Android Graphics

- [Part Overview](chapters/part-19-android-graphics/part-intro.md)
- [Ch 85: Android Compositor: SurfaceFlinger, HardwareBuffer, and the Buffer Pipeline](chapters/part-19-android-graphics/ch85-android-surfaceflinger.md)
- [Ch 86: Vulkan on Android: Drivers, ANGLE, and Mobile GPU Performance](chapters/part-19-android-graphics/ch86-android-vulkan.md)
- [Ch 87: Android AR: ARCore Architecture, Camera HAL Integration, and the Android XR Platform](chapters/part-19-android-graphics/ch87-android-ar-arcore.md)
- [Ch 161: Android Game Development Kit (AGDK) — Native Game Architecture, Input, Audio, and Frame Pacing](chapters/part-19-android-graphics/ch161-android-game-development-kit.md)
- [Ch 164: Android Runtime and Native Interop — ART, JNI, and the NDK](chapters/part-19-android-graphics/ch164-android-runtime-native-interop.md)
- [Ch 191: LiteRT and MediaPipe](chapters/part-19-android-graphics/ch191-litert-mediapipe.md)

[↑ Contents](#contents)

---

### Part XX — AI/ML Inference on Linux

- [Part Overview](chapters/part-20-ai-inference/part-intro.md)
- [Ch 48: ROCm and Machine Learning on Linux GPUs](chapters/part-20-ai-inference/ch48-rocm-ml-linux.md)
- [Ch 88: NPU and AI Accelerator Integration on Linux](chapters/part-20-ai-inference/ch88-npu-ai-accelerators.md)
- [Ch 94: ComfyUI and ComfyScript: Node-Graph AI Image Generation on Linux GPUs](chapters/part-20-ai-inference/ch94-comfyui-comfyscript.md)
- [Ch 108: ROCm and HIP — AMD's GPU Compute Stack](chapters/part-20-ai-inference/ch108-rocm-hip.md)
- [Ch 115: NeRFStudio, Neural Radiance Fields, and 3D Gaussian Splatting on Linux](chapters/part-20-ai-inference/ch115-nerfstudio-gaussian-splatting.md)
- [Ch 124: Local LLM Inference on Linux GPUs](chapters/part-20-ai-inference/ch124-llm-inference-linux.md)
- [Ch 199: Jupyter Internals — Architecture, Python Runtime, Multi-Kernel Support, and GPU Computing](chapters/part-20-ai-inference/ch199-jupyter-internals.md)
- [Ch 212: Python 3D ML Libraries — Open3D, PyTorch3D, and Kaolin](chapters/part-20-ai-inference/ch212-open3d-pytorch3d-kaolin.md)
- [Ch 239: Geometric Abstraction for World-Model Visual Reasoning](chapters/part-20-ai-inference/ch239-geometric-abstraction-world-models.md)

[↑ Contents](#contents)

---

### Part XXI — Platform, Legacy, and History

- [Part Overview](chapters/part-21-platform-legacy/part-intro.md)
- [Ch 95: X11/Xorg Architecture and the DRI Legacy Stack](chapters/part-21-platform-legacy/ch95-x11-xorg-dri-legacy.md)
- [Ch 103: The Linux Graphics Stack: History and Design Philosophy](chapters/part-21-platform-legacy/ch103-history-design-philosophy.md)
- [Ch 197: The Linux Graphics Stack in Context — Comparison with Windows and macOS](chapters/part-21-platform-legacy/ch197-linux-windows-macos-graphics-comparison.md)

[↑ Contents](#contents)

---

### Part XXVII — Display Hardware, Connectors, and Signal Standards

- [Part Overview](chapters/part-27-display-hardware-connectors/part-intro.md)
- [Ch 181: Modern Display Interface Standards](chapters/part-27-display-hardware-connectors/ch181-modern-display-interface-standards.md)
- [Ch 182: Digital Display Connectors and the Physical Layer](chapters/part-27-display-hardware-connectors/ch182-digital-display-connectors-physical-layer.md)
- [Ch 183: EDID and DisplayID — How Linux Discovers Display Capabilities](chapters/part-27-display-hardware-connectors/ch183-edid-displayid-capability-discovery.md)
- [Ch 184: Embedded DisplayPort (eDP) and Laptop Panel Management](chapters/part-27-display-hardware-connectors/ch184-embedded-displayport-laptop-panel.md)
- [Ch 185: Wireless Display Technologies on Linux](chapters/part-27-display-hardware-connectors/ch185-wireless-display-linux.md)
- [Ch 186: Video Pixel Formats and Display Signal Encoding](chapters/part-27-display-hardware-connectors/ch186-video-pixel-formats-signal-encoding.md)
- [Ch 187: HDMI CEC and the Linux CEC Subsystem](chapters/part-27-display-hardware-connectors/ch187-hdmi-cec-linux-subsystem.md)
- [Ch 188: Display Power States — DPMS, Panel Self-Refresh, and Display Idle Management](chapters/part-27-display-hardware-connectors/ch188-display-power-states-dpms-psr.md)

[↑ Contents](#contents)

---

### Part XXVIII — Linux Multimedia: Audio, Communication, and Music Production

- [Ch 213: VoIP on Linux — SIP, WebRTC, and Real-Time Communication](chapters/part-28-linux-multimedia/ch213-voip-on-linux.md)
- [Ch 214: Bluetooth Audio on Linux — BlueZ, A2DP, HFP, and LE Audio](chapters/part-28-linux-multimedia/ch214-bluetooth-audio.md)
- [Ch 215: MPV Architecture — libmpv, libplacebo, and GPU Video Output](chapters/part-28-linux-multimedia/ch215-mpv-architecture.md)
- [Ch 216: Speech Synthesis and ASR on Linux — espeak-ng, Piper, and Whisper](chapters/part-28-linux-multimedia/ch216-speech-synthesis-asr.md)
- [Ch 217: DLNA and Home Theater — GUPnP, Rygel, Kodi, and Network Media Streaming](chapters/part-28-linux-multimedia/ch217-dlna-home-theater.md)
- [Ch 218: Broadcast Streaming on Linux — SRT, NDI, RTSP, and Live Production](chapters/part-28-linux-multimedia/ch218-srt-broadcast-streaming.md)
- [Ch 219: MIDI, Synthesis, and Music Production on Linux — FluidSynth, LV2, SuperCollider, and DAW Integration](chapters/part-28-linux-multimedia/ch219-midi-synthesis-music-production.md)

[↑ Contents](#contents)

---

### Part XXIX — Graphics Algorithms

- [Part Overview](chapters/part-29-graphics-algorithms/part-intro.md)

**Shader Algorithm Catalog (Ch 204–207)**
- [Ch 204: Shader Algorithm Catalog — Rendering Pipeline, Lighting, and Shadows](chapters/part-29-graphics-algorithms/ch204-shader-algorithm-catalog.md)
- [Ch 205: Shader Algorithm Catalog — Global Illumination and Materials](chapters/part-29-graphics-algorithms/ch205-shader-gi-and-materials.md)
- [Ch 206: Shader Algorithm Catalog — Ray Tracing and Procedural Content](chapters/part-29-graphics-algorithms/ch206-shader-raytracing-and-procedural.md)
- [Ch 207: Shader Algorithm Catalog — Visual Effects, Post-Processing, and GPU Compute](chapters/part-29-graphics-algorithms/ch207-shader-vfx-postprocess-compute.md)

**GPU Geometry Algorithms (Ch 208–212)**
- [Ch 208: GPU Geometry Algorithms — Surface Representation and Mesh Processing](chapters/part-29-graphics-algorithms/ch208-gpu-geometry-algorithms.md)
- [Ch 209: GPU Geometry Algorithms — Spatial Structures, Differential Geometry, and Animation](chapters/part-29-graphics-algorithms/ch209-gpu-spatial-differential-animation.md)
- [Ch 210: GPU Geometry Algorithms — Physics Simulation and Volumetric Methods](chapters/part-29-graphics-algorithms/ch210-gpu-physics-and-volumetric.md)
- [Ch 211: GPU Geometry Algorithms — Terrain, Ray Tracing Geometry, and Point Clouds](chapters/part-29-graphics-algorithms/ch211-gpu-terrain-raytracing-pointcloud.md)
- [Ch 212: GPU Geometry Algorithms — Neural Geometry, Specialized Techniques, and GPU Primitives](chapters/part-29-graphics-algorithms/ch212-gpu-neural-specialized-primitives.md)

**Specialist Algorithm Chapters (Ch 220–225)**
- [Ch 220: GPU Image Processing Algorithms](chapters/part-29-graphics-algorithms/ch220-gpu-image-processing-algorithms.md)
- [Ch 221: GPU Algorithm Performance and Optimization](chapters/part-29-graphics-algorithms/ch221-gpu-algorithm-performance.md)
- [Ch 222: Computational Geometry Algorithms on GPU](chapters/part-29-graphics-algorithms/ch222-computational-geometry-algorithms.md)
- [Ch 223: GPU Video Processing Algorithms](chapters/part-29-graphics-algorithms/ch223-gpu-video-processing-algorithms.md)
- [Ch 224: 3D Shape Analysis Algorithms](chapters/part-29-graphics-algorithms/ch224-3d-shape-analysis-algorithms.md)
- [Ch 225: Computational Topology Algorithms on GPU](chapters/part-29-graphics-algorithms/ch225-computational-topology-algorithms.md)

**Mathematical and AI Algorithm Foundations (Ch 226–232)**
- [Ch 226: GPU Linear Algebra and Sparse Solvers](chapters/part-29-graphics-algorithms/ch226-gpu-linear-algebra-sparse-solvers.md)
- [Ch 227: GPU Random Number Generation and Monte Carlo Methods](chapters/part-29-graphics-algorithms/ch227-gpu-rng-monte-carlo.md)
- [Ch 228: GPU Graph Algorithms](chapters/part-29-graphics-algorithms/ch228-gpu-graph-algorithms.md)
- [Ch 229: GPU Machine Learning Inference Algorithms](chapters/part-29-graphics-algorithms/ch229-gpu-ml-inference-algorithms.md)
- [Ch 230: GPU Signal Processing and Audio DSP](chapters/part-29-graphics-algorithms/ch230-gpu-signal-processing-audio-dsp.md)
- [Ch 231: GPU Compression Algorithms](chapters/part-29-graphics-algorithms/ch231-gpu-compression-algorithms.md)
- [Ch 232: GPU Generative AI and LLM Inference on Linux](chapters/part-29-graphics-algorithms/ch232-gpu-generative-ai-llm-inference.md)

**3D Rendering Algorithm Chapters (Ch 233–235)**
- [Ch 233: GPU Denoising Algorithms](chapters/part-29-graphics-algorithms/ch233-gpu-denoising-algorithms.md)
- [Ch 234: GPU Spectral Rendering and Colorimetric Algorithms](chapters/part-29-graphics-algorithms/ch234-gpu-spectral-rendering-colorimetric.md)
- [Ch 235: GPU Vector Graphics and 2D Path Rendering](chapters/part-29-graphics-algorithms/ch235-gpu-vector-graphics-2d-rendering.md)

**Scene Analysis Algorithm Chapters (Ch 236–238)**
- [Ch 236: GPU 3D Scene Understanding and Semantic Segmentation](chapters/part-29-graphics-algorithms/ch236-gpu-scene-understanding-segmentation.md)
- [Ch 237: GPU Depth Estimation and Dense Reconstruction](chapters/part-29-graphics-algorithms/ch237-gpu-depth-estimation-reconstruction.md)
- [Ch 238: GPU Object Detection and 6DoF Pose Estimation](chapters/part-29-graphics-algorithms/ch238-gpu-object-detection-pose-estimation.md)

[↑ Contents](#contents)

---

### Appendices

- [Appendix A: Glossary](chapters/appendices/appendix-a-glossary.md)
- [Appendix B: Environment Variables Reference](chapters/appendices/appendix-b-environment-variables.md)
- [Appendix C: Kernel/Mesa/Driver Version Matrix](chapters/appendices/appendix-c-version-matrix.md)
- [Appendix D: GPU Capability Comparison](chapters/appendices/appendix-d-gpu-capability-comparison.md)
- [Appendix E: Contributing Checklists](chapters/appendices/appendix-e-contributing-checklists.md)
- [Appendix F: Virtio-GPU and Virtualisation](chapters/appendices/appendix-f-virtio-gpu-virtualisation.md)
- [Appendix G: Synchronisation Primitives Reference](chapters/appendices/appendix-g-sync-reference.md)
- [Appendix H: DRM Format Modifiers](chapters/appendices/appendix-h-drm-format-modifiers.md)
- [Appendix I: Wayland Protocols Matrix](chapters/appendices/appendix-i-wayland-protocols-matrix.md)
- [Appendix J: Debugging Quick Reference](chapters/appendices/appendix-j-debugging-quick-reference.md)
- [Appendix K: Remote Display Technologies](chapters/appendices/appendix-k-remote-display.md)
- [Appendix L: Shader Toolchain Matrix](chapters/appendices/appendix-l-shader-toolchain-matrix.md)
- [Appendix M: Kernel Configuration Reference](chapters/appendices/appendix-m-kernel-config-reference.md)
- [Appendix N: Vulkan on Linux — Platform Extensions Reference](chapters/appendices/appendix-n-vulkan-linux-platform-extensions.md)
- [Appendix O: SPIR-V Binary Format Reference](chapters/appendices/appendix-o-spirv-binary-format-reference.md)
- [Appendix P: WGSL Language Reference](chapters/appendices/appendix-p-wgsl-language-reference.md)
- [Appendix Q: Shader Language Comparison Reference](chapters/appendices/appendix-q-shader-language-comparison.md)
- [Appendix R: EGL Platform Reference](chapters/appendices/appendix-r-egl-platform-reference.md)
- [Appendix S: DRM/KMS ioctl Quick Reference](chapters/appendices/appendix-s-drm-kms-ioctl-reference.md)
- [Appendix T: Terminal Graphics Protocol Reference](chapters/appendices/appendix-t-terminal-graphics-protocol-reference.md)
- [Appendix U: WebGPU API Quick Reference](chapters/appendices/appendix-u-webgpu-api-reference.md)

[↑ Contents](#contents)
