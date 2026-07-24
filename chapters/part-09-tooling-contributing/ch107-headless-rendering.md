# Chapter 107: Headless Rendering — Offscreen, CI, and Server Workloads

**Target audiences**: CI/CD engineers, cloud rendering developers, server-side GPU users, test infrastructure engineers.

This chapter explains how to render with GPUs — or without them — on Linux systems that have no physical display connected and no desktop compositor running. Headless rendering is the backbone of:

- **Screenshot CI pipelines**
- **Server-side PDF and image generation**
- **Cloud gaming back-ends**
- **GPU-accelerated video transcoding on bare-metal servers**
- **Automated regression tests** — compare rendered frames against golden images

Every section of the standard EGL/Wayland/X11 path assumes a connected display and a running compositor; headless mode systematically removes those assumptions and replaces them with kernel-level primitives, software rasterizers, and virtual output abstractions.

---

## Table of Contents

1. [Introduction — Defining Headless Rendering](#1-introduction--defining-headless-rendering)
   - [1.1 What is EGL?](#11-what-is-egl)
   - [1.2 What is GBM?](#12-what-is-gbm)
2. [Software Rendering: llvmpipe and lavapipe](#2-software-rendering-llvmpipe-and-lavapipe)
3. [EGL Surfaceless Platform](#3-egl-surfaceless-platform)
4. [GBM-Based Offscreen Rendering](#4-gbm-based-offscreen-rendering)
5. [Xvfb and Headless X11](#5-xvfb-and-headless-x11)
6. [Headless Wayland Compositors](#6-headless-wayland-compositors)
7. [Vulkan Headless](#7-vulkan-headless)
8. [GPU in Containers and Cloud](#8-gpu-in-containers-and-cloud)
9. [Server-Side Rendering Use Cases](#9-server-side-rendering-use-cases)
10. [Virtual Displays and Remote Desktop](#10-virtual-displays-and-remote-desktop)
11. [Integrations](#11-integrations)

---

## 1. Introduction — Defining Headless Rendering

*Headless rendering* is GPU or CPU-based rendering performed on a machine that lacks an active display output, a running Wayland or X11 compositor, or in some cases a GPU at all. The term covers a wide range of production scenarios:

- **CI screenshot testing** — a pull request pipeline renders a UI component or a shader, compares the output frame pixel-by-pixel against a golden reference, and fails the build if the difference exceeds a threshold.
- **Server-side rendering (SSR)** — a web or PDF server generates images, thumbnails, charts, or full-page screenshots using headless Chromium or a Node.js WebGL context.
- **Cloud gaming back-ends** — the game or application executes on a data-centre GPU, the rendered frame is hardware-encoded as H.264/HEVC/AV1, and the compressed stream is forwarded to a thin client over a low-latency network protocol.
- **ML training visualization** — a training loop periodically renders loss curves, attention maps, or 3D model previews without stopping to connect a display.
- **GPU-accelerated video transcoding** — FFmpeg with VAAPI or NVENC encodes video on a GPU-equipped server that has no monitor.
- **Automated UI testing** — Selenium, Playwright, and Puppeteer drive a full browser under headless Chrome or a headless Wayland compositor.

The fundamental challenge is that the standard rendering path on Linux goes through the EGL Window System Integration (WSI) layer, which in turn requires either an X11 connection or a Wayland compositor to obtain a presentable surface. Removing the display and compositor breaks this chain at multiple levels: there is no `DISPLAY` or `WAYLAND_DISPLAY` variable, no compositor to allocate shared buffers through, and in some environments no GPU at all. Each section of this chapter addresses one layer of the problem — from pure-software CPU rendering through GPU render-node access to full virtual compositor stacks.

### DRM render nodes: the foundation of headless GPU access

Every GPU-accelerated headless technique in this chapter relies on the DRM *render node* concept introduced in Linux 3.15. Render nodes (`/dev/dri/renderD128`, `renderD129`, ...) expose only the compute and rendering interfaces of the GPU driver — `DRM_IOCTL_*_SUBMIT`, `DRM_IOCTL_GEM_CLOSE`, and memory allocation — without any modesetting or display-engine control. A process that opens a render node gains GPU compute access without needing to be root and without any risk of disrupting the display server.

```bash
# Enumerate available render nodes
ls /dev/dri/renderD*
# Typical: /dev/dri/renderD128 (first GPU), /dev/dri/renderD129 (second)

# Identify the GPU behind a render node
udevadm info --query=property --name=/dev/dri/renderD128 | grep -E "ID_VENDOR|ID_MODEL|DRIVER"
# or via libdrm:
# drmGetDeviceNameFromFd2(fd) / drmGetDevice2(fd, 0, &dev)
```

[Source: DRM render node documentation — `Documentation/gpu/drm-uapi.rst` in the Linux kernel](https://www.kernel.org/doc/html/latest/gpu/drm-uapi.html)

When multiple GPUs are present — common in CI runners with both an integrated Intel GPU and a discrete AMD or NVIDIA GPU — the choice of render node matters for performance and feature availability. Tools such as `vulkaninfo` and `eglinfo` enumerate all available devices; applications can use `drmGetDevices2()` to iterate them programmatically.

Each section below builds on this render-node foundation. Sections 3–4 open the render node explicitly; software renderers (Section 2) and virtual compositors (Section 6) may open it internally or bypass it entirely when running in pure CPU mode.

### 1.1 What is EGL?

EGL is the Khronos Group's window system integration layer that sits between a platform-specific display system — X11, Wayland, a GBM device, or nothing at all — and a client rendering API such as OpenGL ES, OpenGL, or Vulkan. It provides three core services: display connection (`eglGetDisplay` / `eglGetPlatformDisplay`), rendering surface allocation (`eglCreateWindowSurface`, `eglCreatePbufferSurface`), and context management (`eglCreateContext`, `eglMakeCurrent`). On Linux, EGL is implemented by Mesa (`src/egl/`) and dispatches to hardware drivers through DRI3 on X11 or directly to a DRM render node via platform extensions such as `EGL_EXT_platform_device` and `EGL_MESA_platform_surfaceless`.

In the headless context this chapter covers, EGL's role shifts from bridging an application to a running compositor to bridging it directly to a render node. The `EGL_MESA_platform_surfaceless` extension (Section 3) allows an `EGLDisplay` to be opened against `/dev/dri/renderD*` with no `DISPLAY` or `WAYLAND_DISPLAY` environment variable set. Once the display is obtained and a context is made current, all rendering targets explicitly created framebuffer objects (FBOs) rather than a window surface, and pixels are retrieved via `glReadPixels` or exported as a DMA-BUF for further processing. EGL 1.5, released in 2014, absorbed several earlier extension mechanisms; the `EGL_EXT_platform_base` extension provides backward compatibility with EGL 1.4 implementations.

[Sources: [EGL 1.5 specification, Khronos Group](https://registry.khronos.org/EGL/); [Mesa EGL source `src/egl/`](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/egl)]

### 1.2 What is GBM?

GBM (Generic Buffer Management) is a Mesa library that allocates GPU-backed scanout buffers tied to a DRM device without involving a window system or compositor. It lives at `src/gbm/` in the Mesa tree and exposes a small C API built around three opaque types: `gbm_device` (wraps an open DRM file descriptor), `gbm_bo` (a single allocated buffer object), and `gbm_surface` (a swapchain of buffer objects that EGL can present). An application creates a `gbm_device` from a `/dev/dri/renderD*` file descriptor, requests a `gbm_bo` with a specific pixel format (for example `GBM_FORMAT_ARGB8888` defined in `drm_fourcc.h`) and usage flags (`GBM_BO_USE_RENDERING | GBM_BO_USE_LINEAR`), and then imports that buffer into EGL as an `EGLImage` using the `EGL_LINUX_DMA_BUF_EXT` extension for GPU rendering.

Because GBM buffer objects carry DMA-BUF file descriptors, they can be shared across process boundaries or handed directly to a V4L2 or VAAPI encode pipeline without a CPU copy. In the headless scenario, GBM fills the gap that EGL surfaceless deliberately leaves open: EGL surfaceless provides a context with no presentable surface, while GBM provides the concrete off-screen buffer to render into. The two are commonly combined — open a render node, create a `gbm_device`, allocate a `gbm_bo`, attach it to an FBO via an EGL image, render, then export the DMA-BUF for readback or encoding. Section 4 covers this pipeline in full detail.

[Sources: [Mesa GBM source `src/gbm/`](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gbm); [libdrm `drm_fourcc.h`](https://gitlab.freedesktop.org/mesa/drm/-/blob/main/include/drm/drm_fourcc.h)]

---

## 2. Software Rendering: llvmpipe and lavapipe

When the CI environment is a plain VM or container with no GPU passthrough, the practical choice is Mesa's family of CPU software rasterizers. These implement the full API on the CPU, making them slow but completely portable.

### llvmpipe

llvmpipe is Mesa's LLVM-based software rasterizer for OpenGL. It implements the full Gallium3D pipe driver interface ([Mesa source: `src/gallium/drivers/llvmpipe/`](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/drivers/llvmpipe)) and uses LLVM to JIT-compile fragment shader code to native machine instructions. Performance is roughly 1–5% of a discrete GPU but is sufficient for correctness testing of shaders, state-machine logic, and framebuffer readback.

Enable llvmpipe for any OpenGL application by setting:

```bash
export LIBGL_ALWAYS_SOFTWARE=1
```

[Source: Mesa environment-variable documentation](https://docs.mesa3d.org/envvars.html)

The `LIBGL_ALWAYS_SOFTWARE` variable instructs the Mesa loader (`src/loader/`) to ignore hardware DRI drivers and load the software rasterizer. An older, non-LLVM alternative called **softpipe** exists in the same directory tree (`src/gallium/drivers/softpipe/`); it is slower and maintained only for architectural reference — prefer llvmpipe on any modern system.

### lavapipe

lavapipe is Mesa's CPU Vulkan implementation ([`src/gallium/frontends/lavapipe/`](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/frontends/lavapipe)). It translates Vulkan API calls into Gallium3D pipe calls, which llvmpipe then executes. lavapipe passes the Vulkan Conformance Test Suite (CTS) and is the standard software Vulkan driver for headless CI. Force it by pointing the Vulkan loader at its ICD manifest:

```bash
# Current Vulkan loader variable (VK_ICD_FILENAMES is deprecated)
export VK_DRIVER_FILES=/usr/share/vulkan/icd.d/lvp_icd.x86_64.json

# On older toolchains VK_ICD_FILENAMES still works but is superseded
export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/lvp_icd.x86_64.json
```

[Source: Vulkan Loader driver interface documentation](https://github.com/KhronosGroup/Vulkan-Loader/blob/main/docs/LoaderDriverInterface.md)

The Vulkan loader documentation is explicit: `VK_DRIVER_FILES` takes precedence; `VK_ICD_FILENAMES` is considered deprecated but honoured for backward compatibility. The ICD JSON path varies slightly by distribution — on Fedora/RHEL the architecture suffix may be absent (`lvp_icd.json`).

### Zink

[zink](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/drivers/zink) is a Gallium3D front end that implements OpenGL on top of Vulkan. In a headless CI environment this means: set `VK_DRIVER_FILES` to lavapipe and then set `MESA_LOADER_DRIVER_OVERRIDE=zink` — the result is OpenGL running entirely on CPU via the lavapipe→Gallium pathway. This is valuable when a test harness exercises OpenGL but the CI runner only has lavapipe available:

```bash
export VK_DRIVER_FILES=/usr/share/vulkan/icd.d/lvp_icd.x86_64.json
export MESA_LOADER_DRIVER_OVERRIDE=zink
export LIBGL_ALWAYS_SOFTWARE=1  # belt-and-suspenders
```

### OSMesa: the classic offscreen context

Before EGL surfaceless became widespread, Mesa provided **OSMesa** (Off-Screen Mesa) as a dedicated off-screen OpenGL context that bypasses the window system entirely. OSMesa allocates a user-supplied CPU memory region as the color and depth buffer:

```c
#include <GL/osmesa.h>

OSMesaContext ctx = OSMesaCreateContextExt(
    OSMESA_RGBA, 24, 8, 0, NULL);

uint8_t framebuffer[width * height * 4];
OSMesaMakeCurrent(ctx, framebuffer, GL_UNSIGNED_BYTE, width, height);

// Normal OpenGL rendering calls; pixels accumulate in framebuffer[]
glClearColor(0.2f, 0.4f, 0.8f, 1.0f);
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
// ... render scene ...

// No eglSwapBuffers needed — result is already in framebuffer[]
OSMesaDestroyContext(ctx);
```

[Source: Mesa OSMesa documentation, `src/mesa/drivers/osmesa/osmesa.c`](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/mesa/drivers/osmesa/osmesa.c)

OSMesa is still used by VTK (Section 9) and ParaView when a pure-software, no-EGL path is required. It is CPU-only; the same OSMesa call routing through llvmpipe achieves software rasterization. For GPU-accelerated rendering, use EGL surfaceless or GBM (Sections 3–4) instead.

### Verifying which renderer you actually got

Silent fallback to software rendering is a common CI pitfall: a missing render-node permission causes EGL to silently fall back to llvmpipe, making performance tests meaningless. Always verify before running benchmarks or timing-sensitive tests:

```bash
# OpenGL — which driver is active?
LIBGL_ALWAYS_SOFTWARE=0 glxinfo -B | grep "OpenGL renderer"
# Expect: "llvmpipe (LLVM ...)" for software, "AMD Radeon RX..." for hardware

# EGL surfaceless — what EGL implementation?
EGL_LOG_LEVEL=debug eglinfo 2>&1 | grep -i "platform\|driver\|llvmpipe"

# Vulkan — which ICD loaded?
vulkaninfo 2>/dev/null | grep -i "driverName\|llvmpipe\|lavapipe"
# lavapipe shows: driverName = llvmpipe (lavapipe)

# Mesa debug output
MESA_DEBUG=1 LIBGL_ALWAYS_SOFTWARE=0 glxinfo 2>&1 | head -30

# Check render node access
ls -la /dev/dri/
# renderD128 should be group "render"; check your group membership:
groups | tr ' ' '\n' | grep -E "render|video"
```

[Source: Mesa environment variables reference](https://docs.mesa3d.org/envvars.html)

If `vulkaninfo` shows `lavapipe` when you expected hardware, the most common causes are: the process is not in the `render` group (`sudo usermod -aG render $USER && newgrp render`), the ICD JSON path is wrong (`VK_DRIVER_FILES` points to a non-existent file), or the GPU kernel module failed to load (`dmesg | grep -i amdgpu` or `grep i915`).

### When to use software renderers

Use llvmpipe/lavapipe when:

- The CI job runs in a Docker container or GitHub Actions runner without GPU passthrough.
- You want deterministic pixel-exact output unaffected by GPU microarchitecture differences.
- You need to test shader logic, blend modes, or state-machine correctness rather than performance.
- You are running Vulkan CTS or piglit without real hardware.

For production image-generation workloads or video encode, software renderers are far too slow; use GBM-based GPU rendering or the EGL surfaceless path (Sections 3–4) instead.

---

## 3. EGL Surfaceless Platform

The **`EGL_MESA_platform_surfaceless`** extension ([specification](https://www.khronos.org/registry/EGL/extensions/MESA/EGL_MESA_platform_surfaceless.txt)) defines a way to obtain an `EGLDisplay` without any window system, X server, or Wayland compositor. It is the cleanest path to GPU-accelerated headless rendering when you control the application and can render entirely to framebuffer objects (FBOs).

### Extension and enum

The platform token is:

```c
#define EGL_PLATFORM_SURFACELESS_MESA  0x31DD
```

[Source: EGL_MESA_platform_surfaceless specification](https://github.com/intel/external-mesa/blob/master/docs/specs/EGL_MESA_platform_surfaceless.txt)

### Obtaining a surfaceless EGLDisplay

```c
#include <EGL/egl.h>
#include <EGL/eglext.h>

// Requires EGL 1.4 + EGL_EXT_platform_base, or EGL 1.5
PFNEGLGETPLATFORMDISPLAYEXTPROC eglGetPlatformDisplayEXT =
    (PFNEGLGETPLATFORMDISPLAYEXTPROC)
    eglGetProcAddress("eglGetPlatformDisplayEXT");

EGLDisplay dpy = eglGetPlatformDisplayEXT(
    EGL_PLATFORM_SURFACELESS_MESA,
    EGL_DEFAULT_DISPLAY,   // <native_display> MUST be EGL_DEFAULT_DISPLAY
    NULL);                 // no additional attribs needed

eglInitialize(dpy, &major, &minor);
```

The `<native_display>` argument **must** be `EGL_DEFAULT_DISPLAY`; passing any other value is undefined. The extension was merged into Mesa in 2017 ([Mesa commit 27f4e38](https://cgit.freedesktop.org/mesa/mesa/commit/?id=27f4e381736f0abee35aa25035cb54b5c34f9bef)) and is available on any Mesa release from 17.0 onward.

### Creating a surfaceless context and rendering to an FBO

Without a surface there is no default framebuffer; all rendering must target an explicitly created FBO:

```c
// EGL context with no window system
EGLint attribs[] = {
    EGL_RENDERABLE_TYPE, EGL_OPENGL_ES3_BIT,
    EGL_NONE
};
EGLConfig config;
EGLint num_configs;
eglChooseConfig(dpy, attribs, &config, 1, &num_configs);

EGLint ctx_attribs[] = { EGL_CONTEXT_MAJOR_VERSION, 3,
                          EGL_CONTEXT_MINOR_VERSION, 1, EGL_NONE };
EGLContext ctx = eglCreateContext(dpy, config, EGL_NO_CONTEXT, ctx_attribs);

// Make current with NO surface — only valid on surfaceless platform
eglMakeCurrent(dpy, EGL_NO_SURFACE, EGL_NO_SURFACE, ctx);

// All rendering goes to an explicit FBO
GLuint fbo, color_tex, depth_rb;
glGenFramebuffers(1, &fbo);
glBindFramebuffer(GL_FRAMEBUFFER, fbo);
// ... attach textures, render, call glReadPixels ...
```

`eglMakeCurrent` with `EGL_NO_SURFACE` on both read and draw surfaces is only valid when the underlying platform is surfaceless; on the X11 or Wayland platform it returns `EGL_BAD_MATCH`.

### Requirements

- Access to a DRM render node (`/dev/dri/renderD128`) for GPU acceleration. No primary node (`/dev/dri/card0`) is needed.
- The process must be in the `render` group (or equivalent udev permission) or run as root.
- Without any render node the driver falls back to llvmpipe automatically.

### GBM as an alternative EGL platform

An alternative approach uses GBM to provide a minimal display abstraction:

```c
#include <gbm.h>

int drm_fd = open("/dev/dri/renderD128", O_RDWR);
struct gbm_device *gbm = gbm_create_device(drm_fd);

EGLDisplay dpy = eglGetPlatformDisplayEXT(
    EGL_PLATFORM_GBM_KHR,  // requires EGL_KHR_platform_gbm
    gbm,
    NULL);
```

[Source: Mesa EGL GBM platform source `src/egl/drivers/dri2/platform_drm.c`](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/egl/drivers/dri2/platform_drm.c)

The GBM platform is preferred when you also need to export the rendered buffer as a DMA-BUF (see Section 4).

---

## 4. GBM-Based Offscreen Rendering

GBM (Generic Buffer Manager, `libgbm`) provides a device- and compositor-agnostic way to allocate GPU-tiled buffers and export them as DMA-BUF file descriptors. It is the standard mechanism for GPU-accelerated headless rendering when you need to pass the result to another subsystem — a video encoder, an image writer, or a network streaming pipeline.

[Source: libgbm/GBM API documentation, `include/gbm.h` in libdrm](https://gitlab.freedesktop.org/mesa/drm/-/blob/main/include/gbm.h)

### Opening the render node

```c
#include <fcntl.h>
#include <gbm.h>
#include <EGL/egl.h>
#include <EGL/eglext.h>
#include <GL/gl.h>

int drm_fd = open("/dev/dri/renderD128", O_RDWR | O_CLOEXEC);
if (drm_fd < 0) { perror("open render node"); return 1; }

struct gbm_device *gbm = gbm_create_device(drm_fd);
```

No primary node (`card0`) is required; render nodes carry only rendering privilege, not modesetting or display-engine control. This makes them safe to pass into containers and sandboxed processes.

### Two modes: GBM surface vs GBM buffer object

**Mode A — GBM surface (swapchain-style)**: Allocates a ring of scanout-capable buffers behind a `gbm_surface`. This is the path Wayland compositors use; for pure headless work it is overkill.

```c
struct gbm_surface *gbm_surf = gbm_surface_create(
    gbm, width, height,
    GBM_FORMAT_ARGB8888,
    GBM_BO_USE_RENDERING);

EGLSurface egl_surf = eglCreateWindowSurface(
    dpy, config,
    (EGLNativeWindowType)gbm_surf, NULL);
```

**Mode B — GBM buffer object (fully offscreen)**: Allocates a single buffer object, exports it as a DMA-BUF fd, then imports it into EGL using `EGL_LINUX_DMA_BUF_EXT`. This is the correct, portable headless path — using `EGL_NATIVE_PIXMAP_KHR` with a `gbm_bo` is not portable and is not recommended.

```c
struct gbm_bo *bo = gbm_bo_create(
    gbm, width, height,
    GBM_FORMAT_XRGB8888,
    GBM_BO_USE_RENDERING | GBM_BO_USE_LINEAR);

// Step 1: export the gbm_bo as a DMA-BUF file descriptor
int dmabuf_fd = gbm_bo_get_fd(bo);
EGLint stride  = (EGLint)gbm_bo_get_stride(bo);

// Step 2: import the DMA-BUF into EGL using EGL_LINUX_DMA_BUF_EXT
// Requires EGL_EXT_image_dma_buf_import extension
EGLint img_attribs[] = {
    EGL_WIDTH,                    (EGLint)width,
    EGL_HEIGHT,                   (EGLint)height,
    EGL_LINUX_DRM_FOURCC_EXT,    DRM_FORMAT_XRGB8888,
    EGL_DMA_BUF_PLANE0_FD_EXT,  dmabuf_fd,
    EGL_DMA_BUF_PLANE0_OFFSET_EXT, 0,
    EGL_DMA_BUF_PLANE0_PITCH_EXT,  stride,
    EGL_NONE
};
EGLImageKHR egl_img = eglCreateImageKHR(
    dpy, EGL_NO_CONTEXT,
    EGL_LINUX_DMA_BUF_EXT,
    (EGLClientBuffer)NULL,   // no client buffer; fd provided in attribs
    img_attribs);

// Step 3: attach to FBO as a renderbuffer
GLuint rb;
glGenRenderbuffers(1, &rb);
glBindRenderbuffer(GL_RENDERBUFFER, rb);
glEGLImageTargetRenderbufferStorageOES(GL_RENDERBUFFER, egl_img);
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0,
                          GL_RENDERBUFFER, rb);
```

[Source: `EGL_EXT_image_dma_buf_import` specification](https://registry.khronos.org/EGL/extensions/EXT/EGL_EXT_image_dma_buf_import.txt); [Mesa `src/egl/drivers/dri2/platform_drm.c`](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/egl/drivers/dri2/platform_drm.c)

### Exporting the render result as DMA-BUF

After rendering, export the buffer as a DMA-BUF file descriptor for zero-copy handoff to FFmpeg, V4L2, or a network encoder:

```c
// Export DMA-BUF fd
int dmabuf_fd = gbm_bo_get_fd(bo);
uint32_t stride   = gbm_bo_get_stride(bo);
uint32_t width_px = gbm_bo_get_width(bo);
uint32_t height_px= gbm_bo_get_height(bo);
uint32_t format   = gbm_bo_get_format(bo);  // DRM_FORMAT_XRGB8888

// Now pass dmabuf_fd to FFmpeg's av_buffer_pool, V4L2 OUTPUT queue,
// or map to CPU with drmPrimeHandleToFD + mmap for pixel readback
```

[Source: `gbm_bo_get_fd` in `libgbm` — Mesa drm `lib/gbm.c`](https://gitlab.freedesktop.org/mesa/drm/-/blob/main/lib/gbm.c)

For CPU readback without DMA-BUF (e.g. writing a PNG):

```c
// After rendering
glReadPixels(0, 0, width, height, GL_RGBA, GL_UNSIGNED_BYTE, pixels);
// ... stb_image_write_png("out.png", width, height, 4, pixels, width*4) ...
```

### Complete render-to-PNG skeleton

```c
// Pseudocode sketch; error handling elided
int fd = open("/dev/dri/renderD128", O_RDWR | O_CLOEXEC);
struct gbm_device *gbm = gbm_create_device(fd);
EGLDisplay dpy = eglGetPlatformDisplayEXT(EGL_PLATFORM_GBM_KHR, gbm, NULL);
eglInitialize(dpy, NULL, NULL);
// ... choose config, create context, make current with EGL_NO_SURFACE ...
// ... create FBO backed by gbm_bo, render scene ...
uint8_t pixels[width * height * 4];
glReadPixels(0, 0, width, height, GL_RGBA, GL_UNSIGNED_BYTE, pixels);
stbi_write_png("render.png", width, height, 4, pixels, width * 4);
```

This pattern is the foundation of headless test renderers in projects such as [wlroots' headless test suite](https://gitlab.freedesktop.org/wlroots/wlroots/-/tree/master/tests) and the [Mesa CI frame-comparison infrastructure](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/.gitlab-ci).

---

## 5. Xvfb and Headless X11

For legacy X11 applications — Selenium-driven browsers, GTK2 apps, Qt software that predates Wayland — the standard headless technique is **Xvfb** (X Virtual Framebuffer), a software X11 server that renders into an in-memory framebuffer with no physical display.

### Starting Xvfb

```bash
Xvfb :99 -screen 0 1920x1080x24 &
export DISPLAY=:99

# Run any X11 app
firefox --no-remote https://example.com &
```

[Source: X.Org Xvfb man page](https://www.x.org/releases/current/doc/man/man1/Xvfb.1.xhtml)

Xvfb creates a shared-memory segment (SHM) holding the raw framebuffer pixels. It speaks the full X11 wire protocol, so existing tooling — `xdpyinfo`, `xwd`, `xrandr` — works unchanged.

The `xvfb-run` wrapper handles Xvfb lifecycle and display-number allocation automatically, which is preferable in CI where display numbers may conflict:

```bash
# Automatic display allocation, passes through exit code of the wrapped command
xvfb-run --auto-servernum --server-args="-screen 0 1920x1080x24" \
    pytest tests/ui/

# Fixed display number, disabling access control (for multi-process tests)
xvfb-run -d --server-args="-screen 0 1920x1080x24 -ac" \
    npm run test:e2e
```

[Source: `xvfb-run` man page, part of the `xvfb` package on Debian/Ubuntu]

### Capturing screenshots

```bash
# Capture screen to XWD format, convert to PNG
xwd -root -silent -display :99 | xwdtopnm | pnmtopng > screenshot.png
# Or use ImageMagick:
DISPLAY=:99 import -window root screenshot.png
```

### Xvfb + VirtualGL for hardware-accelerated 3D

Xvfb alone does no 3D acceleration; GLX calls return software-rendered results through Mesa's llvmpipe. **VirtualGL** (VGL) intercepts OpenGL/GLX calls at the `libGL.so` wrapper level, redirects them to a real GPU via a separate X11 server or DRI connection, reads back the rendered pixels, and composes them onto the Xvfb virtual framebuffer:

```bash
# Start VirtualGL-enabled Xvfb
vglserver_config   # one-time setup
Xvfb :99 -screen 0 1920x1080x24 &
export DISPLAY=:99

# Run 3D app through VirtualGL
vglrun glxgears
vglrun -d /dev/dri/renderD128 blender --background scene.blend --render-output /tmp/frame
```

[Source: VirtualGL project documentation](https://virtualgl.org/Documentation/Documentation)

The VGL transport adds a readback round-trip (GPU → CPU → shared framebuffer) which limits throughput; for high-frame-rate workloads GBM-based rendering (Section 4) or the Vulkan headless path (Section 7) is more efficient.

### Xephyr: nested X

For debugging only (not truly headless), **Xephyr** runs as a nested X server inside an existing X11 window. It is useful for isolating a test desktop from the user's session but not for bare-metal server environments.

```bash
Xephyr :10 -screen 1280x800 &
DISPLAY=:10 openbox &
```

---

## 6. Headless Wayland Compositors

Modern display stacks prefer Wayland. Several strategies allow running Wayland-native applications in headless environments without a physical compositor.

### Weston with the headless backend

Weston, the Wayland reference compositor, has a headless backend that creates a virtual output backed by a software-allocated buffer, requiring no KMS/DRM display output:

```bash
# Weston ≥12: short form (no .so suffix required)
weston --backend=headless --width=1920 --height=1080 &
export WAYLAND_DISPLAY=wayland-1

# Verify the virtual output
weston-info
```

[Source: Weston headless backend — `libweston/backend-headless/`](https://gitlab.freedesktop.org/wayland/weston/-/tree/main/libweston/backend-headless)

Note: Weston 12+ accepts `--backend=headless` (without `.so`); older releases required `--backend=headless-backend.so`. Either form is accepted in current releases for compatibility.

### sway with the wlroots headless backend

[sway](https://github.com/swaywm/sway) is a tiling Wayland compositor built on [wlroots](https://gitlab.freedesktop.org/wlroots/wlroots). The wlroots headless backend allocates virtual `wl_output` objects and enables GPU rendering through a render node without any scanout display:

```bash
WLR_BACKENDS=headless WLR_RENDERER=pixman WLR_HEADLESS_OUTPUTS=1 sway &
export WAYLAND_DISPLAY=wayland-1
```

[Source: wlroots environment variable documentation](https://github.com/swaywm/wlroots/blob/master/docs/env_vars.md)

The key environment variables from wlroots:

| Variable | Description |
|---|---|
| `WLR_BACKENDS` | Comma-separated backend list: `libinput`, `drm`, `wayland`, `x11`, `headless`, `noop` |
| `WLR_RENDERER` | Force renderer: `gles2` or `pixman` |
| `WLR_HEADLESS_OUTPUTS` | Number of virtual outputs when using `headless` backend |

`pixman` is a pure-CPU renderer useful in containers without GPU; `gles2` uses Mesa EGL/GBM on a render node for GPU acceleration.

### wayvnc: VNC server for headless Wayland

[wayvnc](https://github.com/any1/wayvnc) attaches to a wlroots-based compositor (including a headless one) via the `wlr-screencopy-v1` protocol and exposes a VNC server. Remote users can connect and interact with the headless compositor over the network:

```bash
WLR_BACKENDS=headless WLR_HEADLESS_OUTPUTS=1 sway &
wayvnc 0.0.0.0 5900 &
# Connect: vncviewer <server>:5900
```

### xdg-desktop-portal-wlr

The [xdg-desktop-portal-wlr](https://github.com/emersion/xdg-desktop-portal-wlr) portal backend enables screenshot and screencast capture from a wlroots compositor (including headless) through the standard XDG portal D-Bus interface, making it compatible with Pipewire-based screen recording tools:

```bash
/usr/libexec/xdg-desktop-portal-wlr &
# Screencast stream available via PipeWire
```

---

## 7. Vulkan Headless

Vulkan's explicit design separates the swapchain from the core device API, making truly headless rendering straightforward: simply do not create a `VkSurfaceKHR` or swapchain at all.

### Instance and device without a surface

```c
// No VK_KHR_surface, no VK_KHR_xcb_surface, no display-related extensions
VkInstanceCreateInfo inst_info = {
    .sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO,
    .enabledExtensionCount = 0,   // zero WSI extensions
    .ppEnabledExtensionNames = NULL,
};
VkInstance instance;
vkCreateInstance(&inst_info, NULL, &instance);

// Pick a physical device
VkPhysicalDevice phys_dev;
uint32_t count = 1;
vkEnumeratePhysicalDevices(instance, &count, &phys_dev);

// Create logical device — no surface-present queue needed
float prio = 1.0f;
VkDeviceQueueCreateInfo queue_info = {
    .sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO,
    .queueFamilyIndex = 0,
    .queueCount = 1,
    .pQueuePriorities = &prio,
};
VkDeviceCreateInfo dev_info = {
    .sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO,
    .queueCreateInfoCount = 1,
    .pQueueCreateInfos = &queue_info,
    .enabledExtensionCount = 0,
};
VkDevice device;
vkCreateDevice(phys_dev, &dev_info, NULL, &device);
```

[Source: Vulkan specification, `VkDeviceCreateInfo`](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#VkDeviceCreateInfo)

### Rendering to VkImage

Create an explicit `VkImage` as the color target, render to it, then read back pixels or export as DMA-BUF. Two tiling strategies exist with different trade-offs.

**Strategy A — LINEAR tiling (direct CPU mapping)**: Simple, but GPUs perform poorly on linearly-tiled images and not all hardware supports it as a color attachment.

```c
VkImageCreateInfo img_info = {
    .sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO,
    .imageType = VK_IMAGE_TYPE_2D,
    .format = VK_FORMAT_R8G8B8A8_UNORM,
    .extent = { width, height, 1 },
    .mipLevels = 1, .arrayLayers = 1,
    .samples = VK_SAMPLE_COUNT_1_BIT,
    .tiling = VK_IMAGE_TILING_LINEAR,   // CPU-mappable, GPU may be slow
    .usage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT
           | VK_IMAGE_USAGE_TRANSFER_SRC_BIT,
    .initialLayout = VK_IMAGE_LAYOUT_UNDEFINED,
};
VkImage render_img;
vkCreateImage(device, &img_info, NULL, &render_img);
// Bind HOST_VISIBLE | HOST_COHERENT memory, render, then vkMapMemory to read
```

**Strategy B — OPTIMAL tiling with staging buffer (recommended)**: The GPU renders to a GPU-optimal image, then `vkCmdCopyImageToBuffer` copies the pixels into a `HOST_VISIBLE` staging buffer for CPU access. This is the pattern used by virtually all production headless renderers.

```c
// 1. Create GPU-optimal render image
VkImageCreateInfo render_img_info = {
    .sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO,
    .format = VK_FORMAT_R8G8B8A8_UNORM,
    .extent = { width, height, 1 },
    .tiling = VK_IMAGE_TILING_OPTIMAL,  // GPU-optimal
    .usage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT
           | VK_IMAGE_USAGE_TRANSFER_SRC_BIT,
    // ... other fields ...
};
VkImage render_img;
vkCreateImage(device, &render_img_info, NULL, &render_img);
// Bind DEVICE_LOCAL memory

// 2. Create HOST_VISIBLE staging buffer for readback
VkBufferCreateInfo staging_info = {
    .sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO,
    .size = width * height * 4,  // RGBA bytes
    .usage = VK_BUFFER_USAGE_TRANSFER_DST_BIT,
};
VkBuffer staging_buf;
vkCreateBuffer(device, &staging_info, NULL, &staging_buf);
// Bind HOST_VISIBLE | HOST_COHERENT memory

// 3. After rendering, record copy in command buffer:
//    Transition render_img from COLOR_ATTACHMENT_OPTIMAL to TRANSFER_SRC_OPTIMAL
VkImageMemoryBarrier barrier = {
    .sType = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER,
    .oldLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL,
    .newLayout = VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL,
    // ... srcAccessMask, dstAccessMask, image, subresourceRange ...
};
vkCmdPipelineBarrier(cmd, VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT,
                     VK_PIPELINE_STAGE_TRANSFER_BIT, 0,
                     0, NULL, 0, NULL, 1, &barrier);

// Copy image pixels into staging buffer
VkBufferImageCopy copy_region = {
    .bufferOffset = 0,
    .bufferRowLength = 0,    // tightly packed
    .bufferImageHeight = 0,
    .imageSubresource = { VK_IMAGE_ASPECT_COLOR_BIT, 0, 0, 1 },
    .imageOffset = { 0, 0, 0 },
    .imageExtent  = { width, height, 1 },
};
vkCmdCopyImageToBuffer(cmd, render_img,
    VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL,
    staging_buf, 1, &copy_region);

// 4. Submit, wait, then map staging buffer
vkQueueSubmit(queue, 1, &submit_info, fence);
vkWaitForFences(device, 1, &fence, VK_TRUE, UINT64_MAX);

void *data;
vkMapMemory(device, staging_mem, 0, width * height * 4, 0, &data);
// data now contains the RGBA pixels
memcpy(pixels, data, width * height * 4);
vkUnmapMemory(device, staging_mem);
```

[Source: Vulkan spec `VkImageCreateInfo`, tiling and host access](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#resources-images); [`vkCmdCopyImageToBuffer`](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#copies-buffers-images)

### Exporting render results as DMA-BUF

For zero-copy handoff to a video encoder or display system, use `VK_KHR_external_memory_fd` with `VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT`:

```c
// During allocation, chain VkExportMemoryAllocateInfo
VkExportMemoryAllocateInfo export_info = {
    .sType = VK_STRUCTURE_TYPE_EXPORT_MEMORY_ALLOCATE_INFO,
    .handleTypes = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
};
// After vkAllocateMemory:
VkMemoryGetFdInfoKHR get_fd_info = {
    .sType = VK_STRUCTURE_TYPE_MEMORY_GET_FD_INFO_KHR,
    .memory = render_memory,
    .handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
};
int dmabuf_fd;
vkGetMemoryFdKHR(device, &get_fd_info, &dmabuf_fd);
```

[Source: `VK_KHR_external_memory_fd` specification](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_external_memory_fd.html)

### VK_EXT_headless_surface

Some applications unconditionally require a `VkSurfaceKHR` even when they never actually present to it. The `VK_EXT_headless_surface` extension creates a stub surface object that satisfies those requirements without requiring any real display:

```c
VkHeadlessSurfaceCreateInfoEXT surf_info = {
    .sType = VK_STRUCTURE_TYPE_HEADLESS_SURFACE_CREATE_INFO_EXT,
};
VkSurfaceKHR surface;
vkCreateHeadlessSurfaceEXT(instance, &surf_info, NULL, &surface);
// Swapchain can now be created; present calls are no-ops
```

[Source: `VK_EXT_headless_surface` specification](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_headless_surface.html)

### NVIDIA on headless servers

NVIDIA's proprietary driver offers `nvidia-headless` and `nvidia-headless-no-dkms` packages for servers without display hardware. These install only the CUDA, NVENC, and render-node components, omitting the X11 GLX libraries:

```bash
# Ubuntu example
sudo apt install nvidia-headless-550-server nvidia-utils-550-server
# Creates /dev/nvidia0, /dev/nvidiactl, /dev/nvidia-uvm
# EGL render via /dev/nvidia0 (NVIDIA's EGL implementation)
```

[Source: NVIDIA Linux driver repository, headless metapackages](https://launchpad.net/ubuntu/+source/nvidia-graphics-drivers-550-server)

NVIDIA's EGL implementation uses `/dev/nvidia0` rather than a DRM render node on proprietary drivers; on the open-source `nvidia-open` kernel module, standard render-node access through Mesa NVK applies.

---

## 8. GPU in Containers and Cloud

Running GPU workloads inside containers requires explicit device node exposure and, for orchestrated clusters, a device plugin that negotiates GPU assignment.

### Docker and Podman

**DRM render-node access (Mesa/AMD/Intel/open NVIDIA)**:

```bash
# Render node only (unprivileged, safe for untrusted workloads)
docker run --device=/dev/dri/renderD128 \
           --group-add video \
           myimage mesa_headless_renderer

# Primary node (modesetting — requires more privilege)
docker run --device=/dev/dri/card0 \
           --device=/dev/dri/renderD128 \
           myimage kms_renderer
```

[Source: Docker device passthrough documentation](https://docs.docker.com/engine/containers/resource_constraints/)

The `render` group (typically GID 109 or 110 on Debian/Ubuntu) controls access to `/dev/dri/renderD*`. Render nodes carry only compute/rendering privilege; they cannot flip a display or access framebuffer memory for the primary display, making them safe for container use. Primary nodes (`card*`) require the `video` group and more privilege.

**Rootless Podman with GPU access**: Rootless Podman can access render nodes without any privilege escalation, provided the host's udev rules allow the user's UID to read `renderD*`:

```bash
# Rootless Podman — DRM render node (no sudo needed if user is in render group)
podman run --device=/dev/dri/renderD128:rwm \
           --security-opt=no-new-privileges \
           myimage mesa_headless_renderer

# Add DRM render node access via cgroup device rules in containers.conf
# /etc/containers/containers.conf:
# [containers]
# devices = ["/dev/dri/renderD128:rwm"]
```

[Source: Podman documentation on device access](https://docs.podman.io/en/latest/markdown/podman-run.1.html#device-path)

**cgroup device rules**: Container runtimes use cgroup v2 device controller entries to grant fine-grained device access. For Kubernetes, the NVIDIA GPU operator and Intel device plugins manage these cgroup entries automatically; for custom container runtimes, add entries to the OCI runtime spec's `linux.devices` and `linux.resources.devices` arrays.

**NVIDIA Container Toolkit**:

```bash
# Install once on host
sudo apt install nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Run with GPU access
docker run --gpus all nvidia/cuda:12.6.0-base-ubuntu24.04 nvidia-smi

# Specific GPU
docker run --gpus '"device=0"' myimage render_scene
```

[Source: NVIDIA Container Toolkit installation guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)

**ROCm (AMD) in containers**:

```bash
docker run \
    --device=/dev/kfd \
    --device=/dev/dri \
    --group-add video \
    --group-add render \
    rocm/rocm-terminal:latest \
    rocminfo
```

[Source: AMD ROCm Docker documentation](https://rocm.docs.amd.com/en/latest/deploy/docker.html)

`/dev/kfd` is the KFD (Kernel Fusion Driver) device used by HIP/ROCm compute; `/dev/dri` provides the DRM render nodes for display-class GPU work.

### Kubernetes GPU scheduling

In Kubernetes, GPUs are scheduled via the [NVIDIA GPU operator](https://github.com/NVIDIA/gpu-operator) or [AMD GPU device plugin](https://github.com/ROCm/k8s-device-plugin):

```yaml
# Pod spec requesting one GPU
resources:
  requests:
    nvidia.com/gpu: 1
  limits:
    nvidia.com/gpu: 1
```

The GPU operator automates driver installation, container toolkit configuration, and device plugin deployment. For Intel and AMD GPUs, the community [intel-device-plugins-for-kubernetes](https://github.com/intel/intel-device-plugins-for-kubernetes) and ROCm device plugin expose `gpu.intel.com/i915` and `amd.com/gpu` resource types.

### CI with headless GPU rendering

GitHub Actions provides GPU runners (on self-hosted runners with GPU passthrough), but many CI pipelines run on CPU-only machines and rely on software rendering or headless Chrome with `--use-angle=swiftshader`:

```bash
# GitHub Actions: headless Chrome + software rendering
- name: Run Playwright tests
  run: |
    export DISPLAY=:99
    Xvfb :99 -screen 0 1920x1080x24 &
    npx playwright test

# Or via xvfb-run wrapper
xvfb-run --auto-servernum --server-args="-screen 0 1920x1080x24" \
    npx playwright test
```

[Source: Playwright CI documentation](https://playwright.dev/docs/ci)

Mesa's own CI pipeline ([`.gitlab-ci.yml` in Mesa](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/.gitlab-ci.yml)) uses DRM render nodes on GitLab runners with GPU passthrough for hardware driver tests, and lavapipe for conformance tests that need no hardware.

---

## 9. Server-Side Rendering Use Cases

### Thumbnail and image generation

Server-side image generation traditionally used ImageMagick (CPU) or PIL/Pillow. GPU acceleration is available via:

- **OpenCL through Mesa's rusticl**: `RUSTICL_ENABLE=radeonsi` enables OpenCL on RADV-supported AMD hardware.
- **Vulkan compute via custom shaders**: image resize, color-space conversion, and compositing at GPU speed without a display.
- **FFmpeg hardware video transcoding** via VAAPI (Intel, AMD) or NVENC (NVIDIA):

```bash
# GPU-accelerated H.264 encode on Intel (VAAPI)
ffmpeg -vaapi_device /dev/dri/renderD128 \
       -i input.mp4 \
       -vf 'format=nv12,hwupload' \
       -c:v h264_vaapi \
       -qp 24 output.mp4

# NVIDIA NVENC
ffmpeg -i input.mp4 -c:v h264_nvenc -preset p7 output.mp4
```

[Source: FFmpeg VAAPI hardware acceleration documentation](https://trac.ffmpeg.org/wiki/Hardware/VAAPI)

### Server-side WebGL via headless-gl

The [`headless-gl`](https://www.npmjs.com/package/headless-gl) npm package provides a WebGL 1.0 context for Node.js using EGL surfaceless + Mesa. It is used by server-side chart libraries (e.g., deck.gl server-side rendering) and ML model visualization tools:

```bash
npm install headless-gl
```

```javascript
// Node.js — renders to an offscreen GL context
const gl = require('headless-gl')(1920, 1080);
const ext = gl.getExtension('STACKGL_resize_drawingbuffer');
ext.resize(1920, 1080);
// Standard WebGL calls work
gl.clearColor(0.0, 0.5, 1.0, 1.0);
gl.clear(gl.COLOR_BUFFER_BIT);
const pixels = new Uint8Array(1920 * 1080 * 4);
gl.readPixels(0, 0, 1920, 1080, gl.RGBA, gl.UNSIGNED_BYTE, pixels);
```

[Source: headless-gl GitHub README](https://github.com/stackgl/headless-gl)

### Chromium headless for PDF and screenshot generation

Chromium's headless mode is the workhorse of server-side screenshot and PDF pipelines. The correct invocations as of Chrome 130+:

```bash
# GPU-accelerated headless (EGL on Mesa, recommended)
google-chrome --headless \
              --use-gl=angle \
              --use-angle=vulkan \
              --screenshot=screenshot.png \
              https://example.com

# Software fallback (CPU, no GPU needed — requires opt-in since Chrome 128)
google-chrome --headless \
              --use-gl=angle \
              --use-angle=swiftshader \
              --enable-unsafe-swiftshader \
              --screenshot=screenshot.png \
              https://example.com
```

[Source: Chromium SwiftShader documentation](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/docs/gpu/swiftshader.md)

**Important**: Automatic SwiftShader WebGL fallback was deprecated in Chrome 128. The `--enable-unsafe-swiftshader` flag is required to re-enable it; without it, WebGL context creation fails on CPU-only machines. For Playwright and Puppeteer, pass these flags via the `launchOptions.args` array.

The standalone `chrome-headless-shell` binary (introduced in Chrome 132) is optimized for server-side use and is the recommended replacement for the old `--headless=old` mode.

### Scientific visualization

**VTK offscreen rendering**: VTK supports a pure-offscreen rendering backend via OSMesa (Mesa's off-screen rendering context) or via the standard EGL backend. The available backends depend on how VTK was compiled; distributions typically ship both:

```bash
# EGL offscreen backend (GPU-accelerated via Mesa EGL)
export VTK_DEFAULT_RENDER_BACKEND=egl
pvpython render_script.py

# OSMesa (pure software, no EGL needed, must be built with -DVTK_OPENGL_HAS_OSMESA=ON)
export VTK_DEFAULT_RENDER_BACKEND=osmesa
pvpython render_script.py
```

> **Note**: The exact CMake options and runtime variable names vary between VTK 9.x releases. If `VTK_DEFAULT_RENDER_BACKEND=egl` has no effect, verify VTK was built with `-DVTK_OPENGL_HAS_EGL=ON`. The `paraview --backend=egl` flag is ParaView-specific and may not be present in all distributions.

[Source: VTK offscreen rendering CMake options](https://vtk.org/doc/nightly/html/md__builds_gitlab-kitware-sciviz-ci_Documentation_Doxygen_DocOffScreenRendering.html)

**ParaView server mode**: ParaView runs `pvserver` as a headless rendering daemon; clients connect via the ParaView remote visualization protocol:

```bash
pvserver --server-port=11111 &
# Client connects to server; rendering happens headlessly on server
```

**OpenFOAM** post-processing without a display:

```bash
paraFoam -builtin -touch    # generate .foam file
pvbatch postProcess.py      # batch ParaView with no display
```

### Golden-image CI regression testing

The canonical headless CI regression pattern compares pixel output against committed reference images:

```bash
# Run renderer headlessly, save PNG
./render_test --headless --output actual.png

# Compare against golden image with acceptable tolerance
compare -metric PSNR actual.png golden.png diff.png
# PSNR > 40 dB = no visible regression; < 30 dB = fail

# Or use perceptual comparison
compare -metric SSIM actual.png golden.png null:
```

Mesa's CI uses a Python-based frame comparison tool in [`src/gallium/tests/ci/`](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/tests) that handles per-driver tolerance levels and flags unexpected pixel differences as regressions.

---

## 10. Virtual Displays and Remote Desktop

Headless rendering is closely related to remote desktop and GPU virtualization: the rendering happens on a server, the output is captured and compressed, and the compressed stream is sent to a client.

### virtio-gpu and rutabaga (Venus)

In KVM/QEMU virtual machines, the `virtio-gpu` virtual device gives guests access to GPU resources via the `rutabaga_gfx` framework and the Venus Vulkan protocol (see Ch89 for full coverage). The guest OS renders via Mesa NVK/RADV as if it had hardware; `rutabaga` forwards the command stream to the host GPU:

```bash
# QEMU with virtio-gpu-gl (rutabaga/Venus)
qemu-system-x86_64 \
    -device virtio-gpu-gl,hostmem=256M \
    -display egl-headless \
    ...
```

[Source: QEMU virtio-gpu documentation](https://www.qemu.org/docs/master/system/devices/virtio-gpu.html)

The `-display egl-headless` flag runs QEMU's display subsystem headlessly using EGL on the host's GPU.

### looking-glass

[looking-glass](https://looking-glass.io/) achieves near-native GPU performance for Windows VMs by sharing the rendered framebuffer in host RAM between the VM and the Linux host. The guest GPU renders frames into a shared memory region; the host reads them using a custom KVM shared-memory mechanism and displays them in a native Linux window or streams them:

```bash
# Host: KVMFR kernel module + looking-glass-client
modprobe kvmfr
looking-glass-client -f /dev/kvmfr0
```

### xrdp

[xrdp](https://github.com/neutrinolabs/xrdp) provides an RDP server for Linux. It drives an Xvfb backend or, with the `xorgxrdp` module, can use hardware-accelerated X11:

```bash
sudo systemctl enable --now xrdp
# Connect with any RDP client: mstsc.exe, Remmina, FreeRDP
xfreerdp /v:server /u:user /p:pass /w:1920 /h:1080
```

### GNOME Remote Desktop

GNOME Remote Desktop (`gnome-remote-desktop`) implements both RDP and VNC using the [xdg-remote-desktop-portal](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.RemoteDesktop.html) D-Bus API and PipeWire for screen capture. It supports GPU-accelerated compositing under GNOME Shell with Mutter as the compositor:

```bash
# Enable RDP in GNOME settings
grdctl rdp enable
grdctl rdp set-credentials myuser mypassword
```

[Source: GNOME Remote Desktop project](https://gitlab.gnome.org/GNOME/gnome-remote-desktop)

### Sunshine: open-source cloud gaming server

[Sunshine](https://github.com/LizardByte/Sunshine) is an open-source implementation of the NVIDIA GameStream protocol. It captures the desktop (via KMS DMA-BUF or X11 `XShmGetImage`) and hardware-encodes with NVENC, AMD AMF, Intel VAAPI, or VA-API software fallback, then streams over RTSP to Moonlight clients:

```bash
sunshine &
# Client: Moonlight on Windows, macOS, iOS, Android, or Linux
```

[Source: Sunshine documentation](https://docs.lizardbyte.dev/projects/sunshine/en/latest/)

The architecture mirrors commercial cloud gaming platforms: the server renders the game at full GPU speed, encodes each frame within 16 ms (60 fps), and sends the compressed stream to the client over a local network or the internet.

### GFN-style architecture

NVIDIA GeForce Now (GFN) and similar services follow the same pattern at data-centre scale:

1. A server-side VM receives user input over a low-latency side channel.
2. A GPU renders the game frame in the VM at the full native resolution.
3. NVENC encodes the frame to H.264 or H.265 in hardware within ~2 ms.
4. The bitstream is delivered via SRTP/QUIC to the client.
5. The client decodes with NVDEC or a hardware H.264 decoder and displays with near-zero latency.

The key technical insight is that all GPU work — including the headless render and the video encode — happens on one physical card, and the render→encode path never touches CPU memory: the rendered framebuffer is passed to NVENC as a CUDA `CUdeviceptr` or a DMA-BUF, eliminating the readback penalty.

---

## Roadmap

### Near-term (6–12 months)

- Mesa 26.x continues expanding lavapipe's Vulkan extension coverage, particularly the extensions ANGLE requires for full WebGL2 support; the current gap leaves WebGL2 disabled on lavapipe-backed ANGLE, and closing it would make headless Chromium on CPU-only CI runners fully feature-complete. [Source](https://botbrowser.io/en/blog/mesa-llvmpipe-vs-swiftshader-chromium-linux/)
- llvmpipe's new ORCJIT backend — upstreamed to Mesa main in 2025 and now shipping in Mesa 26.x — adds RISC-V 64-bit JIT compilation, enabling headless rendering on RISC-V CI runners without a GPU; remaining work includes full RISC-V coverage parity with x86/AArch64. [Source](https://www.deepin.org/en/mesa-llvmpipe-orcjit-deepin/)
- VirtIO-GPU drops legacy VirGL support in Mesa 26.1, pushing all VM-based headless workloads onto the native-context (Venus Vulkan) or gfxstream (GLES) paths; distributions and CI images that have not migrated need updated base images before upgrading to Mesa 26.1. [Source](https://9to5linux.com/mesa-26-1-open-source-graphics-stack-officially-released-heres-whats-new)
- The rutabaga_gfx crate (standalone Rust re-packaging of the gfxstream rutabaga backend) is being integrated into upstream QEMU; once merged, cloud CI runners using QEMU can get gfxstream-accelerated headless rendering without patching QEMU. [Source](https://github.com/magma-gpu/rutabaga_gfx)
- Mesa 26.1 adds VirtIO-GPU native-context support for Intel Iris, Crocus, and ANV drivers, so Intel-GPU-backed QEMU hosts can now expose real Vulkan (not virgl-translated OpenGL) to guest VMs running headless workloads. [Source](https://9to5linux.com/mesa-26-1-open-source-graphics-stack-officially-released-heres-whats-new)

### Medium-term (1–3 years)

- Multiple DRM native-context types — `virtio-freedreno`, `virtio-intel`, `virtio-amdgpu` — are awaiting the addition of sync-object UAPI to the VirtIO-GPU kernel driver; once that UAPI lands, guests can run vendor GPU command streams directly against the host driver without any API translation, dramatically improving headless VM rendering performance. [Source](https://lkml.iu.edu/hypermail/linux/kernel/2304.1/00402.html)
- The Vulkan Roadmap 2026 milestone (published alongside Vulkan 1.4.340) mandates `VK_EXT_descriptor_heap` and other extensions on conformant implementations; lavapipe will need to implement these to remain a valid Vulkan Roadmap 2026 driver, which matters for headless CI runners using CTS-level conformance testing. [Source](https://www.khronos.org/blog/vulkan-introduces-roadmap-2026-and-new-descriptor-heap-extension)
- The Venus Mesa driver team has a long-term plan to define a new Vulkan extension allowing guest drivers to negotiate directly with host drivers, reducing the round-trip overhead of the current command-stream virtualisation and enabling tighter fence/timeline-semaphore integration — important for GPU-heavy headless workloads like training-loop visualisation. [Source](https://docs.mesa3d.org/drivers/venus.html)
- Kubernetes GPU scheduling improvements (device plugins, CDI — Container Device Interface) are expected to converge on a standard render-node allocation model, making headless GPU containers in CI clusters schedulable without per-vendor NVIDIA/AMD/Intel operator plugins. Note: needs verification against upstream CDI and CNCF roadmaps.
- Wlroots headless backend is expected to gain explicit output fractional scaling and HDR passthrough APIs as Wayland's `wp_fractional_scale_v1` and colour-management protocols stabilise, enabling headless CI screenshot testing at HiDPI and wide-gamut resolutions. Note: needs verification against wlroots upstream issue tracker.

### Long-term

- A unified "headless context" standard within the EGL and Vulkan ecosystems — analogous to `EGL_MESA_platform_surfaceless` but ratified as a cross-vendor KHR extension — would eliminate the current patchwork of vendor-specific environment variables (`LIBGL_ALWAYS_SOFTWARE`, `VK_DRIVER_FILES`, `WLR_BACKENDS`) and make headless setup portable across drivers and distributions. Note: needs verification; no formal proposal known as of mid-2026.
- GPU SR-IOV (Single Root I/O Virtualisation) for discrete AMD and Intel GPUs is progressing in both kernel and userspace; once stable, each CI runner in a shared data-centre rack could receive a hardware-isolated GPU partition, turning "headless software rendering" into "headless hardware rendering" for all CI tiers without full GPU passthrough. Note: needs verification against upstream i915 and amdgpu SR-IOV patch status.
- Declarative GPU workload description formats (similar to Docker's `--gpu` flag but standardised across NVIDIA, AMD, Intel, and software-fallback) could allow a single CI job descriptor to express "render with hardware GPU if available, else lavapipe", with the scheduler selecting the backend — removing the current per-project shell-script gymnastics around `LIBGL_ALWAYS_SOFTWARE` and `VK_DRIVER_FILES`. Note: needs verification.
- As WebGPU matures and Dawn/wgpu gain headless compute modes, the distinction between "headless rendering" and "headless GPU compute" is expected to collapse: a single `WGPUDevice` obtained from a Dawn or wgpu instance with no surface will serve both ML inference and frame rendering in server-side CI environments, removing the need for separate OpenGL/Vulkan/CUDA code paths. Note: needs verification against Dawn upstream roadmap.

---

## 11. Integrations

This chapter connects to the following chapters throughout the book:

- **Ch1 — DRM Architecture** (`ch01-drm-architecture.md`): Headless rendering uses DRM *render nodes* (`/dev/dri/renderD*`) exclusively. Render nodes are the DRM design decision that makes GPU access possible without a display or compositor; the primary node (`card0`) is not needed. Understanding file-descriptor ownership, GEM object lifetimes, and the render-node permission model (Section 2.3 of Ch1) is a prerequisite for the GBM and EGL paths in this chapter.

- **Ch4 — GPU Memory Management** (`ch04-gpu-memory-management.md`): GBM-based offscreen rendering (Section 4 of this chapter) allocates GEM buffer objects and exports them as DMA-BUF file descriptors. The GEM/TTM/VRAM object lifecycle, DMA-BUF importer/exporter protocol, and fence signalling described in Ch4 underpin every zero-copy handoff from the GPU renderer to an encoder or a network streamer.

- **Ch24 — EGL for Application Developers**: The `EGL_MESA_platform_surfaceless` and `EGL_PLATFORM_GBM_KHR` platform tokens described in Section 3 are part of the broader EGL WSI story. Ch24 covers the full EGL surface-creation path for Wayland and X11 — `eglGetPlatformDisplayEXT`, config selection, window surfaces, and sync fences — while this chapter covers the headless subset of the same API surface, specifically the surfaceless and GBM platforms.

- **Ch17 — Mesa Software Renderers** (`ch17-software-renderers.md`): llvmpipe and lavapipe, introduced in Section 2, are the subjects of Ch17. That chapter covers the Gallium3D pipe-driver interface, the LLVM JIT pipeline inside llvmpipe, softpipe as the pre-LLVM fallback, and performance profiling methodology for software rendering. This chapter treats the software renderers as deployment tools; Ch17 explains how they work internally.

- **Ch21 — Building Compositors with wlroots** (`ch21-building-compositors-wlroots.md`): The wlroots headless backend (Section 6) is a first-class feature of the library. Ch21 covers the wlroots architecture, output and input backends, and the renderer abstraction. The `WLR_BACKENDS=headless` path described here is a direct instantiation of the backend plugin system Ch21 explains.

- **Ch55 — GPU Containers and Cloud** (`ch55-gpu-containers-cloud.md`): The container-level GPU access techniques in Section 8 (render-node passthrough, NVIDIA Container Toolkit, ROCm devices, Kubernetes GPU scheduling) are covered in greater depth in Ch55, which also covers GPU topology in multi-GPU nodes, MIG (Multi-Instance GPU), and SR-IOV GPU virtualisation.

- **Ch57 — FFmpeg** (`ch57-ffmpeg.md`): Server-side video transcoding (Section 9) depends on the FFmpeg VAAPI and NVENC hardware-encode paths. Ch57 covers the full FFmpeg GPU pipeline — hwaccel context setup, `AVHWFramesContext`, VAAPI filter chains, and the DMA-BUF interop between the decoder, scaler, and encoder — in detail that Section 9 of this chapter only summarises.

- **Ch89 — GPU Virtualisation** (`ch89-gpu-virtualization.md`): Section 10's coverage of `virtio-gpu`, rutabaga, and the Venus Vulkan virtualisation protocol is the application layer of the virtualisation infrastructure Ch89 describes. Ch89 explains the guest Mesa driver → virtio-gpu command stream → host rutabaga → host GPU path from the kernel/driver perspective; this chapter shows how to configure and use that path for headless rendering in a VM.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
