# Chapter 188: Display Power States — DPMS, Panel Self-Refresh, and Display Idle Management

**Target audiences**: Power management engineers optimizing laptop battery life; laptop OEM kernel developers tuning display idle sequences; compositor developers implementing display dimming and blanking; system integrators configuring PSR, DC states, and ACPI suspend on shipping platforms.

---

## Table of Contents

1. [Display Power Problem Space](#1-display-power-problem-space)
2. [VESA DPMS Legacy Standard](#2-vesa-dpms-legacy-standard)
3. [KMS CRTC Active — Modern Blank/Unblank](#3-kms-crtc-active--modern-blankunblank)
4. [Intel Display C-States](#4-intel-display-c-states)
5. [AMD DCN Display Power Management](#5-amd-dcn-display-power-management)
6. [PSR System-Level View](#6-psr-system-level-view)
7. [Backlight Power Management and ALS](#7-backlight-power-management-and-als)
8. [power-profiles-daemon Display Integration](#8-power-profiles-daemon-display-integration)
9. [Compositor Display Idle Management](#9-compositor-display-idle-management)
10. [ACPI S0ix Modern Standby and Display Requirements](#10-acpi-s0ix-modern-standby-and-display-requirements)
11. [Integrations](#11-integrations)

---

## 1. Display Power Problem Space

The display subsystem is one of the largest power consumers on a modern laptop. A 15-inch IPS panel at full brightness can consume 6–8 W in total, with power distributed roughly as follows (figures are illustrative; exact values depend on panel technology, size, and brightness):

| Component | Approximate Power |
|-----------|------------------|
| Panel backlight (LED array, full brightness) | 2–4 W |
| Panel electronics (T-con, column/row drivers, LCDs) | 0.5–1 W |
| Display controller (CRTC, planes, output pipeline) | 0.3–1 W |
| DRAM reads for framebuffer refresh | 0.1–0.5 W |

These numbers are approximate engineering estimates typical of mid-range laptop panels; OLED, Mini-LED, and low-power panels differ substantially. Battery discharge measurements using tools such as `powertop` and `upower` show that display-related power (backlight + display engine) frequently accounts for 30–50 % of total platform power during light idle workloads.

### The Power Reduction Hierarchy

The kernel and compositor cooperate to apply power reductions in a graduated sequence, ordered from least to most disruptive to the user experience:

1. **Brightness reduction**: Dimming the backlight from 100 % to 30 % can save 1–2 W. Visible to the user but generally tolerable. Controlled by writing to `/sys/class/backlight/*/brightness`.

2. **Dynamic Refresh Rate Switching (DRRS)**: Reducing panel refresh rate from 60 Hz to 40 Hz or lower between frames saves memory bandwidth and panel driver power (≈0.1–0.3 W). Transparent to the user when content is static. Covered in detail in Ch184.

3. **Panel Self-Refresh (PSR)**: The panel buffers the last frame locally and stops requesting data over the eDP link. The display controller clock can gate. Saves ≈0.3–0.8 W. Invisible to the user unless corruption occurs.

4. **Display C-states (DC5/DC6)**: The GPU's display engine enters a deep clock-gated or memory-self-refresh state when PSR is active. Saves an additional ≈0.3–1 W beyond PSR alone. Requires PSR as a precondition on Intel hardware.

5. **Panel off (CRTC disabled)**: The display pipeline is shut down entirely — panel goes dark, GPU display engine is powered off. The user must unlock the screen to resume. Saves the full display power budget.

This chapter addresses all five levels as an integrated system: who drives each step, the kernel interfaces that implement each transition, and how compositors, power daemons, and ACPI interact to coordinate the full idle sequence.

---

## 2. VESA DPMS Legacy Standard

### Historical Background

The VESA Display Power Management Signaling (DPMS) standard was designed in the early 1990s for CRT monitors. A CRT's electron gun requires horizontal and vertical synchronisation pulses to maintain its raster scan. DPMS exploited this: by toggling the presence of HSync and VSync signals the GPU could drive the monitor through four power states without any digital sideband communication.

| DPMS State | HSync | VSync | CRT behaviour |
|-----------|-------|-------|---------------|
| On | Active | Active | Normal operation |
| Standby | Off | Active | Guns off, phosphor cools, HV on |
| Suspend | Active | Off | Deeper sleep |
| Off | Off | Off | Full shutdown |

### DRM DPMS Encoding

The Linux DRM subsystem encodes these four states as integer constants in `include/uapi/drm/drm_mode.h` [Source](https://github.com/torvalds/linux/blob/master/include/uapi/drm/drm_mode.h):

```c
/* bit compatible with the xorg definitions. */
#define DRM_MODE_DPMS_ON       0
#define DRM_MODE_DPMS_STANDBY  1
#define DRM_MODE_DPMS_SUSPEND  2
#define DRM_MODE_DPMS_OFF      3
```

Each DRM connector has a `DPMS` KMS property whose value maps to one of these constants. The kernel registers this property for every connector during `drm_connector_init_only()` in `drivers/gpu/drm/drm_connector.c` [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_connector.c):

```c
drm_object_attach_property(&connector->base,
    config->dpms_property, 0);
```

Legacy drivers implement the `drm_connector_funcs.dpms()` callback to respond to property changes. When userspace writes `DRM_MODE_DPMS_OFF` to the DPMS property the driver calls its `.dpms()` hook, which typically disables the CRTC output and asserts panel power-off GPIO signals.

### Why DPMS Is Obsolescent for Digital Panels

DPMS's HSync/VSync approach is meaningless for HDMI, DisplayPort, and eDP panels. These interfaces carry display enable signals, hot-plug detect lines, and AUX channel messages — they have no analogue sync pulses to suppress. The Standby and Suspend states have no defined meaning for digital panels.

Modern KMS drivers using the atomic modesetting API handle blanking through the `drm_crtc_state.active` boolean (see Section 3). The legacy DPMS property still exists for backward compatibility with X11 applications, but on atomic drivers it is implemented as a thin wrapper that issues an atomic commit with `active = false` when the property is set to any non-ON value.

Legacy userspace tools that still use DPMS include `xset dpms force standby/suspend/off` (X11 only) and `xrandr --dpms`. Wayland compositors do not use the DPMS property path; they use the atomic active property directly.

---

## 3. KMS CRTC Active — Modern Blank/Unblank

### The `active` Boolean

In the KMS atomic API, each `drm_crtc_state` carries an `active` boolean field [Source](https://github.com/torvalds/linux/blob/master/include/drm/drm_crtc.h). When `active = true` the CRTC is enabled: the display pipeline scans out a framebuffer, drives encoders, and sends video to the connector. When `active = false` the driver must:

1. Stop the display engine scanout.
2. Issue panel-off sequences (for eDP: T10–T12 power-down timing, see Ch184).
3. Disable encoder clocks.
4. Allow the GPU's power management to gate the display power domain.

Setting `enabled = false` additionally tears down the mode, while `active = false` with `enabled = true` leaves the mode parameters intact so unblank can be fast.

### Atomic Commit for Blanking

A compositor that wants to blank a display constructs an atomic request, sets the CRTC `ACTIVE` property to 0, and submits it. Here is a representative C snippet using the libdrm API:

```c
#include <xf86drm.h>
#include <xf86drmMode.h>

int blank_crtc(int drm_fd, uint32_t crtc_id, uint32_t active_prop_id)
{
    drmModeAtomicReq *req = drmModeAtomicAlloc();
    if (!req)
        return -ENOMEM;

    /* Set CRTC ACTIVE property to 0 (blank) */
    drmModeAtomicAddProperty(req, crtc_id, active_prop_id, 0);

    int ret = drmModeAtomicCommit(drm_fd, req,
                                   DRM_MODE_ATOMIC_NONBLOCK |
                                   DRM_MODE_PAGE_FLIP_EVENT,
                                   NULL);
    drmModeAtomicFree(req);
    return ret;
}
```

To unblank, the compositor repeats the commit with the `ACTIVE` property value set to 1. The driver re-executes the panel power-on sequence (T1–T9 for eDP), re-enables the display engine clocks, and resumes scanout from the current framebuffer.

### wlroots DRM Backend Implementation

The wlroots compositor library implements display blank/unblank in its DRM backend at `backend/drm/drm.c` [Source](https://gitlab.freedesktop.org/wlroots/wlroots/-/blob/master/backend/drm/drm.c). When the compositor's idle manager fires (Section 9), it calls `drm_connector_set_dpms()` on the relevant output, which builds an atomic commit setting the CRTC `active` state to false. The commit is submitted with `DRM_MODE_ATOMIC_NONBLOCK` so the compositor event loop is not blocked while the panel sequences down.

On resume, when the user moves the mouse or presses a key, the compositor's input handler triggers a matching unblank commit. The DRM driver re-runs the panel power-on sequence, which for a typical eDP panel involves waiting for the T3 (panel power stable) and T7 (backlight-enable assertion) timing periods before declaring the display ready.

---

## 4. Intel Display C-States

Intel's display power architecture uses a hierarchy of Display C-states (DC states) managed by the DMC (Display MicroController) firmware, which runs on a dedicated microcontroller embedded in the GPU die. DC states progressively gate clocks and power rails in the display subsystem when the hardware determines conditions allow it.

### DC State Hierarchy

The i915 driver defines the following target DC states in `drivers/gpu/drm/i915/display/intel_display_power.c` [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_display_power.c):

| Constant | Meaning | Approximate Additional Savings |
|----------|---------|-------------------------------|
| `DC_STATE_DISABLE` | Display engine fully active | Baseline |
| `DC_STATE_EN_DC3CO` | DC3 Clock Off — display pipe clock gated between video frames for Type-C connector idle | ≈0.1 W (Note: the "CO" suffix refers to clock-off/context-off behaviour, not Type-C exclusively; see Tiger Lake implementation details) |
| `DC_STATE_EN_UPTO_DC5` | Display clocks gated, DRAM stays active | ≈0.3 W |
| `DC_STATE_EN_UPTO_DC6` | DC5 + DRAM self-refresh enabled | ≈0.5–1 W additional |
| `DC_STATE_EN_DC9` | Full display power domain off (used during S0ix/S3) | Full display power budget |

**Note on DC3CO**: The name's "CO" suffix refers to the display pipe clock-off behaviour between frames rather than exclusively to USB-C PHY cold state, though it was initially introduced for Tiger Lake Type-C connector idle during active video playback. Verify exact semantics against your target SoC's display PRM.

### Power Well Framework

The display engine is partitioned into power wells — independently controllable power domains. The driver tracks power domain reference counts via `intel_display_power_get()` and `intel_display_power_put()`. When all references to a power domain are dropped, the hardware can gate that domain's power. The DC state engine in the DMC firmware observes which power domains are active and autonomously enters the deepest allowed DC state when preconditions are met.

```c
/* Acquire a power domain reference — prevents DC state entry */
intel_wakeref_t wref = intel_display_power_get(i915, POWER_DOMAIN_PIPE_A);

/* ... use the display hardware ... */

/* Release: if last reference, hardware may gate this domain */
intel_display_power_put(i915, POWER_DOMAIN_PIPE_A, wref);
```

### Controlling DC States via Module Parameter

The `i915.enable_dc` kernel module parameter controls the maximum allowed DC state. The current values (verified from the driver source) are [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_display_power.c):

| Value | Effect |
|-------|--------|
| `-1` | Auto (default): the driver selects the deepest state the hardware supports |
| `0` | Disable all DC states |
| `1` | Allow up to DC5 |
| `2` | Allow up to DC6 |
| `3` | Allow DC5 with DC3CO |
| `4` | Allow DC6 with DC3CO |

The driver's `get_allowed_dc_mask()` function further restricts these based on the platform generation and whether the DMC firmware supports the requested state.

### DC State Management Functions

Key functions in `intel_display_power.c`:

- `intel_display_power_set_target_dc_state()` — requests a target DC state; the driver validates preconditions and programs the DMC.
- `sanitize_target_dc_state()` — clamps the target to what the hardware and module parameter allow.
- `intel_display_power_get_current_dc_state()` — queries the current active DC state.

### DC5 and DC6 Preconditions

The hardware enforces strict preconditions before entering DC5 or DC6. All active display pipes must be in PSR mode (or pipes must be off). The `is_dc5_dc6_blocked()` function checks:

1. No active display pipes that are not in self-refresh mode.
2. VBlank interrupts must be disabled on all active pipes.
3. The DMC firmware must be loaded and healthy.

DC6 adds the requirement that DRAM can be placed in self-refresh. Because DC6 gates the display engine's access to DRAM, it is incompatible with PSR2 Selective Update (which needs to read the dirty region from DRAM to construct update packets). Therefore **DC6 is only reachable during PSR1** (full-frame self-refresh where the panel holds the entire framebuffer); PSR2 restricts the system to at most DC5. See also Section 6.

### DC9 and Display Power-Off

`DC_STATE_EN_DC9` represents complete power removal from the display power domain. It is entered only when the display is fully disabled (CRTC `active = false`, panel off), typically as part of the S0ix suspend sequence described in Section 10. DC9 entry is triggered by the runtime PM framework shutting down the GPU, not by the DMC autonomously.

### Debugging DC State Transitions

The i915 driver exposes DC state information via debugfs. On a running system:

```bash
# Show current allowed DC state (0=off, 1=DC5, 2=DC6, ...)
cat /sys/kernel/debug/dri/0/i915_display_info

# Show power domain reference counts (non-zero = domain is held awake)
cat /sys/kernel/debug/dri/0/i915_power_domain_info

# Show PSR status (must be active for DC5/DC6)
cat /sys/kernel/debug/dri/0/i915_edp_psr_status
```

A common scenario during power regression analysis is that DC6 residency is lower than expected. To diagnose, check `i915_power_domain_info` for any power domain with a non-zero reference count while the display is supposed to be idle. The `POWER_DOMAIN_AUX_*` domains held by the AUX channel polling loop are a frequent offender — if the driver is actively polling AUX registers (e.g., for DP MST topology discovery) the display power domains remain held and DC state entry is blocked.

The `intel_gpu_top` utility from the `intel-gpu-tools` package renders a live view of display engine utilisation. When the system is in DC5 or DC6 the display engine row shows 0 % utilisation. Watching `intel_gpu_top` while the system is idle provides a quick smoke test that DC states are engaging correctly:

```bash
sudo intel_gpu_top
# Look for "Display" row in the engine utilisation table
# Should drop to 0 when PSR is active and DC5/DC6 engages
```

### Platform-Specific DC State Notes

**Skylake/Kaby Lake (Gen9)**: DC5 and DC6 are supported; DC3CO is not. PSR1 only; no PSR2 support on early Gen9 silicon. DC6 entry requires all power wells other than PW1 to be off.

**Ice Lake/Tiger Lake (Gen11/12)**: DC3CO introduced. Tiger Lake supports DC3CO during active video with USB-C connectors. PSR2 and DC6 are both supported but mutually exclusive as described in Section 6. Tiger Lake also introduced the `POWER_DOMAIN_TRANSCODER_EDP_VDSC` domain for eDP DSC (Display Stream Compression).

**Alder Lake / Raptor Lake (Gen12.5+)**: DC6 support improved. Multi-display configurations can now enter DC6 on some pipe combinations. Panel Replay support introduced alongside PSR improvements.

**Meteor Lake / Arrow Lake**: Display engine moved to a separate tile (Display IPU) connected via an inter-die fabric. DC power state management involves cross-tile power domain coordination. The `intel_display_power.c` paths for these platforms use tile-aware power well descriptors.

---

## 5. AMD DCN Display Power Management

AMD's Display Core Next (DCN) architecture, used in Ryzen integrated GPUs and discrete RDNA graphics, implements its own hierarchical display power management through the AMD Display Core (DC) software stack in `drivers/gpu/drm/amd/display/`.

### DCN Power Hierarchy

The Display Core Next hardware exposes several independently clockable and gateable blocks:

- **DCCG** (Display Clock Generator): Distributes clocks to all display controller sub-blocks. Gating DCCG stops the entire display engine.
- **DISPCLK**: The pixel clock domain for the display pipeline stages.
- **DCFCLK**: The Display Fabric Clock — the bus clock connecting the display controller to the memory fabric (DRAM interface).
- **SOCCLK**: The SoC/interconnect clock; the SMU (System Management Unit) can lower this during display idle.

The AMD Display Mode Library (DML), found in `drivers/gpu/drm/amd/display/dc/dml/`, calculates the minimum clock and bandwidth requirements for a given display configuration. DML outputs feed into SMU requests for the appropriate voltage and frequency operating points. This ensures the display engine only requests as much power as the active modeset requires [Source](https://docs.kernel.org/gpu/amdgpu/display/dcn-overview.html).

### MALL (Memory Attached Last Level) and SubVP

AMD's APUs and recent discrete GPUs include an on-die **MALL** — the Memory Attached Last Level cache (also known as Infinity Cache). On display-capable APUs such as the Phoenix/Hawk Point generation, the MALL can be 32–64 MB in size and is shared between the GPU shader engines and the display controller.

The **SubVP** (Sub-ViewPort) feature exploits the MALL for display power saving. When SubVP is active, the display controller pre-fetches the active scanout region into the MALL before each frame. During the scanout interval the display controller reads pixels from the MALL rather than DRAM, allowing the system memory controller to enter self-refresh mode. This is AMD's equivalent of PSR's local frame buffering, but implemented in the SoC's last-level cache rather than in the panel's memory.

SubVP is managed by the AMD display allocation logic. The `dc_debug_options` structure (exposed via debugfs at `/sys/kernel/debug/dri/0/`) includes the following SubVP-related flags [Source](https://docs.kernel.org/gpu/amdgpu/display/dc-debug.html):

- `force_disable_subvp` — disables SubVP entirely; useful for debugging flickering artefacts attributable to MALL cache coherency.
- `force_subvp_mclk_switch` — forces the memory clock switch path through SubVP for testing.
- `force_subvp_num_ways` — overrides the number of MALL cache ways allocated to SubVP.

The primary function responsible for selecting SubVP pipe split strategies in DCN3.2 silicon is located in `drivers/gpu/drm/amd/display/dc/dcn32/` (Note: the exact function name `dcn32_find_subvp_pipe_split_strategy()` should be verified against the current kernel source, as internal function names change between releases).

### AMD Power Gating Module Parameters

The `amdgpu` driver exposes several module parameters relevant to display power management [Source](https://docs.kernel.org/gpu/amdgpu/module-parameters.html):

```
amdgpu.pg_mask=<hex>   # Override power gating features (0 = disable all PG)
amdgpu.dc=-1/0/1       # Disable/enable Display Core driver (-1 = auto)
amdgpu.dcfeaturemask=<hex>  # Override DC feature flags (see DC_FEATURE_MASK)
amdgpu.dpm=-1/0/1      # Override dynamic power management (-1 = auto)
```

Setting `amdgpu.pg_mask=0` disables all power gating including display power gating, which is sometimes useful for diagnosing display corruption caused by aggressive power state transitions.

### SMU Voltage Rail Integration

The AMD display engine communicates with the System Management Unit (SMU) via mailbox messages to request voltage and frequency changes as the display workload changes. When all display pipes go idle (PSR active, SubVP filling MALL), the DC stack sends an SMU message to lower DCFCLK and SOCCLK to their idle operating points, and the SMU gates the memory controller clock if DRAM self-refresh is possible. This coordination is handled in the DM (Display Manager) layer at `drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c` [Source](https://docs.kernel.org/6.2/gpu/amdgpu/display/display-manager.html).

The DMUB (Display Micro-controller UBication) firmware, analogous to Intel's DMC, runs on a dedicated micro-controller and handles PSR, MALL prefetch, and ABM (Adaptive Backlight Management) autonomously. DMUB trace groups including `MALL`, `PSR`, `ABM`, and `ALPM` can be enabled via `/sys/kernel/debug/dri/0/amdgpu_dm_dmub_trace_mask` for debugging.

### AMD Display Power Debugging

AMD provides several debugfs paths for display power investigation:

```bash
# Visual confirmation of pipe allocation (shows SubVP pipes in different colour)
echo 1 | sudo tee /sys/kernel/debug/dri/0/amdgpu_dm_visual_confirm

# Capture Display Test Next log (DML bandwidth and pipe allocations)
cat /sys/kernel/debug/dri/0/amdgpu_dm_dtn_log

# Read DMUB trace buffer (requires trace mask set first)
echo 0x100 | sudo tee /sys/kernel/debug/dri/0/amdgpu_dm_dmub_trace_mask  # enable MALL group
cat /sys/kernel/debug/dri/0/amdgpu_dm_dmub_tracebuffer

# Firmware version (confirms DMUB version for PSR/MALL feature availability)
cat /sys/kernel/debug/dri/0/amdgpu_firmware_info
```

If SubVP is suspected to cause display corruption, setting `force_disable_subvp = true` in `dc_debug_options` via a custom kernel build or module parameter is the standard diagnostic step. On production systems, `amdgpu.dcfeaturemask` can be used to disable specific DC features at boot time.

### DCN Display Mode Library (DML)

A distinctive aspect of AMD's display power management is the Display Mode Library (DML), located in `drivers/gpu/drm/amd/display/dc/dml/`. The DML is a static library — essentially a port of AMD's internal specification and validation code — that calculates the memory bandwidth, latency tolerance, and clock requirements for any given display configuration.

The DML operates on a descriptor of all active pipes (resolution, pixel format, rotation, scaling ratios, refresh rate) and returns the minimum DCFCLK, DISPCLK, DPPCLK, and memory bandwidth (BWBND) values needed to sustain that configuration without underruns. These values are then passed to the SMU as power level requests. The DML is versioned per DCN generation (DML 2.1 for DCN 2.1, DML 3.2 for DCN 3.2, etc.) and is one of the largest and most complex components in the display driver.

---

## 6. PSR System-Level View

This section describes PSR's role within the broader system power management picture. For the eDP-level mechanics of PSR1, PSR2, and Panel Replay, see Ch184.

### PSR as a DC-State Enabler (Intel)

On Intel platforms, Panel Self-Refresh is not merely a display feature — it is a **prerequisite for entering DC5 and DC6**. When PSR1 is active, all display pipe activity from the GPU's perspective ceases: the display engine stops updating the framebuffer pointer and the panel feeds from its local memory. From the power well's point of view all display domains except the panel power itself are idle, satisfying the preconditions for DC5 entry.

The relationship is enforced in `intel_psr_notify_dc5_dc6()` and validated by `is_dc5_dc6_blocked()` in `drivers/gpu/drm/i915/display/intel_psr.c` [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_psr.c). PSR activation is therefore a critical link in the power reduction chain:

```
Active display → PSR idle frames elapsed → PSR1 active
                                          ↓
                              is_dc5_dc6_blocked() = false
                                          ↓
                              DMC enters DC5 (clock gating)
                                          ↓
                              (if DRAM idle) → DC6 (DRAM self-refresh)
```

### PSR2 and DC6 Incompatibility

PSR2 Selective Update (SU) allows the GPU to send only the changed regions of the frame to the panel, rather than a full frame. This requires DRAM access to read the dirty region delta — but DC6 requires DRAM to be in self-refresh with no access requests from the display engine. These requirements are mutually exclusive:

- **PSR1 active** → No DRAM access from display → DC6 achievable.
- **PSR2 SU active** → Display engine reads dirty rectangle from DRAM → DC6 blocked; DC5 at most.

The driver therefore limits the target DC state to `DC_STATE_EN_UPTO_DC5` when PSR2 is active. This trade-off is intentional: PSR2 reduces bandwidth and latency for partial-screen updates (important for cursor movement and typing), while PSR1 enables the deeper DC6 state during extended full-screen idle [Source](https://www.mail-archive.com/intel-gfx@lists.freedesktop.org/msg346528.html).

### Panel Replay and Error Recovery

The eDP 1.5 Panel Replay feature (see Ch184) improves on PSR2 with CRC-gated activation, allowing the panel to verify frame integrity before entering self-refresh. Panel Replay also has better error recovery: if a CRC mismatch is detected the panel exits self-refresh and requests a re-send rather than displaying corrupted content.

### PSR Error Recovery and Watchdog

PSR errors (AUX communication failures, panel-side CRC errors) are signalled via the display engine's interrupt subsystem. The `intel_psr_irq_handler()` function in `intel_psr.c` handles the error interrupt: it masks the error IRQ to prevent an interrupt storm, then queues the `intel_dp->psr.work` work item to evaluate the error asynchronously. The work item may disable PSR for a cooldown period before attempting re-activation.

**Note: needs verification** — specific timeout values (e.g., a 200 ms panel-exit threshold or a 60-second cooldown) are implementation details that vary by driver version and platform. Check `intel_psr.c` at your target kernel tag for the current `psr_compute_idle_frames()` and error cooldown parameters.

### AMD PSR and MALL Interaction

On AMD APUs, PSR and SubVP/MALL complement each other: PSR keeps the eDP link idle while MALL prefetch keeps pixels accessible without DRAM. Together they can sustain the display engine in a state where both the eDP link power and DRAM power are minimised simultaneously, analogous to Intel's DC6+PSR1 combination.

### Panel Replay Power Considerations

eDP 1.5 Panel Replay (see Ch184 for protocol details) changes the power management calculus in important ways:

1. **CRC-gated activation**: Rather than relying on a simple idle-frame counter, Panel Replay activates only when the panel confirms via CRC that its locally buffered frame matches the source. This eliminates the brief periods of incorrect display that PSR1 could suffer during entry and exit.

2. **Improved DC state compatibility**: Panel Replay's CRC handshake reduces the number of spurious exits that earlier PSR implementations suffered, which means DC5/DC6 residency improves on Panel Replay-capable systems. Fewer unexpected PSR exits means fewer DC state exit/re-entry cycles.

3. **ALPM (Adaptive Link Power Management)**: Panel Replay commonly pairs with ALPM to put the eDP link in a low-power standby state between panel refresh bursts. ALPM is tracked via DMUB ALPM trace group on AMD and via dedicated registers on Intel (visible in `i915_edp_psr_status` debugfs). ALPM further reduces eDP PHY power during the self-refresh window.

4. **vblank and DC6 interaction**: A known issue (addressed in kernel patches around 2024–2025) is that when Panel Replay is active with vblank interrupts still enabled — as they would be if userspace holds a vblank reference — DC6 entry is blocked. The driver sets the target DC state to `DC_STATE_EN_UPTO_DC5` to prevent DC6 in this scenario [Source](https://www.mail-archive.com/intel-gfx@lists.freedesktop.org/msg346528.html). Ensuring that vblank is disabled when the display is idle is therefore important for achieving maximum DC6 residency on Panel Replay platforms.

---

## 7. Backlight Power Management and ALS

### Sysfs Backlight Interface

Every registered backlight device appears under `/sys/class/backlight/`. For Intel-native backlight control the path is typically `/sys/class/backlight/intel_backlight/`; for AMD systems it may be `/sys/class/backlight/amdgpu_bl0/`. The standard attributes are:

```
/sys/class/backlight/<name>/
    brightness        # Current level (0..max_brightness); writable
    max_brightness    # Maximum level (read-only)
    actual_brightness # What the hardware reports (may differ)
    type              # "firmware", "raw", or "platform"
```

A compositor implementing idle dimming reads `max_brightness` once, then writes a reduced brightness value via timer steps:

```c
/* Read max brightness */
int max_bl = read_sysfs_int("/sys/class/backlight/intel_backlight/max_brightness");

/* Dim to 30% over 2 seconds in 10 steps */
for (int step = 1; step <= 10; step++) {
    int target = (max_bl * (100 - 70 * step / 10)) / 100;
    write_sysfs_int("/sys/class/backlight/intel_backlight/brightness", target);
    usleep(200000); /* 200ms per step */
}
```

### ACPI Video Backlight vs. Native Backlight

Two backlight control paths coexist in the kernel. The ACPI video driver (`drivers/acpi/video.c`) implements backlight control via ACPI control methods [Source](https://docs.kernel.org/firmware-guide/acpi/video_extension.html):

- **`_BCL`** (Brightness Control Levels): Defines available brightness steps. The first two entries are AC/battery power thresholds (unused by Linux); the remaining entries are enumerated levels.
- **`_BCM`** (Brightness Control Method): Called on brightness write to program the firmware.
- **`_BQC`** (Brightness Query Current): Queried on brightness read to get the firmware's view of the current level.

The ACPI video backlight appears under `/sys/class/backlight/acpi_videoX` with type `firmware`.

On Windows 8-ready machines the kernel's `video_detect.c` automatically prefers native GPU backlight interfaces over the ACPI video interface. When a native backlight device registers, `acpi_video_unregister_backlight()` is called to remove the ACPI interface, avoiding conflicts. The `acpi_backlight=native` or `acpi_backlight=vendor` kernel command-line parameters can override this automatic selection when DMI quirks are needed.

### Ambient Light Sensor Integration

Linux IIO (Industrial I/O) subsystem drivers surface ambient light sensors (ALS) as IIO devices under `/sys/bus/iio/devices/`. The `iio-sensor-proxy` service bridges the IIO subsystem to D-Bus, making ALS data available to desktop sessions [Source](https://gitlab.freedesktop.org/hadess/iio-sensor-proxy).

The proxy exposes the `net.hadess.SensorProxy` D-Bus interface at object path `/net/hadess/SensorProxy` with properties including `HasAmbientLight` (boolean, indicates an ALS is present) and `AmbientLightLevel` (double, lux value). Desktop environments subscribe to property-change signals to adjust backlight brightness:

```bash
# Introspect the proxy (requires iio-sensor-proxy running)
gdbus introspect --system \
    --dest net.hadess.SensorProxy \
    --object-path /net/hadess/SensorProxy
```

**Note**: The specific D-Bus property names `HasAmbientLight` and `AmbientLightLevel` are documented in upstream iio-sensor-proxy but could not be verified from fetched source during preparation of this chapter; confirm against the current API documentation at the project repository.

### GNOME Automatic Brightness

GNOME's automatic backlight management has evolved significantly. In GNOME 49 (2025), backlight control moved from `gnome-settings-daemon` into `mutter`, making the compositor the single authority for hardware brightness. GNOME Shell handles the ALS logic and receives ALS level signals from `gnome-settings-daemon`. The `mutter` implementation supports configurable brightness curves, including vendor-defined custom mappings between lux and brightness levels. Earlier GNOME versions used a simpler linear mapping [Source](https://blog.sebastianwick.net/posts/gnome-49-backlight-changes/).

### OLED Burn-In Prevention

OLED panels require additional idle-power considerations beyond brightness control. Static content at high brightness causes differential OLED degradation (burn-in). Common mitigation strategies include:

- **Pixel shift**: Periodically translating the active image by 1–2 pixels. Compositors and panel firmware implement this via DPCD registers or a dedicated kernel timer.
- **Brightness reduction on static content**: Detecting when the framebuffer has been unchanged for N seconds and reducing backlight/OLED brightness, then restoring on user input.
- **Screen dimming before DPMS off**: Fading to black over 1–2 seconds rather than an abrupt blank, reducing the visual impact of high-brightness OLED pixels cut off mid-state.

---

## 8. power-profiles-daemon Display Integration

### Overview

`power-profiles-daemon` is a D-Bus service that exposes hardware power profiles to the desktop session [Source](https://gitlab.freedesktop.org/hadess/power-profiles-daemon). It writes to the kernel `platform_profile` sysfs interface (`/sys/firmware/acpi/platform_profile`) to request platform-level power states, and coordinates driver-specific power settings.

### D-Bus Interface

The daemon exposes the `net.hadess.PowerProfiles` D-Bus interface at `/net/hadess/PowerProfiles` with:

- **`ActiveProfile`** (string, read-write): The currently active profile; one of `"power-saver"`, `"balanced"`, or `"performance"`.
- **`Profiles`** (array of dicts, read-only): Available profiles with metadata including `Profile` (name), `Driver` (e.g., `"platform_profile"`), and `CpuDriver`.
- **`PerformanceDegraded`** (string, read-only): Indicates if the performance profile is constrained by thermal limits.

```bash
# Switch to power-saver profile
powerprofilesctl set power-saver

# Check current profile via D-Bus
gdbus call --system \
    --dest net.hadess.PowerProfiles \
    --object-path /net/hadess/PowerProfiles \
    --method org.freedesktop.DBus.Properties.Get \
    net.hadess.PowerProfiles ActiveProfile
```

### Display-Specific Actions

When `power-saver` is activated, `power-profiles-daemon` triggers a set of display-relevant changes:

1. **`platform_profile` = `"low-power"`**: This sysfs write is picked up by platform firmware (ACPI `_DSM` method) to engage OEM power-saving features, which may include reducing display panel power.

2. **AMDGPU panel power savings**: Version 0.20 added support for writing to the AMDGPU display panel power-saving sysfs entry for Ryzen APU laptops, reducing display panel power consumption at the cost of minor colour accuracy reduction. This is a hardware feature exposed via a sysfs attribute under the DRM device.

3. **DPM clock reduction**: The daemon can request lower Dynamic Power Management (DPM) clock levels on AMDGPU, reducing display engine fabric clocks during power-saver operation.

### Profile Mapping for Display Features

| Profile | DRRS behaviour | PSR idle timeout | Platform profile |
|---------|----------------|------------------|-----------------|
| `performance` | DRRS disabled (fixed high refresh) | Default (driver-set) | `performance` |
| `balanced` | DRRS enabled, default thresholds | Default | `balanced` |
| `power-saver` | DRRS enabled, reduced switch threshold | Reduced (faster PSR entry) | `low-power` |

**Note**: The specific PSR idle timeout manipulation by `power-profiles-daemon` (e.g., directly writing to PSR debugfs) is not well-documented in the daemon's own source; verify against current upstream whether the daemon sets PSR-specific parameters or relies purely on `platform_profile` sysfs to trigger firmware-level adjustments.

### Conflict with TLP

`TLP` (ThinkPad Linux Power control) and `power-profiles-daemon` overlap significantly in scope. Running both simultaneously can result in one overwriting the other's settings. The canonical recommendation is to choose one daemon and disable the other. On `systemd`-based distributions, `power-profiles-daemon` is preferred for desktop environments that integrate with it (GNOME, KDE Plasma), while `TLP` remains popular for headless or custom configurations.

---

## 9. Compositor Display Idle Management

### The Idle Notification Protocol

Wayland compositors implement display idle management through the `ext-idle-notify-v1` protocol, which entered the `wayland-protocols` staging area and has been widely adopted [Source](https://wayland.app/protocols/ext-idle-notify-v1).

The protocol defines two interfaces:

**`ext_idle_notifier_v1`** — the manager interface:
- `get_idle_notification(timeout_ms, seat)` → `ext_idle_notification_v1`: Creates a notification object that fires when the seat has been inactive for `timeout_ms` milliseconds. Respects idle inhibitors.
- `get_input_idle_notification(timeout_ms, seat)` → `ext_idle_notification_v1`: Like above but ignores idle inhibitors; tracks raw input absence only.

**`ext_idle_notification_v1`** — the per-timer notification object:
- Event `.idled()`: Fired when the timeout elapses without seat activity.
- Event `.resumed()`: Fired when user input resumes after an idle period.

### wlroots Implementation

wlroots exposes `wlr_idle_notifier_v1_init()` to initialise the idle notifier on the compositor's Wayland display. Compositors should call the notifier's activity function whenever an input event arrives on a seat [Source](https://sergiogdr.pages.freedesktop.org/wlroots/wlr/types/wlr_idle_notify_v1.h.html):

```c
/* In the compositor's input handler: */
wlr_idle_notifier_v1_notify_activity(server.idle_notifier, seat);
```

The compositor itself registers idle notifications for its internal timers:

```c
/* Create a 2-minute dimming timer */
struct wlr_idle_notification_v1 *dim_notif =
    wlr_idle_notification_v1_create(notifier, seat, 120000 /* ms */);

wl_signal_add(&dim_notif->events.idle, &server.on_idle_dim);
wl_signal_add(&dim_notif->events.resumed, &server.on_resumed);
```

### Idle Action Sequence

A well-behaved compositor implements a staged idle sequence. The following is representative of the sequence used by wlroots-based compositors (exact timings are compositor-configured):

1. **T+0: User activity stops.**

2. **T+2 min: Dimming timer fires** (`idled` event). Compositor writes reduced brightness to `/sys/class/backlight/*/brightness` over a 2-second fade (10 × 200 ms steps to 30 % of max).

3. **T+5 min: Blank timer fires** (`idled` event). Compositor issues an atomic DRM commit setting `CRTC ACTIVE = 0` on all outputs. The DRM driver runs the panel-off sequence. On Intel hardware: PSR exits → display engine shuts down → DC9 entry pathway opens. GPU enters runtime suspend.

4. **User input resumes**: Compositor receives input event → GPU runtime-resumes → DRM driver re-programs display engine → atomic commit with `CRTC ACTIVE = 1` → panel power-on sequence → PSR re-arms.

### Idle Inhibitors

The compositor must respect idle inhibitors that prevent the idle sequence from firing:

- **`xdg-inhibit-idle`** (via `zwp_idle_inhibit_manager_v1` or the newer `ext_idle_notify_v1` inhibitor semantics): Applications such as video players and presentation software assert an idle inhibitor while active. The compositor must not dim or blank while any inhibitor is held.
- **systemd inhibitor locks** (`systemd-inhibit`): Package managers, firmware updaters, and backup daemons assert `systemd` inhibitors via `org.freedesktop.login1.Manager.Inhibit`. Compositors that integrate with `logind` must honour these.
- **Audio playback inhibition**: Some compositors treat active audio output as an implicit idle inhibitor, though this is not standardised.

### Desktop Environment Idle Stacks

- **GNOME**: `gnome-session` tracks session idle state via `org.gnome.Mutter.IdleMonitor`, which wraps `ext-idle-notify-v1`. GNOME Shell handles the dim/blank sequence. The GNOME Power Manager settings panel configures timeouts via `org.gnome.settings-daemon.plugins.power` GSettings keys such as `sleep-inactive-ac-timeout` and `sleep-inactive-battery-timeout`.
- **KDE Plasma**: `PowerDevil` is KDE's power management daemon. It subscribes to `org.freedesktop.login1` session idle signals and manages screen dimming, DPMS, and suspend independently of the compositor for flexibility across both X11 and Wayland sessions.
- **sway/swayidle**: `swayidle` is the reference idle management daemon for sway, consuming `ext-idle-notify-v1` notifications and executing configurable commands for dim, lock, and DPMS actions. Compatible with any compositor implementing the protocol [Source](https://github.com/swaywm/swayidle).

### systemd-logind and D-Bus Session Idle

On `systemd`-based systems, `logind` tracks session idle state and exposes it via the `org.freedesktop.login1.Session` D-Bus interface. The `IdleHint` property reflects whether the session is considered idle. Compositors and display managers set this property by calling the `SetIdleHint(bool)` method when their own idle timers fire.

```bash
# Inspect session idle state via D-Bus
gdbus call --system \
    --dest org.freedesktop.login1 \
    --object-path /org/freedesktop/login1/session/auto \
    --method org.freedesktop.DBus.Properties.Get \
    org.freedesktop.login1.Session IdleHint
```

The logind idle state propagates to `systemd-sleep` targets. When `logind` marks the session idle and the configured `IdleAction` is `suspend`, `systemd` initiates the suspend sequence — which in turn requires the display to be off as described in Section 10.

### swaylock and Screen Blanking Coordination

When a compositor blanks the screen (CRTC `active = false`) it typically also engages a screen locker. The interaction is:

1. Idle timer fires → compositor calls `ext-session-lock-v1` (the Wayland session lock protocol) to activate the lock surface.
2. Lock surface rendered → compositor blanks display (atomic commit `ACTIVE = 0`).
3. On resume → compositor receives CRTC re-enable event → redraws lock surface → waits for user authentication → unlocks session → normal compositing resumes.

This sequence ensures the lock surface is always rendered before the display goes dark, preventing brief glimpses of the unlocked desktop on resume.

---

## 10. ACPI S0ix Modern Standby and Display Requirements

### S0ix vs. S3 Suspend

Modern laptops with ACPI Modern Standby use S0ix (suspend-to-idle) rather than traditional S3 (suspend-to-RAM). S0ix keeps the system in a partially-on state allowing network connectivity and background tasks, while progressively deepening CPU and device idle states.

The kernel selects the suspend mode via `/sys/power/mem_sleep` [Source](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate):

```bash
cat /sys/power/mem_sleep
# Output: s2idle [deep]   ([] = current selection)

# Switch to S0ix (s2idle):
echo s2idle | sudo tee /sys/power/mem_sleep

# Switch to S3 (traditional suspend-to-RAM, if supported):
echo deep | sudo tee /sys/power/mem_sleep
```

On most modern Intel and AMD laptops the firmware exposes only S0ix (`s2idle`), with S3 disabled in the ACPI tables.

### Display Shutdown Requirements for S0ix Entry

S0ix entry requires the display subsystem to be fully shut down. The sequence is:

1. **Compositor blanks display**: Atomic commit with `CRTC ACTIVE = 0`. Panel sequences off (T10–T12 eDP timing).

2. **Display engine shuts down**: PSR exits cleanly. Display power domains are released. On Intel: DC9 entry. On AMD: display engine clocks gate, DMUB firmware suspends.

3. **GPU runtime suspend**: The GPU driver's `.runtime_suspend()` callback runs. For i915: `intel_runtime_pm_disable_interrupts()`, display power domain off, GGTT save. For amdgpu: `amdgpu_device_suspend()` → display subsystem suspend → `amdgpu_acpi_smart_shift_update()`.

4. **CPU enters low-power idle**: With all devices in runtime suspend, the CPU's platform idle driver can enter the deepest S0ix sub-state (PC10 on Intel, Package C-state on AMD).

### S0ix Resume Sequence

```
CPU wakes from S0ix (interrupt: lid, keyboard, RTC alarm)
    ↓
GPU runtime-resumes:
    i915: intel_runtime_pm_enable_interrupts()
    Display power domains re-acquired
    DMC firmware re-loaded / DC9 exit
    ↓
DRM driver re-programs display engine:
    CRTC mode restored from saved state
    eDP panel power-on sequence (T1–T9)
    PSR re-armed
    ↓
Atomic commit with CRTC ACTIVE=1
    ↓
Compositor receives DRM page-flip event, resumes rendering
```

### Debugging S0ix Display Issues

When S0ix entry fails due to display-related issues, the kernel's PM debug infrastructure provides diagnostics:

```bash
# Enable verbose PM debug messages
echo 1 | sudo tee /sys/power/pm_debug_messages

# Check suspend statistics (failed device entries)
cat /sys/kernel/debug/suspend_stats

# Attempt suspend and capture dmesg
dmesg -C
echo s2idle | sudo tee /sys/power/mem_sleep
echo mem | sudo tee /sys/power/state
dmesg | grep -i "PM\|drm\|i915\|amdgpu" | tail -50
```

Common failure indicators in `dmesg`:

```
PM: Device drm:card0 failed to suspend with error -22
i915: DC9 entry preconditions not met: display still active
amdgpu: display subsystem suspend failed
```

The `intel_gpu_top` tool (from `intel-gpu-tools`) shows the display engine's power consumption in real time. When DC9 is active, display engine counters should reach zero. If they do not, a display pipe is still active and preventing deep idle entry.

On AMD systems, `umr` (the AMD GPU debugging tool) can interrogate display registers to identify which DCN block is preventing clock gating. The `amdgpu_dm_dtn_log` debugfs file captures the Display Test Next log showing the current hardware configuration of all active pipes.

### S0ix Checklist for Display Engineers

A platform that passes S0ix validation with display correctly powered down should satisfy:

- [ ] `CRTC ACTIVE = 0` committed and confirmed via DRM event before `echo mem`.
- [ ] No residual IRQs from the display engine (check `/proc/interrupts` for display IRQ line).
- [ ] GPU in runtime suspend: `/sys/bus/pci/devices/<gpu>/power/runtime_status` = `suspended`.
- [ ] Platform reports S0ix residency increase after resume: `cat /sys/kernel/debug/pmc_core/slp_s0_residency_usec` (Intel) or AMD S0ix debug tool.
- [ ] Panel correctly powers up on resume without requiring a display link re-training timeout.

---

## 11. Integrations

- **Ch2 (KMS and DRM Architecture)**: The `drm_crtc_state.active` boolean is the primary mechanism for display blank/unblank in atomic KMS. The DPMS property described in this chapter is a backward-compatibility wrapper around the atomic active property.

- **Ch3 (Atomic Modesetting in Depth)**: PSR2 and Panel Replay are advanced KMS display features whose power state interactions with DC5/DC6 are orchestrated by the mechanisms described in this chapter. The atomic commit flow for CRTC active transitions is covered in Ch3.

- **Ch21 (wlroots Compositor Internals)**: wlroots's idle management implementation (`wlr_idle_notifier_v1`) and DRM backend blanking (`drm_connector_set_dpms()` → atomic commit) are the compositor-side complement to the kernel interfaces described in Sections 3 and 9.

- **Ch22 (KDE Plasma and GNOME Shell)**: KDE PowerDevil and GNOME gnome-session implement the idle action stacks (dim → lock → blank → suspend) described in Section 9. The `org.gnome.Mutter.IdleMonitor` and `org.freedesktop.login1` inhibitor interfaces used by these desktop stacks are referenced in this chapter.

- **Ch51 (GPU Power Management and Runtime PM)**: Intel Display C-states (DC5/DC6/DC9) and AMD MALL/SubVP power savings interact deeply with the GPU's runtime PM framework. DC9 on Intel requires the GPU to enter runtime suspend; AMD SubVP's DRAM self-refresh is coordinated with the SMU via the same GPU power management infrastructure described in Ch51.

- **Ch126 (Hybrid Graphics and Optimus)**: On Optimus laptops, display power-off (CRTC disabled, DC9 entry) is the event that allows the discrete GPU to be fully powered down via the platform's power gating mechanism (ACPI D3cold). The display idle chain described in this chapter is therefore a prerequisite for dGPU power savings on hybrid graphics systems.

- **Ch184 (Embedded DisplayPort and Laptop Panel Management)**: This chapter provides the system-level orchestration for the eDP-specific mechanisms (PSR1/PSR2/Panel Replay, DRRS, panel power sequencing) detailed in Ch184. Understanding both chapters together gives a complete picture of laptop display power management from the panel DPCD registers up through the compositor idle protocol.

## Roadmap

### Near-term (6–12 months)
- Panel Replay adoption is widening from Intel Alder Lake/Raptor Lake into AMD RDNA 3/4 APUs; upstream kernel patches are landing support for Panel Replay on RDNA 4 eDP panels in the `amd-staging-drm-next` branch, with backports expected to stable kernels in late 2026.
- The `ext-idle-notify-v1` Wayland protocol is being extended with a companion `ext-idle-inhibit-v2` proposal to close semantic gaps in how video players and game launchers communicate idle inhibition, with a draft in the `wayland-protocols` staging area as of mid-2026.
- Intel Lunar Lake and Arrow Lake improve cross-tile DC-state coordination between the separate Display IPU and CPU tiles; driver patches adding `POWER_DOMAIN_TRANSCODER_*` descriptors for the new tile topology are under review on the `intel-gfx` mailing list.
- `power-profiles-daemon` 0.22 is adding explicit PSR idle-timeout tuning on Intel platforms (writing to `i915.psr_min_idle_frames` module parameter at runtime), giving the power-saver profile a direct path to faster PSR entry without requiring a kernel reboot.

### Medium-term (1–3 years)
- AMD SubVP/MALL is expected to evolve into a driver-transparent "display hibernation" mode for RDNA 5 APUs, where the DMUB firmware autonomously manages MALL prefetch, PSR, and DRAM self-refresh without any per-frame driver intervention, targeting sub-1 W total display power at idle on 4K panels.
- The KMS uAPI may gain a formal `DRM_CRTC_PROP_SELF_REFRESH_MODE` enum property allowing compositors to express a preference between PSR1 (deep DC6 capable) and PSR2 (low-latency partial updates), replacing the current driver-internal heuristic that selects between the two modes.
- OLED-specific idle management — pixel shift, ABM dimming curves, and burn-in protection timers — is expected to be standardised in a `drm-oled-management-v1` kernel uAPI extension as OLED laptop panels become mainstream across Intel and AMD reference designs.
- `systemd-logind` is being extended with display-device-aware suspend inhibition, allowing devices to register display-specific constraints (e.g., "do not enter S0ix until Panel Replay exit is confirmed") through a new `inhibit-display` lock type, reducing race conditions in the compositor → PM sequencing path.

### Long-term
- As eDP transitions toward embedded DisplayPort 2.1 (UHBR link rates) and next-generation Panel Replay 2.0, the DC-state architecture on both Intel and AMD platforms is expected to merge PSR, ALPM, and display C-states into a single hardware-managed "display power island" FSM that requires no software polling, with the kernel driver role reduced to initial configuration and error recovery.
- The ongoing convergence of SoC display engines (Intel's Display IPU, AMD's DCN, and Qualcomm's MDSS targeting Linux via the `msm` driver) toward unified power management abstractions may eventually produce a generic KMS display-power backend in `drivers/gpu/drm/display/`, analogous to the existing `drm_dp_helper` layer, handling PSR/Panel Replay lifecycle across vendors.
