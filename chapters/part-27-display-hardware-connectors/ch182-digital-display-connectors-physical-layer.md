# Chapter 182: Digital Display Connectors and the Physical Layer

This chapter targets three primary audiences. **Kernel display driver authors** need a precise picture of how connector enumeration, HPD interrupt handling, and DDC reads are wired together in the DRM subsystem. **Hardware engineers** debugging signal integrity failures at high link rates will find the electrical specifications and Linux link-training parameter controls they need. **Linux users troubleshooting multi-monitor setups** — particularly those dealing with USB-C docks, Thunderbolt eGPUs, or active adapters — will find the relevant sysfs knobs and kernel code paths explained in enough depth to understand what the system is actually doing.

---

## Table of Contents

1. [USB-C DisplayPort Alternate Mode](#1-usb-c-displayport-alternate-mode)
   - 1.1 [USB Power Delivery and Alt Mode Entry Sequence](#11-usb-power-delivery-and-alt-mode-entry-sequence)
   - 1.2 [Lane Reconfiguration and Pin Assignments](#12-lane-reconfiguration-and-pin-assignments)
   - 1.3 [SBU Pins and the AUX Channel](#13-sbu-pins-and-the-aux-channel)
   - 1.4 [TCPM in the Linux Kernel](#14-tcpm-in-the-linux-kernel)
2. [Thunderbolt 3/4: Same Connector, Different Stack](#2-thunderbolt-34-same-connector-different-stack)
   - 2.1 [ICM Firmware and Security Levels](#21-icm-firmware-and-security-levels)
   - 2.2 [Linux Thunderbolt Subsystem](#22-linux-thunderbolt-subsystem)
   - 2.3 [USB4 and Thunderbolt Coexistence](#23-usb4-and-thunderbolt-coexistence)
3. [Standard DisplayPort Connector Variants](#3-standard-displayport-connector-variants)
4. [HDMI Connector Variants and FRL](#4-hdmi-connector-variants-and-frl)
   - 4.1 [Type A, C, D, E Physical Differences](#41-type-a-c-d-e-physical-differences)
   - 4.2 [HDMI 2.1 Fixed Rate Link](#42-hdmi-21-fixed-rate-link)
5. [Legacy Connectors: DVI, VGA](#5-legacy-connectors-dvi-vga)
   - 5.1 [DVI Variants and TMDS Compatibility](#51-dvi-variants-and-tmds-compatibility)
   - 5.2 [VGA and the Analog DAC Block](#52-vga-and-the-analog-dac-block)
6. [Hot Plug Detect Electrical Signalling](#6-hot-plug-detect-electrical-signalling)
   - 6.1 [HPD Pin Assignments and Voltage Levels](#61-hpd-pin-assignments-and-voltage-levels)
   - 6.2 [DRM HPD Handling in the Kernel](#62-drm-hpd-handling-in-the-kernel)
7. [DDC: Display Data Channel](#7-ddc-display-data-channel)
   - 7.1 [I2C over HDMI, DVI, and VGA](#71-i2c-over-hdmi-dvi-and-vga)
   - 7.2 [I2C-over-AUX for DisplayPort](#72-i2c-over-aux-for-displayport)
   - 7.3 [DDC/CI Monitor Control](#73-ddcci-monitor-control)
8. [Connector State Machine in DRM](#8-connector-state-machine-in-drm)
   - 8.1 [drm_connector_state Lifecycle](#81-drm_connector_state-lifecycle)
   - 8.2 [Polling and Sysfs Interface](#82-polling-and-sysfs-interface)
9. [Signal Integrity at High Link Rates](#9-signal-integrity-at-high-link-rates)
   - 9.1 [Differential Impedance and Encoding](#91-differential-impedance-and-encoding)
   - 9.2 [Link Training Parameters in Linux](#92-link-training-parameters-in-linux)
   - 9.3 [Why Cables Fail at HBR3](#93-why-cables-fail-at-hbr3)
10. [Active vs Passive Adapters](#10-active-vs-passive-adapters)
11. [Integrations](#11-integrations)

---

## 1. USB-C DisplayPort Alternate Mode

### 1.1 USB Power Delivery and Alt Mode Entry Sequence

The USB Type-C physical connector carries up to 40 Gbps of aggregate signalling bandwidth across four SuperSpeed differential pairs (TX1, RX1, TX2, RX2), two Sideband Use (SBU) pins, a Configuration Channel (CC) pin, and two USB 2.0 differential pairs (D+/D−). DisplayPort Alternate Mode repurposes some or all of the SuperSpeed pairs to carry DisplayPort Main Link lanes, negotiating this reconfiguration over the CC line using the USB Power Delivery (PD) protocol.

The entry sequence proceeds through several clearly defined stages:

1. **PD Contract negotiation.** After cable insertion, the CC line determines cable orientation and port role (DFP/UFP). The Downstream Facing Port (DFP, typically a laptop or GPU) and Upstream Facing Port (UFP, typically a dock or display) exchange `Source_Capabilities` and `Request` messages to establish a PD contract covering voltage, current, and data roles.

2. **Structured VDM Discover Identity.** Once an explicit PD contract is in place, the DFP sends a `Discover_Identity` Structured Vendor Defined Message (SVDM) to the UFP to learn its capabilities. The UFP responds with an Identity VDO containing its product type and supported Standard IDs.

3. **Discover SVIDs.** The DFP issues `Discover_SVIDs` to obtain the list of Alternate Mode Standard IDs the UFP supports. DisplayPort Alt Mode uses SID `0xFF01` [Source](https://github.com/torvalds/linux/blob/master/include/linux/usb/typec_dp.h):

   ```c
   /* include/linux/usb/typec_dp.h */
   #define USB_TYPEC_DP_SID     0xff01
   #define USB_TYPEC_DP_MODE    1
   ```

4. **Discover Modes and Enter Mode.** The DFP issues `Discover_Modes` for SID `0xFF01`. The UFP responds with a Modes VDO describing its supported pin assignments, signalling capabilities (HBR3, UHBR10, etc.), and whether it is a cable or receptacle. The DFP then sends `Enter_Mode` for mode index 1 (`USB_TYPEC_DP_MODE`), and the UFP responds with an `Ack`.

5. **DP Status and Configure.** After Enter Mode acknowledgement, the DFP sends a `DP_Status_Update` to learn the current connection state (whether a display is actually attached), then sends `DP_Configure` with the chosen pin assignment and signalling rate to place the PHY into the correct lane configuration.

The whole exchange happens over the CC line at a data rate of roughly 300 kbps (BMC encoding). From a user's perspective it is invisible and completes within tens of milliseconds of cable insertion.

### 1.2 Lane Reconfiguration and Pin Assignments

The VESA DisplayPort Alternate Mode specification defines six pin assignments (A–F), of which A, B, and F were deprecated after revision 1.0b [Source](https://github.com/torvalds/linux/blob/master/include/linux/usb/typec_dp.h). The active assignments are:

| Assignment | DP Lanes | USB 3.2 SuperSpeed | Notes |
|---|---|---|---|
| C | 4 | None | Full-bandwidth DP, no USB 3.x data |
| D | 2 | 2 (one TX/RX pair each direction) | Multi-function: simultaneous USB 3.x + DP |
| E | 4 | None | Like C, but alternate orientation |

Pin assignments C and E dedicate all four SuperSpeed pairs to DisplayPort Main Link lanes, delivering the full 4-lane bandwidth (up to HBR3 32.4 Gbps raw or UHBR20 80 Gbps raw with DP Alt Mode 2.0). Pin assignment D delivers only 2 DP lanes (maximum HBR3 16.2 Gbps raw) but preserves two SuperSpeed pairs for USB 3.2 data, enabling simultaneous USB 3.x peripherals and display output through a single cable — the "multi-function" mode critical for docking stations.

USB 2.0 capability (D+/D−) is preserved in all assignments. CC and GND remain available for PD communication and power delivery regardless of the lane configuration chosen.

The kernel enum in `include/linux/usb/typec_dp.h` mirrors this:

```c
/* include/linux/usb/typec_dp.h — abridged */
enum typec_dp_pin_assignment {
    DP_PIN_ASSIGN_A,   /* deprecated after v1.0b */
    DP_PIN_ASSIGN_B,   /* deprecated after v1.0b */
    DP_PIN_ASSIGN_C,   /* 4-lane DP, no USB3 SS */
    DP_PIN_ASSIGN_D,   /* 2-lane DP + 2-lane USB3 SS */
    DP_PIN_ASSIGN_E,   /* 4-lane DP, no USB3 SS (alt orientation) */
    DP_PIN_ASSIGN_F,   /* deprecated after v1.0b */
    DP_PIN_ASSIGN_MAX,
};
```

### 1.3 SBU Pins and the AUX Channel

The two SBU (Sideband Use) pins in the USB-C connector carry the DisplayPort AUX channel differential pair (AUXP/AUXN). This is a critical architectural detail: in a standard full-size DisplayPort connector, the AUX channel occupies dedicated pins 13 (AUX CH+) and 11 (AUX CH−). In USB-C DP Alt Mode, those signals are re-routed to SBU1 and SBU2 [Source](https://www.usb.org/sites/default/files/D2T1-4%20-%20VESA%20DP%20Alt%20Mode%20over%20USB%20Type-C.pdf).

The AUX channel carries DPCD register access (EDID reads, link capability advertisement, link training handshake) and MST sideband messages. Because SBU pins are always available regardless of the USB3/DP lane split, the AUX channel is functional in both 2-lane (assignment D) and 4-lane (assignments C/E) modes.

> **Note on HDMI Alt Mode SBU usage:** When HDMI Alternate Mode is negotiated over USB-C (a separate VESA specification), the SBU pins carry the HEAC (HDMI Ethernet and Audio Return Channel) signal rather than the AUX channel. The eARC audio return channel in native HDMI 2.1 cables (not USB-C) uses a dedicated differential pair on pins 14 and the ARC-capable pin 19 shield, not SBU pins.

### 1.4 TCPM in the Linux Kernel

The Linux Type-C Port Manager (TCPM) implements the PD state machine described above. The core implementation lives in `drivers/usb/typec/tcpm/tcpm.c` [Source](https://github.com/torvalds/linux/blob/master/drivers/usb/typec/tcpm/tcpm.c), with hardware-specific TCPC (Type-C Port Controller) drivers sitting beneath it.

The TCPM state machine is defined via a `FOREACH_STATE` macro enumerating states including `DISCOVER_IDENTITY`, `DISCOVER_SVIDS`, `DISCOVER_MODES`, `DFP_TO_UFP_ENTER_MODE`, and `DFP_TO_UFP_EXIT_MODE`. Alt Mode discovery begins once the state machine reaches the `DISCOVERY` phase after explicit contract establishment.

The kernel exposes a clean abstraction layer for Alt Mode drivers. The DisplayPort Alt Mode driver at `drivers/usb/typec/altmodes/displayport.c` [Source](https://github.com/torvalds/linux/blob/master/drivers/usb/typec/altmodes/displayport.c) registers itself as a handler for SID `0xFF01` and receives callbacks when the TCPM completes mode entry. It then calls back into the GPU driver (via a notifier chain or direct callback) to trigger PHY reconfiguration.

The `struct pd_mode_data` in tcpm.c tracks discovered SVIDs (up to 16), the current SVID index during enumeration, and the negotiated revision. CC line state analysis is handled by inline helpers `tcpm_cc_is_sink()` and `tcpm_cc_is_source()`, which decode the CC voltage level to determine the attached cable type and port role.

---

## 2. Thunderbolt 3/4: Same Connector, Different Stack

Thunderbolt 3 and 4 share the USB-C physical connector form factor but use an entirely different protocol stack layered above the PHY. Where DisplayPort Alt Mode negotiates over PD VDMs, Thunderbolt uses an Intel-developed proprietary protocol (now standardised as the foundation of USB4) managed by an Integrated Connection Manager (ICM) running as firmware in the Thunderbolt controller.

### 2.1 ICM Firmware and Security Levels

The Thunderbolt controller contains an embedded processor running ICM firmware. When a TBT device is attached, the ICM negotiates the Thunderbolt tunnel (PCIe, USB 3.x, and DP) using TBT-specific control packets, completely independently of the USB PD state machine. The ICM enforces **security levels** to defend against DMA attacks over PCIe tunnels when IOMMU is not fully engaged:

| Level | Kernel name | Meaning |
|---|---|---|
| 0 | `none` | ICM auto-connects all devices (legacy mode) |
| 1 | `user` | User must approve each device via sysfs |
| 2 | `secure` | User approval plus challenge/response key verification |
| 3 | `dponly` | Only DisplayPort and USB tunnels; PCIe tunnelling disabled |

Additional levels `usbonly` and `nopcie` appear in newer silicon. The effective security level is set in BIOS/firmware and readable from the kernel [Source](https://docs.kernel.org/admin-guide/thunderbolt.html):

```bash
# Read the domain security level
cat /sys/bus/thunderbolt/devices/domain0/security
# → "user"
```

Authorization of a device at `user` level:

```bash
# Authorize a connected TBT device (creates PCIe tunnel)
echo 1 > /sys/bus/thunderbolt/devices/0-1/authorized
```

For `secure` level, a 32-byte random key must first be written:

```bash
key=$(openssl rand -hex 32)
echo "$key" > /sys/bus/thunderbolt/devices/0-3/key
echo 1 > /sys/bus/thunderbolt/devices/0-3/authorized
```

Writing `2` to `authorized` triggers a challenge-response verification against the stored key.

### 2.2 Linux Thunderbolt Subsystem

The kernel Thunderbolt subsystem lives in `drivers/thunderbolt/` [Source](https://github.com/torvalds/linux/blob/master/drivers/thunderbolt/). Key source files:

- `tb.c` — software connection manager (used when ICM is absent or disabled)
- `icm.c` — firmware (ICM) connection manager
- `switch.c` — `struct tb_switch` representing a Thunderbolt switch/device
- `port.c` — `struct tb_port` representing individual TBT ports
- `xdomain.c` — peer-to-peer XDomain protocol between TBT hosts
- `nhi.c` — Native Host Interface (PCIe-level DMA ring management)

`struct tb_switch` represents an attached Thunderbolt device or switch and contains fields including the device UUID, NVM version, safe-mode flag, and a pointer to the owning `struct tb` domain. `struct tb_port` represents an individual port on a switch and tracks PHY capabilities, USB4 capabilities (`cap_usb4`), and hop ID allocation tables (`in_hopids`, `out_hopids`).

The DisplayPort tunnel through Thunderbolt is transparent to the GPU driver: the Thunderbolt controller presents a DP sink to the GPU exactly as if it were a native connector. The kernel's `drivers/thunderbolt/dp_tunnel.c` [Source](https://github.com/torvalds/linux/blob/master/drivers/thunderbolt/dp_tunnel.c) manages bandwidth reservation and tunnel setup.

### 2.3 USB4 and Thunderbolt Coexistence

USB4 is the public specification derived from Thunderbolt 3, standardised by USB-IF. As stated in the kernel documentation: "USB4 is the public specification based on Thunderbolt 3 protocol with some differences at the register level" [Source](https://docs.kernel.org/admin-guide/thunderbolt.html). On modern silicon (Intel Tiger Lake and later, AMD Rembrandt and later), the same controller supports TBT3, TBT4, and USB4 simultaneously.

USB4 permits connection managers implemented in either firmware (as in TBT) or software (as in Apple's approach). Linux supports both. USB4 v2.0 doubles the link speed to 80 Gbps (DP 2.1 UHBR20 becomes achievable through a USB4 v2.0 tunnel), and the Linux kernel USB4 support tracks these capabilities through the `cap_usb4` field in `struct tb_port`.

The coexistence implication for display drivers: a GPU port may receive DP connections from native DP cables, DP Alt Mode over USB-C, or DP tunnelled through TBT/USB4. In all three cases the GPU's AUX hardware and DPCD register map are traversed identically. The Thunderbolt layer is transparent below the GPU's DP controller.

---

## 3. Standard DisplayPort Connector Variants

The VESA DisplayPort specification defines three mechanical connector variants for the standard 20-pin interface:

**Full-size DisplayPort (DP Type 1).** The standard connector measuring 16.1 mm × 4.8 mm with an asymmetric trapezoid profile. It carries 4 Main Link differential pairs (ML0–ML3), AUX CH+/−, HPD, CONFIG1/CONFIG2, a 3.3 V power rail (max 500 mA), and ground and shield connections. A latching clip (optional) prevents accidental disconnection. Full-size DP is the predominant connector on desktop GPUs and monitors.

**Mini DisplayPort.** A significantly smaller connector (7.5 mm × 4.6 mm) carrying the same 20 signals. Originally introduced by Apple in 2008 for thin laptops and widely adopted by Intel for NUC systems, Thunderbolt 1 and 2 ports, and numerous laptop vendors. Pin assignments are identical to full-size DP; passive adaptors between the two are entirely feasible. Mini-DP is largely being superseded by USB-C DP Alt Mode in thin-and-light designs.

**Micro DisplayPort.** A rarely deployed variant (4.6 mm × 2.8 mm), specified by VESA but appearing only on a small number of tablets and embedded devices. Full electrical compatibility with full-size and mini-DP via passive adaptors.

All three share the same electrical specifications. At HBR3 (8.1 Gbps per lane), connector and cable ESD protection circuitry must handle fast signal edges without degrading the 100 Ω differential impedance. TVS diode arrays used for ESD protection on connector pins must have low capacitance (typically under 0.5 pF per channel) or they will attenuate the high-frequency content of the Main Link signal.

The CONFIG1 and CONFIG2 pins (pins 17 and 18 for full-size DP, corresponding pins for mini/micro) are used for DP connector orientation detection on dual-head connectors and for Embedded DisplayPort (eDP) panel identification — they are not HPD. HPD is on pin 18 of the full-size DP plug (see Section 6).

---

## 4. HDMI Connector Variants and FRL

### 4.1 Type A, C, D, E Physical Differences

The HDMI specification defines four connector types, all carrying the same 19 logical signals but in different mechanical packages:

**Type A (Standard HDMI).** 13.9 mm × 4.45 mm, 19 pins in two rows. This is the universal connector on televisions, monitors, set-top boxes, and desktop GPU ports. The 19 pins carry: three TMDS differential data pairs (pins 1–9), a TMDS clock pair (pins 10–12), DDC SDA/SCL (pins 15/16), CEC (pin 13), the HPD line (pin 19), the +5V power supply (pin 18, max 55 mA), and a reserved pin (14, now used for ARC/eARC in HDMI 1.4+). Pin 17 is DDC/CEC ground.

**Type C (Mini HDMI).** 10.42 mm × 2.42 mm, 19 pins. Pin mapping differs from Type A: in Type A, pins 1/3 are TMDS Data 2 +/−, but in Type C the signal ordering changes. Passive Type A–to–Type C cables simply rewire the connector mechanically. Common on camcorders and older compact cameras.

**Type D (Micro HDMI).** 5.83 mm × 2.20 mm, 19 pins in a micro-connector. Carries the same 19 signals with a different pin numbering. Found on smartphones and compact devices where the larger connector would not fit.

**Type E (Automotive).** Adds a secondary locking mechanism (latching tab) to prevent vibration-induced disconnection in vehicle systems. Pins are the same 19 as Type A. Used in automotive head units and camera systems; not common in consumer computing.

Passive adaptors between Type A, C, and D are possible because all carry the same TMDS signals. Active conversion is never required for signal protocol reasons between these types (only for mechanical adaptation).

### 4.2 HDMI 2.1 Fixed Rate Link

Prior HDMI versions (up to 2.0) used TMDS (Transition-Minimised Differential Signalling) with three 10-bit encoded data channels plus a clock channel. HDMI 2.0 topped out at 18 Gbps aggregate bandwidth (approximately 14.4 Gbps after 8b/10b overhead), sufficient for 4K@60Hz 8-bit 4:4:4.

HDMI 2.1 introduces **Fixed Rate Link (FRL)**, a fundamentally different electrical and encoding scheme using the same Type A 19-pin connector [Source](https://www.graniteriverlabs.com/en-us/application-notes/technical-article-hdmi-2.1-fixed-rate-link-frl-mode-overview). FRL replaces the discrete clock channel with embedded clock recovery, uses 16b/18b encoding (88.9% efficiency vs 80% for 8b/10b TMDS), and runs on three or four differential lanes at speeds up to 12 Gbps per lane:

| FRL Rate | Lanes | Per-lane rate | Total raw | Net after 16b/18b |
|---|---|---|---|---|
| FRL 1 | 3 | 3 Gbps | 9 Gbps | ~8 Gbps |
| FRL 2 | 3 | 6 Gbps | 18 Gbps | ~16 Gbps |
| FRL 3 | 4 | 6 Gbps | 24 Gbps | ~21.3 Gbps |
| FRL 4 | 4 | 8 Gbps | 32 Gbps | ~28.4 Gbps |
| FRL 5 | 4 | 10 Gbps | 40 Gbps | ~35.6 Gbps |
| FRL 6 | 4 | 12 Gbps | 48 Gbps | ~42.6 Gbps |

FRL 6 enables 10K@120Hz or 4K@240Hz uncompressed. Crucially, source and sink negotiate the transition from TMDS to FRL via **FRL Link Training**, which occurs over the DDC channel (reading the sink's FRL capabilities from EDID/SCDB and then exchanging training status via I2C registers) before any video is transmitted [Source](https://www.graniteriverlabs.com/en-us/technical-blog/hdmi-earc-compliance-test). A source that does not support FRL falls back to TMDS — both modes are legal on the same Type A connector.

Because HDMI 2.1 FRL cables must handle up to 48 Gbps total while maintaining 100 Ω differential impedance and low skew between lanes, not all cables marketed as "HDMI 2.1" actually pass FRL certification. The HDMI Forum's Ultra High Speed certification program tests cables up to 48 Gbps.

---

## 5. Legacy Connectors: DVI, VGA

### 5.1 DVI Variants and TMDS Compatibility

Digital Visual Interface (DVI) was introduced in 1999 by the Digital Display Working Group and became the dominant digital display connector on desktop systems throughout the 2000s. The DVI connector family shares a common physical housing but differs in which pins are populated:

**DVI-D Single Link.** Uses one set of TMDS channels (3 data + 1 clock differential pair), supporting a pixel clock up to 165 MHz. Maximum practical resolution: 1920×1200@60 Hz (WUXGA). This was the standard for early flat panels.

**DVI-D Dual Link.** Adds a second complete TMDS channel set (total 6 data + 2 clock pairs, pins arranged alongside the single-link set in the same connector body). Pixel clock doubles to 330 MHz. Maximum practical resolution: 2560×1600@60 Hz (WQXGA), which was used by the 30-inch Apple Cinema Display and Dell UltraSharp 3007WFP.

**DVI-I.** "Integrated" — adds four analogue VGA pins (R, G, B, H-Sync, V-Sync) alongside the digital TMDS pins in a physically wider connector. Allows a single GPU port to drive either DVI-D digital or VGA analogue monitors via an appropriate adaptor. The analogue portion requires the GPU to include a DAC block.

**DVI-A.** Carries only the analogue VGA signals in a DVI housing — rarely seen in practice.

The TMDS signalling in DVI-D is electrically identical to HDMI TMDS for the video channels. Pins 1–9 on DVI-D Single Link (data channels 0–2) map directly to HDMI pins 1–9. A passive DVI-D to HDMI adaptor (or cable) is entirely valid for video — it requires no active circuitry [Source](https://en.wikipedia.org/wiki/Digital_Visual_Interface). The critical limitation is audio: HDMI carries audio multiplexed into the TMDS data blanking period using auxiliary audio data packets (ACP), while DVI has no audio provision. A passive DVI-to-HDMI connection therefore delivers video only; the monitor's HDMI audio input will receive no audio data.

Dual-link DVI to HDMI conversion cannot be accomplished passively. HDMI supports only a single TMDS clock domain, whereas dual-link DVI uses two TMDS clocks. An active converter chip is required to merge dual-link DVI into a single HDMI stream, and such devices are rare.

### 5.2 VGA and the Analog DAC Block

The HD-15 connector (commonly called VGA) carries three analogue colour signals (R, G, B) at 0–0.7 V (75 Ω terminated), composite sync or separate H-Sync/V-Sync TTL signals, DDC SDA/SCL for EDID, and a sense line. The GPU silicon implementing VGA requires an integrated multi-channel DAC block running at the pixel clock rate, consuming meaningful die area and power even when no VGA monitor is attached.

As display resolutions grew beyond VGA's practical bandwidth ceiling (approximately 2048×1536@85 Hz at 220 MHz pixel clock, constrained by cable and connector quality), GPU vendors began de-emphasising VGA support. AMD discontinued VGA outputs on discrete GPUs with the Radeon HD 7000 series (2012). Intel dropped VGA from integrated graphics outputs in several SoC generations. The Linux DRM kernel drivers for modern hardware similarly omit VGA encoder support — for example, the `amdgpu` driver has no `vga_encoder_helper_funcs` for current GFX IP generations.

Linux still supports legacy VGA through the `vgacon` (VGA console) and `fbcon` paths for pre-DRM framebuffer initialisation, but production display pipelines on modern hardware are entirely digital.

---

## 6. Hot Plug Detect Electrical Signalling

### 6.1 HPD Pin Assignments and Voltage Levels

Hot Plug Detect (HPD) provides a physical signal allowing the display to notify the source that it has been connected, disconnected, or that the topology has changed. The electrical assignment per connector type:

- **HDMI Type A:** Pin 19. The display drives this pin to nominally 5 V (source interpretation threshold: >2.0 V = connected, <0.8 V = disconnected) when it is powered and ready to present EDID.
- **DisplayPort (full-size):** Pin 18 (HPD). Driven by the sink to 3.3 V when connected.
- **DVI:** The DDC clock (pin 15) is sometimes used as a proxy HPD in systems that lack a dedicated HPD line, but DVI-I/DVI-D connectors have a dedicated HPD pin (pin 16 of the DVI 29-pin layout). Note: needs verification for specific DVI sub-type variants.
- **VGA:** No standardised HPD pin. Presence is typically detected by DDC/I2C bus pull-up or by sensing load on the DDC lines.

DisplayPort defines two HPD signalling semantics based on pulse duration [Source](https://www.vesa.org/vesa-standards/standards/displayport/):

| HPD event | Pulse duration | Meaning |
|---|---|---|
| Connect/disconnect | >2 ms (long pulse) | Sink attached or removed |
| Topology change notification | 0.5 ms – 1 ms (short pulse) | MST branch topology changed |
| IRQ_HPD (interrupt) | 0.5 ms – 1 ms | DPCD interrupt pending (e.g. link status change) |

This pulse-width multiplexing over a single pin allows MST hubs to signal topology changes (when a downstream display is added or removed) without physically disconnecting and reconnecting the HPD line.

HDMI uses HPD differently: there is no short-pulse protocol. Connect and disconnect are both signalled by level transitions. HDMI CEC and ARC/eARC have their own dedicated pins (CEC on pin 13, ARC on pin 14 via the HEAC pair).

### 6.2 DRM HPD Handling in the Kernel

The kernel DRM subsystem processes HPD events through a layered architecture. At the lowest level, the GPU driver's hardware interrupt handler fires when the HPD pin transitions. The driver calls `drm_kms_helper_hotplug_event()` [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_probe_helper.c):

```c
/* drivers/gpu/drm/drm_probe_helper.c */
void drm_kms_helper_hotplug_event(struct drm_device *dev)
{
    /* Fire off the uevent for userspace */
    drm_sysfs_hotplug_event(dev);
    if (dev->mode_config.funcs->output_poll_changed)
        dev->mode_config.funcs->output_poll_changed(dev);
}
```

This fires a uevent on the DRM device's sysfs node, which `udev`/`systemd-udevd` receives and relays to compositors via their own hotplug listeners. KWin, Mutter, and Sway all listen for `DRM_EVENT_HOTPLUG` on the DRM device file descriptor.

For connectors that do not produce HPD interrupts (some VGA, older DVI implementations, and embedded displays), the DRM polling thread provides a fallback. Initialised by `drm_kms_helper_poll_init()`, it runs `output_poll_execute()` on a workqueue at an interval driven by the connector's `DRM_CONNECTOR_POLL_CONNECT` and `DRM_CONNECTOR_POLL_DISCONNECT` flags. The work item calls `drm_helper_probe_detect()` → `detect_connector_status()` for each polled connector, comparing the new status against the cached value and calling `drm_kms_helper_hotplug_event()` on change.

The connector's detection callback is declared in `struct drm_connector_helper_funcs`:

```c
/* include/drm/drm_modeset_helper_vtables.h */
struct drm_connector_helper_funcs {
    ...
    int (*detect_ctx)(struct drm_connector *connector,
                      struct drm_modeset_acquire_ctx *ctx,
                      bool force);
    ...
};
```

The `detect_ctx` variant is preferred over the legacy `.detect` callback because it participates in the DRM modeset locking hierarchy, preventing deadlocks when detection is triggered concurrently with atomic commits.

---

## 7. DDC: Display Data Channel

### 7.1 I2C over HDMI, DVI, and VGA

The Display Data Channel (DDC) is an I2C bus running over the display cable, used primarily to read the monitor's EDID (Extended Display Identification Data) block. The I2C signals are:

- **HDMI Type A:** SDA on pin 16, SCL on pin 15.
- **DVI-D:** SDA on pin 7, SCL on pin 6 (29-pin DVI connector numbering).
- **VGA HD-15:** SDA on pin 12, SCL on pin 15.

The I2C bus runs at 100 kHz. The EDID block resides at I2C address `0x50`. Extension blocks (for CEA-861 audio/video data blocks, DisplayID, etc.) may be addressed at the same slave address using a segment pointer register at address `0x30`.

The kernel reads EDID via `drm_get_edid()` in `drivers/gpu/drm/drm_edid.c` [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_edid.c). This function drives an I2C adapter registered by the GPU driver and issues `i2c_transfer()` calls to read the 128-byte base EDID block and any extension blocks:

```c
/* drivers/gpu/drm/drm_edid.c — simplified illustrative flow */
struct edid *drm_get_edid(struct drm_connector *connector,
                           struct i2c_adapter *adapter)
{
    /* Read segment 0 from I2C address 0x50 */
    /* Validate checksum, read extensions at 0x50 with segment at 0x30 */
    return drm_do_get_edid(connector, drm_do_probe_ddc_edid, adapter);
}
```

GPU drivers that implement DDC must register an `i2c_adapter` struct whose `.master_xfer` callback drives the hardware I2C engine. The adapter is passed to `drm_get_edid()` and is also exposed to userspace via `/sys/class/drm/card0-HDMI-A-1/` for tools such as `edid-decode` and `ddcutil`.

### 7.2 I2C-over-AUX for DisplayPort

DisplayPort does not carry a separate I2C bus. Instead, EDID reads traverse the AUX channel using an I2C-over-AUX tunnelling protocol defined in the DisplayPort specification. The kernel implements this in `drivers/gpu/drm/display/drm_dp_helper.c` [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/display/drm_dp_helper.c) through the `struct drm_dp_aux` abstraction:

```c
/* include/drm/display/drm_dp_helper.h — abridged */
struct drm_dp_aux {
    const char *name;
    struct i2c_adapter ddc;      /* Exposes as an I2C adapter to drm_get_edid() */
    struct device *dev;
    struct drm_device *drm_dev;
    struct mutex hw_mutex;
    /* ... */
    ssize_t (*transfer)(struct drm_dp_aux *aux,
                        struct drm_dp_aux_msg *msg);
    int (*wait_hpd_asserted)(struct drm_dp_aux *aux,
                              unsigned long wait_us);
};
```

After calling `drm_dp_aux_register()` [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/display/drm_dp_helper.c), the `drm_dp_aux` appears as a standard `i2c_adapter` to the rest of the DRM subsystem. When `drm_get_edid()` issues I2C transactions, the I2C-over-AUX layer (`drm_dp_i2c_xfer()`) translates them into sequences of DP AUX MSG transactions with appropriate I2C START/STOP framing, re-tries on NACK, and defers to the `transfer` callback which calls the hardware AUX engine.

This design allows the same EDID-reading code path to work for both HDMI/DVI (real I2C) and DP (AUX tunnelled I2C) connectors with no change to the consumer.

### 7.3 DDC/CI Monitor Control

DDC/CI (Display Data Channel Command Interface), standardised by VESA in DDCCI 1.1 [Source](https://glenwing.github.io/docs/VESA-DDCCI-1.1.pdf), extends DDC to allow bidirectional control of monitor functions over the same I2C bus. The monitor listens on I2C address `0x37`. Commands use the VCP (Virtual Control Panel) code namespace:

| VCP Code | Function | Notes |
|---|---|---|
| `0x10` | Luminance (Brightness) | 0–100 typical range |
| `0x12` | Contrast | 0–100 typical range |
| `0x60` | Input Source Select | Values map to VGA, DVI, HDMI, DP inputs |
| `0xD6` | Power Mode | 1=on, 4=off, 5=hard off |
| `0x62` | Audio Speaker Volume | |
| `0xAC` | Horizontal Frequency | Read-only, diagnostic |

The `ddcutil` userspace tool [Source](https://www.ddcutil.com/) reads and writes VCP codes via the I2C adapter exposed by the GPU driver. Over DP, DDC/CI traffic traverses the AUX channel's I2C-over-AUX tunnel to reach the monitor's DDC/CI controller. Note that some monitors disable DDC/CI by default or implement it incompletely.

---

## 8. Connector State Machine in DRM

### 8.1 drm_connector_state Lifecycle

Each DRM connector has a `struct drm_connector_state` allocated per atomic state snapshot. The connector itself maintains a `status` field of type `enum drm_connector_status`:

```c
/* include/drm/drm_connector.h */
enum drm_connector_status {
    connector_status_connected = 1,
    connector_status_disconnected = 2,
    connector_status_unknown = 3,
};
```

Connectors are initialised to `connector_status_unknown`. The detection callbacks (`detect_ctx` or legacy `detect`) return a new status, which the probe helper commits after comparison with the previous value. Status changes trigger `drm_kms_helper_hotplug_event()`.

Drivers may force a connector to a specific status for debugging via `struct drm_connector_funcs.force`:

```c
/* Connector registration and enumeration */
struct drm_connector_list_iter iter;
struct drm_connector *connector;

drm_connector_list_iter_begin(dev, &iter);
drm_for_each_connector_iter(connector, &iter) {
    /* process connector */
}
drm_connector_list_iter_end(&iter);
```

The `drm_connector_list_iter` API [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_connector.c) provides safe concurrent iteration with proper reference counting. Raw list traversal without this iterator is not permitted in atomic context.

### 8.2 Polling and Sysfs Interface

The polling infrastructure is activated by `drm_kms_helper_poll_init()` [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_probe_helper.c) and runs `output_poll_execute()` as a delayed work item. Connectors opt into polling by setting `DRM_CONNECTOR_POLL_CONNECT` and/or `DRM_CONNECTOR_POLL_DISCONNECT` in their `connector->polled` field. HPD-capable connectors set `DRM_CONNECTOR_POLL_HPD` instead and rely on the interrupt path.

Users and compositors interact with connector state through sysfs:

```
/sys/class/drm/card0-HDMI-A-1/status       # "connected" or "disconnected" or "unknown"
/sys/class/drm/card0-HDMI-A-1/enabled      # "enabled" or "disabled"
/sys/class/drm/card0-HDMI-A-1/modes        # newline-separated list of available modes
/sys/class/drm/card0-HDMI-A-1/edid         # binary EDID blob
```

Writing `"detect"` to the `status` file triggers an immediate re-probe of that connector, useful for forcing EDID re-reads after a cable change. Writing `"connected"` or `"disconnected"` overrides the detected status (uses `drm_connector_funcs.force`), which aids debugging and EDID loading from `drm.edid_firmware` kernel parameter.

---

## 9. Signal Integrity at High Link Rates

### 9.1 Differential Impedance and Encoding

All modern display interfaces — DisplayPort Main Link, HDMI TMDS, HDMI FRL — use differential pair signalling with a nominal characteristic impedance of **100 Ω** (50 Ω single-ended to ground). Maintaining this impedance along the full signal path (GPU die → PCB traces → connector → cable → connector → monitor PCB) is critical to minimising reflections.

DisplayPort defines four nominal **voltage swing levels** for the transmitter, controlled via DPCD register `DP_TRAINING_LANE0_SET` (address `0x0103`):

| Level | Enum | Nominal differential output |
|---|---|---|
| 0 | `DP_TRAIN_VOLTAGE_SWING_LEVEL_0` | ~200 mV peak-to-peak (400 mVpp differential) |
| 1 | `DP_TRAIN_VOLTAGE_SWING_LEVEL_1` | ~400 mV (800 mVpp) |
| 2 | `DP_TRAIN_VOLTAGE_SWING_LEVEL_2` | ~600 mV (1200 mVpp) |
| 3 | `DP_TRAIN_VOLTAGE_SWING_LEVEL_3` | ~800 mV (1600 mVpp) |

The kernel defines these as bit-field values in `DP_TRAIN_VOLTAGE_SWING_LEVEL_x` constants in `include/drm/display/drm_dp.h` [Source](https://github.com/torvalds/linux/blob/master/include/drm/display/drm_dp.h). The mV values are per the VESA DisplayPort CTS (Compliance Test Specification); they are not embedded in the kernel header itself.

**Pre-emphasis** compensates for high-frequency losses in the cable by boosting the signal amplitude at higher frequencies. Four levels are defined, with nominal de-emphasis values [Source](https://www.vesa.org/vesa-standards/standards/displayport/):

| Level | Enum | Nominal pre-emphasis |
|---|---|---|
| 0 | `DP_TRAIN_PRE_EMPH_LEVEL_0` | 0 dB |
| 1 | `DP_TRAIN_PRE_EMPH_LEVEL_1` | 3.5 dB |
| 2 | `DP_TRAIN_PRE_EMPH_LEVEL_2` | 6 dB |
| 3 | `DP_TRAIN_PRE_EMPH_LEVEL_3` | 9.5 dB |

Not all combinations of voltage swing and pre-emphasis are valid. Higher pre-emphasis requires a higher base voltage swing; level 3 pre-emphasis is only available at level 0 voltage swing (specific rules are defined in the DP specification). The kernel's link training code enforces these constraints when converting DPCD-requested levels into PHY register writes.

**Spread-spectrum clocking (SSC)** is optionally applied to the DisplayPort reference clock: a down-spread of ±0.5% (centred at −0.25%) reduces peak electromagnetic emissions by distributing energy across a frequency band. Both source and sink must agree to use SSC; it is negotiated via DPCD `DP_DOWNSPREAD_CTRL` (address `0x0107`).

### 9.2 Link Training Parameters in Linux

The DisplayPort link training sequence occupies two phases: Clock Recovery (CR) and Channel Equalisation (EQ). After each training step, the source reads back lane status registers and requests a new voltage/pre-emphasis setting from the sink. The kernel helper functions encapsulate the mandatory wait times between training steps:

```c
/* drivers/gpu/drm/display/drm_dp_helper.c */

void drm_dp_link_train_clock_recovery_delay(const struct drm_dp_aux *aux,
                                             const u8 dpcd[DP_RECEIVER_CAP_SIZE]);

void drm_dp_link_train_channel_eq_delay(const struct drm_dp_aux *aux,
                                         const u8 dpcd[DP_RECEIVER_CAP_SIZE]);
```

These functions read the sink's DPCD to determine whether the panel's training delay requirements exceed the specification minimums (100 µs for CR, 400 µs for EQ in most cases) and sleep for the appropriate duration. Driver authors must call these between training iterations rather than using fixed `udelay()` values, because some displays — particularly eDP panels in laptops — advertise longer mandatory delays.

Individual GPU drivers implement PHY-level link parameter application in their own AUX `.transfer` callback chains. For example, the `i915` driver applies voltage/pre-emphasis settings to the DDI PHY through `intel_dp_set_signal_levels()`, and `amdgpu` applies them through `dc_link_set_drive_settings()`.

### 9.3 Why Cables Fail at HBR3

At HBR3 (8.1 Gbps/lane), the Nyquist frequency of the 8b/10b-encoded NRZ signal is 4.05 GHz per lane. A cable run with significant insertion loss at 4 GHz will cause the eye diagram at the receiver to close — the bit transitions become ambiguous. The chief culprits are:

- **Excessive cable length.** A 1 m passive DisplayPort cable typically passes HBR3; a 3 m cable of the same gauge may not without a re-driver chip.
- **Poor dielectric.** Low-cost PE (polyethylene) insulation has higher dielectric loss than PTFE, increasing insertion loss per metre.
- **Impedance discontinuities.** Cheap connectors with un-controlled impedance create reflections that add to insertion loss at resonant frequencies.
- **Inadequate shielding.** At GHz frequencies, imperfect shields permit ingress from co-located USB 3.x or other cable emissions, degrading SNR.

VESA's DisplayPort certified cable programme tests passive cables at HBR3 to defined eye mask specifications. UHBR10 (DP 2.0, 10 Gbps/lane with 128b/132b encoding) requires even tighter specifications or active (re-driver/re-timer) cables. Active optical cables (AOC) achieve long runs at UHBR20 by converting to optical fibre internally.

From the Linux kernel's perspective, cable quality is invisible during link training: the hardware simply requests increasing voltage swing and pre-emphasis levels until the EQ phase succeeds. If the cable is so poor that training fails even at maximum voltage and pre-emphasis, the link falls back to a lower link rate (e.g. HBR2 or HBR) and the driver logs a message such as `DP link training failed at HBR3, retrying at HBR2`.

---

## 10. Active vs Passive Adapters

Understanding the capabilities and limitations of display adapters is critical for both driver authors and end users configuring unusual display chains.

**Passive DP-to-HDMI adapters** work by connecting the DisplayPort single-link TMDS-compatible differential pairs directly to the HDMI connector pins. This is electrically valid because single-link DVI and HDMI use the same TMDS signal encoding. However, DP sources must be configured in "DP++ mode" (dual-mode DP, now called DP++ [Source](https://www.vesa.org/vesa-standards/standards/displayport/)) to output TMDS signals on the Main Link pins. DP++ capability is signalled via DPCD register `DP_DUAL_LINK_CAP` and via the HDMI dongle's 1-kΩ pull-down on the HPD line. Passive DP-to-HDMI adapters are limited to HDMI 1.4 (4K@30 Hz) because they are constrained by single-link TMDS bandwidth (~165 MHz pixel clock).

**Active DP-to-HDMI 2.0 adapters** contain a TMDS encoder chip that receives the full 4-lane DP Main Link signal, decodes it internally, and re-encodes it as HDMI 2.0 TMDS at up to 600 MHz pixel clock, enabling 4K@60Hz. From the GPU's perspective the adapter appears as a DP sink (DPCD is present); the adapter's chip handles all TMDS generation internally.

**DP-to-HDMI 2.1 adapters** (per the VESA DP-to-HDMI 2.1 cable specification) use a DP 2.1 UHBR10 Main Link as the input and contain a chip that converts to HDMI 2.1 FRL output. This enables the full 48 Gbps HDMI 2.1 bandwidth from a DP 2.1 source. The adapter appears to the GPU as a DP 2.1 UHBR10 sink.

The kernel can identify some adapter types by reading the Branch Device ID in DPCD at offset `0x0500–0x0505`. Adapter OUI values and device strings allow driver heuristics to work around adapter-specific bugs.

**Active DP-to-VGA adapters** contain a DAC chip. The GPU sources a DP signal at a resolution compatible with VGA; the adapter decodes the DP stream and outputs analogue R/G/B at the appropriate frequency. These allow VGA monitors to be driven from DP-only GPU ports that lack an onboard DAC.

When an active adapter is present, the Linux DRM driver's connector type reported to userspace will typically still reflect the sink type (HDMI or VGA) as detected via EDID, not the cable type. The adapter is largely invisible to the software stack except in failure modes (e.g. link training marginal conditions introduced by the adapter's PHY).

---

## 11. Integrations

This chapter covers the physical connectors and electrical signalling beneath the display stack. It connects to adjacent chapters throughout the book:

- **Ch1 (DRM Driver Architecture):** Connector registration via `drm_connector_init()` and `drm_connector_register()` is part of the fundamental driver model described there. The connector types (`DRM_MODE_CONNECTOR_HDMIA`, `DRM_MODE_CONNECTOR_DisplayPort`, etc.) used in this chapter are defined in the DRM uAPI.

- **Ch2 (KMS Display Pipeline):** The connector is the sink-facing end of the KMS encoder/CRTC/connector pipeline. CRTC and encoder selection during atomic commit relies on connector status and EDID mode lists populated via the DDC path described in Section 7.

- **Ch128 (DisplayPort MST):** MST topology enumeration uses the AUX sideband channel introduced in Section 1.3 and Section 7.2. Short HPD pulses (Section 6.1) are the signalling mechanism for MST topology change notifications. The `drm_dp_aux` struct and `drm_dp_aux_register()` function used in Ch128 are introduced here.

- **Ch140 (HDMI and DisplayPort Audio):** Audio Return Channel (ARC) in HDMI 1.4 uses pin 14 of the Type A connector (the HEAC pair). Enhanced ARC (eARC) in HDMI 2.1 uses the same HEAC differential pair but at higher bandwidth. The USB-C HDMI Alt Mode SBU pins carry the HEAC channel when HDMI Alternate Mode (not DP Alt Mode) is used over USB-C. The HDMI connector pinout described in Section 4.1 is the physical foundation for the audio paths Ch140 covers.

- **Ch172 (eGPU over Thunderbolt):** The Thunderbolt security levels (Section 2.1) and the Linux `drivers/thunderbolt/` subsystem (Section 2.2) are the same infrastructure used for eGPU connection authorisation. The DP tunnel transparency discussed in Section 2.3 is what allows the eGPU's display outputs to appear as native DRM connectors.

- **Ch181 (Display Interface Standards):** Ch181 covers the protocol-level specifications (TMDS encoding, DP packetisation, FRL frame structure) that ride on top of the physical connectors described here. The two chapters are complementary: this chapter covers the pins, impedances, and HPD signalling; Ch181 covers what is sent over those pins.

- **Ch183 (EDID and Display Capability Negotiation):** The DDC I2C bus (Section 7.1) and DP AUX I2C tunnel (Section 7.2) introduced here are the physical transport over which EDID is read. Ch183 describes the EDID binary structure, the Extended Display Identification Data parsing in `drm_edid.c`, and how the kernel translates EDID timing descriptors into `drm_display_mode` structs.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*

## Roadmap

### Near-term (6–12 months)
- **DP 2.1 UHBR adoption in mainline GPU drivers:** Intel Xe2 (Battlemage) and AMD RDNA 4 both support UHBR10/UHBR20; kernel patches enabling full UHBR link training via the `drm_dp_helper` 128b/132b path are landing incrementally, with full enablement expected within the next kernel release cycle.
- **USB4 v2.0 DisplayPort tunnelling:** The Linux Thunderbolt subsystem (`drivers/thunderbolt/`) is gaining support for USB4 v2.0 80 Gbps tunnels, which are required to carry UHBR20 DP 2.1 streams; patches from Intel and the USB4 community are actively under review on the `linux-usb` mailing list.
- **TCPM Alt Mode rework for DP 2.1 UHBR pin assignments:** The `drivers/usb/typec/altmodes/displayport.c` driver is being extended to negotiate UHBR signalling rates during the `DP_Configure` VDM exchange; this requires coordinating PHY rate selection between the TCPM and GPU display controller drivers.
- **Improved HPD debounce and IRQ coalescing:** Several mailing-list patches address spurious HPD event storms from low-quality cables and docks by adding configurable debounce delays in `drm_probe_helper.c`, reducing compositor-level reconnect flicker.

### Medium-term (1–3 years)
- **HDMI 2.1 FRL enablement across open-source drivers:** While FRL is specified, open-source driver support (particularly in `amdgpu` and `nouveau`) lags behind TMDS support; the medium-term trajectory is full FRL 6 (48 Gbps) enablement in DRM, requiring new link-training sequences over DDC and updated SCDC register handling in `drm_scdc_helper.c`.
- **DisplayPort 2.1 DP-to-HDMI 2.1 active cable spec adoption:** As VESA's DP-to-HDMI 2.1 cable standard matures, kernel heuristics for identifying these adapters via DPCD OUI and properly exposing UHBR→FRL bandwidth to userspace will need formalisation in the connector detection path.
- **USB-C connector class unification:** Ongoing kernel work aims to unify the sysfs representation of USB-C ports that may simultaneously act as DP Alt Mode, USB4, and Thunderbolt connectors, giving userspace (and compositors) a single consistent connector model rather than separate `drm` and `thunderbolt` sysfs trees.
- **DDC/CI standardisation for USB-C-attached monitors:** As USB-C becomes the dominant display connection, reliable DDC/CI access over I2C-over-AUX (rather than native I2C) is increasingly important; work on improving `ddcutil`'s DP AUX path and exposing `/dev/i2c-*` interfaces from `drm_dp_aux` is in progress.

### Long-term
- **Optical connector dominance for high-bandwidth runs:** As UHBR20 (80 Gbps) and future higher-rate standards exceed practical passive copper cable reach at 2+ metres, active optical cables (AOC) with SFP-style or integrated optics are expected to become standard for data-centre and professional AV installations, requiring kernel PHY abstraction layers that are agnostic to the copper vs optical medium.
- **Consolidation of legacy connector support:** VGA DAC removal from GPU silicon is already complete in modern discrete GPUs; DVI is similarly being dropped from new monitor and GPU designs. The long-term trajectory for the Linux DRM stack is removal of vgacon/DVI-specific code paths and the analogue encoder infrastructure as hardware support ends across all vendor drivers.
- **Standardised connector health telemetry:** Future display interface revisions (DP 3.x and beyond) are expected to incorporate in-band link-quality reporting beyond the existing voltage/pre-emphasis feedback, enabling kernel-level predictive cable failure detection and automated adaptation without user-visible link retraining disruption.
