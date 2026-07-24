# Chapter 158: HDR Display Hardware and Signaling on Linux

**Target audiences**: Display engineers and driver developers integrating HDR metadata signaling via HDMI/DisplayPort; compositor developers who need to understand how display hardware HDR capability is discovered and programmed; systems engineers building end-to-end HDR pipelines from display detection through to wire signaling.

> **Scope:** This chapter focuses exclusively on **HDR display hardware signaling**:
>
> - **HDMI/DisplayPort wire protocols** — HDR DRM InfoFrames and VSC SDP packet construction and transmission
> - **EDID capability parsing via `libdisplay-info`** — discovering display HDR support via CTA-861 extension blocks
> - **`drm_hdr_output_metadata` kernel structure** — carries mastering display metadata from compositor to driver
> - **Local dimming hardware** — OLED and MiniLED backlight control interfaces in the kernel
> - **VESA DisplayHDR certification** — hardware performance tiers and guarantees for driver developers
>
> **Ch74** is the authoritative chapter for the end-to-end KMS software color pipeline (`DEGAMMA_LUT`, `CTM`, `GAMMA_LUT`, per-plane `drm_colorop` chains, `wp_color_management_v1`, tone-mapping algorithms, and compositor HDR architecture). **Ch101** covers ICC profile internals and color science theory. This chapter bridges the gap between software pipeline decisions (ch74) and what actually travels on the display cable.

---

## Table of Contents

