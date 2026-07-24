# Chapter 101: Color Science and the ICC Profile Pipeline

> **Part**: Part VI — The Display Stack
> **Status**: First draft — 2026-06-19

**Target audiences:**
- Photographers
- Video editors
- UI designers
- Display calibration engineers
- Developers building color-correct applications on Linux

This chapter is the color science companion to Chapter 53 (Display Calibration and colord), which covers the calibration *workflow*; here the focus is on the underlying science and the software pipeline that implements it — from the CIE color model through ICC profile binary format, the LittleCMS engine, display calibration tools, and into the Wayland and Vulkan APIs that carry color metadata to the display hardware. Readers of Chapter 74 (HDR and Wide Color Gamut) will find this chapter provides the SDR foundation that HDR extends.

> **Scope:** This chapter covers color science theory and ICC profile internals:
> - CIE XYZ/Lab/xyY
> - Bradford chromatic adaptation
> - ICC v4 binary format
> - LittleCMS `cmsTransform`
> - VCGT curves
>
> **Ch74** owns the KMS hardware pipeline (`DEGAMMA_LUT`, `CTM`, `GAMMA_LUT` properties) and the `wp_color_management_v1` Wayland protocol end-to-end. **Ch158** covers HDR display signaling (HDMI Dynamic Range InfoFrame, DP HDR metadata descriptor, SMPTE ST 2086). Cross-references to KMS LUT configuration should resolve to Ch74.

---

## Table of Contents

