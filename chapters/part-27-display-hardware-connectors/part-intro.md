# Part XXVII — Display Hardware, Connectors, and Signal Standards

The Linux graphics stack is often described in terms of its software layers — the DRM kernel subsystem, Mesa drivers, Wayland compositors, and application-facing APIs. But beneath every compositor frame is a physical signal travelling through a connector, a protocol negotiating display capabilities, and a hardware encoding standard that determines what the panel can show. Part XXVII addresses that hardware substrate. Where Part VI (The Display Stack) covers the *software* path from Wayland client to compositor to KMS, this part covers the *physical* path from KMS connector to display panel: the interface standards, connector pinouts, signal encoding formats, discovery protocols, and power management contracts that govern every display attached to a Linux system.

This part stands apart from Part VI in scope: its concern is not compositor protocols or colour management pipelines, but the hardware standards bodies (VESA, HDMI Forum, Wi-Fi Alliance, USB-IF) and the kernel subsystems that implement them. Driver authors will encounter this material when implementing connector hotplug, EDID parsing quirks, AUX channel transactions, or CEC physical address assignment. Laptop OEM developers will find the eDP, PSR, DRRS, and display power management chapters directly actionable. Multimedia and compositor engineers will use the pixel format and signal encoding chapter as a reference for why NV12 at limited range over HDMI requires a specific AVI InfoFrame configuration.

## Reader Groups

**Kernel display driver developers** — contributors to i915, amdgpu, nouveau, and embedded DRM drivers — will draw most heavily on Ch181 (interface standards and bandwidth arithmetic), Ch182 (connector and HPD mechanics), Ch183 (EDID/DisplayID parsing in the kernel), Ch184 (eDP AUX transactions and PSR implementation), Ch187 (CEC subsystem driver authoring), and Ch188 (DC state power well management).

**Laptop OEM and embedded engineers** will focus on Ch184 (eDP power sequencing, PSR, DRRS, backlight control methods) and Ch188 (DPMS, DC5/DC6, Panel Replay, ACPI S0ix display requirements).

**Multimedia and compositor developers** will find Ch185 (Miracast and wireless display pipelines), Ch186 (pixel format taxonomy, quantization range, HDR InfoFrame encoding), Ch183 (EDID debugging tools), and Ch187 (CEC automation from userspace) most directly applicable.

## Chapters in This Part

**Chapter 181 — Modern Display Interface Standards** surveys the protocol evolution from DisplayPort 1.1 through DP 2.1 (UHBR20, 80 Gbps), from HDMI 1.0 through HDMI 2.1 (FRL, 48 Gbps, eARC), through USB4 Gen 3 display tunnelling and Thunderbolt 4/5. It provides bandwidth arithmetic worked examples for common resolutions, documents Linux kernel support status per standard, and covers VESA AdaptiveSync and DisplayHDR certification tiers.

**Chapter 182 — Digital Display Connectors and the Physical Layer** examines what sits between the GPU and the cable: USB-C DisplayPort Alternate Mode lane reconfiguration, Thunderbolt security levels and the Linux thunderbolt subsystem, HDMI/DP/DVI/VGA connector variants, HPD electrical signalling and the DRM hotplug interrupt path, and DDC/CI I²C monitor control. It is the physical-layer companion to Ch181's protocol-layer treatment.

**Chapter 183 — EDID and DisplayID: How Linux Discovers Display Capabilities** dissects the 128-byte EDID base block, CEA-861 extension (SVDs, SADs, HDR Static Metadata Block, HDMI VSDB), DisplayID 2.0, and the Linux kernel's `drm_edid.c` parsing pipeline. It covers the EDID quirks database, sysfs EDID override for headless servers, DDC/CI monitor control via ddcutil, HDCP sink capability in DPCD, and audio ELD population from CEA ADB blocks.

**Chapter 184 — Embedded DisplayPort (eDP) and Laptop Panel Management** covers the protocol differences between eDP and external DP, DPCD register space and AUX channel transactions, Panel Self-Refresh (PSR and PSR2), Panel Replay, Dynamic Refresh Rate Switching (DRRS), backlight control methods (PWM, I²C, AUX CABC, OLED), Display Stream Compression on laptop panels, panel power sequencing, and the `drm_panel` subsystem for embedded panel bring-up.

**Chapter 185 — Wireless Display Technologies on Linux** maps the three paradigms — Miracast/Wi-Fi Direct, WiGig 802.11ad/ay, and network display streaming — across their latency/bandwidth/range tradeoffs. It covers the Miracast RTSP control plane, wpa_supplicant P2P group formation, the miraclecast and gnome-network-displays implementations, 60 GHz WiGig display extension, wayvnc and xrdp for Wayland network display, and zero-copy VA-API encode pipelines for low-latency wireless display.

