# Chapter 184: Embedded DisplayPort (eDP) and Laptop Panel Management

**Target audiences**: Laptop OEM kernel developers porting new panel firmware to the i915 or amdgpu display stack; systems developers implementing display drivers for embedded hardware where panels are fixed at manufacture; power management engineers tuning PSR, DRRS, and Panel Replay to hit laptop battery-life targets; contributors to Intel i915 and AMD display core (DC) who need a map of the eDP-specific code paths.

---

## Table of Contents

1. [Introduction](#introduction)
2. [eDP vs. External DisplayPort: Why a Separate Spec](#edp-vs-external-displayport-why-a-separate-spec)
3. [DPCD Register Space and AUX Channel Protocol](#dpcd-register-space-and-aux-channel-protocol)
4. [PSR: Panel Self-Refresh](#psr-panel-self-refresh)
5. [PSR2: Selective Update](#psr2-selective-update)
6. [Panel Replay (eDP 1.5 / DP 2.1)](#panel-replay-edp-15--dp-21)
7. [DRRS: Dynamic Refresh Rate Switching](#drrs-dynamic-refresh-rate-switching)
8. [Backlight Control Methods](#backlight-control-methods)
9. [DSC on eDP: Display Stream Compression](#dsc-on-edp-display-stream-compression)
10. [Panel Power Sequencing: T1–T12 Timing](#panel-power-sequencing-t1t12-timing)
11. [Touch Panels, drm\_panel, and Bridge Chips](#touch-panels-drm_panel-and-bridge-chips)
12. [Debugging Tools and sysfs/debugfs Interfaces](#debugging-tools-and-sysfsdebugfs-interfaces)
13. [Integrations](#integrations)

---

## Introduction

The Embedded DisplayPort (eDP) specification is a sibling to DisplayPort designed for the internal link between a laptop's GPU and its built-in LCD panel. While eDP reuses the DisplayPort PHY, link layer, and AUX channel, it adds a significant set of laptop-specific extensions: dedicated power-sequencing handshakes, backlight control registers, panel self-refresh with local frame buffering, selective update for partial-region refresh, and dynamic refresh rate switching. These extensions collectively account for a large fraction of a laptop's display subsystem power budget.

This chapter is a field guide to every eDP-specific mechanism that appears in the Linux DRM kernel stack, from DPCD register layouts and AUX transaction mechanics to the Intel `intel_psr.c` and `intel_pps.c` implementations and the AMD display core's `amdgpu_dm_psr.c`. It also covers the `drm_panel` subsystem, bridge chips used in Chromebook and embedded designs, and the full complement of sysfs and debugfs interfaces available to diagnose display power issues on shipping laptops.

---

## eDP vs. External DisplayPort: Why a Separate Spec

### Physical Layer Differences

Standard DisplayPort uses a 3.3 V logic supply and differential voltage swing levels up to 1,200 mV peak (400 mV, 600 mV, 800 mV, and 1,200 mV levels). eDP reduces the maximum swing to **400 mV peak differential** for the shortest link lengths found inside a laptop chassis, which permits lower-power PHY designs and reduces electromagnetic emissions inside an enclosed metal chassis. The connector is typically a 30- or 40-pin low-profile ZIF (zero-insertion-force) connector on both the GPU board and the panel, not the full-size DisplayPort jack.

External DP relies on hot-plug detect (HPD): a dedicated signal line transitions when a monitor is connected or disconnected. eDP panels are soldered to a flex cable that is permanently attached at manufacture. There is **no hotplug event** during normal operation — the GPU firmware and kernel driver treat the eDP panel as always present. The HPD line is repurposed for PSR interrupt signalling (the panel asserts HPD to tell the GPU to exit self-refresh) rather than physical connection events.

### eDP Version History

| Version | Base DP | Key additions |
|---------|---------|---------------|
| eDP 1.0 (2008) | DP 1.1 | Baseline eDP: panel-specific DPCD, AUX backlight registers, power sequencing T1–T12 |
| eDP 1.1 (2009) | DP 1.1 | Idle pattern, spread-spectrum clock |
| eDP 1.2 (2010) | DP 1.2 | **PSR (Panel Self-Refresh)**, link-rate table in DPCD 0x010–0x01F |
| eDP 1.3 (2011) | DP 1.2 | **DRRS** (Dynamic Refresh Rate Switching), low-power link standby |
| eDP 1.4 (2013) | DP 1.2 | **PSR2** (Selective Update), **DSC 1.1**, backlight brightness control over AUX (DPCD 0x720+) |
| eDP 1.4a (2015) | DP 1.3 | HBR3 (8.1 Gbps/lane), DSC 1.1a |
| eDP 1.4b (2018) | DP 1.4 | DSC 1.2, VESA CABC |
| eDP 1.5 (2021) | DP 2.1 | **Panel Replay** (replaces PSR2), DSC 1.2a, UHBR10/13.5/20 rates |

[Source: VESA eDP Standard summary, DisplayPort Wikipedia](https://en.wikipedia.org/wiki/DisplayPort#eDP)

### Internal vs. External Feature Set

Several DPCD register ranges are eDP-only and have no meaning on external DP sinks. The eDP DPCD space extends from 0x000 (shared with DP) through a dedicated eDP-specific range at 0x070–0x07F (PSR capability), 0x200–0x2FF (device status), 0x700–0x72F (eDP general capability and backlight registers), and 0x1B0–0x1BF (Panel Replay capability, eDP 1.5). GPU drivers detect eDP by checking the `DP_EDP_DPCD_REV` field at DPCD offset 0x700; a non-zero value indicates an eDP sink rather than a plain DP monitor.

---

## DPCD Register Space and AUX Channel Protocol

### The 1 MB Register Map

The DisplayPort Configuration Data (DPCD) is a 1 MB flat address space (20-bit addresses, 0x00000–0xFFFFF) where each byte is an independently addressable register. The first 256 addresses (0x000–0x0FF) cover link training and capability negotiation; higher ranges contain device status, PSR, backlight, and vendor-specific registers. Key registers appear throughout the chapter; the most fundamental ones are:

| DPCD Address | Constant name | Meaning |
|---|---|---|
| `0x000` | `DP_DPCD_REV` | DPCD specification revision (0x12 = DP 1.2, 0x14 = DP 1.4) |
| `0x001` | `DP_MAX_LINK_RATE` | Maximum link rate (0x06=1.62 Gbps, 0x0A=2.7 Gbps, 0x14=5.4 Gbps HBR2, 0x1E=8.1 Gbps HBR3) |
| `0x002` | `DP_MAX_LANE_COUNT` | Bits 0–4: lane count (1/2/4); bit 5: enhanced framing support; bit 7: TPS3 support |
| `0x003` | `DP_MAX_DOWNSPREAD` | Bit 0: spread-spectrum; bit 6: NO_AUX_HANDSHAKE_LINK_TRAINING |
| `0x006` | `DP_DOWNSTREAMPORT_PRESENT` | Whether downstream port is present (always 0 on eDP) |
| `0x010` | `DP_SUPPORTED_LINK_RATES` | eDP 1.4+: 8×2-byte link-rate table (in units of 200 kbps) |
| `0x070` | `DP_PSR_SUPPORT` | PSR capability byte (bit 0 = PSR1 capable, bit 1 = PSR2 capable) |
| `0x071` | `DP_PSR_CAPS` | PSR capability bits: Y granularity, SU support, SU line count |
| `0x100` | `DP_PSR_EN_CFG` | PSR enable/config (bit 0 = enable; bit 1 = link standby vs. main-link off) |
| `0x200` | `DP_SINK_COUNT` | Number of receiver ports |
| `0x201` | `DP_DEVICE_SERVICE_IRQ_VECTOR` | IRQ source bits: bit 1 = auto test, bit 2 = CP, bit 5 = sink-specific |
| `0x700` | `DP_EDP_DPCD_REV` | eDP DPCD revision (0x11=eDP 1.1 … 0x15=eDP 1.5) |
| `0x701` | `DP_EDP_GENERAL_CAP_1` | Bit 0: tcon backlight adjustment capable; bit 3: auxiliary backlight control |
| `0x720` | `DP_EDP_BACKLIGHT_MODE_SET` | Backlight control mode register |
| `0x722` | `DP_EDP_BACKLIGHT_BRIGHTNESS_MSB` | 16-bit brightness MSB over AUX |
| `0x723` | `DP_EDP_BACKLIGHT_BRIGHTNESS_LSB` | 16-bit brightness LSB over AUX |
| `0x1B0` | `DP_PANEL_REPLAY_CAP` | Panel Replay capability (eDP 1.5) |

[Source: include/drm/display/drm_dp.h in torvalds/linux](https://github.com/torvalds/linux/blob/master/include/drm/display/drm_dp.h)

### AUX Channel Transaction Format

The AUX channel is a half-duplex differential pair running at 1 Mbps, shared between native DPCD access and I2C tunnelling (used for EDID retrieval). Each transaction starts with a **START pattern** (a Manchester-coded bus reset), followed by:

1. **Command nibble** (4 bits): `0x8` = native write, `0x9` = native read, `0x4` = I2C write, `0x5` = I2C read, `0x0` = I2C write-status request.
2. **Address** (20 bits): DPCD register address.
3. **Length** (8 bits): number of bytes minus one (0x00 = 1 byte, max 0x0F = 16 bytes per transaction).
4. **Data** (0–16 bytes for writes): register payload.
5. **STOP pattern**.

The sink responds within 400 µs with an ACK (0x00), NACK (0x01), DEFER (0x02), or AUX_NACK (0x04). A DEFER response requires the source to retry after a short pause.

The Linux kernel abstracts all of this through `drm_dp_aux`:

```c
/* include/drm/display/drm_dp_helper.h */
ssize_t drm_dp_dpcd_read(struct drm_dp_aux *aux,
                         unsigned int offset,
                         void *buffer, size_t size);
ssize_t drm_dp_dpcd_write(struct drm_dp_aux *aux,
                          unsigned int offset,
                          void *buffer, size_t size);
```

These wrappers retry DEFER responses, handle transaction chunking at 16-byte boundaries, and apply the required inter-transaction spacing. The hardware AUX controller (e.g., `intel_dp_aux_xfer()` in `drivers/gpu/drm/i915/display/intel_dp_aux.c`) translates the `drm_dp_aux_msg` structure into the hardware MMIO registers that drive the physical AUX differential pair.

[Source: drivers/gpu/drm/display/drm_dp_helper.c](https://codebrowser.dev/linux/linux/drivers/gpu/drm/display/drm_dp_helper.c.html)

---

## PSR: Panel Self-Refresh

### Problem Statement

An active laptop display continuously requests the GPU to refresh pixels from the framebuffer in DRAM: at 60 Hz with a 2560×1600 panel, the display controller performs roughly 60 full-screen DMA reads per second even when the screen content is completely static. This keeps the DRAM, display controller clocks, and the DP link all active, consuming 0.5–2 W that could otherwise be saved.

PSR (Panel Self-Refresh), introduced in eDP 1.2, addresses this by equipping the panel's timing controller (TCON) with a **local frame buffer** large enough to store one full frame. Once the GPU transmits an idle frame and signals readiness, the TCON takes over scanning from its own buffer while the GPU's display pipeline clocks are partially gated.

### PSR1 Mechanism

1. The GPU detects that the framebuffer has not changed for a configurable idle timeout (typically 500 ms).
2. The GPU sends one final video frame over the eDP link, followed by an **idle pattern** (scrambled DC-balanced symbols with no video payload).
3. The TCON acknowledges receipt by sampling the idle pattern and copying the frame to its local SRAM buffer.
4. The GPU's display controller enters **link standby** or **main-link off** mode (controlled by DPCD `DP_PSR_EN_CFG` bit 1). In link standby, the AUX channel remains active; in main-link off, even the AUX clock is gated.
5. The panel now self-refreshes from its SRAM at the programmed refresh rate, with no demand on GPU memory.

**PSR1 exit**: When the framebuffer changes (e.g., a cursor move, application paint), the GPU asserts a status change that generates an HPD IRQ from the panel back to the GPU. Alternatively, the driver polls DPCD `DP_PSR_STATUS` (0x2006). The GPU re-enables clocks, re-trains the link if necessary, resumes video streaming, and the TCON switches back to live video.

### Intel PSR Implementation

Intel's PSR code lives in `drivers/gpu/drm/i915/display/intel_psr.c`. The principal exported functions are:

```c
/* drivers/gpu/drm/i915/display/intel_psr.c */
void intel_psr_init_dpcd(struct intel_dp *intel_dp);
void intel_psr_enable(struct intel_dp *intel_dp,
                      const struct intel_crtc_state *crtc_state,
                      const struct drm_connector_state *conn_state);
void intel_psr_update(struct intel_dp *intel_dp,
                      const struct intel_crtc_state *crtc_state,
                      const struct drm_connector_state *conn_state);
void intel_psr_disable(struct intel_dp *intel_dp,
                       const struct intel_crtc_state *old_crtc_state);
void intel_psr_invalidate(struct drm_i915_private *dev_priv,
                          unsigned frontbuffer_bits,
                          enum fb_op_origin origin);
void intel_psr_flush(struct drm_i915_private *dev_priv,
                     unsigned frontbuffer_bits,
                     enum fb_op_origin origin);
```

The driver integrates with **frontbuffer tracking**: whenever a GPU command touches the scanout framebuffer (via `drm_fb_helper_set_par`, `drm_gem_fb_vmap`, or a modesetting plane update), `intel_psr_invalidate()` is called to force PSR exit. Once the rendering settles, `intel_psr_flush()` schedules re-entry after the idle timeout via a `delayed_work` item.

The four PSR configurations tracked in `intel_crtc_state` are:

- `has_psr` alone → **PSR1**
- `has_psr` + `has_sel_update` → **PSR2**
- `has_psr` + `has_panel_replay` → **Panel Replay**
- `has_psr` + `has_panel_replay` + `has_sel_update` → **Panel Replay with Selective Update**

[Source: drivers/gpu/drm/i915/display/intel_psr.c — torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_psr.c)

---

## PSR2: Selective Update

### Concept and Bandwidth Savings

PSR1 has a significant limitation: even if only a 100×100 px cursor has moved, the GPU must retransmit the entire frame to exit PSR. PSR2 (introduced in eDP 1.4) solves this with **Selective Update (SU)**: only the display regions that actually changed are retransmitted in a given frame. The panel TCON merges the changed rectangles into its local buffer without overwriting the unchanged regions.

SU rectangles are described at a **64-pixel wide × 4-line tall granularity** — the unit is set by the panel TCON's memory architecture and communicated to the host via DPCD registers `DP_PSR2_SU_X_GRANULARITY` and `DP_PSR2_SU_Y_GRANULARITY`. The GPU driver rounds up dirty regions to these granularity boundaries before encoding the SU descriptor in the VSC SDP (Vertical Synchronisation Secondary Data Packet) transmitted at the start of each frame.

PSR2 enables meaningful power savings during video playback: if a video plays in a quarter of the screen, the SU region covers only that quadrant, and the rest of the panel remains self-refreshed. Without PSR2, any framebuffer write — even a single video frame — would force a full-frame retransmit.

### Intel PSR2 Selective Fetch

In Gen12+ (Tiger Lake and later), Intel added a further optimization called **PSR2 Selective Fetch**: the display controller only reads the changed SU rectangles from DRAM rather than reading the entire plane. This reduces memory bandwidth further because DRAM reads for unchanged scanlines are completely eliminated.

```c
/* Checked during PSR2 enablement */
bool intel_psr2_sel_fetch_enabled(struct drm_i915_private *i915)
{
    return i915->display.psr.psr2_sel_fetch_enabled;
}
```

### Known Quirks and Bugs

PSR2 has historically been a source of visual artifacts on production hardware. Documented issues include:

- **Cursor corruption during PSR2 exit**: When the cursor plane moves while PSR2 is active, the SU region calculation may not include the old cursor position, leaving ghost cursor artifacts. Workaround `Wa_16014451276` adjusts the SU region start coordinate to always include the previous cursor rectangle.
- **SU region calculation errors on ADL-P**: The `PSR2_MAN_TRK_CTL_SU_REGION_END_ADDR` field was calculated incorrectly in the Alder Lake driver, causing the SU region to end one line too early and corrupting the bottom scanline. Fixed in the kernel via patches to `intel_psr.c`.
- **DC6 exit instability**: Certain DMC firmware versions (notably `icl_dmc_ver1_07.bin`) cause PSR2 SU to fail after a DC6 power state exit when `EDP_PSR_TP1_TP3_SEL` is retained in `PSR_CTL`. The workaround clears this bit before each PSR2 re-enable.

[Source: drm/i915 PSR2 selective update patch series](https://lore.kernel.org/all/20210914212507.177511-3-jose.souza@intel.com/t/)

### AMD PSR Implementation

AMD's PSR logic for Raven Ridge/Renoir/Rembrandt APUs lives in `drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_psr.c`. The key entry points are:

```c
/* drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_psr.c */
void amdgpu_dm_set_psr_caps(struct dc_link *link);
bool amdgpu_dm_psr_enable(struct dc_stream_state *stream);
bool amdgpu_dm_psr_disable(struct dc_stream_state *stream);
bool amdgpu_dm_psr_update_active(struct dc_stream_state *stream,
                                 bool enable);
```

AMD enabled PSR by default on capable hardware starting in Linux 5.16. The capability check in `amdgpu_dm_set_psr_caps()` reads the panel's DPCD, verifies that the link signal type is `SIGNAL_TYPE_EDP`, and sets the `psr_feature_enabled` flag in `dc_link_params`. The PSR enable path ultimately calls into the DC (Display Core) abstraction layer: `dc_link_set_psr_allow_active()` → `core_link_enable_output()` → hardware-specific PSR register writes.

[Source: drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_psr.h — torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_psr.h)

---

## Panel Replay (eDP 1.5 / DP 2.1)

### Motivation

PSR2's principal weakness is its trust model: the GPU assumes the panel's local buffer contains the correct image and enters idle mode. If the buffer is corrupted (e.g., panel SRAM bit-flip, power glitch), the panel displays wrong content until the next full-frame refresh, and the GPU has no way to detect this. On 4K and 5K laptop panels where PSR2 would run for extended periods, this is an unacceptable reliability risk.

Panel Replay (introduced in eDP 1.5 and also specified for DP 2.1 external panels) adds a **full-frame CRC gate** before entering replay mode:

1. The GPU computes a CRC of the framebuffer content and transmits it to the panel in the VSC SDP.
2. The panel's TCON independently computes a CRC of the received frame and compares it with the GPU-supplied value.
3. Only if the CRCs match does the panel assert readiness for replay mode.
4. On error, the panel signals a frame-error IRQ (via DPCD `DP_DEVICE_SERVICE_IRQ_VECTOR` bit 5), and the GPU retransmits.

This CRC-gated activation ensures that the GPU and panel agree on exact pixel content before the link goes idle, eliminating the PSR2 stale-buffer failure mode.

### Panel Replay DPCD Registers

```
0x1B0  DP_PANEL_REPLAY_CAP
       bit 0: Panel Replay supported
       bit 1: Panel Replay Selective Update supported
       bit 2: DP_PANEL_REPLAY_EARLY_TRANSPORT_SUPPORT (eDP 1.5)

0x1B1  DP_PANEL_REPLAY_CAP2
       bit 7: VSC SDP CRC supported

0x1B2  PANEL_REPLAY_CONFIG
       bit 0: DP_PANEL_REPLAY_ENABLE
       bit 1: DP_PANEL_REPLAY_VSC_SDP_CRC_EN
```

[Source: drm/panel replay DPCD register patches on intel-gfx](https://www.mail-archive.com/intel-gfx@lists.freedesktop.org/msg340361.html)

### Improved Resume Timing

Panel Replay also improves link re-establishment timing after replay exit. PSR2 required a full link training sequence (TP1 → TP2 → normal video), which could take several milliseconds. Panel Replay specifies a **fast wake** sequence using pre-stored link parameters, reducing exit latency to under 1 ms on modern panels. This makes Panel Replay safe to enter even for shorter idle periods than PSR2.

### Linux Support Status

Panel Replay support landed in the mainline kernel beginning with Linux 6.3, implemented by Intel engineers in `intel_psr.c`. The code unifies PSR1, PSR2, Panel Replay, and Panel Replay + Selective Update under a single state machine controlled by the `has_panel_replay` flag in `intel_crtc_state`. The Panel Replay path uses `_panel_replay_init_dpcd()` to read `DP_PANEL_REPLAY_CAP` and conditionally enables CRC generation in the VSC SDP encoder.

AMD Panel Replay support in `amdgpu_dm` was under active development through 2024–2025 in the DC stack, with capability detection based on the same `DP_PANEL_REPLAY_CAP` register.

> **Note: needs verification** — the exact kernel version in which AMD Panel Replay became stable on production hardware should be confirmed against the amdgpu release notes for kernel 6.7+.

---

## DRRS: Dynamic Refresh Rate Switching

### Concept

DRRS allows the GPU to reduce the panel's vertical refresh rate during periods of display inactivity. Unlike VRR/FreeSync (which adjusts refresh rate per-frame to match rendering speed), DRRS is a coarser power-saving mechanism: the driver steps down from the active refresh rate (typically 60 Hz) to a lower idle rate (40 Hz is common) after a programmable idle timeout, saving approximately 15% of panel scan power.

### Seamless vs. Non-Seamless DRRS

eDP 1.3 defines two DRRS modes:

- **Seamless DRRS**: The panel transitions to the new refresh rate during the vertical blanking interval without visible disruption. Requires the panel TCON to support seamless rate switching, indicated by DPCD `DP_SUPPORTED_LINK_RATES` (0x010–0x01F) listing the alternative rates.
- **Non-seamless DRRS**: The driver inserts a brief blank period during the rate switch. Simpler to implement but produces a visible flash; used as a fallback on older panels.

### Link Rate Table in DPCD

eDP 1.4 panels advertise supported link rates via an 8-entry table at DPCD 0x010–0x01F, each entry a 16-bit value in units of 200 kbps. A panel might list:

```
0x010: 0x006C  → 0x006C × 200 kbps = 21,600 kbps = 2.16 Gbps  (low-rate mode)
0x012: 0x00A0  → 0x00A0 × 200 kbps = 32,000 kbps = 3.24 Gbps  (idle mode)
0x014: 0x0140  → 0x0140 × 200 kbps = 64,000 kbps = 6.48 Gbps  (active mode)
```

The driver selects the appropriate entry based on the required pixel clock and the target refresh rate.

### Intel DRRS Implementation

Intel DRRS is implemented in `drivers/gpu/drm/i915/display/intel_drrs.c`. The key mechanism is a `delayed_work` item that fires after 1 second of display inactivity:

```c
/* drivers/gpu/drm/i915/display/intel_drrs.c */
static void intel_drrs_schedule_work(struct intel_crtc *crtc)
{
    mod_delayed_work(i915->unordered_wq,
                     &crtc->drrs.work,
                     msecs_to_jiffies(1000));
}

static void intel_drrs_downclock_work(struct work_struct *work)
{
    /* If no frontbuffer activity, switch to low refresh */
    if (!crtc->drrs.busy_frontbuffer_bits)
        intel_drrs_set_state(crtc, DRRS_LOW_RR);
}
```

The refresh rate switch itself uses two methods depending on platform generation:
- **Pipeconf registers**: Sets `TRANSCONF_REFRESH_RATE_ALT_VLV` (Valleyview) or `TRANSCONF_REFRESH_RATE_ALT_ILK` (Ivy Bridge+) via `intel_de_rmw()`.
- **M/N divisor method**: Calls `intel_cpu_transcoder_set_m1_n1()` with the pre-computed M/N parameters for the low-rate pixel clock, stored as `m2_n2` in `intel_crtc_state` alongside the high-rate `m_n`.

DRRS integrates with the same frontbuffer tracking mechanism as PSR: any GPU render to a visible plane sets the `busy_frontbuffer_bits` flag, which cancels the delayed work via `cancel_delayed_work_sync()` and immediately switches back to the high refresh rate.

[Source: drivers/gpu/drm/i915/display/intel_drrs.c — torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_drrs.c)

### DRRS and VRR Interaction

DRRS and VRR (Variable Refresh Rate / FreeSync / G-Sync) are mutually exclusive in current Linux drivers: VRR controls the refresh rate on a per-frame basis via the `drm_connector` VRR capability, while DRRS manages it at the link/pixel-clock level. When a compositor enables VRR on an eDP panel, the driver disables DRRS. Intel's `intel_drrs_activate()` checks `intel_crtc_state->vrr.enable` and returns early if VRR is active.

---

## Backlight Control Methods

Laptop backlight control on Linux is handled via one of four mechanisms, and the kernel must select the right one at probe time based on firmware and panel capabilities.

### Method A: PWM via pwm_backlight

The most common method on x86 laptops pre-2015. A dedicated PWM output from the PCH (Platform Controller Hub) or SoC drives the LED backlight controller's dimming input. The Linux driver stack:

1. **Platform device**: `pwm-backlight` binding creates a `platform_device` with PWM channel reference and min/max brightness values.
2. **Kernel driver**: `drivers/video/backlight/pwm_bl.c` registers a `backlight_device` under `/sys/class/backlight/`.
3. **Brightness control**: Writing to `/sys/class/backlight/intel_backlight/brightness` triggers `pwm_bl_update_status()`, which calls `pwm_apply_state()` with the duty cycle computed from the brightness level.

PWM frequency is a significant parameter: too low (< 200 Hz) produces visible flicker at low brightness levels; too high (> 100 kHz) can generate EMI that interferes with touchscreen or WiFi. Most laptop panels specify a recommended PWM frequency in their datasheet (typically 200 Hz to 25 kHz).

### Method B: I2C Backlight Controller ICs

Some designs use a dedicated backlight controller IC connected via I2C rather than raw PWM. Common ICs:

- **TI LP8557**: Constant-current LED driver with I2C interface, 256-step linear brightness, accepts both PWM and I2C commands. Driver: `drivers/video/backlight/lp8557.c`.
- **TI LM3630A**: Dual-string LED driver with I2C brightness control and hardware PWM dimming. Driver: `drivers/video/backlight/lm3630a_bl.c`.
- **TI TPS61165**: Boost converter with single-wire serial interface; less common in mainline Linux.

These ICs appear in the Device Tree or ACPI namespace as I2C slave devices, and their kernel drivers register backlight devices that the power management infrastructure (logind, GNOME Settings Daemon, etc.) controls through the same `/sys/class/backlight/` interface.

### Method C: eDP AUX Backlight (DPCD Control)

eDP 1.4 introduced **backlight brightness control over the AUX channel** via DPCD registers, eliminating the dedicated PWM line entirely. This reduces EMI and provides more accurate brightness control because the GPU directly programs the TCON's brightness register without relying on PWM signal integrity.

Capability detection reads DPCD `DP_EDP_GENERAL_CAP_1` (0x701):
- Bit 0: TCON backlight adjustment capable
- Bit 3: Auxiliary backlight control

If both bits are set, the driver initialises AUX backlight control. Intel's implementation is in `drivers/gpu/drm/i915/display/intel_dp_aux_backlight.c`:

```c
/* drivers/gpu/drm/i915/display/intel_dp_aux_backlight.c */
static int intel_dp_aux_set_backlight(const struct drm_connector_state *conn_state,
                                      u32 level)
{
    struct intel_dp *intel_dp = ...;
    u8 vals[2] = {};

    /* Panel may support 8-bit or 16-bit brightness */
    if (intel_dp->edp_dpcd[2] & DP_EDP_BACKLIGHT_BRIGHTNESS_BYTE_COUNT) {
        vals[0] = (level & 0xFF00) >> 8;  /* MSB at 0x722 */
        vals[1] = level & 0xFF;           /* LSB at 0x723 */
        drm_dp_dpcd_write(&intel_dp->aux, DP_EDP_BACKLIGHT_BRIGHTNESS_MSB,
                          vals, 2);
    } else {
        vals[0] = level & 0xFF;
        drm_dp_dpcd_write(&intel_dp->aux, DP_EDP_BACKLIGHT_BRIGHTNESS_MSB,
                          vals, 1);
    }
    return 0;
}
```

The `intel_dp_aux_init_backlight_funcs()` function registers `intel_dp_aux_set_backlight()` and `intel_dp_aux_get_backlight()` as the backlight ops when AUX control is detected. The module parameter `i915.enable_dpcd_backlight=1` guards this path, as AUX backlight support on early eDP 1.4 panels was inconsistently implemented.

[Source: drm/i915 DPCD backlight commit e7156c8](https://github.com/torvalds/linux/commit/e7156c833903fbfc77407816af56a97945b86b3a)

### Method D: ACPI Video Backlight

Older platforms (pre-Sandy Bridge) rely on ACPI firmware methods for backlight control. The `video.ko` driver calls ACPI `_BCM` (Brightness Control Method) to set brightness and `_BCL` (Brightness Control Levels) to enumerate available levels. On modern hardware, ACPI video backlight is deprioritised via the `video.use_native_backlight=1` kernel parameter (default on most modern systems), which causes the kernel to prefer the native GPU backlight driver over the ACPI firmware method.

### Ambient Light Sensor Integration

Many modern laptops include an ALS (Ambient Light Sensor) on the same SoC or connected via I2C. The Linux IIO (Industrial I/O) subsystem exposes ALS readings via `/sys/bus/iio/devices/iio:device*/in_illuminance_raw`. The `iio-sensor-proxy` daemon ([github.com/hadess/iio-sensor-proxy](https://github.com/hadess/iio-sensor-proxy)) bridges IIO readings to a D-Bus interface that GNOME Settings Daemon's ALS plugin consumes to automatically adjust the `/sys/class/backlight/` brightness target.

---

## DSC on eDP: Display Stream Compression

### Why DSC Is Needed

A 4K (3840×2400) laptop panel at 120 Hz requires a pixel data rate of approximately 44 Gbps at 30 bpp. HBR3 eDP 1.4b with 4 lanes provides 4 × 8.1 Gbps = 32.4 Gbps raw, which cannot carry 4K@120Hz uncompressed. **Display Stream Compression (DSC)** provides visually lossless compression (no perceptible difference at typical viewing distances) that reduces the effective bandwidth requirement by 2:1 to 3:1, making high-refresh 4K eDP feasible.

DSC 1.2a (standardised in VESA DSC 1.2, carried by eDP 1.4b+) divides the image into horizontal **slices** (typically 64–240 px tall, full panel width). Each slice is compressed independently by a hardware **encoder** in the GPU and decompressed by a **decoder** in the panel TCON. The encoder and decoder share a fixed algorithm (three-component prediction + entropy coding) with parameters negotiated during link training.

### Key DSC Parameters

| Parameter | Typical value | Meaning |
|---|---|---|
| Slice width | 2560 px (full width) | Width of one DSC slice |
| Slice height | 64 px | Height of one DSC slice (must align with pipe scanout) |
| Slices per line | 1–4 | Number of parallel slice encoders |
| Bits per pixel (bpp) | 8.0–12.0 (6.25 min) | Compressed output bpp; lower = more compression |
| Chunk size | ceil(slice_width × bpp / 8) | Bytes per slice per line |
| RC buffer size | 8,192 bytes | Rate-control buffer in encoder/decoder |

The target bpp is negotiated by comparing the available link bandwidth against the pixel data rate and selecting the minimum bpp that satisfies both the link and the panel's minimum bpp capability (read from DPCD `DP_DSC_MIN_SLICE_WIDTH`, `DP_DSC_MAX_BITS_PER_PIXEL`).

### Intel DSC Implementation

Intel's DSC code is split between two files:

```c
/* drivers/gpu/drm/i915/display/intel_dsc.c */
void intel_dsc_enable(const struct intel_crtc_state *crtc_state);
void intel_dsc_disable(const struct intel_crtc *crtc);
int intel_dsc_compute_params(struct intel_crtc_state *crtc_state);
```

`intel_dsc_compute_params()` computes the PPS (Picture Parameter Set) — a 128-byte DSC configuration block transmitted to the panel during link training via a DSC PPS SDP (Secondary Data Packet). The PPS includes all slice parameters, the compression ratio, and the rate-control model parameters that both the GPU encoder and panel decoder must use to stay synchronised.

### AMD DSC Implementation

In the AMD Display Core (DC) stack, DSC computation is handled by `dc_dsc_compute_config()` in `drivers/gpu/drm/amd/display/dc/dsc/`. This function takes the pipe's pixel clock, color format, and the sink's DSC capability DPCD block as inputs, and fills a `dc_dsc_config` structure with the agreed parameters. AMD added eDP DSC support (DSC-over-eDP) to allow higher-resolution panels on RDNA2 and later APUs.

[Source: AMDGPU DSC eDP support — Phoronix](https://www.phoronix.com/news/AMDGPU-DSC-eDP)

### DSC and Panel Replay Interaction

eDP 1.5 mandates DSC when Panel Replay is used on panels requiring UHBR (Ultra High Bit Rate) links (UHBR10 = 10 Gbps/lane, UHBR13.5, UHBR20). Panel Replay's CRC verification mechanism is designed to work on the compressed domain — the CRC is computed on the DSC-compressed bitstream, not the uncompressed pixels, so the panel decoder computes its own CRC on the received stream before decompressing.

---

## Panel Power Sequencing: T1–T12 Timing

### The eDP Power Sequence

eDP specifies a deterministic power-on and power-off sequence with named timing intervals. Getting these timings wrong causes panels to fail to initialize after boot or fail to recover after suspend/resume. The key timings in the eDP specification (note: the kernel historically used different naming conventions from the spec):

| Timing | eDP Spec name | Typical range | Description |
|--------|--------------|---------------|-------------|
| T1 | VDD rise | 0–10 ms | Time for VDD supply rail to reach stable voltage |
| T2 | VDD → AUX | 0–200 ms | Minimum delay after VDD stable before AUX channel may be used |
| T3 | AUX → video | 0–200 ms | Minimum delay after AUX communication before video data |
| T4 | Video stable → backlight on | 0–200 ms | Minimum delay from first valid video line to backlight enable |
| T5 | Backlight off → video off | 0–200 ms | Minimum delay from backlight disable to video shutdown |
| T6 | Video off → VDD off | 0–200 ms | Minimum delay from video shutdown to VDD deactivation |
| T12 | VDD off → VDD on | ≥500 ms | Mandatory power-cycle hold: prevents capacitor recharge issues |

The 500 ms minimum T12 is critical: if VDD is toggled faster (e.g., during a rapid suspend/resume cycle), the panel's internal power sequencer may not complete its discharge cycle, and the panel's TCON will not reinitialize correctly — resulting in a blank screen or scrambled display until the next power cycle.

### ACPI Power Sequencing on x86

On ACPI platforms (most x86 laptops), the firmware provides the panel timing parameters via the PP (Panel Power) registers in the PCH. The Intel driver reads these in `intel_pps_init()`:

```c
/* drivers/gpu/drm/i915/display/intel_pps.c */
static void pps_init_delays_bios(struct intel_dp *intel_dp,
                                 struct edp_power_seq *seq)
{
    /* Read T1+T3, T8, T9, T10, T11+T12 from PP_ON_DELAYS
     * and PP_OFF_DELAYS hardware registers programmed by BIOS */
    seq->t1_t3   = REG_FIELD_GET(PANEL_POWER_UP_DELAY_MASK,   pp_on);
    seq->t8      = REG_FIELD_GET(PANEL_LIGHT_ON_DELAY_MASK,   pp_on);
    seq->t9      = REG_FIELD_GET(PANEL_LIGHT_OFF_DELAY_MASK,  pp_off);
    seq->t10     = REG_FIELD_GET(PANEL_POWER_DOWN_DELAY_MASK, pp_off);
    seq->t11_t12 = REG_FIELD_GET(PANEL_POWER_CYCLE_DELAY_MASK, pp_div) * 1000;
}
```

> **Note on naming**: The i915 driver's `edp_power_seq` struct uses field names (`t1_t3`, `t8`, `t9`, `t10`, `t11_t12`) that do not correspond directly to the eDP specification's T1–T12 numbering. A historical analysis by Keith Packard established that the VBIOS naming conventions differ from the eDP spec — specifically, `t7` in the driver corresponds to what the spec calls T11+T12. The kernel always takes the maximum of the BIOS-supplied delay and the panel's DPCD-reported delay, since waiting longer than necessary is safe while waiting too briefly risks panel initialization failure.

[Source: drm/i915 eDP power sequencing delay corrections — LKML](https://lkml.iu.edu/hypermail/linux/kernel/1109.3/03024.html)

### intel_pps Functions

```c
/* Key functions in drivers/gpu/drm/i915/display/intel_pps.c */
void intel_pps_init(struct intel_dp *intel_dp);
void intel_pps_enable(struct intel_dp *intel_dp);
void intel_pps_disable(struct intel_dp *intel_dp);
void intel_pps_wait_power_cycle(struct intel_dp *intel_dp);

/* Called during suspend/resume to enforce T12 */
void intel_pps_wait_panel_power_cycle(struct intel_dp *intel_dp);
```

`intel_pps_wait_panel_power_cycle()` is called during the panel enable path after a previous panel disable. It checks the timestamp of the last VDD-off event and delays until T12 milliseconds have elapsed, ensuring that no firmware or userspace action can accidentally violate the panel's power cycle minimum.

### Device Tree Power Sequencing

On non-ACPI platforms (ARM SoCs, Chromebooks), the panel power sequencing timings come from Device Tree properties on the panel node:

```dts
/* arch/arm64/boot/dts/... (example) */
panel@0 {
    compatible = "samsung,atna33xc20";
    enable-gpios = <&gpio 45 GPIO_ACTIVE_HIGH>;
    power-supply = <&panel_vcc>;

    /* eDP T-sequence timings */
    hpd-gpios = <&gpio 80 GPIO_ACTIVE_HIGH>;
};
```

The `panel-edp.yaml` binding ([kernel.org DT bindings](https://www.kernel.org/doc/Documentation/devicetree/bindings/display/panel/panel-edp.yaml)) formalises these properties for mainline eDP panel drivers.

---

## Touch Panels, drm\_panel, and Bridge Chips

### The drm\_panel Subsystem

The `drm_panel` subsystem (`drivers/gpu/drm/panel/`) provides an abstraction layer for display panels that are controlled separately from the display controller — particularly panels with MIPI DSI interfaces, but also eDP panels on non-Intel platforms. The core structure is `drm_panel_funcs`:

```c
/* include/drm/drm_panel.h */
struct drm_panel_funcs {
    int (*prepare)(struct drm_panel *panel);
    int (*enable)(struct drm_panel *panel);
    int (*disable)(struct drm_panel *panel);
    int (*unprepare)(struct drm_panel *panel);
    int (*get_modes)(struct drm_panel *panel,
                     struct drm_connector *connector);
    enum drm_panel_orientation (*get_orientation)(struct drm_panel *panel);
};
```

The power lifecycle is:
- `prepare()`: Power on the panel (VDD, reset release), allow it to initialize its internal registers. Called before the display controller starts transmitting.
- `enable()`: Enable the backlight and pixel output. Called after the display controller has established video transmission.
- `disable()`: Disable backlight. Called before the display controller stops video.
- `unprepare()`: Power off the panel. Called after video has stopped.

[Source: drivers/gpu/drm/panel/panel-simple.c — torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/panel/panel-simple.c)

### panel-simple Driver

`panel-simple.c` is the mainline driver for panels fully described by Device Tree properties (timing, power-supply, backlight, enable GPIO). It implements `drm_panel_funcs` and the `get_modes()` path that produces `drm_display_mode` structures from DT display-timings nodes. Several hundred panels are registered in the `panels[]` array with vendor-supplied timing data.

For eDP panels on Chromebooks and ARM laptops, `panel-simple` handles the `prepare`/`enable`/`disable`/`unprepare` lifecycle in conjunction with the `drm_panel_bridge` wrapper, which maps drm\_bridge callbacks to panel lifecycle calls and allows the panel to participate in the atomic modesetting commit pipeline.

### Touch Controllers and ACPI/I2C HID

Most laptop touch panels are not electrically integrated with the eDP link — they run on a separate I2C or USB bus. The Linux I2C HID driver (`drivers/hid/i2c-hid/`) handles I2C HID touchscreens (the majority of modern laptop touch panels), binding to ACPI `_HID` / `_CID` entries like `ELAN0001`, `MSFT0001`, or specific PnP IDs in the ACPI namespace. The touch controller's I2C device is listed as a companion device alongside the eDP display in the ACPI firmware, sharing an interrupt line (typically from a GPIO).

Some newer designs embed the touch digitiser ASIC inside the display module itself, sharing the eDP flex cable's ground and power rails but using a separate I2C bus for data.

### TI SN65DSI86: DSI-to-eDP Bridge

On ARM-based Chromebooks and embedded platforms where the SoC has MIPI DSI output but the panel uses eDP, Texas Instruments' **SN65DSI86** bridge chip converts the DSI stream to eDP. The Linux driver at `drivers/gpu/drm/bridge/ti-sn65dsi86.c` implements a full DRM bridge:

```c
/* drivers/gpu/drm/bridge/ti-sn65dsi86.c (CONFIG_DRM_TI_SN65DSI86) */
static const struct drm_bridge_funcs ti_sn_bridge_funcs = {
    .attach          = ti_sn_bridge_attach,
    .pre_enable      = ti_sn_bridge_pre_enable,
    .enable          = ti_sn_bridge_enable,
    .disable         = ti_sn_bridge_disable,
    .post_disable    = ti_sn_bridge_post_disable,
    .detect          = ti_sn_bridge_detect,
    .edid_read       = ti_sn_bridge_edid_read,
    .atomic_check    = ti_sn_bridge_atomic_check,
};
```

The SN65DSI86 requires four power supplies (1.8 V VCCIO, 1.8 V VPLL, 1.2 V VCCA, 1.2 V VCC), a chip-enable GPIO, and a reference clock. On many Chromebooks, the EDID from the eDP panel is retrieved by the bridge chip via its own AUX channel master and proxied to the SoC's DRM subsystem through the bridge's `edid_read()` callback.

[Source: CONFIG_DRM_TI_SN65DSI86 in lkddb](https://cateee.net/lkddb/web-lkddb/DRM_TI_SN65DSI86.html)

### Toshiba TC358775: DSI-to-LVDS

For legacy LVDS panels (still common in industrial and automotive applications), the TC358775 bridge converts MIPI DSI to LVDS. LVDS was the predecessor to eDP on laptops and is not covered by the eDP specification, but the driver pattern (`drivers/gpu/drm/bridge/tc358775.c`) is directly analogous.

---

## Debugging Tools and sysfs/debugfs Interfaces

### PSR Status via debugfs

Intel's i915 driver exposes detailed PSR state via debugfs. With a running i915-based system:

```bash
# Read PSR status (shows current PSR state, entry/exit counts)
cat /sys/kernel/debug/dri/0/i915_edp_psr_status

# Example output:
# Sink support: yes [0x03]
# Source support: yes
# PSR mode: PSR2 enabled
# PSR entry count: 1847
# PSR exit count: 1847
# PSR deepest sleep: 842 ms
# Max sleep: 1024 ms
```

```bash
# Read DRRS status
cat /sys/kernel/debug/dri/0/i915_drrs_status
```

### DPCD Register Inspection

The `intel_dp_dpcd` tool (part of intel-gpu-tools) reads and decodes DPCD registers from an eDP panel:

```bash
# Install intel-gpu-tools, then:
intel_dp_dpcd /dev/dri/card0

# Outputs decoded DPCD registers:
# DPCD rev: 1.4
# Max link rate: 8.10 Gbps (HBR3)
# Max lane count: 4
# PSR support: yes (PSR2 capable)
# eDP DPCD rev: 1.4b
# Backlight control: AUX channel
```

### Backlight Inspection

```bash
# List available backlight controllers
ls /sys/class/backlight/

# Read current brightness and max brightness
cat /sys/class/backlight/intel_backlight/brightness
cat /sys/class/backlight/intel_backlight/max_brightness

# Set brightness to 50% (compute 50% of max_brightness)
echo 500 > /sys/class/backlight/intel_backlight/brightness
```

### Kernel Log Markers for PSR Debugging

PSR entry and exit events appear in `dmesg` (at debug verbosity) as:
```
[drm:intel_psr_enable] *DEBUG* Enabling PSR2
[drm:intel_psr_exit] *DEBUG* PSR2 exit: frontbuffer dirty
[drm:intel_psr_invalidate] *DEBUG* PSR invalidated (origin: FLIP)
```

Enable debug messages with:
```bash
echo 0x1 > /sys/kernel/debug/dri/0/i915_verbose_state_checks
```

Or via the `drm.debug=0x1f` kernel parameter for full DRM debug logging.

### Panel Replay Capability Detection

```bash
# Check Panel Replay capability directly from DPCD
# DPCD 0x1B0 byte contains Panel Replay caps
sudo intel_dp_dpcd /dev/dri/card0 | grep -i "panel replay"

# Or check kernel's parsed capability from sysfs:
cat /sys/kernel/debug/dri/0/i915_edp_psr_status | grep -i "replay"
```

---

## Integrations

This chapter connects to several other chapters in the book:

- **Ch2 (KMS Atomic Modesetting)**: The KMS atomic commit path drives the entire panel enable/disable lifecycle. When userspace commits a `CRTC_ACTIVE=1` modesetting state, the kernel calls `intel_pps_enable()` to sequence VDD on and then `intel_ddi_enable_pipe_clock()` to start the eDP link. The `drm_panel_bridge` translates bridge callbacks into the `drm_panel_funcs` prepare/enable sequence, all orchestrated by the atomic commit tail worker. See Ch2 for the full atomic state machine.

- **Ch3 (PSR2 and Panel Replay as Advanced KMS Features)**: PSR2 and Panel Replay are implemented as properties of the `drm_connector` and `drm_crtc` that participate in the KMS plane update path. The `PANEL_SELF_REFRESH` connector property and the frontbuffer dirty tracking subsystem discussed here are built on the KMS infrastructure described in Ch3.

- **Ch51 (GPU Power Management)**: PSR and DRRS exist to reduce the power budget consumed by the display subsystem, which feeds into the overall GPU power envelope managed by the runtime PM infrastructure described in Ch51. The DC3CO (DC3 Clock Off) state that Intel enters during PSR2 idle periods is one level of a hierarchy of GPU power states; Ch51 describes the full runtime PM / RC6 / DC state machine.

- **Ch126 (Hybrid Graphics Laptop Power Management)**: On laptops with both an integrated GPU (iGPU) and a discrete GPU (dGPU), the eDP panel is always driven by the iGPU's display engine, even when the dGPU is active for rendering (via PRIME). PSR and DRRS run on the iGPU display engine regardless of which GPU is rendering. Ch126 covers the power handoff protocols between iGPU and dGPU.

- **Ch181 (DisplayPort 2.1 Link Rates and Feature Sets)**: eDP 1.5 adopts the DP 2.1 physical layer, including UHBR10 (10 Gbps/lane), UHBR13.5, and UHBR20 link rates. These rates enable 8K eDP panels and make mandatory DSC more common. Ch181 describes the 128b/132b encoding used by UHBR rates and how link training differs from 8b/10b HBR3 links.

- **Ch183 (EDID over I2C-over-AUX for eDP Panels)**: eDP panels expose their EDID (Extended Display Identification Data) via an I2C-over-AUX tunnel — the same AUX channel used for DPCD access is also used to address an I2C device at address 0x50 which serves the 256-byte EDID block. Ch183 covers the I2C-over-AUX transaction encapsulation, EDID block reading via `drm_do_probe_ddc_edid()`, and how the kernel parses the resulting EDID to extract panel timings, preferred modes, and eDP-specific feature flags.

- **Ch188 (Display Power States — DC State Hierarchy)**: PSR and DRRS fit into the broader hierarchy of display controller power states (DC0 through DC6) described in Ch188. PSR1 and DRRS lower display bandwidth without requiring a display controller power state transition; PSR2 selective fetch enables DC5 (display clock gating); Panel Replay on idle enables DC6 (display PLL off). Ch188 describes how the kernel decides which DC state to enter based on active display features.

---

*This chapter covers kernel code primarily in the `v6.8` range of the mainline kernel. Rapidly evolving areas — Panel Replay on AMD hardware, UHBR eDP link training — should be verified against the current DRM mailing list.*
