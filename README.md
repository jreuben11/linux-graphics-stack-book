# The Linux Graphics Stack: From Kernel to Compositor, Browser, and Terminal

An expert-level technical reference (~1,400 pages, 200 chapters) tracing every layer of the Linux graphics stack — from the DRM kernel subsystem and GPU memory management through Mesa's shader compilers, Wayland compositors, application APIs, the browser rendering pipeline, game compatibility layers, terminal pixel protocols, GPU-accelerated video streaming, AI inference, and both open and proprietary GPU ecosystems. Each chapter is written for practitioners: code snippets reference real upstream source at specific commits or releases, API signatures are verified against current kernel and Mesa trees, and architectural claims cite primary sources.

By the final chapter the reader holds a continuous mental model from a `DRM_IOCTL_MODE_ATOMIC` kernel call all the way to a photon leaving a display panel, and from a WebGPU `drawIndexed` call in JavaScript all the way to the same panel.

## Audiences

- **Systems and driver developers** — kernel internals, DRM/Mesa architecture, driver implementation
- **Graphics application developers** — the stack beneath Vulkan, EGL, VA-API, and OpenXR
- **Browser and web platform engineers** — how Chromium/Chrome maps WebGPU, WebGL, and compositing onto Linux hardware
- **Terminal and TUI developers** — GPU-rendered terminal emulators, Sixel, Kitty Graphics Protocol, Ghostty

## Parts and Chapters

### Part I — The Kernel Layer

*The Linux graphics stack begins in the kernel, where the DRM (Direct Rendering Manager) subsystem unifies GPU execution, display programming, memory management, scheduling, and power control. This part provides the foundational concepts and architecture that every subsequent layer depends on.*

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

---

### Part II — GPU Drivers

*Between the abstract DRM contracts and the userspace Mesa drivers sits the kernel GPU driver layer. This part surveys GPU driver families from high-end x86 discrete graphics to constrained embedded platforms, covering amdgpu, i915/Xe, Panfrost, Lima, etnaviv, and others.*

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

---

### Part III — The Open NVIDIA Stack

*This part tells the story of NVIDIA hardware on Linux through 17 years of reverse engineering, the GSP-RM firmware turning point, the Nova Rust kernel driver, the NVK Vulkan driver, and the NAK Rust shader compiler that finally unlocked competitive open-source NVIDIA graphics.*

- [Part Overview](chapters/part-03-nouveau-story/part-intro.md)
- [Ch 7: Reverse Engineering NVIDIA: History and Methodology](chapters/part-03-nouveau-story/ch07-reverse-engineering-nvidia.md)
- [Ch 8: The Nouveau Kernel Driver: nvkm Architecture](chapters/part-03-nouveau-story/ch08-nouveau-kernel-driver.md)
- [Ch 9: GSP-RM, Firmware, and the nvidia-open Connection](chapters/part-03-nouveau-story/ch09-gsp-rm-firmware.md)
- [Ch 10a: Nova — The Rust NVIDIA Kernel Driver](chapters/part-03-nouveau-story/ch10a-nova-rust-nvidia-driver.md)
- [Ch 10b: NVK: Building a Vulkan Driver from Scratch](chapters/part-03-nouveau-story/ch10b-nvk-vulkan-driver.md)
- [Ch 11: Display, Reclocking, and Power Management](chapters/part-03-nouveau-story/ch11-display-reclocking-power.md)
- [Ch 118: NAK — The Nouveau/NVK Rust Shader Compiler](chapters/part-03-nouveau-story/ch118-nak-rust-shader-compiler.md)

---

### Part IV — Mesa Architecture

*Mesa is the userspace half of the Linux graphics stack, containing API frontends for OpenGL, Vulkan, and OpenCL, the shared NIR shader compiler, and vendor-specific GPU backends. This part examines the architecture all Mesa components inherit.*

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

---

### Part V — Mesa GPU Drivers

*Where Mesa's architecture becomes concrete: RADV for AMD, ANV for Intel, NVK for NVIDIA, and Turnip for Qualcomm Adreno. This part traces how abstract Mesa interfaces translate to hardware command streams.*

- [Part Overview](chapters/part-05-mesa-gpu-drivers/part-intro.md)
- [Ch 18: Vulkan Drivers](chapters/part-05-mesa-gpu-drivers/ch18-vulkan-drivers.md)
- [Ch 19: OpenGL and Compatibility Drivers](chapters/part-05-mesa-gpu-drivers/ch19-opengl-compatibility-drivers.md)
- [Ch 177: NVK — NVIDIA Vulkan in Mesa](chapters/part-05-mesa-gpu-drivers/ch177-nvk-nvidia-vulkan.md)

---

### Part VI-A — Wayland Protocol and Compositor Architecture

*The Wayland wire protocol, compositor implementation with wlroots, production compositors (Mutter, KWin, Sway, Hyprland), XWayland, and the wave of staging protocols reaching stability in 2024–2026: explicit sync, colour management, screen capture, frame scheduling, and portal evolution.*

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

---

### Part VI-B — Display Services, Input, and Color

*The layer between compositor and application: calibration, HDR and wide colour gamut, colour science and ICC profiles, the Linux input stack, touch and stylus input, VRR, font rendering, screen capture, remote desktop, HDMI audio, DisplayPort MST, and desktop IPC.*

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

---

### Part VII-A — GPU APIs and Extended Reality

