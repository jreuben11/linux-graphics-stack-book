# The Linux Graphics Stack: From Kernel to Compositor, Browser, and Terminal

An expert-level technical reference (~1,100 pages, 167 chapters) tracing every layer of the Linux graphics stack — from the DRM kernel subsystem and GPU memory management through Mesa's shader compilers, Wayland compositors, application APIs, the browser rendering pipeline, game compatibility layers, terminal pixel protocols, GPU-accelerated video streaming, AI inference, and both open and proprietary GPU ecosystems. Each chapter is written for practitioners: code snippets reference real upstream source at specific commits or releases, API signatures are verified against current kernel and Mesa trees, and architectural claims cite primary sources.

By the final chapter the reader holds a continuous mental model from a `DRM_IOCTL_MODE_ATOMIC` kernel call all the way to a photon leaving a display panel, and from a WebGPU `drawIndexed` call in JavaScript all the way to the same panel.

## Audiences

- **Systems and driver developers** — kernel internals, DRM/Mesa architecture, driver implementation
- **Graphics application developers** — the stack beneath Vulkan, EGL, VA-API, and OpenXR
- **Browser and web platform engineers** — how Chromium/Chrome maps WebGPU, WebGL, and compositing onto Linux hardware
- **Terminal and TUI developers** — GPU-rendered terminal emulators, Sixel, Kitty Graphics Protocol, Ghostty

## Parts and Chapters

| # | Part | Chapters | Count |
|---|------|----------|-------|
| I | The Kernel Layer | Ch 1–4, 51, 102, 120–121, 129, 139, 144, 149, 162–163 | 14 |
| II | GPU Drivers | Ch 5–6, 49, 73, 90, 92, 99–100, 116, 126, 155, 160 | 12 |
| III | The Nouveau Story | Ch 7–11, 10a–10b, 118 | 7 |
| IV | Mesa Architecture | Ch 12–17, 77, 91, 119, 156, 159 | 11 |
| V | Mesa GPU Drivers | Ch 18–19 | 2 |
| VI | The Display Stack | Ch 20–23, 46, 53–54, 74–75, 101, 105, 112, 123, 128, 130–132, 138, 140, 145, 151, 158 | 22 |
| VII | Application APIs & Middleware | Ch 24–27, 38–39, 47, 50, 76, 96, 106, 111, 114, 127, 133, 135, 141, 148, 150, 152, 154, 157, 165 | 23 |
| VIII | Gaming Layer | Ch 28–29, 56, 78, 104 | 5 |
| IX | Tooling & Contributing | Ch 30–32, 55, 79–80, 89, 93, 107, 109, 122, 125, 136–137, 153 | 15 |
| X | Browser Rendering Stack | Ch 33–37, 52, 98, 146–147 | 9 |
| XI | Engines & Creative Tools | Ch 40–42, 97, 176 | 5 |
| XII | Terminal Graphics | Ch 43–45 | 3 |
| XIII | Video Streaming | Ch 57–60, 60b, 142 | 6 |
| XIV | Khronos Extended Ecosystem | Ch 61–65, 110, 134 | 7 |
| XV | NVIDIA Proprietary Stack | Ch 66–70, 117 | 6 |
| XVI | Intel Open Graphics Stack | Ch 71 | 1 |
| XVII | AMD Developer Ecosystem | Ch 72, 143 | 2 |
| XVIII | Rendering Abstraction Libraries | Ch 81–84, 113 | 5 |
| XIX | Android Graphics | Ch 85–87, 166, 191 | 5 |
| XX | AI/ML Inference on Linux | Ch 48, 88, 94, 108, 115, 124 | 6 |
| XXI | Platform, Legacy & History | Ch 95, 103 | 2 |
| — | Appendices | A–M (13 appendices) | 13 |

**Total: 167 chapters + 13 appendices**

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
  part-06-display-stack/
    ch20-wayland-protocol-fundamentals.md
    ch21-building-compositors-wlroots.md
    ch22-production-compositors.md
    ch23-legacy-sandboxed-app-support.md
    ch46-wayland-protocol-ecosystem.md
    ch53-display-calibration-colord.md
    ch54-linux-input-stack.md
    ch74-hdr-wide-color-gamut.md
    ch75-explicit-gpu-sync.md
    ch101-color-science-icc.md
    ch105-font-rendering.md
    ch112-vrr-freesync-frame-pacing.md
    ch123-screen-capture-remote-desktop.md
    ch128-displayport-mst.md
    ch130-wayland-protocol-dev.md
    ch131-touch-tablet-wayland.md
    ch132-wayland-security.md
    ch138-wayland-fractional-scaling.md
    ch140-hdmi-dp-audio.md
    ch145-xwayland-architecture.md
    ch151-wayland-text-input-ime.md
    ch158-hdr-linux-display.md
  part-07-application-apis-middleware/
    ch24-vulkan-egl-application-developers.md
    ch25-gpu-compute.md
    ch26-hardware-video.md
    ch27-vr-ar.md
    ch38-pipewire.md
    ch39-qt-gtk-rendering.md
    ch47-font-text-rendering.md
    ch50-vulkan-video.md
    ch76-modern-vulkan-extensions.md
    ch96-libcamera.md
    ch106-vulkan-memory-model.md
    ch111-flatpak-graphics.md
    ch114-opencv-gpu-vision.md
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
  part-08-gaming-layer/
    ch28-windows-compatibility.md
    ch29-upscaling-effects-overlays.md
    ch56-ray-tracing-linux.md
    ch78-gamescope-steam-deck.md
    ch104-dxvk-vkd3d-proton.md
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
  part-11-engine-creative-tools/
    ch40-bevy-wgpu.md
    ch41-godot4-rendering-device.md
    ch42-blender-gpu.md
    ch97-unreal-engine-5.md
    ch176-opencascade-cad-kernel.md
  part-12-terminal-graphics/
    ch43-terminal-pixel-protocols.md
    ch44-terminal-gpu-rendering.md
    ch45-terminal-compositor-integration.md
  part-13-video-streaming/
    ch57-ffmpeg.md
    ch58-gstreamer.md
    ch59-deepstream.md
    ch60-video-codecs.md
    ch60b-video-streaming-protocols.md
    ch142-v4l2-media-subsystem.md
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
    ch166-android-xr.md
    ch191-litert-mediapipe.md
  part-20-ai-inference/
    ch48-rocm-ml-linux.md
    ch88-npu-ai-accelerators.md
    ch94-comfyui-comfyscript.md
    ch108-rocm-hip.md
    ch115-nerfstudio-gaussian-splatting.md
    ch124-llm-inference-linux.md
  part-21-platform-legacy/
    ch95-x11-xorg-dri-legacy.md
    ch103-history-design-philosophy.md
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
```

## How to Read

Start with [intro.md](intro.md) for curated reading paths by audience. Each part directory contains a `part-intro.md` that summarises the part's scope and links to its chapters. The master chapter outline with per-chapter bullet points is in [plan.md](plan.md).

Suggested paths:
- **Kernel/driver developer:** Parts I → II → III → IV → V
- **Vulkan/compute developer:** Parts IV → V → VII → XIV → XVIII
- **Browser engineer:** Parts IV → VI → X
- **Gaming/compatibility:** Parts VII → VIII → XI → XV
- **Terminal developer:** Parts VI → XII
- **AI/ML practitioner:** Parts VII → XV → XX

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