**Chapter 186 — Video Pixel Formats and Display Signal Encoding** is the reference chapter for raw pixel wire formats: YCbCr vs RGB, chroma subsampling (4:4:4/4:2:2/4:2:0), bit depth, quantization range (full vs limited swing), the DRM fourcc format taxonomy (`DRM_FORMAT_NV12`, `DRM_FORMAT_P010`, etc.), V4L2 pixel format correspondence, HDR metadata in HDMI InfoFrames and DP metadata packets, AVI InfoFrame colorimetry encoding, and a worked end-to-end example of 4K HDR10 video playback through the Linux KMS pipeline.

**Chapter 187 — HDMI CEC and the Linux CEC Subsystem** documents the Consumer Electronics Control protocol — physical/logical address assignment, message categories, frame timing — and the Linux kernel CEC framework in `drivers/media/cec/`. It covers hardware CEC driver authoring (Raspberry Pi vc4, Amlogic, Rockchip), the CEC notifier mechanism for decoupling HDMI and CEC drivers, the `/dev/cecN` userspace API, cec-ctl tooling, libcec integration with Kodi, CEC-over-DisplayPort tunnelling via DPCD registers, and home automation from userspace.

**Chapter 188 — Display Power States: DPMS, Panel Self-Refresh, and Display Idle Management** covers the software orchestration layer above the panel-level mechanisms described in Ch184. It traces the power hierarchy from brightness dimming through DRRS and PSR to Intel DC5/DC6 display C-states, AMD MALL/SubVP DRAM self-refresh, and ACPI S0ix display shutdown. It covers the compositor idle chain (ext-idle-notify-v1, wlroots, GNOME, KDE PowerDevil), power-profiles-daemon display integration, ALS-driven automatic brightness, and ACPI S0ix display requirements for modern standby.

## How the Chapters Interrelate

Chapters 181 and 182 form a pair: Ch181 covers *what* the standards define (bandwidth, features, versions), Ch182 covers *how* signals and connectors physically work (pin assignments, HPD, DDC). Ch183 (EDID/DisplayID) builds on both — it is how Linux reads the display's capabilities declared over the Ch182 DDC channel using the formats governed by the Ch181 standards.

Ch184 (eDP) is a specialisation of Ch182/183 for the laptop-internal case: a different AUX channel usage pattern (DPCD-based backlight, PSR registers), a dedicated power sequencing protocol, and power mechanisms (PSR, DRRS) that Ch188 orchestrates at the system level.

Ch185 (wireless display) sits orthogonally — it replaces the physical connector with a Wi-Fi or network medium and introduces a software encode-and-stream pipeline that draws on Ch26 (VA-API), Ch38 (PipeWire), and Ch58 (GStreamer).

Ch186 (pixel formats) is a cross-cutting reference used by nearly every other chapter: the signal encoding format is determined by Ch181's standard, carried over Ch182's connector, described in Ch183's EDID CEA blocks, scanned out as DRM fourcc codes from Ch184/Ch188 power-managed planes, streamed by Ch185's wireless encode pipeline, and controlled via Ch187's CEC automation.

## Prerequisites and What Comes Next

Readers should arrive with a working understanding of Part I (DRM/KMS: Chapters 1–4, specifically KMS atomic modesetting and the connector/CRTC model), Part V (Mesa GPU drivers, for understanding display driver architecture), and Part VI (the Wayland/compositor display software stack). Chapter 51 (GPU Power Management) is the companion to Ch188's display power treatment.

This part feeds forward into Part IX (Tooling and Contributing — debugging display pipelines with `drm_info`, `ddcutil`, cec-ctl, edid-decode), Part XIII (Video Streaming — pixel formats and VA-API encode connect Ch186 to the multimedia frameworks), and Part XX (AI/ML — display power constraints relevant to accelerator thermal budgets).

---

## Part Roadmap Summary

*Synthesised from the Roadmap sections of this part's chapters.*

### Near-term (6–12 months)