*The Vulkan API in depth for application developers — EGL, swapchains, compute, video decode/encode, ray tracing, mesh shaders, cooperative matrices, descriptor binding, GPU-driven rendering, shader objects, and the full Vulkan extension ecosystem. Also covers VA-API hardware video and OpenXR/Monado for VR and AR.*

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

---

### Part VII-B — Multimedia Frameworks and Desktop Integration

*Where GPU resources meet application-layer multimedia: PipeWire's graph model and session management, ALSA's kernel architecture and libasound API, Qt/GTK GPU rendering, font and text layout, Vulkan Video extensions, libcamera, Flatpak GPU access, and OpenCV GPU acceleration.*

- [Ch 38: PipeWire and the Video Session Layer](chapters/part-07b-multimedia-frameworks/ch38-pipewire.md)
- [Ch 38b: ALSA — The Linux Audio Subsystem](chapters/part-07b-multimedia-frameworks/ch38b-alsa-linux-audio-subsystem.md)
- [Ch 39: Qt and GTK GPU Rendering](chapters/part-07b-multimedia-frameworks/ch39-qt-gtk-rendering.md)
- [Ch 47: Font and Text Rendering Pipeline](chapters/part-07b-multimedia-frameworks/ch47-font-text-rendering.md)
- [Ch 50: Vulkan Video Extensions](chapters/part-07b-multimedia-frameworks/ch50-vulkan-video.md)
- [Ch 96: libcamera and the Linux Camera Stack](chapters/part-07b-multimedia-frameworks/ch96-libcamera.md)
- [Ch 111: Flatpak Graphics — GPU Access in Sandboxed Applications](chapters/part-07b-multimedia-frameworks/ch111-flatpak-graphics.md)
- [Ch 114: OpenCV and GPU-Accelerated Computer Vision on Linux](chapters/part-07b-multimedia-frameworks/ch114-opencv-gpu-vision.md)
- [Ch 206: SDL3 — Cross-Platform Multimedia Integration on Linux](chapters/part-07b-multimedia-frameworks/ch206-sdl-multimedia.md)
- [Ch 209: OpenSLAM — Classical and Graph-Based SLAM on the Linux Stack](chapters/part-07b-multimedia-frameworks/ch209-openslam.md)
- [Ch 210: SLAM Theory and State of the Art](chapters/part-07b-multimedia-frameworks/ch210-slam-theory-sota.md)
- [Ch 211: ROS 2 Multimodal Sensor and Perception Pipeline](chapters/part-07b-multimedia-frameworks/ch211-ros2-sensor-perception-pipeline.md)

---

### Part VIII — The Gaming Layer

*How Windows games run on Linux through Wine, DXVK/VKD3D-Proton translation, the Steam Deck integration, and the gamescope micro-compositor. This part shows how every graphics primitive composes into a coherent gaming product.*

- [Part Overview](chapters/part-08-gaming-layer/part-intro.md)
- [Ch 28: Windows Compatibility](chapters/part-08-gaming-layer/ch28-windows-compatibility.md)
- [Ch 29: Upscaling, Effects & Overlays](chapters/part-08-gaming-layer/ch29-upscaling-effects-overlays.md)
- [Ch 56: Ray Tracing on Linux](chapters/part-08-gaming-layer/ch56-ray-tracing-linux.md)
- [Ch 78: Gamescope and the Steam Deck: A Complete Gaming Graphics Stack](chapters/part-08-gaming-layer/ch78-gamescope-steam-deck.md)
- [Ch 104: DXVK and VKD3D-Proton — D3D-to-Vulkan Translation](chapters/part-08-gaming-layer/ch104-dxvk-vkd3d-proton.md)
- [Ch 167: NTSYNC — NT Synchronization Primitives in the Linux Kernel](chapters/part-08-gaming-layer/ch167-ntsync-nt-synchronization-linux.md)
- [Ch 171: Linux Gaming Anti-Cheat — EasyAntiCheat, BattlEye, and the Ring-0 Problem](chapters/part-08-gaming-layer/ch171-linux-gaming-anti-cheat.md)

---

### Part IX — Tooling & Contributing

*The operational layer: how engineers observe, measure, debug, and improve the graphics stack. Covers profilers, conformance suites, CI infrastructure, GPU virtualisation, security, and contribution workflows.*

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

---

### Part X — The Browser Rendering Stack

*How web browsers—Chrome via ANGLE/Dawn/Skia and Firefox via WebRender/wgpu—sandbox GPU access and translate legacy OpenGL ES and modern WebGPU to Mesa Vulkan drivers, ultimately delivering frames to the Wayland compositor.*

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

---

### Part XI — Engines & Creative Tools

*By the time a frame reaches a game engine or creative suite, it has already traversed the kernel DRM subsystem, been scheduled by a Mesa Vulkan driver, and been handed to a Wayland compositor for presentation. This part examines how engines and creative tools structure their own rendering abstractions, issue Vulkan commands, compile shaders, manage GPU memory, and integrate with the windowing stack.*

- [Part Overview](chapters/part-11-engine-creative-tools/part-intro.md)
- [Ch 40: Bevy and wgpu](chapters/part-11-engine-creative-tools/ch40-bevy-wgpu.md)
- [Ch 41: Godot 4 RenderingDevice](chapters/part-11-engine-creative-tools/ch41-godot4-rendering-device.md)
- [Ch 42: Blender GPU — Cycles and EEVEE](chapters/part-11-engine-creative-tools/ch42-blender-gpu.md)
- [Ch 97: Unreal Engine 5 on Linux](chapters/part-11-engine-creative-tools/ch97-unreal-engine-5.md)
- [Ch 176: OpenCASCADE Technology — The BRep Kernel and 3D Visualization Stack](chapters/part-11-engine-creative-tools/ch176-opencascade-cad-kernel.md)
- [Ch 190: VTK — Scientific Visualization on the Linux Graphics Stack](chapters/part-11-engine-creative-tools/ch190-vtk-scientific-visualization.md)
- [Ch 205: AI-Driven 3D Creation — Blender MCP, Claude Code, and Generative Tools](chapters/part-11-engine-creative-tools/ch205-blender-ai-mcp.md)

