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
