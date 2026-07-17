# Chapter 181: Modern Display Interface Standards

## Scope

This chapter targets three audiences:

- **Systems and driver developers** — detailed coverage of how DisplayPort 2.1, HDMI 2.1, USB4, and Thunderbolt 5 are plumbed into the Linux DRM/KMS subsystem, including which kernel config options gate each feature and which upstream commits introduced UHBR and FRL support.
- **Hardware engineers** — the complete link-rate tables, encoding-overhead analysis, and bandwidth comparisons they need to select the right interface for a given resolution and refresh-rate target.
- **Application developers** — the bandwidth arithmetic worked examples that show precisely where each interface hits its limit, and what Display Stream Compression (DSC) buys in practice.

---

## Table of Contents

- [1. DisplayPort Evolution: DP 1.1 through DP 2.1](#1-displayport-evolution-dp-11-through-dp-21)
  - [1.1 Legacy Link Rates and 8b/10b Encoding](#11-legacy-link-rates-and-8b10b-encoding)
  - [1.2 DisplayPort 2.0 and the 128b/132b Leap](#12-displayport-20-and-the-128b132b-leap)
  - [1.3 DisplayPort 2.1: Refinements, Cables, and Mandatory DSC](#13-displayport-21-refinements-cables-and-mandatory-dsc)
  - [1.4 Panel Replay: Replacing PSR2](#14-panel-replay-replacing-psr2)
  - [1.5 Forward Error Correction (FEC)](#15-forward-error-correction-fec)
- [2. HDMI Evolution: 1.0 through 2.1 and Beyond](#2-hdmi-evolution-10-through-21-and-beyond)
  - [2.1 TMDS Era: HDMI 1.0 to 2.0b](#21-tmds-era-hdmi-10-to-20b)
  - [2.2 HDMI 2.1: Fixed Rate Link Replaces TMDS](#22-hdmi-21-fixed-rate-link-replaces-tmds)
  - [2.3 FRL Levels: From 9 Gbps to 48 Gbps](#23-frl-levels-from-9-gbps-to-48-gbps)
  - [2.4 HDMI 2.1 Feature Suite: eARC, ALLM, VRR, QFT, QMS](#24-hdmi-21-feature-suite-earc-allm-vrr-qft-qms)
  - [2.5 HDMI 2.1a and 2.1b](#25-hdmi-21a-and-21b)
  - [2.6 DisplayPort vs HDMI Bandwidth Comparison](#26-displayport-vs-hdmi-bandwidth-comparison)
- [3. USB4 Display Tunnelling](#3-usb4-display-tunnelling)
  - [3.1 USB4 v1 vs v2: 40 and 80 Gbps](#31-usb4-v1-vs-v2-40-and-80-gbps)
  - [3.2 DisplayPort Tunnels over USB4](#32-displayport-tunnels-over-usb4)
  - [3.3 Bandwidth Sharing: Display Tunnel vs USB Data](#33-bandwidth-sharing-display-tunnel-vs-usb-data)
  - [3.4 Thunderbolt Compatibility Mode](#34-thunderbolt-compatibility-mode)
- [4. Thunderbolt 3, 4, and 5](#4-thunderbolt-3-4-and-5)
  - [4.1 Thunderbolt 3: 40 Gbps with Dual DP 1.2](#41-thunderbolt-3-40-gbps-with-dual-dp-12)
  - [4.2 Thunderbolt 4: Mandatory DP 2.0](#42-thunderbolt-4-mandatory-dp-20)
  - [4.3 Thunderbolt 5: 120 Gbps Bandwidth Boost and Dual 8K](#43-thunderbolt-5-120-gbps-bandwidth-boost-and-dual-8k)
  - [4.4 Daisy-Chain Topology and Multi-Display Limits](#44-daisy-chain-topology-and-multi-display-limits)
- [5. HDBaseT: AV Over Structured Cabling](#5-hdbaset-av-over-structured-cabling)
  - [5.1 HDBaseT 2.0 Technical Architecture](#51-hdbaset-20-technical-architecture)
  - [5.2 Linux Support Status](#52-linux-support-status)
- [6. VESA Adaptive-Sync and DisplayHDR Certification](#6-vesa-adaptive-sync-and-displayhdr-certification)
  - [6.1 Adaptive-Sync and Adaptive-Sync Display 1.1a](#61-adaptive-sync-and-adaptive-sync-display-11a)
  - [6.2 DisplayHDR Certification Tiers](#62-displayhdr-certification-tiers)
  - [6.3 What the Kernel Enforces vs What the Panel Certifies](#63-what-the-kernel-enforces-vs-what-the-panel-certifies)
- [7. Linux Kernel Support Status per Standard](#7-linux-kernel-support-status-per-standard)
  - [7.1 DisplayPort UHBR in i915](#71-displayport-uhbr-in-i915)
  - [7.2 HDMI 2.1 FRL in amdgpu](#72-hdmi-21-frl-in-amdgpu)
  - [7.3 USB4 Display Tunnel in the Thunderbolt Driver](#73-usb4-display-tunnel-in-the-thunderbolt-driver)
  - [7.4 Key Kernel Configuration Options](#74-key-kernel-configuration-options)
- [8. Bandwidth Arithmetic: Worked Examples](#8-bandwidth-arithmetic-worked-examples)
  - [8.1 4K/144Hz: Does DP 1.4 Suffice?](#81-4k144hz-does-dp-14-suffice)
  - [8.2 5K/60Hz Dual-Monitor](#82-5k60hz-dual-monitor)
  - [8.3 8K/60Hz: Where Only DP 2.1 and HDMI 2.1 Qualify](#83-8k60hz-where-only-dp-21-and-hdmi-21-qualify)
  - [8.4 DSC Compression Ratio Math](#84-dsc-compression-ratio-math)
- [Integrations](#integrations)

---

## 1. DisplayPort Evolution: DP 1.1 through DP 2.1

### 1.1 Legacy Link Rates and 8b/10b Encoding

DisplayPort launched in 2007 with a single link rate and has evolved through a series of revisions that doubled or tripled bandwidth at each major step. All versions prior to DP 2.0 use **8b/10b line coding**: every 8 data bits are transmitted as 10 bits, absorbing DC balance requirements at the cost of 20 % overhead. The effective data efficiency is therefore always 80 % of the raw symbol rate.

| Mode | Introduced | Per-Lane Rate | ×4 Raw | Effective Data Rate |
|------|-----------|--------------|--------|---------------------|
| RBR (Reduced Bit Rate) | DP 1.0 | 1.62 Gbps | 6.48 Gbps | 5.18 Gbps |
| HBR (High Bit Rate) | DP 1.0 | 2.70 Gbps | 10.80 Gbps | 8.64 Gbps |
| HBR2 | DP 1.2 | 5.40 Gbps | 21.60 Gbps | 17.28 Gbps |
| HBR3 | DP 1.3 | 8.10 Gbps | 32.40 Gbps | 25.92 Gbps |
| UHBR10 | DP 2.0/2.1 | 10.00 Gbps | 40.00 Gbps | 38.69 Gbps |
| UHBR13.5 | DP 2.0/2.1 | 13.50 Gbps | 54.00 Gbps | 52.22 Gbps |
| UHBR20 | DP 2.0/2.1 | 20.00 Gbps | 80.00 Gbps | 77.37 Gbps |

[Source: TFTCentral DisplayPort 2.1 Guide](https://tftcentral.co.uk/articles/a-guide-to-displayport-2-1-and-previously-2-0-certifications-standards-cables-and-areas-of-confusion-and-concern)

The DPCD (DisplayPort Configuration Data) auxiliary channel (1 Mbps, half-duplex, over the AUX differential pair) is the out-of-band channel used for link training, EDID/DisplayID reads, DSC capability negotiation, Panel Replay control, and MST topology discovery. It has remained architecturally constant across all DP versions, although the register map has expanded substantially.

**DP 1.2** (2010) added Multi-Stream Transport (MST), allowing up to 63 logical sub-streams to be time-division multiplexed over a single physical link. MST is covered in depth in Ch128.

**DP 1.4** (2016) introduced DSC 1.2 (Display Stream Compression), Forward Error Correction (FEC), and HDR signalling through the VSC SDP (Video Stream Configuration Supplemental Data Packet). FEC became mandatory to enable DSC: the FEC parity symbols allow the sink to correct bit errors that DSC's lossy pipeline cannot tolerate. DP 1.4 also added the Panel Self-Refresh 2 (PSR2) selective-update mechanism.

### 1.2 DisplayPort 2.0 and the 128b/132b Leap

DisplayPort 2.0 was ratified by VESA in June 2019. It replaced 8b/10b encoding with **128b/132b** encoding, in which 128 data bits are carried in 132 transmitted bits — an intrinsic efficiency of 128/132 ≈ 96.96 %. After accounting for Forward Error Correction overhead (see §1.5), the net efficiency is approximately **96.7 %**, nearly 17 percentage points better than 8b/10b.

128b/132b also brought three new Ultra High Bit Rate (UHBR) link rates:

- **UHBR10**: 10 Gbps per lane → 40 Gbps aggregate (×4)
- **UHBR13.5**: 13.5 Gbps per lane → 54 Gbps aggregate
- **UHBR20**: 20 Gbps per lane → 80 Gbps aggregate

These rates require a new PHY layer using NRZ (Non-Return-to-Zero) signalling with enhanced signal conditioning rather than the MLT-3 and NRZ combinations of earlier versions.

[Source: VESA DisplayPort FAQ](https://www.displayport.org/faq/)

### 1.3 DisplayPort 2.1: Refinements, Cables, and Mandatory DSC

VESA released DisplayPort 2.1 in October 2022 as a refinement of DP 2.0 carrying the same UHBR link rates and 80 Gbps ceiling. Its primary additions were:

1. **USB4 interoperability tightening**: DP 2.1 formally aligns its tunnelling semantics with USB4 Gen 3 and Thunderbolt 4/5 so that a DP 2.1 stream can be carried over USB4 tunnels without custom adaptation.

2. **Certified cable programmes**: VESA introduced the **DP40** and **DP80** cable certifications. A DP40 cable is certified to carry UHBR10 (40 Gbps); a DP80 cable carries UHBR20 (80 Gbps). The specifications accommodate cable lengths beyond 2 m for DP40 and beyond 1 m for DP80 through tighter connector and cable electrical tolerances.

3. **Mandatory DSC 1.2a**: All DP 2.1 sources and sinks are required to implement DSC 1.2a. This is a stricter version of the DP 1.4 DSC 1.2 requirement, adding extended syntax support for higher colour depths and improved slice layout.

4. **Mandatory Panel Replay**: All DP 2.1 endpoints must support Panel Replay (see §1.4).

[Source: VESA DisplayPort 2.1 Press Release](https://vesa.org/featured-articles/vesa-releases-displayport-2-1-specification/)

The DPCD UHBR capability register is at offset `0x002205` (`DP_128B132B_SUPPORTED_LINK_RATES`) in the sink's DPCD map. The i915 driver reads this register in `intel_dp_set_dpcd_sink_rates()` in [`drivers/gpu/drm/i915/display/intel_dp.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_dp.c):

```c
/* drivers/gpu/drm/i915/display/intel_dp.c */
/* Populate supported UHBR rates from DPCD */
static void intel_dp_set_dpcd_sink_rates(struct intel_dp *intel_dp)
{
    /* ... 8b/10b rates read first ... */

    /* 128b/132b (UHBR) rates */
    if (intel_dp->dpcd[DP_DPCD_REV] >= DP_DPCD_REV_14) {
        u8 supported = intel_dp->dpcd[DP_128B132B_SUPPORTED_LINK_RATES];
        if (supported & DP_UHBR10)
            intel_dp->sink_rates[count++] = 1000000;  /* 10 Gbps */
        if (supported & DP_UHBR13_5)
            intel_dp->sink_rates[count++] = 1350000;  /* 13.5 Gbps */
        if (supported & DP_UHBR20)
            intel_dp->sink_rates[count++] = 2000000;  /* 20 Gbps */
    }
}
```

> **Note:** The exact register offset and bitmask names should be verified against the current kernel header `include/drm/display/drm_dp.h`, as they are updated with each DPCD revision.

### 1.4 Panel Replay: Replacing PSR2

Panel Replay (PR) is the DP 2.1 successor to Panel Self-Refresh 2 (PSR2). PSR2 allowed the GPU to write only changed regions of the frame to the display's self-refresh buffer between VBLANK periods, reducing link utilisation during static or near-static content. Panel Replay extends this to the DP 2.1 tunnelling context and changes the VSC SDP revision:

- PSR2 uses VSC SDP revision `0x05`
- Panel Replay uses VSC SDP revision `0x07`, and sets new fields in the DP_TRANS_DP2 register range

VESA claims Panel Replay can reduce DisplayPort tunnelling packet transport bandwidth by more than **99 %** during fully static frames, since only selective region updates are transmitted.

[Source: VESA DisplayPort 2.1 specification overview](https://vesa.org/featured-articles/vesa-releases-displayport-2-1-specification/)

In the Linux i915 driver, Panel Replay is controlled via `intel_psr.c` alongside PSR2. A kernel module parameter `enable_psr` controls both, and Panel Replay can be independently disabled via:

```bash
# Disable Panel Replay while keeping PSR2 (i915 kernel parameter)
# /etc/modprobe.d/i915.conf
options i915 enable_psr=1
```

The driver detects Panel Replay capability from DPCD register `DP_PANEL_REPLAY_CAP` and distinguishes it from PSR2 via the VSC SDP revision field. The `TRANS_DP2_CTL` register on Meteor Lake (MTL) and later platforms is used to enable Panel Replay at the transcoder level.

[Source: Linux kernel i915 Panel Replay patches](https://lists.freedesktop.org/archives/intel-gfx/2024-June/351920.html)

### 1.5 Forward Error Correction (FEC)

FEC was introduced in DP 1.4 for use with DSC and carried forward into DP 2.1. The DP specification uses a **Reed-Solomon RS(254,250)** code: each FEC block contains 250 data symbols plus 4 RS parity symbols, with 5 additional FEC overhead bits and 1 CD_ADJ (disparity adjustment) bit, producing a total overhead of approximately **2.4 %**.

In the 128b/132b system, FEC is applied before the 128b/132b scrambler, so the net channel efficiency is:

```
128/132 × (1 − 0.024) ≈ 0.9696 × 0.976 ≈ 0.9462
```

However, VESA reports the effective efficiency as ≈96.7 % because FEC and the 128b/132b overhead interact differently at the symbol boundary level than a simple multiplication suggests.

[Source: DisplayPort FEC and DSC overview](https://unigraf.fi/app/uploads/2020/02/Forward-error-correction-and-display-stream-correction.pdf)

FEC is enabled through the DPCD `FEC_CONFIGURATION` register (`0x120`) and must be negotiated before DSC is activated. In the kernel the helper `drm_dp_get_fec_ready_flag()` in `drivers/gpu/drm/display/drm_dp_helper.c` reads this register and returns whether the sink is prepared to accept an FEC-protected stream.

---

## 2. HDMI Evolution: 1.0 through 2.1 and Beyond

### 2.1 TMDS Era: HDMI 1.0 to 2.0b

HDMI was launched in 2002 as a royalty-bearing consumer-AV successor to DVI. Like DVI, the original HDMI used **Transition-Minimised Differential Signalling (TMDS)**, an 8b/10b line code driven at the pixel clock rate on three data channels plus a separate clock channel — four differential pairs in total.

Key TMDS-era milestones:

| Version | Year | Max TMDS Clock | Bandwidth | Capability |
|---------|------|---------------|-----------|------------|
| 1.0 | 2002 | 165 MHz | 4.95 Gbps | 1080p/60 |
| 1.3 | 2006 | 340 MHz | 10.2 Gbps | Deep colour, xvYCC |
| 1.4 | 2009 | 340 MHz | 10.2 Gbps | 4K/30 (3840×2160), ARC, 3D |
| 2.0 | 2013 | 600 MHz | 18.0 Gbps | 4K/60, 32-audio-ch |
| 2.0b | 2016 | 600 MHz | 18.0 Gbps | HDR10, HLG support |

TMDS uses three data pairs (each at the pixel clock × 10 for 8b/10b overhead) plus a dedicated clock pair. At the HDMI 2.0 maximum of 600 MHz pixel clock, each data lane carries 6 Gbps → 3 lanes × 6 = 18 Gbps aggregate. The 8b/10b overhead reduces effective payload to 80 %, giving 14.4 Gbps of usable data bandwidth.

[Source: HDMI Forum HDMI 2.1 Announcement](https://www.hdmi.org/announce/detail/172)

### 2.2 HDMI 2.1: Fixed Rate Link Replaces TMDS

HDMI 2.1, published by the HDMI Forum in 2017, replaced TMDS with **Fixed Rate Link (FRL)** — a completely new physical layer that eliminates the separate clock lane, adds a fourth data lane, and uses **16b/18b** encoding instead of 8b/10b. Encoding efficiency rises from 80 % to 88.9 % (16/18).

FRL also switches from DC-coupled to AC-coupled signalling, which simplifies long-run cabling and allows the forward error correction scheme to operate more aggressively.

The architecture change also enabled a **link-training protocol**: the source and sink negotiate the FRL rate by starting at the highest supported rate and stepping down if the physical link cannot sustain it, analogous to DisplayPort's link training but specifically designed for the HDMI cable plant.

[Source: Allion Labs HDMI 2.1 8K Concepts and Specifications](https://www.allion.com/hdmi-8k-protocol-concept/)

### 2.3 FRL Levels: From 9 Gbps to 48 Gbps

HDMI 2.1 defines seven FRL operating modes, designated FRL0 through FRL6:

| FRL Level | Lanes | Per-Lane Rate | Total Raw | Effective (×88.9%) |
|-----------|-------|--------------|-----------|---------------------|
| FRL0 | — | TMDS mode | ≤18 Gbps | ≤14.4 Gbps |
| FRL1 | 3 | 3 Gbps | 9 Gbps | 8.0 Gbps |
| FRL2 | 3 | 6 Gbps | 18 Gbps | 16.0 Gbps |
| FRL3 | 4 | 6 Gbps | 24 Gbps | 21.3 Gbps |
| FRL4 | 4 | 8 Gbps | 32 Gbps | 28.4 Gbps |
| FRL5 | 4 | 10 Gbps | 40 Gbps | 35.6 Gbps |
| FRL6 | 4 | 12 Gbps | 48 Gbps | 42.7 Gbps |

[Source: Granite River Labs HDMI 2.1 FRL Technical Article](https://www.graniteriverlabs.com/en-us/application-notes/technical-article-hdmi-2.1-fixed-rate-link-frl-mode-overview) [Source: AVPro Global FRL Data Rate Chart](https://www.avproglobal.com/pages/murideo-brand-frl-data-rate-chart)

FRL6 at 48 Gbps raw is sufficient for 4K/120Hz (10-bit HDR RGB) uncompressed, and 8K/60Hz (4:2:0 10-bit) — the canonical "HDMI 2.1 headline" resolutions. DSC 1.2 support in HDMI 2.1 (optional) enables 8K/60 in full 4:4:4 or 10K and beyond.

### 2.4 HDMI 2.1 Feature Suite: eARC, ALLM, VRR, QFT, QMS

**eARC (Enhanced Audio Return Channel)**

The original HDMI ARC feature (introduced in HDMI 1.4) carries compressed audio from a TV back to a soundbar/AVR on the same HDMI cable, but only at approximately 1 Mbps — enough for Dolby Digital 5.1 and DTS 5.1 but not for lossless formats.

eARC upgrades the return channel to a dedicated differential pair operating at up to **37 Mbps**, sufficient for:
- Uncompressed LPCM (up to 32 channels, 192 kHz, 24-bit)
- Dolby TrueHD with Atmos object metadata
- DTS-HD Master Audio and DTS:X

eARC requires an HDMI 2.1 cable and HDMI 2.1 ports, though some HDMI 2.0 devices have implemented eARC via firmware update using the existing HDMI 2.0 pin 14 ARC wire repurposed for eARC signalling.

[Source: HDMI Forum eARC specification](https://www.hdmi.org/spec2sub/earc)

**ALLM (Auto Low Latency Mode)**

ALLM allows a source device (game console, PC) to signal to the display that it should automatically engage its lowest-latency "Game Mode" processing path. The source sets a bit in the HDMI Data Island Packet; the sink reads it and switches mode without user interaction.

[Source: HDMI 2.1 Feature Descriptions](https://www.hdmi.org/spec2sub/allm)

**HDMI Forum VRR**

HDMI Forum VRR allows the source to vary the frame delivery rate dynamically, causing the display to adjust its vertical blanking period accordingly. This is functionally similar to DisplayPort Adaptive-Sync (the basis for AMD FreeSync and NVIDIA G-Sync Compatible). VRR in HDMI 2.1 requires a V-Total negotiation mechanism using the Vendor Specific Video InfoFrame (VS-IF).

**QFT (Quick Frame Transport)**

QFT reduces the *display latency* component of end-to-end input lag by transmitting each frame's active video region at the highest possible rate and spending the remaining line time in a compressed blanking period. A frame is delivered to the display faster than at nominal rate, even though the display then waits for the next VSYNC trigger.

[Source: HDMI Forum Quick Frame Transport](https://www.hdmi.org/spec2sub/quickframetransport)

**QMS (Quick Media Switching)**

QMS eliminates the black-screen blank that occurs when a source switches refresh rate (e.g. from a 60 Hz menu to a 24 Hz film). QMS pre-arms the display for the incoming rate before the switch, so the transition is visually seamless.

[Source: HDMI Forum Quick Media Switching](https://www.hdmi.org/spec2sub/quickmediaswitching)

### 2.5 HDMI 2.1a and 2.1b

**HDMI 2.1a** (2022) added **Source-Based Tone Mapping (SBTM)**: the source performs the HDR tone-mapping itself rather than delegating it to the display, which is useful when blending SDR application UI with HDR game or video content. SBTM metadata is signalled through the extended HDMI Dynamic HDR InfoFrame.

**HDMI 2.1b** (2023) extended VRR, ALLM, and QFT to explicitly support **4K/144Hz** gaming monitors and TVs, adding higher refresh rate modes that were technically possible at FRL6 bandwidth but not previously profiled.

[Source: HDMI 2.1 vs 2.1a vs 2.1b guide](https://refreshratetest.online/articles/hdmi-2-1-vs-2-1a-vs-2-1b-cable-standards)

### 2.6 DisplayPort vs HDMI Bandwidth Comparison

| Interface | Max Raw | Effective Payload | 4K/60 | 4K/120 | 8K/60 |
|-----------|---------|------------------|-------|--------|-------|
| HDMI 2.0 (TMDS) | 18 Gbps | 14.4 Gbps | Yes (4:4:4 8b) | No | No |
| DP 1.4 (HBR3) | 32.4 Gbps | 25.9 Gbps | Yes | Yes (DSC) | Yes (DSC) |
| HDMI 2.1 (FRL6) | 48 Gbps | 42.7 Gbps | Yes | Yes | Yes (4:2:0) |
| DP 2.1 (UHBR10) | 40 Gbps | 38.7 Gbps | Yes | Yes | Yes (DSC) |
| DP 2.1 (UHBR20) | 80 Gbps | 77.4 Gbps | Yes | Yes | Yes |
| USB4 v2 (asymmetric) | 120 Gbps | ~116 Gbps | Yes | Yes | Yes |

---

## 3. USB4 Display Tunnelling

### 3.1 USB4 v1 vs v2: 40 and 80 Gbps

USB4 is a public specification based on the Thunderbolt 3 protocol, published by the USB Implementers Forum in 2019. USB4 unifies USB, DisplayPort, and PCIe tunnelling over a common physical layer (USB Type-C, four differential pairs, Gen 3 signalling).

[Source: Linux Kernel USB4 and Thunderbolt documentation](https://docs.kernel.org/admin-guide/thunderbolt.html)

**USB4 v1 (Gen 2×2 and Gen 3×2)**:
- Gen 2×2: 2 × 10 Gbps lanes = 20 Gbps symmetric
- Gen 3×2: 2 × 20 Gbps lanes = **40 Gbps** symmetric

**USB4 v2**, announced in September 2022:
- Gen 4×2: 2 × 40 Gbps lanes = **80 Gbps** symmetric
- Asymmetric boost mode: 3 upstream lanes + 1 downstream = **120 Gbps** unidirectional (40 Gbps return)
- First generation to tunnel DisplayPort 2.0 UHBR streams natively

[Source: Tom's Hardware USB4 Version 2.0 Announcement](https://www.tomshardware.com/news/usb-4-version-2-announced-80gbps)

### 3.2 DisplayPort Tunnels over USB4

USB4 encapsulates a DisplayPort stream inside a **DP Tunnel** using the USB4 tunnel protocol. The tunnel carries the DP Main Link (pixel data), the AUX channel (DPCD/EDID), and HPD (Hot Plug Detect) as separate logical sub-channels over USB4 bandwidth allocation units (BAUs).

The tunnel negotiation sequence:

1. The USB4 host router (in the CPU or Thunderbolt controller) creates a path to the device router (in the dock or display).
2. A Connection Manager (CM) — either in firmware or as a Linux kernel software CM — allocates bandwidth to the DP tunnel using the **DP Bandwidth Allocation (DPB)** protocol.
3. The CM reserves a static or dynamic slice of the USB4 Gen allocation for the DP path, expressed in units of USB4 Gen 3 lanes × rate.
4. The host GPU's DisplayPort source sees a logical DP AUX channel on the other end of the tunnel and performs normal DPCD negotiation, link training, and stream transmission.

[Source: AMD USB4 DisplayPort Tunneling driver, Phoronix](https://www.phoronix.com/news/AMDGPU-USB4-DP-Tunneling)

The DPCD link rate and lane count advertised through the tunnel correspond to the actual DP 2.0/2.1 capability of the display, subject to the USB4 bandwidth available. If the USB4 link is saturated by PCIe or USB 3 traffic, the CM may negotiate a lower DP bandwidth allocation, potentially forcing DSC or a reduced link rate.

### 3.3 Bandwidth Sharing: Display Tunnel vs USB Data

USB4's bandwidth management is asymmetric:

- **DisplayPort tunnel bandwidth** is allocated **statically** before the tunnel opens. The CM reserves a contiguous allocation from the USB4 fabric. This prevents glitches during pixel transmission.
- **USB 3.x tunnel bandwidth** is managed **dynamically**: the firmware can increase or decrease allocation in response to observed USB 3 traffic demand.
- **PCIe tunnel bandwidth** is allocated statically for the duration of the PCIe device being present.

The practical implication: plugging a high-bandwidth USB 3.2 device (e.g. a 10GbE NIC) into a USB4 dock while driving a 4K/60 display over the same USB4 link requires the CM to have pre-allocated sufficient bandwidth for both. The kernel emits a `TB_TUNNEL_DP` event with a "low bandwidth" notification if the display tunnel is not receiving its requested allocation.

[Source: Linux Thunderbolt/USB4 kernel documentation](https://docs.kernel.org/admin-guide/thunderbolt.html)

### 3.4 Thunderbolt Compatibility Mode

USB4 defines a **Thunderbolt compatibility mode** that allows USB4 hosts to interoperate with Thunderbolt 3 devices (docks, displays, eGPUs). In this mode the USB4 controller emulates TBT3 at the protocol level, including the router configuration space, tunnel setup, and security handshake. The Linux Thunderbolt driver (`drivers/thunderbolt/`) detects the peer device type and switches to the appropriate CM behaviour.

---

## 4. Thunderbolt 3, 4, and 5

### 4.1 Thunderbolt 3: 40 Gbps with Dual DP 1.2

Thunderbolt 3 (Intel, 2015) moved from the proprietary Mini DisplayPort connector to USB Type-C and doubled bandwidth from TBT2's 20 Gbps to **40 Gbps** (2 × 20 Gbps lanes, each full duplex). TBT3 provides:

- Up to 2 × DisplayPort 1.2 streams simultaneously (each up to HBR2, 21.6 Gbps raw)
- PCIe ×4 Gen 3 tunnel (32 Gbps)
- USB 3.1 Gen 2 tunnel (10 Gbps)
- 100 W USB-PD charging

TBT3 requires DisplayPort 1.2 as a **minimum** but supported optional DP 1.4 on later revisions (Intel's Titan Ridge controller added DP 1.4 support). The total 40 Gbps shared bandwidth constrains the display + data combination.

[Source: Thunderbolt 3 vs. Thunderbolt 4 comparison](https://www.windowscentral.com/thunderbolt-4-usb4-usb)

### 4.2 Thunderbolt 4: Mandatory DP 2.0

Thunderbolt 4 (Intel, 2020) kept the same 40 Gbps physical bandwidth as TBT3 but raised the mandatory minimum requirements:

- **DisplayPort 2.0 mandatory**: unlike TBT3 where DP 1.2 was the floor, TBT4 requires DP 2.0 support. This means Intel does not certify a TBT4 implementation if the connected GPU only supports DP 1.2.
- **Two simultaneous 4K/60 displays minimum** (vs TBT3's optional second display)
- **100 W charging** (mandated, vs optional in TBT3)
- **PCIe tunnelling at ×32 Gbps** with enhanced wake-from-sleep reliability
- **Mandatory IOMMU DMA protection** for PCIe tunnel security

[Source: Intel Thunderbolt 4 specification overview](https://www.intel.com/content/www/us/en/gaming/resources/upgrade-gaming-accessories-thunderbolt-4.html)

Because TBT4 still operates at 40 Gbps, driving two DP 2.0 UHBR10 streams simultaneously (each needing up to 40 Gbps) is only possible with DSC compression or reduced lane counts. The practical TBT4 display envelope is two 4K/120 Hz displays (each using DP HBR3 or UHBR10 with DSC) or one 8K/30 Hz display uncompressed.

### 4.3 Thunderbolt 5: 120 Gbps Bandwidth Boost and Dual 8K

Intel announced Thunderbolt 5 in September 2023 on the same USB4 v2 physical layer. TBT5 specifications:

- **Symmetric mode**: 80 Gbps (2 × 40 Gbps lanes)
- **Bandwidth Boost (asymmetric)**: 120 Gbps downstream / 40 Gbps upstream
- **DisplayPort 2.1** tunnelling (UHBR20 capable)
- **240 W USB-PD charging**
- **PCIe tunnelling at ×64 Gbps** in Bandwidth Boost mode

Bandwidth Boost engages automatically when the controller detects high-volume display traffic. In this mode three of the four USB4 Gen 4 lanes are configured in the downstream direction, delivering 120 Gbps to the display path while maintaining 40 Gbps for upstream USB and PCIe.

[Source: Intel Thunderbolt 5 Newsroom Announcement](https://newsroom.intel.com/client-computing/intel-introduces-thunderbolt-5-standard)

Display capabilities in Bandwidth Boost mode:

- **Triple 4K/144 Hz** displays
- **Dual 8K/60 Hz** displays (with DSC)
- Single **540 Hz** monitor at 1080p

[Source: Intel Thunderbolt 5 for Gaming](https://www.intel.com/content/www/us/en/learn/thunderbolt-5-for-gaming.html)

### 4.4 Daisy-Chain Topology and Multi-Display Limits

All Thunderbolt generations support **daisy-chaining** up to 6 devices in series off a single host port (plus the host itself = 7 nodes). Each Thunderbolt device contains an internal switch that passes the TBT fabric downstream. The 40 Gbps (TBT3/4) or 80/120 Gbps (TBT5) bandwidth is **shared** across all daisy-chained nodes; it is not replicated per device.

A daisy chain of 3 devices each consuming 10 Gbps of PCIe bandwidth leaves only 10 Gbps for the last device, which may be insufficient for a 4K display at high refresh rates without DSC. The Intel TBT controller's firmware connection manager handles bandwidth arbitration and will fail tunnel establishment if insufficient bandwidth is available, generating a `dmesg` error such as:

```
thunderbolt 0-1: not enough bandwidth for display
```

---

## 5. HDBaseT: AV Over Structured Cabling

### 5.1 HDBaseT 20 Technical Architecture

HDBaseT is a standard for carrying uncompressed HD/UHD video, audio, Ethernet, power (up to 100 W), USB, and RS-232/IR control over Category-5e/6/6A twisted-pair cabling. It is published and maintained by the HDBaseT Alliance.

[Source: HDBaseT Alliance Technology Overview](https://hdbaset.org/hdbaset-technology/)

**HDBaseT 1.0** (2010): Up to 10.2 Gbps (matching HDMI 1.4 TMDS rate), distances up to 100 m over Cat-6.

**HDBaseT 2.0** (2013): Extended to carry **4K/60 Hz (4:2:0)** or **4K/30 Hz (4:4:4)** — approximately 18 Gbps throughput matching HDMI 2.0's TMDS ceiling. Maximum distance is **100 m over Cat-6** (90 m over Cat-5e). Key specifications:

- 5PLAY™ feature set: video, audio, Ethernet (100Base-TX), power (PoH, up to 100 W), control
- Proprietary HDBaseT physical layer over 4 pairs of twisted copper (500 MHz bandwidth per pair)
- Supports HDCP 2.2 for content protection

**HDBaseT Slim**: A reduced-feature variant for consumer AV installations, omitting PoH and Ethernet to allow thinner cables and smaller connectors.

**HDBaseT 3.0**: Announced to support DisplayPort 4K/60 Hz uncompressed and up to 100 Gbps aggregate signalling for commercial installs. As of mid-2026, HDBaseT 3.0 products are emerging in commercial AV.

### 5.2 Linux Support Status

HDBaseT extenders present to the Linux system as a **network display device** at the protocol level: the transmitter converts HDMI or DisplayPort signals to HDBaseT, and the receiver converts back. There is no kernel KMS/DRM driver for HDBaseT because the conversion happens entirely in the extender hardware; the GPU sees a standard HDMI or DP sink at the transmitter end.

The practical consequence: a system connecting to an HDBaseT extender's HDMI input drives it exactly as it would drive any other HDMI 2.0 display. The extender's distance extension, audio embedding, and control-signal pass-through are transparent to the kernel.

Some HDBaseT chipsets (e.g. Valens Semiconductor VA6000 series) appear on I2C buses for EDID pass-through and HDCP re-authentication; these are generally handled by the HDMI bridge driver infrastructure in `drivers/gpu/drm/bridge/`. However, there is no generic upstream kernel HDBaseT driver as of Linux 6.15, and commercial HDBaseT controller drivers are typically provided out-of-tree by the vendor.

---

## 6. VESA Adaptive-Sync and DisplayHDR Certification

### 6.1 Adaptive-Sync and Adaptive-Sync Display 1.1a

VESA's **Adaptive-Sync** is the base DisplayPort mechanism that allows the source to vary the pixel clock and vertical blanking period on a per-frame basis, enabling displays to track the GPU's instantaneous frame rate. The mechanism is defined in the DisplayPort specification's VRR (Variable Refresh Rate) extension, governed by DPCD register `DP_EDP_CONFIGURATION_SET`.

VESA launched its **AdaptiveSync Display** certification programme (separate from the base Adaptive-Sync protocol) in 2021, with **version 1.1** updated in January 2024 to **AdaptiveSync Display 1.1a**. Key requirements for 1.1a certification:

- Minimum certified refresh rate: **144 Hz**
- Minimum VRR range floor: **60 Hz** (the display must track GPU frame rate from at least 60 Hz to its maximum, with no flicker or judder below that floor)
- Extended GtG (Grey-to-Grey) test matrix: **9×9** combinations (vs 5×5 in v1.0) with tightened overshoot/undershoot tolerances
- **Dual-Mode support** (new in 1.1a): a display can be certified at two resolution/refresh combinations (e.g. 4K/144 Hz and 1080p/280 Hz) when it uses resolution-dependent refresh rate scaling
- **Overclocked mode** certification: manufacturers can certify modes above the panel's factory default

[Source: VESA AdaptiveSync Display 1.1a Press Release](https://vesa.org/featured-articles/vesa-updates-adaptive-sync-display-standard-with-new-dual-mode-support/)

**VESA MediaSync** is a companion standard for low-frame-rate content (24 Hz film, 48 Hz HFR cinema), requiring smooth 1:1 frame delivery without blended frames.

### 6.2 DisplayHDR Certification Tiers

VESA's **DisplayHDR** programme certifies panels against objectively measured peak brightness, black level, colour gamut, and local dimming capability. As of version 1.2 (December 2024) the tiers are:

**LCD (transmissive) tiers:**

| Tier | Peak Brightness | Black Level | Local Dimming | Full-Screen Sustained |
|------|----------------|-------------|---------------|-----------------------|
| DisplayHDR 400 | 400 nits | 0.40 nits | Not required | 320 nits |
| DisplayHDR 600 | 600 nits | 0.10 nits | 1D (edge) required | 400 nits |
| DisplayHDR 1000 | 1000 nits | 0.05 nits | 2D (local) required | 600 nits |
| DisplayHDR 1400 | 1400 nits | 0.02 nits | 2D (local) required | 900 nits |

**Emissive (OLED, MicroLED) tiers:**

| Tier | Peak Brightness | Black Level |
|------|----------------|-------------|
| DisplayHDR True Black 400 | 400 nits | 0.0005 nits |
| DisplayHDR True Black 500 | 500 nits | 0.0005 nits |
| DisplayHDR True Black 600 | 600 nits | 0.0005 nits |

DisplayHDR 1400 also requires 95 % of the DCI-P3-D65 colour gamut, compared to 90 % for lower LCD tiers.

[Source: VESA DisplayHDR Performance Criteria](https://displayhdr.org/performance-criteria/) [Source: TFTCentral VESA DisplayHDR 1400 Analysis](https://tftcentral.co.uk/news/vesa-update-displayhdr-certification-requirements-including-new-hdr-1400-tier)

### 6.3 What the Kernel Enforces vs What the Panel Certifies

A critical distinction: **DisplayHDR and AdaptiveSync are panel-level certifications; the Linux DRM/KMS subsystem does not enforce them**.

The kernel's role is to:
1. **Negotiate HDR metadata** via the InfoFrame (HDMI) or VSC SDP (DP) — signalling to the display that the stream is HDR10, HLG, or Dolby Vision
2. **Enable VRR** via the connector's `VRR_CAPABLE` property and `CRTC_VRR_ENABLED` property
3. **Set peak luminance** in the `hdr_output_metadata` DRM property, which populates the HDMI HDR Static Metadata InfoFrame or the DP VSC SDP fields

The display firmware interprets these signals and activates its certified HDR processing pipeline independently. A DisplayHDR 1000 panel receiving a correctly-formed HDR10 stream will engage its local-dimming algorithm autonomously — the kernel does not drive that algorithm directly.

Application developers should use the Wayland `wp_color_management_v1` or equivalent protocols (covered in adjacent chapters) to pass tone-mapping intent from compositor to kernel, rather than assuming the kernel will enforce specific luminance targets.

---

## 7. Linux Kernel Support Status per Standard

### 7.1 DisplayPort UHBR in i915

Intel's i915 driver gained initial infrastructure for DP 2.0 128b/132b UHBR support in a patch series by Jani Nikula merged in 2021 (`drm/i915/dp: read sink UHBR rates`). Full 128b/132b link training — including FFE (Feed-Forward Equalisation) preset negotiation and the revised link training sequence — was progressively implemented through 2022–2024, targeting the Meteor Lake (MTL) and Battlemage (BMG) display engines.

[Source: Patchwork i915 UHBR sink rates patch](https://patchwork.kernel.org/project/intel-gfx/patch/089d807e887d308c52c84cf58dfb6777de18872d.1629735412.git.jani.nikula@intel.com/)

Key source files for UHBR support:

- [`drivers/gpu/drm/i915/display/intel_dp.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_dp.c): DPCD rate discovery, `intel_dp_is_uhbr()`, `intel_dp_set_dpcd_sink_rates()`
- [`drivers/gpu/drm/i915/display/intel_dp_link_training.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_dp_link_training.c): 128b/132b link training sequences including `intel_dp_get_lane_adjust_tx_ffe_preset()`, `intel_dp_128b132b_intra_hop()`, and `intel_dp_training_pattern()` (which selects TPS2 for UHBR vs TPS4 for HBR3)

The `intel_dp_is_uhbr()` function is central to the UHBR dispatch path:

```c
/* drivers/gpu/drm/i915/display/intel_dp.c */
bool intel_dp_is_uhbr(const struct intel_crtc_state *crtc_state)
{
    return drm_dp_is_uhbr_rate(crtc_state->port_clock);
}
```

Platform support as of Linux 6.x:

| Platform | Max UHBR Rate | Notes |
|----------|--------------|-------|
| Ice Lake (ICL) | UHBR13.5 | Partial; depends on firmware |
| Meteor Lake (MTL) | UHBR20 | Full 128b/132b SST DSC support |
| Battlemage (BMG) | UHBR13.5 | Per `intel_dp_set_dpcd_sink_rates()` |

> **Note:** The specific kernel versions in which each platform's UHBR capability was fully enabled should be verified against the kernel changelog at [kernel.org](https://kernel.org) for the relevant release. DP 2.1 SST DSC was targeted for Linux 6.15 (see Phoronix coverage of Intel Linux 6.15 graphics).

[Source: Phoronix Intel Linux 6.15 graphics](https://www.phoronix.com/news/Intel-Linux-6.15-Graphics-Start)

### 7.2 HDMI 2.1 FRL in amdgpu

AMD's AMDGPU driver gained initial HDMI 2.1 FRL support in a patch series submitted by Harry Wentland to DRM-Next on 4 June 2026, targeting **Linux 7.2**. The series includes:

- Initial FRL link training and rate negotiation
- DCN 3.x display engine integration (missing function pointers filled in)
- DSC over HDMI FRL support (submitted 11 May 2026, extending the FRL series)

FRL is **disabled by default** in the initial merge to prevent regressions on FRL-capable displays that were previously operating in TMDS mode. Users can enable it via:

```bash
# Enable AMDGPU HDMI 2.1 FRL (Linux 7.2+)
# /etc/kernel/cmdline or /etc/default/grub GRUB_CMDLINE_LINUX
amdgpu.dc_feature_mask=0x400
```

[Source: GamingOnLinux — AMD HDMI 2.1 FRL for Linux](https://www.gamingonlinux.com/2026/05/further-expanded-amd-hdmi-2-1-support-is-coming-to-linux-now-with-frl-and-dsc/)

For Intel: Intel's GPU driver also supports HDMI 2.1 FRL on recent discrete GPUs. NVIDIA's open-source `nova-drm` / `nouveau` path for HDMI 2.1 is in development; NVIDIA's proprietary driver handles HDMI 2.1 independently.

### 7.3 USB4 Display Tunnel in the Thunderbolt Driver

The Linux Thunderbolt driver (`drivers/thunderbolt/`) handles USB4 and Thunderbolt connection management. It supports both **firmware CM** (most Intel PC platforms with TBT3/4) and **software CM** (Apple systems, USB4 Gen 3/4 devices advertising only `user` security level).

For AMDGPU + USB4 DP tunnelling, a dedicated bring-up was submitted (approximately 2,000 lines) to wire the GPU's DP output into the USB4 tunnel fabric rather than using a direct DP physical connection. The thunderbolt driver handles:

- Tunnel creation/teardown in response to `drm_connector` hotplug events
- Bandwidth allocation negotiation via the `TB_TUNNEL_DP` abstraction
- Notification of "low bandwidth" or "insufficient bandwidth" conditions to the DRM layer

[Source: Phoronix AMD USB4 DP Tunneling](https://www.phoronix.com/news/AMDGPU-USB4-DP-Tunneling)

### 7.4 Key Kernel Configuration Options

| Config Option | Purpose |
|--------------|---------|
| `CONFIG_DRM_I915` | Intel i915 driver; required for all Intel DP/HDMI features including UHBR |
| `CONFIG_DRM_AMDGPU` | AMD GPU driver; required for HDMI 2.1 FRL and USB4 DP tunnelling |
| `CONFIG_THUNDERBOLT` | Thunderbolt/USB4 controller driver and connection manager |
| `CONFIG_USB4` | USB4 fabric support (dependency of `CONFIG_THUNDERBOLT` on modern kernels) |
| `CONFIG_DRM_DP_AUX_CHARDEV` | Expose DPCD AUX channel as `/dev/drm_dp_aux*` for userspace debugging |
| `CONFIG_DRM_DISPLAY_CONNECTOR` | Generic display connector infrastructure used by bridge drivers |

---

## 8. Bandwidth Arithmetic: Worked Examples

The following calculations use uncompressed RGB 8-bit-per-component (3 bytes per pixel) unless noted. All figures assume 4 lanes unless lane count is specified explicitly.

### 8.1 4K/144 Hz: Does DP 1.4 Suffice?

Target: 3840 × 2160 pixels, 144 frames per second, 8 bits per component (RGB = 24 bits per pixel).

**Raw pixel bandwidth:**

```
3840 × 2160 × 144 Hz × 3 bytes = 3,583,180,800 bytes/s
                                = 28.66 Gbps
```

**With 8b/10b encoding overhead (DP 1.4):**

```
28.66 Gbps ÷ 0.80 = 35.83 Gbps required link capacity
```

DP 1.4 HBR3 provides 25.92 Gbps effective (32.4 Gbps raw × 80 %). **25.92 Gbps < 28.66 Gbps** — DP 1.4 cannot carry 4K/144 Hz uncompressed at 8 bits per pixel.

With **DSC at 2:1 ratio** (visually lossless):

```
28.66 Gbps ÷ 2 = 14.33 Gbps required
```

DP 1.4 HBR3 effective capacity (25.92 Gbps) is **comfortably above** 14.33 Gbps, so 4K/144 Hz over DP 1.4 is achievable with 2:1 DSC.

**DP 2.1 (UHBR10, 38.69 Gbps effective):** 38.69 Gbps > 28.66 Gbps → 4K/144 Hz **uncompressed** fits within a single UHBR10 link.

**HDMI 2.1 (FRL6, ~42.7 Gbps effective):** Also fits 4K/144 Hz uncompressed (42.7 Gbps > 28.66 Gbps).

### 8.2 5K/60 Hz Dual-Monitor

Target: Two 5K displays (5120 × 2880) at 60 Hz, each delivering a separate stream.

**Raw pixel bandwidth per display:**

```
5120 × 2880 × 60 Hz × 3 bytes = 2,654,208,000 bytes/s
                               = 21.23 Gbps per display
```

**With 8b/10b encoding:**

```
21.23 Gbps ÷ 0.80 = 26.54 Gbps per display required link capacity
Total for two displays: 53.08 Gbps
```

**With 128b/132b (DP 2.1):**

```
21.23 Gbps ÷ 0.967 = 21.96 Gbps per display
Total: 43.92 Gbps
```

**Analysis:**
- DP 1.4 MST (25.92 Gbps effective, single cable): Cannot carry two 5K/60 streams (53.08 Gbps > 25.92 Gbps). Even one stream barely fits.
- Thunderbolt 3/4 (40 Gbps USB4 shared): 40 Gbps < 43.92 Gbps — cannot carry two uncompressed 5K/60 Hz streams simultaneously.
- **DP 2.1 UHBR20** (77.37 Gbps effective): 77.37 Gbps > 43.92 Gbps — **both 5K/60 Hz displays fit over a single UHBR20 link** via MST.
- **Thunderbolt 5 Bandwidth Boost** (120 Gbps): Easily fits two 5K/60 Hz streams.

### 8.3 8K/60 Hz: Where Only DP 2.1 and HDMI 2.1 Qualify

Target: 7680 × 4320 (8K) at 60 Hz, 10 bits per component (HDR), 4:4:4 chroma.

**Raw pixel bandwidth:**

```
7680 × 4320 × 60 Hz × 30 bits × (1/8 bytes/bit)
= 7680 × 4320 × 60 × 3.75 bytes
= 7,464,960,000 bytes/s
= 59.72 Gbps
```

**Interface comparison:**

| Interface | Effective Capacity | 8K/60 10b 4:4:4? |
|-----------|-------------------|------------------|
| DP 1.4 HBR3 | 25.92 Gbps | **No** (need 59.72 Gbps) |
| HDMI 2.0 | 14.4 Gbps | **No** |
| HDMI 2.1 FRL6 | 42.7 Gbps | **No** (59.72 > 42.7) |
| HDMI 2.1 FRL6 + DSC 2:1 | ~85.4 Gbps equivalent | **Yes** (29.86 Gbps with 2:1) |
| DP 2.1 UHBR20 | 77.37 Gbps | **Yes** (59.72 < 77.37) |
| USB4 v2 asymmetric | ~116 Gbps | **Yes** |

Note that **HDMI 2.1 FRL6 cannot carry 8K/60 Hz 4:4:4 10-bit uncompressed** — it requires DSC at approximately 1.4:1 or higher, or fall-back to 4:2:0 chroma subsampling. **DP 2.1 UHBR20 is the only single-cable uncompressed 8K/60 10-bit solution at the current specification ceiling.**

8K/60 Hz in 4:2:0 (chroma-subsampled):

```
7680 × 4320 × 60 Hz × 20 bits × (1/8)
= 7680 × 4320 × 60 × 2.5 bytes
= 4,976,640,000 bytes/s
= 39.81 Gbps
```

HDMI 2.1 FRL6 effective capacity (42.7 Gbps) > 39.81 Gbps → **8K/60 Hz in 4:2:0 fits over HDMI 2.1 without DSC**.

### 8.4 DSC Compression Ratio Math

DSC 1.2a supports compression ratios from **1.5:1 up to 3:1** for visually lossless output. Higher ratios (up to 6:1 for 8K) are defined in the specification but introduce visible artefacts in rapidly changing content.

**Effective bandwidth with DSC:**

```
Required_BW_with_DSC = Uncompressed_BW / Compression_Ratio
```

Example — 8K/60 Hz 4:4:4 10-bit over DP 1.4 HBR3:

```
Uncompressed:    59.72 Gbps
With 3:1 DSC:    59.72 / 3 = 19.91 Gbps
DP 1.4 effective: 25.92 Gbps

25.92 > 19.91 → DP 1.4 + 3:1 DSC achieves 8K/60 10b 4:4:4
```

This is the basis for VESA's claim that DP 1.4 + DSC can drive 8K monitors — but it requires the display's DSC decoder to support the full 3:1 ratio at 8K slice dimensions.

For **DP 2.1 UHBR20 with DSC 3:1**, the theoretical maximum increases:

```
77.37 Gbps × 3 = 232.1 Gbps equivalent uncompressed
```

This headroom accommodates 16K/60 Hz in 4:4:4 10-bit (≈ 238 Gbps uncompressed), which VESA specifically calls out as the ceiling scenario for UHBR20 + DSC.

**DSC slice anatomy** (relevant for driver developers):

DSC divides the image into horizontal slice rows. For a 3840×2160 display, a common configuration is 2 slices of 1920×2160 (2 parallel DSC codec instances). The DPCD `DSC_SLICE_COUNT` and `DSC_MAX_SLICE_WIDTH` registers in the sink constrain allowable configurations. The DRM DSC helper (`drivers/gpu/drm/display/drm_dsc_helper.c`) provides `drm_dsc_compute_rc_parameters()` to compute the rate control parameters for a given slice geometry and compression ratio.

---

## Integrations

- **Ch2 — KMS Display Pipeline**: The `drm_connector`, `drm_encoder`, and `drm_crtc` abstractions that manage the physical DP/HDMI/USB4 link at the kernel level. The `hdr_output_metadata` connector property described in §6.3 is defined here. See also how `drm_mode_valid()` enforces bandwidth constraints.

- **Ch128 — DisplayPort MST and Multi-Monitor Topology**: MST is the primary mechanism for driving multiple displays over a single DP cable or USB4 DP tunnel, building directly on the link rates and bandwidth arithmetic in §1 and §8.

- **Ch140 — HDMI and DisplayPort Audio on Linux**: eARC (§2.4) provides the high-bandwidth audio return path; the ALSA/ASoC side of that 37 Mbps channel and its ACR (Audio Clock Regeneration) interaction with FRL frame rates are covered there.

- **Ch172 — eGPU on Linux — Thunderbolt, USB4, and PCIe Hot-Plug**: The Thunderbolt 3/4/5 host-side infrastructure described in §4 is the physical layer over which eGPUs connect. Bandwidth partitioning between the eGPU's PCIe tunnel and any DP tunnels on the same Thunderbolt link is a critical design constraint.

- **Ch182 — Connectors and Physical Layer**: The mechanical and electrical specifications of USB-C, DisplayPort, HDMI, and Mini-HDMI connectors; impedance matching, ESD protection, and cable assembly requirements for DP80 and HDMI Ultra-High-Speed cable certification.

- **Ch183 — EDID and DisplayID — Capability Reporting**: The negotiation path by which a connected display communicates its supported resolutions, refresh rates, DSC capability, FRL capability (via an HDMI Sink Capability Data Structure in the EDID), and HDR metadata support back to the GPU driver.

- **Ch184 — eDP — The Laptop-Internal Variant**: Embedded DisplayPort uses the same DPCD/MST/PSR2/Panel Replay architecture as external DP, but with simplified connector requirements, different HPD signalling, and specific power-sequencing constraints. UHBR rates are beginning to appear in eDP 1.5 for high-refresh-rate laptop panels.

- **Ch186 — Pixel Formats and Signal Encoding — Bandwidth Consumption Per Format**: The per-format bandwidth costs referenced in §8 are derived from the RGB/YCbCr, bit-depth, and chroma subsampling combinations detailed in Ch186, together with the packing rules for HDMI and DP transport streams.

## Roadmap

### Near-term (6–12 months)
- AMDGPU HDMI 2.1 FRL support, merged targeting Linux 7.2, is expected to be enabled by default once the initial opt-in period confirms no regressions on FRL-capable displays that previously fell back to TMDS mode.
- Intel i915 DP 2.1 SST DSC on Meteor Lake and Battlemage (targeted for Linux 6.15–6.16) will enable 4K/144 Hz and 5K/60 Hz uncompressed over a single UHBR10 cable without requiring MST, simplifying single-monitor setups.
- VESA is expected to publish the **AdaptiveSync Display 1.2** revision, tightening the dual-mode certification matrix and adding explicit 240 Hz and 480 Hz profiling for high-refresh esports displays.
- The `nova-drm` Rust NVIDIA kernel driver (Ch10) is progressing towards basic modesetting support, with HDMI 2.1 FRL and DP 2.1 UHBR planned as follow-on features once the initial DCB (Device Control Block) parser and display engine enumeration land upstream.

### Medium-term (1–3 years)
- **DisplayPort 2.2** is anticipated from VESA, building on UHBR20 with additional PHY-layer improvements targeting reliable passive cable performance at 80 Gbps over longer runs, and potentially introducing a new UHBR25 (25 Gbps per lane, 100 Gbps × 4) tier to close the gap below USB4 v2 bandwidth.
- **HDMI 2.2** is in development at the HDMI Forum, targeting an increase beyond the current 48 Gbps FRL6 ceiling; early industry reports suggest 96 Gbps aggregate, which would enable 8K/120 Hz and 10K/60 Hz uncompressed without DSC.
- USB4 v2 asymmetric (120 Gbps) DP tunnelling support in the Linux Thunderbolt driver is expected to mature with the arrival of USB4 v2 hubs and docks, requiring Thunderbolt 5 firmware CM updates and new `TB_TUNNEL_DP` bandwidth allocation accounting in the kernel software CM.
- Panel Replay (§1.4) is expected to become the de-facto replacement for PSR2 across all DisplayPort 2.1 laptops, with the i915, amdgpu, and nouveau drivers all converging on a unified `drm_panel_replay` helper analogous to the existing `drm_dp_psr_helper.c` infrastructure.

### Long-term
- As MicroLED panels approach commercial volume production, the DisplayHDR True Black tier is expected to expand to a **True Black 1000** or higher tier, requiring the Linux `hdr_output_metadata` DRM property and compositor-side tone mapping to be extended to handle per-zone peak luminance metadata beyond the current 10,000 nit HDR10 static metadata ceiling.
- The convergence of display and compute interconnects — with PCIe 7.0, USB4 v3 (anticipated at 160+ Gbps), and CXL 4.0 sharing the same physical Type-C ecosystem — will likely require the Linux Thunderbolt/USB4 connection manager to implement more sophisticated quality-of-service arbitration between DP tunnel, PCIe tunnel, and CXL memory-semantic traffic on a shared fabric.
- HDBaseT 3.0's 100 Gbps aggregate capability may eventually attract an upstream Linux KMS bridge driver for commercial AV deployments, consolidating the currently out-of-tree Valens Semiconductor and other vendor bridge drivers into a generic `drm/bridge/hdbaset` framework.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
