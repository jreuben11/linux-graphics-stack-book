<!-- Reference baseline: Linux kernel 6.12 / Mesa 25.1 / Vulkan spec 1.3.290 / June 2026 -->
<!-- Primary sources: https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html -->
<!-- Mesa WSI: src/vulkan/wsi/ (Mesa 25.1, https://gitlab.freedesktop.org/mesa/mesa) -->

# Appendix N: Vulkan on Linux — Platform Extensions Reference

**Audience**: Systems developers, Vulkan application developers, and compositor authors who need a precise reference for the Linux-specific Vulkan extensions that underpin Wayland WSI, zero-copy dma-buf buffer sharing, direct KMS display acquisition, and cross-API synchronisation. This appendix assumes familiarity with the core Vulkan object model (instances, physical devices, logical devices, memory, synchronisation primitives) and with Linux kernel concepts such as DRM, dma-buf file descriptors, and sync_file fences. Ch24 (Vulkan and EGL for Application Developers) provides the entry-level treatment; Ch1 (DRM Architecture) and Ch4 (GPU Memory Management) cover the kernel side.

---

## Table of Contents

1. [Extension Discovery and Function Pointer Loading](#1-extension-discovery-and-function-pointer-loading)
2. [VK_KHR_wayland_surface — Wayland WSI](#2-vk_khr_wayland_surface--wayland-wsi)
3. [VK_EXT_image_drm_format_modifier — Format Modifiers for Vulkan Images](#3-vk_ext_image_drm_format_modifier--format-modifiers-for-vulkan-images)
4. [VK_KHR_external_memory_fd — File-Descriptor External Memory](#4-vk_khr_external_memory_fd--file-descriptor-external-memory)
5. [VK_EXT_external_memory_dma_buf — dma-buf Handle Type](#5-vk_ext_external_memory_dma_buf--dma-buf-handle-type)
6. [VK_KHR_external_semaphore_fd — Sync-FD Semaphore Import/Export](#6-vk_khr_external_semaphore_fd--sync-fd-semaphoreimportexport)
7. [VK_KHR_external_fence_fd — Fence File Descriptor Export](#7-vk_khr_external_fence_fd--fence-file-descriptor-export)
8. [VK_EXT_acquire_drm_display — Direct KMS Display Ownership](#8-vk_ext_acquire_drm_display--direct-kms-display-ownership)
9. [VK_KHR_display and VK_KHR_display_swapchain — Direct Scanout](#9-vk_khr_display-and-vk_khr_display_swapchain--direct-scanout)
10. [VK_EXT_headless_surface — Offscreen and Headless Rendering](#10-vk_ext_headless_surface--offscreen-and-headless-rendering)
11. [End-to-End: Zero-Copy dma-buf Import Flow](#11-end-to-end-zero-copy-dma-buf-import-flow)
12. [Compositor Extension Chain](#12-compositor-extension-chain)
13. [Limitations and Gotchas](#13-limitations-and-gotchas)
14. [Extension Support Matrix](#14-extension-support-matrix)
15. [Mesa Source Map](#15-mesa-source-map)
16. [Integrations](#16-integrations)
17. [References](#17-references)


---

## 1. Extension Discovery and Function Pointer Loading

Before using any of the extensions described in this appendix, the application must confirm they are available and load every entry point explicitly. The Vulkan loader does **not** export extension entry points as ordinary symbols; they must be resolved at runtime via `vkGetInstanceProcAddr` (for instance-level commands) or `vkGetDeviceProcAddr` (for device-level commands).

```c
/* Extension discovery: check instance-level extensions */
uint32_t extCount;
vkEnumerateInstanceExtensionProperties(NULL, &extCount, NULL);
VkExtensionProperties *exts = malloc(sizeof(*exts) * extCount);
vkEnumerateInstanceExtensionProperties(NULL, &extCount, exts);

/* Required instance extensions for Wayland WSI + DRM display */
static const char *instanceExts[] = {
    "VK_KHR_surface",
    "VK_KHR_wayland_surface",
    "VK_KHR_get_physical_device_properties2",  /* prerequisite for several EXTs */
    "VK_KHR_display",
    "VK_EXT_acquire_drm_display",
    "VK_EXT_headless_surface",
};

/* Required device extensions for dma-buf interop + sync */
static const char *deviceExts[] = {
    "VK_KHR_swapchain",
    "VK_EXT_image_drm_format_modifier",
    "VK_KHR_external_memory_fd",
    "VK_EXT_external_memory_dma_buf",
    "VK_KHR_external_semaphore_fd",
    "VK_KHR_external_fence_fd",
    "VK_KHR_external_memory",           /* base KHR external memory */
    "VK_KHR_external_semaphore",        /* base KHR external semaphore */
    "VK_KHR_external_fence",            /* base KHR external fence */
    "VK_KHR_image_format_list",         /* often needed alongside DRM modifier ext */
};

/* Load a device-level function pointer (do this once after vkCreateDevice): */
PFN_vkGetMemoryFdKHR           pfnGetMemoryFdKHR;
PFN_vkGetMemoryFdPropertiesKHR pfnGetMemoryFdPropertiesKHR;
PFN_vkImportSemaphoreFdKHR     pfnImportSemaphoreFdKHR;
PFN_vkGetSemaphoreFdKHR        pfnGetSemaphoreFdKHR;
PFN_vkImportFenceFdKHR         pfnImportFenceFdKHR;
PFN_vkGetFenceFdKHR            pfnGetFenceFdKHR;

#define LOAD_DEV(name) \
    pfn##name = (PFN_vk##name)vkGetDeviceProcAddr(device, "vk" #name); \
    assert(pfn##name)

LOAD_DEV(GetMemoryFdKHR);
LOAD_DEV(GetMemoryFdPropertiesKHR);
LOAD_DEV(ImportSemaphoreFdKHR);
LOAD_DEV(GetSemaphoreFdKHR);
LOAD_DEV(ImportFenceFdKHR);
LOAD_DEV(GetFenceFdKHR);
```

Instance-level extension entry points (`vkCreateWaylandSurfaceKHR`, `vkAcquireDrmDisplayEXT`, `vkGetDrmDisplayEXT`, etc.) are loaded via `vkGetInstanceProcAddr(instance, "vkFunctionName")`.

[Vulkan 1.3 spec §3.1 — Command Function Pointers](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#initialization-functionpointers)

---

## 2. VK_KHR_wayland_surface — Wayland WSI

**Problem it solves.** Vulkan's platform-agnostic `VkSurfaceKHR` abstraction requires a platform-specific creation extension that knows how to bind a Vulkan swapchain to the native windowing system. On Linux Wayland compositors, the native window handle is a pair `(wl_display *, wl_surface *)`. `VK_KHR_wayland_surface` provides that binding. [Spec](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_wayland_surface.html)

**Key structs and calls.**

```c
#include <vulkan/vulkan_wayland.h>
#include <wayland-client.h>

/* --- Surface creation --- */
struct wl_display *wlDisplay = wl_display_connect(NULL);
struct wl_surface *wlSurface = ...; /* obtained from wl_compositor_create_surface() */

VkWaylandSurfaceCreateInfoKHR surfaceCI = {
    .sType   = VK_STRUCTURE_TYPE_WAYLAND_SURFACE_CREATE_INFO_KHR,
    .pNext   = NULL,
    .flags   = 0,
    .display = wlDisplay,
    .surface = wlSurface,
};

PFN_vkCreateWaylandSurfaceKHR pfnCreateWaylandSurface =
    (PFN_vkCreateWaylandSurfaceKHR)
    vkGetInstanceProcAddr(instance, "vkCreateWaylandSurfaceKHR");

VkSurfaceKHR surface;
VkResult res = pfnCreateWaylandSurface(instance, &surfaceCI, NULL, &surface);
assert(res == VK_SUCCESS);

/* --- Presentation support query --- */
PFN_vkGetPhysicalDeviceWaylandPresentationSupportKHR pfnWlSupport =
    (PFN_vkGetPhysicalDeviceWaylandPresentationSupportKHR)
    vkGetInstanceProcAddr(instance, "vkGetPhysicalDeviceWaylandPresentationSupportKHR");

VkBool32 supported = pfnWlSupport(physicalDevice,
                                   queueFamilyIndex,
                                   wlDisplay);
```

**Mesa support.** Mesa has supported `VK_KHR_wayland_surface` since Mesa 17.x (the WSI Wayland backend was part of the initial ANV/RADV Vulkan driver launch). The implementation lives in `src/vulkan/wsi/wsi_common_wayland.c` ([Mesa 25.1 source](https://gitlab.freedesktop.org/mesa/mesa/-/blob/mesa-25.1.0/src/vulkan/wsi/wsi_common_wayland.c)). All hardware Mesa Vulkan drivers (ANV, RADV, NVK, Turnip, panvk, v3dv) share this WSI layer.

**What Mesa WSI does internally.** When `vkCreateSwapchainKHR` is called on a Wayland surface, the WSI layer performs a format/modifier negotiation dance with the compositor before allocating any images. The sequence is:
1. The layer connects to the compositor's `zwp_linux_dmabuf_feedback_v1` global (Mesa 22.0+; older Mesa used `zwp_linux_dmabuf_v1`) and receives a feedback table enumerating `(format, modifier)` pairs ordered by the compositor's preference (first entry being the scanout-capable pair).
2. The layer allocates `minImageCount` GBM `gbm_bo` objects using the preferred modifier from that table, then exports each as a dma-buf fd.
3. It imports each dma-buf fd into a `VkDeviceMemory` (via `VK_KHR_external_memory_fd`) and creates a `VkImage` over it (via `VK_EXT_image_drm_format_modifier`).
4. It wraps each image in a `wl_buffer` object advertised to the compositor via `zwp_linux_buffer_params_v1`.
5. At `vkQueuePresentKHR` time it calls `wl_surface_commit` with the current back-buffer attached. If explicit sync is enabled (`wp_linux_drm_syncobj_v1`, landed in Mesa 24.1), it also attaches the acquire and release syncobj timeline points for that frame.

This means the Wayland protocol traffic is completely hidden from the application; from the application's perspective the frame loop is just `vkAcquireNextImageKHR` → render → `vkQueuePresentKHR`.

**Where it appears in the frame loop.** Surface creation is one-time at startup. The `VkSwapchainKHR` created from this surface then drives the compositor's `wl_buffer` submission protocol internally — the application never touches Wayland buffer APIs directly when using `VK_KHR_swapchain`. The WSI layer negotiates buffer format, modifier, and number of images with the compositor through `zwp_linux_dmabuf_v1` or `zwp_linux_dmabuf_feedback_v1` (see §12).

---

## 3. VK_EXT_image_drm_format_modifier — Format Modifiers for Vulkan Images

**Problem it solves.** GPU hardware stores textures and render targets in vendor-specific tiled or compressed layouts described by DRM format modifiers (see Appendix H). Without this extension, Vulkan has no mechanism to create a `VkImage` with a specific tiling modifier, nor to query which modifier a driver chose. This matters for zero-copy: if a GBM allocation uses `AMD_FMT_MOD_TILE_GFX9_64K_S`, the corresponding `VkImage` must use the same modifier to avoid a copy. [Spec](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_image_drm_format_modifier.html)

**Key structs and calls.**

```c
/* Query which modifiers the driver supports for a given format */
VkDrmFormatModifierPropertiesListEXT modList = {
    .sType = VK_STRUCTURE_TYPE_DRM_FORMAT_MODIFIER_PROPERTIES_LIST_EXT,
};
VkFormatProperties2 fmtProps2 = {
    .sType = VK_STRUCTURE_TYPE_FORMAT_PROPERTIES_2,
    .pNext = &modList,
};
vkGetPhysicalDeviceFormatProperties2(physDev, VK_FORMAT_B8G8R8A8_UNORM, &fmtProps2);

/* First call: get count */
uint32_t n = modList.drmFormatModifierCount;
VkDrmFormatModifierPropertiesEXT *modProps =
    malloc(n * sizeof(VkDrmFormatModifierPropertiesEXT));
modList.pDrmFormatModifierProperties = modProps;
vkGetPhysicalDeviceFormatProperties2(physDev, VK_FORMAT_B8G8R8A8_UNORM, &fmtProps2);

/* Choose a modifier — e.g. the first one that supports USAGE_COLOR_ATTACHMENT */
uint64_t chosenMod = DRM_FORMAT_MOD_LINEAR;
for (uint32_t i = 0; i < n; i++) {
    if (modProps[i].drmFormatModifierTilingFeatures &
        VK_FORMAT_FEATURE_COLOR_ATTACHMENT_BIT) {
        chosenMod = modProps[i].drmFormatModifier;
        break;
    }
}

/* Create image with an explicit list of modifiers — driver picks the best */
uint64_t modifiers[] = { chosenMod, DRM_FORMAT_MOD_LINEAR };
VkImageDrmFormatModifierListCreateInfoEXT modListCI = {
    .sType                  = VK_STRUCTURE_TYPE_IMAGE_DRM_FORMAT_MODIFIER_LIST_CREATE_INFO_EXT,
    .drmFormatModifierCount = 2,
    .pDrmFormatModifiers    = modifiers,
};

VkImageCreateInfo imageCI = {
    .sType         = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO,
    .pNext         = &modListCI,
    .imageType     = VK_IMAGE_TYPE_2D,
    .format        = VK_FORMAT_B8G8R8A8_UNORM,
    .extent        = { width, height, 1 },
    .mipLevels     = 1,
    .arrayLayers   = 1,
    .samples       = VK_SAMPLE_COUNT_1_BIT,
    /* tiling MUST be VK_IMAGE_TILING_DRM_FORMAT_MODIFIER_EXT */
    .tiling        = VK_IMAGE_TILING_DRM_FORMAT_MODIFIER_EXT,
    .usage         = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | VK_IMAGE_USAGE_SAMPLED_BIT,
    .sharingMode   = VK_SHARING_MODE_EXCLUSIVE,
    .initialLayout = VK_IMAGE_LAYOUT_UNDEFINED,
};
VkImage image;
vkCreateImage(device, &imageCI, NULL, &image);

/* Alternatively: pin an exact modifier (e.g. when importing a GBM-allocated buffer) */
VkSubresourceLayout planeLayout = { .offset = 0, .size = 0, .rowPitch = stride };
VkImageDrmFormatModifierExplicitCreateInfoEXT explicitCI = {
    .sType                = VK_STRUCTURE_TYPE_IMAGE_DRM_FORMAT_MODIFIER_EXPLICIT_CREATE_INFO_EXT,
    .drmFormatModifier    = actualModifierFromGbm,
    .drmFormatModifierPlaneCount = 1,
    .pPlaneLayouts        = &planeLayout,
};

/* Query the modifier the driver chose after creation */
PFN_vkGetImageDrmFormatModifierPropertiesEXT pfnGetModifier =
    (PFN_vkGetImageDrmFormatModifierPropertiesEXT)
    vkGetDeviceProcAddr(device, "vkGetImageDrmFormatModifierPropertiesEXT");

VkImageDrmFormatModifierPropertiesEXT imgModProps = {
    .sType = VK_STRUCTURE_TYPE_IMAGE_DRM_FORMAT_MODIFIER_PROPERTIES_EXT,
};
pfnGetModifier(device, image, &imgModProps);
uint64_t actualMod = imgModProps.drmFormatModifier;
```

**Multi-plane images.** For YCbCr formats (e.g. `VK_FORMAT_G8_B8R8_2PLANE_420_UNORM` for NV12), the subresource layout array must have one entry per plane. The GBM side exposes per-plane offsets via `gbm_bo_get_offset(bo, planeIndex)` and per-plane strides via `gbm_bo_get_stride_for_plane(bo, planeIndex)`. These must map exactly to the `pPlaneLayouts` array in `VkImageDrmFormatModifierExplicitCreateInfoEXT`. Multi-plane imports additionally require `VkBindImagePlaneMemoryInfo` at bind time — the single-plane `vkBindImageMemory` overload is insufficient.

**Mesa support.** Enabled in Mesa since approximately Mesa 19.1 (first landed in ANV; RADV followed in Mesa 19.3). Supported in ANV, RADV, NVK, Turnip, panvk, and v3dv as of Mesa 25.1. The per-driver modifier list comes from the driver's `wsi_device` descriptor; for RADV it reflects what `amdgpu_bo_import` accepts for direct scanout. [Mesa source](https://gitlab.freedesktop.org/mesa/mesa/-/blob/mesa-25.1.0/src/vulkan/wsi/wsi_common.c)

---

## 4. VK_KHR_external_memory_fd — File-Descriptor External Memory

**Problem it solves.** Vulkan's portable external memory model requires a platform mechanism to export allocated `VkDeviceMemory` out of the driver and import foreign allocations in. On Linux the mechanism is a file descriptor — either a dma-buf fd (cross-driver, cross-API, cross-process) or an opaque fd (driver-private, same driver different process). This extension defines the import/export calls; the handle type (`DMA_BUF` vs. `OPAQUE_FD`) selects the fd semantics. [Spec](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_external_memory_fd.html)

**Key structs and calls.**

```c
/* Export: get an fd from an existing VkDeviceMemory */
VkMemoryGetFdInfoKHR getFdInfo = {
    .sType      = VK_STRUCTURE_TYPE_MEMORY_GET_FD_INFO_KHR,
    .memory     = deviceMemory,
    .handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
    /* or VK_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_FD_BIT for same-driver sharing */
};
int exportedFd;
pfnGetMemoryFdKHR(device, &getFdInfo, &exportedFd);
/* exportedFd is now a dma-buf fd you can share with a V4L2 decoder, GBM, etc. */

/* Query what memory types the driver accepts for a foreign fd */
VkMemoryFdPropertiesKHR fdProps = {
    .sType = VK_STRUCTURE_TYPE_MEMORY_FD_PROPERTIES_KHR,
};
pfnGetMemoryFdPropertiesKHR(device,
                             VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
                             importedFd,
                             &fdProps);
/* fdProps.memoryTypeBits: bitmask of compatible memory types */

/* Import: allocate VkDeviceMemory backed by a foreign fd */
VkImportMemoryFdInfoKHR importInfo = {
    .sType      = VK_STRUCTURE_TYPE_IMPORT_MEMORY_FD_INFO_KHR,
    .handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
    .fd         = importedFd, /* dma-buf fd; Vulkan now owns the fd */
};

/* Find a compatible memory type */
VkPhysicalDeviceMemoryProperties memProps;
vkGetPhysicalDeviceMemoryProperties(physDev, &memProps);
uint32_t memTypeIdx = UINT32_MAX;
for (uint32_t i = 0; i < memProps.memoryTypeCount; i++) {
    if ((fdProps.memoryTypeBits & (1u << i)) &&
        (memProps.memoryTypes[i].propertyFlags & VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT)) {
        memTypeIdx = i;
        break;
    }
}

VkMemoryAllocateInfo allocInfo = {
    .sType           = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO,
    .pNext           = &importInfo,
    .allocationSize  = imageMemReqs.size,
    .memoryTypeIndex = memTypeIdx,
};
VkDeviceMemory importedMem;
vkAllocateMemory(device, &allocInfo, NULL, &importedMem);
```

**Important ownership rule.** After a successful `vkAllocateMemory` with `VkImportMemoryFdInfoKHR`, the Vulkan implementation takes ownership of the fd and the caller **must not** close it. After `vkGetMemoryFdKHR`, the caller owns the exported fd and is responsible for closing it (or passing ownership to another consumer). If `vkAllocateMemory` fails, the fd is still consumed — the driver closes it internally as part of the failure path on most Mesa implementations.

**Opaque vs. dma-buf fds.** `VK_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_FD_BIT` produces a `drm_prime` fd that only the same driver can import — it encodes driver-internal state and is not interoperable. It is useful for cross-process sharing of Vulkan memory within the same GPU driver (e.g. sharing a large texture atlas across Vulkan contexts in a browser's GPU process). `VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT` produces a true dma-buf fd, interoperable across drivers, APIs (VA-API, V4L2, GBM), and processes. Mixing the two types — e.g. exporting as opaque and importing as dma-buf — results in `VK_ERROR_INVALID_EXTERNAL_HANDLE`.

**Mesa support.** Supported in all Mesa Vulkan drivers since Mesa 18.x. Source: `src/vulkan/runtime/vk_device_memory.c` and per-driver backends (e.g. `src/amd/vulkan/radv_device_memory.c`).

---

## 5. VK_EXT_external_memory_dma_buf — dma-buf Handle Type

**Problem it solves.** `VK_KHR_external_memory_fd` defines the import/export mechanism but does not introduce the dma-buf handle type — that is done by `VK_EXT_external_memory_dma_buf`, which adds the enum value `VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT`. The two extensions always appear together in practice. Additionally, this extension documents the interaction with DRM format modifiers: a dma-buf fd carries an implicit modifier (the one the producing driver used), and `VK_EXT_image_drm_format_modifier` must be used to make the Vulkan image agree with that modifier. [Spec](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_external_memory_dma_buf.html)

```c
/* Typical allocation of an externally sharable image + memory */
VkExternalMemoryImageCreateInfo externalCI = {
    .sType       = VK_STRUCTURE_TYPE_EXTERNAL_MEMORY_IMAGE_CREATE_INFO,
    .handleTypes = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
};
/* chain into VkImageCreateInfo.pNext alongside VkImageDrmFormatModifierListCreateInfoEXT */

/* Dedicated allocation is required by most drivers for dma-buf exports */
VkMemoryDedicatedAllocateInfo dedicatedInfo = {
    .sType  = VK_STRUCTURE_TYPE_MEMORY_DEDICATED_ALLOCATE_INFO,
    .image  = image,
    .buffer = VK_NULL_HANDLE,
};
VkExportMemoryAllocateInfo exportInfo = {
    .sType       = VK_STRUCTURE_TYPE_EXPORT_MEMORY_ALLOCATE_INFO,
    .pNext       = &dedicatedInfo,
    .handleTypes = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
};
VkMemoryAllocateInfo memAI = {
    .sType           = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO,
    .pNext           = &exportInfo,
    .allocationSize  = memReqs.size,
    .memoryTypeIndex = memTypeIdx,
};
VkDeviceMemory exportableMem;
vkAllocateMemory(device, &memAI, NULL, &exportableMem);
vkBindImageMemory(device, image, exportableMem, 0);

/* Now export as dma-buf */
VkMemoryGetFdInfoKHR getFd = {
    .sType      = VK_STRUCTURE_TYPE_MEMORY_GET_FD_INFO_KHR,
    .memory     = exportableMem,
    .handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
};
int dmaBufFd;
pfnGetMemoryFdKHR(device, &getFd, &dmaBufFd);
/* dmaBufFd can now be passed to GBM, VA-API, V4L2, KMS, or another process */
```

**Mesa support.** Present in all Mesa Vulkan drivers that support `VK_KHR_external_memory_fd`. The capability bit is advertised by querying `VkPhysicalDeviceExternalImageFormatInfo` via `vkGetPhysicalDeviceImageFormatProperties2`.

---

## 6. VK_KHR_external_semaphore_fd — Sync-FD Semaphore Import/Export

**Problem it solves.** When a GPU workload needs to synchronise across API or process boundaries — for example, signalling a Wayland compositor that rendering is done, or ordering a VA-API decode before a Vulkan shader — both sides need to share a sync primitive. On Linux the currency is a `sync_file` fd (also called a fence fd), backed by a `dma_fence` in the kernel. `VK_KHR_external_semaphore_fd` adds the ability to export a `VkSemaphore` as a `SYNC_FD` (Linux `sync_file`) or as an `OPAQUE_FD` (`drm_syncobj` fd). [Spec](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_external_semaphore_fd.html)

**Handle types.**
- `VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_SYNC_FD_BIT` — a Linux `sync_file` fd; transient (one-shot signal). Used by the Wayland explicit sync protocol (`wp_linux_drm_syncobj_v1`) to pass acquire/release fences.
- `VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_OPAQUE_FD_BIT` — a `drm_syncobj` fd; persistent binary or timeline point. Useful for same-driver cross-process sharing.

```c
/* Create a semaphore that can be exported as a sync_file fd */
VkExportSemaphoreCreateInfo exportSemCI = {
    .sType       = VK_STRUCTURE_TYPE_EXPORT_SEMAPHORE_CREATE_INFO,
    .handleTypes = VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_SYNC_FD_BIT,
};
VkSemaphoreCreateInfo semCI = {
    .sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO,
    .pNext = &exportSemCI,
};
VkSemaphore renderDoneSem;
vkCreateSemaphore(device, &semCI, NULL, &renderDoneSem);

/* Submit GPU work; signal renderDoneSem when done */
VkSubmitInfo submit = {
    .sType                = VK_STRUCTURE_TYPE_SUBMIT_INFO,
    .signalSemaphoreCount = 1,
    .pSignalSemaphores    = &renderDoneSem,
    /* ... commandBuffers ... */
};
vkQueueSubmit(queue, 1, &submit, VK_NULL_HANDLE);

/* Export as sync_file fd (usable by KMS IN_FENCE_FD, V4L2, etc.) */
VkSemaphoreGetFdInfoKHR getSemFd = {
    .sType      = VK_STRUCTURE_TYPE_SEMAPHORE_GET_FD_INFO_KHR,
    .semaphore  = renderDoneSem,
    .handleType = VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_SYNC_FD_BIT,
};
int syncFileFd;
pfnGetSemaphoreFdKHR(device, &getSemFd, &syncFileFd);
/* syncFileFd represents completion of the GPU work; pass to compositor */

/* Import: receive a sync_file fd from another producer (e.g. compositor release) */
VkImportSemaphoreFdInfoKHR importSemFd = {
    .sType      = VK_STRUCTURE_TYPE_IMPORT_SEMAPHORE_FD_INFO_KHR,
    .semaphore  = waitSem,
    .flags      = VK_SEMAPHORE_IMPORT_TEMPORARY_BIT, /* temporary import for SYNC_FD */
    .handleType = VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_SYNC_FD_BIT,
    .fd         = incomingFd,
};
pfnImportSemaphoreFdKHR(device, &importSemFd);
/* waitSem is now armed; use it as a wait semaphore in the next vkQueueSubmit */
```

**Note on temporary imports.** `SYNC_FD` imports must use `VK_SEMAPHORE_IMPORT_TEMPORARY_BIT`. The semaphore reverts to its unsignalled state after the next wait.

**Timeline semaphores and OPAQUE_FD.** For timeline (`VkSemaphoreTypeTimeline`) semaphores, only `OPAQUE_FD` is supported — `SYNC_FD` is defined only for binary semaphores. A timeline semaphore exported as `OPAQUE_FD` is backed by a `drm_syncobj` with timeline capability (`DRM_SYNCOBJ_CREATE_SIGNALED` + point tracking). This is the mechanism used by `wp_linux_drm_syncobj_v1` to carry timeline points across the Wayland protocol: both compositor and client reference the same `drm_syncobj` fd, and the protocol message conveys which timeline point (a `uint64_t`) is the acquire or release boundary for a given frame. Appendix G §3.5 has the full `drm_syncobj` timeline state machine.

**Mesa support.** Supported in all Mesa Vulkan drivers since Mesa 18.x. Mesa's WSI Wayland layer uses this internally to produce the release fence passed back to the compositor. Source: `src/vulkan/wsi/wsi_common_wayland.c` and `src/vulkan/runtime/vk_semaphore.c`.

---

## 7. VK_KHR_external_fence_fd — Fence File Descriptor Export

**Problem it solves.** `VkFence` is a CPU-side synchronisation object — it lets the application call `vkWaitForFences` to block until GPU work is done. `VK_KHR_external_fence_fd` extends fences with the ability to export a `SYNC_FD` or `OPAQUE_FD` that external consumers (DRM KMS, V4L2 capture, VA-API) can wait on without involving Vulkan. This is critical for the direct scanout path: KMS can be given a `sync_file` fd on the `IN_FENCE_FD` plane property and will only scan out the buffer after the GPU signals that fence. [Spec](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_external_fence_fd.html)

```c
/* Create an exportable fence */
VkExportFenceCreateInfo exportFenceCI = {
    .sType       = VK_STRUCTURE_TYPE_EXPORT_FENCE_CREATE_INFO,
    .handleTypes = VK_EXTERNAL_FENCE_HANDLE_TYPE_SYNC_FD_BIT,
};
VkFenceCreateInfo fenceCI = {
    .sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO,
    .pNext = &exportFenceCI,
};
VkFence renderFence;
vkCreateFence(device, &fenceCI, NULL, &renderFence);

/* Submit work that signals this fence on completion */
vkQueueSubmit(queue, 1, &submitInfo, renderFence);

/* Export as sync_file for KMS or other kernel consumers */
VkFenceGetFdInfoKHR getFenceInfo = {
    .sType      = VK_STRUCTURE_TYPE_FENCE_GET_FD_INFO_KHR,
    .fence      = renderFence,
    .handleType = VK_EXTERNAL_FENCE_HANDLE_TYPE_SYNC_FD_BIT,
};
int fenceFd;
pfnGetFenceFdKHR(device, &getFenceInfo, &fenceFd);

/*
 * fenceFd is now a Linux sync_file fd.
 * Pass it to KMS via drmModeAtomicAddProp(..., "IN_FENCE_FD", fenceFd)
 * KMS will wait for GPU completion before scanning out the framebuffer.
 */

/* Import a fence fd from another source (e.g. OUT_FENCE_PTR from KMS) */
VkImportFenceFdInfoKHR importFenceInfo = {
    .sType      = VK_STRUCTURE_TYPE_IMPORT_FENCE_FD_INFO_KHR,
    .fence      = waitFence,
    .flags      = VK_FENCE_IMPORT_TEMPORARY_BIT,
    .handleType = VK_EXTERNAL_FENCE_HANDLE_TYPE_SYNC_FD_BIT,
    .fd         = outFenceFd,
};
pfnImportFenceFdKHR(device, &importFenceInfo);
/* vkWaitForFences(device, 1, &waitFence, ...) will now wait for KMS flip completion */
```

**Difference from semaphore FDs.** `VK_KHR_external_fence_fd` and `VK_KHR_external_semaphore_fd` serve complementary roles. A `VkFence` is CPU-visible: `vkWaitForFences` blocks the calling thread until the fence signals, making it suitable for GPU→CPU synchronisation. A `VkSemaphore` is GPU-visible: it is consumed as a dependency in another `vkQueueSubmit`, enabling GPU→GPU ordering without a CPU stall. When you export either as `SYNC_FD`, both become a Linux `sync_file` fd that KMS or another driver can wait on. The practical difference for direct display is that exportable fences are better suited to "wait until frame is fully rendered" queries (CPU-side), while exportable semaphores are better suited to "signal the KMS flip engine once rendering is done" (GPU-to-kernel).

**Mesa support.** Supported in all Mesa Vulkan drivers since approximately Mesa 18.3. Source: `src/vulkan/runtime/vk_fence.c`. The fence FD mechanism is also used internally by Mesa's WSI layer to implement `vkAcquireNextImageKHR` on Wayland (the acquire semaphore/fence is backed by the compositor's release `sync_file`).

---

## 8. VK_EXT_acquire_drm_display — Direct KMS Display Ownership

**Problem it solves.** A Vulkan application that wants to drive a display directly — without a compositor — needs to acquire exclusive ownership of a KMS display (a `VkDisplayKHR` object) after getting its DRM fd. `VK_KHR_display` can enumerate displays but cannot claim ownership. `VK_EXT_acquire_drm_display` bridges from a DRM connector (identified by `drmModeConnector.connector_id`) to a `VkDisplayKHR`, then requests exclusive access. This is used by VR runtimes (Monado in direct mode), headset compositors, and kiosk applications. [Spec](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_acquire_drm_display.html)

```c
PFN_vkAcquireDrmDisplayEXT pfnAcquireDrm =
    (PFN_vkAcquireDrmDisplayEXT)
    vkGetInstanceProcAddr(instance, "vkAcquireDrmDisplayEXT");
PFN_vkGetDrmDisplayEXT pfnGetDrmDisplay =
    (PFN_vkGetDrmDisplayEXT)
    vkGetInstanceProcAddr(instance, "vkGetDrmDisplayEXT");

/* Open the DRM device for the physical device */
int drmFd = open("/dev/dri/card0", O_RDWR);

/* Map a DRM connector_id to a VkDisplayKHR */
uint32_t drmConnectorId = ...; /* from drmModeGetResources / drmModeGetConnector */
VkDisplayKHR display;
VkResult r = pfnGetDrmDisplay(physDev, drmFd, drmConnectorId, &display);
assert(r == VK_SUCCESS);

/* Acquire exclusive ownership — will fail if a compositor already holds the display */
r = pfnAcquireDrm(physDev, drmFd, display);
if (r != VK_SUCCESS) {
    fprintf(stderr, "Could not acquire display: is a compositor running?\n");
}

/* After acquisition, use VK_KHR_display to create a surface (see §9) */
```

**Relationship to VK_EXT_direct_mode_display.** `VK_EXT_acquire_drm_display` supersedes the older `VK_EXT_acquire_xlib_display` and `VK_EXT_direct_mode_display` on Linux; those extensions used X11 RandR for the same purpose. On a pure Wayland or compositor-free system, `VK_EXT_acquire_drm_display` is the only supported path. The transition from X11 display acquisition to DRM display acquisition removed the X11 server dependency from VR runtimes — Monado in particular moved to this path to support headsets on purely Wayland desktops without XWayland. [Monado VK_EXT_acquire_drm_display integration](https://gitlab.freedesktop.org/monado/monado/-/blob/main/src/xrt/compositor/main/comp_target_swapchain.c)

**Mesa support.** Supported since Mesa 21.1 (landed as part of the Direct Display / VR direct mode work). Requires that the process hold the DRM master role or that the DRM device supports `DRM_AUTH` without master. [Note: verify exact minimum Mesa version against Mesa changelog.]

---

## 9. VK_KHR_display and VK_KHR_display_swapchain — Direct Scanout

**Problem it solves.** These two companion extensions implement the full "display owns the output" path independent of any compositor. `VK_KHR_display` enumerates physical displays and their supported modes as Vulkan objects, and lets the application create a `VkSurfaceKHR` directly over a display plane. `VK_KHR_display_swapchain` adds `vkCreateSharedSwapchainsKHR` for creating multiple swapchains (e.g. two eyes in stereo) that share the same display. [VK_KHR_display spec](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_display.html) | [VK_KHR_display_swapchain spec](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_display_swapchain.html)

```c
/* Enumerate displays on the physical device */
uint32_t displayCount;
vkGetPhysicalDeviceDisplayPropertiesKHR(physDev, &displayCount, NULL);
VkDisplayPropertiesKHR *displays = malloc(displayCount * sizeof(*displays));
vkGetPhysicalDeviceDisplayPropertiesKHR(physDev, &displayCount, displays);

for (uint32_t i = 0; i < displayCount; i++) {
    printf("Display %u: %s (%ux%u mm)\n",
           i, displays[i].displayName,
           displays[i].physicalDimensions.width,
           displays[i].physicalDimensions.height);
}

/* Enumerate modes for the first display */
VkDisplayKHR chosenDisplay = displays[0].display;
uint32_t modeCount;
vkGetDisplayModePropertiesKHR(physDev, chosenDisplay, &modeCount, NULL);
VkDisplayModePropertiesKHR *modes = malloc(modeCount * sizeof(*modes));
vkGetDisplayModePropertiesKHR(physDev, chosenDisplay, &modeCount, modes);

/* Select the first mode (e.g. native resolution at highest refresh) */
VkDisplayModeKHR mode = modes[0].displayMode;

/* Query display planes */
uint32_t planeCount;
vkGetPhysicalDeviceDisplayPlanePropertiesKHR(physDev, &planeCount, NULL);

/* Create a display plane surface */
VkDisplaySurfaceCreateInfoKHR displaySurfCI = {
    .sType           = VK_STRUCTURE_TYPE_DISPLAY_SURFACE_CREATE_INFO_KHR,
    .displayMode     = mode,
    .planeIndex      = 0,
    .planeStackIndex = 0,
    .transform       = VK_SURFACE_TRANSFORM_IDENTITY_BIT_KHR,
    .globalAlpha     = 1.0f,
    .alphaMode       = VK_DISPLAY_PLANE_ALPHA_OPAQUE_BIT_KHR,
    .imageExtent     = modes[0].parameters.visibleRegion,
};
VkSurfaceKHR displaySurface;
vkCreateDisplayPlaneSurfaceKHR(instance, &displaySurfCI, NULL, &displaySurface);

/* Shared swapchains (stereo / multi-view) */
VkSwapchainCreateInfoKHR scInfoLeft = { /* ... surface, imageExtent, etc. ... */ };
VkSwapchainCreateInfoKHR scInfoRight = { /* ... same display, different offset ... */ };

PFN_vkCreateSharedSwapchainsKHR pfnCreateShared =
    (PFN_vkCreateSharedSwapchainsKHR)
    vkGetDeviceProcAddr(device, "vkCreateSharedSwapchainsKHR");
VkSwapchainKHR sharedChains[2];
pfnCreateShared(device, 2,
                (VkSwapchainCreateInfoKHR[]){ scInfoLeft, scInfoRight },
                NULL, sharedChains);
```

**Typical use pattern.** VR runtimes (Monado, SteamVR) use `VK_KHR_display` + `VK_EXT_acquire_drm_display` together: first acquire the HMD's DRM connector (§8), then create a display surface over it with `vkCreateDisplayPlaneSurfaceKHR`, and finally present both eye views via shared swapchains.

**Plane stack ordering.** `VkDisplaySurfaceCreateInfoKHR.planeStackIndex` controls which hardware overlay plane the swapchain uses. On multi-plane KMS hardware (common on Intel and AMD), lower stack indices are higher on screen. Most direct-display applications use plane 0 (the primary plane). The `VK_DISPLAY_PLANE_ALPHA_*` enum controls blend mode: `OPAQUE` disables per-pixel alpha, while `GLOBAL` applies `globalAlpha` uniformly, and `PER_PIXEL` enables alpha blending with the plane beneath.

**Mesa support.** `VK_KHR_display` has been supported in Mesa's WSI layer since approximately Mesa 21.x. Note: display enumeration on most Linux systems requires DRM master privileges; unprivileged applications typically fall through to the Wayland path instead. [Note: needs verification for exact Mesa version support per driver.]

---

## 10. VK_EXT_headless_surface — Offscreen and Headless Rendering

**Problem it solves.** CI/CD pipelines, GPU render servers, cloud rendering workers, and conformance test suites need to exercise the full Vulkan API surface — including swapchains, present queues, and frame timing — without an actual display or compositor. `VK_EXT_headless_surface` creates a `VkSurfaceKHR` that is valid for all surface capability queries and swapchain creation, but discards frames at present time rather than outputting them. This allows applications to use the swapchain API unmodified and simplifies testing. [Spec](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_headless_surface.html)

```c
PFN_vkCreateHeadlessSurfaceEXT pfnCreateHeadless =
    (PFN_vkCreateHeadlessSurfaceEXT)
    vkGetInstanceProcAddr(instance, "vkCreateHeadlessSurfaceEXT");

VkHeadlessSurfaceCreateInfoEXT headlessCI = {
    .sType = VK_STRUCTURE_TYPE_HEADLESS_SURFACE_CREATE_INFO_EXT,
    .pNext = NULL,
    .flags = 0,
};
VkSurfaceKHR headlessSurface;
pfnCreateHeadless(instance, &headlessCI, NULL, &headlessSurface);

/* From here the API is identical to any other VkSurfaceKHR path */
VkSwapchainCreateInfoKHR scCI = {
    .sType           = VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR,
    .surface         = headlessSurface,
    .minImageCount   = 2,
    .imageFormat     = VK_FORMAT_B8G8R8A8_UNORM,
    .imageColorSpace = VK_COLOR_SPACE_SRGB_NONLINEAR_KHR,
    .imageExtent     = { 1920, 1080 },
    .imageArrayLayers = 1,
    .imageUsage      = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT,
    .presentMode     = VK_PRESENT_MODE_FIFO_KHR,
    .clipped         = VK_FALSE,
    .oldSwapchain    = VK_NULL_HANDLE,
};
VkSwapchainKHR sc;
vkCreateSwapchainKHR(device, &scCI, NULL, &sc);
/* Render normally, vkQueuePresentKHR succeeds and discards the frame */
```

**Typical use pattern.** The Vulkan CTS (`dEQP-VK`) can use a headless surface when `$DISPLAY` and `$WAYLAND_DISPLAY` are both unset. Mesa's CI pipelines (GitLab CI with GPU-less runners using `lavapipe`) combine `VK_EXT_headless_surface` with the software renderer for regression testing without hardware. Chapter 107 (Headless Rendering) covers the broader headless-rendering workflow.

**Mesa support.** Present in all Mesa Vulkan drivers since Mesa 20.0 (when it was added to the common WSI layer). Also supported by Vulkan validation layers and by `lvp` (lavapipe).

---

## 11. End-to-End: Zero-Copy dma-buf Import Flow

This section demonstrates how the extensions chain together in the most common real-world scenario: importing a GBM-allocated buffer (e.g. from a Wayland client's render target) into Vulkan for post-processing or compositing, with no intermediate GPU copy.

```c
#include <gbm.h>
#include <xf86drm.h>
#include <drm_fourcc.h>
#include <vulkan/vulkan.h>

/*
 * Assumed: device, physDev, queue, commandPool, and all extension function
 * pointers (pfnGet*, pfnImport*) are already initialised.
 * The pfn* variables from §1 are in scope.
 */

/* ---- Step 1: Allocate a GBM buffer with a known modifier ---- */
struct gbm_device *gbmDev = gbm_create_device(drmFd);
struct gbm_bo *bo = gbm_bo_create_with_modifiers2(
    gbmDev, 1920, 1080, GBM_FORMAT_ARGB8888,
    (uint64_t[]){ AMD_FMT_MOD_TILE_GFX9_64K_S, DRM_FORMAT_MOD_LINEAR }, 2, /* illustrative; see App H §3 for real AMD modifier encoding */
    GBM_BO_USE_RENDERING | GBM_BO_USE_SCANOUT);

int   dma_buf_fd  = gbm_bo_get_fd(bo);
int   stride      = gbm_bo_get_stride(bo);
uint64_t modifier = gbm_bo_get_modifier(bo);

/* ---- Step 2: Create the matching VkImage ---- */
VkSubresourceLayout planeLayout = {
    .offset   = gbm_bo_get_offset(bo, 0),
    .rowPitch = stride,
};
VkImageDrmFormatModifierExplicitCreateInfoEXT modCI = {
    .sType                       = VK_STRUCTURE_TYPE_IMAGE_DRM_FORMAT_MODIFIER_EXPLICIT_CREATE_INFO_EXT,
    .drmFormatModifier           = modifier,
    .drmFormatModifierPlaneCount = 1,
    .pPlaneLayouts               = &planeLayout,
};
VkExternalMemoryImageCreateInfo externalImgCI = {
    .sType       = VK_STRUCTURE_TYPE_EXTERNAL_MEMORY_IMAGE_CREATE_INFO,
    .pNext       = &modCI,
    .handleTypes = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
};
VkImageCreateInfo imageCI = {
    .sType         = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO,
    .pNext         = &externalImgCI,
    .imageType     = VK_IMAGE_TYPE_2D,
    .format        = VK_FORMAT_B8G8R8A8_UNORM, /* matches GBM_FORMAT_ARGB8888 */
    .extent        = { 1920, 1080, 1 },
    .mipLevels     = 1,
    .arrayLayers   = 1,
    .samples       = VK_SAMPLE_COUNT_1_BIT,
    .tiling        = VK_IMAGE_TILING_DRM_FORMAT_MODIFIER_EXT,
    .usage         = VK_IMAGE_USAGE_SAMPLED_BIT,
    .initialLayout = VK_IMAGE_LAYOUT_UNDEFINED,
};
VkImage importedImage;
vkCreateImage(device, &imageCI, NULL, &importedImage);

/* ---- Step 3: Query memory requirements for the imported image ---- */
VkMemoryFdPropertiesKHR fdProps = {
    .sType = VK_STRUCTURE_TYPE_MEMORY_FD_PROPERTIES_KHR,
};
pfnGetMemoryFdPropertiesKHR(device,
                             VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
                             dma_buf_fd, &fdProps);

VkMemoryRequirements memReqs;
vkGetImageMemoryRequirements(device, importedImage, &memReqs);

/* Find a device-local type compatible with the dma-buf */
uint32_t compatBits = fdProps.memoryTypeBits & memReqs.memoryTypeBits;
uint32_t memTypeIdx = __builtin_ctz(compatBits); /* first compatible type */

/* ---- Step 4: Import dma-buf as VkDeviceMemory (dedicated allocation) ---- */
VkMemoryDedicatedAllocateInfo dedicatedAI = {
    .sType  = VK_STRUCTURE_TYPE_MEMORY_DEDICATED_ALLOCATE_INFO,
    .image  = importedImage,
};
VkImportMemoryFdInfoKHR importFdInfo = {
    .sType      = VK_STRUCTURE_TYPE_IMPORT_MEMORY_FD_INFO_KHR,
    .pNext      = &dedicatedAI,
    .handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
    .fd         = dma_buf_fd, /* Vulkan takes ownership; do not close */
};
VkMemoryAllocateInfo memAI = {
    .sType           = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO,
    .pNext           = &importFdInfo,
    .allocationSize  = memReqs.size,
    .memoryTypeIndex = memTypeIdx,
};
VkDeviceMemory importedMem;
vkAllocateMemory(device, &memAI, NULL, &importedMem);
vkBindImageMemory(device, importedImage, importedMem, 0);

/* ---- Step 5: Transition layout and use in a shader ---- */
/*
 * The image is now ready for use as VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL
 * via a pipeline barrier. No copy occurred; the Vulkan driver maps directly
 * onto the dma-buf's physical pages.
 */
```

**Key correctness points:**
1. The `VkImage` must use `VK_IMAGE_TILING_DRM_FORMAT_MODIFIER_EXT` and pin the exact modifier via `VkImageDrmFormatModifierExplicitCreateInfoEXT`.
2. A dedicated allocation (`VkMemoryDedicatedAllocateInfo`) is required by most drivers when importing a dma-buf into an image.
3. After `vkAllocateMemory` with `VkImportMemoryFdInfoKHR`, the driver owns the fd.
4. The image starts in `VK_IMAGE_LAYOUT_UNDEFINED`; an image memory barrier must transition it before the first shader access.

---

## 12. Compositor Extension Chain

A Wayland compositor using Vulkan for rendering and composition connects all the extensions above into a single pipeline per frame. The following describes the standard chain used by wlroots-based compositors (see Ch21) and Mutter/KWin variants:

```
Client renders with Vulkan or GL:
  └─> produces a dma-buf via linux-dmabuf / zwp_linux_dmabuf_feedback_v1
  └─> attaches a sync_file (acquire fence) via wp_linux_drm_syncobj_v1

Compositor receives wl_surface commit:
  ├─ 1. Reads acquire fence fd from Wayland surface
  │    └─ VkImportSemaphoreFdInfoKHR (SYNC_FD) → compositor waits on it
  ├─ 2. Imports client dma-buf as VkImage
  │    ├─ VkImageDrmFormatModifierExplicitCreateInfoEXT (from negotiated modifier)
  │    ├─ VK_EXT_external_memory_dma_buf + VK_KHR_external_memory_fd → import fd
  │    └─ VkImage with VK_IMAGE_TILING_DRM_FORMAT_MODIFIER_EXT
  ├─ 3. Compositor renders scene (sampling client texture) → output VkImage
  ├─ 4. Compositor exports release fence for client
  │    └─ VkGetSemaphoreFdKHR(SYNC_FD) → release sync_file → wp_linux_drm_syncobj_v1
  └─ 5. Output to display
       ├─ Wayland path: present via VK_KHR_wayland_surface swapchain
       │   (Mesa WSI submits to KMS via drmModeAtomicCommit internally)
       └─ Direct KMS path: VK_EXT_acquire_drm_display + VK_KHR_display
            └─ Fence for KMS: VkGetFenceFdKHR(SYNC_FD) → drmModeAtomicAddProp IN_FENCE_FD
```

The modifier is negotiated at surface creation time via `zwp_linux_dmabuf_feedback_v1`: the compositor advertises a `VkFormat`+modifier combination for each scanout target, and the client allocates with exactly that modifier. This is how zero-copy direct scanout is achieved without any format-conversion blit.

**Image layout transitions in the compositor.** After importing a client dma-buf as a `VkImage`, the compositor must perform an image memory barrier to transition from `VK_IMAGE_LAYOUT_UNDEFINED` to `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL` before sampling it. Because the image was produced by another queue (or even another driver), the barrier must include a pipeline stage mask of `VK_PIPELINE_STAGE_ALL_COMMANDS_BIT` and an access mask of `VK_ACCESS_MEMORY_READ_BIT | VK_ACCESS_MEMORY_WRITE_BIT` on the source side, unless the acquire fence from `wp_linux_drm_syncobj_v1` already guarantees memory visibility — in which case the barrier can use `VK_PIPELINE_STAGE_NONE` and `VK_ACCESS_NONE` as the source.

**Direct scanout path.** For a compositor that itself writes to a display via `VK_KHR_wayland_surface`, the final step in §12 is transparent — Mesa WSI delivers the swapchain image to KMS internally. For a compositor that controls KMS directly (such as a VR runtime using `VK_KHR_display`), the explicit fence export at step 5 is mandatory: the KMS `IN_FENCE_FD` property must receive the render-completion `sync_file` before `drmModeAtomicCommit` is called, or the display hardware may scan out an incomplete frame.

---

## 13. Limitations and Gotchas

**Format modifier negotiation failures.** If the client allocates with a modifier the compositor's Vulkan driver does not support for the given format (`VkDrmFormatModifierPropertiesEXT.drmFormatModifierTilingFeatures` does not include the required usage), the compositor must fall back to a CPU-side copy or a shader blit. The `zwp_linux_dmabuf_feedback_v1` protocol was designed to avoid this: clients ask for the feedback table first and only then allocate. Applications that skip the feedback protocol and guess a modifier will hit this silently — the compositor fallback is invisible to the client.

**Plane count for multi-planar formats.** YCbCr formats (e.g. `VK_FORMAT_G8_B8_R8_3PLANE_420_UNORM`) require multiple dma-buf fds or a multi-planar dma-buf. `VkImageDrmFormatModifierExplicitCreateInfoEXT.drmFormatModifierPlaneCount` must match the actual plane count, and `VkBindImagePlaneMemoryInfo` must bind each plane separately. Many compositors restrict themselves to single-planar ARGB/XRGB buffers from clients and handle YUV conversion internally.

**GPU vendor support differences.** `VK_EXT_acquire_drm_display` requires DRM master. On desktop Linux, the logged-in user's session holds DRM master via `logind`; a background process attempting to take master will fail with `EPERM`. NVIDIA's proprietary Vulkan driver supports most of these extensions as of driver 525+, but the dma-buf import path for `VK_EXT_external_memory_dma_buf` on NVIDIA requires that the buffer was allocated with `CUDA_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_FD` or with GBM + `EGL_LINUX_DMA_BUF_EXT` — NVIDIA's memory manager has additional constraints. See Ch5 (x86 GPU Drivers) for NVIDIA-specific considerations.

**Implicit vs. explicit synchronisation.** Older wl_buffer-based clients (X11 via XWayland, or Wayland clients that do not support `wp_linux_drm_syncobj_v1`) do not pass explicit fences. The compositor must insert a CPU stall or a `dma_fence` wait to ensure the buffer is safe to sample. Mesa's WSI layer handles this for `VK_KHR_wayland_surface` swapchains automatically, but compositors receiving arbitrary dma-bufs from `zwp_linux_dmabuf_v1` (the older protocol, without explicit sync) must issue an implicit sync wait before importing. Appendix G covers the implicit/explicit sync transition in detail.

**Sync FD transience.** A semaphore imported with `VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_SYNC_FD_BIT | VK_SEMAPHORE_IMPORT_TEMPORARY_BIT` reverts to its unsignalled state after being waited. If you need persistent signalling across multiple submits, use `OPAQUE_FD` (backed by a `drm_syncobj`) instead. Mixing the two types on the same semaphore without re-importing each frame is a common bug.

**Dedicated allocation requirement.** Most Mesa Vulkan drivers require `VkMemoryDedicatedRequirements.requiresDedicatedAllocation == VK_TRUE` for dma-buf imported images. Failing to provide a dedicated allocation causes `VK_ERROR_OUT_OF_DEVICE_MEMORY` or undefined behaviour on some drivers. Always query `VkMemoryDedicatedRequirementsKHR` via `vkGetImageMemoryRequirements2` before allocating.

**NV12/P010 dma-buf import.** Importing video decoder output (VA-API YCbCr frames) into Vulkan as `VK_FORMAT_G8_B8R8_2PLANE_420_UNORM` with a vendor tiling modifier is supported on Intel (ANV) and AMD (RADV) but has historically had modifier coverage gaps on ARM Mali (panvk) and NVIDIA (NVK). Always check `vkGetPhysicalDeviceFormatProperties2` with the specific modifier before committing to the path. [Note: needs verification for NVK/panvk status as of Mesa 25.1.]

**Present mode constraints on headless surfaces.** `VK_EXT_headless_surface` supports all present modes (`FIFO`, `MAILBOX`, `IMMEDIATE`), but does not guarantee any particular frame timing. Applications that use `VK_PRESENT_MODE_FIFO_KHR` on a headless surface will not block waiting for a vsync signal — `vkQueuePresentKHR` returns immediately. This makes headless surfaces inappropriate as a drop-in replacement for a real display surface in latency benchmarks, but correct for functional testing.

**Cross-vendor zero-copy.** The dma-buf/format-modifier pipeline is theoretically cross-vendor, but in practice the sharing matrix has gaps. AMD→Intel zero-copy (e.g. RADV produces a buffer with `AMD_FMT_MOD_*` that ANV imports) does not work because Intel's display engine has no knowledge of AMD tile patterns. The only universally interoperable modifier is `DRM_FORMAT_MOD_LINEAR`, which trades performance for portability. For cross-vendor sharing, always negotiate with `DRM_FORMAT_MOD_LINEAR` as the fallback and verify that `VkDrmFormatModifierPropertiesEXT.drmFormatModifierTilingFeatures` includes the required usage bits on the importing driver as well as the exporting driver. Appendix H §8 (Cross-Driver DMA-BUF Sharing Compatibility Matrix) enumerates the specific modifier combinations known to work across driver pairs.

**Fence fd leak on present failure.** If `vkQueuePresentKHR` returns an error (e.g. `VK_ERROR_SURFACE_LOST_KHR`), and the application had attached a wait semaphore exported as `SYNC_FD` to the present operation, the `sync_file` fd has already been consumed — do not attempt to close it separately. By contrast, if the application had obtained the fd independently (e.g. via `pfnGetSemaphoreFdKHR`) and was going to pass it to KMS, a present failure means the KMS commit did not happen and the fd still belongs to the application and must be closed.

---

## 14. Extension Support Matrix

The table below summarises which Mesa Vulkan drivers advertise each extension as of Mesa 25.1. "All" means ANV (Intel), RADV (AMD), NVK (NVIDIA/Nouveau), Turnip (Qualcomm Adreno), panvk (ARM Mali), and v3dv (Broadcom VideoCore). Where driver support differs the individual drivers are listed. Mesa version in parentheses is the earliest release where the extension was enabled in that driver.

**Note: the version numbers and Partial/No entries in this table are approximate.** They are derived from Mesa release notes, mailing-list threads, and per-driver `vk_extensions` tables available at time of writing, but Mesa extension enablement changes frequently. For authoritative per-driver status, check `src/gallium/drivers/<driver>/` or `src/<vendor>/vulkan/` in the Mesa source tree at the tag for your release, or use `vulkaninfo --json` and cross-reference against [vulkan.gpuinfo.org](https://vulkan.gpuinfo.org/).

| Extension | ANV | RADV | NVK | Turnip | panvk | v3dv | Minimum Mesa |
|---|---|---|---|---|---|---|---|
| `VK_KHR_wayland_surface` | Yes | Yes | Yes | Yes | Yes | Yes | 17.x (all) |
| `VK_EXT_image_drm_format_modifier` | Yes (19.1) | Yes (19.3) | Yes (23.x) | Yes | Yes | Yes | 19.1 |
| `VK_KHR_external_memory_fd` | Yes | Yes | Yes | Yes | Yes | Yes | 18.x |
| `VK_EXT_external_memory_dma_buf` | Yes | Yes | Yes | Yes | Yes | Yes | 18.x |
| `VK_KHR_external_semaphore_fd` | Yes | Yes | Yes | Yes | Yes | Yes | 18.x |
| `VK_KHR_external_fence_fd` | Yes | Yes | Yes | Yes | Yes | Yes | 18.3 |
| `VK_EXT_acquire_drm_display` | Yes | Yes | Yes | Partial | No | No | 21.1 |
| `VK_KHR_display` | Yes | Yes | Yes | Partial | No | No | 21.x |
| `VK_KHR_display_swapchain` | Yes | Yes | Yes | Partial | No | No | 21.x |
| `VK_EXT_headless_surface` | Yes | Yes | Yes | Yes | Yes | Yes | 20.0 |

Notes:
- "Partial" for Turnip means the extension is advertised but the display enumeration path may not enumerate real hardware on all Qualcomm SoC variants.
- panvk and v3dv do not implement `VK_KHR_display` because ARM Mali and Broadcom VideoCore display controllers are not enumerated via the same `drm_mode_connector` interface that the Mesa display backend assumes. [Note: needs verification for Mesa 25.1 panvk status.]
- NVK (Nova/Nouveau Vulkan) added `VK_EXT_image_drm_format_modifier` support beginning in Mesa 23.x as part of the broader NVK external memory work; modifier support for tiled layouts is constrained to what the `nouveau` kernel driver can import via `nouveau_bo_prime`.
- For NVIDIA's proprietary Vulkan driver (not Mesa), all these extensions are supported as of driver version 525.85.12+, with additional constraints documented in the NVIDIA Vulkan extension specification notes.

---

## 15. Mesa Source Map

| Extension | Primary Mesa source path |
|---|---|
| `VK_KHR_wayland_surface` | `src/vulkan/wsi/wsi_common_wayland.c` |
| `VK_EXT_image_drm_format_modifier` | `src/vulkan/wsi/wsi_common.c`, per-driver `vk_image.c` |
| `VK_KHR_external_memory_fd` | `src/vulkan/runtime/vk_device_memory.c` |
| `VK_EXT_external_memory_dma_buf` | same as above; handle type enum only |
| `VK_KHR_external_semaphore_fd` | `src/vulkan/runtime/vk_semaphore.c` |
| `VK_KHR_external_fence_fd` | `src/vulkan/runtime/vk_fence.c` |
| `VK_EXT_acquire_drm_display` | `src/vulkan/wsi/wsi_common_display.c` |
| `VK_KHR_display` | `src/vulkan/wsi/wsi_common_display.c` |
| `VK_KHR_display_swapchain` | `src/vulkan/wsi/wsi_common.c` |
| `VK_EXT_headless_surface` | `src/vulkan/wsi/wsi_common_headless.c` |

All paths relative to the Mesa repository root at [Mesa 25.1](https://gitlab.freedesktop.org/mesa/mesa/-/tree/mesa-25.1.0/src/vulkan). Per-driver implementations (RADV, ANV, NVK, etc.) typically call through to these common helpers; vendor-specific overrides appear in `src/amd/vulkan/`, `src/intel/vulkan/`, and `src/nouveau/vulkan/`.

---

## 16. Integrations

- **Ch1 (DRM Architecture & the Driver Model)** — DRM render nodes, `drmModeAtomicCommit`, `IN_FENCE_FD`/`OUT_FENCE_PTR` plane properties; the kernel side of everything Vulkan fences and dma-bufs sit on top of.
- **Ch2 (KMS: The Display Pipeline)** — KMS atomic commit with fences; how `VK_KHR_display` and `VK_EXT_acquire_drm_display` map onto DRM CRTC/plane/connector objects.
- **Ch4 (GPU Memory Management)** — GBM buffer allocation with modifiers; dma-buf lifecycle; `drm_syncobj` kernel objects that back `VkSemaphore` and `VkFence`.
- **Ch20 (Wayland Protocol Fundamentals)** — `zwp_linux_dmabuf_v1`, `zwp_linux_dmabuf_feedback_v1`, and `wp_linux_drm_syncobj_v1`; the Wayland side of the buffer-sharing and sync protocol chain that Vulkan WSI implements.
- **Ch21 (Building Compositors with wlroots)** — How wlroots-based compositors consume dma-buf surfaces from Vulkan clients, negotiate modifiers, and drive KMS; the compositor view of the §12 extension chain.
- **Ch24 (Vulkan and EGL for Application Developers)** — The application-level Vulkan API; swapchain creation, queue submission, and the timeline semaphore model that timeline `VkSemaphore` extends.
- **Ch75 (Explicit GPU Synchronisation)** — `wp_linux_drm_syncobj_v1` protocol design; the full explicit sync stack from `dma_fence` through `drm_syncobj` to Vulkan semaphore fd; why NVIDIA's implicit sync absence forced the ecosystem to adopt this path.
- **Ch150 (EGL Architecture and DMA-BUF Integration)** — EGL's parallel dma-buf import path (`EGL_LINUX_DMA_BUF_EXT`, `EGLImageKHR`); how ANGLE (Ch34) and VA-API clients reach the same dma-buf objects by a different API route.
- **Appendix G (Synchronisation Primitives Reference)** — Comprehensive state machine diagrams for `sync_file`, `drm_syncobj`, `VkFence`, and `VkSemaphore`; conversion paths between Vulkan sync objects and kernel primitives.
- **Appendix H (DRM Format Modifiers Reference)** — Vendor modifier encoding, per-driver modifier tables, and the negotiation flowchart that produces the `uint64_t` modifier value used in `VK_EXT_image_drm_format_modifier`.

---

## 17. References

- Vulkan specification — all extensions: [https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html)
- `VK_KHR_wayland_surface`: [https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_wayland_surface.html](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_wayland_surface.html)
- `VK_EXT_image_drm_format_modifier`: [https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_image_drm_format_modifier.html](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_image_drm_format_modifier.html)
- `VK_KHR_external_memory_fd`: [https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_external_memory_fd.html](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_external_memory_fd.html)
- `VK_EXT_external_memory_dma_buf`: [https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_external_memory_dma_buf.html](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_external_memory_dma_buf.html)
- `VK_KHR_external_semaphore_fd`: [https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_external_semaphore_fd.html](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_external_semaphore_fd.html)
- `VK_KHR_external_fence_fd`: [https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_external_fence_fd.html](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_external_fence_fd.html)
- `VK_EXT_acquire_drm_display`: [https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_acquire_drm_display.html](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_acquire_drm_display.html)
- `VK_KHR_display`: [https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_display.html](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_display.html)
- `VK_KHR_display_swapchain`: [https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_display_swapchain.html](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_display_swapchain.html)
- `VK_EXT_headless_surface`: [https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_headless_surface.html](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_headless_surface.html)
- Mesa WSI Wayland source: [https://gitlab.freedesktop.org/mesa/mesa/-/blob/mesa-25.1.0/src/vulkan/wsi/wsi_common_wayland.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/mesa-25.1.0/src/vulkan/wsi/wsi_common_wayland.c)
- Mesa vulkan/runtime source: [https://gitlab.freedesktop.org/mesa/mesa/-/tree/mesa-25.1.0/src/vulkan/runtime](https://gitlab.freedesktop.org/mesa/mesa/-/tree/mesa-25.1.0/src/vulkan/runtime)
- Linux kernel dma-buf documentation: [https://www.kernel.org/doc/html/latest/driver-api/dma-buf.html](https://www.kernel.org/doc/html/latest/driver-api/dma-buf.html)
- Wayland linux-dmabuf-unstable-v1 protocol: [https://gitlab.freedesktop.org/wayland/wayland-protocols/-/tree/main/unstable/linux-dmabuf](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/tree/main/unstable/linux-dmabuf)
- GBM API reference: [https://gitlab.freedesktop.org/mesa/drm/-/tree/main/lib/gbm](https://gitlab.freedesktop.org/mesa/drm/-/tree/main/lib/gbm)
- Vulkan GPU Info (extension support per GPU): [https://vulkan.gpuinfo.org/](https://vulkan.gpuinfo.org/)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
