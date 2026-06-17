# The Linux Graphics Stack: From Kernel to Compositor, Browser, and Terminal

This is an expert-level technical reference (~800 pages) tracing every layer of the Linux graphics stack — from the DRM kernel subsystem and GPU memory management through Mesa's shader compilers, Wayland compositors, application APIs, the browser rendering pipeline, game compatibility layers, terminal pixel protocols, GPU-accelerated video streaming, and NVIDIA's proprietary SDK ecosystem. Each chapter is written for practitioners: code snippets reference real upstream source at specific commits or releases, API signatures are verified against current kernel and Mesa trees, and architectural claims cite primary sources. The book targets four overlapping audiences — systems and driver developers who live in kernel space and Mesa internals; graphics application developers working with Vulkan, EGL, VA-API, or OpenXR; browser and web platform engineers mapping WebGPU and WebGL onto Linux hardware; and terminal and TUI developers building GPU-rendered terminal emulators. By the final chapter the reader holds a continuous mental model from a `DRM_IOCTL_MODE_ATOMIC` kernel call all the way to a photon leaving a display panel, and from a WebGPU `drawIndexed` call in JavaScript all the way to the same panel.

**Audiences:** Systems and driver developers · Graphics application developers · Browser and web platform engineers · Terminal and TUI developers

The master outline (with full per-chapter bullet points) is in [plan.md](plan.md).

---

## Part I — The Kernel Layer

| Chapter | File |
|---|---|
| Ch 1: DRM Architecture & the Driver Model | [ch01-drm-architecture.md](chapters/part-01-kernel-layer/ch01-drm-architecture.md) |
| Ch 2: KMS: The Display Pipeline | [ch02-kms-display-pipeline.md](chapters/part-01-kernel-layer/ch02-kms-display-pipeline.md) |
| Ch 3: Advanced Display Features | [ch03-advanced-display-features.md](chapters/part-01-kernel-layer/ch03-advanced-display-features.md) |
| Ch 4: GPU Memory Management | [ch04-gpu-memory-management.md](chapters/part-01-kernel-layer/ch04-gpu-memory-management.md) |
| Ch 51: GPU Power Management and Thermal | [ch51-gpu-power-management.md](chapters/part-01-kernel-layer/ch51-gpu-power-management.md) |

## Part II — GPU Drivers

| Chapter | File |
|---|---|
| Ch 5: x86 GPU Drivers | [ch05-x86-gpu-drivers.md](chapters/part-02-gpu-drivers/ch05-x86-gpu-drivers.md) |
| Ch 6: ARM & Embedded GPU Drivers | [ch06-arm-embedded-gpu-drivers.md](chapters/part-02-gpu-drivers/ch06-arm-embedded-gpu-drivers.md) |
| Ch 49: Multi-GPU and PRIME Render Offload | [ch49-multi-gpu-prime.md](chapters/part-02-gpu-drivers/ch49-multi-gpu-prime.md) |

## Part III — The Nouveau / Open NVIDIA Story

| Chapter | File |
|---|---|
| Ch 7: Reverse Engineering NVIDIA: History and Methodology | [ch07-reverse-engineering-nvidia.md](chapters/part-03-nouveau-story/ch07-reverse-engineering-nvidia.md) |
| Ch 8: The Nouveau Kernel Driver: nvkm Architecture | [ch08-nouveau-kernel-driver.md](chapters/part-03-nouveau-story/ch08-nouveau-kernel-driver.md) |
| Ch 9: GSP-RM, Firmware, and the nvidia-open Connection | [ch09-gsp-rm-firmware.md](chapters/part-03-nouveau-story/ch09-gsp-rm-firmware.md) |
| Ch 10: NVK: Building a Vulkan Driver from Scratch | [ch10-nvk-vulkan-driver.md](chapters/part-03-nouveau-story/ch10-nvk-vulkan-driver.md) |
| Ch 10a: Nova — The Rust NVIDIA Kernel Driver | [ch10-nova-rust-nvidia-driver.md](chapters/part-03-nouveau-story/ch10-nova-rust-nvidia-driver.md) |
| Ch 11: Display, Reclocking, and Power Management | [ch11-display-reclocking-power.md](chapters/part-03-nouveau-story/ch11-display-reclocking-power.md) |