---

### Part XII — Terminal Graphics

*Terminal emulators occupy an unusual position in the Linux graphics stack: they are simultaneously text-processing engines descended from 1970s hardware terminals and first-class Wayland clients that drive modern GPU render pipelines. This part examines how the character-cell abstraction has been extended to carry pixel graphics, and how GPU-accelerated terminals integrate with the same GBM, DMA-BUF, KMS, and compositor machinery as every other graphical client.*

- [Part Overview](chapters/part-12-terminal-graphics/part-intro.md)
- [Ch 43: Terminal Pixel Protocols — Sixel, Kitty, and iTerm2](chapters/part-12-terminal-graphics/ch43-terminal-pixel-protocols.md)
- [Ch 44: Terminal GPU Rendering Architectures](chapters/part-12-terminal-graphics/ch44-terminal-gpu-rendering.md)
- [Ch 45: Terminal Integration with the Compositor Stack](chapters/part-12-terminal-graphics/ch45-terminal-compositor-integration.md)
- [Ch 174: WezTerm and Alacritty — GPU Terminal Rendering Architectures](chapters/part-12-terminal-graphics/ch174-wezterm-alacritty-architecture.md)
- [Ch 178: The PTY/TTY Kernel Layer and Line Disciplines](chapters/part-12-terminal-graphics/ch178-pty-tty-kernel-layer.md)
- [Ch 192: Ratatui — The Rust TUI Application Framework](chapters/part-12-terminal-graphics/ch192-ratatui-tui-framework.md)

---

### Part XIII — Video Streaming on Linux

*Video streaming occupies the layer directly above hardware-accelerated codec paths—above VA-API decode surfaces and Vulkan Video queues—and connects those low-level GPU resources to application-facing pipeline frameworks, adaptive delivery protocols, and AI-driven analytics systems.*

- [Part Overview](chapters/part-13-video-streaming/part-intro.md)
- [Ch 57: FFmpeg Architecture and Programming](chapters/part-13-video-streaming/ch57-ffmpeg.md)
- [Ch 58: GStreamer: Pipeline-Based Multimedia](chapters/part-13-video-streaming/ch58-gstreamer.md)
- [Ch 59: NVIDIA DeepStream SDK](chapters/part-13-video-streaming/ch59-deepstream.md)
- [Ch 60: Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs](chapters/part-13-video-streaming/ch60-video-codecs.md)
- [Ch 60b: Video Streaming Protocols and Adaptive Bitrate Delivery](chapters/part-13-video-streaming/ch60b-video-streaming-protocols.md)
- [Ch 60c: WebRTC Server Infrastructure on Linux](chapters/part-13-video-streaming/ch60c-webrtc-server-infrastructure.md)
- [Ch 60d: BitTorrent Adaptive Streaming on Linux — libtorrent, WebTorrent, and the GPU Decode Pipeline](chapters/part-13-video-streaming/ch60d-bittorrent-streaming.md)
- [Ch 142: V4L2 and the Linux Media Subsystem](chapters/part-13-video-streaming/ch142-v4l2-media-subsystem.md)
- [Ch 189: VLC Media Player — Architecture, GPU Acceleration, and the Linux Graphics Stack](chapters/part-13-video-streaming/ch189-vlc-architecture-gpu-linux.md)

---

### Part XIV — The Khronos Extended Ecosystem

*The Khronos Group defines most of the open APIs sitting between Linux GPU drivers and application code: Vulkan, OpenGL, OpenCL, OpenXR, and supporting standards for shader intermediate representation, asset interchange, compute portability, and safety-critical deployment. This part covers the cross-cutting Khronos-ratified standards that tie those pieces together.*

- [Part Overview](chapters/part-14-khronos-ecosystem/part-intro.md)
- [Ch 61: SPIR-V Ecosystem in Depth](chapters/part-14-khronos-ecosystem/ch61-spirv-ecosystem.md)
- [Ch 62: SYCL 2020 and Portable Heterogeneous Compute](chapters/part-14-khronos-ecosystem/ch62-sycl.md)
- [Ch 63: KTX2, Basis Universal, and GPU Texture Compression](chapters/part-14-khronos-ecosystem/ch63-ktx2-texture-compression.md)
- [Ch 64: glTF 2.0 — The 3D Asset Pipeline Standard](chapters/part-14-khronos-ecosystem/ch64-gltf.md)
- [Ch 65: Vulkan Safety Critical and OpenVX](chapters/part-14-khronos-ecosystem/ch65-vulkan-sc-openvx.md)
- [Ch 110: SPIR-V Tooling — spirv-tools, SPIRV-Cross, and the Shader Ecosystem](chapters/part-14-khronos-ecosystem/ch110-spirv-tooling.md)
- [Ch 134: OpenCL on Linux](chapters/part-14-khronos-ecosystem/ch134-opencl-linux.md)

---

### Part XV — The NVIDIA Proprietary Graphics Stack

