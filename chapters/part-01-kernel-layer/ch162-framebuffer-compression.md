# Chapter 162: Framebuffer Compression: AFBC, DCC, CCS, and UBWC

**Target audiences**: Graphics driver developers implementing or debugging hardware compression; systems engineers optimising memory bandwidth on embedded and mobile platforms; and display pipeline engineers understanding how compressed buffers flow through the KMS stack.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Why Framebuffer Compression?](#why-framebuffer-compression)
3. [AFBC: ARM Frame Buffer Compression](#afbc-arm-frame-buffer-compression)
4. [AMD DCC: Delta Colour Compression](#amd-dcc-delta-colour-compression)
5. [Intel CCS: Colour Control Surface](#intel-ccs-colour-control-surface)
6. [Qualcomm UBWC: Universal Bandwidth Compression](#qualcomm-ubwc-universal-bandwidth-compression)
7. [DRM Format Modifiers: The Kernel API](#drm-format-modifiers-the-kernel-api)
8. [EGL and Wayland Modifier Negotiation](#egl-and-wayland-modifier-negotiation)
9. [Debugging Compression](#debugging-compression)
10. [Integrations](#integrations)

---

## Introduction

Modern GPUs implement hardware framebuffer compression to reduce memory bandwidth — a critical resource, especially on mobile SoCs where GPU and CPU share LPDDR memory. Without compression, a single 4K@60 display at 32 bpp consumes ~4 GB/s of bandwidth just for display readout.

Vendors use proprietary compression schemes:
- **AFBC** (ARM Frame Buffer Compression) — ARM Mali, and also supported by ARM's display IP (not GPU-specific)
- **DCC** (Delta Colour Compression) — AMD GCN, RDNA
- **CCS** (Colour Control Surface) — Intel Gen9+
- **UBWC** (Universal Bandwidth Compression) — Qualcomm Adreno

The Linux kernel represents compressed buffers via **DRM format modifiers** — a 64-bit value appended to the DRM format (e.g. `DRM_FORMAT_XRGB8888 | modifier`) that encodes the compression scheme, tiling layout, and tile size. This chapter covers each compression scheme and the kernel/Mesa/Wayland infrastructure for modifier negotiation.

---

## Why Framebuffer Compression?

### Bandwidth Numbers

| Use Case | Bandwidth (uncompressed) |
|---|---|
| 1080p@60 XRGB8888 render | ~480 MB/s |
| 4K@60 XRGB8888 render | ~1920 MB/s |
| 4K@60 with 4× MSAA | ~7680 MB/s |
| Mobile LPDDR5 bandwidth | ~50–70 GB/s |

With 3–4× compression, bandwidth usage drops to ~500 MB/s for 4K@60, well within LPDDR5 budget.

### Cache Efficiency

Compressed tiles pack more data per cache line. A 4KB cache line fetching 64×4=256 bytes uncompressed pixels fetches 1024 bytes compressed — 4× better cache utilisation for textures.

---

## AFBC: ARM Frame Buffer Compression

### AFBC Format

AFBC organises the framebuffer into **superblocks** (16×16 pixels for 32bpp). Each superblock has:
- A **header** (16 bytes) describing which sub-blocks are solid-colour or compressed
- A **body** of variable-length compressed data

```
AFBC layout in memory:
┌─────────────────────────────────┐
│ Header table (N superblocks × 16B) │
├─────────────────────────────────┤
│ Body: compressed pixel data     │
│ (variable size per superblock)  │
└─────────────────────────────────┘
```

Compression exploits:
- **Solid colour sub-blocks**: 4×4 regions filled with one colour → 0 bytes body
- **Intra-frame prediction**: sub-blocks similar to neighbours → high compression
- Typical compression ratio: 2.5–4× for UI, 2× for game scenes

### AFBC Variants

| Modifier | Description |
|---|---|
| `AFBC_FORMAT_MOD_BLOCK_SIZE_16x16` | Standard 16×16 superblock |
| `AFBC_FORMAT_MOD_BLOCK_SIZE_32x8` | Wide superblock (for video) |
| `AFBC_FORMAT_MOD_YTR` | YUV-to-RGB transform in header |
| `AFBC_FORMAT_MOD_SPARSE` | Body in separate allocation |
| `AFBC_FORMAT_MOD_TILED` | Tiled header arrangement |

### AFBC DRM Modifiers

```c
/* include/uapi/drm/drm_fourcc.h */
#define DRM_FORMAT_MOD_ARM_AFBC(afbc_mode) \
    DRM_FORMAT_MOD_VENDOR_ARM | (afbc_mode)

/* Common AFBC modifier: 16×16 superblock, lossless: */
#define AFBC_FORMAT_MOD_BLOCK_SIZE_16x16  (1ULL)
#define AFBC_FORMAT_MOD_YTR               (1ULL << 4)
#define AFBC_FORMAT_MOD_SPLIT             (1ULL << 5)

uint64_t afbc_modifier = DRM_FORMAT_MOD_ARM_AFBC(
    AFBC_FORMAT_MOD_BLOCK_SIZE_16x16 |
    AFBC_FORMAT_MOD_YTR);
```

AFBC is supported by ARM's Bifrost/Valhall Mali GPUs and ARM's display IP (DPU) found in RK3399, RK3588, and many mobile SoCs.

---

## AMD DCC: Delta Colour Compression

### DCC Format

AMD DCC (available since GCN 1.2, mandatory on RDNA) compresses the colour buffer using delta encoding per 256-byte micro-tile:

```
DCC compressed surface:
┌──────────────────────────────────┐
│ Main surface (micro-tiled data)  │
├──────────────────────────────────┤
│ DCC key (256 bytes per 256B tile)│
│ = 1/256 overhead                 │
└──────────────────────────────────┘
```

DCC key values:
- `0x00` — tile is a single constant colour (0 bandwidth to read)
- `0x55` — tile is highly compressible (e.g. a gradient)
- `0xFF` — tile is uncompressed (random data)

### DCC and Display (DC)

AMD's Display Core (DC) can scanout DCC-compressed surfaces directly — the hardware decompressor sits in the display output path. This saves a full decompression pass before scanout:

```
GPU renders (DCC-compressed FBO)
  → Direct scanout via AMD DC
  → Display Pipe decompresses on-the-fly
  → HDMI/DP output
```

When direct scanout is not possible (format not supported by DC), Mesa decompresses to a linear buffer before passing to the display.

### DCC Modifiers

```c
/* AMD DCC modifier encoding (Mesa/radeonsi): */
/* include/uapi/drm/drm_fourcc.h */
#define AMD_FMT_MOD_DCC         (1ULL << 13)
#define AMD_FMT_MOD_DCC_RETILE  (1ULL << 14)  /* retile for display scanout */
#define AMD_FMT_MOD_DCC_PIPE_ALIGN (1ULL << 15)

/* A typical RDNA2 DCC modifier for a 2D texture: */
uint64_t dcc_mod = AMD_FMT_MOD |
    AMD_FMT_MOD_SET(TILE, AMD_FMT_MOD_TILE_GFX9_64K_R_X) |
    AMD_FMT_MOD_SET(DCC, 1) |
    AMD_FMT_MOD_SET(DCC_INDEPENDENT_64B, 1) |
    AMD_FMT_MOD_SET(DCC_MAX_COMPRESSED_BLOCK, AMD_FMT_MOD_DCC_BLOCK_64B);
```

### RADV DCC Management

```c
/* radv/radv_image.c: decide if DCC should be enabled: */
static bool radv_use_dcc_for_image_late(struct radv_device *device,
    struct radv_image *image)
{
    if (!device->physical_device->rad_info.has_dcc)
        return false;
    if (image->vk.samples > 1)
        return device->physical_device->rad_info.has_dcc_msaa;
    if (image->vk.format == VK_FORMAT_BC1_RGB_UNORM_BLOCK)
        return false;  /* BCn formats not DCC-compatible */
    return true;
}
```

---

## Intel CCS: Colour Control Surface

### CCS Format

Intel's CCS (Gen9–Gen12) stores a 2-bit compression tag per 8×8 pixel tile:

```
CCS tag values:
  00 = fully uncompressed
  01 = fully compressed (single colour via clear colour)
  10 = partially compressed
  11 = unresolvable (requires decompress pass)
```

CCS uses 1/256 overhead like DCC.

### Gen12 and XeHPC Variants

Intel Gen12 (Tiger Lake+) introduced multi-level CCS:

```c
/* drm_fourcc.h Intel modifiers: */
/* CCS for render targets: */
#define I915_FORMAT_MOD_Y_TILED_CCS        fourcc_mod_code(INTEL, 4)
#define I915_FORMAT_MOD_Yf_TILED_CCS       fourcc_mod_code(INTEL, 5)
/* Gen12 flat CCS: */
#define I915_FORMAT_MOD_Y_TILED_GEN12_RC_CCS    fourcc_mod_code(INTEL, 6)
#define I915_FORMAT_MOD_Y_TILED_GEN12_MC_CCS    fourcc_mod_code(INTEL, 7)
/* Xe2 (Lunar Lake) CCS: */
#define I915_FORMAT_MOD_4_TILED_BMG_CCS   fourcc_mod_code(INTEL, 14)
```

### ANV CCS Integration

```c
/* anv/anv_image.c: CCS plane management */
struct anv_image {
    /* CCS aux surface (separate from main surface): */
    struct anv_surface planes[ANV_IMAGE_MEMORY_BINDING_END];
    /* Planes: main colour, CCS aux, stencil, etc. */
};

static VkResult anv_image_init_from_create_info(
    struct anv_device *device,
    struct anv_image *image,
    const VkImageCreateInfo *pCreateInfo)
{
    /* Allocate CCS if format is renderable and CCS-compatible: */
    if (image->usage & VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT &&
        isl_format_supports_ccs_e(device->info, isl_format))
    {
        image->planes[plane].aux_usage = ISL_AUX_USAGE_CCS_E;
    }
}
```

---

## Qualcomm UBWC: Universal Bandwidth Compression

### UBWC Format

Qualcomm UBWC (introduced A5xx) uses 4×4 micro-tiles with per-tile compression metadata:

```
UBWC layout:
┌────────────────────────────────────┐
│ Meta surface (1 byte per 4×4 tile) │  → 1/16 overhead
├────────────────────────────────────┤
│ Data surface (tiled 64×16 pixels)  │
└────────────────────────────────────┘
```

UBWC metadata byte:
- `0x00` — tile is a fast-clear colour
- `0x01`–`0x03` — tile is compressed at different levels
- `0xFF` — tile is uncompressed

### UBWC Modifiers

```c
/* Freedreno UBWC modifier (from Mesa): */
/* drm_fourcc.h */
#define DRM_FORMAT_MOD_QCOM_COMPRESSED fourcc_mod_code(QCOM, 1)
#define DRM_FORMAT_MOD_QCOM_TILED      fourcc_mod_code(QCOM, 2)
```

Turnip and Freedreno enable UBWC on all A5xx+ hardware for both textures and render targets.

---

## DRM Format Modifiers: The Kernel API

### What a Modifier Is

A modifier is a 64-bit value with the upper 8 bits identifying the vendor:

```c
/* drm_fourcc.h */
#define DRM_FORMAT_MOD_VENDOR_NONE   0
#define DRM_FORMAT_MOD_VENDOR_AMD    0x02
#define DRM_FORMAT_MOD_VENDOR_INTEL  0x03
#define DRM_FORMAT_MOD_VENDOR_NVIDIA 0x04
#define DRM_FORMAT_MOD_VENDOR_ARM    0x08
#define DRM_FORMAT_MOD_VENDOR_QCOM   0x05

/* A linear (uncompressed) modifier: */
#define DRM_FORMAT_MOD_LINEAR        0ULL
/* A modifier with invalid/unknown value: */
#define DRM_FORMAT_MOD_INVALID       (uint64_t)(-1)
```

### Advertising Modifiers via IN_FORMATS

KMS planes advertise supported modifiers via the `IN_FORMATS` blob property:

```c
/* Query supported modifiers for a plane: */
drmModeObjectProperties *props = drmModeObjectGetProperties(fd, plane_id,
    DRM_MODE_OBJECT_PLANE);
uint32_t in_formats_prop_id = /* find prop id for "IN_FORMATS" */;
uint64_t in_formats_blob_id = /* value of IN_FORMATS prop */;

drmModePropertyBlobRes *blob = drmModeGetPropertyBlob(fd, in_formats_blob_id);
struct drm_format_modifier_blob *fmtmod = blob->data;

/* Iterate formats and their supported modifiers: */
for (uint32_t i = 0; i < fmtmod->count_formats; i++) {
    uint32_t *formats = (void*)fmtmod + fmtmod->formats_offset;
    for (uint32_t j = 0; j < fmtmod->count_modifiers; j++) {
        struct drm_format_modifier *mods =
            (void*)fmtmod + fmtmod->modifiers_offset;
        if (mods[j].formats & (1ULL << i)) {
            printf("Format %4.4s supports modifier 0x%016llx\n",
                (char*)&formats[i], mods[j].modifier);
        }
    }
}
```

---

## EGL and Wayland Modifier Negotiation

### eglQueryDmaBufModifiersEXT

Mesa exposes supported modifiers per format:

```c
/* Query which modifiers are supported for rendering: */
EGLint num_modifiers;
eglQueryDmaBufModifiersEXT(display, DRM_FORMAT_XRGB8888,
    0, NULL, NULL, &num_modifiers);

uint64_t *modifiers = malloc(num_modifiers * sizeof(uint64_t));
EGLBoolean *external_only = malloc(num_modifiers * sizeof(EGLBoolean));
eglQueryDmaBufModifiersEXT(display, DRM_FORMAT_XRGB8888,
    num_modifiers, modifiers, external_only, &num_modifiers);

/* modifiers[0] may be: AMD_FMT_MOD | DCC | ...  */
/* external_only: if true, modifier only usable as texture, not render target */
```

### zwp_linux_dmabuf_v1 Modifier Feedback

Wayland's `zwp_linux_dmabuf_feedback_v1` protocol lets the compositor tell clients which modifiers to use for scanout:

```c
/* Compositor sends feedback: "use AFBC for this screen" */
/* Client receives: */
static void dmabuf_feedback_format_table(void *data,
    struct zwp_linux_dmabuf_feedback_v1 *feedback,
    int32_t fd, uint32_t size)
{
    /* Parse format+modifier table from fd */
}

static void dmabuf_feedback_tranche_formats(void *data,
    struct zwp_linux_dmabuf_feedback_v1 *feedback,
    struct wl_array *indices)
{
    /* indices reference into the format table */
    /* Client should allocate DMA-BUFs using these format+modifier pairs */
}
```

This allows zero-copy scanout: the client allocates in the compositor's preferred compressed format, avoiding decompression before display.

---

## Debugging Compression

### Checking Active Modifiers

```bash
# What modifier is the compositor using?
WAYLAND_DEBUG=1 weston 2>&1 | grep modifier

# Check modifier on a buffer:
# (requires drm-info or custom tool reading plane properties)
drm-info /dev/dri/card0 | grep -A5 "IN_FORMATS"

# AMD: check if DCC is enabled:
AMD_DEBUG=nodcc app  # disable DCC to compare performance
AMD_DEBUG=nodcc      # check if DCC was causing issues

# Intel: disable CCS for debugging:
INTEL_DEBUG=noccs app

# Check framebuffer compression via GPU trace:
apitrace trace -a egl app
# Then look at eglCreateImageKHR attributes for modifier
```

### Compression Effectiveness

```bash
# AMD: GPU counter for DCC compression ratio:
# Via Mesa's GALLIUM_HUD:
GALLIUM_HUD="dcc-compressed,dcc-uncompressed" app

# ARM: check AFBC via kernel:
cat /sys/kernel/debug/dri/0/state  # shows plane modifier
```

---

## Roadmap

### Near-term (6–12 months)

- **AMD DCC for multi-plane and video formats on RDNA4 (GFX12)**: Mesa 25.1 landed DCC support for multi-plane formats on RDNA4 hardware, and tiling is now enabled by default for VAAPI/VCN encode, decode, and JPEG paths — extending bandwidth savings to video workloads previously left uncompressed. [Source](https://www.phoronix.com/news/Mesa-25.1-Multi-Plane-DCC-RDNA4)
- **RADV DCC fast clears on RDNA3 (GFX11)**: Samuel Pitoiset (Valve) expanded DCC fast-clear support to 8 bpp and 16 bpp block sizes on RDNA3, reducing the number of cases where a full clear pass is required. [Source](https://www.phoronix.com/forums/forum/linux-graphics-x-org-drivers/open-source-amd-linux/1530895-radv-driver-expands-use-of-performance-helping-dcc-fast-clears-on-rdna3-gpus)
- **Intel CCS restoration for Xe2 (Battlemage) via SR-IOV**: The Xe kernel driver is adding SR-IOV support to restore Compression Control Surface (CCS) state for Xe2 and newer GPUs; Battlemage requires 64 KB scanout-buffer alignment for compressed formats — enforcement patches are in flight for Linux 6.18. [Source](https://www.phoronix.com/news/Linux-6.18-More-Xe-SR-IOV)
- **Qualcomm UBWC gralloc detection fixes in Turnip/Freedreno**: Upstream Mesa commits are addressing UBWC detection failures introduced by newer Qualcomm gralloc versions (`u_gralloc` magic-check removal), preventing display corruption on A6xx/A7xx devices. [Source](https://deepwiki.com/bminor/mesa-mesa/2.3-turnip-qualcomm-vulkan-driver)
- **AFBC for additional Rockchip and MediaTek DPU display cores**: Ongoing upstreaming of AFBC modifier support for newer Rockchip (RK3588 DPU) and MediaTek display subsystems in the `drm-misc` tree. Note: needs verification for specific kernel cycle targets.

### Medium-term (1–3 years)

- **Unified modifier negotiation via KMS `IN_FORMATS` v2 / format feedback**: The existing `zwp_linux_dmabuf_feedback_v1` mechanism works well but is Wayland-only; there is ongoing discussion around making format-modifier capability tables (producer/consumer matching) a first-class DRM ioctl so headless and non-Wayland consumers benefit. Note: needs verification — no merged RFC as of 2026.
- **AFBC v3 and AFBC wide-block (32×8) broader driver adoption**: ARM's newer Immortalis GPU IP supports wider AFBC superblock variants and the AFBC v3 encoding; display driver support (Komeda, DW-HDMI, and SoC display engines) needs to catch up to allow zero-copy compressed scanout on the latest ARM Cortex/Immortalis platforms. [Source](https://developer.arm.com/Architectures/Arm%20Frame%20Buffer%20Compression)
- **AMD DCC on Infinity Cache (RDNA4)**: AMD's RDNA4 architecture extends DCC compression into the Infinity Cache hierarchy; Mesa and radeonsi will need cache-coherence and flush-sequence updates to exploit this safely. [Source](https://chipsandcheese.com/p/amds-rdna4-gpu-architecture-at-hot)
- **Qualcomm UBWC for A8xx (Gen 4) in mainline**: Newer Adreno A810/A825/A830 GPU configurations have appeared in upstream Mesa; full UBWC scanout support for these cores (including KMS modifier advertisement in the msm DRM driver) is expected to follow as the hardware matures in the mainline kernel. [Source](https://deepwiki.com/bminor/mesa-mesa/2.3-turnip-qualcomm-vulkan-driver)
- **Cross-vendor modifier validation helpers**: A kernel-level helper library to validate modifier constraints (alignment, pitch, format compatibility) common across AFBC, DCC, and UBWC, reducing duplicated validation code in each driver. Note: needs verification — discussed on dri-devel but no patchset merged.

### Long-term

- **Compressed memory import/export across GPU vendors**: Long-term ambition to allow, for example, a Qualcomm VPU to decompress a frame that a downstream Adreno GPU reads without an explicit decompress blit — requires a common modifier namespace and cross-IP coherency protocol that no current SoC fully implements.
- **Lossy compression tiers via DRM modifiers**: Some hardware (e.g., ARM's Immortalis "Fixed-Rate Compression") supports configurable lossy compression at fixed bit-rates; exposing this via a modifier quality field or a new KMS property is an open design question in the DRM community. Note: needs verification.
- **Transparent compression below the DRM layer (SMMU/IOMMU)**: Research directions propose offloading compression entirely to the SMMU/IOMMU or memory controller so that producers and consumers see uncompressed addresses — eliminating the need for per-driver modifier negotiation at the cost of hardware complexity.
- **Standardised compression metadata for cross-process zero-copy**: Extending `dma_buf` with a standardised sidecar for compression metadata (block map, header tables) so that any consumer (display, codec, ML accelerator) can decompress a buffer produced by any vendor GPU without a driver-specific path. Note: needs verification — no agreed standard as of 2026.

---

## Integrations

- **Ch04 (DRM Framebuffers)** — `drmModeAddFB2WithModifiers` is the kernel API for creating a compressed framebuffer; the modifier must be supported by both the plane and the GPU driver
- **Ch06 (KMS Atomic)** — the `IN_FORMATS` plane property is the atomic-mode mechanism for advertising modifier support; modifier-bearing framebuffers are committed via the same atomic path
- **Ch11 (DMA-BUF)** — DMA-BUF FDs carry the modifier as metadata; `EGL_LINUX_DMA_BUF_EXT` and `zwp_linux_dmabuf_v1` both transmit the modifier alongside the FD
- **Ch150 (EGL/DMA-BUF)** — `eglQueryDmaBufModifiersEXT` and `eglCreateImageKHR` with modifier attributes are the EGL API for compressed buffer import
- **Ch22 (RADV)** — RADV DCC management and fast-clear elimination; DCC clearing uses a special compute pass rather than memset
- **Ch23 (ANV)** — Intel CCS is tightly integrated with ISL (Intel Surface Layout library) in ANV; ISL abstracts tiling and CCS placement
- **Ch159 (Panfrost)** — Panfrost/Panthor support AFBC on Mali hardware; RK3588's Mali-G610 + ARM DPU both understand AFBC, enabling zero-copy scanout