## Part IV — Mesa Architecture

| Chapter | File |
|---|---|
| Ch 12: The Mesa Loader and Driver Dispatch | [ch12-mesa-loader-dispatch.md](chapters/part-04-mesa-architecture/ch12-mesa-loader-dispatch.md) |
| Ch 13: Gallium3D: The OpenGL State Tracker | [ch13-gallium3d.md](chapters/part-04-mesa-architecture/ch13-gallium3d.md) |
| Ch 14: NIR: Mesa's Shader Intermediate Representation | [ch14-nir-shader-ir.md](chapters/part-04-mesa-architecture/ch14-nir-shader-ir.md) |
| Ch 15: ACO: AMD's Optimising Compiler | [ch15-aco-compiler.md](chapters/part-04-mesa-architecture/ch15-aco-compiler.md) |
| Ch 16: Mesa's Vulkan Common Infrastructure | [ch16-mesa-vulkan-common.md](chapters/part-04-mesa-architecture/ch16-mesa-vulkan-common.md) |
| Ch 17: Software Renderers | [ch17-software-renderers.md](chapters/part-04-mesa-architecture/ch17-software-renderers.md) |

## Part V — Mesa GPU Drivers

| Chapter | File |
|---|---|
| Ch 18: Vulkan Drivers | [ch18-vulkan-drivers.md](chapters/part-05-mesa-gpu-drivers/ch18-vulkan-drivers.md) |
| Ch 19: OpenGL and Compatibility Drivers | [ch19-opengl-compatibility-drivers.md](chapters/part-05-mesa-gpu-drivers/ch19-opengl-compatibility-drivers.md) |

## Part VI — The Display Stack

| Chapter | File |
|---|---|
| Ch 20: Wayland Protocol Fundamentals | [ch20-wayland-protocol-fundamentals.md](chapters/part-06-display-stack/ch20-wayland-protocol-fundamentals.md) |
| Ch 21: Building Compositors with wlroots | [ch21-building-compositors-wlroots.md](chapters/part-06-display-stack/ch21-building-compositors-wlroots.md) |
| Ch 22: Production Compositors | [ch22-production-compositors.md](chapters/part-06-display-stack/ch22-production-compositors.md) |
| Ch 23: Legacy and Sandboxed App Support | [ch23-legacy-sandboxed-app-support.md](chapters/part-06-display-stack/ch23-legacy-sandboxed-app-support.md) |
| Ch 46: The Evolving Wayland Protocol Ecosystem | [ch46-wayland-protocol-ecosystem.md](chapters/part-06-display-stack/ch46-wayland-protocol-ecosystem.md) |
| Ch 53: Display Calibration and colord | [ch53-display-calibration-colord.md](chapters/part-06-display-stack/ch53-display-calibration-colord.md) |
| Ch 54: The Linux Input Stack | [ch54-linux-input-stack.md](chapters/part-06-display-stack/ch54-linux-input-stack.md) |

## Part VII — Application APIs & Middleware

| Chapter | File |
|---|---|
| Ch 24: Vulkan and EGL for Application Developers | [ch24-vulkan-egl-application-developers.md](chapters/part-07-application-apis-middleware/ch24-vulkan-egl-application-developers.md) |
| Ch 25: GPU Compute | [ch25-gpu-compute.md](chapters/part-07-application-apis-middleware/ch25-gpu-compute.md) |
| Ch 26: Hardware Video | [ch26-hardware-video.md](chapters/part-07-application-apis-middleware/ch26-hardware-video.md) |
| Ch 27: VR & AR | [ch27-vr-ar.md](chapters/part-07-application-apis-middleware/ch27-vr-ar.md) |
| Ch 38: PipeWire and the Video Session Layer | [ch38-pipewire.md](chapters/part-07-application-apis-middleware/ch38-pipewire.md) |
| Ch 39: Qt and GTK GPU Rendering | [ch39-qt-gtk-rendering.md](chapters/part-07-application-apis-middleware/ch39-qt-gtk-rendering.md) |
| Ch 47: Font and Text Rendering Pipeline | [ch47-font-text-rendering.md](chapters/part-07-application-apis-middleware/ch47-font-text-rendering.md) |
| Ch 48: ROCm and Machine Learning on Linux GPUs | [ch48-rocm-ml-linux.md](chapters/part-07-application-apis-middleware/ch48-rocm-ml-linux.md) |
| Ch 50: Vulkan Video Extensions | [ch50-vulkan-video.md](chapters/part-07-application-apis-middleware/ch50-vulkan-video.md) |