*The NVIDIA proprietary stack is not a replacement for the Linux graphics stack; it is built on top of `nvidia.ko`, `nvidia-uvm.ko`, and `nvidia-drm.ko`, consuming the same DRM scheduler slots and GEM object lifetime rules, but exposing programming models and AI-driven rendering features that have no open-source counterpart.*

- [Part Overview](chapters/part-15-nvidia-stack/part-intro.md)
- [Ch 66: CUDA Runtime, Streams, and NVRTC](chapters/part-15-nvidia-stack/ch66-cuda-runtime.md)
- [Ch 67: OptiX 9 — NVIDIA's Ray Tracing Framework](chapters/part-15-nvidia-stack/ch67-optix.md)
- [Ch 68: DLSS 4, Neural Rendering, and Frame Generation](chapters/part-15-nvidia-stack/ch68-dlss-neural-rendering.md)
- [Ch 69: NVIDIA Omniverse, OpenUSD, and the RTX Renderer](chapters/part-15-nvidia-stack/ch69-omniverse-usd.md)
- [Ch 70: RTX Kit — RTXDI, RTXGI, NRD, RTXNS, and RTXNTC](chapters/part-15-nvidia-stack/ch70-rtx-kit.md)
- [Ch 117: Slang — Differentiable and Modular Shading Language](chapters/part-15-nvidia-stack/ch117-slang-differentiable-shading.md)

---

### Part XVI — The Intel Open Graphics Stack

*Intel's graphics hardware is simultaneously the most widely deployed GPU on Linux desktops and, since 2022, a serious discrete GPU competitor through the Arc and Battlemage product lines. This part examines the complete open-source software stack Intel ships for these GPUs — from the kernel driver through the Vulkan driver, compute runtime, media decode pipeline, and AI upscaling SDK.*

- [Part Overview](chapters/part-16-intel-stack/part-intro.md)
- [Ch 71: Intel Xe Kernel Driver, Arc GPU Architecture, and the Intel Open Stack](chapters/part-16-intel-stack/ch71-intel-xe-arc.md)

---

### Part XVII — The AMD Developer Ecosystem

*AMD's open developer toolchain sits above the kernel DRM/AMDGPU driver and the Mesa/RADV Vulkan driver, providing the image-quality libraries, hardware media encoding APIs, and profiling infrastructure that turn raw GPU capability into polished, measurable performance.*

- [Part Overview](chapters/part-17-amd-ecosystem/part-intro.md)
- [Ch 72: AMD FidelityFX SDK and Radeon Developer Tools](chapters/part-17-amd-ecosystem/ch72-amd-fidelityfx-tools.md)
- [Ch 143: RADV Internals: The Mesa AMD Vulkan Driver](chapters/part-17-amd-ecosystem/ch143-radv-internals.md)
- [Ch 170: AMDVLK vs. RADV — AMD's Two Open Vulkan Drivers](chapters/part-17-amd-ecosystem/ch170-amdvlk-vs-radv.md)

---

### Part XVIII — Rendering Abstraction Libraries

*The libraries covered here — SDL3 GPU, VMA, Filament, bgfx, and CGAL — are not part of the OS or the driver; they are userspace libraries that applications link against to gain portable, ergonomic access to GPU resources without writing raw Vulkan bootstrap code or implementing their own memory allocators, frame graphs, or physically based shading pipelines.*

- [Part Overview](chapters/part-18-rendering-abstractions/part-intro.md)
- [Ch 81: SDL3 GPU API: A Portable High-Level GPU Abstraction](chapters/part-18-rendering-abstractions/ch81-sdl3-gpu.md)
- [Ch 82: Vulkan Ecosystem Toolkit: VMA, volk, vk-bootstrap, and Friends](chapters/part-18-rendering-abstractions/ch82-vma-vulkan-helpers.md)
- [Ch 83: Filament: Google's Physically Based Rendering Engine on Linux](chapters/part-18-rendering-abstractions/ch83-filament.md)
- [Ch 84: bgfx, Cross-Platform Rendering Abstractions, and the Frame Graph Pattern](chapters/part-18-rendering-abstractions/ch84-bgfx-render-graph.md)
- [Ch 113: CGAL and Computational Geometry on the Linux Graphics Stack](chapters/part-18-rendering-abstractions/ch113-cgal-computational-geometry.md)

---

### Part XIX — Android Graphics

*Android is a Linux-based operating system built directly on the same kernel primitives — DRM/KMS, DMA-BUF, sync_file, and dma_fence — that underpin the Wayland desktop compositor ecosystem. What distinguishes Android is the proprietary middleware erected above those primitives: Gralloc, HWComposer, SurfaceFlinger, and ARCore.*

- [Part Overview](chapters/part-19-android-graphics/part-intro.md)
- [Ch 85: Android Compositor: SurfaceFlinger, HardwareBuffer, and the Buffer Pipeline](chapters/part-19-android-graphics/ch85-android-surfaceflinger.md)
- [Ch 86: Vulkan on Android: Drivers, ANGLE, and Mobile GPU Performance](chapters/part-19-android-graphics/ch86-android-vulkan.md)
- [Ch 87: Android AR: ARCore Architecture, Camera HAL Integration, and the Android XR Platform](chapters/part-19-android-graphics/ch87-android-ar-arcore.md)
- [Ch 161: Android Game Development Kit (AGDK) — Native Game Architecture, Input, Audio, and Frame Pacing](chapters/part-19-android-graphics/ch161-android-game-development-kit.md)
- [Ch 164: Android Runtime and Native Interop — ART, JNI, and the NDK](chapters/part-19-android-graphics/ch164-android-runtime-native-interop.md)
- [Ch 166: Android XR](chapters/part-19-android-graphics/ch166-android-xr.md)
- [Ch 191: LiteRT and MediaPipe](chapters/part-19-android-graphics/ch191-litert-mediapipe.md)

---

### Part XX — AI/ML Inference on Linux

*AI/ML workloads are now first-class consumers of GPU compute on Linux: the same VRAM, the same PCIe bus, and the same DRM scheduling infrastructure that draws your desktop also run transformer attention heads, diffusion model denoising steps, and low-power NPU inference for always-on voice and vision tasks.*

- [Part Overview](chapters/part-20-ai-inference/part-intro.md)
- [Ch 48: ROCm and Machine Learning on Linux GPUs](chapters/part-20-ai-inference/ch48-rocm-ml-linux.md)
- [Ch 88: NPU and AI Accelerator Integration on Linux](chapters/part-20-ai-inference/ch88-npu-ai-accelerators.md)
- [Ch 94: ComfyUI and ComfyScript: Node-Graph AI Image Generation on Linux GPUs](chapters/part-20-ai-inference/ch94-comfyui-comfyscript.md)
- [Ch 108: ROCm and HIP — AMD's GPU Compute Stack](chapters/part-20-ai-inference/ch108-rocm-hip.md)
- [Ch 115: NeRFStudio, Neural Radiance Fields, and 3D Gaussian Splatting on Linux](chapters/part-20-ai-inference/ch115-nerfstudio-gaussian-splatting.md)
- [Ch 124: Local LLM Inference on Linux GPUs](chapters/part-20-ai-inference/ch124-llm-inference-linux.md)
- [Ch 199: Jupyter Internals — Architecture, Python Runtime, Multi-Kernel Support, and GPU Computing](chapters/part-20-ai-inference/ch199-jupyter-internals.md)
- [Ch 212: Python 3D ML Libraries — Open3D, PyTorch3D, and Kaolin](chapters/part-20-ai-inference/ch212-open3d-pytorch3d-kaolin.md)

---

### Part XXI — Platform, Legacy, and History

*The Linux graphics stack is not a single coherent design; it is four decades of accumulated engineering decisions. This part examines the protocol and infrastructure the modern stack replaced, and the narrative history that explains why every layer is the shape it is.*

- [Part Overview](chapters/part-21-platform-legacy/part-intro.md)
- [Ch 95: X11/Xorg Architecture and the DRI Legacy Stack](chapters/part-21-platform-legacy/ch95-x11-xorg-dri-legacy.md)
- [Ch 103: The Linux Graphics Stack: History and Design Philosophy](chapters/part-21-platform-legacy/ch103-history-design-philosophy.md)
- [Ch 197: The Linux Graphics Stack in Context — Comparison with Windows and macOS](chapters/part-21-platform-legacy/ch197-linux-windows-macos-graphics-comparison.md)

---

### Part XXVII — Display Hardware, Connectors, and Signal Standards

*Beneath every compositor frame is a physical signal travelling through a connector, a protocol negotiating display capabilities, and a hardware encoding standard that determines what the panel can show. This part covers the physical path from KMS connector to display panel: interface standards, connector pinouts, signal encoding formats, discovery protocols, and power management contracts.*

- [Part Overview](chapters/part-27-display-hardware-connectors/part-intro.md)
- [Ch 181: Modern Display Interface Standards](chapters/part-27-display-hardware-connectors/ch181-modern-display-interface-standards.md)
- [Ch 182: Digital Display Connectors and the Physical Layer](chapters/part-27-display-hardware-connectors/ch182-digital-display-connectors-physical-layer.md)
- [Ch 183: EDID and DisplayID — How Linux Discovers Display Capabilities](chapters/part-27-display-hardware-connectors/ch183-edid-displayid-capability-discovery.md)
- [Ch 184: Embedded DisplayPort (eDP) and Laptop Panel Management](chapters/part-27-display-hardware-connectors/ch184-embedded-displayport-laptop-panel.md)
- [Ch 185: Wireless Display Technologies on Linux](chapters/part-27-display-hardware-connectors/ch185-wireless-display-linux.md)
- [Ch 186: Video Pixel Formats and Display Signal Encoding](chapters/part-27-display-hardware-connectors/ch186-video-pixel-formats-signal-encoding.md)
- [Ch 187: HDMI CEC and the Linux CEC Subsystem](chapters/part-27-display-hardware-connectors/ch187-hdmi-cec-linux-subsystem.md)
- [Ch 188: Display Power States — DPMS, Panel Self-Refresh, and Display Idle Management](chapters/part-27-display-hardware-connectors/ch188-display-power-states-dpms-psr.md)

---

### Part XXVIII — Linux Multimedia: Audio, Communication, and Music Production

*Beyond the graphics stack lies a rich multimedia layer that shares the same Linux audio and video infrastructure: PipeWire, ALSA, V4L2, GStreamer, and the Wayland compositor. This part covers VoIP, Bluetooth audio, video playback architecture, speech synthesis, DLNA home media, broadcast streaming, and MIDI/synthesis — topics that application developers encounter when building communication and creative software on Linux.*

- [Ch 213: VoIP on Linux — SIP, WebRTC, and Real-Time Communication](chapters/part-28-linux-multimedia/ch213-voip-on-linux.md)
- [Ch 214: Bluetooth Audio on Linux — BlueZ, A2DP, HFP, and LE Audio](chapters/part-28-linux-multimedia/ch214-bluetooth-audio.md)
- [Ch 215: MPV Architecture — libmpv, libplacebo, and GPU Video Output](chapters/part-28-linux-multimedia/ch215-mpv-architecture.md)
- [Ch 216: Speech Synthesis and ASR on Linux — espeak-ng, Piper, and Whisper](chapters/part-28-linux-multimedia/ch216-speech-synthesis-asr.md)
- [Ch 217: DLNA and Home Theater — GUPnP, Rygel, Kodi, and Network Media Streaming](chapters/part-28-linux-multimedia/ch217-dlna-home-theater.md)
- [Ch 218: Broadcast Streaming on Linux — SRT, NDI, RTSP, and Live Production](chapters/part-28-linux-multimedia/ch218-srt-broadcast-streaming.md)
- [Ch 219: MIDI, Synthesis, and Music Production on Linux — FluidSynth, LV2, SuperCollider, and DAW Integration](chapters/part-28-linux-multimedia/ch219-midi-synthesis-music-production.md)

---

### Part XXIX — Graphics Algorithms

*Fifteen chapters covering the algorithms that run on top of the Linux graphics stack: shader techniques, GPU geometry, image processing, video processing, performance optimization, computational geometry, shape analysis, and computational topology.*

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

**Total: 220 chapters + 21 appendices**

## Repository Structure

```
plan.md                          # Master outline with full per-chapter bullet points
intro.md                         # Holistic introduction and reading paths
chapters/
  intro.md                       # Book introduction (duplicate entry point)
  part-01-kernel-layer/
    part-intro.md                # Part overview (present in every part directory)
    ch01-drm-architecture.md
    ch02-kms-display-pipeline.md
    ch03-advanced-display-features.md
    ch04-gpu-memory-management.md
    ch51-gpu-power-management.md
    ch102-drm-gpu-scheduler.md
    ch120-gpu-memory-management.md
    ch121-drm-lease-vr-direct-display.md
    ch129-gpu-firmware.md
    ch139-drm-hardware-planes.md
    ch144-boot-graphics-pipeline.md
    ch149-gpu-hang-detection-recovery.md
    ch162-framebuffer-compression.md
    ch163-vkms-virtual-display.md
  part-02-gpu-drivers/
    ch05-x86-gpu-drivers.md
    ch06-arm-embedded-gpu-drivers.md
    ch49-multi-gpu-prime.md
    ch73-asahi-apple-silicon.md
    ch90-panfrost-panthor-lima.md
    ch92-raspberry-pi-videocore.md
    ch99-automotive-embedded-graphics.md
    ch100-etnaviv-vivante.md
    ch116-riscv-gpu-drivers.md
    ch126-hybrid-graphics-laptop.md
    ch155-usb-displaylink-evdi.md
    ch160-freedreno-turnip-adreno.md
    ch169-snapdragon-x-elite-linux.md
    ch172-egpu-thunderbolt-usb4.md
    ch179-linux-accel-subsystem.md
    ch196-gpu-firmware-as-os.md
  part-03-nouveau-story/
    ch07-reverse-engineering-nvidia.md
    ch08-nouveau-kernel-driver.md
    ch09-gsp-rm-firmware.md
    ch10a-nova-rust-nvidia-driver.md
    ch10b-nvk-vulkan-driver.md
    ch11-display-reclocking-power.md
    ch118-nak-rust-shader-compiler.md
  part-04-mesa-architecture/
    ch12-mesa-loader-dispatch.md
    ch13-gallium3d.md
    ch14-nir-shader-ir.md
    ch15-aco-compiler.md
    ch16-mesa-vulkan-common.md
    ch17-software-renderers.md
    ch77-shader-toolchain.md
    ch91-mlir-gpu-compilation.md
    ch119-zink-opengl-on-vulkan.md
    ch156-mesa-nine-direct3d9.md
    ch159-vulkan-mesa-drm-nvidia-stack.md
  part-05-mesa-gpu-drivers/
    ch18-vulkan-drivers.md
    ch19-opengl-compatibility-drivers.md
    ch177-nvk-nvidia-vulkan.md
  part-06a-wayland-compositor/
    ch20-wayland-protocol-fundamentals.md
    ch21-building-compositors-wlroots.md
    ch22-production-compositors.md
    ch23-legacy-sandboxed-app-support.md
    ch46-wayland-protocol-ecosystem.md
    ch130-wayland-protocol-dev.md
    ch132-wayland-security.md
    ch138-wayland-fractional-scaling.md
    ch145-xwayland-architecture.md
    ch151-wayland-text-input-ime.md
    ch175-atspie2-compositor-accessibility.md
    ch207-xdg-desktop-portal.md
  part-06b-display-services/
    ch53-display-calibration-colord.md
    ch54-linux-input-stack.md
    ch74-hdr-wide-color-gamut.md
    ch75-explicit-gpu-sync.md
    ch101-color-science-icc.md
    ch105-font-rendering.md
    ch112-vrr-freesync-frame-pacing.md
    ch123-screen-capture-remote-desktop.md
    ch128-displayport-mst.md
    ch131-touch-tablet-wayland.md
    ch140-hdmi-dp-audio.md
    ch158-hdr-linux-display.md
    ch194-cross-stack-integration.md
    ch198-dbus-modern-linux-ipc.md
  part-07a-gpu-apis/
    ch24-vulkan-egl-application-developers.md
    ch25-gpu-compute.md
    ch26-hardware-video.md
    ch27-vr-ar.md
    ch76-modern-vulkan-extensions.md
    ch106-vulkan-memory-model.md
    ch127-mesh-shaders-vrs.md
    ch133-vulkan-compute-queues.md
    ch135-vulkan-ray-tracing.md
    ch141-vulkan-cooperative-matrices.md
    ch148-vulkan-synchronisation.md
    ch150-egl-architecture-dmabuf.md
    ch152-rust-gpu-ecosystem.md
    ch154-gpu-driven-rendering.md
    ch157-vulkan-descriptor-binding.md
    ch165-vulkan-video.md
    ch173-vk-ext-shader-object.md
    ch192-vk-ext-device-generated-commands.md
    ch204-shader-algorithm-catalog.md
    ch208-gpu-geometry-algorithms.md
  part-07b-multimedia-frameworks/
    ch38-pipewire.md
    ch38b-alsa-linux-audio-subsystem.md
    ch39-qt-gtk-rendering.md
    ch47-font-text-rendering.md
    ch50-vulkan-video.md
    ch96-libcamera.md
    ch111-flatpak-graphics.md
    ch114-opencv-gpu-vision.md
    ch206-sdl-multimedia.md
    ch209-openslam.md
    ch210-slam-theory-sota.md
    ch211-ros2-sensor-perception-pipeline.md
  part-08-gaming-layer/
    ch28-windows-compatibility.md
    ch29-upscaling-effects-overlays.md
    ch56-ray-tracing-linux.md
    ch78-gamescope-steam-deck.md
    ch104-dxvk-vkd3d-proton.md
    ch167-ntsync-nt-synchronization-linux.md
    ch171-linux-gaming-anti-cheat.md
  part-09-tooling-contributing/
    ch30-debugging-profiling.md
    ch31-conformance-regression-testing.md
    ch32-contributing.md
    ch55-gpu-containers-cloud.md
    ch79-remote-display-streaming.md
    ch80-gpu-security.md
    ch89-gpu-virtualization.md
    ch93-gpu-performance-analysis.md
    ch107-headless-rendering.md
    ch109-mesa-testing-ci.md
    ch122-dkms-out-of-tree-modules.md
    ch125-renderdoc-linux.md
    ch136-wsl2-linux-graphics.md
    ch137-gpu-performance-profiling.md
    ch153-obs-studio-gpu-pipeline.md
    ch180-gpu-reverse-engineering.md
  part-10-browser-rendering-stack/
    ch33-chromium-gpu-architecture.md
    ch34-angle-webgl.md
    ch35-dawn-webgpu.md
    ch36-chromium-compositor.md
    ch37-skia-2d-rendering.md
    ch52-firefox-webrender.md
    ch98-webassembly-webgpu.md
    ch146-webcodecs-browser-hardware-acceleration.md
    ch147-chrome-firefox-vaapi-video-decode.md
    ch168-webnn-web-neural-network-api.md
    ch193-tauri-webkitgtk-desktop.md
    ch195-browser-image-formats.md
    ch203-webxr.md
  part-11-engine-creative-tools/
    ch40-bevy-wgpu.md
    ch41-godot4-rendering-device.md
    ch42-blender-gpu.md
    ch97-unreal-engine-5.md
    ch176-opencascade-cad-kernel.md
    ch190-vtk-scientific-visualization.md
    ch205-blender-ai-mcp.md
  part-12-terminal-graphics/
    ch43-terminal-pixel-protocols.md
    ch44-terminal-gpu-rendering.md
    ch45-terminal-compositor-integration.md
    ch174-wezterm-alacritty-architecture.md
    ch178-pty-tty-kernel-layer.md
    ch192-ratatui-tui-framework.md
  part-13-video-streaming/
    ch57-ffmpeg.md
    ch58-gstreamer.md
    ch59-deepstream.md
    ch60-video-codecs.md
    ch60b-video-streaming-protocols.md
    ch60c-webrtc-server-infrastructure.md
    ch60d-bittorrent-streaming.md
    ch142-v4l2-media-subsystem.md
    ch189-vlc-architecture-gpu-linux.md
  part-14-khronos-ecosystem/
    ch61-spirv-ecosystem.md
    ch62-sycl.md
    ch63-ktx2-texture-compression.md
    ch64-gltf.md
    ch65-vulkan-sc-openvx.md
    ch110-spirv-tooling.md
    ch134-opencl-linux.md
  part-15-nvidia-stack/
    ch66-cuda-runtime.md
    ch67-optix.md
    ch68-dlss-neural-rendering.md
    ch69-omniverse-usd.md
    ch70-rtx-kit.md
    ch117-slang-differentiable-shading.md
  part-16-intel-stack/
    ch71-intel-xe-arc.md
  part-17-amd-ecosystem/
    ch72-amd-fidelityfx-tools.md
    ch143-radv-internals.md
    ch170-amdvlk-vs-radv.md
  part-18-rendering-abstractions/
    ch81-sdl3-gpu.md
    ch82-vma-vulkan-helpers.md
    ch83-filament.md
    ch84-bgfx-render-graph.md
    ch113-cgal-computational-geometry.md
  part-19-android-graphics/
    ch85-android-surfaceflinger.md
    ch86-android-vulkan.md
    ch87-android-ar-arcore.md
    ch161-android-game-development-kit.md
    ch164-android-runtime-native-interop.md
    ch166-android-xr.md
    ch191-litert-mediapipe.md
  part-20-ai-inference/
    ch48-rocm-ml-linux.md
    ch88-npu-ai-accelerators.md
    ch94-comfyui-comfyscript.md
    ch108-rocm-hip.md
    ch115-nerfstudio-gaussian-splatting.md
    ch124-llm-inference-linux.md
    ch199-jupyter-internals.md
    ch212-open3d-pytorch3d-kaolin.md
  part-21-platform-legacy/
    ch95-x11-xorg-dri-legacy.md
    ch103-history-design-philosophy.md
    ch197-linux-windows-macos-graphics-comparison.md
  part-27-display-hardware-connectors/
    ch181-modern-display-interface-standards.md
    ch182-digital-display-connectors-physical-layer.md
    ch183-edid-displayid-capability-discovery.md
    ch184-embedded-displayport-laptop-panel.md
    ch185-wireless-display-linux.md
    ch186-video-pixel-formats-signal-encoding.md
    ch187-hdmi-cec-linux-subsystem.md
    ch188-display-power-states-dpms-psr.md
  part-28-linux-multimedia/
    ch213-voip-on-linux.md
    ch214-bluetooth-audio.md
    ch215-mpv-architecture.md
    ch216-speech-synthesis-asr.md
    ch217-dlna-home-theater.md
    ch218-srt-broadcast-streaming.md
    ch219-midi-synthesis-music-production.md
  part-29-graphics-algorithms/
    part-intro.md
    ch204-shader-algorithm-catalog.md
    ch205-shader-gi-and-materials.md
    ch206-shader-raytracing-and-procedural.md
    ch207-shader-vfx-postprocess-compute.md
    ch208-gpu-geometry-algorithms.md
    ch209-gpu-spatial-differential-animation.md
    ch210-gpu-physics-and-volumetric.md
    ch211-gpu-terrain-raytracing-pointcloud.md
    ch212-gpu-neural-specialized-primitives.md
    ch220-gpu-image-processing-algorithms.md
    ch221-gpu-algorithm-performance.md
    ch222-computational-geometry-algorithms.md
    ch223-gpu-video-processing-algorithms.md
    ch224-3d-shape-analysis-algorithms.md
    ch225-computational-topology-algorithms.md
    ch226-gpu-linear-algebra-sparse-solvers.md
    ch227-gpu-rng-monte-carlo.md
    ch228-gpu-graph-algorithms.md
    ch229-gpu-ml-inference-algorithms.md
    ch230-gpu-signal-processing-audio-dsp.md
    ch231-gpu-compression-algorithms.md
    ch232-gpu-generative-ai-llm-inference.md
  appendices/
    appendix-a-glossary.md
    appendix-b-environment-variables.md
    appendix-c-version-matrix.md
    appendix-d-gpu-capability-comparison.md
    appendix-e-contributing-checklists.md
    appendix-f-virtio-gpu-virtualisation.md
    appendix-g-sync-reference.md
    appendix-h-drm-format-modifiers.md
    appendix-i-wayland-protocols-matrix.md
    appendix-j-debugging-quick-reference.md
    appendix-k-remote-display.md
    appendix-l-shader-toolchain-matrix.md
    appendix-m-kernel-config-reference.md
    appendix-n-vulkan-linux-platform-extensions.md
    appendix-o-spirv-binary-format-reference.md
    appendix-p-wgsl-language-reference.md
    appendix-q-shader-language-comparison.md
    appendix-r-egl-platform-reference.md
    appendix-s-drm-kms-ioctl-reference.md
    appendix-t-terminal-graphics-protocol-reference.md
    appendix-u-webgpu-api-reference.md
```

## How to Read

Start with [intro.md](intro.md) for curated reading paths by audience. Each part directory contains a `part-intro.md` that summarises the part's scope and links to its chapters. The master chapter outline with per-chapter bullet points is in [plan.md](plan.md). For a forward-looking synthesis of where the stack is heading, read [conclusion.md](conclusion.md), which weaves the roadmap sections from all part introductions into a coherent picture across short-, medium-, and long-term horizons.

Suggested paths:
- **Kernel/driver developer:** Parts I → II → III → IV → V
- **Vulkan/compute developer:** Parts IV → V → VII-A → XIV → XVIII
- **Browser engineer:** Parts IV → VI-A → X
- **Gaming/compatibility:** Parts VII-A → VIII → XI → XV
- **Terminal developer:** Parts VI-A → XII
- **AI/ML practitioner:** Parts VII-A → XV → XX
- **Multimedia/audio developer:** Parts VII-B → XIII → XXVIII

## Writing Standards

- **Accuracy first.** Code snippets reference real upstream source pinned to a commit or release tag. API signatures are verified against current kernel and Mesa trees.
- **References.** Every non-trivial claim and code excerpt carries an inline `[Source](url)` link to kernel.org, gitlab.freedesktop.org, or the relevant upstream repository.
- **Chapter structure.** Each chapter opens with a scope paragraph, a local table of contents, main sections, and a closing **Integrations** section cross-linking related chapters.
- **Code blocks.** Every block is labelled with the language (`c`, `rust`, `bash`, `glsl`, `spirv`, etc.) and includes enough context (file path, function name, struct) for a reader to locate the source.
- **Depth target.** ~6,000–8,000 words (15–20 pages) per chapter; 1–2 pages per part introduction.

## Contributing

Corrections, additions, and new chapter proposals are welcome. Before opening a pull request, read [Ch 32: Contributing to the Linux Graphics Stack](chapters/part-09-tooling-contributing/ch32-contributing.md) and [Appendix E: Contributing Checklists](chapters/appendices/appendix-e-contributing-checklists.md). All contributions must meet the accuracy and reference standards above.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