- **DP 2.1 and UHBR enablement across open-source GPU drivers:** Intel Xe2 (Battlemage) and AMD RDNA 4 UHBR10/UHBR20 link training via the `drm_dp_helper` 128b/132b path is landing incrementally; parallel work in `drivers/usb/typec/altmodes/displayport.c` extends the TCPM Alt Mode driver to negotiate UHBR signalling rates during `DP_Configure` VDM exchange, and the Thunderbolt subsystem is gaining USB4 v2.0 80 Gbps tunnel support to carry UHBR20 streams.
- **Panel Replay rollout on AMD and Intel APUs:** AMD RDNA 3/4 Panel Replay stabilisation is progressing through the amdgpu DC stack targeting kernels 6.11–6.12, while Intel Lunar Lake (Xe2) ships Panel Replay with Selective Update by default; both require updates to PSR state machines (`intel_psr.c`, amdgpu DC) for the revised eDP 1.5 VSC SDP CRC format.
- **EDID and DisplayID infrastructure modernisation:** The `drm_edid.c` migration from raw `struct edid *` to the opaque `struct drm_edid` wrapper is completing for remaining bridge-chip and HDMI drivers; DisplayID 2.1 block parsers (VRR capability, improved DSC negotiation) and CTA-861-I 240 Hz VRR/HDR10+ metadata support are expected to land.
- **HDR pipeline wiring in compositors:** Automatic HDR metadata propagation from EDID-parsed MaxCLL/MaxFALL into `drm_connector_state.hdr_output_metadata` is being completed in KWin and Mutter; the `hdr_output_metadata` UAPI is being extended to carry SMPTE ST 2094-10 dynamic metadata frames for HDR10+ displays; and the DRM `colorspace` property is gaining BT.2100 ICtCp entries.
- **CEC and wireless display tooling improvements:** The CEC subsystem gains debugfs error injection for CI compliance testing, cec-pin adaptive sampling against timer jitter, and broader `drm_dp_cec` tunnelling coverage; GNOME Network Displays Miracast-over-Infrastructure (MICE) and PipeWire timestamp fixes for wireless display pipelines are also stabilising.

### Medium-term (1–3 years)

- **Convergence on Panel Replay as the universal laptop power-save mechanism:** Panel Replay is expected to subsume PSR1/PSR2 across all new eDP panels certified after eDP 1.5; both Intel and AMD drivers will treat PSR1/PSR2 as legacy fallbacks; AMD SubVP/MALL is evolving toward driver-transparent DMUB firmware-managed "display hibernation" targeting sub-1 W display idle on 4K RDNA 5 APUs.
- **Unified display colour and format pipeline abstractions:** A `drm_colorop` pipeline object (prototyped by AMD and Red Hat) will let userspace compose CSC, 1D LUT, and 3D LUT operations per-plane; HDMI 2.1a Source-Based Tone Mapping (SBTM) requires new DRM connector properties; and DisplayID 2.x is expected to become the primary capability signalling mechanism, demoting the legacy EDID base block to a compatibility stub.
- **USB-C and physical connector stack unification:** Kernel work targets a unified sysfs representation of USB-C ports acting simultaneously as DP Alt Mode, USB4, and Thunderbolt connectors; HDMI 2.1 FRL 6 (48 Gbps) enablement in `amdgpu` and `nouveau` via updated `drm_scdc_helper.c` SCDC register handling is also in this window; DDC/CI over I2C-over-AUX (`drm_dp_aux`) is being formalised for USB-C-attached monitors.
- **Wireless display next generation and KMS uAPI extensions:** Wi-Fi Alliance WFD 3.0 (Wi-Fi 6/7, <20 ms, 4K/120 Hz Miracast) and AV1 hardware-encode adoption across Intel/AMD/NVIDIA will reshape wireless display pipelines; the KMS uAPI may gain `DRM_CRTC_PROP_SELF_REFRESH_MODE` for compositor PSR preference expression; OLED-specific `drm-oled-management-v1` and `systemd-logind` display-aware suspend inhibition are also targeted.

### Long-term

- **Rust adoption and unified cross-vendor display power backends:** Following Nova and drm-rs, Rust helper crates for `drm_dp_aux`, `drm_panel`, and PSR state machines are plausible long-term targets; convergence of Intel Display IPU, AMD DCN, and Qualcomm MDSS may produce a generic KMS display-power backend in `drivers/gpu/drm/display/` handling PSR/Panel Replay lifecycle across vendors without per-driver polling.
- **Next-generation interconnect and bandwidth standards:** DisplayPort 2.2 (UHBR25, improved passive cable reach), HDMI 2.2 (96 Gbps FRL, 8K/120 Hz uncompressed), and USB4 v3 (160+ Gbps) will require the Thunderbolt/USB4 connection manager to implement QoS arbitration between DP, PCIe, and CXL traffic on a shared fabric; optical active cables will require PHY abstraction layers agnostic to copper vs optical medium.
- **Supersession of legacy protocol layers:** VGA and DVI code paths are approaching removal as hardware support ends across all vendor drivers; DDC I²C may be supplanted by firmware-mediated capability exchange (ACPI `_DSM`, devicetree overlays) for USB4-tunnelled and optical DP; CEC itself may be extended or replaced by a higher-bandwidth in-band control channel as HDMI 2.2 matures; and a unified display capability ABI merging EDID, DisplayID, DPCD, and DDC/CI into a single DRM property object has been discussed at XDC as a long-term replacement for the current piecemeal approach.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