## Part VIII — Gaming Layer

| Chapter | File |
|---|---|
| Ch 28: Windows Compatibility (Wine, DXVK, VKD3D-Proton, RTX Remix) | [ch28-windows-compatibility.md](chapters/part-08-gaming-layer/ch28-windows-compatibility.md) |
| Ch 29: Upscaling, Effects & Overlays | [ch29-upscaling-effects-overlays.md](chapters/part-08-gaming-layer/ch29-upscaling-effects-overlays.md) |
| Ch 56: Ray Tracing on Linux | [ch56-ray-tracing-linux.md](chapters/part-08-gaming-layer/ch56-ray-tracing-linux.md) |

## Part IX — Tooling & Contributing

| Chapter | File |
|---|---|
| Ch 30: Debugging & Profiling (incl. NvPerf SDK) | [ch30-debugging-profiling.md](chapters/part-09-tooling-contributing/ch30-debugging-profiling.md) |
| Ch 31: Conformance & Regression Testing | [ch31-conformance-regression-testing.md](chapters/part-09-tooling-contributing/ch31-conformance-regression-testing.md) |
| Ch 32: Contributing to the Linux Graphics Stack | [ch32-contributing.md](chapters/part-09-tooling-contributing/ch32-contributing.md) |
| Ch 55: GPU Containers and Cloud Compute | [ch55-gpu-containers-cloud.md](chapters/part-09-tooling-contributing/ch55-gpu-containers-cloud.md) |

## Part X — The Browser Rendering Stack

| Chapter | File |
|---|---|
| Ch 33: Chromium's Multi-Process GPU Architecture | [ch33-chromium-gpu-architecture.md](chapters/part-10-browser-rendering-stack/ch33-chromium-gpu-architecture.md) |
| Ch 34: ANGLE — WebGL on Linux | [ch34-angle-webgl.md](chapters/part-10-browser-rendering-stack/ch34-angle-webgl.md) |
| Ch 35: Dawn and WebGPU | [ch35-dawn-webgpu.md](chapters/part-10-browser-rendering-stack/ch35-dawn-webgpu.md) |
| Ch 36: The Chromium Compositor — CC and Viz | [ch36-chromium-compositor.md](chapters/part-10-browser-rendering-stack/ch36-chromium-compositor.md) |
| Ch 37: Skia and 2D Rendering | [ch37-skia-2d-rendering.md](chapters/part-10-browser-rendering-stack/ch37-skia-2d-rendering.md) |
| Ch 52: Firefox and WebRender | [ch52-firefox-webrender.md](chapters/part-10-browser-rendering-stack/ch52-firefox-webrender.md) |

## Part XI — Engines & Creative Tools

| Chapter | File |
|---|---|
| Ch 40: Bevy and wgpu — A Rust-Native Vulkan Client | [ch40-bevy-wgpu.md](chapters/part-11-engine-creative-tools/ch40-bevy-wgpu.md) |
| Ch 41: Godot 4 — RenderingDevice and the Explicit Vulkan Path | [ch41-godot4-rendering-device.md](chapters/part-11-engine-creative-tools/ch41-godot4-rendering-device.md) |
| Ch 42: Blender — Cycles, EEVEE Next, and GPU Compute on Linux | [ch42-blender-gpu.md](chapters/part-11-engine-creative-tools/ch42-blender-gpu.md) |

## Part XII — Terminal Graphics

| Chapter | File |
|---|---|
| Ch 43: Terminal Pixel Protocols — Sixel, Kitty, and iTerm2 | [ch43-terminal-pixel-protocols.md](chapters/part-12-terminal-graphics/ch43-terminal-pixel-protocols.md) |
| Ch 44: Terminal GPU Rendering Architectures | [ch44-terminal-gpu-rendering.md](chapters/part-12-terminal-graphics/ch44-terminal-gpu-rendering.md) |
| Ch 45: Terminal Integration with the Compositor Stack | [ch45-terminal-compositor-integration.md](chapters/part-12-terminal-graphics/ch45-terminal-compositor-integration.md) |

