# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a 700-page expert-level technical book: **"The Linux Graphics Stack: From Kernel to Compositor and Browser"**. The full chapter plan lives in `plan.md`.

**Three audiences** (chapters signal which perspective is emphasised):
- **Systems and driver developers** — kernel internals, DRM/Mesa architecture, driver implementation
- **Graphics application developers** — Vulkan, EGL, VA-API, OpenXR usage and the stack beneath them
- **Browser and web platform engineers** — how Chromium/Chrome maps WebGPU, WebGL, and compositing onto the Linux graphics stack

## Repository Structure

Each chapter becomes its own `.md` file, e.g. `ch01-drm-architecture.md`. Parts may have subdirectories. The master outline is `plan.md` — do not alter it without user instruction.

## Writing Standards

**Accuracy first.** This is a technical reference. Code snippets must compile/run against current upstream versions. When citing kernel or Mesa source, pin the file path and commit/tag.

**References.** Every non-trivial claim and every code excerpt needs an inline reference with a source URL (kernel.org, gitlab.freedesktop.org, etc.). Use footnote-style or inline `[Source](url)` links.

**Chapter structure.** Each chapter `.md` must open with:
1. A brief scope paragraph naming which audience(s) it targets.
2. A local Table of Contents (heading anchors).
3. Sections following the outline in `plan.md`.
4. An **Integrations** section at the end (as described in `plan.md`) cross-linking to related chapters.

**Code blocks.** Label every block with the language (`c`, `rust`, `bash`, `glsl`, `spirv`, etc.). Include enough context (file path, function name, relevant struct/enum) that a reader can locate the source. Prefer real upstream snippets over invented examples; annotate any simplifications.

**Depth target.** Each chapter is approximately 15–20 pages (≈6,000–8,000 words). Part introductions are 1–2 pages.

## Content Research Approach

Use `WebSearch` and `WebFetch` to retrieve current upstream source, mailing-list discussions, and documentation. Key upstream repositories:

- Linux kernel DRM: `https://cgit.freedesktop.org/drm/drm-tip` / `https://github.com/torvalds/linux`
- Mesa: `https://gitlab.freedesktop.org/mesa/mesa`
- Wayland/wlroots: `https://gitlab.freedesktop.org/wayland/wayland` / `https://gitlab.freedesktop.org/wlroots/wlroots`
- libdrm: `https://gitlab.freedesktop.org/mesa/drm`
- Nouveau/NVK: `https://gitlab.freedesktop.org/nouveau/mesa` (NVK lives in main Mesa repo)
- Envytools: `https://github.com/envytools/envytools`
- Monado (OpenXR): `https://gitlab.freedesktop.org/monado/monado`
- Chromium source: `https://source.chromium.org/chromium`
- Dawn (WebGPU implementation): `https://dawn.googlesource.com/dawn`
- Tint (WGSL compiler): `https://dawn.googlesource.com/dawn/+/refs/heads/main/src/tint/`
- Skia (2D graphics): `https://skia.googlesource.com/skia`
- ANGLE (OpenGL ES → Vulkan/Metal/D3D): `https://chromium.googlesource.com/angle/angle`

Kernel doc: `https://www.kernel.org/doc/html/latest/gpu/`

## Workflow

1. One chapter per writing session. Research first, then write.
2. Use `WebSearch`/`WebFetch` to verify API signatures, struct definitions, and kernel/Mesa version where features landed.
3. After drafting, check cross-references against `plan.md` **Integrations** bullets to ensure nothing is missed.
4. Do not invent kernel interfaces, Mesa internals, or GPU hardware behaviour — if uncertain, say so explicitly in the text with a "Note: needs verification" callout, or look it up.
