# Appendix R: EGL Platform Reference

> **Reference baseline**: EGL 1.5 + KHR/EXT/MESA extensions / Mesa 25.1 / June 2026
> **Primary sources**: [Khronos EGL Registry](https://registry.khronos.org/EGL/) | Mesa `src/egl/` | [EGL spec 1.5](https://registry.khronos.org/EGL/specs/eglspec.1.5.pdf)

**Audience**: Graphics application developers creating EGL contexts for OpenGL ES, Vulkan WSI, GBM-based compositors, and headless rendering. See Chapter 24 (EGL and GBM) for architecture and Chapter 4 (DMA-BUF) for buffer sharing.

---

## Table of Contents

1. [EGL Initialisation Sequence](#1-egl-initialisation-sequence)
2. [Platform Extension Matrix](#2-platform-extension-matrix)
3. [eglChooseConfig Attribute Reference](#3-eglchooseconfig-attribute-reference)
4. [Surface Types and Creation](#4-surface-types-and-creation)
5. [Context Creation Attributes](#5-context-creation-attributes)
6. [Key EGL Extensions for Linux](#6-key-egl-extensions-for-linux)
7. [EGL Error Codes](#7-egl-error-codes)
8. [Common Patterns](#8-common-patterns)

---

## 1. EGL Initialisation Sequence

```c
// 1. Get a platform display (preferred over eglGetDisplay)
EGLDisplay dpy = eglGetPlatformDisplay(EGL_PLATFORM_WAYLAND_KHR, wl_display, NULL);
// or: EGL_PLATFORM_GBM_KHR, EGL_PLATFORM_X11_KHR, EGL_PLATFORM_SURFACELESS_MESA

// 2. Initialise and query version
EGLint major, minor;
eglInitialize(dpy, &major, &minor);           // sets major=1, minor=5 on Mesa

// 3. Choose a config
EGLConfig config;
EGLint n_configs;
eglChooseConfig(dpy, attribs, &config, 1, &n_configs);

// 4. Create a surface
EGLSurface surf = eglCreateWindowSurface(dpy, config, native_window, NULL);
// or: eglCreatePbufferSurface / eglCreatePixmapSurface

// 5. Create a context
EGLContext ctx = eglCreateContext(dpy, config, EGL_NO_CONTEXT, ctx_attribs);

// 6. Make current
eglMakeCurrent(dpy, surf, surf, ctx);

// 7. Render loop: swap buffers
eglSwapBuffers(dpy, surf);

// 8. Cleanup
eglDestroyContext(dpy, ctx);
eglDestroySurface(dpy, surf);
eglTerminate(dpy);
```

---

## 2. Platform Extension Matrix

| Platform                      | Extension token                  | Native display type         | Native window type        | Mesa support |
|-------------------------------|----------------------------------|-----------------------------|---------------------------|--------------|
| Wayland                       | `EGL_PLATFORM_WAYLAND_KHR`       | `struct wl_display *`       | `struct wl_egl_window *`  | Full         |
| GBM (DRM/KMS)                 | `EGL_PLATFORM_GBM_KHR`           | `struct gbm_device *`       | `struct gbm_surface *`    | Full         |
| X11 / Xlib                    | `EGL_PLATFORM_X11_KHR`           | `Display *`                 | `Window`                  | Full         |
| X11 / XCB                     | `EGL_PLATFORM_XCB_EXT`           | `xcb_connection_t *`        | `xcb_window_t`            | Mesa 23+     |
| Surfaceless (headless)        | `EGL_PLATFORM_SURFACELESS_MESA`  | `EGL_DEFAULT_DISPLAY`       | N/A (pbuffer/no surface)  | Full         |
| Android                       | `EGL_PLATFORM_ANDROID_KHR`       | `EGLNativeDisplayType`      | `ANativeWindow *`         | N/A on Linux |
| Device (EGLDevice)            | `EGL_PLATFORM_DEVICE_EXT`        | `EGLDeviceEXT`              | N/A                       | Mesa 23+     |

**Querying supported platforms:**
```c
const char *exts = eglQueryString(EGL_NO_DISPLAY, EGL_EXTENSIONS);
// Contains space-separated extension names; check with strstr()
```

**Device enumeration (for multi-GPU headless):**
```c
EGLint n_devs;
eglQueryDevicesEXT(0, NULL, &n_devs);            // EGL_EXT_device_enumeration
EGLDeviceEXT devs[8];
eglQueryDevicesEXT(n_devs, devs, &n_devs);
// Query DRM device node:
const char *path = eglQueryDeviceStringEXT(devs[i], EGL_DRM_DEVICE_FILE_EXT);
EGLDisplay dpy = eglGetPlatformDisplay(EGL_PLATFORM_DEVICE_EXT, devs[i], NULL);
```

---

## 3. eglChooseConfig Attribute Reference

Pass as `EGLint attribs[]` terminated with `EGL_NONE`.

### Colour Buffer

| Attribute                | Values / Default    | Description                                          |
|--------------------------|---------------------|------------------------------------------------------|
| `EGL_RED_SIZE`           | int / 0             | Minimum red bits                                     |
| `EGL_GREEN_SIZE`         | int / 0             | Minimum green bits                                   |
| `EGL_BLUE_SIZE`          | int / 0             | Minimum blue bits                                    |
| `EGL_ALPHA_SIZE`         | int / 0             | Minimum alpha bits                                   |
| `EGL_COLOR_BUFFER_TYPE`  | `EGL_RGB_BUFFER` / `EGL_LUMINANCE_BUFFER` | Buffer type        |
| `EGL_BUFFER_SIZE`        | int / 0             | Minimum total colour bits (sum of RGBA)              |
| `EGL_LUMINANCE_SIZE`     | int / 0             | Minimum luminance bits (luminance buffer only)       |

### Depth and Stencil

| Attribute                | Values / Default    | Description                                          |
|--------------------------|---------------------|------------------------------------------------------|
| `EGL_DEPTH_SIZE`         | int / 0             | Minimum depth buffer bits (0 = no depth buffer)      |
| `EGL_STENCIL_SIZE`       | int / 0             | Minimum stencil buffer bits                          |

### Multisample

| Attribute                | Values / Default    | Description                                          |
|--------------------------|---------------------|------------------------------------------------------|
| `EGL_SAMPLE_BUFFERS`     | 0 or 1 / 0         | Enable multisample buffer                            |
| `EGL_SAMPLES`            | int / 0             | Samples per pixel (2, 4, 8, 16)                      |

### Rendering API and Surface Type

| Attribute                | Values / Default                     | Description                              |
|--------------------------|--------------------------------------|------------------------------------------|
| `EGL_RENDERABLE_TYPE`    | bitmask / `EGL_OPENGL_ES2_BIT`       | Required rendering APIs                  |
|                          | `EGL_OPENGL_ES_BIT` (ES 1.x)         |                                          |
|                          | `EGL_OPENGL_ES2_BIT` (ES 2.0)        |                                          |
|                          | `EGL_OPENGL_ES3_BIT` (ES 3.x)        |                                          |
|                          | `EGL_OPENGL_BIT` (desktop GL)        |                                          |
|                          | `EGL_OPENVG_BIT`                      |                                          |
| `EGL_SURFACE_TYPE`       | bitmask / `EGL_WINDOW_BIT`           | Required surface types                   |
|                          | `EGL_WINDOW_BIT`                      | Window surface                           |
|                          | `EGL_PBUFFER_BIT`                     | Off-screen pixel buffer                  |
|                          | `EGL_PIXMAP_BIT`                      | Native pixmap                            |
|                          | `EGL_MULTISAMPLE_RESOLVE_BOX_BIT`    | Box filter for MSAA resolve              |
|                          | `EGL_SWAP_BEHAVIOR_PRESERVED_BIT`    | Back buffer preserved after swap         |

### Config Selection and Caveat

| Attribute                | Values / Default                    | Description                               |
|--------------------------|-------------------------------------|-------------------------------------------|
| `EGL_CONFIG_CAVEAT`      | `EGL_NONE` / `EGL_SLOW_CONFIG` / `EGL_NON_CONFORMANT_CONFIG` | Performance/conformance flag |
| `EGL_CONFIG_ID`          | int                                 | Select a specific config by ID            |
| `EGL_NATIVE_RENDERABLE`  | `EGL_TRUE` / `EGL_FALSE` / `EGL_DONT_CARE` | Native rendering API usable     |
| `EGL_TRANSPARENT_TYPE`   | `EGL_NONE` / `EGL_TRANSPARENT_RGB` | Transparency support                      |
| `EGL_LEVEL`              | int / 0                             | Framebuffer overlay/underlay level        |
| `EGL_MAX_PBUFFER_WIDTH`  | int (query only)                    | Maximum pbuffer width                     |
| `EGL_MAX_PBUFFER_HEIGHT` | int (query only)                    | Maximum pbuffer height                    |

**Typical config for OpenGL ES 3.2, RGBA8, depth24, MSAA×4:**
```c
EGLint attribs[] = {
    EGL_RENDERABLE_TYPE, EGL_OPENGL_ES3_BIT,
    EGL_SURFACE_TYPE,    EGL_WINDOW_BIT,
    EGL_RED_SIZE,        8,
    EGL_GREEN_SIZE,      8,
    EGL_BLUE_SIZE,       8,
    EGL_ALPHA_SIZE,      8,
    EGL_DEPTH_SIZE,      24,
    EGL_SAMPLE_BUFFERS,  1,
    EGL_SAMPLES,         4,
    EGL_NONE
};
```

---

## 4. Surface Types and Creation

### Window Surface

```c
// Wayland: wl_egl_window is created from a wl_surface
struct wl_egl_window *win = wl_egl_window_create(wl_surface, width, height);
EGLSurface surf = eglCreateWindowSurface(dpy, config, (EGLNativeWindowType)win, NULL);

// GBM: gbm_surface wraps a gbm_device + format + usage
struct gbm_surface *gbm_surf = gbm_surface_create(gbm_dev, w, h,
    GBM_FORMAT_XRGB8888, GBM_BO_USE_SCANOUT | GBM_BO_USE_RENDERING);
EGLSurface surf = eglCreateWindowSurface(dpy, config, (EGLNativeWindowType)gbm_surf, NULL);
```

### Pbuffer Surface (Headless Rendering)

```c
EGLint pb_attribs[] = {
    EGL_WIDTH,  1920,
    EGL_HEIGHT, 1080,
    EGL_NONE
};
EGLSurface pb = eglCreatePbufferSurface(dpy, config, pb_attribs);
```

### Surfaceless Rendering (`EGL_KHR_surfaceless_context`)

```c
// No surface at all — render to FBO only
eglMakeCurrent(dpy, EGL_NO_SURFACE, EGL_NO_SURFACE, ctx);
// Requires config with EGL_SURFACE_TYPE containing EGL_PBUFFER_BIT or similar
```

### Swap Interval

```c
eglSwapInterval(dpy, 0);  // 0 = no vsync; 1 = vsync; -1 = adaptive vsync (ext)
```

### DMA-BUF Import as EGLImage (`EGL_EXT_image_dma_buf_import`)

```c
EGLint img_attribs[] = {
    EGL_WIDTH,                          width,
    EGL_HEIGHT,                         height,
    EGL_LINUX_DRM_FOURCC_EXT,           DRM_FORMAT_ARGB8888,
    EGL_DMA_BUF_PLANE0_FD_EXT,         dmabuf_fd,
    EGL_DMA_BUF_PLANE0_OFFSET_EXT,     0,
    EGL_DMA_BUF_PLANE0_PITCH_EXT,      stride,
    EGL_DMA_BUF_PLANE0_MODIFIER_LO_EXT, modifier & 0xFFFFFFFF,
    EGL_DMA_BUF_PLANE0_MODIFIER_HI_EXT, modifier >> 32,
    EGL_NONE
};
EGLImageKHR image = eglCreateImageKHR(dpy, EGL_NO_CONTEXT,
    EGL_LINUX_DMA_BUF_EXT, NULL, img_attribs);
// Bind to texture:
glEGLImageTargetTexture2DOES(GL_TEXTURE_EXTERNAL_OES, image);
```

---

## 5. Context Creation Attributes

Pass as `EGLint attribs[]` to `eglCreateContext(dpy, config, share_ctx, attribs)`.

| Attribute                            | Values                                      | Description                             |
|--------------------------------------|---------------------------------------------|-----------------------------------------|
| `EGL_CONTEXT_MAJOR_VERSION`          | int / 1                                     | API major version (ES: 1/2/3; GL: 1–4)  |
| `EGL_CONTEXT_MINOR_VERSION`          | int / 0                                     | API minor version                       |
| `EGL_CONTEXT_OPENGL_PROFILE_MASK`    | `EGL_CONTEXT_OPENGL_CORE_PROFILE_BIT` / `EGL_CONTEXT_OPENGL_COMPATIBILITY_PROFILE_BIT` | GL profile (desktop GL only) |
| `EGL_CONTEXT_OPENGL_DEBUG`           | `EGL_TRUE` / `EGL_FALSE`                    | Enable GL debug output                  |
| `EGL_CONTEXT_OPENGL_FORWARD_COMPATIBLE` | `EGL_TRUE` / `EGL_FALSE`                | Forward-compatible context (no deprecated features) |
| `EGL_CONTEXT_OPENGL_ROBUST_ACCESS`   | `EGL_TRUE` / `EGL_FALSE`                    | Enable robust buffer access             |
| `EGL_CONTEXT_OPENGL_RESET_NOTIFICATION_STRATEGY` | `EGL_NO_RESET_NOTIFICATION` / `EGL_LOSE_CONTEXT_ON_RESET` | GPU reset handling |
| `EGL_CONTEXT_FLAGS_KHR`              | bitmask: `EGL_CONTEXT_OPENGL_DEBUG_BIT_KHR` / `EGL_CONTEXT_OPENGL_FORWARD_COMPATIBLE_BIT_KHR` / `EGL_CONTEXT_OPENGL_ROBUST_ACCESS_BIT_KHR` | Combined flags |

**Selecting the rendering API before context creation:**
```c
eglBindAPI(EGL_OPENGL_ES_API);   // for OpenGL ES contexts
eglBindAPI(EGL_OPENGL_API);       // for desktop OpenGL contexts
eglBindAPI(EGL_OPENVG_API);       // for OpenVG (rare)
```

**Shared context (share texture/buffer objects):**
```c
EGLContext shared_ctx = eglCreateContext(dpy, config, EGL_NO_CONTEXT, attribs);
EGLContext worker_ctx = eglCreateContext(dpy, config, shared_ctx, attribs);
```

---

## 6. Key EGL Extensions for Linux

### Surface / Platform

| Extension                          | Purpose                                                              |
|------------------------------------|----------------------------------------------------------------------|
| `EGL_KHR_platform_wayland`         | Wayland native display/window platform                               |
| `EGL_KHR_platform_gbm`             | GBM device as EGL platform (DRM/KMS compositors)                     |
| `EGL_KHR_platform_x11`             | X11 Xlib display as EGL platform                                     |
| `EGL_EXT_platform_xcb`             | X11 XCB connection as EGL platform                                   |
| `EGL_MESA_platform_surfaceless`    | Surfaceless rendering (headless, no native display)                  |
| `EGL_EXT_platform_device`          | EGLDevice enumeration for multi-GPU headless selection               |

### Buffer Sharing

| Extension                              | Purpose                                                          |
|----------------------------------------|------------------------------------------------------------------|
| `EGL_EXT_image_dma_buf_import`         | Import a DMA-BUF fd as `EGLImageKHR` (zero-copy texture)       |
| `EGL_EXT_image_dma_buf_import_modifiers`| DMA-BUF import with DRM format modifiers                      |
| `EGL_MESA_image_dma_buf_export`        | Export an `EGLImage` as a DMA-BUF fd                           |
| `EGL_KHR_image_base`                   | Base `EGLImageKHR` infrastructure                               |
| `EGL_KHR_gl_texture_2D_image`          | Create `EGLImage` from a GL texture                             |

### Sync / Fence

| Extension                          | Purpose                                                              |
|------------------------------------|----------------------------------------------------------------------|
| `EGL_KHR_fence_sync`               | `EGLSyncKHR` fence inserted into the GPU command stream              |
| `EGL_KHR_wait_sync`                | Server-side sync wait (GPU waits without CPU wakeup)                 |
| `EGL_ANDROID_native_fence_sync`    | Export `EGLSyncKHR` as a Linux sync_file fd (used for explicit sync) |
| `EGL_KHR_reusable_sync`            | Reusable sync object (for event signalling)                          |

### Context and Display

| Extension                          | Purpose                                                              |
|------------------------------------|----------------------------------------------------------------------|
| `EGL_KHR_create_context`           | Context major/minor version, profile, debug flags                    |
| `EGL_KHR_surfaceless_context`      | `eglMakeCurrent` with `EGL_NO_SURFACE`                               |
| `EGL_KHR_no_config_context`        | Create context without a config (for Vulkan/EGL interop)             |
| `EGL_MESA_configless_context`      | Mesa extension for configless context (superseded by KHR version)    |
| `EGL_EXT_swap_buffers_with_damage` | Partial swap — supply dirty rectangles to compositor for optimisation|
| `EGL_KHR_swap_buffers_with_damage` | KHR version of above                                                 |
| `EGL_EXT_present_opaque`           | Treat alpha channel as opaque (avoid pre-multiplied alpha issues)    |

### Query

| Extension                          | Purpose                                                              |
|------------------------------------|----------------------------------------------------------------------|
| `EGL_KHR_query_surface_pointer`    | Map `EGLSurface` back to the native window handle                    |
| `EGL_EXT_device_query`             | Query the `EGLDeviceEXT` for a display                               |
| `EGL_EXT_device_drm`               | Query DRM device node path from `EGLDeviceEXT`                       |

---

## 7. EGL Error Codes

| Error                            | `eglGetError()` value | Common cause                                            |
|----------------------------------|-----------------------|---------------------------------------------------------|
| `EGL_SUCCESS`                    | `0x3000`              | No error                                                |
| `EGL_NOT_INITIALIZED`            | `0x3001`              | `eglInitialize` not called or failed                    |
| `EGL_BAD_ACCESS`                 | `0x3002`              | Thread already has a context; resource already bound    |
| `EGL_BAD_ALLOC`                  | `0x3003`              | Out of memory                                           |
| `EGL_BAD_ATTRIBUTE`              | `0x3004`              | Unknown attribute in attrib list                        |
| `EGL_BAD_CONFIG`                 | `0x3005`              | Invalid `EGLConfig`                                     |
| `EGL_BAD_CONTEXT`                | `0x3006`              | Invalid `EGLContext`                                    |
| `EGL_BAD_CURRENT_SURFACE`        | `0x3007`              | Current surface is no longer valid                      |
| `EGL_BAD_DISPLAY`                | `0x3008`              | Invalid `EGLDisplay`                                    |
| `EGL_BAD_MATCH`                  | `0x3009`              | Config/surface/context attribute mismatch               |
| `EGL_BAD_NATIVE_PIXMAP`          | `0x300A`              | Invalid native pixmap                                   |
| `EGL_BAD_NATIVE_WINDOW`          | `0x300B`              | Invalid native window (e.g. destroyed `wl_egl_window`)  |
| `EGL_BAD_PARAMETER`              | `0x300C`              | NULL pointer where disallowed; negative size            |
| `EGL_BAD_SURFACE`                | `0x300D`              | Invalid `EGLSurface`                                    |
| `EGL_CONTEXT_LOST`               | `0x300E`              | GPU reset; context must be recreated                    |

```c
// Error checking helper
static void check_egl(const char *op) {
    EGLint e = eglGetError();
    if (e != EGL_SUCCESS)
        fprintf(stderr, "%s: EGL error 0x%04x\n", op, e);
}
```

---

## 8. Common Patterns

### GBM Compositor Loop (DRM/KMS direct rendering)

```c
struct gbm_device  *gbm = gbm_create_device(drm_fd);
EGLDisplay dpy = eglGetPlatformDisplay(EGL_PLATFORM_GBM_KHR, gbm, NULL);
eglInitialize(dpy, NULL, NULL);
// ... create config, surface from gbm_surface, context ...
eglMakeCurrent(dpy, surf, surf, ctx);

// Render loop:
while (1) {
    render_frame();
    eglSwapBuffers(dpy, surf);                       // locks front BO
    struct gbm_bo *bo = gbm_surface_lock_front_buffer(gbm_surf);
    uint32_t fb = drm_import_bo(bo);                 // drmModeAddFB2
    drmModePageFlip(drm_fd, crtc_id, fb, DRM_MODE_PAGE_FLIP_EVENT, NULL);
    wait_for_flip();
    gbm_surface_release_buffer(gbm_surf, prev_bo);   // release previous
}
```

### Headless EGL with DMA-BUF Export

```c
// Create surfaceless context
EGLDisplay dpy = eglGetPlatformDisplay(EGL_PLATFORM_SURFACELESS_MESA,
    EGL_DEFAULT_DISPLAY, NULL);
eglInitialize(dpy, NULL, NULL);
eglBindAPI(EGL_OPENGL_ES_API);
EGLContext ctx = eglCreateContext(dpy, config, EGL_NO_CONTEXT, ctx_attribs);
eglMakeCurrent(dpy, EGL_NO_SURFACE, EGL_NO_SURFACE, ctx);

// Render into FBO, then export texture as DMA-BUF:
EGLImageKHR img = eglCreateImageKHR(dpy, ctx, EGL_GL_TEXTURE_2D_KHR,
    (EGLClientBuffer)(uintptr_t)texture_id, NULL);
int dmabuf_fd;
EGLint stride, fourcc;
eglExportDMABUFImageMESA(dpy, img, &dmabuf_fd, &stride, &fourcc);
```

---

## Cross-References

- **Chapter 24** — EGL and GBM: platform architecture, `eglChooseConfig` selection strategy, Wayland/GBM surface lifecycle
- **Chapter 4** — DMA-BUF: `EGL_EXT_image_dma_buf_import` in the buffer sharing pipeline
- **Chapter 20** — linux-dmabuf Wayland protocol: how Wayland compositors use EGL DMA-BUF import
- **Chapter 21** — wlroots: GBM/EGL compositor backend
- **Appendix G** — Synchronisation Primitives: `EGL_KHR_fence_sync` and `EGL_ANDROID_native_fence_sync` in context
- **Appendix N** — Vulkan Linux Platform Extensions: Vulkan WSI extensions that complement EGL on the same DRM stack

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