## Part XIII — Video Streaming on Linux

| Chapter | File |
|---|---|
| Ch 57: FFmpeg Architecture and Programming | [ch57-ffmpeg.md](chapters/part-13-video-streaming/ch57-ffmpeg.md) |
| Ch 58: GStreamer: Pipeline-Based Multimedia | [ch58-gstreamer.md](chapters/part-13-video-streaming/ch58-gstreamer.md) |
| Ch 59: NVIDIA DeepStream SDK | [ch59-deepstream.md](chapters/part-13-video-streaming/ch59-deepstream.md) |
| Ch 60: Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs | [ch60-video-codecs.md](chapters/part-13-video-streaming/ch60-video-codecs.md) |
| Ch 60b: Video Streaming Protocols and Adaptive Bitrate Delivery | [ch60b-video-streaming-protocols.md](chapters/part-13-video-streaming/ch60b-video-streaming-protocols.md) |

## Part XIV — Khronos Extended Ecosystem

| Chapter | File |
|---|---|
| Ch 61: SPIR-V Ecosystem in Depth | [ch61-spirv-ecosystem.md](chapters/part-14-khronos-ecosystem/ch61-spirv-ecosystem.md) |
| Ch 62: SYCL 2020 and Portable Heterogeneous Compute | [ch62-sycl.md](chapters/part-14-khronos-ecosystem/ch62-sycl.md) |
| Ch 63: KTX2, Basis Universal, and GPU Texture Compression | [ch63-ktx2-texture-compression.md](chapters/part-14-khronos-ecosystem/ch63-ktx2-texture-compression.md) |
| Ch 64: glTF 2.0 — The 3D Asset Pipeline Standard | [ch64-gltf.md](chapters/part-14-khronos-ecosystem/ch64-gltf.md) |
| Ch 65: Vulkan Safety Critical and OpenVX | [ch65-vulkan-sc-openvx.md](chapters/part-14-khronos-ecosystem/ch65-vulkan-sc-openvx.md) |

## Part XV — NVIDIA Proprietary Graphics Stack

| Chapter | File |
|---|---|
| Ch 66: CUDA Runtime, Streams, and NVRTC | [ch66-cuda-runtime.md](chapters/part-15-nvidia-stack/ch66-cuda-runtime.md) |
| Ch 67: OptiX 9 — NVIDIA's Ray Tracing Framework | [ch67-optix.md](chapters/part-15-nvidia-stack/ch67-optix.md) |
| Ch 68: DLSS 4, Neural Rendering, and Frame Generation (incl. TensorRT) | [ch68-dlss-neural-rendering.md](chapters/part-15-nvidia-stack/ch68-dlss-neural-rendering.md) |
| Ch 69: NVIDIA Omniverse, OpenUSD, and the RTX Renderer (incl. PhysX 5, Warp, Project Zorah) | [ch69-omniverse-usd.md](chapters/part-15-nvidia-stack/ch69-omniverse-usd.md) |
| Ch 70: RTX Kit — RTXDI, RTXGI, NRD, RTXNS, and RTXNTC | [ch70-rtx-kit.md](chapters/part-15-nvidia-stack/ch70-rtx-kit.md) |

## Part XVI — Intel Open Graphics Stack

| Chapter | File |
|---|---|
| Ch 71: Intel Xe Kernel Driver, Arc GPU Architecture, and the Intel Open Stack | [ch71-intel-xe-arc.md](chapters/part-16-intel-stack/ch71-intel-xe-arc.md) |

## Part XVII — AMD Developer Ecosystem

| Chapter | File |
|---|---|
| Ch 72: AMD FidelityFX SDK and Radeon Developer Tools | [ch72-amd-fidelityfx-tools.md](chapters/part-17-amd-ecosystem/ch72-amd-fidelityfx-tools.md) |

## Additional Chapters in Existing Parts