1. [Introduction — The Journey from (r, g, b) to Photons](#1-introduction--the-journey-from-r-g-b-to-photons)
   - [1.1 What is Color Management?](#11-what-is-color-management)
   - [1.2 What is a CIE Color Space?](#12-what-is-a-cie-color-space)
   - [1.3 What is an ICC Profile?](#13-what-is-an-icc-profile)
   - [1.4 What is LittleCMS (lcms2)?](#14-what-is-littlecms-lcms2)
2. [CIE Color Science Foundations](#2-cie-color-science-foundations)
3. [Transfer Functions and Gamma](#3-transfer-functions-and-gamma)
4. [ICC Profiles: Structure and Content](#4-icc-profiles-structure-and-content)
5. [LittleCMS (lcms2) — The Reference ICC Engine](#5-littlecms-lcms2--the-reference-icc-engine)
6. [colord: The Color Management Daemon](#6-colord-the-color-management-daemon)
7. [Display Calibration: ArgyllCMS and DisplayCAL](#7-display-calibration-argyllcms-and-displaycal)
8. [Applying Calibration: vcgt vs KMS Color Pipeline](#8-applying-calibration-vcgt-vs-kms-color-pipeline)
9. [Application-Level Color Management](#9-application-level-color-management)
10. [Soft-Proofing and Print Workflows](#10-soft-proofing-and-print-workflows)
11. [HDR Color Management](#11-hdr-color-management)
12. [Integrations](#12-integrations)

---

## 1. Introduction — The Journey from (r, g, b) to Photons

When a shader outputs `vec3(0.8, 0.2, 0.1)`, that triplet carries no absolute meaning on its own. The GPU writes it to a framebuffer, the compositor scans it out to a display, and the display's backlight and phosphors (or OLEDs) convert it to photons. At every step, there is a device-specific transformation: the framebuffer may encode values in sRGB; the GPU may or may not apply a gamma correction; the display's primary colors may deviate from the sRGB reference; and the white point of the room illuminant shifts the viewer's adaptation. Without color management, the photograph that looks perfect on a calibrated wide-gamut monitor in the photographer's studio will appear washed out on a web viewer's laptop and oversaturated in print.

Color science solves this by defining device-*independent* reference spaces through which colors pass on their way between devices. The International Commission on Illumination (CIE) defined such a space — CIE XYZ — from human psychophysical experiments in 1931. The International Color Consortium (ICC) codified a profile format that characterizes each device relative to this reference space. Software color management engines such as LittleCMS (lcms2) apply the profiles to transform pixels between device spaces. And the Linux graphics stack — from KMS color pipeline properties through colord to compositor protocols — carries this metadata from hardware measurement to final display.

This chapter works through each layer in that pipeline: the science, the data formats, the APIs, and their Linux-specific plumbing.

### 1.1 What is Color Management?

Color management is the process of preserving the visual intent of an image as it moves between devices — cameras, displays, printers, projectors — each of which has different physical characteristics. A photograph captured by a camera sensor and stored as RGB values carries no guaranteed meaning: the numbers 0.8, 0.2, 0.1 describe red, green, and blue channel intensities, but the actual wavelengths of light they correspond to depend entirely on the primaries that specific device uses. Without color management, images appear different on different displays — some too saturated, others washed out — because each device maps those code values to different colors of light.

Color management solves this by routing every color transformation through a device-independent reference space. The color management system (CMS) characterizes each device by measuring how it responds to known stimuli, encodes those measurements in a standardized profile format (the ICC profile), and uses a color management engine (CME) to transform pixel data from the source device's color space, through the reference space, into the destination device's color space. On Linux this pipeline begins with hardware measurement tools such as ArgyllCMS, passes through the colord daemon which assigns profiles to connected devices, reaches applications through APIs such as LittleCMS, and ultimately reaches the display hardware through KMS color pipeline properties or the `wp_color_management_v1` Wayland protocol.

### 1.2 What is a CIE Color Space?

The International Commission on Illumination (CIE) defined a family of device-independent color spaces that serve as the common reference in which all device-specific measurements are expressed. The foundation is CIE XYZ, derived from psychophysical experiments measuring how human observers match colored light: the result is three tristimulus values X, Y, Z that uniquely describe any color stimulus visible to the standard observer, independent of which device produced it. Y encodes luminance (perceptual brightness); X and Z carry chromaticity information without direct perceptual significance.

From CIE XYZ, the CIE derived additional spaces suited to different tasks. The xy chromaticity diagram projects XYZ to a 2D plane, enabling visualization of color gamuts as triangles whose vertices are the xy coordinates of device primaries. CIELAB (L\*a\*b\*) applies a non-linear transform to XYZ to produce a more perceptually uniform space in which equal numerical distances correspond more closely to equal perceived color differences — a property raw XYZ lacks. This uniformity makes Lab the practical working space for color difference measurement (the ΔE metric) and gamut-mapping algorithms. The ICC Profile Connection Space (PCS) is defined in CIE XYZ, or equivalently in CIELAB, relative to the D50 reference illuminant, making CIE spaces the language that every ICC-compliant color management engine speaks.

### 1.3 What is an ICC Profile?

An ICC (International Color Consortium) profile is a binary file that characterizes a color device or color space by describing the mathematical relationship between that device's native values and the CIE XYZ reference space. Defined by the ICC specification (currently ICC.1:2022, [icc.color.org](https://www.color.org/specification/ICC.1-2022-05.pdf)), a profile begins with a 128-byte header identifying the profile class (display, printer, scanner, or abstract), the device's native color space, and the rendering intents it supports. The body consists of a tag table followed by tag data elements: each tag carries specific colorimetric data such as primary chromaticity (rXYZ, gXYZ, bXYZ tags), tone response curves (rTRC, gTRC, bTRC), or full multidimensional lookup tables (A2B and B2A tags) for complex device behaviors that cannot be expressed as a simple matrix.

On Linux, ICC profiles for monitors are stored in `/usr/share/color/icc/` (system-wide) or `~/.local/share/icc/` (per-user), and the colord daemon manages which profile is active for each connected display. Applications retrieve the active profile through colord's D-Bus API, then pass it to a color management engine such as LittleCMS to build pixel-transforming pipelines. The kernel itself is profile-unaware; ICC profile application happens in userspace, and the final hardware color correction is typically applied either through Video Card Gamma Table (VCGT) data loaded into the KMS gamma LUT, or through the full KMS color pipeline properties (`DEGAMMA_LUT`, `CTM`, `GAMMA_LUT`) on modern kernels.

### 1.4 What is LittleCMS (lcms2)?

LittleCMS (lcms2) is the open-source color management engine that implements ICC profile interpretation and pixel transformation on Linux and most other platforms. Where ICC profiles are passive data files, lcms2 is the engine that reads them and builds optimized transform pipelines: given a source profile, a destination profile, and a rendering intent, it constructs a `cmsTransform` object that converts pixels from the source color space to the destination color space with ICC-compliant gamut mapping. LittleCMS handles the full ICC tag vocabulary — matrix-shaper profiles for simple display characterizations, CLUT-based A2B/B2A transforms for print devices with non-linear gamut behavior, and device-link profiles that chain multiple transforms into a single optimized operation.

In the Linux graphics stack, lcms2 is used by GIMP, Inkscape, darktable, LibreOffice, and the Ghostscript print pipeline, as well as by colord itself for verifying profile consistency. The lcms2 library exposes a C API whose central types — `cmsHPROFILE`, `cmsHTRANSFORM`, and `cmsColorSpaceSignature` — map directly to ICC profile concepts. A compositor that needs to apply a monitor profile to composited output either uses lcms2 to build a software transform, or extracts the profile's VCGT data and loads it into the KMS display controller's hardware gamma LUT for zero-overhead per-scanout correction. Section 5 of this chapter covers the lcms2 API in detail.

---

## 2. CIE Color Science Foundations

### 2.1 The CIE 1931 XYZ Color Space

In 1931, the CIE published color-matching functions derived from experiments in which human observers matched monochromatic light by mixing three primaries. The result was three functions — x̄(λ), ȳ(λ), z̄(λ) — that describe, at each wavelength λ, how much of each primary a standard observer requires to match that wavelength. Integrating any spectral power distribution S(λ) against these functions yields the *tristimulus values* X, Y, Z:

```
X = ∫ S(λ) x̄(λ) dλ
Y = ∫ S(λ) ȳ(λ) dλ
Z = ∫ S(λ) z̄(λ) dλ
```

Y encodes *luminance* (perceptual brightness): ȳ(λ) was chosen to equal the photopic luminosity function V(λ). X and Z carry chromatic information but no special perceptual significance. CIE XYZ is *device-independent* — it describes the color stimulus itself, not a device's reproduction of it. All ICC profiles use CIE XYZ as the Profile Connection Space (PCS). [Source: CIE 1931 color space on Wikipedia](https://en.wikipedia.org/wiki/CIE_1931_color_space)

### 2.2 xy Chromaticity Diagram

Normalizing away luminance gives the 2D *chromaticity*:

```
x = X / (X + Y + Z)
y = Y / (X + Y + Z)
```

The resulting xy diagram is horseshoe-shaped, with the spectral locus (pure monochromatic colors) forming the boundary. All real colors that the human visual system can perceive fall within this boundary. Color spaces are triangles inside it, with vertices at the xy coordinates of the three primaries:

| Color Space | Red (x, y)   | Green (x, y)  | Blue (x, y)   | White (x, y)     |
|-------------|--------------|---------------|---------------|------------------|
| sRGB / Rec.709 | (0.640, 0.330) | (0.300, 0.600) | (0.150, 0.060) | D65: (0.3127, 0.3290) |
| Display-P3  | (0.680, 0.320) | (0.265, 0.690) | (0.150, 0.060) | D65: (0.3127, 0.3290) |
| BT.2020      | (0.708, 0.292) | (0.170, 0.797) | (0.131, 0.046) | D65: (0.3127, 0.3290) |

Note: **Display-P3** (used by Apple and most consumer wide-gamut displays) shares the DCI-P3 primaries but uses the D65 white point. True **DCI-P3** (digital cinema projectors) uses the same primaries but with a greenish DCI white point near (0.314, 0.351, ≈6300K) — a critical distinction for calibration engineers. The table above gives Display-P3 (D65 white), which is the form referenced in Vulkan's `VK_COLOR_SPACE_DISPLAY_P3_NONLINEAR_EXT` and Apple's color management stack.

BT.2020's primaries lie very close to the spectral locus; no current display technology can produce all of them, but the space defines the outer envelope for HDR and wide-gamut standards. Display-P3 covers roughly 45% of the CIE diagram; most modern smartphone and laptop displays cover 90–100% Display-P3. sRGB covers about 35% and has been the web/desktop standard since 1996. [Source: sRGB specification](https://www.color.org/srgb.pdf)

### 2.3 The D65 White Point

D65 is the CIE standard illuminant representing average northern hemisphere daylight at a correlated color temperature of approximately 6504 K (commonly rounded to 6500 K). At this white point, x = 0.3127, y = 0.3290. All three color spaces in the table above share D65 as their white point, which is why color-managed content appears consistent across them (once transformed). A display factory may calibrate panels to a different white point — sometimes 9300 K for Asian markets — which shifts all reproduced colors until calibrated back to D65.

The ICC Profile Connection Space uses D50 (x = 0.3457, y = 0.3585, ~5003 K) rather than D65, because D50 is the standard for graphic arts viewing and print environments. When storing D65-primary display profiles in the PCS, a Bradford chromatic adaptation transform adapts from D65 to D50. [Source: Nine Degrees Below Photography — sRGB to ICC Profile](https://ninedegreesbelow.com/photography/srgb-color-space-to-profile.html)

### 2.4 CIELAB (CIE L\*a\*b\*) — Perceptual Uniformity

CIE XYZ is not perceptually uniform: equal numerical distances do not correspond to equal perceived color differences. The 1976 CIE L\*a\*b\* space (CIELAB) addresses this. It transforms XYZ non-linearly into:

```
L* = 116 f(Y/Yn) − 16
a* = 500 [f(X/Xn) − f(Y/Yn)]
b* = 200 [f(Y/Yn) − f(Z/Zn)]

where f(t) = t^(1/3)            for t > (6/29)^3
             (1/3)(29/6)^2 t + 4/29   otherwise
```

Here (Xn, Yn, Zn) is the XYZ of the reference white (D50 for ICC). L\* ranges 0–100; a\* and b\* are opponent-color channels (green–red and blue–yellow respectively). The *deltaE* (ΔE) metric measures the Euclidean distance in Lab: ΔE₇₆ = √((ΔL\*)² + (Δa\*)² + (Δb\*)²). A ΔE of 1 is approximately the just-noticeable difference under optimal viewing conditions. Print and proofing workflows target ΔE ≤ 2. [Source: CIELAB on Wikipedia](https://en.wikipedia.org/wiki/CIELAB_color_space)

### 2.5 LCH — Polar Form of Lab

LCH transforms Lab into a polar coordinate system more intuitive for designers:

```
L* — lightness (same as Lab)
C* = √(a*² + b*²)    — chroma (saturation)
H  = atan2(b*, a*) × (180/π)  — hue angle, 0–360°
```

Red is near 40°, yellow near 90°, green near 130°, blue near 270°. LCH appears in perceptual gamut-mapping algorithms: when an out-of-gamut color must be clipped, LCH-based clipping preserves hue better than clipping in RGB or Lab. Newer color spaces such as OKLab and OKLCH (used in CSS Color Level 4) improve on Lab's residual non-uniformity, but Lab/LCH remain the dominant representation in ICC color management engines.

---

## 3. Transfer Functions and Gamma

### 3.1 Gamma Encoding

Early CRT displays had a power-law input-to-output response: luminance ≈ voltage^γ with γ ≈ 2.2. Encoding images as v^(1/γ) before storage compensates for this response so the display emits linear-looking light. As a side effect, gamma encoding allocates more code values in dark regions — where human vision is more sensitive — which is efficient for 8-bit storage.

### 3.2 The sRGB Transfer Function

sRGB (IEC 61966-2-1:1999) does not use a pure power law. Instead it defines a piecewise function:

**Encoding (linear → sRGB):**

```
C_srgb = 12.92 × C_linear                    if C_linear ≤ 0.0031308
C_srgb = 1.055 × C_linear^(1/2.4) − 0.055   if C_linear > 0.0031308
```

**Decoding (sRGB → linear):**

```
C_linear = C_srgb / 12.92                             if C_srgb ≤ 0.04045
C_linear = ((C_srgb + 0.055) / 1.055)^2.4            if C_srgb > 0.04045
```

The threshold 0.04045 in the decoder corresponds to the encoding threshold 0.0031308 at the 12.92 linear slope. The exponent 2.4 (not 2.2) is the nominal gamma of the power segment; the overall effective gamma of the full piecewise curve is approximately 2.2 for most of the range, which is why "gamma 2.2" is a common approximation. [Source: sRGB formula analysis](https://entropymine.com/imageworsener/srgbformula/)

### 3.3 Linear vs. sRGB in Shaders

Lighting calculations (Phong, PBR, shadows) must be performed in linear light. Blending two half-intensity sRGB values incorrectly: 50% gray in sRGB is approximately 21.4% linear intensity, so blending two 50% grays gives 50% sRGB, but their actual intensities summed are 42.8% — not 50%. The result is noticeably too dark along alpha-blended edges and incorrectly bright in specular highlights.

The correct example of gamma-blending error: if you blend black (0.0 sRGB, 0.0 linear) and white (1.0 sRGB, 1.0 linear) as a 50/50 alpha composite naively in sRGB, you get 0.5 sRGB ≈ 0.214 linear. The physically correct midpoint is 0.5 linear ≈ 0.735 sRGB — noticeably brighter. Doing lighting, shadows, and compositing in gamma-encoded space produces edges that look too dark and specular highlights that are incorrectly clipped.

OpenGL: `GL_FRAMEBUFFER_SRGB` enables automatic linearization on read and gamma-encoding on write for sRGB framebuffer attachments (`GL_SRGB8_ALPHA8` format). Shaders operate in linear light; the fixed function converts on the way in/out. Without this enable, the GPU performs no conversion and blending operates on gamma-compressed values.

Vulkan: the distinction is in the image format. `VK_FORMAT_R8G8B8A8_SRGB` causes the hardware to linearize on sample and encode on store (in render passes). `VK_FORMAT_R8G8B8A8_UNORM` treats values as raw linear and applies no conversion. A shader sampling an sRGB texture should use the `_SRGB` format; if it must perform its own gamma math (e.g., because it needs direct access to the stored values), it samples `_UNORM`. [Source: Vulkan specification — format descriptions](https://registry.khronos.org/vulkan/specs/1.3/html/chap43.html)

### 3.4 OETF, EOTF, and the Standards Vocabulary

The color science community uses precise terms:

- **OETF** (Opto-Electronic Transfer Function): camera → signal encoding (gamma encoding, what a camera sensor applies before writing a video file)
- **EOTF** (Electro-Optical Transfer Function): signal → display luminance (what a monitor applies to render the signal). The sRGB decode function above is an EOTF.
- **OOTF** (Opto-Optical Transfer Function): the scene-to-display luminance mapping that results from the OETF and EOTF chain, possibly including any rendering pipeline adjustments. For SDR Rec.709 content, OOTF ≈ identity (the system is calibrated to cancel). For HDR HLG content, OOTF is explicitly defined by ITU-R BT.2100 to account for different viewing environments.

### 3.5 HDR Transfer Functions: PQ and HLG

**PQ (Perceptual Quantizer, SMPTE ST 2084):** maps 0–10,000 cd/m² absolute luminance to code values using a curve tuned to the threshold of human contrast sensitivity. At 0.01 nits, each code step is ≈ 0.0001 nits; at 1,000 nits, each step is ≈ 0.15 nits. The result is perceptually uniform across the full HDR range. PQ code values are absolute (not scene-referred); a code value of 0.5 means approximately 130 nits on any compliant display. PQ is not backward-compatible: an SDR display receiving a PQ signal renders it far too dark.

**HLG (Hybrid Log-Gamma, ITU-R BT.2100):** jointly developed by the BBC and NHK, HLG is scene-referred. The lower segment (E ≤ 1/12) is `E' = √(3E)` — a gamma-2.0-like curve compatible with SDR displays. The upper segment uses `E' = a·ln(12E − b) + c`. An SDR display sees the linear+log signal as a plausible image; an HDR display recovers extended range from the upper segment. HLG's OOTF is explicitly defined because the scene-referred encoding must be mapped to the display's luminance range at decode time.

See Chapter 74 for the complete mathematical definitions and constants for both transfer functions.

---

## 4. ICC Profiles: Structure and Content

### 4.1 Binary Format Overview

An ICC profile is a portable binary blob. Its structure:

```
[128-byte header]
[4-byte tag count]
[tag count × 12-byte tag table entries]
[tag data elements, variable length, at addressed offsets]
```

The header carries the metadata that allows any ICC-aware tool to parse and use the profile without reading any tags:

| Offset | Size | Field |
|--------|------|-------|
| 0 | 4 | Profile size (big-endian uint32) |
| 4 | 4 | Preferred CMM (engine signature, e.g., `lcms` = 0x6C636D73) |
| 8 | 4 | Profile version (e.g., 0x04400000 = v4.4.0.0) |
| 12 | 4 | Profile class (`mntr`=Display, `prtr`=Output, `scnr`=Input, `link`=DeviceLink) |
| 16 | 4 | Data color space (e.g., `RGB ` = 0x52474220) |
| 20 | 4 | PCS (always `XYZ ` or `Lab `) |
| 24 | 12 | Creation date/time (Y, M, D, h, m, s each uint16) |
| 36 | 4 | Profile file signature: `acsp` = **0x61637370** |
| 40 | 4 | Primary platform (`APPL`, `MSFT`, `*nix`) |
| 44 | 4 | Profile flags |
| 48 | 4 | Device manufacturer |
| 52 | 4 | Device model |
| 56 | 8 | Device attributes |
| 64 | 4 | Rendering intent (0=perceptual, 1=relative colorimetric, 2=saturation, 3=absolute colorimetric) |
| 68 | 12 | PCS illuminant (D50 in XYZ = 0xF6D6, 0x10000, 0xD32D in s15Fixed16) |
| 80 | 4 | Profile creator signature |
| 84 | 16 | Profile ID (MD5 of profile with this field zeroed) |

[Source: ICC.1:2022 specification](https://www.color.org/specification/ICC.1-2022-05.pdf)

### 4.2 Tag Table

Immediately after the header, a uint32 tag count is followed by that many 12-byte entries:

```
struct icc_tag_entry {
    uint32_t signature;   /* 4-char tag signature, e.g. 'rXYZ' */
    uint32_t offset;      /* byte offset from start of profile */
    uint32_t size;        /* size of tag data in bytes */
};
```

Tags can be shared (multiple entries pointing to the same offset), and the data section is unordered. The ICC specification requires 4-byte alignment for each tag.

### 4.3 Required Tags for Matrix+TRC Display Profiles

The simplest and most common display profile type (the "matrix+TRC" model) requires these tags:

| Tag Signature | Content |
|---------------|---------|
| `rXYZ` | Red primary XYZ in the PCS (D50-adapted) |
| `gXYZ` | Green primary XYZ |
| `bXYZ` | Blue primary XYZ |
| `rTRC` | Red Tone Reproduction Curve |
| `gTRC` | Green TRC |
| `bTRC` | Blue TRC |
| `wtpt` | Media white point in XYZ |
| `bkpt` | Media black point (optional but common) |
| `cprt` | Copyright string |
| `desc` | Profile description (multilingual string) |

For sRGB, the `rXYZ`/`gXYZ`/`bXYZ` tags contain the Bradford-adapted D65 primaries. The `rTRC`/`gTRC`/`bTRC` tags contain the sRGB piecewise function, either as a parametric curve tag (`para`) with the appropriate parameters or as a sampled 1D LUT (`curv` with 1024 or 4096 entries).

To inspect a profile:

```bash
# iccdump from ArgyllCMS (most complete, understands all tag types including vcgt)
iccDump -v 3 /usr/share/color/icc/colord/sRGB.icc

# colprof from ArgyllCMS can dump profile information
# Some distros also ship `iccexamin` (GUI) or `icc2ps` for profile visualization
```

### 4.4 LUT-Based Profiles

For devices with non-linear or channel-interaction color responses — inkjet printers, scanners — the matrix model is insufficient. LUT-based profiles use multidimensional color lookup tables:

- **A2B0** tag: device to PCS transform (forward direction). Contains: input 1D curves → N-dimensional CLUT → output 1D curves.
- **B2A0** tag: PCS to device transform (inverse). Applies the rendering intent (different B2A tags for different intents: `B2A0`=perceptual, `B2A1`=relative colorimetric, `B2A2`=saturation).

A CMYK printer profile typically uses a 9×9×9×9 CLUT for the B2A direction, meaning 9 sample points along each of the four device ink channels, totaling 9⁴ = 6,561 XYZ samples that the CMM interpolates.

### 4.5 ICC v2 vs. v4

ICC v2 profiles (version field 0x02) differ from v4 in several ways relevant to software implementors:

- **PCS encoding:** v2 uses media-relative colorimetry throughout the PCS; v4 adds ICC-absolute colorimetry encoded differently in the tag headers. The illuminant XYZ in the v4 header is always the actual D50 XYZ; in v2 it was always Y = 1.0 regardless.
- **Tag encoding:** v4 standardizes `para` (parametric curve) tag more strictly; v2 implementations often used `curv` with sampled data for everything.
- **Gray TRC:** v4 permits a single `kTRC` tag for grayscale output profiles; v2 requires three separate channels even for gray.
- **Interoperability:** lcms2 handles both transparently. However, some embedded implementations (older printers, scanner firmware) support only v2. GIMP has historically preferred v2 profiles for compatibility.

The current specification is [ICC.1:2022 (v4.4)](https://www.color.org/specification/ICC.1-2022-05.pdf); an April 2025 amendment added an Adaptive Gain Curve tag for perceptual HDR rendering within the ICC framework.

---

## 5. LittleCMS (lcms2) — The Reference ICC Engine

### 5.1 Overview

LittleCMS 2 (lcms2) is the dominant open-source ICC color management engine on Linux, used by GIMP, Krita, Darktable, Inkscape, libvips, and numerous others. As of April 2026, the current release is version 2.19. It is written in C, has a small footprint, and runs on all major platforms. (Firefox uses its own **qcms** library; Chrome/Skia use **skcms** — both purpose-built replacements that avoid lcms2's dependency.) [Source: LittleCMS](https://www.littlecms.com/)

### 5.2 Basic Transform Creation

The core workflow is: open profiles → create transform → apply to pixels.

```c
#include <lcms2.h>

/* Open profiles from file or memory */
cmsHPROFILE srcProfile = cmsOpenProfileFromFile("/usr/share/color/icc/sRGB.icc", "r");
cmsHPROFILE dstProfile = cmsOpenProfileFromFile("AdobeRGB1998.icc", "r");

if (!srcProfile || !dstProfile) {
    /* Handle error: file not found, or not an ICC profile */
    ...
}

/* Create a transform.
 * TYPE_RGB_8     = 8-bit interleaved RGB input/output
 * INTENT_RELATIVE_COLORIMETRIC = rendering intent
 * 0              = no flags (use cmsFLAGS_* constants for options)
 */
cmsHTRANSFORM hTransform = cmsCreateTransform(
    srcProfile,  TYPE_RGB_8,
    dstProfile,  TYPE_RGB_8,
    INTENT_RELATIVE_COLORIMETRIC,
    0);

/* Apply to a raster (width × height × 3 bytes) */
cmsDoTransform(hTransform, src_pixels, dst_pixels, width * height);

/* Cleanup */
cmsDeleteTransform(hTransform);
cmsCloseProfile(srcProfile);
cmsCloseProfile(dstProfile);
```

The `TYPE_*` constants encode pixel format (bit depth, channel count, channel order, planar vs. interleaved, premultiplied alpha) in a compact uint32. Common values:

| Constant | Meaning |
|----------|---------|
| `TYPE_RGB_8` | 8-bit interleaved RGB |
| `TYPE_RGBA_8` | 8-bit interleaved RGBA |
| `TYPE_RGB_16` | 16-bit interleaved RGB |
| `TYPE_Lab_8` | 8-bit CIELab |
| `TYPE_CMYK_8` | 8-bit interleaved CMYK |
| `TYPE_GRAY_8` | 8-bit grayscale |

[Source: lcms2 API in GitHub](https://github.com/mm2/Little-CMS/blob/master/include/lcms2.h)

### 5.3 Rendering Intents

ICC defines four rendering intents, used when a color cannot be reproduced exactly (out-of-gamut) or when the white points of source and destination differ:

**Perceptual (Intent 0):** Compresses the entire source gamut into the destination gamut, preserving hue relationships at the cost of some saturation and absolute accuracy. Best for photographs where overall tonal balance matters more than exact reproduction of any particular color.

**Relative Colorimetric (Intent 1):** Maps the source white point to the destination white point (chromatic adaptation), then clips any remaining out-of-gamut colors. Colors within both gamuts are reproduced exactly. Best for converting between similar gamuts (sRGB to P3) where most colors are in-gamut.

**Saturation (Intent 2):** Maximizes saturation at the expense of hue accuracy. Used for business graphics (pie charts, logos) where vivid colors matter more than accuracy.

**Absolute Colorimetric (Intent 3):** No white-point mapping; absolute luminance is preserved. Used for soft-proofing to simulate a print substrate's white point — the simulation appears slightly off-white to represent paper color.

The appropriate constants are defined in `lcms2.h`:

```c
#define INTENT_PERCEPTUAL               0
#define INTENT_RELATIVE_COLORIMETRIC    1
#define INTENT_SATURATION               2
#define INTENT_ABSOLUTE_COLORIMETRIC    3
```

### 5.4 Thread Safety

`cmsCreateTransform` is *not* inherently thread-safe when using global (non-context) objects if the CMM's internal tables are modified concurrently. The correct pattern for multi-threaded applications is to use the context API:

```c
/* Create a per-thread context */
cmsContext ctx = cmsCreateContext(NULL, NULL);

/* Open profiles within the context */
cmsHPROFILE src = cmsOpenProfileFromFileTHR(ctx, "sRGB.icc", "r");
cmsHPROFILE dst = cmsOpenProfileFromFileTHR(ctx, "AdobeRGB.icc", "r");

cmsHTRANSFORM xform = cmsCreateTransformTHR(
    ctx, src, TYPE_RGB_8, dst, TYPE_RGB_8,
    INTENT_RELATIVE_COLORIMETRIC, 0);

/* This is thread-safe: */
cmsDoTransformLineStride(
    xform,
    src_buf, dst_buf,    /* input / output buffers */
    width,               /* pixels per line */
    height,              /* number of lines */
    src_stride,          /* input bytes per line */
    dst_stride,          /* output bytes per line */
    0, 0);               /* planar channel offsets (0 = interleaved) */

cmsDeleteTransform(xform);
cmsCloseProfile(src);
cmsCloseProfile(dst);
cmsDeleteContext(ctx);
```

Once a transform is created, `cmsDoTransform` and `cmsDoTransformLineStride` are both thread-safe: multiple threads may call them on the same transform object simultaneously, because the transform's internal pipeline is read-only after construction.

### 5.5 Tone Curves and the vcgt Profile Tag

lcms2 provides direct access to tone curves for gamma and TRC manipulation:

```c
/* Build a gamma 2.2 curve */
cmsToneCurve *gamma22 = cmsBuildGamma(NULL, 2.2);

/* Build a parametric sRGB-style curve (type 4: IEC 61966-2-1) */
/* Parameters: [gamma, a, b, c, d, e, f] for the IEC 61966-2-1 piecewise form */
cmsFloat64Number params[5] = {2.4, 1.0/1.055, 0.055/1.055, 1.0/12.92, 0.04045};
cmsToneCurve *srgbTRC = cmsBuildParametricToneCurve(NULL, 4, params);

/* Evaluate the curve at a single value */
cmsFloat32Number out = cmsEvalToneCurveFloat(srgbTRC, 0.5f);  /* → ~0.2140 */

cmsFreeToneCurve(gamma22);
cmsFreeToneCurve(srgbTRC);
```

The **vcgt** (Video Card Gamma Tag) is not part of the ICC specification core; it is a private tag (signature `vcgt`) commonly added by profiling software to store per-channel 1D LUTs for loading into the GPU's RAMDAC. lcms2's `cmsReadTag` can extract the vcgt data, and the `dispwin` and `xcalib` tools use this to program hardware gamma.

### 5.6 Major Consumers of lcms2

- **GIMP 3.x:** `app/operations/gimp-operation-color-*` and the color management pipeline in `app/config/gimpcoreconfig.c`. GIMP 3 supports per-image and per-layer color spaces, all routed through lcms2.
- **Krita:** `libs/pigment/` — `KoColorTransformation`, `KoColorConversionSystem`. Krita supports per-layer color spaces and HDR painting in ScRGB (linear extended-range sRGB).
- **Darktable:** `src/iop/colorout.c` — the *output color profile* module selects the display rendering intent and passes pixels through lcms2 before display. Input profile comes from libraw's camera metadata.
- **Inkscape:** uses lcms2 for color proof display and SVG color management.
- **libvips:** `libvips/colour/icc_*.c` — uses lcms2 for efficient pipeline-based color conversion on large images.
- **Firefox:** uses **qcms** (Firefox's own Rust color management library, `gfx/qcms/`) to transform embedded ICC profiles in images to the display color space. Firefox switched from LittleCMS to qcms in Firefox 3.5 for security and maintenance reasons. [Source: qcms on GitHub](https://github.com/FirefoxGraphics/qcms)

Note: The exact source paths above are correct as of early 2026 but should be verified against each project's current upstream for minor path changes.

---

## 6. colord: The Color Management Daemon

### 6.1 Architecture

colord is a system daemon that acts as a persistent registry for device–profile associations on Linux. It exposes a D-Bus API on `org.freedesktop.ColorManager` and stores device→profile mappings, installed profiles, and sensor information. On session start, compositors and desktop environments query colord to load the active ICC profile for each connected display. [Source: colord on GitHub](https://github.com/hughsie/colord)

The key D-Bus object types:

| Object | D-Bus Interface | Purpose |
|--------|-----------------|---------|
| `CdClient` | `org.freedesktop.ColorManager` | Session entry point; enumerate devices and profiles |
| `CdDevice` | `org.freedesktop.ColorManager.Device` | Represents a physical device (display, printer, scanner) |
| `CdProfile` | `org.freedesktop.ColorManager.Profile` | An ICC profile with metadata |
| `CdSensor` | `org.freedesktop.ColorManager.Sensor` | A connected colorimeter or spectrophotometer |

### 6.2 Device–Profile Mapping Workflow

```
Application boots
  → colord D-Bus client queries CdClient.GetDevices()
  → For the display CdDevice, call CdDevice.GetProfiles()
  → The first returned CdProfile is the active one
  → Fetch CdProfile.Filename → load into lcms2 or compositor
```

Profile associations persist in `/var/lib/colord/` as a GKeyFile-based database. User-installed profiles go in `~/.local/share/icc/`; system-wide profiles in `/usr/share/color/icc/` and `/var/lib/colord/icc/`. colord watches these directories via inotify and automatically registers new profiles.

### 6.3 colormgr CLI

```bash
# List all registered devices
colormgr get-devices

# List all registered profiles
colormgr get-profiles

# Associate a profile with a device
colormgr device-add-profile /org/freedesktop/ColorManager/devices/xrandr_LG_HDR_4K \
    /org/freedesktop/ColorManager/profiles/icc_abc12345

# Query sensor information
colormgr get-sensors

# Import an ICC profile (copies to user ICC dir and registers)
colormgr import-profile ~/Downloads/MyMonitor.icc
```

### 6.4 Compositor Integration

GNOME (Mutter) and KDE Plasma (KWin) both integrate colord:

- **GNOME Color Manager** (`gnome-color-manager`): a GUI frontend for colord. It displays connected devices, shows their active profiles, and provides a calibration workflow dialog that launches ArgyllCMS.
- **colord-kde**: a KDE-specific service that bridges colord's D-Bus API into KDE's settings framework, allowing KWin to pick up the active profile automatically.
- On X11 sessions: the active profile's vcgt tag is loaded into the GPU's RAMDAC using `xcalib` or ArgyllCMS's `dispwin`.
- On Wayland sessions: the compositor queries colord via D-Bus and programs the KMS `GAMMA_LUT` or `CTM` properties directly (see §8).

### 6.5 cd-sensor: Colorimeter and Spectrophotometer Access

colord includes a sensor daemon component (`cd-sensor`) that abstracts access to hardware colorimeters. Supported hardware includes X-Rite i1Display series, Calibrite ColorChecker Display, and Datacolor Spyder series. `cd-sensor` exposes each sensor as a `CdSensor` D-Bus object with methods for taking spot measurements and running measurement sweeps. ArgyllCMS can also talk directly to the sensor hardware via its own hidraw/libusb driver layer, bypassing colord's sensor abstraction for maximum precision.

---

## 7. Display Calibration: ArgyllCMS and DisplayCAL

### 7.1 ArgyllCMS — The Measurement and Profile Engine

ArgyllCMS is the reference open-source colorimetric measurement and profiling tool on Linux. It speaks directly to colorimeter and spectrophotometer hardware and implements the full ICC profile generation pipeline. Version 3.5.0 is current as of 2026. [Source: ArgyllCMS](https://www.argyllcms.com/)

**Full calibration + profiling workflow:**

```bash
# Step 1: Calibrate — iteratively adjust the display's hardware controls
# to reach target: D65 white point (-t 6500), 120 cd/m² luminance (-b 120),
# gamma 2.2 (-g 2.2), display 1 (-d 1), verbose output (-v), LCD type (-yl)
dispcal -v -yl -t 6500 -b 120 -g 2.2 -d 1 monitor_cal

# Step 2: Generate a test target — 400 patches (-f 400) for RGB display (-d 3)
targen -d 3 -f 400 monitor_cal

# Step 3: Measure the patches on the display
# (-d 1: display device 1; reads monitor_cal.ti2, writes monitor_cal.ti3)
dispread -d 1 monitor_cal

# Step 4: Build the ICC profile from measurements
# -v: verbose; -D "name": description string; -qm: medium quality;
# -b m: medium bandwidth for gamut; -a m: matrix+curves output type
colprof -v -D "My Monitor Profile" -qm -b m -a m monitor_cal

# Output: monitor_cal.icc — install with colord
colormgr import-profile monitor_cal.icc
```

`dispcal` performs the calibration by displaying a series of gray patches and adjusting the display's hardware controls (brightness, contrast, RGB gain) interactively. The calibration curves are baked into the profile's vcgt tag.

`targen` generates a set of measurement patches distributed through color space. More patches produce a more accurate profile; `-f 400` is a common trade-off between measurement time (~10 minutes) and accuracy.

`dispread` sends each patch to the display and measures it with the attached colorimeter. It writes a `.ti3` file containing per-patch XYZ measurements.

`colprof` fits an ICC profile to the measurements. The `-a m` flag selects a matrix+curves model (sufficient for well-behaved displays); `-a g` would select a more complex LUT-based model for displays with channel interactions.

### 7.2 Supported Hardware

| Category | Examples |
|----------|---------|
| Colorimeters (fast, ~$100–200) | X-Rite i1Display Pro, Calibrite ColorChecker Display, Datacolor Spyder X |
| Spectrophotometers (accurate, ~$1,000+) | X-Rite i1Pro 3, Klein K-10 |
| High-end reference | Konica Minolta CA-410, Photo Research PR-740 |

Colorimeters measure fast and are adequate for most display profiling. Spectrophotometers measure actual spectral power distributions and are required for print profiling and HDR display characterization.

### 7.3 DisplayCAL

DisplayCAL is a Python GUI application that wraps ArgyllCMS, providing a user-friendly calibration and profiling experience. The `displaycal-py3` fork maintains active Python 3 compatibility. [Source: DisplayCAL](https://displaycal.net/)

DisplayCAL adds:
- Colorimeter correction matrices (CCMXs): per-display correction files for colorimeters that were characterized against a reference spectrophotometer
- 3D gamut visualizations of the resulting profile
- Delta-E verification charts comparing the profile prediction against additional measurements
- Integration with colord for automatic profile registration

### 7.4 Profile Output: the vcgt Tag

Both `dispcal` and `colprof` embed a `vcgt` (Video Card Gamma Table) tag in the output profile. This tag stores per-channel 1D LUTs (typically 256 or 1024 entries, 16 bits each) that represent the calibration corrections needed to bring the display to the target gamma and white point. Loading this LUT into the GPU's hardware gamma transforms input pixel values before they reach the display, compensating for the display's native non-linearity.

---

## 8. Applying Calibration: vcgt vs KMS Color Pipeline

### 8.1 The vcgt Mechanism (X11)

On X11 sessions, the vcgt LUT is loaded into the GPU's RAMDAC (Random Access Memory Digital-to-Analog Converter) via the XF86VidMode or RANDR extension. Two tools perform this:

```bash
# xcalib: load vcgt from an ICC profile into the X11 gamma ramp
xcalib -d :0 -s 0 /usr/share/color/icc/MyMonitor.icc

# ArgyllCMS dispwin: load calibration with more options
dispwin -d 1 -L monitor_cal.icc     # -L: load LUT from vcgt tag
dispwin -d 1 -c monitor_cal.icc     # -c: clear/reset gamma
```

The xcalib-loaded LUT has 256 entries per channel (R/G/B), each 16 bits. Older RAMDAC hardware was limited to 256×8-bit; modern hardware typically stores 1024 or 4096 entries.

### 8.2 The KMS Color Pipeline (Wayland)

On Wayland, the X11 RAMDAC interface is unavailable. Instead, compositors use the KMS atomic property interface directly:

```c
/* Legacy API (deprecated for atomic): */
drmModeSetGamma(fd, crtc_id, gamma_size, red, green, blue);

/* Modern atomic API: */
/* GAMMA_LUT is a CRTC property containing a drm_color_lut array */
/* DEGAMMA_LUT: applies linearization before the CTM */
/* CTM: 3×3 matrix in s31.32 fixed-point for primary gamut mapping */

/* Setting via atomic commit: */
drmModeAtomicAddProperty(req, crtc_id, gamma_lut_prop_id, lut_blob_id);
drmModeAtomicAddProperty(req, crtc_id, ctm_prop_id, ctm_blob_id);
drmModeAtomicCommit(fd, req, DRM_MODE_ATOMIC_ALLOW_MODESET, NULL);
```

The `drm_color_lut` struct used for `GAMMA_LUT` values:

```c
/* from include/uapi/drm/drm_mode.h */
struct drm_color_lut {
    __u16 red;    /* 16-bit value, range [0, 65535] */
    __u16 green;
    __u16 blue;
    __u16 reserved;
};
```

[Source: Linux kernel drm_mode.h](https://github.com/torvalds/linux/blob/master/include/uapi/drm/drm_mode.h)

### 8.3 KMS CTM — Color Transformation Matrix

The CTM (Color Transformation Matrix) property takes a 3×3 matrix in `drm_color_ctm` format:

```c
/* from include/uapi/drm/drm_mode.h */
struct drm_color_ctm {
    /*
     * Conversion matrix in S31.32 fixed-point format.
     * Negative values use two's complement: bit 63 is sign,
     * bits 62–32 are integer part, bits 31–0 are fractional.
     * Matrix is row-major: [0][0..2] = row 0, [1][0..2] = row 1, etc.
     */
    __u64 matrix[9];
};
```

The CTM is applied in linear light between `DEGAMMA_LUT` (linearization) and `GAMMA_LUT` (re-encoding):

```
framebuffer pixels
  → DEGAMMA_LUT (linearize from sRGB/gamma encoding)
  → CTM (3×3 matrix: gamut mapping, white balance, night mode)
  → GAMMA_LUT (apply display calibration + re-encode for display)
  → display panel
```

This pipeline matches what ICC profiles do in software: the TRC tags linearize (DEGAMMA), the primary matrix maps gamut (CTM), and the output TRC re-encodes for the display (GAMMA_LUT). When a compositor applies an ICC profile via KMS, it decomposes the profile into these hardware stages.

### 8.4 Accuracy Comparison

| Mechanism | LUT size | Bit depth | Gamut mapping |
|-----------|----------|-----------|---------------|
| Legacy vcgt/xcalib | 256 entries | 8-bit output | None (1D only) |
| KMS GAMMA_LUT (legacy) | 256 entries | 16-bit | None (1D only) |
| KMS GAMMA_LUT (atomic, typical AMD/Intel) | 1024–4096 entries | 16-bit | Via separate CTM |
| KMS CTM | 3×3 float matrix | s31.32 fixed-point | Yes |

The combination of CTM + GAMMA_LUT provides the most accurate color-managed display path. However, CTM support is driver-dependent: AMD (amdgpu) and Intel (i915) expose CTM via atomic properties; NVIDIA's proprietary driver and older open-source drivers may not. For drivers without CTM support, compositors must fall back to applying the ICC matrix in software (shader-based color management) before the pixels reach the KMS plane.

### 8.5 Compositor Implementations

**Mutter (GNOME):** Queries colord via D-Bus for the active profile on each monitor, decomposes it into DEGAMMA/CTM/GAMMA blobs, and programs them via KMS atomic commit. The implementation lives in `src/backends/native/meta-kms-crtc.c` and related files.

**KWin (KDE Plasma):** Similarly programs KMS color properties and implements the `wp_color_management_v1` Wayland protocol, allowing clients to attach image descriptions with ICC profile references to their surfaces.

Note: verify current source paths against upstream as Mutter/KWin are actively evolving their color management implementations.

---

## 9. Application-Level Color Management

### 9.1 GIMP

GIMP 3.x implements a full color-managed pipeline:

- Each image carries an embedded ICC profile. Images loaded without a profile are assumed sRGB.
- *Edit > Color Management* sets the display profile (queried from colord), the rendering intent for display, and soft-proof options.
- All color operations occur in the image's working color space. GIMP uses lcms2 to transform between the working space and the display profile before pixels are shown on screen.
- *Image > Color Management > Assign Color Profile* changes the profile tag without converting pixels (changes interpretation).
- *Image > Color Management > Convert to Color Profile* transforms pixels from current to target profile.
- Soft-proofing: *View > Color Management > Proof Colors* enables a two-pass transform through the selected proofing (printer) profile, simulating print output on screen.

[Source: GIMP 3.0 Color Management documentation](https://docs.gimp.org/3.0/en/gimp-imaging-color-management.html)

### 9.2 Krita

Krita's color management architecture (`libs/pigment/`) is more deeply integrated than GIMP's:

- **`KoColorSpace`:** abstract color space with specific implementations for sRGB, Lab, CMYK, XYZ, etc. Each KoColorSpace holds an lcms2 profile handle.
- **`KoColorTransformation`:** lcms2 transform wrapper, applied per-operation (brushstroke, filter, layer composite).
- **Per-layer color spaces:** each paint layer can operate in a different color space. Krita composites layers by converting all to the document's working space.
- **HDR painting:** Krita supports 16-bit and 32-bit float (ScRGB, linear P3) working spaces, enabling HDR content creation.
- **Soft-proofing:** live soft-proof in the canvas using a printer ICC profile and the absolute colorimetric intent to simulate paper white.

[Source: Krita Color Managed Workflow documentation](https://docs.krita.org/en/general_concepts/colors/color_managed_workflow.html)

### 9.3 Darktable

Darktable implements a fully linear light processing pipeline:

1. Raw decoder (RawSpeed) → camera-native linear values
2. *Input color profile* module: reads camera ICC/matrix from metadata or embeds a DCP profile
3. All subsequent processing in linear light (exposure, denoise, tone mapping, color grading)
4. *Output color profile* module (`src/iop/colorout.c`): converts from the working space to the display or export profile using lcms2 with the selected rendering intent
5. Display: passes the final values through the display profile for color-accurate preview

The `dt_iop_colorout` module supports soft-proofing and gamut-check visualization for print preflighting within the darkroom view. [Source: darktable colorout module](https://github.com/darktable-org/darktable/blob/master/src/iop/colorout.c)

### 9.4 Firefox

Firefox color-manages images using **qcms** — its own Rust-based color management library living in `gfx/qcms/`. JPEG, PNG, and WebP images with embedded ICC profiles are transformed to the display color space by qcms. Firefox switched from LittleCMS to qcms at Firefox 3.5 for security and maintenance reasons; qcms currently supports ICC v2 profiles, with v4 support being an ongoing effort. [Source: qcms Bugzilla — ICC v4 support](https://bugzilla.mozilla.org/show_bug.cgi?id=488800)

The relevant preference in `about:config` is `gfx.color_management.mode`:

```
0 = disabled (treat everything as sRGB)
1 = full color management (images and UI)
2 = images only (default in most builds)
```

Firefox on Linux reads the display color profile from the compositor (via the Wayland `wp_color_management_v1` protocol or the X11 `_ICC_PROFILE` root window property) and uses it as the destination for all color transforms. [Source: ICC color correction in Firefox — MDN](https://developer.mozilla.org/en-US/docs/Mozilla/Firefox/Releases/3.5/ICC_color_correction_in_Firefox)

The command-line flag `--force-color-profile=srgb` forces Firefox to treat the display as sRGB regardless of the actual profile, useful for regression testing and screenshot comparison.

### 9.5 Chrome/Chromium

Chrome's color management is handled in the Skia layer using **skcms** — a purpose-built, minimal color management library (`//modules/skcms` in the Skia tree). skcms was developed specifically to replace LittleCMS for Chrome/Skia use cases, providing a smaller attack surface and better optimization for the common display-P3/sRGB case. [Source: skcms in Skia](https://skia.googlesource.com/skcms/)

- Skia maintains an `SkColorSpace` object derived from the display's ICC profile, constructed via `skcms_ICCProfile` parsing
- `SkColorSpace::MakeICC()` accepts a raw ICC profile blob and returns an `SkColorSpace` that Skia uses for canvas operations
- The `--force-color-profile=srgb` flag clamps to sRGB for consistent cross-platform rendering
- Chrome's `--enable-hdr` flag enables the wide-gamut path on capable displays, selecting a P3 or BT.2020 `SkColorSpace` for the canvas

---

## 10. Soft-Proofing and Print Workflows

### 10.1 The Soft-Proofing Concept

Soft-proofing simulates how a document will appear when printed on a specific paper and ink combination, on a calibrated display. This allows a photographer or designer to catch gamut problems before committing to an expensive print run.

The simulation is a two-step ICC transform:

```
Original image (source profile, e.g., Adobe RGB)
  → [Proofing transform] → CMYK printer profile (e.g., FOGRA39)
  → [Display backconvert] → Display profile (monitor ICC)
  → Screen pixels
```

lcms2 performs both transforms in a single `cmsCreateProofingTransform` call:

```c
cmsHTRANSFORM proof = cmsCreateProofingTransform(
    srcProfile,         /* image source profile (e.g., AdobeRGB) */
    TYPE_RGB_8,
    displayProfile,     /* display ICC (destination) */
    TYPE_RGB_8,
    proofProfile,       /* printer/paper profile (CMYK output) */
    INTENT_RELATIVE_COLORIMETRIC,  /* input→proof intent */
    INTENT_ABSOLUTE_COLORIMETRIC,  /* proof→display intent (paper white sim.) */
    cmsFLAGS_SOFTPROOFING);       /* enable the proofing path */
```

`cmsFLAGS_SOFTPROOFING` causes lcms2 to chain both transforms internally. `INTENT_ABSOLUTE_COLORIMETRIC` for the proof→display step preserves the paper white — the display will show a slightly off-white background to simulate unprinted paper, and the shadow detail will appear lifted to simulate ink black vs. paper black.

### 10.2 Gamut Warning (Out-of-Gamut Alarm)

Adding `cmsFLAGS_GAMUTCHECK` to the flags produces a gamut warning: pixels that fall outside the proofing profile's gamut are replaced with a warning color (typically bright green or pink). Applications implement gamut check by running the proofing transform twice — once without the flag (normal preview) and once with (to identify out-of-gamut regions) — or by using lcms2's `cmsIsIntentSupported` and `cmsCheckA2B` APIs.

```c
/* Gamut-check transform: out-of-gamut pixels become alarm color */
cmsHTRANSFORM gamut_check = cmsCreateProofingTransform(
    srcProfile, TYPE_RGB_8,
    displayProfile, TYPE_RGB_8,
    proofProfile,
    INTENT_RELATIVE_COLORIMETRIC,
    INTENT_ABSOLUTE_COLORIMETRIC,
    cmsFLAGS_SOFTPROOFING | cmsFLAGS_GAMUTCHECK);
```

### 10.3 Standard Print Profiles

The European Color Initiative and Fogra institute publish reference press profiles:

| Profile | Standard | Use Case |
|---------|----------|---------|
| `FOGRA39` | ISO 12647-2 coated | European offset, coated paper |
| `FOGRA47` | ISO 12647-2 uncoated | European offset, uncoated paper |
| `FOGRA51` | ISO 12647-2 coated v2 | European offset, coated, updated |
| `SWOP v2` | GRACoL/SWOP | North American web offset |
| `GRACoL 2006` | GRACoL/SWOP | North American commercial printing |

These CMYK profiles are freely downloadable and used as soft-proof targets in GIMP, Darktable, Scribus, and Inkscape. In Scribus (the open-source desktop publishing application), they are selected under *File > Document Settings > Color Management* as the document output profile.

### 10.4 GIMP and Darktable Soft-Proof Workflow

**GIMP:**
1. *View > Color Management > Proof Colors* — enables soft-proofing on the canvas
2. *View > Color Management > Proof Setup* — selects the printer ICC profile and rendering intent
3. *View > Color Management > Mark Out of Gamut Colors* — highlights gamut alarm areas

**Darktable:**
1. In darkroom view, enable the *gamut check* button (triangle warning icon in the bottom toolbar) — this overlays a gamut alarm visualization from the selected output profile
2. The soft-proof display is triggered from *View > Soft Proof* and uses the profile set in `$HOME/.config/darktable/preferences.xml` or the active output profile module

---

## 11. HDR Color Management

### 11.1 Beyond sRGB: the BT.2020 Container

High dynamic range content requires both a wider *chromaticity gamut* (to specify more saturated colors) and a wider *luminance range* (to specify brighter highlights). BT.2020 provides the chromaticity container: with primaries at (0.708, 0.292), (0.170, 0.797), and (0.131, 0.046), BT.2020 covers approximately 75.8% of the CIE 1931 diagram — twice the area of sRGB. Current display hardware cannot produce all of BT.2020, but it defines the outer bound for future devices and enables lossless content mastering.

HDR10, the dominant consumer HDR format, combines BT.2020 primaries with PQ (SMPTE ST 2084) transfer function and 10-bit depth for a nominal range of 0–10,000 cd/m². HDR10+ and Dolby Vision extend this with dynamic scene-level metadata.

### 11.2 ICC Profiles for HDR Displays

The ICC v4.4 specification (ICC.1:2022, amended April 2025) added support for scHDR (scene-referred HDR) profiles. These use the PQ or HLG transfer function in the TRC tags and BT.2020 primaries in the XYZ tags. The challenge: the ICC PCS (D50 XYZ) was designed for SDR luminance ranges; XYZ values for 1,000 nit highlights exceed the standard PCS range. ICC v4.4 addresses this with normalized values and a luminance scaling field in the profile header.

Practical HDR ICC profiles remain less common than their SDR counterparts. The Wayland `wp_color_management_v1` protocol (see §11.4) uses a parallel representation — specifying primaries, transfer function, and luminance range as explicit parameters rather than as an opaque ICC blob — which avoids the PCS normalization issue.

### 11.3 Vulkan HDR Color Spaces

Vulkan applications declare their swapchain color space when creating the swapchain:

```c
VkSwapchainCreateInfoKHR sci = {
    .sType = VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR,
    /* ... */
    .imageFormat = VK_FORMAT_A2B10G10R10_UNORM_PACK32,   /* 10-bit per channel */
    .imageColorSpace = VK_COLOR_SPACE_HDR10_ST2084_EXT,  /* PQ transfer function */
};
```

The `VK_EXT_swapchain_colorspace` extension defines color space enumerations beyond the base sRGB:

| Enum | Meaning |
|------|---------|
| `VK_COLOR_SPACE_SRGB_NONLINEAR_KHR` | sRGB primaries, sRGB gamma-encoded |
| `VK_COLOR_SPACE_DISPLAY_P3_NONLINEAR_EXT` | DCI-P3 with sRGB-like TF |
| `VK_COLOR_SPACE_HDR10_ST2084_EXT` | BT.2020 + PQ (HDR10) |
| `VK_COLOR_SPACE_DOLBYVISION_EXT` | Dolby Vision metadata-driven |
| `VK_COLOR_SPACE_HDR10_HLG_EXT` | BT.2020 + HLG |
| `VK_COLOR_SPACE_BT2020_LINEAR_EXT` | BT.2020 primaries, linear (no TF) |
| `VK_COLOR_SPACE_PASS_THROUGH_EXT` | No color space conversion by the driver |

When a non-sRGB color space is selected, the application is responsible for applying the correct transfer function in its shaders before presentation — the Vulkan driver does not apply OETF encoding. [Source: VK_EXT_swapchain_colorspace specification](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_swapchain_colorspace.html)

### 11.4 Wayland color-management-v1

The `wp_color_management_v1` Wayland protocol allows applications and compositors to exchange color space metadata at the protocol level. A client that renders in BT.2020/PQ attaches an image description to its `wl_surface`:

```c
/* Create an image description from explicit parameters */
struct wp_image_description_creator_params_v1 *params =
    wp_color_manager_v1_create_parametric_creator(color_manager);

wp_image_description_creator_params_v1_set_primaries_named(
    params, WP_COLOR_MANAGER_V1_PRIMARIES_BT2020);

wp_image_description_creator_params_v1_set_tf_named(
    params, WP_COLOR_MANAGER_V1_TRANSFER_FUNCTION_ST2084_PQ);

wp_image_description_creator_params_v1_set_luminances(
    params, 0, 1000, 203);  /* min_lum, max_lum, reference_white nits */

struct wp_image_description_v1 *desc =
    wp_image_description_creator_params_v1_create(params);

wp_color_management_surface_v1_set_image_description(
    cm_surface, desc, WP_COLOR_MANAGER_V1_RENDER_INTENT_PERCEPTUAL);
```

Alternatively, an ICC profile can be referenced using `wp_image_description_creator_icc_v1` when the compositor advertises `icc_v2_v4` feature support:

```c
struct wp_image_description_creator_icc_v1 *icc_creator =
    wp_color_manager_v1_create_icc_creator(color_manager);

/* Send the ICC blob via file descriptor */
wp_image_description_creator_icc_v1_set_icc_file(
    icc_creator, icc_fd, 0, icc_size);

struct wp_image_description_v1 *desc =
    wp_image_description_creator_icc_v1_create(icc_creator);
```

The protocol was added to wayland-protocols in 2024 and reached a stable, deployment-ready state by late 2025 (wayland-protocols 1.47, December 2025). KWin and Mutter both implement it. [Source: Wayland color management protocol](https://wayland.app/protocols/color-management-v1)

### 11.5 Tone Mapping: HDR to SDR

When HDR content must be displayed on an SDR screen (or when the compositor must map a wide luminance range to a limited display), a tone-mapping operator (TMO) is applied. Pure clipping (hard clip at the display maximum) destroys highlight detail. Common approaches:

**ACES Filmic:** the Academy Color Encoding System reference transform, widely used in games and film. Applies an S-curve that rolls off highlights gently and gives rich, saturated shadows. The simplified "ACES approximation" by Krzysztof Narkowicz is commonly used in game compositors.

**Hable/Uncharted 2:** developed by John Hable for Uncharted 2, this TMO provides configurable shoulder and toe controls. Also used in gamescope.

**Reinhard:** simple `L_out = L / (1 + L)` global operator. Perceptually soft but reduces contrast in SDR range.

**ST 2094:** SMPTE and HDR10+ define dynamic metadata that carries the mastering display's per-scene luminance mapping. When available, this provides scene-optimal tone mapping.

The compositor applies tone mapping in linear light (post-DEGAMMA, pre-GAMMA_LUT) so the tone mapping interacts correctly with the physical color space. See Chapter 74 §10 for compositor-level tone mapping implementations in Mutter and KWin.

---

## Roadmap

### Near-term (6–12 months)

- **`wp_color_management_v1` ecosystem consolidation:** The protocol reached stable status in wayland-protocols 1.47 (December 2025), but application and toolkit adoption is ongoing. GTK4, Qt6, and SDL3 are in various stages of implementing the protocol surface; wider deployment across the GNOME and KDE desktops is expected through 2026. [Source: Wayland Protocols 1.47 release, Phoronix](https://www.phoronix.com/news/Wayland-Protocols-1.47)
- **mpv ICC-on-Wayland stabilization:** mpv 0.41.0 landed `--icc-auto` support via `wp_color_management_v1`, but the path from colord-provided profiles to accurate compositor-side transforms is still maturing. Further integration work (profile negotiation, fallback paths) is expected in subsequent mpv releases. [Source: mpv PR #15178](https://github.com/mpv-player/mpv/pull/15178)
- **Chromium `color-management-v1` rollout:** Chromium merged initial support for `color-management-v1` on Wayland to enable HDR surface rendering. Ongoing stabilization of this path — including correct handling of ICC-based image descriptions for SDR pages — is expected to land in stable Chrome within the 2026 release cadence. [Source: Chromium commit 07c9a59](https://github.com/chromium/chromium/commit/07c9a59c2a5256ce49c22445a6c5108182c7da11)
- **LittleCMS 2.17+ maintenance:** lcms2 2.17 (February 2025) added large-file profile support (up to 4 GB) and black-point compensation on multi-channel profiles. Near-term releases are expected to focus on correctness and compatibility fixes rather than new features, as the library is in maintenance mode for the v2 series. Note: no public lcms3 roadmap has been announced; check the [upstream repository](https://github.com/mm2/Little-CMS) for developments.
- **KWin and Mutter HDR/ICC pipeline hardening:** Both compositors are actively extending their KMS color pipeline implementations. KWin's color management path, which already supports `wp_color_management_v1`, is expected to receive tone-mapping improvements and better ICC-to-KMS LUT accuracy through 2026 desktop releases. Note: needs verification against KDE and GNOME release schedules.

### Medium-term (1–3 years)

- **iccMAX (ICC v5) library and tooling maturity:** The ICC's iccMAX specification (branded iccDEV since September 2025) extends ICC v4 with scene-referred workflows, spectral data, and CAM (color appearance model) tags that go beyond D50 colorimetry. The open-source iccDEV reference library needs broader integration into the Linux ecosystem — LittleCMS and colord would each require significant extension to support v5 profiles. [Source: ICC iccMAX](https://www.color.org/iccmax/index.xalter)
- **OKLab/OKLCH integration into compositor pipelines:** CSS Color Level 4 and Level 5 standardize OKLab and OKLCH as perceptually uniform spaces with better hue linearity than CIELAB. As web content increasingly uses `oklch()` values, compositors and ICC tooling will need paths that either convert these to ICC transforms or handle them natively in the KMS pipeline. [Source: Oklab color space, Wikipedia](https://en.wikipedia.org/wiki/Oklab_color_space)
- **Vulkan `VK_EXT_swapchain_colorspace` and `VK_EXT_hdr_metadata` wider driver support:** Full wide-gamut and HDR Vulkan paths require driver-side support beyond the current Intel and RADV leaders. Broader NVIDIA (NVK) and other Mesa driver adoption of these extensions — and validation of their interaction with compositor color management — is a medium-term goal for the Linux graphics stack.
- **Standardized colord D-Bus API stabilization:** colord's D-Bus interface has evolved organically; a formal API freeze and documentation effort would improve integration by application developers and toolkit authors. Note: needs verification — no formal freeze announcement found.
- **Per-plane color management in KMS:** The `DEGAMMA_LUT`, `CTM`, and `GAMMA_LUT` properties are currently CRTC-wide. Ongoing kernel work toward per-plane color pipeline properties (tracking in the DRM color management redesign discussions) would allow compositors to apply per-surface ICC transforms in hardware rather than compositing to a single color space first. [Source: Developing Wayland Color Management and HDR, Collabora](https://www.collabora.com/news-and-blog/blog/2020/11/19/developing-wayland-color-management-and-high-dynamic-range/)

### Long-term

- **ICC v5 / iccMAX mainstream adoption:** If the iccDEV specification achieves broad industry adoption, the Linux color management stack (colord, compositors, GIMP, Krita, Darktable) would migrate from ICC v4 to v5, enabling spectral-based color management, improved appearance modeling across viewing conditions, and native support for scene-referred (linear light) HDR workflows without per-standard kludges.
- **Scene-referred color pipelines end-to-end:** The professional photography and VFX industries are moving toward scene-referred, open-domain (values > 1.0) color management using ACES or OpenColorIO. Long-term, the Linux graphics stack may expose scene-referred color spaces at the Wayland protocol level, allowing applications to deliver scene-referred content that the compositor tone-maps on the fly based on real-time display capability metadata.
- **Hardware-accelerated ICC transforms via Vulkan compute:** As displays become more varied (OLED, quantum dot, micro-LED with different gamuts and transfer curves), applying per-pixel ICC transforms in CPU-based LUTs becomes a bottleneck. A long-term architectural direction is offloading ICC profile application to Vulkan compute shaders, using the GPU's SIMD throughput to apply full 3D LUT color transforms at display refresh rate — effectively hardware-accelerating the color management pipeline that today runs in colord and the compositor CPU threads. Note: needs verification — no concrete upstream proposal identified.
- **Unified display fingerprinting and auto-calibration:** Combining EDID/DisplayID color metadata, hardware sensor feedback (ambient light, display spectroradiometer), and open calibration data formats (beyond VCGT) into an automated closed-loop calibration system is a long-term goal articulated in freedesktop color management discussions. This would allow displays to self-calibrate without user intervention.

---

## 12. Integrations

This chapter is the color science foundation for several other chapters. The key connections:

**Chapter 2 (KMS and Modesetting):** The `GAMMA_LUT`, `DEGAMMA_LUT`, and `CTM` CRTC properties described in §8 are defined in the KMS atomic API. Ch2 covers the full KMS property model and atomic commit protocol; this chapter shows how colord and compositors populate those properties with ICC-derived values.

**Chapter 22 (Production Compositors — Mutter, KWin, gamescope):** Both Mutter and KWin implement the ICC-to-KMS pipeline described in §8.5. KWin's HDR and color management implementation is more advanced, supporting the full `wp_color_management_v1` protocol. gamescope implements its own Vulkan-based color pipeline for gaming, using the same primaries and transfer function concepts covered here.

**Chapter 37 (Skia):** Skia maintains its own color management layer with `SkColorSpace` and the **skcms** library for ICC profile parsing and color space transforms. skcms was built to replace LittleCMS within the Chrome/Skia ecosystem. Skia's wide-gamut canvas path (Display-P3 `SkColorSpace`) connects to §9.5 and §11.

**Chapter 46 (Wayland Protocol Ecosystem):** The `wp_color_management_v1` and `wp_color_representation_v1` protocols connect the application color space declarations (§11.4) to the compositor's KMS backend. Ch46 covers the full protocol design and negotiation flow; this chapter covers the ICC profile concept those protocols reference.

**Chapter 52 (Firefox on Linux):** Firefox's color management (`gfx.color_management.mode`, qcms integration) described in §9.4 relies on the infrastructure of this chapter — the display ICC profile from colord, the qcms transform pipeline. Note that Firefox uses qcms (its own Rust library), not LittleCMS.

**Chapter 53 (Display Calibration and colord):** Chapter 53 is the workflow companion to this chapter. Ch53 covers the calibration measurement process, colord's profile database, VCGT loading, and GNOME/KDE GUI integration in more depth. This chapter provides the underlying color science (CIE XYZ, ICC format, rendering intents) that calibration workflow rests upon.

**Chapter 74 (HDR and Wide Color Gamut):** HDR extends ICC concepts in luminance and gamut. Ch74 covers the PQ and HLG mathematics in full, the per-plane KMS color pipeline for HDR, Mutter/KWin HDR implementations, and the Vulkan HDR swapchain in depth. §§2–3 and 11 of this chapter are the SDR/ICC foundation that Ch74 builds upon.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
