# Chapter 186: Video Pixel Formats and Display Signal Encoding

> **Part**: Part XXVII — Display Hardware and Connectors
> **Audiences**: Graphics application developers working with video surfaces and format-negotiation APIs; kernel driver authors implementing DRM plane format support and InfoFrame generation; multimedia engineers connecting hardware decode pipelines to the display stack.
> **Status**: First draft — 2026-06-20

## Table of Contents

- [Overview](#overview)
- [1. Color Space Fundamentals — RGB vs YCbCr](#1-color-space-fundamentals--rgb-vs-ycbcr)
- [2. Chroma Subsampling](#2-chroma-subsampling)
- [3. Bit Depth](#3-bit-depth)
- [4. Quantization Range — Full vs Limited Swing](#4-quantization-range--full-vs-limited-swing)
- [5. DRM fourcc Pixel Format Taxonomy](#5-drm-fourcc-pixel-format-taxonomy)
- [6. V4L2 Pixel Formats and the Multiplanar API](#6-v4l2-pixel-formats-and-the-multiplanar-api)
- [7. HDR Metadata in the Display Signal](#7-hdr-metadata-in-the-display-signal)
- [8. AVI InfoFrame Color Encoding](#8-avi-infoframe-color-encoding)
- [9. Format Negotiation Pipeline](#9-format-negotiation-pipeline)
- [10. Worked Example — 4K HDR10 Video Playback](#10-worked-example--4k-hdr10-video-playback)
- [Integrations](#integrations)
- [References](#references)

---

## Overview

This chapter covers the end-to-end journey of a video pixel from codec decoder to display panel:

- **Color space model** — the YCbCr or RGB space the pixel lives in, governed by BT.601, BT.709, or BT.2020 matrix coefficients
- **Chroma subsampling** — the spatial compression applied to chroma components
- **Bit depth** — the quantization depth of each component
- **DRM fourcc code** — the kernel format identifier used in plane configuration
- **V4L2 structure** — the descriptor for decoder buffer formats in the multiplanar API
- **HDR metadata** — the static or dynamic metadata accompanying the signal over HDMI or DisplayPort
- **AVI InfoFrame** — the colorimetry declaration embedded in the HDMI data island
- **Format-negotiation path** — the pipeline determining whether a compositor can hand the buffer to a KMS overlay plane for direct scanout

- **Graphics application developers** — DRM fourcc codes, quantization range mismatches, and the `zwp_linux_dmabuf_v1` format/modifier negotiation protocol, directly actionable when writing video playback paths or GPU compositors
- **Kernel driver authors** — `drm_fourcc.h` format enumerations, the `IN_FORMATS` plane property, the `hdr_output_metadata` blob, and how amdgpu and i915 drive the HDMI AVI InfoFrame
- **Multimedia engineers** — the VA-API → DMA-BUF → DRM overlay pipeline, V4L2 multiplanar format negotiation, and the worked 4K HDR10 example covering the whole stack from decoder buffer to signal wire

---

## 1. Color Space Fundamentals — RGB vs YCbCr

### Why YCbCr?

The human visual system has three types of cone photoreceptors, but their sensitivities are unequal: roughly 64 % of cones are sensitive to long (red-ish) wavelengths, 32 % to medium (green-ish) wavelengths, and only 2 % to short (blue-ish) wavelengths. More importantly, spatial acuity — the ability to resolve fine detail — is dominated by luminance (brightness) differences rather than chrominance (color) differences. This perceptual asymmetry motivated the YCbCr representation: encode brightness as a single luma component Y and encode color as two difference signals Cb (blue-difference chroma) and Cr (red-difference chroma). The eye's lower sensitivity to chroma detail then allows those components to be subsampled in space without visible loss, achieving significant bandwidth reduction.

YCbCr was also engineered to be backward compatible with the monochrome NTSC television broadcast system (1941–2009). In NTSC, the Y signal is the complete black-and-white picture; the chroma subcarrier modulates Cb and Cr onto the same composite signal in a frequency interleaving scheme that a black-and-white receiver ignores. Digital video inherited this decomposition and has retained it ever since. [Source](https://www.kernel.org/doc/html/v4.11/media/uapi/v4l/pixfmt-008.html)

### BT.601, BT.709, and BT.2020

The exact weights used to derive Y from R, G, B differ between standards because each standard defines different display primaries — the actual chromaticities of the red, green, and blue phosphors or filters of the target display.

**ITU-R BT.601** (1982, revised 2011) targets standard-definition (SD) content at 525-line and 625-line frame rates and the SMPTE C / EBU primaries of CRT broadcast monitors. [Source](https://www.itu.int/rec/R-REC-BT.601)

**ITU-R BT.709** (1990, revised 2015) targets high-definition (HD) content at 1080i/1080p/720p and defines the primaries used by virtually all consumer HD content today:

```text
Y  =  0.2126·R  +  0.7152·G  +  0.0722·B
Cb = (B − Y) / 1.8556
Cr = (R − Y) / 1.5748
```

[Source](https://www.itu.int/rec/R-REC-BT.709)

In digital representation with 8 bits and studio swing (see Section 4), the Y channel occupies 16–235 and the Cb/Cr channels occupy 16–240, with 128 as the chroma neutral axis.

**ITU-R BT.2020** (2012, revised 2015) targets ultra-high-definition (UHD) content and HDR. It specifies wider primaries that encompass a larger fraction of the CIE 1931 color gamut — approximately 75.8 % of the visible gamut versus BT.709's ~35.9 %. The luma coefficients shift to reflect the new primaries:

```text
Y  =  0.2627·R  +  0.6780·G  +  0.0593·B
Cb = (B − Y) / 1.8814
Cr = (R − Y) / 1.4746
```

[Source](https://www.itu.int/rec/R-REC-BT.2020)

Applying BT.709 coefficients to content mastered against BT.2020 primaries — or the reverse — produces a color shift that is visible to most observers, particularly in highly saturated reds and greens. Accurate color management therefore requires signaling the colorimetry in the display data stream (see Section 8).

### Transfer Functions

A **transfer function** relates linear scene light to nonlinear encoded signal values, or equivalently maps a stored numerical code back to display light output. There are three transfer functions relevant to the Linux graphics stack:

**sRGB / BT.709 gamma (~2.2):** A piecewise function with a linear segment near black and a power-law exponent of ~2.2 elsewhere. BT.709 and sRGB use nearly identical curves; IEC 61966-2-1 defines the sRGB variant. [Source](https://www.w3.org/Graphics/Color/srgb) Appropriate for consumer HD monitors and web content.

**SMPTE ST 2084 Perceptual Quantizer (PQ):** Designed for HDR10 and Dolby Vision HDR. The OETF maps linear scene light in the range 0–10,000 cd/m² into code values using the formula:

```text
L' = ((c1 + c2·L^m1) / (1 + c3·L^m1))^m2
where m1 = 0.1593017578125, m2 = 78.84375
      c1 = 0.8359375, c2 = 18.8515625, c3 = 18.6875
```

[Source](https://pub.smpte.org/latest/st2084/st2084-2014.pdf)

The PQ curve's absolute luminance encoding (code 0 = 0 cd/m², code 10000 = 10,000 cd/m²) is a fundamental difference from BT.709 whose codes are relative to some unspecified peak white.

**ARIB STD-B67 Hybrid Log-Gamma (HLG):** An alternative HDR transfer function designed for live broadcast compatibility. HLG uses a logarithmic curve for high luminance and a linear curve for low luminance, maintaining backward compatibility with SDR displays on the same signal. [Source](https://www.arib.or.jp/english/html/overview/doc/2-STD-B67v1_0.pdf) The Linux kernel DRM layer supports HLG as EOTF type 3 in the `hdr_metadata_infoframe` structure.

### The Four Components of a Fully Specified Color Space

A complete color-space specification requires:
1. **Primary chromaticities** — CIE xy coordinates of R, G, B reference primaries.
2. **White point** — usually D65 (x=0.3127, y=0.3290) for consumer standards.
3. **Transfer function** — the OETF/EOTF mapping between linear and nonlinear.
4. **Color difference encoding** — the YCbCr matrix coefficients (if applicable).

All four must be consistently signaled from source to display. Mismatches at any layer produce visible errors. The DRM `colorspace` connector property and HDMI AVI InfoFrame together encode items 1–4 in the display signal.

---

## 2. Chroma Subsampling

Chroma subsampling exploits the eye's reduced spatial acuity for color. The notation `J:a:b` describes a block of `J` pixels wide:

- **4:4:4** — every pixel has its own Y, Cb, Cr. No subsampling. Full quality. 24 bits/pixel at 8-bit depth.
- **4:2:2** — Cb and Cr are sampled at half the horizontal rate. 16 bits/pixel average (8Y + 4Cb + 4Cr per two pixels).
- **4:2:0** — Cb and Cr are sampled at half rate both horizontally and vertically. 12 bits/pixel average. Saves 50 % bandwidth versus 4:4:4 at the same bit depth.
- **4:1:1** — Cb and Cr at one quarter horizontal rate. 12 bits/pixel average, same as 4:2:0, but different spatial distribution; historically used in DV and NTSC DVCam. Rare in the Linux display stack.

The dominant format in video codecs and display pipelines is **4:2:0** for its favorable quality/bandwidth tradeoff.

### Memory Layout of NV12 (4:2:0 Semi-Planar)

NV12 is the most widely supported 4:2:0 format in both the Linux DRM layer and hardware decoders. Its layout for a 1920×1080 frame:

```text
Plane 0 (luma):   1920 × 1080 bytes  = 2,073,600 bytes (Y samples)
Plane 1 (chroma): 1920 × 540  bytes  = 1,036,800 bytes (interleaved Cb, Cr)
Total:                                = 3,110,400 bytes ≈ 2.97 MB
```

Each chroma sample in plane 1 is a pair `[Cb, Cr]` at 8 bits each. The Cb sample is at even byte offsets, Cr at odd byte offsets. The plane 1 stride is the same as plane 0 stride (the width in bytes), not half of it, because the two chroma components are interleaved rather than packed side-by-side.

```c
/* NV12 manual layout calculation */
size_t y_plane_size  = stride * height;
size_t uv_plane_size = stride * (height / 2);  /* interleaved Cb+Cr */
size_t total_size    = y_plane_size + uv_plane_size;

/* Access pixel at (x, y): */
uint8_t luma   = y_plane[y * stride + x];
uint8_t chroma_cb = uv_plane[(y/2) * stride + (x & ~1)];     /* even */
uint8_t chroma_cr = uv_plane[(y/2) * stride + (x & ~1) + 1]; /* odd  */
```

### Chroma Siting

Chroma siting describes where within the 2×2 luma block the single shared chroma sample is positioned.

**MPEG-2 / H.264 / H.265 cosited:** The chroma sample is collocated with the top-left luma sample of each 2×2 block. This is the convention used by all DRM YUV formats and is signaled as `DRM_FORMAT_MOD_CHROMASITING_COSITED` in modifier metadata. Hardware scalers must apply a half-pixel shift when converting between cosited and interstitial representations.

**JPEG / JFIF interstitial:** The chroma sample is centered within the 2×2 luma block, i.e., at position (0.5, 0.5) relative to the top-left luma sample. Common in JPEG files but rare on the display signal path.

The cosited convention is now universally assumed by DRM hardware planes. If a decoder produces interstitial-sited chroma (e.g., from a JPEG source), a phase-corrected scaling step is required before the surface can be scanned out directly. [Source](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/pixfmt-yuv-planar.html)

---

## 3. Bit Depth

### 8-Bit Consumer sRGB

The baseline. Code values 0–255. Sufficient for sRGB content (≈100 cd/m² peak white, BT.709 gamut). All consumer displays and GPU outputs have supported 8-bit RGB since the late 1990s. Storage formats: DRM_FORMAT_XRGB8888, DRM_FORMAT_NV12.

### 10-Bit HDR Consumer (P010)

HDR10 mastering and playback both require 10 bits to avoid visible contouring (banding) in smooth gradients when the PQ EOTF is applied. The industry-standard format for 10-bit 4:2:0 video is **P010**: a semi-planar NV12-like layout where each Y, Cb, Cr sample occupies 16 bits in a little-endian word, with the 10 significant bits in the **10 most significant bit positions** (bits 15–6) and six zero-pad bits in positions 5–0. [Source](https://learn.microsoft.com/en-us/windows/win32/medfound/10-bit-and-16-bit-yuv-video-formats)

```text
P010 word layout (little-endian 16-bit word):
Bits 15:6  — 10-bit sample value (MSB-aligned)
Bits  5:0  — zero padding
```

Memory footprint for 4K (3840×2160): 3840×2160×2 (luma) + 3840×1080×2 (chroma) = 24,883,200 bytes ≈ 23.7 MB per frame.

### 12-Bit Dolby Vision Mastering

Dolby Vision profile 5 (single-layer IPTPQc2) uses 12-bit encoding. Linux userspace support is emerging through the `DRM_FORMAT_P012` format code. Most hardware decoders export 12-bit surfaces as P016 (16-bit words, top-12-bits meaningful, bottom 4 zero) for alignment. Note: hardware display engine support for 12-bit pixel processing is uncommon outside professional monitor pipelines as of 2026.

### 16-Bit Float

Used internally by GPU compositors and color management pipelines but rarely for scanout. Mesa's Vulkan compositors operate in `VK_FORMAT_R16G16B16A16_SFLOAT` for HDR compositing before final tone mapping. Direct scanout in 16-bit float is not supported by consumer display engines. Note: needs verification against specific hardware generation capability registers.

### True 10-bit vs 8-bit + FRC

Consumer monitors often advertise "10-bit color" while internally being 8-bit panels using **Frame Rate Control (FRC)**: temporal dithering that alternates between adjacent 8-bit levels across frames to simulate a 10-bit average. An EDID-decode inspection distinguishes true 10-bit from FRC:

```bash
edid-decode /sys/class/drm/card0-HDMI-A-1/edid 2>&1 | grep -A2 "Color Bit Depth"
# True 10-bit output:
#   Color Bit Depth: 10 bits per primary color
# 8-bit + FRC (typical):
#   Color Bit Depth: 8 bits per primary color
```

[Source](https://www.kernel.org/doc/html/latest/gpu/edid_decoder.html)

True 10-bit panels expose the `10 bits per primary color` EDID field. When negotiating color depth via DP or HDMI, the kernel uses this field to determine the maximum bits-per-component (BPC) to offer, controlled by the DRM `max bpc` connector property.

---

## 4. Quantization Range — Full vs Limited Swing

### The Two Ranges

**Full range** (also called "PC range"): luma and chroma both use the full 8-bit code space 0–255 (or 0–1023 at 10 bit). Black = 0, peak white = 255.

**Limited range** (also called "studio swing" or "broadcast range"): luma is constrained to 16–235; chroma to 16–240 (8-bit). Code values 0–15 and 236–255 are reserved as "super-black" and "super-white" headroom for synchronization signals in the original analog broadcast chain.

The reservation is defined in SMPTE 125M and ITU-R BT.601 and has been carried forward into digital video to maintain consistency with broadcast infrastructure. [Source](https://www.itu.int/rec/R-REC-BT.601)

### Mismatch Symptoms

When the source is full-range but the display or sink expects limited range, the display clips values above 235 and below 16 is displayed too bright. The visible symptom is **washed-out, flat-looking image** — blacks appear grey and highlights are exaggerated.

When the source is limited-range but the display expects full range, the display stretches only 16–235 across the full panel range. Values 0–15 are crushed to absolute black, and values 236–255 are clipped to peak white. The visible symptom is **crushed blacks and blown-out highlights**.

### DRM "Broadcast RGB" Connector Property

The i915 (Intel) driver exposes a `"Broadcast RGB"` connector property that controls what quantization range is signaled in the HDMI AVI InfoFrame and, correspondingly, what range the display engine outputs: [Source](https://www.mail-archive.com/dri-devel@lists.freedesktop.org/msg490843.html)

```text
Property "Broadcast RGB":
  "Automatic"      — driver selects range per HDMI 1.4b §6.6 rules
  "Full"           — force full range (RGB 0–255)
  "Limited 16:235" — force limited/studio range
```

The "Automatic" value implements HDMI 1.4b Section 6.6: CE (Consumer Electronics) video timings (VIC codes) default to limited range; PC timings (no VIC code, custom) default to full range.

Query the current value via libdrm:

```c
drmModeConnectorPtr conn = drmModeGetConnector(fd, connector_id);
for (int i = 0; i < conn->count_props; i++) {
    drmModePropertyPtr prop = drmModeGetProperty(fd, conn->props[i]);
    if (strcmp(prop->name, "Broadcast RGB") == 0) {
        printf("Broadcast RGB enum value: %llu\n", conn->prop_values[i]);
    }
    drmModeFreeProperty(prop);
}
```

### amdgpu Quantization in HDMI AVI InfoFrame

The amdgpu driver sets the quantization range in the HDMI AVI InfoFrame's `Q1:Q0` bits via `hdmi_avi_infoframe_set_quantization_range()` when building the InfoFrame (see Section 8). The amdgpu display engine can independently apply a hardware quantization scaler in the Display Core output pipe to remap full-range source data to limited-range signal output, or vice versa, without a software intermediate step. [Source](https://docs.kernel.org/gpu/amdgpu/display/index.html)

---

## 5. DRM fourcc Pixel Format Taxonomy

### Naming Convention

DRM fourcc codes are 32-bit values encoding four ASCII characters, defined in `include/uapi/drm/drm_fourcc.h`. [Source](https://github.com/torvalds/linux/blob/master/include/uapi/drm/drm_fourcc.h) The macro `fourcc_code(a, b, c, d)` packs characters a–d into a little-endian 32-bit integer: `a | (b << 8) | (c << 16) | (d << 24)`. For example, `DRM_FORMAT_NV12` = `fourcc_code('N','V','1','2')` = 0x3231564E.

The descriptive comment next to each definition uses the notation `[31:0] A:B:C:D w:x:y:z` meaning bit 31 is component A with width `w`, bit (31-w) is component B with width `x`, and so on, describing the little-endian in-memory word layout.

### RGB Formats

| Macro | fourcc | Description |
|---|---|---|
| `DRM_FORMAT_XRGB8888` | `'X','R','2','4'` | `[31:0] x:R:G:B 8:8:8:8` |
| `DRM_FORMAT_ARGB8888` | `'A','R','2','4'` | `[31:0] A:R:G:B 8:8:8:8` |
| `DRM_FORMAT_BGR888`   | `'B','G','2','4'` | `[23:0] B:G:R 8:8:8` |
| `DRM_FORMAT_RGB565`   | `'R','G','1','6'` | `[15:0] R:G:B 5:6:5` |
| `DRM_FORMAT_XRGB2101010` | `'X','R','3','0'` | `[31:0] x:R:G:B 2:10:10:10` |
| `DRM_FORMAT_XBGR2101010` | `'X','B','3','0'` | `[31:0] x:B:G:R 2:10:10:10` |

`XRGB8888` is the universal fallback for primary planes on every DRM driver. `XRGB2101010` / `XBGR2101010` are the 10-bit variants used for HDR compositing surfaces and are supported on amdgpu, i915, and nouveau from roughly 2018 hardware onwards.

### YUV Packed Formats

Packed formats interleave luma and chroma samples in a single plane using a macro-pixel of two luma samples and one shared chroma pair:

| Macro | fourcc | Byte order in memory |
|---|---|---|
| `DRM_FORMAT_YUYV` | `'Y','U','Y','V'` | Y0, Cb0, Y1, Cr0 |
| `DRM_FORMAT_UYVY` | `'U','Y','V','Y'` | Cb0, Y0, Cr0, Y1 |
| `DRM_FORMAT_YVYU` | `'Y','V','Y','U'` | Y0, Cr0, Y1, Cb0 |

These 4:2:2 packed formats appear on older overlay planes and V4L2 capture devices. They are 16 bits/pixel average.

### YUV Planar and Semi-Planar Formats

```text
Fully planar (I420 / YV12):
  DRM_FORMAT_YUV420  ('Y','U','1','2')  — Y plane, then Cb plane, then Cr plane
  DRM_FORMAT_YVU420  ('Y','V','1','2')  — Y plane, then Cr plane, then Cb plane

Semi-planar (NV12 / NV21):
  DRM_FORMAT_NV12    ('N','V','1','2')  — Y plane, then interleaved Cb:Cr plane
  DRM_FORMAT_NV21    ('N','V','2','1')  — Y plane, then interleaved Cr:Cb plane

10-bit semi-planar:
  DRM_FORMAT_P010    ('P','0','1','0')  — 10-bit MSB-aligned in 16-bit words, NV12 layout
  DRM_FORMAT_P016    ('P','0','1','6')  — 16-bit per component, NV12 layout
```

`NV12` and `P010` are the two most important formats for video overlay planes. VA-API decoders universally output NV12 (8-bit) or P010 (10-bit HDR), and nearly every display engine with an overlay plane supports at least NV12 for direct scanout.

### Enumerating Supported Formats from a DRM Plane

```c
#include <xf86drm.h>
#include <xf86drmMode.h>
#include <drm/drm_fourcc.h>

void list_plane_formats(int drm_fd, uint32_t plane_id)
{
    drmModePlanePtr plane = drmModeGetPlane(drm_fd, plane_id);
    if (!plane) {
        perror("drmModeGetPlane");
        return;
    }

    printf("Plane %u supports %u formats:\n", plane_id, plane->count_formats);
    for (uint32_t i = 0; i < plane->count_formats; i++) {
        uint32_t fmt = plane->formats[i];
        /* Format is a fourcc: print its ASCII characters */
        printf("  0x%08x  %c%c%c%c\n", fmt,
               (char)(fmt & 0xff),
               (char)((fmt >> 8) & 0xff),
               (char)((fmt >> 16) & 0xff),
               (char)((fmt >> 24) & 0xff));
    }
    drmModeFreePlane(plane);
}
```

[Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_plane.c)

For modifier-aware enumeration, read the `IN_FORMATS` plane property blob, which contains a `drm_format_modifier_blob` header listing every (format, modifier) pair the plane accepts. [Source](https://github.com/torvalds/linux/blob/master/include/uapi/drm/drm_mode.h)

---

## 6. V4L2 Pixel Formats and the Multiplanar API

### V4L2 Format Definitions

The Video4Linux2 subsystem defines its own pixel format namespace in `include/uapi/linux/videodev2.h`. [Source](https://github.com/torvalds/linux/blob/master/include/uapi/linux/videodev2.h) Formats use the same `v4l2_fourcc()` macro as DRM's `fourcc_code()` but the two namespaces are independent — the same four-character code may denote identical layouts in both, but userspace must not assume identity.

Key format constants:

```c
#define V4L2_PIX_FMT_YUYV    v4l2_fourcc('Y', 'U', 'Y', 'V') /* 4:2:2 packed  */
#define V4L2_PIX_FMT_YUV420  v4l2_fourcc('Y', 'U', '1', '2') /* 4:2:0 planar  */
#define V4L2_PIX_FMT_NV12    v4l2_fourcc('N', 'V', '1', '2') /* 4:2:0 s-planar*/
#define V4L2_PIX_FMT_P010    v4l2_fourcc('P', '0', '1', '0') /* 10-bit 4:2:0  */
```

[Source](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/pixfmt-yuv-planar.html)

### The Multiplanar API

Hardware video decoders frequently DMA Y and UV data to separate memory regions. The V4L2 **multiplanar API** accommodates this by extending the buffer description to list up to `VIDEO_MAX_PLANES` (8) separate planes:

```c
struct v4l2_pix_format_mplane {
    __u32                   width;
    __u32                   height;
    __u32                   pixelformat;      /* V4L2_PIX_FMT_* */
    __u32                   field;            /* V4L2_FIELD_* */
    __u32                   colorspace;       /* V4L2_COLORSPACE_* */
    struct v4l2_plane_pix_format plane_fmt[VIDEO_MAX_PLANES]; /* per-plane */
    __u8                    num_planes;
    __u8                    flags;
    union { __u8 ycbcr_enc; __u8 hsv_enc; };
    __u8                    quantization;     /* V4L2_QUANTIZATION_* */
    __u8                    xfer_func;        /* V4L2_XFER_FUNC_* */
    __u8                    reserved[7];
};

struct v4l2_plane_pix_format {
    __u32   sizeimage;       /* max buffer size for this plane */
    __u32   bytesperline;    /* stride in bytes */
};
```

For `V4L2_PIX_FMT_NV12` with two planes: `plane_fmt[0]` describes the Y plane and `plane_fmt[1]` describes the interleaved UV plane.

### Format Negotiation with VIDIOC_ENUM_FMT / TRY_FMT / S_FMT

The three-step format negotiation idiom for a V4L2 decoder:

```c
#include <linux/videodev2.h>
#include <sys/ioctl.h>

int negotiate_decoder_format(int video_fd, uint32_t width, uint32_t height)
{
    /* Step 1: Enumerate formats the driver accepts on the CAPTURE queue */
    struct v4l2_fmtdesc fmtdesc = {
        .type  = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE,
        .index = 0,
    };
    uint32_t preferred_fmts[] = {
        V4L2_PIX_FMT_P010,   /* 10-bit HDR path */
        V4L2_PIX_FMT_NV12,   /* 8-bit standard path */
    };
    int chosen = -1;

    while (ioctl(video_fd, VIDIOC_ENUM_FMT, &fmtdesc) == 0) {
        for (size_t i = 0; i < sizeof(preferred_fmts)/sizeof(preferred_fmts[0]); i++) {
            if (fmtdesc.pixelformat == preferred_fmts[i] &&
                (chosen == -1 || i < (size_t)chosen))
                chosen = (int)i;
        }
        fmtdesc.index++;
    }
    if (chosen < 0) { errno = EINVAL; return -1; }

    /* Step 2: Try the format — driver adjusts parameters to what it can support */
    struct v4l2_format fmt = {
        .type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE,
    };
    fmt.fmt.pix_mp.width       = width;
    fmt.fmt.pix_mp.height      = height;
    fmt.fmt.pix_mp.pixelformat = preferred_fmts[chosen];
    fmt.fmt.pix_mp.num_planes  = 2;
    if (ioctl(video_fd, VIDIOC_TRY_FMT, &fmt) < 0) {
        perror("VIDIOC_TRY_FMT");
        return -1;
    }

    /* Step 3: Commit the negotiated format */
    if (ioctl(video_fd, VIDIOC_S_FMT, &fmt) < 0) {
        perror("VIDIOC_S_FMT");
        return -1;
    }

    printf("Negotiated: %ux%u fmt=0x%08x planes=%u\n",
           fmt.fmt.pix_mp.width, fmt.fmt.pix_mp.height,
           fmt.fmt.pix_mp.pixelformat, fmt.fmt.pix_mp.num_planes);
    return 0;
}
```

[Source](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/vidioc-enum-fmt.html)

After `VIDIOC_S_FMT`, the decoder produces buffers in the negotiated layout. The `colorspace`, `ycbcr_enc`, `quantization`, and `xfer_func` fields of `v4l2_pix_format_mplane` propagate color space metadata through the V4L2 layer to the DMA-BUF import path.

---

## 7. HDR Metadata in the Display Signal

### SMPTE ST 2086 Static Metadata

**SMPTE ST 2086** (2018) defines the "Mastering Display Color Volume" metadata that describes the display on which the content was color-graded. This static metadata is transmitted once per stream and does not change frame to frame. Fields:

| Field | Type | Units |
|---|---|---|
| `display_primaries[3].x/y` | uint16 | 0.00002 per unit (1.0 = 0xC350 = 50000) |
| `white_point.x/y` | uint16 | 0.00002 per unit |
| `max_display_mastering_luminance` | uint16 | 0.0001 nits per unit (10000 = 1 nit) |
| `min_display_mastering_luminance` | uint16 | 0.0001 nits per unit |
| `max_cll` | uint16 | cd/m², Maximum Content Light Level |
| `max_fall` | uint16 | cd/m², Maximum Frame Average Light Level |

Typical HDR10 master: primaries Rx=0.708, Ry=0.292, Gx=0.170, Gy=0.797, Bx=0.131, By=0.046 (P3-D65 mastering); MaxCLL=1000, MaxFALL=400.

### HDMI Dynamic Range and Mastering InfoFrame

Over HDMI, ST 2086 metadata travels in the **Dynamic Range and Mastering InfoFrame** (type 0x87, version 1, 26 data bytes), standardized in CTA-861-G Section 6.9. The display uses this InfoFrame to tone-map content to its own capabilities.

### DP HDR Metadata SDP

Over DisplayPort, the same SMPTE ST 2086 payload is carried in a Secondary Data Packet (SDP) of type 0x04, defined in VESA DisplayPort 1.4a Section 2.2.5. The data layout is identical to the HDMI InfoFrame payload.

### DRM Representation: `hdr_output_metadata`

The kernel UAPI structure that carries HDR metadata from userspace to the driver is defined in `include/uapi/drm/drm_mode.h`: [Source](https://github.com/torvalds/linux/blob/master/include/uapi/drm/drm_mode.h)

```c
struct hdr_metadata_infoframe {
    __u8  eotf;                         /* 0=SDR, 1=Trad HDR, 2=PQ, 3=HLG */
    __u8  metadata_type;                /* 0 = Static Metadata Type 1 */
    struct {
        __u16 x;                        /* in units of 0.00002 */
        __u16 y;
    } display_primaries[3];             /* R, G, B primaries */
    struct {
        __u16 x;
        __u16 y;
    } white_point;
    __u16 max_display_mastering_luminance;  /* in units of 1 cd/m² */
    __u16 min_display_mastering_luminance;  /* in units of 0.0001 cd/m² */
    __u16 max_cll;                      /* Max Content Light Level */
    __u16 max_fall;                     /* Max Frame Average Light Level */
};

struct hdr_output_metadata {
    __u32 metadata_type;                /* DRM_HDMI_STATIC_METADATA_TYPE1 = 0 */
    union {
        struct hdr_metadata_infoframe hdmi_metadata_type1;
    };
};
```

### Setting HDR Metadata via Atomic Properties

```c
#include <drm/drm_mode.h>
#include <xf86drmMode.h>

int set_hdr_metadata(int drm_fd, uint32_t connector_id,
                     uint32_t hdr_prop_id,
                     uint16_t max_cll, uint16_t max_fall)
{
    struct hdr_output_metadata meta = {
        .metadata_type = 0,  /* DRM_HDMI_STATIC_METADATA_TYPE1 */
        .hdmi_metadata_type1 = {
            .eotf          = 2,  /* SMPTE ST 2084 PQ */
            .metadata_type = 0,
            /* P3-D65 mastering primaries in units of 0.00002 */
            .display_primaries = {
                [0] = { .x = 35400, .y = 14600 }, /* R: 0.708, 0.292 */
                [1] = { .x =  8500, .y = 39850 }, /* G: 0.170, 0.797 */
                [2] = { .x =  6550, .y =  2300 }, /* B: 0.131, 0.046 */
            },
            .white_point = { .x = 15635, .y = 16450 }, /* D65 */
            .max_display_mastering_luminance = 1000,    /* 1000 cd/m² */
            .min_display_mastering_luminance = 50,      /* 0.005 cd/m² */
            .max_cll  = max_cll,
            .max_fall = max_fall,
        },
    };

    uint32_t blob_id = 0;
    if (drmModeCreatePropertyBlob(drm_fd, &meta, sizeof(meta), &blob_id) < 0) {
        perror("drmModeCreatePropertyBlob");
        return -1;
    }

    drmModeAtomicReqPtr req = drmModeAtomicAlloc();
    drmModeAtomicAddProperty(req, connector_id, hdr_prop_id, blob_id);

    int ret = drmModeAtomicCommit(drm_fd, req,
                                  DRM_MODE_ATOMIC_ALLOW_MODESET, NULL);
    drmModeAtomicFree(req);
    drmModeDestroyPropertyBlob(drm_fd, blob_id);
    return ret;
}
```

[Source](https://github.com/mpv-player/mpv/commit/9c89e94032e73869c0004adbf1a497c2f1084a4b)

The driver (amdgpu, i915, nouveau) reads the blob from the atomic state, builds the InfoFrame, and programs the display engine's InfoFrame buffer at the start of each frame.

---

## 8. AVI InfoFrame Color Encoding

### Structure Overview

The **HDMI Auxiliary Video Information (AVI) InfoFrame** is defined in CTA-861-G Section 6.4. Type code 0x82, version 2, length 13 data bytes. It is transmitted in every HDMI active video frame (repeated every VBI interval). The AVI InfoFrame informs the sink about color space, colorimetry, quantization range, and VIC (Video Identification Code) of the active video stream. [Source](https://thedigitallifestyle.com/w/2010/03/making-sense-out-of-hdmi-1-4-performance-and-the-ceas-861-infoframes-installment-027/)

### Key Bit Fields

**Data Byte 1:**
- Bits 6:5 — `Y1:Y0` Color Space: `00`=RGB, `01`=YCbCr 4:2:2, `10`=YCbCr 4:4:4, `11`=YCbCr 4:2:0 (added in HDMI 2.0 via CTA-861-F)
- Bit 4 — `A0` Active Format Information Present
- Bits 3:2 — `B1:B0` Bar Data Present
- Bits 1:0 — `S1:S0` Scan Information

**Data Byte 2:**
- Bits 7:6 — `C1:C0` Colorimetry: `00`=none/unknown, `01`=BT.601, `10`=BT.709, `11`=extended (see EC2–EC0)
- Bits 5:4 — `M1:M0` Picture Aspect Ratio
- Bits 3:0 — `R3:R0` Active Format Aspect Ratio

**Data Byte 3:**
- Bit 7 — `ITC` IT Content
- Bits 6:4 — `EC2:EC0` Extended Colorimetry (when C1:C0 = `11`):
  - `100` = ITU-R BT.2020 YCbCr
  - `110` = DCI-P3 RGB (via additional_colorimetry byte in HDMI 2.1)
- Bits 3:2 — `Q1:Q0` RGB Quantization Range: `00`=default, `01`=limited, `10`=full
- Bits 1:0 — `SC1:SC0` Non-Uniform Picture Scaling

**Data Byte 5** (VIC): Video Identification Code. VIC 97 = 4K@60Hz, VIC 96 = 4K@50Hz.

**YCC Quantization** is separate from RGB quantization and encoded in data byte 5 bits 7:6 (`YQ1:YQ0`): `00`=limited range, `01`=full range.

### Linux struct hdmi_avi_infoframe

The kernel represents the AVI InfoFrame as: [Source](https://github.com/torvalds/linux/blob/master/include/linux/hdmi.h)

```c
struct hdmi_avi_infoframe {
    enum hdmi_infoframe_type type;      /* HDMI_INFOFRAME_TYPE_AVI = 0x82 */
    unsigned char version;              /* 2 */
    unsigned char length;               /* 13 */
    enum hdmi_colorspace colorspace;    /* HDMI_COLORSPACE_RGB / _YUV422 etc. */
    enum hdmi_scan_mode scan_mode;
    enum hdmi_colorimetry colorimetry;  /* HDMI_COLORIMETRY_ITU_709 etc. */
    enum hdmi_picture_aspect picture_aspect;
    enum hdmi_active_aspect active_aspect;
    enum hdmi_extended_colorimetry extended_colorimetry;
    enum hdmi_quantization_range quantization_range;
    enum hdmi_nups nups;
    unsigned char video_code;           /* VIC */
    enum hdmi_ycc_quantization_range ycc_quantization_range;
    enum hdmi_content_type content_type;
    /* ... bar info fields ... */
};
```

### How amdgpu Fills It

The amdgpu display core fills the AVI InfoFrame in `drivers/gpu/drm/amd/display/modules/hdcp/` and the dc layer. The helper API:

```c
/* Initialize and fill AVI InfoFrame for YCbCr 4:2:0 BT.2020 limited range */
struct hdmi_avi_infoframe frame;
hdmi_avi_infoframe_init(&frame);

frame.colorspace            = HDMI_COLORSPACE_YUV420;
frame.colorimetry           = HDMI_COLORIMETRY_EXTENDED;
frame.extended_colorimetry  = HDMI_EXTENDED_COLORIMETRY_BT2020;
frame.quantization_range    = HDMI_QUANTIZATION_RANGE_DEFAULT;
frame.ycc_quantization_range = HDMI_YCC_QUANTIZATION_RANGE_LIMITED;
frame.video_code            = 97;   /* VIC 97: 3840x2160 @ 60Hz */
frame.picture_aspect        = HDMI_PICTURE_ASPECT_16_9;

/* Pack into raw bytes for hardware FIFO */
uint8_t buf[HDMI_INFOFRAME_SIZE(AVI)];
hdmi_avi_infoframe_pack(&frame, buf, sizeof(buf));
/* ... write buf to DC InfoFrame register set ... */
```

[Source](https://github.com/torvalds/linux/blob/master/include/linux/hdmi.h)

The `hdmi_avi_infoframe_pack()` function handles checksum computation and the raw byte packing per the CTA-861 wire format. The DC (Display Core) hardware then embeds the raw bytes into the HDMI packet memory to be transmitted in every VBI.

---

## 9. Format Negotiation Pipeline

### The Full Stack from Decoder to Scanout

A video frame follows this chain before reaching the display panel:

```text
VA-API / V4L2 decoder
  │  Exports DMA-BUF fd + drm_fourcc (NV12 or P010)
  │  + modifier (LINEAR or AFBC/DCC/CCS for tiled output)
  ▼
Wayland client (video player / MPV)
  │  Imports DMA-BUF, wraps as wl_buffer via zwp_linux_dmabuf_v1
  │  Format+modifier advertised by compositor in feedback events
  ▼
Wayland compositor (e.g., KWin, Sway, Weston)
  │  Checks DRM overlay plane IN_FORMATS for (format, modifier) support
  │  Attempts atomic TEST_ONLY commit with overlay plane
  ▼
KMS atomic commit (DRM_MODE_ATOMIC_TEST_ONLY)
  │  drm_plane_check_pixel_format() validates format against plane caps
  │  Driver's .format_mod_supported() validates (fmt, modifier) pair
  ▼
If TEST passes → REAL atomic commit → hardware overlay scanout
If TEST fails  → GPU blit fallback (compositor renders video into XRGB8888)
```

### zwp_linux_dmabuf_v1 Format Negotiation

The compositor advertises supported (format, modifier) pairs to clients through `zwp_linux_dmabuf_v1`. Starting from protocol version 4, this uses the feedback mechanism: `zwp_linux_dmabuf_feedback_v1` events deliver a list of `(drm_format, drm_modifier)` pairs that the compositor can accept without needing a blit. [Source](https://wayland.app/protocols/linux-dmabuf-v1)

A VA-API video player receives this list and configures its VA-API surface format to match one of the advertised pairs. If `DRM_FORMAT_P010` with modifier `I915_FORMAT_MOD_Y_TILED_CCS` appears in the list, the player can export that exact surface layout and the compositor can hand it to KMS without touching the pixels.

### Atomic TEST_ONLY and Format Validation

```c
/* Compositor: probe whether the overlay plane accepts NV12 with a modifier */
drmModeAtomicReqPtr req = drmModeAtomicAlloc();

/* Attach the video framebuffer to the overlay plane */
drmModeAtomicAddProperty(req, overlay_plane_id,
                         prop_fb_id, fb_id);
drmModeAtomicAddProperty(req, overlay_plane_id,
                         prop_crtc_id, crtc_id);
/* ... add SRC_W, SRC_H, CRTC_W, CRTC_H ... */

int ret = drmModeAtomicCommit(drm_fd, req,
                              DRM_MODE_ATOMIC_TEST_ONLY, NULL);
if (ret == 0) {
    /* Plane accepts this format+modifier combination — proceed with real commit */
} else {
    /* Fallback: blit to XRGB8888 primary plane */
}
drmModeAtomicFree(req);
```

[Source](https://dri.freedesktop.org/docs/drm/gpu/amdgpu/display/mpo-overview.html)

### Display Engine CSC and EOTF

When the overlay plane accepts NV12 or P010, the display engine's Color Space Converter (CSC) converts YCbCr to RGB using the BT.709 or BT.2020 matrix before the pixel reaches the panel. On amdgpu, the DC layer programs the CSC matrix in the DPP (Display Pipe and Plane) unit. For HDR10 content (P010 + PQ EOTF), the DC layer applies the inverse PQ EOTF in the OGAM (Output Gamma) block to convert from PQ code values to linear light before the panel's own EOTF re-applies. [Source](https://docs.kernel.org/gpu/amdgpu/display/index.html)

When the overlay path fails, the GPU compositor renders the YCbCr video into the XRGB8888 primary plane using a shader that applies the color matrix and tone-mapping. This is the "blit fallback" path and adds a full GPU draw call per video frame.

---

## 10. Worked Example — 4K HDR10 Video Playback

This section traces a single 3840×2160 HDR10 frame from a hardware HEVC decoder all the way to the photons leaving the display.

### Step 1: VA-API Decode → DRM_FORMAT_P010

An HEVC Main 10 decoder (e.g., AMD VCN, Intel Quick Sync, or V4L2-based Mediatek VDEC) produces output in P010 format, typically with a hardware-specific tiling modifier. On amdgpu (RX 6000 / RX 7000 series), the VCN decoder outputs `DRM_FORMAT_P010` with the `AMD_FMT_MOD` modifier family (encodes DCC, bank swizzle, pipe interleave). [Source](https://www.kernel.org/doc/html/v5.7/gpu/afbc.html)

### Step 2: DMA-BUF Export and Wayland Buffer Import

The VA-API library exports the decode surface as a DMA-BUF file descriptor, accompanied by the DRM fourcc code and 64-bit modifier. The video player (e.g., MPV or GStreamer) wraps this in a `wl_buffer` via `zwp_linux_dmabuf_v1`:

```c
struct zwp_linux_buffer_params_v1 *params =
    zwp_linux_dmabuf_v1_create_params(dmabuf_interface);

/* Luma plane */
zwp_linux_buffer_params_v1_add(params, luma_dmabuf_fd, 0 /*plane*/,
                                luma_offset, luma_stride,
                                modifier_hi, modifier_lo);
/* Chroma plane */
zwp_linux_buffer_params_v1_add(params, chroma_dmabuf_fd, 1 /*plane*/,
                                chroma_offset, chroma_stride,
                                modifier_hi, modifier_lo);

struct wl_buffer *buf =
    zwp_linux_buffer_params_v1_create_immed(params,
        3840, 2160,
        DRM_FORMAT_P010,
        ZWP_LINUX_BUFFER_PARAMS_V1_FLAGS_NONE);
```

[Source](https://wayland.app/protocols/linux-dmabuf-v1)

### Step 3: KMS Overlay Plane Atomic Commit

The compositor probes the KMS overlay plane with `DRM_MODE_ATOMIC_TEST_ONLY`. If accepted, it builds the real atomic request and adds the HDR metadata property:

```c
/* Build hdr_output_metadata for HDR10 content */
struct hdr_output_metadata hdr_meta = {
    .metadata_type = 0,          /* DRM_HDMI_STATIC_METADATA_TYPE1 */
    .hdmi_metadata_type1 = {
        .eotf          = 2,      /* PQ — SMPTE ST 2084 */
        .metadata_type = 0,
        .display_primaries = {
            /* P3-D65 mastering display, values × 50000 */
            [0] = { .x = 35400, .y = 14600 },  /* R */
            [1] = { .x =  8500, .y = 39850 },  /* G */
            [2] = { .x =  6550, .y =  2300 },  /* B */
        },
        .white_point              = { .x = 15635, .y = 16450 },
        .max_display_mastering_luminance = 10000, /* 1000 nits × 10 */
        .min_display_mastering_luminance = 500,   /* 0.05 nits × 10000 */
        .max_cll  = 1000,   /* MaxCLL  = 1000 cd/m² */
        .max_fall = 400,    /* MaxFALL =  400 cd/m² */
    },
};

uint32_t hdr_blob = 0;
drmModeCreatePropertyBlob(drm_fd, &hdr_meta, sizeof(hdr_meta), &hdr_blob);

drmModeAtomicReqPtr req = drmModeAtomicAlloc();
/* Overlay plane: P010 video buffer */
drmModeAtomicAddProperty(req, overlay_plane_id, prop_fb_id,     video_fb_id);
drmModeAtomicAddProperty(req, overlay_plane_id, prop_crtc_id,   crtc_id);
drmModeAtomicAddProperty(req, overlay_plane_id, prop_src_w,     3840 << 16);
drmModeAtomicAddProperty(req, overlay_plane_id, prop_src_h,     2160 << 16);
drmModeAtomicAddProperty(req, overlay_plane_id, prop_crtc_w,    3840);
drmModeAtomicAddProperty(req, overlay_plane_id, prop_crtc_h,    2160);
/* Connector: HDR metadata blob */
drmModeAtomicAddProperty(req, connector_id, prop_hdr_output_metadata, hdr_blob);
/* Connector: colorspace */
drmModeAtomicAddProperty(req, connector_id, prop_colorspace,
                         DRM_MODE_COLORIMETRY_BT2020_YCC);

drmModeAtomicCommit(drm_fd, req, DRM_MODE_ATOMIC_ALLOW_MODESET, NULL);
```

### Step 4: amdgpu Sends HDMI InfoFrames

On `drmModeAtomicCommit`, the amdgpu DC layer constructs two HDMI InfoFrames:

**AVI InfoFrame (0x82)** — signals to the HDMI sink:
- `Y1:Y0 = 11` → YCbCr 4:2:0
- `C1:C0 = 11`, `EC2:EC0 = 100` → Extended Colorimetry = BT.2020 YCbCr
- `YQ1:YQ0 = 00` → YCC limited range (16–235 luma, 16–240 chroma)
- VIC = 97 (3840×2160 @ 60 Hz)

**HDR Dynamic Range and Mastering InfoFrame (0x87, version 1)** — carries the 26-byte SMPTE ST 2086 payload with the `max_cll=1000`, `max_fall=400` and P3-D65 primaries filled in from `hdr_output_metadata.hdmi_metadata_type1`.

The HDMI sink (TV or monitor) reads the HDR InfoFrame and applies its internal tone mapping algorithm, mapping the 0–10,000 cd/m² PQ signal to its own peak luminance. A 600-nit panel applies more aggressive roll-off than a 1,000-nit panel; the MaxCLL and MaxFALL values allow the display's tone mapper to optimize for the content's actual light level distribution.

### Step 5: Display Engine Processing

Inside the amdgpu DC, the DPP unit for the P010 overlay plane performs:

1. **Deswizzle / untile** the AMD_FMT_MOD tiled buffer to raster order.
2. **CSC matrix** — BT.2020 YCbCr → linear light RGB using the BT.2020 matrix coefficients.
3. **OGAM/DEGAM** — apply the inverse PQ EOTF (decode PQ code values to linear scene light). For a display-referred path the display's panel handles the final forward EOTF.
4. **Blending** — blend with the cursor plane and any overlay OSD in linear light.
5. **Output encoding** — re-apply the panel's EOTF in reverse for signal encoding, or in direct HDMI HDR mode, re-apply PQ before the signal exits the chip.

This processing is invisible to userspace; it is configured entirely by the DC layer when it programs the `HDR_OUTPUT_METADATA` and `Colorspace` connector state.

---

## Integrations

- **Ch2 (KMS Planes):** The primary and overlay planes that scan out pixel data accept only the DRM fourcc formats listed in their `IN_FORMATS` blob property; this chapter defines what those codes mean and how to enumerate them.
- **Ch3 (HDR KMS Properties):** The `HDR_OUTPUT_METADATA` connector property and `Colorspace` plane property introduced here are the KMS primitives through which HDR color management flows; Ch3 covers the full set of HDR-relevant KMS properties.
- **Ch26 (VA-API):** VA-API surfaces use NV12 (8-bit) and P010 (10-bit HDR) as their primary export formats; Ch26 covers the VA-API surface creation and DMA-BUF export sequence that feeds the pipeline described in this chapter.
- **Ch60 (Codec Compression):** HEVC, AV1, and VP9 codec compression is entirely separate from the pixel wire format; Ch60 explains entropy coding, DCT, and motion compensation. This chapter picks up where Ch60 leaves off — at the reconstructed pixel in the decoded frame buffer.
- **Ch74 (HDR Wide Color Gamut Application Side):** Ch74 addresses how applications request HDR surfaces via Vulkan (`VK_EXT_swapchain_colorspace`) and EGL. The BT.2020 primaries and PQ transfer function discussed here are the wire-level expression of the colorimetry Ch74 covers at the API level.
- **Ch101 (Color Science — ICC Profiles):** ICC profiles describe input and output device color characteristics in matrix/LUT form. The primaries in the `hdr_output_metadata` blob and the AVI InfoFrame EC bits represent a simplified subset of the same colorimetric information ICC profiles carry at fuller precision.
- **Ch142 (V4L2 Format Negotiation API):** Ch142 provides a broader treatment of the V4L2 ioctl interface, buffer management, and request API. The `VIDIOC_ENUM_FMT` / `VIDIOC_TRY_FMT` / `VIDIOC_S_FMT` sequence introduced in Section 6 of this chapter is a specific application of the framework Ch142 describes in full.
- **Ch158 (HDR Display Color Management):** Ch158 covers high-level HDR color management policy — tone mapping curves, gamut mapping, ICC pipeline integration in Wayland compositors. This chapter provides the low-level signal encoding mechanisms (InfoFrames, quantization range, PQ EOTF signaling) that Ch158's policies ultimately drive.
- **Ch162 (AFBC/DCC/CCS/UBWC Modifiers):** The DRM format modifier system, which expresses tiling and compression schemes for NV12 and P010 surfaces, is detailed in Ch162. The `AMD_FMT_MOD` modifier family mentioned in the worked example in this chapter is one instance of the broader modifier taxonomy Ch162 covers.

---

## References

- [ITU-R BT.709 — Basic parameter values for the HDTV standard](https://www.itu.int/rec/R-REC-BT.709)
- [ITU-R BT.601 — Studio encoding parameters of digital television for standard 4:3 and widescreen 16:9 aspect ratios](https://www.itu.int/rec/R-REC-BT.601)
- [ITU-R BT.2020 — Parameter values for ultra-high definition television systems](https://www.itu.int/rec/R-REC-BT.2020)
- [SMPTE ST 2084:2014 — High Dynamic Range EOTF of Mastering Reference Displays](https://pub.smpte.org/latest/st2084/st2084-2014.pdf)
- [ARIB STD-B67:2015 — Essential Parameter Values for the Extended Image Dynamic Range Television (EIDRTV) System for Programme Production](https://www.arib.or.jp/english/html/overview/doc/2-STD-B67v1_0.pdf)
- [Linux kernel drm_fourcc.h](https://github.com/torvalds/linux/blob/master/include/uapi/drm/drm_fourcc.h)
- [Linux kernel include/linux/hdmi.h](https://github.com/torvalds/linux/blob/master/include/linux/hdmi.h)
- [Linux kernel include/uapi/drm/drm_mode.h](https://github.com/torvalds/linux/blob/master/include/uapi/drm/drm_mode.h)
- [V4L2 Planar YUV Formats — Linux kernel docs](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/pixfmt-yuv-planar.html)
- [AMDGPU Display Core — Linux kernel docs](https://docs.kernel.org/gpu/amdgpu/display/index.html)
- [AMDGPU Multiplane Overlay Overview](https://docs.kernel.org/6.1/gpu/amdgpu/display/mpo-overview.html)
- [DRM KMS — Kernel Mode Setting documentation](https://docs.kernel.org/gpu/drm-kms.html)
- [Wayland linux-dmabuf-v1 protocol](https://wayland.app/protocols/linux-dmabuf-v1)
- [drm/connector: HDMI "Broadcast RGB" property patches — dri-devel](https://www.mail-archive.com/dri-devel@lists.freedesktop.org/msg490843.html)
- [mpv commit: DRM HDR metadata sending](https://github.com/mpv-player/mpv/commit/9c89e94032e73869c0004adbf1a497c2f1084a4b)
- [10-bit and 16-bit YUV Video Formats — Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/medfound/10-bit-and-16-bit-yuv-video-formats)
- [HDR in Linux: Part 2 — Jeremy Cline](https://www.jcline.org/blog/fedora/graphics/hdr/2021/06/28/hdr-in-linux-p2.html)
- [ARM Framebuffer Compression (AFBC) — Linux kernel docs](https://www.kernel.org/doc/html/v5.7/gpu/afbc.html)

## Roadmap

### Near-term (6–12 months)
- The DRM `colorspace` connector property is being extended to carry ITU-R BT.2100 ICtCp colorimetry alongside the existing BT.2020 YCC entries, enabling Dolby Vision display-referred signaling without out-of-band metadata blobs; patches are in review on dri-devel as of early 2026.
- The `zwp_linux_dmabuf_v1` protocol version 5 work aims to let compositors advertise per-surface format/modifier feedback for cursor and overlay planes separately, reducing the need for TEST_ONLY atomic probing loops in video players such as MPV and GStreamer.
- HEVC and AV1 hardware decoders on Intel Xe (Battlemage) and AMD RDNA 4 are gaining native P012 (12-bit 4:2:0) DMA-BUF export, which will allow Dolby Vision Profile 5 content to travel the NV12/P010 overlay path without a 16-bit upcast.
- The `hdr_output_metadata` UAPI is being amended to carry SMPTE ST 2094-10 dynamic metadata frames alongside the existing ST 2086 static payload, enabling per-scene tone-map hints to HDR10+ capable displays over HDMI 2.1.

### Medium-term (1–3 years)
- HDMI 2.1a Source-Based Tone Mapping (SBTM) requires the kernel to expose a new DRM connector property for negotiating SBTM capability and sending the corresponding InfoFrame; Mesa/KWin coordination on this path is in early design as of 2026.
- The V4L2 multiplanar API is expected to grow a formal `V4L2_PIX_FMT_P016` negotiation path for 16-bit-per-component decode output, driven by AV1 12-bit profile 2 content from next-generation streaming services.
- A unified `drm_colorop` pipeline object — currently being prototyped by AMD and Red Hat engineers — will allow userspace to specify a composed sequence of CSC, 1D LUT, and 3D LUT operations on a per-plane basis, replacing the current driver-internal hardwiring of BT.709/BT.2020 matrix selection.
- DisplayPort 2.1 UHBR20 tunneled over USB4 v2 requires the kernel's DP AUX and link-training code to recognize and negotiate new YCbCr 4:2:0 modes at 8K@120Hz, influencing the `IN_FORMATS` plane property content for upcoming integrated GPU platforms.

### Long-term
- As displays with native BT.2020 and DCI-P3 primaries become mainstream, the kernel's AVI InfoFrame construction path may adopt ITU-R BT.2100 ICtCp as the primary signaling colorimetry for HDR content, replacing the current BT.2020 YCbCr path and enabling more perceptually uniform tone mapping at the sink.
- The convergence of video decode, display, and AI super-resolution (e.g., AMD FSR, Intel XeSS at the kernel/firmware boundary) may require new DRM plane properties that express upscaling algorithm selection and sharpness parameters, extending the current format-negotiation pipeline beyond raw pixel format into processing intent.
- Long-term standardization of a royalty-free open HDR dynamic metadata format (potentially an AOM-driven successor to HDR10+) could introduce new HDMI/DP InfoFrame types and corresponding DRM UAPI structures that parallel the current `hdr_output_metadata` blob design.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