| Chapter | Part | File |
|---|---|---|
| Ch 73: Asahi Linux and the Apple Silicon AGX Driver | Part II | [ch73-asahi-apple-silicon.md](chapters/part-02-gpu-drivers/ch73-asahi-apple-silicon.md) |
| Ch 74: HDR and Wide Color Gamut on Linux | Part VI | [ch74-hdr-wide-color-gamut.md](chapters/part-06-display-stack/ch74-hdr-wide-color-gamut.md) |
| Ch 75: Explicit GPU Synchronisation | Part VI | [ch75-explicit-gpu-sync.md](chapters/part-06-display-stack/ch75-explicit-gpu-sync.md) |
| Ch 76: Modern Vulkan Extensions | Part VII | [ch76-modern-vulkan-extensions.md](chapters/part-07-application-apis-middleware/ch76-modern-vulkan-extensions.md) |
| Ch 77: Shader Source-to-ISA: The Complete Compilation Toolchain | Part IV | [ch77-shader-toolchain.md](chapters/part-04-mesa-architecture/ch77-shader-toolchain.md) |
| Ch 78: Gamescope and the Steam Deck | Part VIII | [ch78-gamescope-steam-deck.md](chapters/part-08-gaming-layer/ch78-gamescope-steam-deck.md) |
| Ch 79: Remote Display, Screen Casting, and GPU-Accelerated Game Streaming | Part IX | [ch79-remote-display-streaming.md](chapters/part-09-tooling-contributing/ch79-remote-display-streaming.md) |
| Ch 80: GPU Security: Isolation, Content Protection, and Confidential Computing | Part IX | [ch80-gpu-security.md](chapters/part-09-tooling-contributing/ch80-gpu-security.md) |

## Part XVIII — Rendering Abstraction Libraries

| Chapter | File |
|---|---|
| Ch 81: SDL3 GPU API: A Portable High-Level GPU Abstraction | [ch81-sdl3-gpu.md](chapters/part-18-rendering-abstractions/ch81-sdl3-gpu.md) |
| Ch 82: Vulkan Ecosystem Toolkit: VMA, volk, vk-bootstrap, and Friends | [ch82-vma-vulkan-helpers.md](chapters/part-18-rendering-abstractions/ch82-vma-vulkan-helpers.md) |
| Ch 83: Filament: Google's Physically Based Rendering Engine on Linux | [ch83-filament.md](chapters/part-18-rendering-abstractions/ch83-filament.md) |
| Ch 84: bgfx, Cross-Platform Rendering Abstractions, and the Frame Graph Pattern | [ch84-bgfx-render-graph.md](chapters/part-18-rendering-abstractions/ch84-bgfx-render-graph.md) |

## Part XIX — Android Graphics

| Chapter | File |
|---|---|
| Ch 85: Android Compositor: SurfaceFlinger, HardwareBuffer, and the Buffer Pipeline | [ch85-android-surfaceflinger.md](chapters/part-19-android-graphics/ch85-android-surfaceflinger.md) |
| Ch 86: Vulkan on Android: Drivers, ANGLE, and Mobile GPU Performance | [ch86-android-vulkan.md](chapters/part-19-android-graphics/ch86-android-vulkan.md) |

---

## Appendices

| Appendix | File |
|---|---|
| A: Glossary | [appendix-a-glossary.md](chapters/appendices/appendix-a-glossary.md) |
| B: Environment Variables Reference | [appendix-b-environment-variables.md](chapters/appendices/appendix-b-environment-variables.md) |
| C: Version Matrix | [appendix-c-version-matrix.md](chapters/appendices/appendix-c-version-matrix.md) |
| D: GPU Capability Comparison | [appendix-d-gpu-capability-comparison.md](chapters/appendices/appendix-d-gpu-capability-comparison.md) |
| E: Contributing Checklists | [appendix-e-contributing-checklists.md](chapters/appendices/appendix-e-contributing-checklists.md) |
| F: Virtio-GPU and Virtualisation | [appendix-f-virtio-gpu-virtualisation.md](chapters/appendices/appendix-f-virtio-gpu-virtualisation.md) |
| G: Sync Object Reference | [appendix-g-sync-reference.md](chapters/appendices/appendix-g-sync-reference.md) |
| H: DRM Format Modifiers | [appendix-h-drm-format-modifiers.md](chapters/appendices/appendix-h-drm-format-modifiers.md) |
| I: Wayland Protocols Matrix | [appendix-i-wayland-protocols-matrix.md](chapters/appendices/appendix-i-wayland-protocols-matrix.md) |
| J: Debugging Quick Reference | [appendix-j-debugging-quick-reference.md](chapters/appendices/appendix-j-debugging-quick-reference.md) |
| K: Remote Display and GPU Virtualisation | [appendix-k-remote-display.md](chapters/appendices/appendix-k-remote-display.md) |
| L: Shader Toolchain Matrix | [appendix-l-shader-toolchain-matrix.md](chapters/appendices/appendix-l-shader-toolchain-matrix.md) |
| M: Kernel Configuration Reference | [appendix-m-kernel-config-reference.md](chapters/appendices/appendix-m-kernel-config-reference.md) |

