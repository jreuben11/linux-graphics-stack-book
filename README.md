# The Linux Graphics Stack: From Kernel to Compositor and Browser

An expert-level technical book (~700 pages) covering the Linux graphics stack from DRM kernel drivers through Mesa, Wayland compositors, application APIs, and the browser rendering pipeline.

**Audiences:** Systems and driver developers · Graphics application developers · Browser and web platform engineers

The master outline is in [plan.md](plan.md).

---

## Part I — The Kernel Layer

| Chapter | File |
|---|---|
| Ch 1: DRM Architecture & the Driver Model | [ch01-drm-architecture.md](chapters/part-01-kernel-layer/ch01-drm-architecture.md) |
| Ch 2: KMS: The Display Pipeline | [ch02-kms-display-pipeline.md](chapters/part-01-kernel-layer/ch02-kms-display-pipeline.md) |
| Ch 3: Advanced Display Features | [ch03-advanced-display-features.md](chapters/part-01-kernel-layer/ch03-advanced-display-features.md) |
| Ch 4: GPU Memory Management | [ch04-gpu-memory-management.md](chapters/part-01-kernel-layer/ch04-gpu-memory-management.md) |

## Part II — GPU Drivers

| Chapter | File |
|---|---|
| Ch 5: x86 GPU Drivers | [ch05-x86-gpu-drivers.md](chapters/part-02-gpu-drivers/ch05-x86-gpu-drivers.md) |
| Ch 6: ARM & Embedded GPU Drivers | [ch06-arm-embedded-gpu-drivers.md](chapters/part-02-gpu-drivers/ch06-arm-embedded-gpu-drivers.md) |

## Part III — The Nouveau Story

| Chapter | File |
|---|---|
| Ch 7: Reverse Engineering NVIDIA: History and Methodology | [ch07-reverse-engineering-nvidia.md](chapters/part-03-nouveau-story/ch07-reverse-engineering-nvidia.md) |
| Ch 8: The Nouveau Kernel Driver: nvkm Architecture | [ch08-nouveau-kernel-driver.md](chapters/part-03-nouveau-story/ch08-nouveau-kernel-driver.md) |
| Ch 9: GSP-RM, Firmware, and the nvidia-open Connection | [ch09-gsp-rm-firmware.md](chapters/part-03-nouveau-story/ch09-gsp-rm-firmware.md) |
| Ch 10: NVK: Building a Vulkan Driver from Scratch | [ch10-nvk-vulkan-driver.md](chapters/part-03-nouveau-story/ch10-nvk-vulkan-driver.md) |
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

## Part VII — Application APIs & Middleware

| Chapter | File |
|---|---|
| Ch 24: Vulkan and EGL for Application Developers | [ch24-vulkan-egl-application-developers.md](chapters/part-07-application-apis-middleware/ch24-vulkan-egl-application-developers.md) |
| Ch 25: GPU Compute | [ch25-gpu-compute.md](chapters/part-07-application-apis-middleware/ch25-gpu-compute.md) |
| Ch 26: Hardware Video | [ch26-hardware-video.md](chapters/part-07-application-apis-middleware/ch26-hardware-video.md) |
| Ch 27: VR & AR | [ch27-vr-ar.md](chapters/part-07-application-apis-middleware/ch27-vr-ar.md) |
| Ch 38: PipeWire and the Video Session Layer | [ch38-pipewire.md](chapters/part-07-application-apis-middleware/ch38-pipewire.md) |
| Ch 39: Qt and GTK GPU Rendering | [ch39-qt-gtk-rendering.md](chapters/part-07-application-apis-middleware/ch39-qt-gtk-rendering.md) |

## Part VIII — Gaming Layer

| Chapter | File |
|---|---|
| Ch 28: Windows Compatibility | [ch28-windows-compatibility.md](chapters/part-08-gaming-layer/ch28-windows-compatibility.md) |
| Ch 29: Upscaling, Effects & Overlays | [ch29-upscaling-effects-overlays.md](chapters/part-08-gaming-layer/ch29-upscaling-effects-overlays.md) |

## Part IX — Tooling & Contributing

| Chapter | File |
|---|---|
| Ch 30: Debugging & Profiling | [ch30-debugging-profiling.md](chapters/part-09-tooling-contributing/ch30-debugging-profiling.md) |
| Ch 31: Conformance & Regression Testing | [ch31-conformance-regression-testing.md](chapters/part-09-tooling-contributing/ch31-conformance-regression-testing.md) |
| Ch 32: Contributing to the Linux Graphics Stack | [ch32-contributing.md](chapters/part-09-tooling-contributing/ch32-contributing.md) |

## Part X — The Browser Rendering Stack

| Chapter | File |
|---|---|
| Ch 33: Chromium's Multi-Process GPU Architecture | [ch33-chromium-gpu-architecture.md](chapters/part-10-browser-rendering-stack/ch33-chromium-gpu-architecture.md) |
| Ch 34: ANGLE — WebGL on Linux | [ch34-angle-webgl.md](chapters/part-10-browser-rendering-stack/ch34-angle-webgl.md) |
| Ch 35: Dawn and WebGPU | [ch35-dawn-webgpu.md](chapters/part-10-browser-rendering-stack/ch35-dawn-webgpu.md) |
| Ch 36: The Chromium Compositor — CC and Viz | [ch36-chromium-compositor.md](chapters/part-10-browser-rendering-stack/ch36-chromium-compositor.md) |
| Ch 37: Skia and 2D Rendering | [ch37-skia-2d-rendering.md](chapters/part-10-browser-rendering-stack/ch37-skia-2d-rendering.md) |

## Part XI — Engines & Creative Tools

| Chapter | File |
|---|---|
| Ch 40: Bevy and wgpu — A Rust-Native Vulkan Client | [ch40-bevy-wgpu.md](chapters/part-11-engine-creative-tools/ch40-bevy-wgpu.md) |
| Ch 41: Godot 4 — RenderingDevice and the Explicit Vulkan Path | [ch41-godot4-rendering-device.md](chapters/part-11-engine-creative-tools/ch41-godot4-rendering-device.md) |
| Ch 42: Blender — Cycles, EEVEE Next, and GPU Compute on Linux | [ch42-blender-gpu.md](chapters/part-11-engine-creative-tools/ch42-blender-gpu.md) |

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
| Monado (OpenXR) | https://gitlab.freedesktop.org/monado/monado |
| Chromium | https://source.chromium.org/chromium |
| Dawn (WebGPU) | https://dawn.googlesource.com/dawn |
| ANGLE | https://chromium.googlesource.com/angle/angle |
| Skia | https://skia.googlesource.com/skia |
| Kernel GPU docs | https://www.kernel.org/doc/html/latest/gpu/ |

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
