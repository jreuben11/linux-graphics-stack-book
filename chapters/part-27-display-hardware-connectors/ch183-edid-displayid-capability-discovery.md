# Chapter 183 — EDID and DisplayID: How Linux Discovers Display Capabilities

**Target audiences:** Kernel display driver developers implementing connector helpers that probe and parse monitor capability data; system integrators debugging monitor detection failures (wrong resolution, missing audio, broken HDR); application developers querying display properties at runtime through DRM properties or userspace libraries.

---

## Table of Contents

1. [EDID History and the 128-Byte Base Block](#1-edid-history-and-the-128-byte-base-block)
   - [1.1 From VESA EDID 1.0 to E-EDID 1.4](#11-from-vesa-edid-10-to-e-edid-14)
   - [1.2 Base Block Layout: Byte-by-Byte](#12-base-block-layout-byte-by-byte)
   - [1.3 Detailed Timing Descriptors](#13-detailed-timing-descriptors)
2. [CEA-861 / CTA-861 Extension Block](#2-cea-861--cta-861-extension-block)
   - [2.1 Data Block Collection Format](#21-data-block-collection-format)
   - [2.2 Video Data Block and VIC Codes](#22-video-data-block-and-vic-codes)
   - [2.3 Audio Data Block and Short Audio Descriptors](#23-audio-data-block-and-short-audio-descriptors)
   - [2.4 HDMI Vendor Specific Data Block](#24-hdmi-vendor-specific-data-block)
3. [HDR and Wide Color Gamut Signalling](#3-hdr-and-wide-color-gamut-signalling)
   - [3.1 HDR Static Metadata Block](#31-hdr-static-metadata-block)
   - [3.2 Colorimetry Data Block](#32-colorimetry-data-block)
   - [3.3 Video Capability Data Block](#33-video-capability-data-block)
4. [DisplayID 2.0](#4-displayid-20)
   - [4.1 Block Architecture](#41-block-architecture)
   - [4.2 Key Block Types](#42-key-block-types)
   - [4.3 DisplayID and EDID Coexistence](#43-displayid-and-edid-coexistence)
   - [4.4 Linux drm_displayid.c APIs](#44-linux-drm_displayidc-apis)
5. [Linux EDID Parsing in drm_edid.c](#5-linux-edid-parsing-in-drm_edidc)
   - [5.1 Entry Points and DDC Reading](#51-entry-points-and-ddc-reading)
   - [5.2 Checksum Validation and Vendor Identification](#52-checksum-validation-and-vendor-identification)
   - [5.3 CEA Extension Parsing Pipeline](#53-cea-extension-parsing-pipeline)
   - [5.4 Mode Construction and Connector Properties](#54-mode-construction-and-connector-properties)
6. [EDID Quirks Database](#6-edid-quirks-database)
   - [6.1 Quirk Flag Definitions](#61-quirk-flag-definitions)
   - [6.2 Submitting a New Quirk Upstream](#62-submitting-a-new-quirk-upstream)
7. [EDID Override and Firmware Injection](#7-edid-override-and-firmware-injection)
   - [7.1 The drm.edid_firmware Kernel Parameter](#71-the-drmedid_firmware-kernel-parameter)
   - [7.2 Sysfs Write Interface](#72-sysfs-write-interface)
   - [7.3 Use Cases](#73-use-cases)
8. [DDC/CI Monitor Control with ddcutil](#8-ddcci-monitor-control-with-ddcutil)
   - [8.1 VCP Code Space](#81-vcp-code-space)
   - [8.2 Device Access: I2C vs DP AUX](#82-device-access-i2c-vs-dp-aux)
   - [8.3 Example Session](#83-example-session)
9. [HDCP Sink Capability Discovery](#9-hdcp-sink-capability-discovery)
   - [9.1 HDCP 1.4 Key Exchange](#91-hdcp-14-key-exchange)
   - [9.2 HDCP 2.3 Capability via DPCD](#92-hdcp-23-capability-via-dpcd)
   - [9.3 The Linux DRM HDCP Framework](#93-the-linux-drm-hdcp-framework)
10. [Practical Debugging](#10-practical-debugging)
    - [10.1 Reading EDID from sysfs](#101-reading-edid-from-sysfs)
    - [10.2 Interpreting edid-decode Output](#102-interpreting-edid-decode-output)
    - [10.3 Common Failure Modes and Remedies](#103-common-failure-modes-and-remedies)
    - [10.4 Kernel Log Messages](#104-kernel-log-messages)
11. [Integrations](#11-integrations)

---

## 1. EDID History and the 128-Byte Base Block

### 1.1 From VESA EDID 1.0 to E-EDID 1.4

Before EDID existed, a PC had no way to interrogate what monitor was attached. The IBM PC connected a VGA monitor by detecting which of five resistors on the DB-15 connector were grounded — a total of 32 possible codes for a handful of fixed resolutions. The real revolution came when VESA published **EDID 1.0 in 1994**, a 128-byte binary structure that a display stores in a small EEPROM and exposes over a two-wire I²C bus (DDC2B, 100 kHz) embedded in the VGA, DVI, or HDMI cable. The host GPU reads that EEPROM at address `0x50` to discover timings, size, and the manufacturer identity, then selects an appropriate mode before the user touches a settings menu.

Version history:

| Version | Year | Key additions |
|---------|------|---------------|
| EDID 1.0 | 1994 | 128-byte base block, 8 DTDs, established timings |
| EDID 1.1 | 1996 | Clarifications, no structural changes |
| EDID 1.2 | 1997 | Color management data, manufactured week |
| EDID 1.3 | 1997 | GTF support flag, manufacturer logo descriptor |
| E-EDID 1.4 | 2006 | Digital interface type byte, bit-depth field, CVT support, 4th DTD improved |

[Source: VESA E-EDID Standard Release A2 (2006)](https://glenwing.github.io/docs/VESA-EEDID-A2.pdf)

Extension blocks (each 128 bytes, count stored at offset 126) allow unlimited capability signalling. The CEA-861 extension (tag `0x02`) carries HDMI audio and video data blocks; the DisplayID extension (tag `0x70`) replaces most of the base structure for modern displays; and several other tags exist for VTB, VESA DI-EXT, and LS-EXT blocks.

### 1.2 Base Block Layout: Byte-by-Byte

The 128-byte base block is precisely specified down to the bit. The layout below follows E-EDID 1.4:

```
Offset  Length  Field
------  ------  -----
0       8       Header: 0x00 0xFF 0xFF 0xFF 0xFF 0xFF 0xFF 0x00
8       2       Manufacturer ID (PNP code, 5 bits per letter, big-endian)
10      2       Product code (little-endian, manufacturer assigned)
12      4       Serial number (little-endian, 0x00000000 if unused)
16      1       Week of manufacture (1–53, or 0xFF = model year)
17      1       Year of manufacture (year − 1990)
18      1       EDID version (0x01)
19      1       EDID revision (0x04 for E-EDID 1.4)
20      1       Video input parameters
21      1       Horizontal screen size in cm (or aspect ratio)
22      1       Vertical screen size in cm (or aspect ratio)
23      1       Display gamma: (gamma × 100) − 100  [e.g., 0x78 = 2.20]
24      1       Feature support bitmap
25      10      Chromaticity coordinates (CIE 1931 xy for R, G, B, W)
35      3       Established timings I, II, III
38      16      Standard timings (8 × 2 bytes)
54      72      Detailed Timing Descriptors (4 × 18 bytes)
126     1       Extension block count
127     1       Checksum (sum of all 128 bytes must be 0x00 mod 256)
```

**Manufacturer ID** (bytes 8–9): Three letters of the manufacturer's EISA/PNP code, packed with 5 bits per letter (A=1, B=2, … Z=26), stored big-endian in a 16-bit field with bit 15 reserved as zero. For example, "DEL" (Dell) encodes as `(4 << 10) | (5 << 5) | 12` = `0x10AC`. The full PNP database is maintained by the IANA and used by the kernel's `edid_vendor()` function to identify the make of a monitor.

**Video input parameters byte** (offset 20): Bit 7 distinguishes digital (`1`) from analog (`0`) interfaces. For digital:
- Bits 6:4 — color bit depth: `001`=6 bpc, `010`=8 bpc, `011`=10 bpc, `100`=12 bpc, `101`=14 bpc, `110`=16 bpc, `000`=undefined
- Bits 3:0 — video interface type: `0010`=HDMIa, `0011`=HDMIb, `0100`=MDDI, `0101`=DisplayPort

**Screen size** (bytes 21–22): When non-zero, centimeters of the physical active area (e.g., 53 cm × 30 cm for a 24-inch 16:9 panel). A zero in either field indicates the aspect ratio is given in the feature byte instead.

**Display gamma** (byte 23): Encoded as `(gamma − 1) × 100`, stored as an 8-bit integer. A value of `0x78` = 120 decimal → gamma = 2.20 (sRGB default). A value of `0xFF` means gamma is defined in a Display Transfer Characteristics descriptor block.

**Feature support bitmap** (byte 24):
- Bit 7: standby support (DPMS)
- Bit 6: suspend support (DPMS)
- Bit 5: active-off / low-power (DPMS)
- Bits 4:3 — display color type for analog, or color encoding formats for digital (00=RGB 4:4:4 only, 01=RGB + YCbCr 4:4:4, 10=RGB + YCbCr 4:2:2, 11=all three)
- Bit 2: sRGB is the default color space (chromaticity matches sRGB exactly)
- Bit 1: preferred timing is in DTD slot 1
- Bit 0: continuous timings supported (GTF/CVT, not a fixed list)

**Chromaticity coordinates** (bytes 25–34): Ten bytes encoding the CIE 1931 xy chromaticity for Red, Green, Blue, and White primaries. Each coordinate is a 10-bit fixed-point value in range [0, 1]. The two least-significant bits of each (Rx[1:0], Ry[1:0], Gx[1:0], …) are packed together in bytes 25–26; the eight most-significant bits are in bytes 27–34 in order RxH, RyH, GxH, GyH, BxH, ByH, WxH, WyH. For example, sRGB white D65 encodes as approximately xy = (0.3127, 0.3290).

**Established timings** (bytes 35–37): A 24-bit bitmap where each bit represents a historically common resolution. Examples: bit 0 of byte 35 = 720×400@70 Hz, bit 5 = 640×480@60 Hz (mandatory for all VGA-compatible displays), bit 0 of byte 36 = 800×600@60 Hz, bit 3 = 1024×768@60 Hz, bit 7 = 1280×1024@75 Hz.

**Standard timings** (bytes 38–53): Eight 2-byte entries. Byte 0: `(horizontal active pixels / 8) − 31`, giving horizontal resolution in steps of 8 pixels. Byte 1, bits 7:6: aspect ratio (00=16:10, 01=4:3, 10=5:4, 11=16:9). Byte 1, bits 5:0: vertical refresh rate − 60 Hz. A value of `0x0101` (`01 01`) marks an unused slot.

### 1.3 Detailed Timing Descriptors

Bytes 54–125 hold four 18-byte blocks. The first slot **must** describe the preferred mode (when bit 1 of byte 24 is set). Each DTD encodes a complete VESA-format timing:

```c
/* struct detailed_pixel_timing in include/drm/drm_edid.h */
struct detailed_pixel_timing {
    u8 hactive_lo;           /* lower 8 bits of horizontal active */
    u8 hblank_lo;            /* lower 8 bits of horizontal blanking */
    u8 hactive_hblank_hi;    /* upper 4 bits each of hactive, hblank */
    u8 vactive_lo;           /* lower 8 bits of vertical active */
    u8 vblank_lo;            /* lower 8 bits of vertical blanking */
    u8 vactive_vblank_hi;    /* upper 4 bits each of vactive, vblank */
    u8 hsync_offset_lo;      /* lower 8 bits of horizontal sync offset */
    u8 hsync_pulse_width_lo; /* lower 8 bits of hsync pulse width */
    u8 vsync_offset_pulse_width_lo; /* upper nibble = vsync offset lo, lower nibble = vpulse lo */
    u8 hsync_vsync_offset_pulse_width_hi; /* bits 7:6 ho_hi, 5:4 hpw_hi, 3:2 vo_hi, 1:0 vpw_hi */
    u8 width_mm_lo;          /* lower 8 bits of horizontal size in mm */
    u8 height_mm_lo;         /* lower 8 bits of vertical size in mm */
    u8 width_height_mm_hi;   /* bits 7:4 width hi, 3:0 height hi */
    u8 hborder;              /* horizontal border pixels */
    u8 vborder;              /* vertical border lines */
    u8 misc;                 /* sync type, interlaced flag */
} __packed;
```

[Source: `include/drm/drm_edid.h`, Linux kernel](https://github.com/torvalds/linux/blob/master/include/drm/drm_edid.h)

The pixel clock is stored as a 16-bit little-endian value in bytes 0–1 of the DTD in units of 10 kHz (minimum 10 kHz). A value of `0x0000` in bytes 0–1 signals a non-timing descriptor (monitor name, serial string, or range limits).

**Monitor Range Limits Descriptor** (tag `0xFD`): Encodes minimum/maximum horizontal and vertical frequencies (Hz), and maximum pixel clock (MHz/10). EDID 1.4 extended this with CVT timing support: the descriptor can optionally include preferred CVT vertical/horizontal refresh rate and the supported aspect ratios.

---

## 2. CEA-861 / CTA-861 Extension Block

### 2.1 Data Block Collection Format

The CEA-861 extension (tag `0x02` at byte 0) carries audio, video, and AV metadata beyond what the base EDID can express. All HDMI sinks are required to include at least one CEA extension. The Consumer Technology Association rebranded the standard CTA-861 starting with revision G (2016); the two names are used interchangeably in kernel code.

Extension block layout:

```
Byte 0:  0x02 (extension tag)
Byte 1:  revision (03 = CEA-861-B, 03 also for all later revisions with
                    data block collection)
Byte 2:  d (byte offset to start of 18-byte DTDs from start of extension)
Byte 3:  Feature flags: underscan, audio, YCbCr 4:4:4, YCbCr 4:2:2
Bytes 4..(d-1): Data Block Collection (DBC)
Bytes d..125:   Detailed Timing Descriptors (same 18-byte format)
Byte 126: padding (0x00)
Byte 127: checksum
```

The Data Block Collection is a packed sequence of variable-length data blocks. Each block begins with a one-byte tag+length header:
- Bits 7:5 — block tag code
- Bits 4:0 — payload length in bytes (not counting the header byte itself)

An Extended Tag Code (tag = `0x07`) uses a second byte for the extended tag, giving the full namespace for HDR, colorimetry, video capability, HDMI 2.x, and more.

### 2.2 Video Data Block and VIC Codes

Tag code `0x02` identifies a Video Data Block (VDB). Its payload is a sequence of 1-byte **Short Video Descriptors (SVDs)**:

- Bit 7: **native indicator** — this VIC is the sink's preferred format
- Bits 6:0: **Video Identification Code (VIC)**

VIC codes are defined in CTA-861-H (Table 3). Selected examples:

| VIC | Format | Frame rate | Aspect |
|-----|--------|------------|--------|
| 1   | 640×480p | 59.94/60 Hz | 4:3 |
| 2   | 720×480p | 59.94/60 Hz | 4:3 |
| 4   | 1280×720p | 59.94/60 Hz | 16:9 |
| 16  | 1920×1080p | 59.94/60 Hz | 16:9 |
| 18  | 720×576p | 50 Hz | 16:9 |
| 31  | 1920×1080p | 50 Hz | 16:9 |
| 32  | 1920×1080p | 23.97/24 Hz | 16:9 |
| 95  | 3840×2160p | 29.97/30 Hz | 16:9 |
| 96  | 3840×2160p | 25 Hz | 16:9 |
| 97  | 3840×2160p | 59.94/60 Hz | 16:9 |
| 107 | 3840×2160p | 119.88/120 Hz | 16:9 |

VIC 0 is reserved; VICs 128–192 are extended (7-bit field is not sufficient, so a separate Extended Video Data Block using extended tag `0x00` can carry VIC codes up to 255).

[Source: CTA-861-H, published by Consumer Technology Association; archived partial text at archive.org/stream/CTA-861-G/](https://archive.org/stream/CTA-861-G/CTA-861-G_djvu.txt)

### 2.3 Audio Data Block and Short Audio Descriptors

Tag code `0x01` identifies an Audio Data Block (ADB). Its payload is a series of 3-byte **Short Audio Descriptors (SADs)**:

```
Byte 0, bits 6:3: Audio format code
        bits 2:0: Maximum channel count minus 1 (so 0x01 = 2 channels)
Byte 1, bits 6:0: Supported sample rate bitmap
        bit 6 = 192 kHz, bit 5 = 176.4 kHz, bit 4 = 96 kHz,
        bit 3 = 88.2 kHz, bit 2 = 48 kHz, bit 1 = 44.1 kHz, bit 0 = 32 kHz
Byte 2: Format-dependent:
  LPCM (code 1): bits 2:0 = bit depth bitmap (bit0=16, bit1=20, bit2=24-bit)
  Compressed (codes 2–8): maximum bitrate in units of 8 kbps
  HBR (codes 9–13): vendor-defined or zero
```

Audio format codes (AFC):

| Code | Format |
|------|--------|
| 1    | L-PCM (IEC 60958) |
| 2    | AC-3 (Dolby Digital) |
| 3    | MPEG-1 |
| 4    | MP3 (MPEG-1 Layer 3) |
| 5    | MPEG-2 |
| 6    | AAC LC |
| 7    | DTS |
| 8    | ATRAC |
| 9    | DSD (One-Bit Audio) |
| 10   | E-AC-3 / Dolby Digital Plus |
| 11   | DTS-HD |
| 12   | MAT (Dolby TrueHD) |
| 13   | DST |
| 14   | WMA Pro |
| 15   | Extension (uses extended code in byte 2) |

A **Speaker Allocation Data Block** (tag `0x04`, always 3 bytes of payload) signals which speaker channels the sink can drive: FL/FR, LFE, FC, RLC/RRC, FLC/FRC, RC, FLH/FRH, FLW/FRW, and TC bits.

### 2.4 HDMI Vendor Specific Data Block

Tag `0x03` identifies a Vendor Specific Data Block (VSDB). The first 3 bytes of payload carry an IEEE OUI identifying the vendor:

- `0x000C03` — HDMI Licensing, LLC (standard HDMI VSDB)
- `0xC45DD8` — HDMI Forum VSDB (HDMI 2.0+)
- `0x00001A` — AMD FreeSync VSDB

The HDMI VSDB (OUI `0x000C03`) layout:

```
Bytes 0-2:  OUI 0x000C03 (little-endian: 03 0C 00)
Bytes 3-4:  Source Physical Address (SPA) for CEC topology
            [A.B.C.D] where A=byte3 bits 7:4, B=byte3 bits 3:0,
            C=byte4 bits 7:4, D=byte4 bits 3:0
Byte 5:     Capability flags:
            bit 7: supports_ai (ACP/ISRC1/ISRC2 packets)
            bit 6: DC_48bit (48-bit deep color)
            bit 5: dc_36bit (36-bit deep color)
            bit 4: dc_30bit (30-bit deep color)
            bit 3: dc_y444 (YCbCr 4:4:4 in deep color modes)
            bit 2: dvi_dual (dual-link DVI)
Byte 6:     Max TMDS clock in units of 5 MHz (max 340 MHz = 0x44)
Byte 7:     Latency field flags
Byte 8-9:   Progressive/interlace video and audio latency
Bytes 10+:  HDMI Video Sub-block (3D formats, extended colorimetry, etc.)
```

[Source: HDMI Specification 1.4b, Section 8.3.2; relevant structures in Linux at `include/linux/hdmi.h`](https://github.com/torvalds/linux/blob/master/include/linux/hdmi.h)

---

## 3. HDR and Wide Color Gamut Signalling

### 3.1 HDR Static Metadata Block

The HDR Static Metadata Block is an extended data block: tag byte = `0x07`, extended tag byte = `0x06`. It was standardised in CTA-861.3 (2015) and incorporated into CTA-861-G. Layout:

```
Byte 0: 0x07 (block tag = use extended tag)
Byte 1: length (= 5 typically, can be up to 6 with MaxCLL/MaxFALL)
Byte 2: extended tag = 0x06
Byte 3: Electro-optical Transfer Function (EOTF) bitmap
        bit 0: SDR Gamma (CEA / BT.1886)
        bit 1: HDR Gamma (HLG, ARIB STD-B67)
        bit 2: SMPTE ST 2084 (PQ, used by HDR10 and Dolby Vision)
        bit 3: Hybrid Log-Gamma (HLG, BBC/NHK)
Byte 4: Static Metadata Descriptor Type bitmap
        bit 0: Type 1 (SMPTE ST 2086 mastering display metadata)
Byte 5: Desired Content Max Luminance
        encoded as: (50 × 2^(val/32)) nits — byte value 0 means not indicated
Byte 6: Desired Content Max Frame-Average Light Level (MaxFALL)
        same encoding
Byte 7 (optional): Desired Content Min Luminance
        encoded as: Desired_Content_Max_Luminance × (val/255)^2 / 100
```

MaxCLL and MaxFALL values above 0 indicate the content metadata range the display is designed for. The kernel populates `drm_connector_state.hdr_output_metadata` from these fields when setting up HDR output. Note: the luminance encoding in bytes 5–7 differs from the SMPTE ST 2086 MaxCLL InfoFrame encoding — the EDID bytes use a compressed scale tied to the display's maximum luminance rather than an absolute nit value.

[Source: CTA-861.3-2015, *HDR Static Metadata Extensions*; archived overview at standards.globalspec.com](https://standards.globalspec.com/std/10037169/CTA-861.3)

### 3.2 Colorimetry Data Block

Extended tag `0x05`: the Colorimetry Data Block signals wide-gamut color space support:

```
Byte 3 (colorimetry bitmask):
  bit 0: xvYCC601 (IEC 61966-2-4 based on BT.601)
  bit 1: xvYCC709 (IEC 61966-2-4 based on BT.709)
  bit 2: sYCC601
  bit 3: opYCC601
  bit 4: opRGB (IEC 61966-2-5)
  bit 5: BT2020cYCC (constant luminance)
  bit 6: BT2020YCC
  bit 7: BT2020RGB (UHDTV wide-gamut primaries)
Byte 4 (gamut metadata):
  bit 2: DCI-P3 (extended via CTA-861-G)
  bit 3: HDR10+
```

When bit 7 (`BT2020RGB`) is set and the HDR Static Metadata Block sets the PQ EOTF bit, the display is declaring HDR10 capability — the combination that Linux's DRM HDR layer keys on in `drm_connector_attach_hdr_output_metadata_property()`.

### 3.3 Video Capability Data Block

Extended tag `0x00`: the VCDB qualifies how the sink applies scan behaviour and quantization:

```
Byte 3:
  bits 7:6: S_PT — preferred quantization format for non-CE and non-IT video
  bits 5:4: S_IT — scan behavior for IT video
  bits 3:2: S_CE — scan behavior for CE video (00=always overscan, 01=always underscan,
             10=CE, 11=selectable)
  bit 1:    QS — quantization range selectable (YCC)
  bit 0:    QY — YCC quantization range selectable
```

The QS and QY bits are important for applications sending full-range RGB to an HDMI display: if neither is set, the display will silently apply BT.601/709 limited-range clipping.

---

## 4. DisplayID 2.0

### 4.1 Block Architecture

DisplayID was designed from the outset as EDID's successor. Rather than a fixed 128-byte structure, DisplayID uses a variable-length sequence of typed data blocks. A DisplayID payload starts with a mandatory header:

```
Byte 0: Version (0x20 for DisplayID 2.0)
Byte 1: Revision (0x00 for 2.0r0)
Byte 2: Total payload length in bytes (excluding header and checksum)
Byte 3: DisplayID usage (primary = 0x00; embedded = 0x01)
Byte 4: Reserved
Bytes 5..N: Data blocks
Byte N+1: Checksum
```

Each data block begins with a 3-byte block header:
- Byte 0: block type code
- Byte 1: revision and data block flags
- Byte 2: payload length (bytes following the 3-byte header)

[Source: VESA DisplayID Standard Version 2.0; Wikipedia summary at en.wikipedia.org/wiki/DisplayID](https://en.wikipedia.org/wiki/DisplayID)

### 4.2 Key Block Types

| Type code | Block name | Purpose |
|-----------|------------|---------|
| `0x20`    | Product Identification | Manufacturer, product code, serial, product name string |
| `0x21`    | Display Parameters | Physical size (mm), pixel count, orientation, rotation, sub-pixel layout, color depth, HDR/WCG flags |
| `0x22`    | Type VII Detailed Timing (v2) | 20-byte per-mode timing records; replaces EDID DTD |
| `0x23`    | Type VIII Enumerated Timing | Pre-defined timing codes from a VESA table |
| `0x24`    | Type IX Formula-based Timing | Encode GTF/CVT formula parameters |
| `0x25`    | Dynamic Video Timing Range | Min/max frequency ranges for adaptive sync (DisplayID 2.0 only) |
| `0x26`    | Display Interface Features | Bit-depth support, YCbCr encoding, DSC version, audio |
| `0x27`    | Stereo Display Interface | Side-by-side, top-bottom, frame sequential stereoscopy |
| `0x28`    | Tiled Display Topology | Video wall tile coordinates, total tile count, bezel dimensions |
| `0x29`    | Container ID | UUID linking tiles of the same logical display |
| `0x7E`    | Vendor-specific | Arbitrary vendor payload with IEEE OUI |
| `0x81`    | CTA Data Block | Embeds CTA-861 data blocks for audio, HDR, and AV metadata |

The **Display Parameters Block** (type `0x21`) is particularly important: it carries horizontal and vertical pixel count, horizontal and vertical physical size in mm, orientation bits, sub-pixel shape and arrangement (RGB/BGR/RGBW/pentile/delta triangle), supported color bit depths, and capability flags for HDR, wide color gamut, and audio.

The **Tiled Display Topology Block** (type `0x28`) enables multi-panel video walls. It encodes:
- Total tiles horizontally and vertically
- This tile's column and row position (zero-based)
- Physical horizontal and vertical bezel widths in pixels
- The native resolution of the tiled region

A 4K display built from four 1080p panels in a 2×2 grid uses four EDID streams (one per physical connector), each containing a `0x28` block with positions (0,0), (1,0), (0,1), (1,1), and the container ID (`0x29`) linking them.

### 4.3 DisplayID and EDID Coexistence

There are three deployment scenarios:

1. **EDID extension block**: The display presents a standard 128-byte EDID base block, followed by one or more 128-byte extension blocks with tag `0x70` (DisplayID). The DisplayID payload is split across these 128-byte chunks; extension block byte 1 holds a continuation counter. This is the most common form for HDMI 2.1 and DP 2.x displays.

2. **DisplayPort DPCD**: A DP sink may store a DisplayID payload in DPCD registers starting at offset `0x0500` (the "Sink Device Identification" area). Bit 0 of DPCD `0x00E` (`TRAINING_AUX_RD_INTERVAL`) flags whether this area holds a valid DisplayID structure. This allows DisplayPort MST branch devices and docking stations to present per-output DisplayID without true DDC access.

3. **HDMI 2.1 Forum VSDB + DisplayID**: HDMI 2.1 sinks that implement DSC, FRL (Fixed-Rate Link), or 4K@120 Hz are required to include a DisplayID extension.

When a DP or HDMI driver reads the EDID blob and encounters a `0x70` extension, the DRM core's `drm_edid.c` calls into the DisplayID parser to extract additional modes and capabilities that supplement (or in some cases override) those from the EDID base block and CEA extension.

### 4.4 Linux drm_displayid.c APIs

The DisplayID parsing code lives in `drivers/gpu/drm/drm_edid.c` (for integration with EDID parsing) with shared declarations. Key public APIs (from `include/drm/display/drm_dp_helper.h` and internal drm_edid.c):

```c
/* Iterator API for walking DisplayID blocks embedded in EDID */
void displayid_iter_edid_begin(const struct drm_edid *drm_edid,
                               struct displayid_iter *iter);
const struct displayid_block *displayid_iter_next(struct displayid_iter *iter);
void displayid_iter_end(struct displayid_iter *iter);

/* Unpack a DisplayID payload from an EDID extension block */
/* (internal to drm_edid.c, called during drm_edid parsing) */
static void drm_edid_displayid_unpack(struct drm_edid *drm_edid);
```

[Source: `drivers/gpu/drm/drm_edid.c`, Linux kernel; patch discussion at lkml.iu.edu](https://lkml.iu.edu/hypermail/linux/kernel/2603.3/12490.html)

The iterator pattern allows driver code to walk all DisplayID blocks without knowing the internal chunking across EDID extension blocks. A driver that wants to check for Tiled Display Topology:

```c
/* Example usage — drivers/gpu/drm/display/my_driver.c */
struct displayid_iter iter;
const struct displayid_block *block;

displayid_iter_edid_begin(drm_edid, &iter);
while ((block = displayid_iter_next(&iter))) {
    if (block->tag == DATA_BLOCK_2_TILED_DISPLAY)
        parse_tile_block(connector, block);
}
displayid_iter_end(&iter);
```

---

## 5. Linux EDID Parsing in drm_edid.c

### 5.1 Entry Points and DDC Reading

The primary EDID reading functions in `drivers/gpu/drm/drm_edid.c` form a layered API. The modern opaque `struct drm_edid` wrapper replaced the legacy raw `struct edid *` approach starting around Linux 5.19 to enable safer memory management and atomic update.

```c
/* High-level entry point — probes DDC, validates, returns drm_edid */
const struct drm_edid *drm_edid_read(struct drm_connector *connector);

/* Read EDID from a specific I2C adapter (HDMI/DVI DDC) */
const struct drm_edid *drm_edid_read_ddc(struct drm_connector *connector,
                                          struct i2c_adapter *adapter);

/* Read EDID from a DisplayPort AUX channel */
const struct drm_edid *drm_edid_read_dp(struct drm_connector *connector,
                                         struct drm_dp_aux *aux);

/* Free the drm_edid object */
void drm_edid_free(const struct drm_edid *drm_edid);
```

[Source: `include/drm/drm_edid.h`, Linux kernel; EDID documentation at docs.kernel.org](https://docs.kernel.org/admin-guide/edid.html)

`drm_edid_read_ddc()` issues the I²C read at slave address `0x50` using `drm_do_get_edid()`, which retries up to 4 times per block (DDC2B does not guarantee first-read success on hot-plug). Each 128-byte block is checksummed independently. If the first block is valid but extension blocks fail checksum, the kernel logs a warning and truncates to the base block.

For DisplayPort, `drm_edid_read_dp()` reads the EDID using AUX channel transactions (I2C-over-AUX), accessing address `0x50` for the base block and `0x50`/`0x51` for segment-addressed extension blocks (EDID segment protocol). It then also checks DPCD `0x000E` for a DisplayID presence flag.

### 5.2 Checksum Validation and Vendor Identification

```c
/* drivers/gpu/drm/drm_edid.c (simplified) */
static bool edid_block_valid(const void *_block, bool base)
{
    const struct edid *block = _block;
    u8 sum = 0;

    /* Verify the 8-byte EDID header for the base block */
    if (base && memcmp(block->header, edid_header, sizeof(edid_header)))
        return false;

    /* Fletcher-0 checksum: sum of all 128 bytes must be 0x00 mod 256 */
    for (int i = 0; i < EDID_LENGTH; i++)
        sum += ((const u8 *)block)[i];
    return sum == 0;
}
```

[Source: `drivers/gpu/drm/drm_edid.c`, Linux kernel GitHub](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_edid.c)

Manufacturer identification is extracted by `edid_manufacturer_id()`, which decodes the 5-bit-per-letter PNP packing from bytes 8–9. The resulting 3-letter string (e.g., "SAM" for Samsung, "LGD" for LG Display) is stored in `drm_display_info.name` and used as the key for quirk lookup.

### 5.3 CEA Extension Parsing Pipeline

When `drm_edid_read*()` returns a valid blob, calling `drm_edid_connector_update()` triggers the full parsing pipeline:

```c
int drm_edid_connector_update(struct drm_connector *connector,
                               const struct drm_edid *drm_edid);
```

Internally this calls:

1. `drm_edid_to_eld()` — generates the Audio ELD (EDID-Like Data) structure for HDA audio drivers, populated from the CEA ADB (see Ch140).
2. `drm_parse_cea_ext()` — iterates the Data Block Collection in each CEA extension block:
   - Dispatches to `drm_parse_cea_ext_db_audio()` for ADB (tag 1)
   - Dispatches to `drm_parse_cea_ext_db_video()` for VDB (tag 2), calling `drm_add_cmdb_modes()` for each SVD
   - Dispatches to `drm_parse_cea_ext_db_vendor_specific()` for VSDB (tag 3): if OUI is `0x000C03`, extracts deep color flags, SPA, and HDMI version
   - Dispatches to `drm_parse_cea_ext_db_hdr_static_metadata()` for extended tag 6: populates `drm_connector_state.hdr_output_metadata`
   - Dispatches to `drm_parse_cea_ext_db_colorimetry()` for extended tag 5

The parsed capabilities accumulate in `drm_display_info`:

```c
/* include/drm/drm_connector.h */
struct drm_display_info {
    unsigned int width_mm, height_mm;
    unsigned int bpc;               /* bits per component from EDID */
    u8 cea_rev;                     /* CEA-861 revision */
    bool is_hdmi;                   /* true if HDMI VSDB found */
    bool has_audio;                 /* true if ADB found */
    bool has_hdmi_infoframe;
    u32 color_formats;              /* DRM_COLOR_FORMAT_* bitmask */
    u32 hdmi_colorimetry;           /* Colorimetry DB flags */
    struct drm_hdmi_info hdmi;      /* HDMI 2.0+ specific info */
    struct hdr_sink_metadata hdr_sink_metadata; /* HDR SMB data */
    /* ... */
};
```

### 5.4 Mode Construction and Connector Properties

```c
/* Convert parsed EDID data to drm_display_mode list entries */
int drm_edid_connector_add_modes(struct drm_connector *connector);

/* Legacy API (still widely used): add modes and update property */
int drm_add_edid_modes(struct drm_connector *connector,
                       struct edid *edid);

/* Store raw EDID blob in the "EDID" DRM connector property */
int drm_connector_update_edid_property(struct drm_connector *connector,
                                        const struct edid *edid);
```

Mode construction sources, in priority order:
1. **DTDs in EDID base block** — parsed by `do_detailed_mode()`, flag `DRM_MODE_TYPE_PREFERRED` is set on the first DTD when the feature-support bit is set.
2. **SVDs in CEA VDB** — parsed by `do_cea_modes()`, added as `DRM_MODE_TYPE_DRIVER`.
3. **Standard timings** — 8 entries decoded by `drm_mode_std()` using GTF or CVT formulas.
4. **Established timings** — bitmap decoded from `edid_est_modes[]` table (16 classic modes).
5. **DMT modes** — if GTF/CVT tables don't produce a match for a standard timing, the driver falls back to `drm_dmt_modes[]` (87 entries from the VESA DMT standard).

The resulting `drm_display_mode` list is attached to `connector->probed_modes` and presented to userspace through the `DRM_IOCTL_MODE_GETCONNECTOR` ioctl. The raw EDID binary blob is exposed via the "EDID" connector property, readable at `/sys/class/drm/card0-HDMI-A-1/edid`.

---

## 6. EDID Quirks Database

### 6.1 Quirk Flag Definitions

Hundreds of monitors ship with subtly broken EDID data. Rather than refusing to drive them, the kernel maintains a quirk database keyed by manufacturer code and product ID. Quirk flags are defined as an enum in `drivers/gpu/drm/drm_edid.c`:

```c
/* drivers/gpu/drm/drm_edid.c */
enum drm_edid_internal_quirk {
    EDID_QUIRK_PREFER_LARGE_60 = DRM_EDID_QUIRK_NUM,
    EDID_QUIRK_135_CLOCK_TOO_HIGH,
    EDID_QUIRK_PREFER_LARGE_75,
    EDID_QUIRK_DETAILED_IN_CM,        /* Physical dimensions in cm, not mm */
    EDID_QUIRK_DETAILED_USE_MAXIMUM_SIZE,
    EDID_QUIRK_DETAILED_SYNC_PP,      /* Positive polarity on sync */
    EDID_QUIRK_FORCE_REDUCED_BLANKING,
    EDID_QUIRK_FORCE_8BPC,            /* Ignore claimed >8 bpc support */
    EDID_QUIRK_FORCE_12BPC,
    EDID_QUIRK_FORCE_6BPC,
    EDID_QUIRK_FORCE_10BPC,
    EDID_QUIRK_NON_DESKTOP,           /* VR headsets, projectors */
    EDID_QUIRK_CAP_DSC_15BPP,         /* DSC capability at 15 bpp */
};
```

Entries in `edid_quirk_list[]` use the macro:

```c
#define EDID_QUIRK(vend_chr_0, vend_chr_1, vend_chr_2, product_id, _quirks) \
{ \
    .ident = { \
        .panel_id = drm_edid_encode_panel_id(vend_chr_0, vend_chr_1, \
                                              vend_chr_2, product_id), \
    }, \
    .quirks = _quirks \
}
```

Representative entries (illustrative, from the kernel source):

```c
/* drivers/gpu/drm/drm_edid.c — edid_quirk_list[] (sample entries) */

/* Acer AL1706: first DTD is wrong; use largest 60 Hz mode instead */
EDID_QUIRK('A', 'C', 'R', 44358, BIT(EDID_QUIRK_PREFER_LARGE_60)),

/* Envision EN-7100e: reports wrong dimensions in millimeters */
EDID_QUIRK('E', 'N', 'V', 0x0602, BIT(EDID_QUIRK_DETAILED_IN_CM)),

/* Valve Index HMD: mark as non-desktop so compositors skip it */
EDID_QUIRK('V', 'L', 'V', 0x91a8, BIT(EDID_QUIRK_NON_DESKTOP)),

/* HTC Vive: same */
EDID_QUIRK('H', 'V', 'R', 0xaa01, BIT(EDID_QUIRK_NON_DESKTOP)),

/* BenQ GW2765: claims 10 bpc but hardware can't do it cleanly */
EDID_QUIRK('B', 'N', 'Q', 0x78d6, BIT(EDID_QUIRK_FORCE_8BPC)),
```

[Source: `drivers/gpu/drm/drm_edid.c` in torvalds/linux on GitHub; constification patch discussion at patchwork.kernel.org](https://patchwork.kernel.org/patch/9490169/)

The `EDID_QUIRK_NON_DESKTOP` flag is especially important for VR headsets: compositors query `drm_connector.display_info.non_desktop` (set from this quirk) and skip those connectors when building the desktop output list, preventing the headset from appearing as a mirror or extended desktop target.

### 6.2 Submitting a New Quirk Upstream

If you encounter a monitor with broken EDID that the kernel mishandles, the process is:

1. Read the raw EDID: `cat /sys/class/drm/card0-HDMI-A-1/edid | xxd > monitor.hex`
2. Decode it: `edid-decode /sys/class/drm/card0-HDMI-A-1/edid` (from the `edid-decode` package, or `v4l-utils` which bundles it)
3. Identify the manufacturer PNP code (3 letters) and product code (4 hex digits) from the decode output
4. Identify the correct quirk flag: `EDID_QUIRK_PREFER_LARGE_60` if DTD slot 1 is wrong; `EDID_QUIRK_FORCE_8BPC` if bit-depth reporting is incorrect; `EDID_QUIRK_DETAILED_IN_CM` if physical dimensions are off by a factor of 10
5. Send a patch to `dri-devel@lists.freedesktop.org` with subject `[PATCH] drm/edid: Add quirk for <vendor> <model>`, including the `edid-decode` output as a code comment above the new entry and attaching the raw binary EDID

[Source: freedesktop.org mailing list `dri-devel@lists.freedesktop.org`; Patchwork at patchwork.freedesktop.org](https://patchwork.freedesktop.org/)

---

## 7. EDID Override and Firmware Injection

### 7.1 The drm.edid_firmware Kernel Parameter

When a monitor returns corrupt or missing EDID data — common with cheap KVM switches, HDMI capture cards, or headless servers — the kernel can be told to substitute a firmware-supplied EDID binary:

```bash
# Kernel command line (GRUB, systemd-boot, etc.)
drm.edid_firmware=HDMI-A-1:edid/1920x1080.bin

# Multiple outputs:
drm.edid_firmware=DP-1:edid/4k60.bin,DP-3:edid/1080p60.bin
```

The firmware path (e.g., `edid/1920x1080.bin`) is relative to `/lib/firmware/`. The kernel ships a set of pre-built EDID binaries compiled in when `CONFIG_DRM_LOAD_EDID_FIRMWARE=y`:

```
/lib/firmware/edid/1024x768.bin
/lib/firmware/edid/1280x1024.bin
/lib/firmware/edid/1600x1200.bin
/lib/firmware/edid/1680x1050.bin
/lib/firmware/edid/1920x1080.bin
/lib/firmware/edid/2560x1440.bin
/lib/firmware/edid/4096x2160.bin
```

[Source: Linux kernel docs, *EDID*, at docs.kernel.org/admin-guide/edid.html](https://docs.kernel.org/admin-guide/edid.html)

To use a custom monitor's EDID on a different output (for example, copying a working EDID to a broken connector):

```bash
# 1. Identify the working output
for p in /sys/class/drm/*/status; do
    con=${p%/status}
    echo -n "${con#*/card?-}: "; cat $p
done

# 2. Copy the EDID binary
sudo mkdir -p /usr/lib/firmware/edid
sudo cp /sys/class/drm/card0-DP-3/edid /usr/lib/firmware/edid/my-monitor.bin

# 3. Add the kernel parameter
sudo grubby --update-kernel=ALL \
    --args="drm.edid_firmware=HDMI-A-2:edid/my-monitor.bin"
```

[Source: *How to override the EDID data of a monitor under Linux*, foosel.net](https://foosel.net/til/how-to-override-the-edid-data-of-a-monitor-under-linux/)

Internally, the override is implemented by `drm_load_edid_firmware()` in `drivers/gpu/drm/drm_edid_load.c`. When the connector name matches the firmware parameter, the kernel calls `request_firmware()` instead of issuing DDC reads. The returned blob is validated as a normal EDID and then passed through the same parsing pipeline as a hardware-read EDID.

### 7.2 Sysfs Write Interface

The sysfs EDID attribute at `/sys/class/drm/<card>-<output>/edid` is normally read-only (populated by the kernel from the DDC read). Some kernels and configurations expose a writable path for runtime override:

```bash
# Write a custom EDID binary to the connector at runtime
sudo cp /path/to/custom.edid /sys/class/drm/card0-HDMI-A-1/edid
```

Note: writable sysfs EDID injection is not universally available in upstream kernels and may require a driver-specific patch. The `drm_connector_override_edid()` function in `drm_edid.c` provides the in-kernel API that accomplishes the same effect without a DDC re-read.

### 7.3 Use Cases

**Headless servers with virtual EDID dongles**: A physical "EDID emulator" dongle (HDMI or DP) plugs into a GPU output and presents a fixed EDID over DDC without any real display attached. This allows the GPU to allocate a framebuffer and run a compositor in headless mode. Software EDID override (`drm.edid_firmware`) achieves the same result without hardware.

**Broken monitor firmware**: Monitors that ship with incorrect preferred timings, wrong screen size dimensions, or missing CEA blocks can be driven correctly by substituting a hand-crafted EDID binary. Tools like `edid-generator` or Lightware's Easy EDID Creator can produce compliant EDID binaries for any timing specification.

**Automated testing**: CI systems that run display driver tests (IGT GPU Tools) use virtual connectors with injected EDID blobs to exercise specific parsing code paths without physical hardware.

---

## 8. DDC/CI Monitor Control with ddcutil

### 8.1 VCP Code Space

DDC/CI (Display Data Channel Command Interface) is an extension of the DDC protocol that adds bidirectional monitor control. The **Monitor Control Command Set** (MCCS) defines a **Virtual Control Panel** (VCP) feature code space — a 256-entry table of monitor adjustable parameters:

| VCP code | Feature | Type |
|----------|---------|------|
| `0x10`   | Luminance (Brightness) | Continuous 0–100 |
| `0x12`   | Contrast | Continuous 0–100 |
| `0x14`   | Select Color Preset | Non-continuous (5500K, 6500K, 7500K…) |
| `0x16`   | Red Video Gain | Continuous |
| `0x60`   | Input Source Select | Non-continuous |
| `0xAC`   | Horizontal Frequency | Read-only |
| `0xAE`   | Vertical Frequency | Read-only |
| `0xB2`   | Flat Panel Sub-Pixel Layout | Read-only |
| `0xD6`   | Power Mode | DPMS states |
| `0xDF`   | VCP Version | Read-only |

[Source: ddcutil documentation at ddcutil.com/commands](https://www.ddcutil.com/commands/); [GitHub repository at github.com/rockowitz/ddcutil](https://github.com/rockowitz/ddcutil)

### 8.2 Device Access: I2C vs DP AUX

On **HDMI and DVI** connections, DDC/CI commands travel over the same I²C bus as EDID reads, but at the **management address `0x37`** (7-bit) rather than `0x50`. The kernel exposes this bus as `/dev/i2c-N` where N is the adapter index. `ddcutil` discovers the right adapter by checking which I²C device on the bus acknowledges `0x37`.

On **DisplayPort**, DDC/CI commands are tunnelled through the DP AUX channel using the I²C-over-AUX protocol at the same address `0x37`. The kernel exposes the AUX channel as `/dev/drm_dp_auxN`. `ddcutil` (version 1.3+) supports DP AUX access with the `--ddc-method=i2c-dev-dp` option.

To discover which kernel device maps to which monitor:

```bash
ddcutil detect
```

Example output:
```
Display 1
   I2C bus:  /dev/i2c-4
   EDID synopsis:
      Mfg id:               DEL
      Model:                DELL U2722D
      Product code:         16690
      Serial number:        ...
      Binary serial number: 0
      Manufacture year:     2021
```

### 8.3 Example Session

```bash
# Read current brightness
ddcutil getvcp 10 --display 1
# Output: VCP code 0x10 (Brightness): current value = 75, max value = 100

# Set brightness to 60%
ddcutil setvcp 10 60 --display 1

# Read contrast
ddcutil getvcp 12 --display 1

# Switch input to DisplayPort (VCP code 0x60, value 0x0F = DP-1)
ddcutil setvcp 60 0x0f --display 1

# Switch input to HDMI-1 (value 0x11)
ddcutil setvcp 60 0x11 --display 1

# Read monitor capability string (MCCS capabilities)
ddcutil capabilities --display 1

# Dump all VCP values for the display
ddcutil getvcp all --display 1
```

For programmatic access, `libddcutil` provides a C API. Applications that need to control monitor brightness (e.g., power management daemons) can link against `libddcutil.so` rather than forking the `ddcutil` binary.

---

## 9. HDCP Sink Capability Discovery

### 9.1 HDCP 1.4 Key Exchange

**HDCP 1.4** (High-bandwidth Digital Content Protection) protects digital video streams from unauthorised recording. Critically, **HDCP 1.4 capability is not signalled in EDID**. Instead, the source (GPU) discovers whether the sink (monitor) supports HDCP 1.4 by initiating the authentication protocol directly:

1. The source reads the sink's 40-bit **Bksv** (Block Key Selection Vector) from HDCP DDC address `0x74`, register `0x00` (HDMI) or from DPCD register `0x68000` (DP).
2. The source checks the Bksv against the SRM (System Renewability Message) revocation list to ensure the key is not revoked.
3. The source writes its own Aksv and initiates the HDCP 1.4 handshake.

For HDMI, all HDCP registers live in the DDC channel at I²C address `0x74`. For DisplayPort, they live in DPCD:

```
DPCD 0x0068:  Bksv (5 bytes)
DPCD 0x006D:  Bcaps register (bit 1 = REPEATER, bit 0 = HDCP_CAPABLE)
DPCD 0x006E:  Bstatus
```

### 9.2 HDCP 2.3 Capability via DPCD

**HDCP 2.2/2.3** capability *is* signalled explicitly in the sink's device capability registers:

- **DisplayPort**: DPCD register `0x006A`, bit 1 (`HDCP_2_2_REP_CAPABLE` or sink-side `HDCP2_2_CAPABLE`). Bit 0 signals HDCP 1.x support.
- **HDMI**: Via the HDMI Forum VSDB (OUI `0xC45DD8`) at byte offset `[n+4]`, bit 2 = `SCDC_Present`, and independently the HDCP 2.2 authentication uses the SCDC (Status and Control Data Channel) at DDC address `0x54`.

[Source: DPCD register map in `include/drm/display/drm_dp.h`, Linux kernel](https://github.com/torvalds/linux/blob/master/include/drm/display/drm_dp.h)

### 9.3 The Linux DRM HDCP Framework

The Linux DRM HDCP framework lives in `drivers/gpu/drm/display/drm_hdcp.c` and `include/drm/display/drm_hdcp.h`. It provides:

```c
/* Attach the "Content Protection" property to a connector */
int drm_connector_attach_content_protection_property(
        struct drm_connector *connector, bool hdcp_content_type);

/* Verify that a set of KSVs (HDCP 1.x keys) are not revoked */
bool drm_hdcp_check_ksvs_revoked(struct drm_device *dev,
                                  u8 *ksvs, int count);

/* Check HDCP link status over DP */
int drm_dp_hdcp_check_link(struct drm_dp_aux *aux,
                            struct drm_hdcp_info *hdcp_info);
```

[Source: HDCP integration patch discussion at spinics.net/lists/intel-gfx](https://www.spinics.net/lists/intel-gfx/msg204309.html); [kernel DRM KMS documentation at docs.kernel.org/gpu/drm-kms.html](https://docs.kernel.org/gpu/drm-kms.html)

The "Content Protection" connector property has three states exposed to userspace via KMS:

| Value | Constant | Meaning |
|-------|----------|---------|
| `0`   | `DRM_MODE_CONTENT_PROTECTION_UNDESIRED` | No content protection requested or active |
| `1`   | `DRM_MODE_CONTENT_PROTECTION_DESIRED` | Userspace requests protection; kernel is negotiating |
| `2`   | `DRM_MODE_CONTENT_PROTECTION_ENABLED` | Protection successfully established |

A Wayland compositor or video player that wants to play DRM-protected content sets the property to `DESIRED` via `drmModeAtomicAddProperty()`. The kernel driver (i915, amdgpu, etc.) then invokes the HDCP authentication state machine. If authentication fails (revoked key, unsupported HDCP version, repeater list exceeded), the property stays at `DESIRED` and the driver may block scanout of protected content. An associated "HDCP Content Type" property (value 0 = Type 0, value 1 = Type 1) controls whether HDCP 1.x is acceptable or HDCP 2.2+ is required.

---

## 10. Practical Debugging

### 10.1 Reading EDID from sysfs

Every DRM connector exposes its EDID blob in sysfs:

```bash
# List all connectors and their status
for p in /sys/class/drm/*/status; do
    con=${p%/status}
    printf "%-30s  %s\n" "${con##*/}" "$(cat $p)"
done

# Read the raw EDID of a specific connector
cat /sys/class/drm/card0-HDMI-A-1/edid | xxd | head -20

# Decode it in human-readable form (requires edid-decode or v4l-utils)
edid-decode /sys/class/drm/card0-HDMI-A-1/edid

# Alternative: use get-edid from i2c-tools package
get-edid -b 4 | decode-edid   # -b = I2C bus number
```

Sample `edid-decode` output (abridged):

```
EDID (hex):
00ffffffffffff00410c28031c000000
...

Block 0, Base EDID:
  EDID Structure Version & Revision: 1.4
  Vendor & Product Identification:
    Manufacturer: AUO
    Model: 0x2803
    Made in: week 28 of 2019
  Basic Display Parameters & Features:
    Digital display
    8 bits per primary color channel
    DisplayPort interface
    34 cm x 19 cm (13.4" x 7.5" diag, 15.0" total)
    Gamma: 2.20
    ...
  Standard Timings: none
  Detailed Timing Descriptors:
    DTD 1:  2560x1440   59.951 Hz  16:9    88.787 kHz 241.500 MHz
             Hfront   48 Hsync  32 Hback  80 Hpol P
             Vfront    3 Vsync   5 Vback  33 Vpol N
             Serial number: --
             Monitor name: --
CEA-861 Extension Block #1:
  ...
  Audio Data Block:
    Linear PCM:
      Max channels: 2
      Supported Sample Rates (kHz): 48 44.1 32
      Supported Bits: 24 20 16
  Video Data Block:
    VIC  16:  1920x1080   60.000000 Hz  16:9    67.500 kHz 148.500 MHz (native)
```

### 10.2 Interpreting edid-decode Output

Key things to check:

1. **"EDID Structure Version & Revision"** — anything less than 1.3 is ancient; 1.4 is standard for modern displays.
2. **"Bits per primary color channel"** — must match what the GPU is configured to send. A mismatch causes washed-out or clipped colour.
3. **DTD 1 timing** — this must match the monitor's native resolution. If it shows a non-native timing, the kernel will pick the wrong preferred mode. A quirk entry may be needed.
4. **CEA extension present** — required for HDMI audio. If absent, HDMI audio will not work (the kernel's `drm_edid_to_eld()` will produce an empty ELD).
5. **"HDR Static Metadata Data Block"** — if absent, the kernel will not offer HDR output. Common on older monitors and TVs that pre-date CTA-861.3.
6. **"Colorimetry Data Block"** — if absent but HDR SMB is present, wide-gamut signalling is incomplete.
7. **Physical size** — if wrong, applications will compute incorrect DPI, leading to HiDPI scaling failures.

### 10.3 Common Failure Modes and Remedies

| Symptom | Likely EDID cause | Remedy |
|---------|------------------|--------|
| Wrong default resolution on first plug | DTD slot 1 is not the native mode | Add `EDID_QUIRK_PREFER_LARGE_60` if largest 60 Hz mode is correct |
| Audio over HDMI does not appear | No CEA extension or ADB missing | Check with `edid-decode`; the CEA must have ADB with LPCM entry |
| HDR mode not offered in compositor | No HDR SMB in CEA extension | For older monitors: the display genuinely lacks HDR capability |
| Wrong physical DPI reported | Screen size bytes 21–22 incorrect | Add `EDID_QUIRK_DETAILED_IN_CM` (if mm/cm confusion) or custom EDID |
| 30/36-bit deep color not working | HDMI VSDB missing or deep-color bits not set | Check VSDB byte 5; may require firmware update from monitor manufacturer |
| Refresh rate wrong (e.g., 59 Hz shown as 60 Hz) | DTD pixel clock encoded incorrectly by manufacturer | Usually cosmetic; use `xrandr --rate 60` to force the closest rate |
| Display not detected at all | EDID EEPROM read failure (I²C NAK) | Check cable; try `drm.edid_firmware` override; check dmesg for "EDID block 0 is invalid" |

### 10.4 Kernel Log Messages

Enable DRM debug output with:

```bash
# At runtime (requires root)
echo 0x7f > /sys/module/drm/parameters/debug

# Or via kernel parameter
drm.debug=0x7f
```

Relevant `drm_edid.c` log messages:

```
# Normal operation
drm_edid: [HDMI-A-1] EDID block 0 is valid
drm_edid: [HDMI-A-1] Applying EDID quirk mask 0x00000002

# CEA parsing
drm_edid: [HDMI-A-1] CEA extension found
drm_edid: [HDMI-A-1] Number of extension blocks: 1

# Failure cases
drm_edid: [HDMI-A-1] EDID block 0 is invalid (bad header)
drm_edid: [HDMI-A-1] EDID block 1 is invalid (bad checksum)
drm_edid: [HDMI-A-1] No EDID after retries, assuming disconnected
```

The quirk application is logged at `DRM_DEBUG_KMS` level, so it appears only with debug enabled. Checking the log for `"Applying EDID quirk"` confirms which quirk flags were matched.

---

## 11. Integrations

This chapter is closely coupled to several other chapters in the book:

**Ch2 — KMS Connector Modes**: The `drm_display_mode` list that KMS presents to userspace (via `DRM_IOCTL_MODE_GETCONNECTOR`) is built entirely by the EDID parsing pipeline described here: DTDs become preferred modes, SVDs become driver modes, and established/standard timings fill in the remainder. The `DRM_MODE_TYPE_PREFERRED` flag that Wayland compositors use to select the default mode is set by `drm_edid.c` based on the EDID feature-support bit and DTD slot 1.

**Ch128 — DisplayPort MST**: Each downstream sink in an MST tree has its own EDID, retrieved over the sideband channel using `drm_dp_mst_get_edid()`. The EDID is then parsed through exactly the same `drm_edid_read_dp()` → `drm_edid_connector_update()` pipeline. Tiled Display Topology (DisplayID block `0x28`) and Container ID (`0x29`) allow the MST layer to group multiple physical connectors into a single logical display.

**Ch140 — HDMI/DP Audio (ELD)**: The Audio ELD (EDID-Like Data) that the HDA audio driver uses to configure the audio codec is generated from the CEA Audio Data Block by `drm_edid_to_eld()`. The SAD list (Short Audio Descriptors), speaker allocation, and the monitor name string (from the DTD Monitor Name descriptor) all flow from EDID parsing into the ELD structure.

**Ch158 — HDR on Linux**: The HDR output pipeline keys on `drm_display_info.hdr_sink_metadata`, which is populated here from the CEA HDR Static Metadata Block (extended tag `0x06`). MaxCLL, MaxFALL, and EOTF support bits parsed in section 3.1 of this chapter are the direct inputs to HDR tonemapping decisions and the `hdr_output_metadata` blob that the compositor sets on the CRTC.

**Ch181 — Display Standards**: The physical layer standards (HDMI 1.4, 2.0, 2.1; DisplayPort 1.4, 2.0) each expand the capability block vocabulary that EDID/DisplayID can carry. HDMI 2.1 mandates the HDMI Forum VSDB and specific DisplayID extension blocks. DP 2.0 adds UHBR link rates signalled in DPCD that complement the timing information in EDID/DisplayID.

**Ch182 — DDC I²C Channel**: This chapter describes the I²C bus over which EDID is read (DDC2B at 100 kHz, slave address `0x50`) and over which DDC/CI commands travel (address `0x37`). The `drm_edid_read_ddc()` call described in section 5.1 directly uses the I²C adapter provided by the DDC implementation in Ch182.

**Ch184 — Embedded DisplayPort (eDP)**: For laptop panels, the DPCD supplement to EDID is particularly important: DPCD registers (`0x000`–`0x00D`) describe link rate capability, lane count, and downspread support. Many eDP panels provide only minimal EDID data and rely on the host driver to look up panel-specific parameters from DMI tables or device-tree properties, supplementing or overriding what EDID reports.

**Ch187 — CEC (Consumer Electronics Control)**: The HDMI CEC physical address used to construct the logical address topology is read from bytes 4–5 of the HDMI VSDB in the CEA extension — the Source Physical Address (SPA) field described in section 2.4. The CEC subsystem (`drivers/media/cec/`) reads this value through `cec_notifier_set_phys_addr_from_edid()`, which parses the VSDB directly from the EDID blob.

---

## Roadmap

### Near-term (6–12 months)
- The kernel's `drm_edid.c` is receiving ongoing refactoring to fully migrate drivers from the legacy raw `struct edid *` API to the opaque `struct drm_edid` wrapper; remaining in-tree drivers (primarily older HDMI drivers and bridge chips) are expected to complete this transition, closing the last paths where invalid EDID memory could escape the parsing layer.
- DisplayID 2.1 support (released by VESA in 2024) adds new block types for variable refresh rate (VRR) capability and improved DSC negotiation; kernel patches are expected to land the new block parsers in `drm_edid.c` within this window.
- The `edid-decode` reference tool (maintained at `https://git.linuxtv.org/edid-decode.git`) is tracking CTA-861-I, the next CTA-861 revision that adds explicit signalling for 240 Hz VRR ranges and expanded HDR10+ metadata; kernel parser updates will follow the spec.
- Automatic HDR metadata application from EDID-parsed MaxCLL/MaxFALL into the compositor pipeline (via `drm_connector_state.hdr_output_metadata`) is being wired up in KWin and Mutter, eliminating the need for per-compositor workarounds to enable HDR on capable displays.

### Medium-term (1–3 years)
- DisplayID 2.x is expected to become the sole capability signalling mechanism for HDMI 2.1b and DisplayPort 2.1 Ultra-High Bit Rate (UHBR) displays; the legacy EDID base block is likely to be demoted to a compatibility stub, with Linux parsing logic shifting to treat DisplayID as the primary source of truth.
- The VESA MCCS 3.1 specification, which extends the DDC/CI VCP code space for HDR luminance target control and wide-gamut preset selection, is expected to gain first-class `libddcutil` support, enabling power managers to adjust HDR peak brightness at runtime through a standardised interface.
- Coordinated multi-tile display support (DisplayID Tiled Display Topology + DisplayPort MST) is targeted for Wayland protocols via the `ext-workspace` and potential `tiled-output` Wayland extensions, allowing compositors to expose a unified logical surface across physically tiled panels without driver-level stitching.
- HDCP 2.3 per-stream content type enforcement (distinguishing Type 0 from Type 1 content on HDMI 2.1 streams) is expected to reach mainline via the Intel and AMDGPU HDCP state machines, enabling protected streaming of 4K HDR Blu-ray-equivalent content on Linux.

### Long-term
- A unified capability discovery ABI — potentially merging EDID, DisplayID, DPCD, and DDC/CI VCP data into a single structured DRM property object — has been discussed at XDC conferences as a long-term successor to the current piecemeal property set; such an ABI would allow compositors to query all display capabilities through a single `drmModeGetConnector` extension rather than parsing raw EDID blobs in userspace.
- As display interfaces transition to optical and USB4-tunnelled DP, the I²C DDC channel may be supplanted by firmware-mediated capability exchange (similar to USB Billboard devices); the Linux DRM EDID subsystem would need to support capability data arriving over platform firmware tables (ACPI `_DSM` or devicetree overlays) rather than runtime DDC reads.
- Long-term adoption of VESA's proposed Adaptive Sync Display (ASD) certification metadata in DisplayID may allow the kernel to precisely characterise VRR range, overdrive latency, and frame-rate multiplier capabilities, enabling Mesa and Wayland compositors to implement frame-pacing algorithms without per-monitor heuristics.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