---

## Key Upstream Repositories

| Project | URL |
|---|---|
| Linux kernel DRM | https://github.com/torvalds/linux |
| Mesa | https://gitlab.freedesktop.org/mesa/mesa |
| libdrm | https://gitlab.freedesktop.org/mesa/drm |
| Wayland | https://gitlab.freedesktop.org/wayland/wayland |
| wlroots | https://gitlab.freedesktop.org/wlroots/wlroots |
| Nouveau / NVK | https://gitlab.freedesktop.org/nouveau/mesa |
| Nova (Rust NVIDIA driver) | https://gitlab.freedesktop.org/drm/nova |
| Monado (OpenXR) | https://gitlab.freedesktop.org/monado/monado |
| Chromium | https://source.chromium.org/chromium |
| Dawn (WebGPU) | https://dawn.googlesource.com/dawn |
| ANGLE | https://chromium.googlesource.com/angle/angle |
| Skia | https://skia.googlesource.com/skia |
| Kernel GPU docs | https://www.kernel.org/doc/html/latest/gpu/ |
| Kitty terminal | https://github.com/kovidgoyal/kitty |
| Ghostty | https://github.com/ghostty-org/ghostty |
| foot | https://codeberg.org/dnkl/foot |
| libsixel | https://github.com/libsixel/libsixel |
| FFmpeg | https://github.com/FFmpeg/FFmpeg |
| GStreamer | https://gitlab.freedesktop.org/gstreamer/gstreamer |
| NVIDIA Video Codec SDK | https://github.com/NVIDIA/video-codec-sdk |
| SRT (Haivision) | https://github.com/Haivision/srt |
| OpenUSD | https://github.com/PixarAnimationStudios/OpenUSD |
| PhysX 5 | https://github.com/NVIDIA-Omniverse/PhysX |
| NVIDIA Warp | https://github.com/NVIDIA/warp |
| RTX Kit (RTXDI/RTXGI/NRD/RTXNS/RTXNTC) | https://github.com/NVIDIA-RTX |
| RTX Remix | https://github.com/NVIDIAGameWorks/rtx-remix |
| OptiX samples | https://github.com/NVIDIA/OptiX_Apps |
| Intel Xe kernel driver | https://github.com/torvalds/linux (drivers/gpu/drm/xe/) |
| Intel Media Driver (iHD) | https://github.com/intel/media-driver |
| Level Zero | https://github.com/oneapi-src/level-zero |
| XeSS SDK | https://github.com/intel/xess |
| FidelityFX SDK | https://github.com/GPUOpen-LibrariesAndSDKs/FidelityFX-SDK |
| AMF (Advanced Media Framework) | https://github.com/GPUOpen-LibrariesAndSDKs/AMF |
| RenderDoc | https://github.com/baldurk/renderdoc |
| Asahi Linux kernel | https://github.com/AsahiLinux/linux |
| SDL3 | https://github.com/libsdl-org/SDL |
| Vulkan Memory Allocator | https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator |
| volk | https://github.com/zeux/volk |
| vk-bootstrap | https://github.com/charles-lunarg/vk-bootstrap |
| SPIRV-Reflect | https://github.com/KhronosGroup/SPIRV-Reflect |
| shaderc | https://github.com/google/shaderc |
| Tracy profiler | https://github.com/wolfpld/tracy |
| Filament | https://github.com/google/filament |
| bgfx | https://github.com/bkaradzic/bgfx |
| Gamescope | https://github.com/ValveSoftware/gamescope |
| Sunshine game streaming | https://github.com/LizardByte/Sunshine |
| Moonlight client | https://github.com/moonlight-stream/moonlight-qt |

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