1. [Introduction](#introduction)
   - [1.1 What is HDR (High Dynamic Range)?](#11-what-is-hdr-high-dynamic-range)
   - [1.2 What is EDID and the CTA-861 HDR Capability Block?](#12-what-is-edid-and-the-cta-861-hdr-capability-block)
   - [1.3 What is drm_hdr_output_metadata?](#13-what-is-drm_hdr_output_metadata)
2. [HDR Fundamentals and Standards](#hdr-fundamentals-and-standards)
3. [HDMI HDR Signaling](#hdmi-hdr-signaling)
4. [DisplayPort HDR Descriptor](#displayport-hdr-descriptor)
5. [libdisplay-info: EDID and HDR Capability Parsing](#libdisplay-info-edid-and-hdr-capability-parsing)
6. [EDID HDR Capability: CTA-861 Data Block Internals](#edid-hdr-capability-cta-861-data-block-internals)
7. [OLED and MiniLED Local Dimming](#oled-and-miniled-local-dimming)
8. [The drm_hdr_output_metadata Kernel Structure](#the-drm_hdr_output_metadata-kernel-structure)
9. [Compositor-to-KMS HDR Metadata Flow](#compositor-to-kms-hdr-metadata-flow)
10. [HDR Certification: VESA DisplayHDR Tiers](#hdr-certification-vesa-displayhdr-tiers)
11. [Integrations](#integrations)

---

## Introduction

High Dynamic Range (HDR) displays — capable of peak luminances of 400–10,000 nits and wider colour gamuts (DCI-P3, Rec. 2020) — have become common on desktop monitors and TVs. Getting HDR to work on Linux requires both a software pipeline (covered in Ch74) and correct hardware signaling from the kernel to the display over HDMI or DisplayPort. These two layers are distinct: a compositor could set up a perfect PQ-encoded framebuffer with the right KMS LUT chain, but if the kernel does not send the correct HDR InfoFrame to the display, the panel will either misinterpret the signal or refuse to enter HDR mode.

This chapter traces the hardware signaling path: how the kernel discovers that a display supports HDR (via EDID CTA-861 extension blocks), how the `drm_hdr_output_metadata` structure carries mastering metadata from compositor to driver, how HDMI 2.x DRM InfoFrames and DisplayPort VSC SDP packets are constructed and transmitted, and what local dimming hardware the kernel can control. It also covers the VESA DisplayHDR certification framework so that driver developers understand the hardware guarantees they are targeting.

[Source: drm HDR docs](https://www.kernel.org/doc/html/latest/gpu/drm-kms.html#hdr-metadata) | [Source: LWN HDR patch series](https://lwn.net/Articles/783575/)

### 1.1 What is HDR (High Dynamic Range)?

High Dynamic Range refers to display technology capable of reproducing a significantly wider range of luminance values than Standard Dynamic Range (SDR) content. A conventional SDR display operates at a peak of roughly 100 cd/m² (nits) with a minimum of approximately 0.1 nit; HDR displays target peaks from 400 nit (the entry-level VESA DisplayHDR 400 tier) to 10,000 nit (the ceiling of SMPTE ST 2084), and minimum blacks below 0.001 nit on OLED panels. HDR content also uses wider colour gamuts — DCI-P3 and BT.2020 — compared to the sRGB/BT.709 gamut of SDR.

On Linux, HDR support is a two-layer problem. The software pipeline must produce correctly encoded pixel data using an appropriate transfer function (PQ for HDR10, HLG for broadcast HDR), and the kernel must simultaneously signal the display over the interface protocol — HDMI or DisplayPort — that HDR content is being delivered, along with the mastering parameters that tell the panel how to interpret it. Without the hardware signal, a display defaults to SDR interpretation even if the framebuffer contains PQ-encoded data. This chapter covers the hardware signaling half; the KMS software pipeline (LUT chains, tone mapping, compositor colour management) is covered in Ch74.

### 1.2 What is EDID and the CTA-861 HDR Capability Block?

The Extended Display Identification Data (EDID) is a standardised binary structure stored in non-volatile memory on every HDMI and DisplayPort display and read by the kernel over I²C (DDC) during connector hotplug detection. The base EDID 1.4 block (128 bytes) describes basic display attributes — resolution, refresh rate, physical dimensions — but HDR capability is declared in a 128-byte extension block defined by the Consumer Technology Association standard CTA-861-H.

Within the CTA-861 extension block, a Colorimetry Data Block and an HDR Static Metadata Data Block together specify which transfer functions the display supports (SDR, HDR10 PQ, HLG), what its peak and minimum luminance are, and whether it accepts BT.2020 wide-gamut content. The kernel parses these blocks via the `libdisplay-info` library (or the legacy `drm_edid` helpers for older drivers). The parsed values determine which EOTF code and colorimetry field the kernel places in the HDMI DRM InfoFrame or DisplayPort VSC SDP when the compositor requests HDR output. Misreading or ignoring the CTA-861 HDR block is the most common reason a display fails to enter HDR mode despite correct software configuration above the driver layer.

### 1.3 What is drm_hdr_output_metadata?

`drm_hdr_output_metadata` is the kernel UAPI data structure that carries HDR mastering display metadata from a Wayland compositor (or any DRM userspace client) into the KMS driver. Compositors populate this structure with the SMPTE ST 2086 mastering display colour volume — the CIE xy chromaticity primaries, white point, and minimum and maximum luminance of the mastering monitor — plus the MaxCLL (Maximum Content Light Level) and MaxFALL (Maximum Frame-Average Light Level) values defined by CTA-861-H.

The compositor attaches the populated structure as an opaque blob to the DRM connector property named `HDR_OUTPUT_METADATA`. When the kernel commits the display state, the display driver extracts this blob from `connector_state->hdr_output_metadata` and uses it to fill and transmit the HDMI DRM InfoFrame (type 0x87) or the DisplayPort VSC SDP HDR metadata payload to the physical panel. The structure is defined in `include/uapi/drm/drm_mode.h` and is the single handoff point between software pipeline decisions and the wire protocol. Correctly translating its luminance fields — which use units of 0.0001 cd/m² for minimum luminance — into the InfoFrame wire format without unit errors is a recurring source of driver bugs, as discussed in the Luminance Encoding Conventions section below.

---

## HDR Fundamentals and Standards

### Transfer Functions

| Standard | Transfer Function | Peak Luminance | Gamut |
|---|---|---|---|
| SDR (BT.709) | BT.1886 (gamma 2.4) | 100 nit | sRGB |
| HDR10 | PQ (SMPTE ST 2084) | 1000–10000 nit | BT.2020 |
| HLG | Hybrid Log-Gamma (ITU-R BT.2100) | ~1000 nit | BT.2020 |
| Dolby Vision | PQ + proprietary metadata | 4000–10000 nit | BT.2020 |
| scRGB | Linear (FP16) | Unbounded | sRGB (extended) |

**PQ (Perceptual Quantizer)**: absolute-luminance EOTF standardised as SMPTE ST 2084; 0.0 = 0 nit, 1.0 = 10,000 nit. HDR10 uses PQ in a 10-bit BT.2020 frame. The display applies the EOTF to convert encoded values back to light.

**HLG (Hybrid Log-Gamma)**: scene-referred, broadcast-backward-compatible. No mastering display primaries are carried in the signal; the display infers appropriate luminance from the system gamma defined in ITU-R BT.2100.

**HDR10 metadata standards**:
- **SMPTE ST 2086** defines mastering display colour volume metadata (primaries, white point, max/min luminance).
- **CTA-861-H** defines how static metadata is carried in the HDMI Dynamic Range and Mastering (DRM) InfoFrame and how display capability is reported in EDID.
- **SMPTE ST 2094-40** defines per-frame dynamic metadata (HDR10+), carried as a Vendor-Specific InfoFrame on HDMI 2.0+.

### Luminance Encoding Conventions

A recurring source of bugs in HDR metadata code is the inconsistency of units across structures. Understanding these encodings is critical before reading or writing any HDR metadata structure:

| Field | Units | Example: 1000-nit display |
|---|---|---|
| `max_display_mastering_luminance` (DRM InfoFrame, `hdr_metadata_infoframe`) | cd/m² (1:1 integer) | 1000 |
| `min_display_mastering_luminance` (same struct) | 0.0001 cd/m² | 50 (= 0.005 nit) |
| `max_cll`, `max_fall` (same struct) | cd/m² (1:1 integer) | 1000, 400 |
| EDID Byte 3 `Desired Content Max Luminance` | Encoded: `50 × 2^(code/32)` | 0xA0 → 1600 nit |
| EDID Byte 5 `Desired Content Min Luminance` | Relative to max × 0.01% | code-dependent |
| `VAHdrMetaDataHDR10.max_display_mastering_luminance` | 0.0001 cd/m² | 10,000,000 |
| `VkHdrMetadataEXT.maxLuminance` | float cd/m² | 1000.0f |

Note in particular the unit mismatch between `hdr_metadata_infoframe` (min luminance in 0.0001 cd/m²) and Vulkan's `VkHdrMetadataEXT` (all luminances in float cd/m²). Compositors bridging these APIs must convert explicitly.

---

## HDMI HDR Signaling

### InfoFrame Types and Wire Transport

HDMI carries auxiliary metadata in **InfoFrame** packets transmitted during the vertical blanking period. The relevant InfoFrame types for HDR are defined in `include/linux/hdmi.h`:

```c
/* include/linux/hdmi.h — HDMI InfoFrame type codes */
enum hdmi_infoframe_type {
    HDMI_INFOFRAME_TYPE_VENDOR = 0x81, /* Vendor Specific InfoFrame (VSIF) */
    HDMI_INFOFRAME_TYPE_AVI    = 0x82, /* Auxiliary Video Information */
    HDMI_INFOFRAME_TYPE_SPD    = 0x83, /* Source Product Descriptor */
    HDMI_INFOFRAME_TYPE_AUDIO  = 0x84, /* Audio InfoFrame */
    HDMI_INFOFRAME_TYPE_DRM    = 0x87, /* Dynamic Range and Mastering */
};
```

[Source: linux/include/linux/hdmi.h](https://github.com/torvalds/linux/blob/master/include/linux/hdmi.h)

For HDR10, two InfoFrames work together:

1. **AVI InfoFrame (type 0x82)**: Carries the colourimetry field (extended to signal BT.2020 via `Extended Colorimetry = EC2` and `Colorimetry = C2`), and the quantisation range. This tells the display what colour space the pixels are encoded in.

2. **DRM InfoFrame (type 0x87)**: Carries the SMPTE ST 2086 mastering display metadata and MaxCLL/MaxFALL. This is defined in CTA-861-H as the HDR Dynamic Range and Mastering InfoFrame.

For HDR10+ dynamic metadata, the **HDMI Forum Vendor-Specific InfoFrame (HF-VSIF)** carries the SMPTE ST 2094-40 payload as a Vendor Specific InfoFrame with the HDMI Forum OUI (0xC45DD8). The HF-VSIF payload can be up to 4,095 bytes — far larger than the static DRM InfoFrame — enabling per-frame tone-mapping curves to be signaled. Linux kernel support for assembling the SMPTE ST 2094-40 dynamic metadata payload in the HF-VSIF is not yet merged upstream as of Linux 6.x; HDR10+ in practice requires out-of-tree or vendor-specific support.

### The HDMI DRM InfoFrame Kernel Structure

The `hdmi_drm_infoframe` struct mirrors the wire format of the type 0x87 InfoFrame:

```c
/* include/linux/hdmi.h — HDMI DRM InfoFrame (CTA-861-H, type 0x87) */
struct hdmi_drm_infoframe {
    enum hdmi_infoframe_type type;    /* = HDMI_INFOFRAME_TYPE_DRM (0x87) */
    unsigned char version;            /* = 1 */
    unsigned char length;             /* payload length in bytes */
    enum hdmi_eotf eotf;             /* EOTF: SDR/HDR/PQ/HLG */
    enum hdmi_metadata_type metadata_type; /* = HDMI_STATIC_METADATA_TYPE1 */
    struct { u16 x, y; } display_primaries[3]; /* CIE xy × 50000 */
    struct { u16 x, y; } white_point;          /* CIE xy × 50000 */
    u16 max_display_mastering_luminance; /* cd/m² (1 cd/m² units) */
    u16 min_display_mastering_luminance; /* 0.0001 cd/m² units */
    u16 max_cll;                         /* MaxCLL in cd/m² */
    u16 max_fall;                        /* MaxFALL in cd/m² */
};
```

The `eotf` field encodes the transfer function:

```c
enum hdmi_eotf {
    HDMI_EOTF_TRADITIONAL_GAMMA_SDR    = 0, /* SDR (BT.1886/gamma 2.2) */
    HDMI_EOTF_TRADITIONAL_GAMMA_HDR    = 1, /* Traditional HDR gamma */
    HDMI_EOTF_SMPTE_ST2084             = 2, /* PQ — used for HDR10 */
    HDMI_EOTF_BT_2100_HLG             = 3, /* HLG */
};
```

### drm_hdmi_infoframe_set_hdr_metadata()

The kernel helper `drm_hdmi_infoframe_set_hdr_metadata()` fills an `hdmi_drm_infoframe` from the connector state's `hdr_output_metadata` blob:

```c
/* drivers/gpu/drm/display/drm_hdmi_state_helper.c (simplified) */
int drm_hdmi_infoframe_set_hdr_metadata(struct hdmi_drm_infoframe *frame,
                                        struct drm_connector_state *conn_state)
{
    struct hdr_output_metadata *hdr_metadata;

    if (!conn_state->hdr_output_metadata)
        return -EINVAL;

    hdr_metadata = conn_state->hdr_output_metadata->data;

    hdmi_drm_infoframe_init(frame);
    frame->eotf = hdr_metadata->hdmi_metadata_type1.eotf;
    frame->metadata_type = hdr_metadata->hdmi_metadata_type1.metadata_type;

    for (int i = 0; i < 3; i++) {
        frame->display_primaries[i].x =
            hdr_metadata->hdmi_metadata_type1.display_primaries[i].x;
        frame->display_primaries[i].y =
            hdr_metadata->hdmi_metadata_type1.display_primaries[i].y;
    }
    frame->white_point.x = hdr_metadata->hdmi_metadata_type1.white_point.x;
    frame->white_point.y = hdr_metadata->hdmi_metadata_type1.white_point.y;
    frame->max_display_mastering_luminance =
        hdr_metadata->hdmi_metadata_type1.max_display_mastering_luminance;
    frame->min_display_mastering_luminance =
        hdr_metadata->hdmi_metadata_type1.min_display_mastering_luminance;
    frame->max_cll  = hdr_metadata->hdmi_metadata_type1.max_cll;
    frame->max_fall = hdr_metadata->hdmi_metadata_type1.max_fall;
    return 0;
}
```

[Source: drm_hdmi_state_helper.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/display/drm_hdmi_state_helper.c)

### HDMI 2.0 SCDC and Scrambling

HDMI 2.0 introduced the **Status and Control Data Channel (SCDC)**, an I²C-like channel over the DDC pins used for high-bandwidth link management. When driving a display at pixel rates above 340 Mcsc (requiring TMDS scrambling), the kernel uses the SCDC to negotiate the link:

```c
/* drivers/gpu/drm/bridge/synopsys/dw-hdmi.c — enabling TMDS scrambling */
if (scdc_present) {
    drm_scdc_set_scrambling(connector, true);
    drm_scdc_set_high_tmds_clock_ratio(connector, true);
}
```

[Source: dw-hdmi.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/bridge/synopsys/dw-hdmi.c)

SCDC does not carry HDR metadata itself — that goes through InfoFrames — but it is a prerequisite for 4K/60 Hz or higher resolutions commonly used with HDR10, which require the higher TMDS clock rate that SCDC enables. The `drm_scdc_helper.c` helpers (`drm_scdc_readb()`, `drm_scdc_writeb()`) provide the I²C I/O. [Source: drm_scdc_helper.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/display/drm_scdc_helper.c)

### AVI InfoFrame and BT.2020 Colourimetry Signaling

For HDR10, the AVI InfoFrame must signal BT.2020 colourimetry to the display so it interprets the pixel values correctly. The DRM helper `drm_hdmi_avi_infoframe_from_display_mode()` fills the AVI InfoFrame based on the display mode; callers must additionally set the extended colourimetry fields for HDR:

```c
/* Construct AVI InfoFrame for HDR10 (BT.2020, limited range): */
struct hdmi_avi_infoframe avi_frame;
drm_hdmi_avi_infoframe_from_display_mode(&avi_frame, connector, mode);

/* Override colorimetry for BT.2020 HDR: */
avi_frame.colorimetry      = HDMI_COLORIMETRY_EXTENDED;
avi_frame.extended_colorimetry = HDMI_EXTENDED_COLORIMETRY_BT2020;
avi_frame.quantization_range   = HDMI_QUANTIZATION_RANGE_LIMITED;
avi_frame.ycc_quantization_range = HDMI_YCC_QUANTIZATION_RANGE_LIMITED;

hdmi_avi_infoframe_pack(&avi_frame, buf, sizeof(buf));
```

[Source: drm_hdmi_helper.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/display/drm_hdmi_helper.c)

---

## DisplayPort HDR Descriptor

### HDR Metadata Transport on DisplayPort

DisplayPort does not use InfoFrames in the HDMI sense. Instead, HDR metadata is carried in the **Secondary Data Packet (SDP)** stream as the **VSC SDP (Vendor-Specific Command SDP)** extended with colourimetry and HDR metadata. VESA DisplayPort 1.4 (2016) added HDR metadata transport in the VSC SDP extended format.

The kernel constant for this SDP type is `DP_SDP_VSC` and the colourimetry fields are in the `DP_SDP_VSC_EXT_COLORIMETRY` payload byte. The VSC SDP carries:
- Colourimetry (BT.601, BT.709, BT.2020, xvYCC)
- Dynamic range indicator (VESA DSC / CTA-861 compatibility)
- EOTF value (SDR/HDR/PQ/HLG)

[Source: VESA DP 1.4 spec; amd-gfx patch](https://www.mail-archive.com/amd-gfx@lists.freedesktop.org/msg104970.html)

### DPCD Capability Discovery

A DisplayPort sink reports HDR capability via the **Display Port Configuration Data (DPCD)** register map. The relevant registers:

```c
/* include/drm/display/drm_dp.h — DPCD capability registers */
#define DP_RECEIVER_CAP_SIZE       0xf
#define DP_DOWNSTREAMPORT_PRESENT  0x05
#define DP_MAIN_LINK_CHANNEL_CODING_SET 0x108

/* eDP HDR support is signaled via DPCD 0x070 (EDP_GENERAL_CAP3) */
#define DP_EDP_GENERAL_CAP3        0x070
#define DP_EDP_BACKLIGHT_BRIGHTNESS_AUX_SET_CAP  (1 << 2)
```

For external DisplayPort displays, HDR capability is primarily discovered through the EDID CTA extension (parsed with `libdisplay-info`, see next section), not through DPCD registers. The DPCD registers are more relevant for embedded DisplayPort (eDP) panels in laptops, where the DPCD `EDP_GENERAL_CAP3` register and backlight-related registers expose HDR backlight control capability.

[Source: include/drm/display/drm_dp.h](https://github.com/torvalds/linux/blob/master/include/drm/display/drm_dp.h)

### VSC SDP Structure for HDR Colourimetry

The VSC SDP (Secondary Data Packet) extended format carries colourimetry and HDR signaling data. In DisplayPort 1.4, when the sink supports VSC SDP (DPCD register `0x05` `DP_DOWNSTREAMPORT_PRESENT` bit 4 set, or DPCD revision >= 1.4), the source may send the VSC SDP with DB16–DB18 extended for colourimetry:

```
VSC SDP Header:
  HB0 = 0x00 (secondary data packet type: VSC SDP)
  HB1 = 0x07 (version: 0x07 for colourimetry extension)
  HB2 = 0x05 (number of valid data bytes in each DB: DB0–DB17 → 18 bytes)
  HB3 = 0x13 (number of valid data bytes per line)

VSC SDP Payload:
  DB0–DB3:   reserved (0x00)
  DB4:       Pixel Encoding / Colorimetry Format
               Bits 7:4 = Pixel Encoding
                 0x0 = RGB
                 0x1 = YCbCr 4:2:2
                 0x2 = YCbCr 4:4:4
               Bits 3:0 = Colorimetry Format
                 0x0 = Default / sRGB (RGB) or BT.601 (YCbCr)
                 0x1 = sRGB (RGB) / BT.709 (YCbCr)
                 0x4 = DCI-P3 (RGB) / xvYCC 601 (YCbCr)
                 0x6 = BT.2020 cycc / BT.2020 RGB
  DB5:       Bit 7 = Dynamic Range:
               0 = VESA range (SDR)
               1 = CTA range (HDR, requires DB16–DB17 fields)
  DB16:      Electro-Optical Transfer Function (EOTF)
               0x00 = Traditional Gamma SDR
               0x01 = Traditional Gamma HDR
               0x02 = SMPTE ST 2084 (PQ) — HDR10
               0x03 = HLG (ITU-R BT.2100)
  DB17:      Static Metadata Descriptor ID (0x00 = Type 1)
  DB18–DB27: HDR static metadata fields (primaries, white point, luminance)
```

This structure is directly analogous to the HDMI DRM InfoFrame (type 0x87) content, but carried as a DP SDP rather than an HDMI InfoFrame.

### AMD and i915 VSC SDP HDR Programming

The AMD `amdgpu` driver programs HDR metadata via the VSC SDP in `drivers/gpu/drm/amd/display/dc/link/`. For DP sinks with DPCD revision >= 1.4, the driver sends the VSC SDP with colourimetry extension:

```c
/* drivers/gpu/drm/amd/display/dc/link/ — VSC SDP for HDR colourimetry */
/* Simplified from amd-gfx patch: Program VSC SDP colorimetry */
static void set_vsc_sdp_colorimetry(struct dc_stream_state *stream,
                                    struct encoder_info_frame *info)
{
    /* VSC SDP payload: */
    info->vsc.valid              = true;
    info->vsc.use_vsc_sdp_for_colorimetry = true;
    info->vsc.colorimetry        = COLOR_SPACE_2020_RGB_LIMITEDRANGE;
    info->vsc.dynamic_range_mastering = true;
    info->vsc.hdr_enabled        = true;
    /* Mastering metadata (from stream hdr_metadata): */
    info->vsc.sdp_header.HB2     = 0x05; /* VSC SDP DB16~DB18 */
    info->vsc.sdp_header.HB3     = 0x13; /* colorimetry data size */
}
```

[Source: amd-gfx mailing list — VSC SDP patch](https://www.mail-archive.com/amd-gfx@lists.freedesktop.org/msg104970.html)

The Intel i915 driver handles DP HDR metadata similarly via `intel_dp_hdcp.c` and the display engine's secondary data packet programming in `intel_hdmi.c` / `intel_dp.c`. For eDP panels the driver writes HDR content luminance through the AUX channel using the `intel_dp_aux_backlight.c` DPCD write path.

### HDR10 vs. Dolby Vision on DisplayPort

**HDR10** is fully supported in upstream Linux DP drivers using the VSC SDP mechanism above. The mastering display metadata travels in the SDP alongside the colourimetry flags.

**Dolby Vision** on DisplayPort requires a proprietary tunnel: the sink must support the Dolby Vision-specific DPCD capability bits, and the source must send the Dolby Vision-specific metadata in the VSC SDP `DB16`–`DB26` fields. This is not yet fully implemented in mainline Linux display drivers as of Linux 6.x and relies on vendor-specific downstream support.

> **Note: needs verification** — Dolby Vision over DP status in mainline Linux. Check the dri-devel mailing list for current status.

---

## libdisplay-info: EDID and HDR Capability Parsing

### What libdisplay-info Is

**libdisplay-info** is a C library (maintained by Simon Ser / Collabora) for parsing EDID (Extended Display Identification Data) and DisplayID structures. It is the intended replacement for the aging `drm_edid.h` parsing code scattered across Mesa, wlroots, and various compositor codebases. [Source: libdisplay-info homepage](https://gitlab.freedesktop.org/emersion/libdisplay-info)

The library provides two API layers:
- **Low-level API** (`edid.h`, `cta.h`, `displayid.h`): Direct access to all parsed EDID structures, data blocks, and timing entries.
- **High-level API** (`info.h`): Convenience functions that aggregate parsed data, including `di_info_get_hdr_static_metadata()`.

Adoption status (2025–2026): wlroots uses libdisplay-info to query HDR capability before setting KMS HDR connector properties; GNOME Mutter calls `di_info_get_hdr_static_metadata()` to decide whether to advertise HDR to applications via `wp_color_management_v1`. [Source: Phoronix wlroots color management](https://www.phoronix.com/news/wlroots-color-management)

### Creating a di_info Object from EDID

The entry point for compositor code is `di_info_create_from_edid()`:

```c
/* libdisplay-info/include/libdisplay-info/info.h */
#include <libdisplay-info/info.h>

/* Read EDID from DRM connector sysfs or drmModeGetConnector: */
uint8_t *edid_data;
size_t   edid_size;
/* ... obtain via drmModeGetPropertyBlob() on the EDID connector prop ... */

struct di_info *info = di_info_create_from_edid(edid_data, edid_size);
if (!info) {
    /* EDID parse failure */
    return;
}

/* High-level: query HDR capability */
const struct di_hdr_static_metadata *hdr =
    di_info_get_hdr_static_metadata(info);

if (hdr->pq) {
    printf("Display supports PQ (HDR10)\n");
    printf("Max luminance: %.0f nits\n", hdr->desired_content_max_luminance);
}

di_info_destroy(info);
```

[Source: libdisplay-info info.h documentation](https://emersion.pages.freedesktop.org/libdisplay-info/libdisplay-info/info.h.html)

### The di_hdr_static_metadata Structure

```c
/* libdisplay-info/include/libdisplay-info/info.h */
struct di_hdr_static_metadata {
    /* Desired content luminance targets from the display's EDID
     * (from CTA-861-H HDR Static Metadata Data Block).
     * These are the luminance levels the display is optimised for.
     * Values are 0.0 if not advertised. */
    float desired_content_max_luminance;          /* peak, in cd/m² */
    float desired_content_max_frame_avg_luminance;/* MaxFALL, in cd/m² */
    float desired_content_min_luminance;          /* black floor, in cd/m² */

    bool  type1;           /* Static Metadata Type 1 (HDR10) supported */
    bool  traditional_sdr; /* EOTF bit 0: SDR gamma supported */
    bool  traditional_hdr; /* EOTF bit 1: HDR gamma supported */
    bool  pq;              /* EOTF bit 2: PQ (SMPTE ST 2084) supported */
    bool  hlg;             /* EOTF bit 3: HLG (ITU-R BT.2100) supported */
};

/* Function never returns NULL — absent metadata → all-zero luminances,
 * only traditional_sdr = true. */
const struct di_hdr_static_metadata *
di_info_get_hdr_static_metadata(const struct di_info *info);
```

[Source: libdisplay-info documentation](https://emersion.pages.freedesktop.org/libdisplay-info/libdisplay-info/info.h.html)

### Low-Level CTA Extension Traversal

For compositors needing finer-grained control (e.g., to detect HDR Dynamic Metadata Data Blocks introduced in CTA-861-H for HDR10+), the low-level API walks the EDID extension list:

```c
/* libdisplay-info/include/libdisplay-info/edid.h + cta.h */
#include <libdisplay-info/edid.h>
#include <libdisplay-info/cta.h>

const struct di_edid *edid = di_info_get_edid(info);

/* Iterate extension blocks: */
const struct di_edid_ext * const *exts = di_edid_get_extensions(edid);
for (int i = 0; exts[i] != NULL; i++) {
    if (di_edid_ext_get_tag(exts[i]) != DI_EDID_EXT_CEA)
        continue;

    const struct di_edid_cta *cta = di_edid_ext_get_cta(exts[i]);
    if (!cta)
        continue;

    /* Walk CTA data blocks: */
    const struct di_cta_data_block * const *dbs =
        di_edid_cta_get_data_blocks(cta);

    for (int j = 0; dbs[j] != NULL; j++) {
        enum di_cta_data_block_tag tag = di_cta_data_block_get_tag(dbs[j]);

        if (tag == DI_CTA_DATA_BLOCK_HDR_STATIC_METADATA) {
            const struct di_cta_hdr_static_metadata_block *hsm =
                di_cta_data_block_get_hdr_static_metadata(dbs[j]);
            /* Process HDR static metadata data block */
        } else if (tag == DI_CTA_DATA_BLOCK_HDR_DYNAMIC_METADATA) {
            const struct di_cta_hdr_dynamic_metadata_block *hdm =
                di_cta_data_block_get_hdr_dynamic_metadata(dbs[j]);
            /* Process HDR dynamic metadata data block (HDR10+, Dolby) */
        }
    }
}
```

[Source: libdisplay-info edid.h API](https://emersion.pages.freedesktop.org/libdisplay-info/libdisplay-info/edid.h.html)

### How wlroots Uses libdisplay-info for HDR

The wlroots compositor library queries libdisplay-info when a DRM connector is connected, to determine whether to enable KMS HDR output properties. The flow is:

1. Read EDID blob from `drmModeGetConnector()`.
2. Call `di_info_create_from_edid()` to parse it.
3. Call `di_info_get_hdr_static_metadata()` on the result.
4. If `hdr->pq` is true and `hdr->desired_content_max_luminance >= 400.0`, set the `HDR_OUTPUT_METADATA` KMS connector property with a matching PQ EOTF blob.
5. Destroy the `di_info` after extracting all needed fields.

This pattern separates display capability discovery (libdisplay-info) from display programming (KMS atomic commit), making it straightforward to conditionally enable HDR only on capable displays.

[Source: wlroots color management merge](https://www.phoronix.com/news/wlroots-color-management)

---

## EDID HDR Capability: CTA-861 Data Block Internals

### The CTA-861 HDR Static Metadata Data Block (Tag 0x06)

The HDR Static Metadata Data Block is defined in **CTA-861-H** (formerly CEA-861-G) as an extended data block with Extended Tag Code **0x06**. It appears in the CTA-861 extension block of the EDID (block tag 0x02 in the EDID extension map). Its structure:

```
Byte 0: Extended Tag Code = 0x06 (identifies this as HDR Static Metadata DB)
Byte 1: EOTF support bitmap
         Bit 0 = Traditional Gamma SDR (always present on compliant displays)
         Bit 1 = Traditional Gamma HDR
         Bit 2 = SMPTE ST 2084 (PQ)    — required for HDR10
         Bit 3 = Hybrid Log-Gamma (HLG)
         Bits 4–7 = reserved
Byte 2: Static Metadata Descriptor Type bitmap
         Bit 0 = Static Metadata Type 1 — required for HDR10
         Bits 1–7 = reserved
Byte 3: Desired Content Max Luminance (optional)
         Encoded as: nits = 50 × (2 ^ (code / 32))
         Example: code=0xA0 (160) → 50 × 2^5 = 1600 nit peak
Byte 4: Desired Content Max Frame-Average Luminance (optional)
         Same encoding as Byte 3
Byte 5: Desired Content Min Luminance (optional)
         Encoded relative to Desired Max: min = (code/255)^2 × Max × 0.01
```

[Source: CTA-861.3-A standard; CTA-861-H]

### Raw EDID Parsing (Kernel drm_edid.c)

The Linux kernel's historical EDID parser in `drivers/gpu/drm/drm_edid.c` reads these bytes during connector hotplug and populates the `drm_connector.hdr_sink_metadata` field:

```c
/* drivers/gpu/drm/drm_edid.c — parse_hdr_metadata_common() (simplified) */
static void parse_hdr_metadata_common(struct drm_connector *connector,
                                      const u8 *db)
{
    struct hdr_static_metadata *hdr = &connector->hdr_sink_metadata.hdmi_type1;

    hdr->eotf = db[1] & DRM_EDID_HDMI_HDR_EOTF_MASK;
    hdr->sm_type = db[2] & DRM_EDID_HDMI_HDR_SMD_MASK;

    /* Byte 3: desired max luminance (optional, length >= 3) */
    if (db_length >= 3 && db[3])
        hdr->max_luminance = db[3];  /* stored as raw code; compute above */

    /* Byte 4: desired max frame-average luminance */
    if (db_length >= 4 && db[4])
        hdr->max_average_luminance = db[4];

    /* Byte 5: desired min luminance */
    if (db_length >= 5 && db[5])
        hdr->min_luminance = db[5];
}
```

[Source: drivers/gpu/drm/drm_edid.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_edid.c)

The raw luminance codes are not pre-decoded to nit values in the kernel — compositors receive the raw `hdr_sink_metadata` structure or, preferably, obtain decoded values from `libdisplay-info`.

### HDR Dynamic Metadata Data Block

CTA-861-H also defines the **HDR Dynamic Metadata Data Block** (Extended Tag 0x07) for displays supporting HDR10+ or Dolby Vision. This block reports which dynamic metadata formats the display understands. `libdisplay-info` exposes this via `di_cta_data_block_get_hdr_dynamic_metadata()`. As of 2025–2026, most Linux compositor stacks do not yet use HDR dynamic metadata in production.

---

## OLED and MiniLED Local Dimming

### Display Hardware Capabilities

Two display technologies provide the contrast ratios that make HDR perceptually impactful:

**OLED**: Each pixel is self-emissive and can be fully turned off, giving a theoretical infinite contrast ratio (pixel-level "local dimming"). OLED panels appear in laptops (Dell XPS, Framework 13, Lenovo ThinkPad OLED), external monitors (LG UltraFine OLED), and Steam Deck OLED.

**MiniLED**: A traditional LCD with a backlight divided into hundreds or thousands of independently controllable LED zones. Each zone can dim or boost independently based on the image content in that region, approaching (but not equalling) OLED contrast for HDR. MiniLED panels appear in Apple Pro Display XDR, ASUS ProArt series, and Samsung Odyssey Neo G9.

The key difference for the Linux driver stack: OLED local dimming is handled entirely within the panel's own pixel circuitry and is invisible to the display controller. MiniLED local dimming, however, can be controlled by the display controller or firmware, and exposing that control to compositors is an active development area.

### Adaptive Backlight Modulation in DRM

For laptop eDP panels (both OLED and MiniLED), the kernel exposes backlight control through the `drm_backlight` interface and, more recently, through a DRM connector property for **Adaptive Backlight Modulation (ABM)**. A November 2025 patch by Mario Limonciello proposed moving the AMD-specific ABM property to DRM core:

```c
/* [PATCH] drm/amd: Move adaptive backlight modulation property to drm core
 * Property attached to eDP connectors supporting ABM.
 * Values: "off", "min", "bias min", "bias max", "max", "sysfs"
 * Default: "sysfs" (controlled via /sys/class/drm/cardX-eDP-Y/panel_power_savings)
 */
drm_object_attach_property(&connector->base,
    dev->mode_config.abm_property,
    DRM_PANEL_ORIENTATION_UNKNOWN);
```

[Source: LKML patch — drm/amd ABM to drm core](https://lkml.iu.edu/hypermail/linux/kernel/2511.1/08238.html)

For HDR compositing, ABM interacts with HDR metadata: if the compositor is sending HDR metadata indicating high-luminance highlights, the ABM algorithm should not aggressively dim zones containing those highlights. The relationship between `HDR_OUTPUT_METADATA` and ABM is not yet formally specified in the DRM API and is an active area of compositor/driver co-design.

### Local Dimming and HDR Plane Interaction

When a compositor draws HDR content on a display with zonal MiniLED dimming, the display firmware reads the scanout content to determine zone dimming levels. On Intel's **Display Micro Controller (DMC)** in Meteor Lake and Lunar Lake platforms, the firmware can autonomously compute MiniLED zone dimming from scanout pixel data — a feature called **Panel Replay with Local Dimming** for eDP panels. This requires that the GPU output pixel-accurate HDR values so the DMC can compute correct zone luminance targets.

The kernel's eDP AUX backlight driver (`drivers/gpu/drm/i915/display/intel_dp_aux_backlight.c`) programs DPCD registers for the panel's backlight controller. For HDR-aware local dimming, the driver also writes the `INTEL_EDP_HDR_CONTENT_LUMINANCE` DPCD register to inform the panel of the content luminance range, allowing the panel's tone-mapping and dimming algorithms to adapt.

[Source: intel_dp_aux_backlight.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_dp_aux_backlight.c)

### drm_backlight and Compositor Coordination

The traditional Linux backlight interface at `/sys/class/backlight/` operates as a simple brightness scalar in the range `[0, max_brightness]`. For HDR compositing this is insufficient for two reasons:

1. **Brightness interaction with HDR metadata**: When a compositor enables HDR mode (EOTF=PQ via `HDR_OUTPUT_METADATA`), it expects the display to apply the PQ EOTF to code values — setting a PQ output level of 0.5 should correspond to approximately 100 nits (the SDR reference white in the PQ scale). If the backlight ABM simultaneously dims the entire backlight by 50%, the resulting luminance will be half what the metadata implies, breaking the HDR calibration.

2. **Power vs. accuracy tradeoff**: ABM aggressively dims the backlight on content with mostly dark pixels to save power. On OLED this is handled per-pixel by the panel. On MiniLED the backlight zones need to know about the HDR content range to avoid incorrectly dimming zones containing HDR highlights.

The Display Next hackfest 2024–2025 discussions identified a requirement for compositors to signal their power/colour accuracy preferences to the kernel before the kernel adjusts backlight or ABM parameters. The proposed solution is a DRM connector property that lets compositors declare their preference between `"performance"` (full brightness, colour accuracy) and `"power_savings"` (allow ABM/dimming). This complements the `HDR_OUTPUT_METADATA` property by giving the display controller enough context to make sensible local dimming decisions.

```c
/* Proposed compositor interaction (not yet upstream as of June 2026): */
drmModeAtomicAddProperty(req, connector_id, panel_orientation_prop,
    DRM_PANEL_ORIENTATION_UNKNOWN); /* placeholder until upstream API lands */
```

The existing `/sys/class/drm/card0-eDP-1/panel_power_savings` sysfs file (exposed by the AMD ABM property) is the current way for compositors to opt out of backlight dimming when HDR mode is active.

> **Note: needs verification** — The specific DRM plane property for per-zone backlight control on MiniLED displays has been discussed but not yet merged to drm-misc-next as of June 2026. Check current DRM patch queue.

[Source: LWN — DRM backlight capability](https://lwn.net/Articles/1069679/) | [Source: LKML ABM patch](https://lkml.iu.edu/hypermail/linux/kernel/2511.1/08238.html)

---

## The drm_hdr_output_metadata Kernel Structure

### Structure Definition

`struct hdr_output_metadata` is the UAPI structure that compositors write into a DRM blob property and attach to the connector's `HDR_OUTPUT_METADATA` property. It is defined in `include/uapi/drm/drm_mode.h`:

```c
/* include/uapi/drm/drm_mode.h */

/* EOTF values for hdr_metadata_infoframe.eotf: */
#define HDMI_EOTF_TRADITIONAL_GAMMA_SDR    0
#define HDMI_EOTF_TRADITIONAL_GAMMA_HDR    1
#define HDMI_EOTF_SMPTE_ST2084             2  /* PQ — use for HDR10 */
#define HDMI_EOTF_BT_2100_HLG             3

/* Static Metadata Type 1 (SM1) — the only standardised type as of 2026: */
#define HDMI_STATIC_METADATA_TYPE1         0

/**
 * struct hdr_metadata_infoframe - HDR Metadata Infoframe Data
 *
 * HDR Metadata Infoframe as per CTA 861.G spec. This is expected
 * to match the metadata sent via the HDMI DRM InfoFrame (type 0x87).
 */
struct hdr_metadata_infoframe {
    __u8 eotf;              /* HDMI_EOTF_* value */
    __u8 metadata_type;     /* HDMI_STATIC_METADATA_TYPE1 */
    struct {
        __u16 x, y;         /* CIE xy chromaticity × 50000 */
    } display_primaries[3]; /* Index 0=R, 1=G, 2=B (ordering per CTA-861) */
    struct {
        __u16 x, y;         /* D65: x=15635, y=16450 */
    } white_point;
    __u16 max_display_mastering_luminance; /* Mastering display peak, cd/m² */
    __u16 min_display_mastering_luminance; /* Mastering display black, ×0.0001 cd/m² */
    __u16 max_cll;          /* MaxCLL: max content light level, cd/m² */
    __u16 max_fall;         /* MaxFALL: max frame-average light level, cd/m² */
};

/**
 * struct hdr_output_metadata - HDR output metadata
 *
 * Metadata Information to be passed from userspace (compositor)
 * to the kernel driver via the HDR_OUTPUT_METADATA connector property blob.
 */
struct hdr_output_metadata {
    __u32 metadata_type; /* HDMI_STATIC_METADATA_TYPE1 (currently only type) */
    union {
        struct hdr_metadata_infoframe hdmi_metadata_type1;
    };
};
```

[Source: include/uapi/drm/drm_mode.h](https://github.com/torvalds/linux/blob/master/include/uapi/drm/drm_mode.h) | [Source: LWN HDR patch](https://lwn.net/Articles/783575/)

### Encoding the Display Primary Coordinates

The `display_primaries` and `white_point` coordinates are encoded in CIE 1931 xy chromaticity units scaled by 50,000 (i.e., a value of 1.0 → 0xC350 = 50000):

```c
/* Encoding BT.2020 primaries for hdr_metadata_infoframe:
 * BT.2020 Red: CIE xy = (0.708, 0.292)
 * BT.2020 Green: CIE xy = (0.170, 0.797)
 * BT.2020 Blue: CIE xy = (0.131, 0.046)
 * D65 White: CIE xy = (0.3127, 0.3290)
 */
struct hdr_metadata_infoframe *m = &meta.hdmi_metadata_type1;

m->display_primaries[0].x = (u16)(0.708  * 50000); /* Red x = 35400 */
m->display_primaries[0].y = (u16)(0.292  * 50000); /* Red y = 14600 */
m->display_primaries[1].x = (u16)(0.170  * 50000); /* Green x = 8500  */
m->display_primaries[1].y = (u16)(0.797  * 50000); /* Green y = 39850 */
m->display_primaries[2].x = (u16)(0.131  * 50000); /* Blue x = 6550  */
m->display_primaries[2].y = (u16)(0.046  * 50000); /* Blue y = 2300  */
m->white_point.x          = (u16)(0.3127 * 50000); /* D65 x  = 15635 */
m->white_point.y          = (u16)(0.3290 * 50000); /* D65 y  = 16450 */
```

### Minimum Luminance Encoding

`min_display_mastering_luminance` is in units of 0.0001 cd/m² (so 1 cd/m² = 10000 in this field). A mastering display black of 0.005 nit would be encoded as 50 (= 0.005 × 10000). This is a common source of confusion since `max_display_mastering_luminance`, `max_cll`, and `max_fall` are all in direct cd/m² units.

### Attaching the Property to a Connector

Driver authors call `drm_connector_attach_hdr_output_metadata_property()` during connector initialisation:

```c
/* drivers/gpu/drm/drm_connector.c */
int drm_connector_attach_hdr_output_metadata_property(
    struct drm_connector *connector)
{
    struct drm_device *dev = connector->dev;
    struct drm_property *prop =
        dev->mode_config.hdr_output_metadata_property;

    drm_object_attach_property(&connector->base, prop, 0);
    return 0;
}
```

This creates a blob property on the connector. At atomic commit, the driver reads `conn_state->hdr_output_metadata` (a `drm_property_blob *`) and passes it to the display engine.

---

## Compositor-to-KMS HDR Metadata Flow

### Overview

When a compositor (wlroots, Mutter, KWin) decides to enable HDR output on a connector, it must construct the `hdr_output_metadata` blob and commit it atomically alongside other display state changes. This section describes that flow, complementing the software pipeline description in Ch74.

### Step 1: Display HDR Capability Discovery

The compositor reads the EDID from the DRM connector's `EDID` property blob and passes it to `libdisplay-info`. The `di_info_get_hdr_static_metadata()` result determines:
- Whether PQ is supported (`hdr->pq == true`)
- The display's preferred content luminance targets
- Whether the display claims HLG support

Only if `hdr->pq` (or `hdr->hlg`) is true should the compositor attempt to send HDR metadata. Sending HDR metadata to an SDR display may cause undefined behaviour.

### Step 2: Constructing the Metadata Blob

```c
/* Example: compositor constructing HDR_OUTPUT_METADATA for HDR10 output */
struct hdr_output_metadata meta = { 0 };
meta.metadata_type = HDMI_STATIC_METADATA_TYPE1;

struct hdr_metadata_infoframe *m = &meta.hdmi_metadata_type1;
m->eotf          = HDMI_EOTF_SMPTE_ST2084;    /* PQ for HDR10 */
m->metadata_type = HDMI_STATIC_METADATA_TYPE1;

/* BT.2020 primaries (from display EDID or fixed HDR10 container values): */
m->display_primaries[0].x = 35400; /* R: 0.708 × 50000 */
m->display_primaries[0].y = 14600;
m->display_primaries[1].x = 8500;  /* G: 0.170 × 50000 */
m->display_primaries[1].y = 39850;
m->display_primaries[2].x = 6550;  /* B: 0.131 × 50000 */
m->display_primaries[2].y = 2300;
m->white_point.x = 15635;          /* D65: 0.3127 × 50000 */
m->white_point.y = 16450;

/* Mastering display luminance (from display's EDID HDR metadata block,
 * or compositor-defined safe defaults): */
m->max_display_mastering_luminance = 1000;   /* nits */
m->min_display_mastering_luminance = 50;     /* = 0.005 nit × 10000 */

/* Content-level metadata (from surface's wp_color_management metadata): */
m->max_cll  = 1000;   /* MaxCLL in nits */
m->max_fall = 400;    /* MaxFALL in nits */

/* Create DRM blob and attach to connector: */
uint32_t blob_id;
drmModeCreatePropertyBlob(drm_fd, &meta, sizeof(meta), &blob_id);
drmModeAtomicAddProperty(req, connector_id, hdr_output_metadata_prop, blob_id);
```

### Step 3: Ordering with the KMS Color Pipeline

As described in Ch74, the full HDR output requires coordinating multiple KMS properties in the same atomic commit:

```
Atomic commit must include:
  connector: HDR_OUTPUT_METADATA = <blob>   ← this chapter
  crtc:      DEGAMMA_LUT = <linearisation>  ← ch74
  crtc:      CTM = <gamut matrix>           ← ch74
  crtc:      GAMMA_LUT = <PQ OETF curve>   ← ch74 (or empty for display-side PQ)
  plane:     COLOR_PIPELINE = <colorop chain> ← ch74 (Linux 6.19+)
```

The `HDR_OUTPUT_METADATA` blob tells the **display** how to interpret the pixels (EOTF and mastering display volume). The `DEGAMMA_LUT`/`CTM`/`GAMMA_LUT` pipeline controls what pixel values the **GPU** writes to the framebuffer. Both must be consistent: if the metadata says `EOTF = PQ`, the GPU pipeline must be writing PQ-encoded 10-bit values in BT.2020 container.

### Step 4: Receiving Application Metadata via wp_color_management_v1

When an application declares its surface as HDR10 via `wp_color_management_v1` (creating a parametric image description with `tf_name = "pq"` and `max_luminance`, `max_cll`, `max_fall` set — see Ch74 for the protocol details), the compositor receives this metadata and uses it to populate the `max_cll` and `max_fall` fields of the `hdr_output_metadata` blob. This ensures the InfoFrame sent to the display reflects the actual content metadata, allowing the display's own tone-mapping or local dimming algorithm to make optimal decisions.

### Driver-Level Flow

At the driver level, the atomic commit callback sequence is:

1. `drm_atomic_helper_commit()` → driver's `.atomic_commit_tail()`.
2. Driver's connector helper `.atomic_set_property()` reads `conn_state->hdr_output_metadata`.
3. For HDMI: `hdmi_generate_hdr_infoframe()` → `drm_hdmi_infoframe_set_hdr_metadata()` → driver's `.write_infoframe()` callback sends the DRM InfoFrame (type 0x87) over HDMI.
4. For DisplayPort: the driver programs the VSC SDP HDR payload through the display engine's SDP registers.
5. The AVI InfoFrame (type 0x82) is also updated to reflect BT.2020 colourimetry.

[Source: commit 2cdbfd66 — Enable HDR infoframe support](https://github.com/torvalds/linux/commit/2cdbfd66a82969770ce1a7032fb1e2155a08cee8)

---

## HDR Certification: VESA DisplayHDR Tiers

### The DisplayHDR Standard

**VESA DisplayHDR** is the independent certification standard for monitor HDR performance, managed by VESA (Video Electronics Standards Association). Launched in 2017 and updated to version 1.2 (December 2024), it defines minimum performance requirements across brightness, contrast, local dimming, colour gamut, and bit depth. [Source: VESA DisplayHDR performance criteria](https://displayhdr.org/performance-criteria/)

Linux driver developers targeting HDR displays need to understand these tiers because:
1. **Test methodology**: VESA testing uses specific test patterns that stress-test HDR metadata handling and tone mapping.
2. **Backlight control**: Higher tiers require local dimming, which intersects with the ABM/backlight kernel interfaces described above.
3. **Minimum peak luminance**: The HDR_OUTPUT_METADATA `max_display_mastering_luminance` field should ideally reflect the display's VESA-certified peak value, obtained from the EDID HDR Static Metadata Block.

### DisplayHDR Tier Requirements (Summary, v1.2)

| Tier | Peak Luminance | Full-screen Sustained | Black Level | Local Dimming | Min Colour |
|---|---|---|---|---|---|
| **DisplayHDR 400** | 400 nit | 320 nit (2 min) | ≤0.4 nit | None (or 1D) | — |
| **DisplayHDR 500** | 500 nit | 400 nit | ≤0.1 nit | 1D required | 95% DCI-P3 |
| **DisplayHDR 600** | 600 nit | 450 nit | ≤0.1 nit | 1D required | 95% DCI-P3 |
| **DisplayHDR 1000** | 1000 nit | 600 nit | ≤0.05 nit | 2D required | 95% DCI-P3 |
| **DisplayHDR 1400** | 1400 nit | 900 nit | ≤0.02 nit | 2D required | 95% DCI-P3 |
| **True Black 400** | 400 nit | 320 nit | ≤0.0005 nit | OLED pixel | 95% DCI-P3 |
| **True Black 500** | 500 nit | 400 nit | ≤0.0005 nit | OLED pixel | 95% DCI-P3 |
| **True Black 600** | 600 nit | 450 nit | ≤0.0005 nit | OLED pixel | 95% DCI-P3 |

[Source: VESA DisplayHDR certification tiers](https://displayhdr.org/performance-criteria/) | [Source: TFTCentral DisplayHDR analysis](https://tftcentral.co.uk/news/vesa-update-displayhdr-certification-requirements-including-new-hdr-1400-tier)

**Notes on local dimming tiers**:
- **1D dimming** = backlight zones across one axis only (horizontal rows or vertical columns of LED zones).
- **2D dimming** = independent zones in both axes (full MiniLED-style zone control).
- **True Black** tiers are for OLED panels where individual pixels self-dim to zero.

### DisplayHDR 400: The Baseline Tier

DisplayHDR 400 is the most common certification on desktop monitors and the minimum tier where HDR content can produce a visible improvement over SDR on a typical office display. However, DisplayHDR 400 panels:
- Have no local dimming requirement (many are edge-lit LCD with no zone dimming).
- Often have poor simultaneous contrast (bright highlights + dark shadows cannot coexist).
- May not cover DCI-P3 (no gamut requirement at 400 tier under v1.0/v1.1; v1.2 tightens this).

Linux compositors should check the EDID `desired_content_max_luminance` (via `libdisplay-info`) before assuming a certified 400-nit display actually benefits from HDR tone mapping. Many SDR-authored games and video will look better with a well-calibrated SDR output than a naive HDR upscale on a DisplayHDR 400 display.

### Verifying HDR Compliance from Linux

Driver and compositor developers can test HDR output compliance using:

```bash
# 1. Check connector HDR output metadata property:
drmdb --connector DP-1 --property HDR_OUTPUT_METADATA

# 2. Inspect EDID HDR capability (requires libdisplay-info cli tools):
edid-query /sys/class/drm/card0-DP-1/edid | grep -A10 "HDR Static"

# 3. Verify infoframe is being sent (requires connected HDR display + HDMI analyzer,
#    or check kernel debug output):
sudo dmesg | grep -i "hdr\|infoframe"

# 4. For automated VESA compliance testing, see:
# https://displayhdr.org/test-methodology/ (proprietary test suite)
```

The `drmdb` tool (part of `libdisplay-info` utility suite) can inspect all DRM connector properties including `HDR_OUTPUT_METADATA` blobs and display the mastering display primaries and luminance values in human-readable form. [Source: drmdb property database](https://drmdb.emersion.fr/properties/3233857728/HDR_OUTPUT_METADATA)

---

## Roadmap

### Near-term (6–12 months)

- **HDMI 2.1 FRL support lands in Linux 7.2**: AMD submitted HDMI 2.1 Fixed Rate Link (FRL) patches to drm-next in June 2026, targeting the Linux 7.2 merge window. FRL enables bandwidth for 4K/120Hz and 8K HDR — a prerequisite for full HDR10 and HDR10+ at high refresh rates over HDMI. Support is initially disabled by default pending VRR readiness. [Source: Phoronix HDMI 2.1 FRL submission](https://www.phoronix.com/news/HDMI-FRL-2.1-Submitted-DRM)
- **DRM Color Pipeline API CSC support**: The drm_colorop color-space conversion (CSC) extension to the Color Pipeline API (merged in Linux 6.19) is in active development by AMD's Harry Wentland. CSC colorop support will allow GPU display hardware to accelerate BT.2020→sRGB gamut conversion in the hardware display pipeline rather than in compositor software, directly benefiting HDR10 signaling accuracy. [Source: Phoronix AMD HDR/Color improvement](https://www.phoronix.com/news/AMD-More-HDR-KWin-Claude-Code)
- **NVIDIA per-plane color pipeline API**: NVIDIA announced a preview of DRM per-plane Color Pipeline API support in April 2026, enabling their proprietary driver to participate in the same hardware-accelerated HDR pipeline that AMD and Intel have been building toward. [Source: GamingOnLinux NVIDIA color pipeline](https://www.gamingonlinux.com/2026/04/nvidia-announce-a-preview-of-drm-per-plane-color-pipeline-api-support-on-linux-good-for-hdr/)
- **NVIDIA HDR metadata via Vulkan on Wayland**: NVIDIA's 580.94.11 Linux driver introduced `VK_EXT_hdr_metadata` support on Wayland, enabling games and applications to supply `VkHdrMetadataEXT` mastering display metadata that the driver maps to `HDR_OUTPUT_METADATA` KMS properties. [Source: Phoronix NVIDIA 580.94.11](https://www.phoronix.com/news/NVIDIA-580.94.11-Linux-Driver)
- **libdisplay-info CTA-861 / DisplayID completion**: The library's CTA-861-H support remains partial. Ongoing work to expose the full set of HDR-related data blocks (including HDR10+ VSDB and colorimetry data blocks) will allow compositors to query display capabilities more accurately without falling back to kernel-internal EDID parsing. [Source: libdisplay-info 0.1 announcement](https://www.phoronix.com/news/libdisplay-info-0.1)

### Medium-term (1–3 years)

- **HDR10+ (SMPTE ST 2094-40) dynamic metadata upstream**: As of Linux 6.x, kernel support for assembling the SMPTE ST 2094-40 per-frame dynamic metadata payload in the HDMI Forum Vendor-Specific InfoFrame (HF-VSIF) is not yet merged upstream. HDMI 2.1 FRL landing in 7.2 creates the bandwidth foundation; standardised kernel-level HF-VSIF assembly and a `HDR10PLUS_OUTPUT_METADATA` KMS blob property analogous to `HDR_OUTPUT_METADATA` is the natural next step. Note: needs verification for specific patchset timeline.
- **Dolby Vision signaling kernel support**: Dolby Vision uses a proprietary metadata tunnel inside the HDMI VSIF payload. Standardised kernel-level support for Dolby Vision metadata blobs (beyond what HDMI Forum licenses allow in open-source) is a long-standing unresolved area; industry pressure for open implementations may change this. Note: needs verification.
- **DisplayPort 2.1 UHBR and HDR SDP extensions**: DisplayPort 2.1 Ultra High Bit Rate (UHBR) modes unlock 8K HDR at 120 Hz. Linux DP 2.1 UHBR link training support has been progressing in Intel and AMD drivers; once stable, the associated Video Stream Packet (VSP) extended SDPs that carry HDR10+ dynamic metadata over DP will require matching kernel changes. Note: needs verification for upstream status.
- **Structured local dimming kernel API**: The existing `BACKLIGHT_BRIGHTNESS`/ABM interface for local dimming is coarse. A structured DRM property API for zone-level MiniLED dimming metadata — allowing compositors to hint content brightness per-zone — is a recurring request in dri-devel discussions but has no merged upstream proposal as of 2026. Note: needs verification.
- **Wider compositor adoption of `wp_color_management_v1`**: The Wayland color management protocol (which supersedes the ad-hoc HDR10 Wayland surface metadata used by Gamescope) will increasingly drive the compositor-to-KMS flow described in this chapter. Stable protocol release and adoption in GNOME/KDE/wlroots will make `HDR_OUTPUT_METADATA` population more uniform.

### Long-term

- **Unified HDR/color pipeline across GPU vendors via drm_colorop**: The long-term architectural goal is a vendor-agnostic `drm_colorop` chain that can express the full HDR pipeline — EOTF, CSC, tone-mapping, and OETF — in terms the KMS core can validate and optimise regardless of whether the hardware is AMD DCN, Intel XE, or NVIDIA. Complete standardisation remains years away but the building blocks (Color Pipeline API, `drm_colorop` primitives) are now in place.
- **Open HDR display certification testing tooling**: VESA DisplayHDR compliance testing currently requires proprietary test equipment. A Linux-native open-source HDR compliance test harness — analogous to IGT GPU Tools for KMS — would allow kernel and driver developers to verify HDR signaling correctness automatically in CI. No concrete upstream proposal exists yet.
- **DisplayID 2.1 dynamic capability reporting**: DisplayID 2.1 (successor to EDID) supports richer display capability reporting including dynamic HDR formats. `libdisplay-info` and the kernel's EDID/DisplayID stack will need to evolve to parse and expose DisplayID 2.1 HDR capability data as displays migrate away from legacy EDID CTA-861 extensions.

---

## Integrations

- **Ch74 (HDR and Wide Color Gamut)** — the software complement to this chapter: KMS `DEGAMMA_LUT`/`CTM`/`GAMMA_LUT`, `wp_color_management_v1`, tone-mapping algorithms, and compositor HDR architecture. Ch74 and Ch158 together form the complete HDR picture; Ch74 owns the GPU/compositor pipeline, Ch158 owns the display wire signaling.

- **Ch2 (KMS Display Pipeline)** — the foundational KMS/DRM concepts (CRTC, connector, atomic commit, property blobs) used throughout this chapter for the `HDR_OUTPUT_METADATA` property and atomic commit flow.

- **Ch3 (Advanced Display Features)** — EDID parsing infrastructure (`drm_edid.c`), connector hotplug, and the `drm_display_info` structure that stores `hdr_sink_metadata` parsed from EDID.

- **Ch20 (Wayland Protocol Fundamentals)** — the Wayland binding and object lifecycle prerequisites for understanding how `wp_color_management_v1` HDR metadata flows from application to compositor (the compositor-side of what this chapter covers on the KMS output side).

- **Ch22 (Production Compositors)** — Mutter and KWin's display pipeline architectures; the HDR capability discovery patterns using `libdisplay-info` described here are used by both compositors.

- **Ch26 (Hardware Video Acceleration)** — VA-API HDR10 decode (`VAProfileHEVCMain10`, P010 surfaces, `VAHdrMetaDataHDR10`) feeds content-level MaxCLL/MaxFALL values that ultimately populate the `hdr_output_metadata` blob described in this chapter.

- **Ch36 (Gamescope)** — Gamescope's AMD DCN plane colour pipeline usage includes setting `HDR_OUTPUT_METADATA` for HDR10 output; its approach to MaxCLL/MaxFALL population from game-supplied HDR metadata is described there.

- **Ch53 (Display Calibration)** — ICC profile measurement and display colorimetry are the SDR counterpart to the HDR metadata described here. `libdisplay-info` is used in both workflows.

- **Ch75 (Explicit GPU Sync)** — HDR atomic KMS commits that change `HDR_OUTPUT_METADATA` simultaneously with framebuffer content require accurate GPU-to-display synchronisation via `drm_syncobj` and timeline semaphores, as covered in Ch75.

- **Ch101 (Color Science)** — The CIE xy chromaticity coordinate system, PQ mathematics, and mastering display colour volume concepts underpinning the `hdr_metadata_infoframe` field encodings are developed in depth there.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
