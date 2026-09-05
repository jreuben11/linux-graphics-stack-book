# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a multi-thousand-page expert-level technical book (283 chapters across 30 parts as of writing): **"The Linux Graphics Stack: From Kernel to Compositor, Browser, and Terminal"**. The title names its origin, but the book's scope has grown beyond graphics rendering to cover everything a GPU does on Linux: the same DRM driver stack and scheduler that draws a desktop frame also runs AI/ML training and inference, hardware video codec pipelines, and real-time audio — treat those as first-class subjects, not asides. The full chapter plan lives in `plan.md`.

**Eight audiences** (chapters signal which perspective is emphasised):
- **Systems and driver developers** — kernel internals, DRM/Mesa architecture, driver implementation
- **Graphics application developers** — Vulkan, EGL, VA-API, OpenXR usage and the stack beneath them
- **Browser and web platform engineers** — how Chromium/Chrome maps WebGPU, WebGL, and compositing onto the Linux graphics stack
- **Terminal and TUI developers** — how terminal emulators render pixel graphics (Sixel, Kitty Graphics Protocol, Ghostty) on top of the compositor stack
- **Production rendering and render farm engineers** — offline/production renderer architecture (path tracing, shading languages, USD/Hydra pipeline integration) and Linux-based render farm infrastructure (Nucleus, NGC containers, multi-GPU distribution)
- **AI/ML systems engineers on Linux GPUs** — compute kernels, tensor throughput, and inference/training infrastructure (ROCm, local LLM inference, NPUs, JAX/PyTorch internals) built on the same driver stack as graphics
- **Video/streaming and codec engineers** — hardware and software video pipelines (FFmpeg internals, GStreamer plugin development, DeepStream, AV1/HEVC codec algorithms) beneath every hardware decode path
- **Linux audio/multimedia engineers** — real-time audio, VoIP and Bluetooth audio, media playback, broadcast streaming, and music production infrastructure on the same kernel and userspace media stack

## Repository Structure

Each chapter becomes its own `.md` file, e.g. `ch01-drm-architecture.md`. Parts may have subdirectories. The master outline is `plan.md` — do not alter it without user instruction.

## Writing Standards

**Accuracy first.** This is a technical reference. Code snippets must compile/run against current upstream versions. When citing kernel, Mesa, or framework source (PyTorch, JAX, ROCm, FFmpeg, GStreamer, etc.), pin the file path and commit/tag or release version.

**References.** Every non-trivial claim and every code excerpt needs an inline reference with a source URL (kernel.org, gitlab.freedesktop.org, arXiv, or the relevant framework/vendor docs, etc.). Use footnote-style or inline `[Source](url)` links.

**Chapter structure.** Each chapter `.md` must open with:
1. A brief scope paragraph naming which audience(s) it targets.
2. A local Table of Contents (heading anchors).
3. Sections following the outline in `plan.md`.
4. An **Integrations** section at the end (as described in `plan.md`) cross-linking to related chapters.

**Code blocks.** Label every block with the language (`c`, `rust`, `bash`, `glsl`, `spirv`, etc.). Include enough context (file path, function name, relevant struct/enum) that a reader can locate the source. Prefer real upstream snippets over invented examples; annotate any simplifications.

**Depth target.** Each chapter is approximately 15–20 pages (≈6,000–8,000 words). Part introductions are 1–2 pages.

## Content Research Approach

Use `WebSearch` and `WebFetch` to retrieve current upstream source, mailing-list discussions, and documentation. Key upstream repositories:

**Kernel, Mesa, and display:**
- Linux kernel DRM: `https://cgit.freedesktop.org/drm/drm-tip` / `https://github.com/torvalds/linux`
- Mesa: `https://gitlab.freedesktop.org/mesa/mesa`
- Wayland/wlroots: `https://gitlab.freedesktop.org/wayland/wayland` / `https://gitlab.freedesktop.org/wlroots/wlroots`
- libdrm: `https://gitlab.freedesktop.org/mesa/drm`
- Nouveau/NVK: `https://gitlab.freedesktop.org/nouveau/mesa` (NVK lives in main Mesa repo)
- Envytools: `https://github.com/envytools/envytools`
- Monado (OpenXR): `https://gitlab.freedesktop.org/monado/monado`

**Browser and terminal:**
- Chromium source: `https://source.chromium.org/chromium`
- Dawn (WebGPU implementation): `https://dawn.googlesource.com/dawn`
- Tint (WGSL compiler): `https://dawn.googlesource.com/dawn/+/refs/heads/main/src/tint/`
- Skia (2D graphics): `https://skia.googlesource.com/skia`
- ANGLE (OpenGL ES → Vulkan/Metal/D3D): `https://chromium.googlesource.com/angle/angle`
- Kitty terminal: `https://github.com/kovidgoyal/kitty` (Kitty Graphics Protocol reference implementation)
- Ghostty: `https://github.com/ghostty-org/ghostty` (GPU-accelerated terminal, libghosty)
- foot: `https://codeberg.org/dnkl/foot` (Wayland-native terminal with Sixel support)
- xterm Sixel: `https://invisible-island.net/xterm/` (reference Sixel implementation)
- libsixel: `https://github.com/libsixel/libsixel` (encoder/decoder library)

**AI/ML compute and inference:**
- ROCm/HIP: `https://github.com/ROCm/ROCm` / `https://github.com/ROCm/HIP`
- PyTorch: `https://github.com/pytorch/pytorch`
- JAX/XLA: `https://github.com/jax-ml/jax` / `https://github.com/openxla/xla`
- NCCL/RCCL: `https://github.com/NVIDIA/nccl` / `https://github.com/ROCm/rccl`
- vLLM / llama.cpp: `https://github.com/vllm-project/vllm` / `https://github.com/ggml-org/llama.cpp`
- Hugging Face: `https://github.com/huggingface`
- arXiv (papers cited for architecture/benchmark claims): `https://arxiv.org`

**Video, audio, and production rendering:**
- FFmpeg: `https://github.com/FFmpeg/FFmpeg`
- GStreamer: `https://gitlab.freedesktop.org/gstreamer/gstreamer`
- PipeWire/WirePlumber: `https://gitlab.freedesktop.org/pipewire/pipewire`
- OpenUSD: `https://github.com/PixarAnimationStudios/OpenUSD`
- OpenColorIO: `https://github.com/AcademySoftwareFoundation/OpenColorIO`

Kernel doc: `https://www.kernel.org/doc/html/latest/gpu/`

## Workflow

1. One chapter per writing session. Research first, then write.
2. Use `WebSearch`/`WebFetch` to verify API signatures, struct definitions, and the kernel/Mesa/framework version where a feature landed.
3. After drafting, check cross-references against `plan.md` **Integrations** bullets to ensure nothing is missed.
4. Do not invent kernel interfaces, Mesa internals, GPU hardware behaviour, framework/library APIs, or protocol semantics (graphics, ML, audio, video, or otherwise) — if uncertain, say so explicitly in the text with a "Note: needs verification" callout, or look it up.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
