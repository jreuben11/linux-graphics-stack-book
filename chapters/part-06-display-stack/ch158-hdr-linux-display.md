# Chapter 158: HDR Display Support on Linux

**Target audiences**: Compositor developers adding HDR output support; application developers using HDR colour spaces in Vulkan or EGL; and display engineers integrating HDR metadata signalling via HDMI/DisplayPort.

> **Scope:** This chapter focuses on HDR-specific display signaling (HDMI Dynamic Range InfoFrame, `drm_hdr_output_metadata`, DP HDR descriptor, SMPTE ST 2086 static metadata) and display-side tone-mapping options. **Ch74** is the authoritative chapter for the end-to-end KMS color pipeline (`DEGAMMA_LUT`, `CTM`, `GAMMA_LUT`, per-plane HDR properties) and the `wp_color_management_v1` Wayland protocol. **Ch101** covers ICC profile internals and color science theory. KMS LUT code examples belong in Ch74 — reference it rather than duplicating here.

---

## Table of Contents

1. [Introduction](#introduction)
2. [HDR Fundamentals and Standards](#hdr-fundamentals-and-standards)
3. [EDID HDR Capability Signalling](#edid-hdr-capability-signalling)
4. [DRM HDR Properties](#drm-hdr-properties)
5. [HDR Metadata: SMPTE ST 2086 and CTA-861.3](#hdr-metadata-smpte-st-2086-and-cta-8613)
6. [Color Space Conversion in KMS](#color-space-conversion-in-kms)
7. [Wayland HDR Protocols](#wayland-hdr-protocols)
8. [Compositor HDR: Mutter, KWin, Gamescope](#compositor-hdr-mutter-kwin-gamescope)
9. [Vulkan HDR Swapchain](#vulkan-hdr-swapchain)
10. [Integrations](#integrations)

---

## Introduction

High Dynamic Range (HDR) displays — capable of peak luminances of 400–10,000 nits and wider colour gamuts (DCI-P3, Rec. 2020) — have become common on desktop monitors and TVs. Linux HDR support has matured significantly since 2022, with KMS plane properties, Wayland protocols, and compositor support now available.

The Linux HDR stack involves: DRM kernel properties for signalling HDR metadata to the display, Wayland protocols for surface colour space and HDR intent, and compositor tone-mapping to handle SDR content on HDR displays. Valve's Gamescope compositor led the way with practical HDR implementation; GNOME 47 and KDE Plasma 6 followed.

Sources: [drm HDR docs](https://www.kernel.org/doc/html/latest/gpu/drm-kms.html#hdr-metadata) | [Wayland color protocols](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/tree/main/staging/color-management)

---

## HDR Fundamentals and Standards

### Transfer Functions

| Standard | Transfer Function | Peak Luminance | Gamut |
|---|---|---|---|
| SDR (BT.709) | BT.1886 (gamma 2.4) | 100 nit | sRGB |
| HDR10 | PQ (SMPTE ST 2084) | 1000–10000 nit | BT.2020 |
| HLG | Hybrid Log-Gamma | ~1000 nit | BT.2020 |
| Dolby Vision | PQ + metadata | 4000–10000 nit | BT.2020 |
| scRGB | Linear (linear FP16) | Unbounded | sRGB (extended) |

**PQ (Perceptual Quantizer)**: absolute-luminance EOTF; 0.0 = 0 nit, 1.0 = 10,000 nit. HDR10 uses PQ in a 10-bit BT.2020 frame.

**scRGB**: Microsoft's extended-range linear colour space used by Windows 11 HDR, encoded in FP16 (values >1.0 = above SDR white). Used by Gamescope on Linux for compositing.

### Tone Mapping

When an HDR scene is displayed on an SDR display (or when SDR content plays on an HDR display), the compositor must apply tone mapping:

```
HDR → SDR (tone mapping down):  Reinhard / ACES filmic curve
SDR → HDR (inverse tone mapping): Gamma expansion + gamut mapping
```

Linux compositors implement tone mapping in GPU shaders run during composition.

---

## EDID HDR Capability Signalling

### HDR Static Metadata Block

HDMI displays report HDR capability in the EDID's CEA-861 extension block via the **HDR Static Metadata Descriptor** (data block tag 0x06):

```c
/* Linux kernel: drivers/gpu/drm/drm_edid.c */
struct drm_edid_hdr_metadata_static {
    uint8_t eotf_supported;         /* bit field: SDR, HDR, PQ, HLG */
    uint8_t sm_descriptors;         /* SM Type 1 supported */
    uint8_t max_luminance;          /* peak: code → nit via (50*(2^(code/32))) */
    uint8_t max_frame_avg_luminance;
    uint8_t min_luminance;          /* minimum black level */
};

/* Parse from EDID: */
if (db_tag == CEA_DB_HDR_STATIC_METADATA) {
    parse_hdr_metadata_common(connector, db);
    connector->hdr_sink_metadata.hdmi_type1.eotf =
        db[2] & DRM_EDID_HDMI_HDR_EOTF_MASK;
}
```

Exposed to userspace via the `HDR_OUTPUT_METADATA` connector property.

### DisplayPort HDR

On DisplayPort, HDR capability is in the sink's DPCD (Display Port Configuration Data):

```c
/* drm/display/drm_dp_helper.c */
drm_dp_dpcd_read(aux, DP_DOWNSTREAMPORT_PRESENT, dpcd, sizeof(dpcd));
bool hdr_capable = dpcd[DP_RECEIVER_CAP_SIZE] & DP_EDID_CAPS_HDR_SUPPORT;
```

---

## DRM HDR Properties

### HDR_OUTPUT_METADATA Connector Property

The kernel exposes HDR to userspace via the `HDR_OUTPUT_METADATA` property on DRM connectors:

```c
/* Set HDR metadata on connector (userspace DRM client): */
struct hdr_output_metadata metadata = {
    .metadata_type = HDMI_STATIC_METADATA_TYPE1,
    .hdmi_metadata_type1 = {
        .eotf               = HDMI_EOTF_SMPTE_ST2084,   /* PQ */
        .metadata_type      = HDMI_STATIC_METADATA_TYPE1,
        .max_display_mastering_luminance = 1000,  /* nits */
        .min_display_mastering_luminance = 5,     /* 0.005 nit in 0.0001 nit units */
        .max_cll            = 1000,  /* max content light level (nit) */
        .max_fall           = 400,   /* max frame average light level (nit) */
        /* Colour primaries (BT.2020): */
        .display_primaries  = {
            { 0.708, 0.292 },   /* red   */
            { 0.170, 0.797 },   /* green */
            { 0.131, 0.046 },   /* blue  */
        },
        .white_point = { 0.3127, 0.3290 },
    },
};

uint32_t blob_id;
drmModeCreatePropertyBlob(drm_fd, &metadata, sizeof(metadata), &blob_id);
drmModeAtomicAddProperty(req, connector_id, hdr_output_metadata_prop, blob_id);
drmModeAtomicCommit(drm_fd, req, 0, NULL);
```

### CRTC Colour Space Properties

DRM CRTCs expose the `CTM` (Colour Transform Matrix) and `DEGAMMA_LUT` / `GAMMA_LUT` properties for colour pipeline control:

```c
/* Set colour transform matrix (3×3, fixed-point 31.32): */
struct drm_color_ctm ctm;
/* BT.2020 → BT.709 matrix (simplified): */
ctm.matrix[0] = 1.6605 * (1ULL << 32);
ctm.matrix[4] = 0.8729 * (1ULL << 32);
ctm.matrix[8] = 1.0    * (1ULL << 32);
/* ... off-diagonals ... */

uint32_t ctm_blob;
drmModeCreatePropertyBlob(drm_fd, &ctm, sizeof(ctm), &ctm_blob);
drmModeAtomicAddProperty(req, crtc_id, ctm_prop, ctm_blob);
```

### Plane Colour Encoding

For each plane, the `COLOR_ENCODING` and `COLOR_RANGE` properties select the input colour space:

```c
/* For BT.2020 HDR content on a plane: */
drmModeAtomicAddProperty(req, plane_id, color_encoding_prop,
    DRM_COLOR_YCBCR_BT2020);
drmModeAtomicAddProperty(req, plane_id, color_range_prop,
    DRM_COLOR_YCBCR_LIMITED_RANGE);
```

---

## HDR Metadata: SMPTE ST 2086 and CTA-861.3

### Static Metadata (HDR10)

HDR10 uses static metadata — set once per stream, not per frame. The metadata is carried in:
- HDMI InfoFrame (HDR Dynamic Metadata InfoFrame, HDMI 2.1)
- HDMI Static Metadata Descriptor (for display capability)
- AVI InfoFrame (colour space, colorimetry)

```c
/* drivers/gpu/drm/drm_atomic_helper.c: update HDR InfoFrame */
static void drm_hdmi_set_hdr_infoframe(struct drm_connector *connector,
    const struct hdr_output_metadata *hdr_metadata)
{
    struct hdmi_hdr_infoframe frame;
    hdmi_hdr_infoframe_init(&frame);
    frame.eotf = hdr_metadata->hdmi_metadata_type1.eotf;
    frame.metadata_desc.max_display_mastering_luminance =
        hdr_metadata->hdmi_metadata_type1.max_display_mastering_luminance;
    /* ... fill other fields ... */
    hdmi_hdr_infoframe_pack(&frame, buf, sizeof(buf));
    connector->funcs->hdr_output_metadata_changed(connector);
}
```

### Dynamic Metadata (HDR10+, Dolby Vision)

HDR10+ and Dolby Vision use per-frame dynamic metadata. These are not yet fully supported in the upstream Linux DRM stack — they require proprietary metadata formats and display-side processing.

---

## Color Space Conversion in KMS

### The KMS Colour Pipeline

Modern hardware (AMD DCN, Intel XE) provides a full colour pipeline per CRTC:

```
Plane pixel data
  → Pre-blending CSC (per plane colour matrix)
  → Pre-blending degamma (linearise)
  → Plane blending (alpha composite)
  → Post-blending CSC (output colour matrix)
  → Post-blending gamma / EOTF mapping
  → Connector: HDR metadata InfoFrame
```

DRM exposes this as properties:
- `DEGAMMA_LUT` — linearise input (e.g. sRGB → linear)
- `CTM` — 3×3 matrix (gamut conversion, e.g. BT.2020 → BT.709)
- `GAMMA_LUT` — output transfer function (e.g. linear → PQ)

### Advanced Colour Properties (drm-next 6.8+)

Newer DRM adds per-plane colour pipelines:

```c
/* Plane properties (AMD DCN 3.x, Intel Xe2): */
"PLANE_DEGAMMA_LUT"       /* per-plane input linearisation */
"PLANE_CTM"               /* per-plane gamut matrix */
"PLANE_HDR_MULT"          /* luminance multiplier (FP16) */
```

The Gamescope compositor actively uses these AMD-specific plane properties for per-layer tone mapping.

---

## Wayland HDR Protocols

### color-management-v1 (Staging Protocol)

The `wp_color_management_v1` protocol (in wayland-protocols staging, 2024) lets applications describe their colour space:

```xml
<!-- staging/color-management/color-management-v1.xml -->
<interface name="wp_color_management_surface_v1">
  <request name="set_image_description">
    <arg name="image_description" type="object"
         interface="wp_image_description_v1"/>
    <arg name="render_intent" type="uint"/>
  </request>
</interface>

<interface name="wp_image_description_v1">
  <!-- Describes a colour space (primaries + transfer function): -->
</interface>
```

Applications call `set_image_description` to declare their surface's colour space. The compositor reads this and performs appropriate tone mapping during composition.

### Gamescope's HDR Extension

Gamescope uses a non-standard `gamescope_hdr_surface_v1` extension for immediate HDR intent signalling:

```c
/* gamescope hdr extension (proprietary): */
gamescope_hdr_surface_set_hdr(surface,
    GAMESCOPE_HDR_METADATA_STATIC_MASTERING,
    max_luminance_nits,
    min_luminance_nits * 10000,
    max_cll, max_fall);
```

### frog-color-management-v1

`frog-color-management-v1` is a staging protocol from Joshua Ashton (Valve) used by Gamescope and adopted by KDE:

```xml
<interface name="frog_color_managed_surface">
  <request name="set_known_transfer_function">
    <arg name="transfer_function" type="uint"/>
    <!-- Values: undefined, linear, srgb, st2084_pq, hlg, scrgb_linear -->
  </request>
  <request name="set_known_container_color_volume">
    <arg name="container" type="uint"/>
    <!-- Values: srgb, bt2020 -->
  </request>
</interface>
```

---

## Compositor HDR: Mutter, KWin, Gamescope

### Gamescope HDR

Gamescope (Valve's game compositor) was the first production Linux compositor with full HDR support (2022):

```bash
# Run Gamescope in HDR mode:
gamescope --hdr-enabled --hdr-sdr-content-nits 400 \
    --hdr-itm-enable -- %command%

# Check HDR status:
gamescope --list-displays  # shows HDR capability
```

Gamescope implements:
1. AMD DCN plane `HDR_MULT` for per-layer brightness scaling
2. Tone mapping via GPU compute shaders (Reinhard / ACES)
3. scRGB FP16 compositing colour space
4. Output in PQ + BT.2020 for HDR10

### KDE Plasma 6 HDR

KDE Plasma 6 (2024) added HDR support to KWin:

```bash
# Enable HDR in KDE:
# System Settings → Display and Monitor → HDR → Enable
# Or via kscreen-doctor:
kscreen-doctor output.DP-1.hdr.enable
```

KWin's HDR implementation:
- Uses `wp_color_management_v1` for application colour space declaration
- Tone-maps SDR windows to HDR using a configurable SDR brightness level
- Passes HDR windows through at native PQ

### GNOME Mutter (47+)

GNOME 47 added HDR support:

```bash
# GNOME HDR (experimental in 47):
gsettings set org.gnome.mutter experimental-features "['hdr']"
```

---

## Vulkan HDR Swapchain

### Extension Requirements

```c
/* Required extensions: */
VK_EXT_swapchain_colorspace         /* colour space selection */
VK_KHR_surface                      /* base */
VK_KHR_swapchain

/* Surface capabilities: */
VkSurfaceCapabilities2KHR caps2;
vkGetPhysicalDeviceSurfaceCapabilities2KHR(phys_dev, &surface_info2, &caps2);

/* Enumerate supported formats: */
uint32_t format_count;
vkGetPhysicalDeviceSurfaceFormats2KHR(phys_dev, &surface_info2,
    &format_count, NULL);
VkSurfaceFormat2KHR *formats = malloc(format_count * sizeof(*formats));
vkGetPhysicalDeviceSurfaceFormats2KHR(phys_dev, &surface_info2,
    &format_count, formats);
```

### Selecting HDR Format

```c
/* Find HDR10 format: */
VkFormat hdr_format = VK_FORMAT_UNDEFINED;
VkColorSpaceKHR hdr_colorspace;

for (uint32_t i = 0; i < format_count; i++) {
    VkSurfaceFormatKHR sf = formats[i].surfaceFormat;
    if (sf.format == VK_FORMAT_A2B10G10R10_UNORM_PACK32 &&
        sf.colorSpace == VK_COLOR_SPACE_HDR10_ST2084_EXT) {
        hdr_format     = sf.format;
        hdr_colorspace = sf.colorSpace;
        break;
    }
    /* scRGB (FP16 linear): */
    if (sf.format == VK_FORMAT_R16G16B16A16_SFLOAT &&
        sf.colorSpace == VK_COLOR_SPACE_EXTENDED_SRGB_LINEAR_EXT) {
        hdr_format     = sf.format;
        hdr_colorspace = sf.colorSpace;
        /* continue looking for HDR10 */
    }
}
```

### HDR Swapchain Creation

```c
VkSwapchainCreateInfoKHR swapchain_ci = {
    /* ... standard fields ... */
    .imageFormat      = VK_FORMAT_A2B10G10R10_UNORM_PACK32,
    .imageColorSpace  = VK_COLOR_SPACE_HDR10_ST2084_EXT,
};
vkCreateSwapchainKHR(device, &swapchain_ci, NULL, &swapchain);

/* Shader must output PQ-encoded values in BT.2020:
   linear_nits → PQ_EOTF_inverse(linear_nits / 10000.0)
   Values in [0, 1] map to [0, 10000] nit */
```

### PQ Encoding in Shader

```glsl
/* Convert linear scene-referred luminance to PQ for HDR10 output: */
vec3 linear_to_pq(vec3 linear_nits) {
    const float m1 = 0.1593017578125;
    const float m2 = 78.84375;
    const float c1 = 0.8359375;
    const float c2 = 18.8515625;
    const float c3 = 18.6875;

    vec3 xp = pow(linear_nits / 10000.0, vec3(m1));
    return pow((c1 + c2 * xp) / (1.0 + c3 * xp), vec3(m2));
}
```

---

## Integrations

- **Ch05 (DRM Color Management)** — `DEGAMMA_LUT`, `CTM`, `GAMMA_LUT` CRTC properties are the kernel-side mechanism for HDR colour pipelines; this chapter applies those primitives to HDR
- **Ch06 (KMS Atomic)** — `HDR_OUTPUT_METADATA` is set via atomic commit on the connector; format switching for HDR requires a full modeset
- **Ch35 (Mutter/KWin)** — GNOME and KDE implement compositor-side HDR; this chapter describes their HDR architectures
- **Ch36 (Gamescope)** — Gamescope is the reference implementation for Linux HDR; its AMD DCN plane property usage and tone mapping shaders are described here
- **Ch140 (HDMI/DP Audio)** — HDR metadata InfoFrames travel over the same HDMI AVI/Vendor infoframe channel as audio ELD; the HDMI 2.1 spec defines both
- **Ch150 (EGL/DMA-BUF)** — HDR EGL surfaces need `EGL_GL_COLORSPACE_BT2020_PQ_EXT` from `EGL_EXT_gl_colorspace_bt2020_pq`; DMA-BUF modifiers carry HDR format metadata
