# The Linux Graphics Stack: From Kernel to Compositor, Browser, and Terminal

An expert-level technical reference (~900 pages, 135 chapters) tracing every layer of the Linux graphics stack — from the DRM kernel subsystem and GPU memory management through Mesa's shader compilers, Wayland compositors, application APIs, the browser rendering pipeline, game compatibility layers, terminal pixel protocols, GPU-accelerated video streaming, AI inference, and both open and proprietary GPU ecosystems. Each chapter is written for practitioners: code snippets reference real upstream source at specific commits or releases, API signatures are verified against current kernel and Mesa trees, and architectural claims cite primary sources.

By the final chapter the reader holds a continuous mental model from a `DRM_IOCTL_MODE_ATOMIC` kernel call all the way to a photon leaving a display panel, and from a WebGPU `drawIndexed` call in JavaScript all the way to the same panel.

## Audiences

- **Systems and driver developers** — kernel internals, DRM/Mesa architecture, driver implementation
- **Graphics application developers** — the stack beneath Vulkan, EGL, VA-API, and OpenXR
- **Browser and web platform engineers** — how Chromium/Chrome maps WebGPU, WebGL, and compositing onto Linux hardware
- **Terminal and TUI developers** — GPU-rendered terminal emulators, Sixel, Kitty Graphics Protocol, Ghostty

## Parts and Chapters

| # | Part | Chapters | Count |
|---|------|----------|-------|
| I | The Kernel Layer | Ch 1–4, 51, 102, 120–121, 129 | 9 |
| II | GPU Drivers | Ch 5–6, 49, 73, 90, 92, 99–100, 116, 126 | 10 |
| III | The Open NVIDIA Stack | Ch 7–11, 10a–10b, 118 | 7 |
| IV | Mesa Architecture | Ch 12–17, 77, 91, 119 | 9 |
| V | Mesa GPU Drivers | Ch 18–19 | 2 |
| VI | The Display Stack | Ch 20–23, 46, 53–54, 74–75, 101, 105, 112, 123, 128, 130–132 | 17 |
| VII | Application APIs & Middleware | Ch 24–27, 38–39, 47–48, 50, 76, 96, 111, 114, 124, 127, 133 | 16 |
| VIII | Gaming Layer | Ch 28–29, 56, 78 | 4 |
| IX | Tooling & Contributing | Ch 30–32, 55, 79–80, 89, 93, 107, 109, 122, 125 | 12 |
| X | Browser Rendering Stack | Ch 33–37, 52, 98 | 7 |
| XI | Engines & Creative Tools | Ch 40–42, 97, 104 | 5 |
| XII | Terminal Graphics | Ch 43–45 | 3 |
| XIII | Video Streaming | Ch 57–60, 60b | 5 |
| XIV | Khronos Extended Ecosystem | Ch 61–65, 110 | 6 |
| XV | NVIDIA Proprietary Stack | Ch 66–70, 117 | 6 |
| XVI | Intel Open Graphics Stack | Ch 71 | 1 |
| XVII | AMD Developer Ecosystem | Ch 72 | 1 |
| XVIII | Rendering Abstraction Libraries | Ch 81–84, 113 | 5 |
| XIX | Android Graphics | Ch 85–87 | 3 |
| XX | AI/ML Inference on Linux | Ch 88, 94, 108, 115, 124 | 5 |
| XXI | Platform, Legacy & History | Ch 95, 103 | 2 |
| — | Appendices | A–M (13 appendices) | 13 |

**Total: 135 chapters + 13 appendices**

## Repository Structure

```
plan.md                          # Master outline with full per-chapter bullet points
intro.md                         # Holistic introduction and reading paths
chapters/
  intro.md                       # Book introduction (duplicate entry point)
  part-01-kernel-layer/
    part-intro.md                # Part overview (present in every part directory)
    ch01-drm-architecture.md
    ...
  part-02-gpu-drivers/
  ...
  part-21-platform-legacy/
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
