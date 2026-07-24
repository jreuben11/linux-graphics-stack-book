# Chapter 112: Variable Refresh Rate — FreeSync, G-Sync, and Frame Pacing

**Target audiences:**
- **Game developers** — targeting Linux and Steam Deck
- **Compositor authors** — integrating VRR into display pipelines
- **Display engineers** — validating adaptive sync hardware
- **Application developers** — understanding present mode selection, frame pacing, and the interaction between Vulkan swapchains and kernel KMS properties

---

## Table of Contents

1. [The Tearing and Stuttering Problem](#1-the-tearing-and-stuttering-problem)
   - [1.1 What is Variable Refresh Rate (VRR)?](#11-what-is-variable-refresh-rate-vrr)
   - [1.2 What is FreeSync and Adaptive-Sync?](#12-what-is-freesync-and-adaptive-sync)
   - [1.3 What is Frame Pacing?](#13-what-is-frame-pacing)
2. [Display Refresh Fundamentals](#2-display-refresh-fundamentals)
3. [VRR Technologies: FreeSync and G-Sync](#3-vrr-technologies-freesync-and-g-sync)
4. [KMS VRR Support](#4-kms-vrr-support)
5. [Wayland Presentation Feedback](#5-wayland-presentation-feedback)
6. [Frame Pacing Strategies](#6-frame-pacing-strategies)
7. [Gamescope VRR Implementation](#7-gamescope-vrr-implementation)
8. [Mutter and KWin VRR](#8-mutter-and-kwin-vrr)
9. [Vulkan Swapchain and VRR](#9-vulkan-swapchain-and-vrr)
10. [Measuring and Debugging Frame Timing](#10-measuring-and-debugging-frame-timing)
11. [wp_tearing_control_v1 — Client-Side Tearing Opt-In](#11-wp_tearing_control_v1--client-side-tearing-opt-in)
12. [wp_fifo_v1 — Explicit FIFO Presentation Semantics](#12-wp_fifo_v1--explicit-fifo-presentation-semantics)
13. [Frame Pacing on RDNA3+ — AMD Hardware Frame Pacing](#13-frame-pacing-on-rdna3--amd-hardware-frame-pacing)
14. [VRR on OLED Displays — Burn-In Mitigation and Refresh Rate Floors](#14-vrr-on-oled-displays--burn-in-mitigation-and-refresh-rate-floors)
15. [Explicit Sync Interaction with VRR](#15-explicit-sync-interaction-with-vrr)
16. [Integrations](#16-integrations)

---

## 1. The Tearing and Stuttering Problem

Every frame a GPU renders must eventually appear on a display. The fundamental challenge is that GPU rendering time is variable — it depends on scene complexity, driver overhead, thermal throttling, and hundreds of other factors — while traditional displays refresh at a fixed, immutable rate. This mismatch between variable GPU output and fixed display cadence is the root cause of two of the most visible quality problems in real-time graphics: *tearing* and *stuttering*.

**Tearing** occurs when the GPU writes a new frame into the framebuffer while the display controller is in the middle of scanning out the previous frame. The result is a horizontal discontinuity where the top portion of the screen shows the new frame and the bottom shows the old one. This is especially visible during fast camera pans in games or when scrolling long pages.

**Stuttering** occurs when the GPU cannot sustain the display's fixed refresh rate. At 60 Hz, the display expects a new frame every 16.67 ms. If the GPU takes 20 ms to render a frame, it misses the VBLANK window. The display must then show the previous frame for another full refresh period — 33.33 ms total — while the next frame completes. The sudden doubling of frame duration is perceived as a stutter even if the average frame rate is 50 FPS.

The traditional fix for tearing is **VSync** (vertical synchronization): the application or compositor delays buffer swap until the display reaches its VBLANK interval — the brief period between scan-out cycles when the display is not updating pixel rows. VSync eliminates tearing because the display never reads from a framebuffer that is being written. However, VSync does not solve stuttering; it merely converts the problem. A frame that takes 17 ms (just over one 60 Hz period) must now wait for the *next* VBLANK, producing the same 33.33 ms presentation latency. VSync also introduces up to one full frame of input latency because the application must queue the swap before VBLANK occurs.

**Variable Refresh Rate (VRR)** resolves both problems simultaneously. Rather than the display driving scan-out at a fixed cadence, a VRR-capable display waits for a signal from the GPU before beginning the next refresh cycle. The display adapts its refresh period to match the GPU's actual output timing. If the GPU delivers a frame in 8 ms, the display refreshes in 8 ms; if the next frame takes 12 ms, the next refresh period is 12 ms. Tearing cannot occur because the display only starts scanning a frame after the GPU signals it is complete. Stutter cannot occur because there is no fixed period for a frame to "miss." The GPU and display are effectively locked together by a handshake rather than an external clock.

### 1.1 What is Variable Refresh Rate (VRR)?

Variable Refresh Rate (VRR) is a display technology that allows a monitor to dynamically adjust its refresh period to match the rate at which a GPU delivers completed frames, rather than operating at a fixed frequency such as 60 Hz or 144 Hz. Without VRR, a display drives its own refresh clock independently of the GPU, creating a timing mismatch that produces tearing when frames arrive mid-scan and stutter when the GPU misses a refresh window. VRR eliminates this mismatch by making the display subordinate to the GPU's output timing within a defined operating range, typically expressed as a minimum and maximum Hz (for example, 48–165 Hz). When the GPU signals that a frame is ready, the display controller closes the vertical blanking interval and begins scanning the new frame. If the GPU delivers frames faster than the maximum ceiling, VRR falls back to fixed-rate behavior at that ceiling; if frame time exceeds the minimum period, Low Framerate Compensation or a fixed fallback rate takes over. In the Linux graphics stack, VRR is exposed through the kernel's Direct Rendering Manager (DRM) subsystem via the `VRR_ENABLED` CRTC property and the `vrr_capable` connector property, both manipulated through atomic KMS commits. Compositors such as Gamescope, Mutter, and KWin control VRR state; applications interact through Vulkan swapchain present modes and Wayland presentation protocols. This chapter covers the full path from kernel KMS properties through compositor policy to application-level present mode selection.

### 1.2 What is FreeSync and Adaptive-Sync?

FreeSync is AMD's brand for VRR technology built on top of the VESA Adaptive-Sync specification. VESA Adaptive-Sync, introduced in the DisplayPort 1.2a standard in 2014, defines an open, royalty-free mechanism by which a GPU extends the vertical blanking interval of a DisplayPort link dynamically, deferring the start of the next active video period until the source signals readiness. FreeSync applies AMD's certification and testing requirements on top of this standard, defining tiers — FreeSync, FreeSync Premium, and FreeSync Premium Pro — that impose additional constraints such as minimum refresh rates, Low Framerate Compensation (LFC), and HDR support. Because FreeSync uses the VESA Adaptive-Sync signalling protocol, any VESA Adaptive-Sync display is compatible with AMD GPUs and, critically, with the Linux kernel's KMS VRR infrastructure. The kernel's DRM EDID parser reads the display's advertised VRR range from the Display Range Limits descriptor in the EDID, sets the `vrr_capable` connector property, and exposes this range through connector state. HDMI 2.1 also includes a VRR signalling mechanism that AMD GPUs support through the same FreeSync stack, extending VRR coverage to HDMI connections. NVIDIA's G-Sync Compatible program uses the same underlying VESA Adaptive-Sync protocol, meaning the kernel treats a G-Sync Compatible monitor and a FreeSync monitor identically at the KMS layer. In this chapter, FreeSync and Adaptive-Sync are used interchangeably when discussing kernel-level support, since the kernel interacts with the underlying DisplayPort or HDMI signalling rather than the vendor marketing tier.

### 1.3 What is Frame Pacing?

Frame pacing refers to the regularity with which successive frames are delivered to the display. An application running at a nominal 60 FPS does not automatically produce smooth motion: if individual frame times vary significantly — alternating between 12 ms and 21 ms, for example — the viewer perceives uneven motion even though the average frame rate is acceptable. Frame pacing is the practice of measuring, controlling, and smoothing these inter-frame intervals so that consecutive frames arrive at the display at consistent intervals. Poor frame pacing manifests as micro-stutter: brief, irregular hesitations that are distinct from the gross stutter caused by missed VBLANK windows. VRR partially addresses frame pacing by eliminating the quantization effect of fixed refresh windows, but it does not eliminate frame time variance itself: a frame that takes twice as long as its predecessor still produces visible jitter under VRR because the display refresh period doubles for that one frame. Effective frame pacing therefore requires coordination between the GPU render scheduler, the compositor's present timing logic, and the display's VRR behavior. On Linux, frame pacing strategies include compositor-side pacing using `wp_presentation` feedback timestamps, Vulkan present timing extensions (`VK_GOOGLE_display_timing`), and hardware-assisted mechanisms such as AMD's Hardware Frame Pacing feature on RDNA3+ GPUs. The Linux kernel's DRM infrastructure, the Wayland `wp_fifo_v1` and `wp_tearing_control_v1` protocols, and the Gamescope compositor all implement or expose pacing controls covered in this chapter.

---

## 2. Display Refresh Fundamentals

### VBLANK and the Refresh Cycle

A display's output pipeline is organized around horizontal and vertical scan lines. The display electron gun (CRT) or timing controller (TCON on flat panels) sweeps through rows from top to bottom. After the last pixel row of the active area is emitted, the display enters the **Vertical Blanking Interval (VBLANK)**: a period during which the scan returns to the top of the screen. VBLANK is historically when the display controller swaps the active framebuffer — no pixels are being drawn, so a new frame can be made live without the display reading a partially-written image.

Modern displays do not use electron guns, but the VBLANK concept persists. Display controllers on SoCs and GPU output blocks still implement a programmed vertical front porch, sync pulse, and back porch. The sum of active video time and blanking determines the refresh period.

### Fixed-Rate Constraints and Frame Time Variance

At 60 Hz, each refresh period is exactly 16.67 ms. An application targeting smooth animation must deliver a new frame before every VBLANK. The challenge is that frame time is never constant. A GPU that averages 8 ms per frame on a static scene may spike to 22 ms when loading new geometry, decoding a cutscene, or responding to a driver preemption. A single 22 ms frame on a 60 Hz display takes two refresh periods — 33.33 ms — to present. The user perceives this as a stutter even though the GPU immediately recovers.

### Double and Triple Buffering

**Double buffering** uses two framebuffers: one being scanned out (front buffer) and one being rendered into (back buffer). At VBLANK, the buffers swap. If the GPU finishes before VBLANK, it waits idle. If it finishes after VBLANK, the previous frame is shown again. This is correct and tear-free but stutters whenever frame time exceeds the refresh period.

**Triple buffering** adds a third buffer. The GPU renders into one back buffer while a second waits to be swapped in. If the GPU finishes a frame early, it immediately begins the next frame rather than waiting. At VBLANK, the most recently completed frame is swapped in. Triple buffering reduces stutter frequency because the GPU can "run ahead" and absorb small variations. The cost is one additional frame of latency: the frame currently being rendered may not appear until the *next* VBLANK after the one that swaps in the already-completed frame.

### Vulkan Present Modes

Vulkan exposes the application-level analog of these strategies through present modes, selected at swapchain creation:

- **`VK_PRESENT_MODE_FIFO_KHR`**: Frame submissions queue in a FIFO; each is presented at the next available VBLANK. Equivalent to double-buffered VSync. Guaranteed to be supported on all Vulkan implementations.
- **`VK_PRESENT_MODE_FIFO_RELAXED_KHR`**: Like FIFO, but if the application is late (the queue was empty when VBLANK occurred), the frame is presented immediately — allowing a visible tear rather than an extra refresh of the previous frame. Useful when occasional tearing is preferable to guaranteed stutter.
- **`VK_PRESENT_MODE_MAILBOX_KHR`**: The swapchain holds a single "pending" slot; a newly submitted frame replaces any previously queued but not yet displayed frame. The result is that the display always shows the newest completed frame, but frames may be discarded. This is Vulkan's approximation of triple buffering. On Wayland compositors, mailbox is typically implemented as FIFO because the Wayland protocol requires the compositor to eventually present every submitted buffer.
- **`VK_PRESENT_MODE_IMMEDIATE_KHR`**: No synchronization. The frame is displayed as soon as it is submitted, potentially tearing. Used for benchmarking unconstrained GPU throughput.

[Source: Vulkan Specification — VkPresentModeKHR](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkPresentModeKHR.html)

---

## 3. VRR Technologies: FreeSync and G-Sync

The consumer VRR ecosystem is fragmented across multiple branding tiers from AMD and NVIDIA, each built on top of — or replacing — the open VESA Adaptive-Sync standard. Understanding the hierarchy matters for Linux compatibility: the kernel KMS driver interacts with the underlying hardware standard, not the marketing tier, so a FreeSync Premium Pro monitor and a plain VESA Adaptive-Sync monitor are handled by the same `VRR_ENABLED` property and differ only in their EDID-advertised capabilities. The table below summarises how each tier maps to the open standard, its hardware requirements, and its Linux support status.

| Brand / Tier | Underlying standard | Hardware requirement | HDR support | LFC (Low Framerate Compensation) | Linux/KMS support | Monitor cost premium |
|---|---|---|---|---|---|---|
| VESA Adaptive-Sync | DisplayPort 1.2a / HDMI 2.1 VRR | Panel scaler must support variable Htotal | Not specified by standard | Optional | Yes (`drm_property` `VRR_ENABLED`; atomic commit) | None (base standard) |
| AMD FreeSync | VESA Adaptive-Sync (AMD-certified) | Same as Adaptive-Sync | Not required | Yes (FreeSync requirement) | Yes (amdgpu + KMS) | Low (~$0–10 premium) |
| AMD FreeSync Premium | FreeSync + ≥120 Hz at native res + LFC | Same | Required (HDR10) | Yes | Yes | Moderate |
| AMD FreeSync Premium Pro | FreeSync Premium + validated HDR tone mapping | Same | Required (HDR10 + local dimming validated) | Yes | Yes | Higher |
| NVIDIA G-Sync Compatible | VESA Adaptive-Sync (NVIDIA-validated) | No proprietary module | Not required | Yes | Yes (nouveau/NVK, requires kernel 5.12+) | None |
| NVIDIA G-Sync (module) | Proprietary NVIDIA scaler module in monitor | Dedicated NVIDIA G-Sync ASIC in monitor | Required (some modules) | Yes | Limited (requires nvidia-open or proprietary) | High ($100–200 premium) |

### VESA Adaptive-Sync

The technical foundation of consumer VRR is **VESA Adaptive-Sync**, introduced in the DisplayPort 1.2a specification in 2014. [Source: VESA DisplayPort Standard](https://vesa.org/vesa-standards/standards-for-display/) Adaptive-Sync extends the VBLANK period dynamically: instead of a fixed vertical front porch duration, the display's timing controller waits for a GPU frame-ready signal before closing the VBLANK and beginning the next active video period. The GPU drives the timing, and the display follows.

Adaptive-Sync defines a **VRR operating range**, for example 48–144 Hz. The display can adapt its refresh period to anywhere within this range. Below the minimum (48 Hz in this example), the display must fall back to a fixed minimum refresh rate — it cannot extend the blanking period indefinitely.

### FreeSync (AMD)

**FreeSync** is AMD's marketing name for VESA Adaptive-Sync over DisplayPort (and later HDMI). [Source: AMD FreeSync Technology](https://www.amd.com/en/technologies/freesync) Announced in 2015, FreeSync was designed as an open standard: any display implementing VESA Adaptive-Sync could be certified as FreeSync-compatible without royalties. AMD's GPU hardware supported Adaptive-Sync signalling natively through the Display Core (DC) subsystem.

FreeSync comes in tiers:
- **FreeSync**: Basic Adaptive-Sync compliance over DisplayPort.
- **FreeSync Premium**: Minimum 120 Hz at 1080p, Low Framerate Compensation (LFC) required.
- **FreeSync Premium Pro** (formerly FreeSync 2 HDR): HDR support, low latency, wider color gamut.

### G-Sync (NVIDIA, Proprietary)

**G-Sync** (introduced 2013) predates VESA Adaptive-Sync and uses NVIDIA's proprietary hardware module embedded in the monitor. The G-Sync module replaces the display's scalar and timing controller, communicating directly with NVIDIA GPUs over a proprietary DisplayPort protocol. This allows NVIDIA to guarantee stricter VRR behavior but requires monitor manufacturers to purchase and integrate the G-Sync module. [Source: NVIDIA G-Sync](https://www.nvidia.com/en-us/geforce/products/g-sync-monitors/)

**G-Sync Compatible** is NVIDIA's certification program for monitors implementing VESA Adaptive-Sync to a standard NVIDIA deems acceptable. No proprietary module is required; compatible monitors advertise Adaptive-Sync in their EDID and pass NVIDIA's validation suite.

**G-Sync Ultimate** is NVIDIA's premium tier combining HDR with a peak brightness target, wide color gamut, and Adaptive-Sync.

### Low Framerate Compensation (LFC)

When GPU frame time exceeds the display's minimum VRR period (e.g., the GPU takes 25 ms but the display's minimum is 48 Hz = 20.8 ms), the display cannot extend its blanking period far enough. **Low Framerate Compensation (LFC)** handles this by multiplying the frame's presentation: the display shows the same frame twice (or more) to fill the gap, keeping the display's refresh rate within the VRR range while the GPU catches up. LFC requires a VRR range ratio of at least 2.5:1 (e.g., 40–144 Hz) so the display can always find an integer multiple within its range.

### HDMI VRR

**HDMI 2.1** introduced VRR support for HDMI connections. HDMI VRR uses a different signalling mechanism from DisplayPort Adaptive-Sync but achieves similar results. HDMI VRR is required for console compatibility (PlayStation 5, Xbox Series X) and is increasingly supported on PC monitors. AMD GPUs support HDMI VRR through the FreeSync stack.

### Monitor EDID Advertisement

A VRR-capable monitor advertises its capability in the EDID (Extended Display Identification Data) block. The relevant EDID structures include:

- **Display Range Limits descriptor** (EDID Standard version 1.4, detailed timing descriptor tag `0xFD`): The `max_vfreq` and `min_vfreq` fields encode the VRR operating range in Hz. The kernel's EDID parser reads these to determine the connector's VRR range.
- **Adaptive-Sync block** (DisplayID 2.0, Section 4.4.2): Explicit Adaptive-Sync support flag.

The DRM subsystem parses the EDID on connector probe and sets the `vrr_capable` connector property accordingly. [Source: Linux kernel DRM EDID parsing, `drivers/gpu/drm/drm_edid.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_edid.c)

---

## 4. KMS VRR Support

### DRM Properties for VRR

The Linux DRM subsystem exposes VRR through two atomic properties:

**Connector property: `vrr_capable`** (read-only)
- Type: range [0, 1] (boolean)
- Set by: the driver, based on EDID parsing
- Purpose: indicates whether the connected display supports VRR

**CRTC property: `VRR_ENABLED`** (read-write)
- Type: range [0, 1] (boolean)
- Set by: userspace (compositor or display manager)
- Purpose: instructs the kernel to activate VRR timing on this CRTC

This design cleanly separates hardware capability (`vrr_capable`, immutable from userspace) from policy (`VRR_ENABLED`, controlled by userspace). [Source: LWN.net — DRM API for adaptive sync](https://lwn.net/Articles/771165/)

The driver helpers for attaching these properties are:

```c
/* Connector capability property — called by driver during connector init */
int drm_connector_attach_vrr_capable_property(struct drm_connector *connector);
void drm_connector_set_vrr_capable_property(struct drm_connector *connector, bool capable);

/* CRTC enable property — called by driver or core */
int drm_mode_crtc_attach_vrr_enabled_property(struct drm_crtc *crtc);
```

[Source: Linux kernel — `drivers/gpu/drm/drm_connector.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_connector.c)

The `VRR_ENABLED` property is described by the kernel as indicating "content on the CRTC is suitable for variable refresh rate presentation." When set to 1, the driver extends the vertical front porch dynamically — holding off the VBLANK signal until a page-flip arrives — rather than generating VBLANK at the fixed refresh period. [Source: DRM property database — VRR_ENABLED](https://drmdb.emersion.fr/properties/3435973836/VRR_ENABLED)

The precise property name strings, as printed by `drm_info` and used in `drmModeObjectGetProperties()`, are: connector property `vrr_capable` (all lowercase) and CRTC property `VRR_ENABLED` (uppercase). The asymmetric naming reflects when each was added to the kernel.

### Enabling VRR via Atomic Commit

Compositors enable VRR with a standard atomic commit:

```c
/* Pseudocode — compositor enabling VRR on a CRTC */
drmModeAtomicReqPtr req = drmModeAtomicAlloc();

/* Set the CRTC VRR_ENABLED property */
drmModeAtomicAddProperty(req, crtc_id, vrr_enabled_prop_id, 1);

/* Commit — DRM_MODE_ATOMIC_NONBLOCK | DRM_MODE_PAGE_FLIP_EVENT */
drmModeAtomicCommit(drm_fd, req, DRM_MODE_ATOMIC_NONBLOCK, NULL);

drmModeAtomicFree(req);
```

The CRTC driver implementation receives the `VRR_ENABLED` property value as the `.vrr_enabled` field in `drm_crtc_state` during `atomic_check` and `atomic_commit`. Drivers validate that the connector's `vrr_capable` is 1 before accepting a new state with `.vrr_enabled = 1`.

### AMDGPU Display Core VRR Implementation

The AMDGPU driver implements VRR through its Display Core (DC) abstraction. The key source files are `drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c` and the companion CRTC-specific code.

[Source: Linux kernel — AMDGPU DM](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c)

Important functions in the AMDGPU VRR path:

- **`amdgpu_dm_crtc_vrr_active()`**: Returns whether VRR is currently active on a given `dm_crtc_state`. Called from interrupt handlers to gate VRR-specific logic.
- **`amdgpu_dm_crtc_vrr_active_irq()`**: IRQ context variant; called from `dm_crtc_high_irq()` and `dm_vupdate_high_irq()` to handle VRR VBLANK events.
- **`is_freesync_video_mode()`**: Checks whether a given `drm_display_mode` is a FreeSync video mode entry — used for special-casing video refresh rate modes that appear in the connector's mode list for VRR-range hint.
- **`reset_freesync_config_for_crtc()`**: Resets the FreeSync configuration on a `dm_crtc_state`, called during atomic state initialization.
- **`is_timing_unchanged_for_freesync()`**: Determines if the display timing has changed in a way that requires reinitializing the FreeSync state.
- **`update_freesync_state_on_stream()`**: Called from `amdgpu_dm_commit_planes()` to push updated FreeSync/VRR configuration down to the DC stream.

The DC layer communicates VRR state to the hardware via `VRR_STATE_ACTIVE_VARIABLE`, which causes the display engine to hold the VBLANK pulse until the next flip event rather than triggering it on a fixed timer.

### wlroots VRR

wlroots exposes VRR through the output state mechanism. The relevant field is `WLR_OUTPUT_STATE_ADAPTIVE_SYNC_ENABLED`, which triggers the backend to set `VRR_ENABLED` on the underlying KMS CRTC. The wlr-output-management protocol (version 4+) exposes an `adaptive_sync` event on `zwlr_output_head_v1` to report current VRR state to output management clients. See Ch21 for wlroots output management details. [Source: wlroots output management protocol](https://wayland.app/protocols/wlr-output-management-unstable-v1)

---

## 5. Wayland Presentation Feedback

### The wp_presentation Protocol

The **Wayland Presentation Time** protocol (`wp_presentation`) enables clients to receive precise timestamps for each frame's actual presentation moment, replacing the approximations that `wl_surface.frame` callbacks provide. [Source: Wayland presentation-time protocol](https://wayland.app/protocols/presentation-time)

The protocol flow:

1. The client calls `wp_presentation.feedback(wl_surface)` in association with a `wl_surface.commit`. This returns a `wp_presentation_feedback` object.
2. The compositor sends back one of two events:
   - `presented(...)`: The frame was shown on the display.
   - `discarded`: The frame was replaced by a newer commit before it could be displayed.

### The `presented` Event Signature

```
wp_presentation_feedback.presented(
    tv_sec_hi: uint,   /* high 32 bits of POSIX clock_realtime seconds */
    tv_sec_lo: uint,   /* low 32 bits of POSIX clock_realtime seconds */
    tv_nsec: uint,     /* nanoseconds within the second */
    refresh: uint,     /* nanoseconds until next predicted refresh */
    seq_hi: uint,      /* high 32 bits of refresh counter */
    seq_lo: uint,      /* low 32 bits of refresh counter */
    flags: uint        /* bitmask of kind flags */
)
```

The `refresh` field is especially valuable for VRR: on a fixed-refresh display it is constant (16,666,667 ns at 60 Hz), but on a VRR display it reflects the actual measured interval between the presented frame and the next expected frame. A client can use this to pace the next frame — begin rendering as soon as the `presented` callback arrives and target completion by `tv_nsec + refresh`.

### Presentation Feedback Kind Flags

The `flags` field is a bitmask combining zero or more of:

| Flag | Value | Meaning |
|------|-------|---------|
| `WP_PRESENTATION_FEEDBACK_KIND_VSYNC` | `0x1` | Presentation was synchronized to VBLANK; no tearing |
| `WP_PRESENTATION_FEEDBACK_KIND_HW_CLOCK` | `0x2` | Hardware clock used for the timestamp |
| `WP_PRESENTATION_FEEDBACK_KIND_HW_COMPLETION` | `0x4` | Hardware signalled start of scan-out |
| `WP_PRESENTATION_FEEDBACK_KIND_ZERO_COPY` | `0x8` | Buffer handed to hardware as-is (direct scanout) |

[Source: Wayland Presentation Time Protocol Spec](https://wayland.app/protocols/presentation-time)

For VRR, the `ZERO_COPY` flag is particularly significant: it indicates the compositor performed direct scanout of the client's buffer, meaning the GPU and display were directly coupled without compositor compositing latency.

### Using Presentation Feedback for Frame Pacing

A naive frame loop issues a new frame immediately after the previous submit:

```c
/* Naive: no pacing, submits as fast as possible */
while (running) {
    render_frame();
    wl_surface_commit(surface);
}
```

A presentation-feedback-aware loop uses the callback to anchor frame timing:

```c
static void on_presented(void *data,
    struct wp_presentation_feedback *feedback,
    uint32_t tv_sec_hi, uint32_t tv_sec_lo, uint32_t tv_nsec,
    uint32_t refresh, uint32_t seq_hi, uint32_t seq_lo, uint32_t flags)
{
    /* Schedule next render to begin NOW; target GPU completion
       by (tv_nsec + refresh) — the next predicted VBLANK */
    uint64_t next_vblank_ns = tv_nsec + refresh;
    schedule_render_for(next_vblank_ns);
    wp_presentation_feedback_destroy(feedback);
}
```

Chromium's Viz compositor uses `wp_presentation` feedback to drive its BeginFrame timing mechanism on Wayland. Gamescope uses it to measure actual vblank intervals and report frame latency to the Steam overlay.

---

## 6. Frame Pacing Strategies

### The Naive Strategy

Rendering "as fast as possible" and submitting immediately is the simplest approach: `VK_PRESENT_MODE_IMMEDIATE_KHR` on Vulkan, or unlocked swap on OpenGL. This maximizes GPU utilization and minimizes render-to-display latency for each individual frame, but produces constant tearing on fixed-refresh displays and wastes GPU power by rendering frames that are discarded.

### FIFO + Double Buffer (VSync)

`VK_PRESENT_MODE_FIFO_KHR` with two swapchain images is the standard VSync configuration. The application's `vkQueuePresentKHR` call blocks until VBLANK if the swapchain queue is full. This eliminates tearing at the cost of:
- Input latency up to one full frame (the frame may sit in the queue until VBLANK).
- Stutter whenever render time exceeds the refresh period.

### Triple Buffer / Mailbox

`VK_PRESENT_MODE_MAILBOX_KHR` with three swapchain images allows the application to submit frames continuously. The newest completed frame is always available at VBLANK. Stutter is reduced because the GPU can render ahead. Extra latency from the "mailbox" slot is variable but typically one half to one frame at target frame rate.

**Note on Wayland**: Wayland compositors are required to display every committed buffer at least once (the "present all buffers" semantic). Many Wayland compositors implement `VK_PRESENT_MODE_MAILBOX_KHR` by falling back to FIFO behavior — the Mesa Vulkan WSI layer on Wayland may expose mailbox but serialize presentation. This is an area of active protocol development as of 2026; the `wp_fifo_v1` protocol extension was proposed precisely to provide explicit FIFO semantics that distinguish latency-sensitive clients.

### Predictive Pacing

Instead of waiting passively for VBLANK, advanced frame pacers predict the next VBLANK and schedule the GPU submission to arrive just before it:

```
target_present_time = last_present_time + refresh_period
target_submit_time  = target_present_time - estimated_gpu_time - safety_margin
```

The `wp_presentation` `refresh` field gives an accurate `refresh_period` even on VRR displays. This strategy is used by Gamescope, Android's Choreographer, and game engines like Godot.

### Frame Pacing with VRR

VRR fundamentally changes the frame pacing problem. On a fixed-refresh display:
- Frame time must be < 16.67 ms or the frame is late and stutter occurs.
- Predictive pacing targets 16.5 ms to have a small safety margin.

On a VRR display:
- The display waits for the GPU. A 17 ms frame results in a 17 ms refresh period — no stutter, no tearing.
- A 10 ms frame results in a 10 ms refresh period — motion appears smoother.
- Frame pacing can target "complete as fast as possible" without worrying about VBLANK timing.
- The optimal Vulkan present mode on a VRR display is `VK_PRESENT_MODE_FIFO_KHR`: the FIFO ensures no tearing, and the display's adaptive timing means FIFO does not cause stutter.

### GPU Synchronization

Waiting for the GPU to complete frame rendering uses `vkWaitForFences` with a timeout that implements a sleep-wait pattern:

```c
/* Wait for GPU to finish rendering the previous frame */
VkResult result = vkWaitForFences(
    device,
    1, &in_flight_fence,
    VK_TRUE,
    UINT64_MAX   /* no timeout — sleep until done */
);
```

For frame pacing, a deadline-based wait is more useful:

```c
uint64_t deadline_ns = predicted_next_vblank_ns - safety_margin_ns;
vkWaitForFences(device, 1, &in_flight_fence, VK_TRUE,
                deadline_ns - current_time_ns);
/* If timed out, submit anyway and let VRR absorb the late frame */
```

### LFC and the VRR Floor

When GPU frame time exceeds the display's maximum VRR period (i.e., FPS drops below the VRR minimum refresh rate), LFC kicks in. The display repeats the last frame multiple times to stay within its VRR range while the GPU works on the next frame. For example, if the VRR range is 48–144 Hz and the GPU takes 30 ms (33 FPS), the display shows the frame twice at 16 ms each (60 Hz equivalent) using LFC. LFC requires a 2.5:1 VRR range ratio per FreeSync Premium specification.

---

## 7. Gamescope VRR Implementation

### Overview

**Gamescope** is Valve's micro-compositor for Steam Deck and Linux gaming. [Source: ValveSoftware/gamescope](https://github.com/ValveSoftware/gamescope) It is a Wayland compositor that accepts game content as Wayland surfaces (or XWayland windows) and composites them to a KMS output — or, in nested mode, presents them to a host compositor. Gamescope's primary design goal is minimal latency and maximum frame rate stability for gaming workloads.

### VRR in Gamescope

Gamescope enables VRR by setting the KMS `VRR_ENABLED` CRTC property to 1 during output initialization, provided the connected display advertises `vrr_capable`. The `--adaptive-sync` flag on the gamescope command line activates this path:

```bash
gamescope --adaptive-sync -w 1920 -h 1080 -- %command%
```

[Source: gamescope issue tracker — VRR support](https://github.com/ValveSoftware/gamescope/issues/199)

### Direct Scanout

Gamescope's most powerful optimization is **direct scanout**: when the game's Vulkan swapchain image meets the display's format, modifier, and size requirements, Gamescope promotes it directly to a KMS primary plane, bypassing compositor blending entirely. In direct scanout mode:

1. The game renders into a `VkImage` backed by a DMA-BUF.
2. Gamescope receives the buffer via `linux-dmabuf-unstable-v1`.
3. Instead of compositing the buffer onto a compositor framebuffer, Gamescope places the DMA-BUF directly on the KMS plane.
4. The KMS driver scans out the game's buffer directly to the display.

Combined with VRR, direct scanout means the game's `vkQueuePresentKHR` call is the proximate cause of the KMS page-flip event, which in turn signals the display to refresh. The end-to-end latency is as low as possible.

### Frame Rate Limiting with VRR

Gamescope's frame rate limiter is exposed via the `-r` flag:

```bash
gamescope -r 60 --adaptive-sync -- %command%
```

When a frame rate limit is set alongside VRR, Gamescope does not simply drop frames. Instead, it paces presentation so that frames arrive at the target rate while the VRR display adapts. A 60 FPS cap on a 48–144 Hz VRR display produces smooth 60 FPS output with no tearing — exactly equivalent to VSync at 60 Hz but without the stutter risk if a frame takes 17 ms rather than 16.67 ms.

Note: there is a known interaction between the `-r` flag and adaptive sync where, in some configurations, the monitor can get stuck at a static refresh rate. [Source: gamescope issue #975](https://github.com/ValveSoftware/gamescope/issues/975)

### VRR + HDR on Steam Deck OLED

The Steam Deck OLED (2023 generation) combines VRR with HDR on its internal OLED panel. Gamescope manages this combined path: VRR timing via KMS `VRR_ENABLED`, HDR metadata via KMS HDR color property, and direct scanout of the game's HDR swapchain. See Ch74 for HDR implementation details.

### Presentation Timing in Gamescope

Gamescope uses `wp_presentation` feedback to measure actual VBLANK intervals and compute accurate frame time statistics. These measurements are exposed to the Steam overlay frame time graph, allowing players to see per-frame GPU time, CPU time, and VRR adaptation in real time.

---

## 8. Mutter and KWin VRR

### GNOME Mutter VRR (GNOME 46+)

VRR support was merged into GNOME's Mutter compositor in the GNOME 46 development cycle (released March 2024) as an experimental feature, after approximately three years of development. [Source: Phoronix — Mutter VRR merged for GNOME 46](https://www.phoronix.com/news/Mutter-VRR-Merged-GNOME-46)

To enable the experimental VRR feature in GNOME:

```bash
# Enable experimental VRR feature flag
gsettings set org.gnome.mutter experimental-features "['variable-refresh-rate']"

# Then enable per-monitor in Settings > Displays > Refresh Rate > Variable Refresh Rate
```

Mutter's VRR implementation is in the KMS backend. The `MetaKmsConnector` object tracks the `vrr_capable` property of the underlying DRM connector. When VRR is enabled for an output, Mutter sets `VRR_ENABLED = 1` on the CRTC via an atomic commit.

Mutter's frame scheduling integrates VRR through the presentation feedback loop: Mutter measures the actual vblank interval from `wp_presentation` timestamps and schedules the next atomic commit to arrive just before the predicted next VBLANK. This means Mutter acts as a frame pacer even when VRR is enabled — it targets smooth timing rather than submitting at random times.

**Cursor behavior with VRR**: Early Mutter VRR implementations (GNOME 46–48) had a bug where cursor movement was limited to the display's minimum VRR refresh rate (e.g., 30 Hz) rather than the higher actual frame rate. GNOME 49 addressed this with improved cursor update scheduling that takes the minimum VRR refresh rate into account. [Source: WebProNews — GNOME 49 VRR cursor improvements](https://www.webpronews.com/gnome-49s-mutter-boosts-vrr-cursor-for-smoother-performance/)

### KWin VRR (Plasma 6+)

KDE's KWin compositor has supported VRR since Plasma 6.0 (released February 2024). KWin's Wayland backend implements the full VRR pipeline: EDID parsing → `vrr_capable` detection → `VRR_ENABLED` atomic commit → presentation feedback timing. [Source: KWin VRR merge request](https://invent.kde.org/plasma/kwin/-/merge_requests/718)

KWin VRR has undergone significant refinement across Plasma 6.x:

- **Plasma 6.4** (June 2025): Improved cursor smoothness by accounting for the minimum VRR refresh rate when scheduling cursor updates.
- **Plasma 6.5.3** (November 2025): Fixed visual smoothness issues when switching modes on multi-monitor VRR setups. [Source: 9to5Linux — KDE Plasma 6.5.3](https://9to5linux.com/kde-plasma-6-5-3-improves-visual-smoothness-on-multi-monitor-vrr-setups)
- **Plasma 6.6** (February 2026): Numerous fixes to the HDR + adaptive sync combined path.
- **Plasma 6.7** (June 2026): Per-screen VRR improvements enabling correct behavior on mixed-refresh-rate multi-monitor setups.

KWin exposes VRR through the system settings display configuration panel. Per-monitor VRR can be toggled independently.

### X11 Compositor VRR

VRR on X11 is more limited. The X server historically manages CRTC timing and does not expose VRR properties to compositors. Some workarounds exist:

- **xrandr VRR**: Some desktop environments (Xfce4, MATE) running on X11 can enable `VRR_ENABLED` via the underlying KMS DRM interface, but without compositor integration the benefit is limited.
- **AMDGPU DC FreeSync on X11**: The `amdgpu` DDX (Device-Dependent X driver) supports `Option "VariableRefresh" "true"` in `xorg.conf`, which enables FreeSync for fullscreen applications on X11. [Source: ArchWiki — VRR on X11](https://wiki.archlinux.org/title/Variable_refresh_rate)

X11's synchronization model does not provide a clean equivalent to `wp_presentation`, making frame pacing substantially harder to implement correctly on X11 with VRR.

---

## 9. Vulkan Swapchain and VRR

### VK_EXT_swapchain_maintenance1 / VK_KHR_swapchain_maintenance1

`VK_EXT_swapchain_maintenance1` (promoted to `VK_KHR_swapchain_maintenance1`) adds several features omitted from the original `VK_KHR_swapchain`:

1. **Changing present mode without swapchain recreation**: Applications can specify a new present mode at each `vkQueuePresentKHR` call via `VkSwapchainPresentModeInfoEXT`, without tearing down and rebuilding the swapchain.
2. **`VkReleaseSwapchainImagesEXT`**: Allows the application to release queued swapchain images back to the swapchain without presenting them. This is essential for mailbox-style rendering without wasting allocation.
3. **Fence signalling on present completion**: A fence can be signalled when the resources associated with a present operation are safe to destroy.

[Source: Vulkan Documentation — VK_EXT_swapchain_maintenance1](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_swapchain_maintenance1.html)

For VRR workflows, the ability to switch present modes at runtime is particularly valuable. An application might start with `VK_PRESENT_MODE_FIFO_KHR` during the main menu (stable 60 Hz) and switch to `VK_PRESENT_MODE_MAILBOX_KHR` during gameplay for lower latency — or switch to `VK_PRESENT_MODE_IMMEDIATE_KHR` for a benchmark run, all without destroying and recreating the swapchain:

```c
VkSwapchainPresentModeInfoEXT present_mode_info = {
    .sType          = VK_STRUCTURE_TYPE_SWAPCHAIN_PRESENT_MODE_INFO_EXT,
    .swapchainCount = 1,
    .pPresentModes  = &new_present_mode,
};

VkPresentInfoKHR present_info = {
    .sType          = VK_STRUCTURE_TYPE_PRESENT_INFO_KHR,
    .pNext          = &present_mode_info,
    /* ... swapchain, image index, semaphores ... */
};

vkQueuePresentKHR(graphics_queue, &present_info);
```

### VK_PRESENT_MODE_FIFO_RELAXED_KHR and VRR

`VK_PRESENT_MODE_FIFO_RELAXED_KHR` is useful on fixed-refresh displays where occasional tearing is preferable to guaranteed stutter when a frame is slightly late. On VRR displays, this mode is largely unnecessary: the display waits for the GPU anyway, so a late frame causes a longer refresh period rather than a missed VBLANK. On VRR + FIFO, there is no "missed VBLANK" scenario.

### Present Mode Selection for VRR

For applications targeting VRR displays on Linux/Wayland, the recommended present mode is:

| Scenario | Recommended Present Mode |
|---|---|
| VRR display, Wayland, gaming | `VK_PRESENT_MODE_FIFO_KHR` — no tearing, VRR absorbs variance |
| Fixed-refresh display, gaming | `VK_PRESENT_MODE_MAILBOX_KHR` — low latency, minor tearing risk |
| Fixed-refresh display, stutter tolerance | `VK_PRESENT_MODE_FIFO_RELAXED_KHR` |
| Benchmarking | `VK_PRESENT_MODE_IMMEDIATE_KHR` |

Querying available present modes at swapchain creation:

```c
uint32_t mode_count;
vkGetPhysicalDeviceSurfacePresentModesKHR(physical_device, surface,
    &mode_count, NULL);

VkPresentModeKHR *modes = malloc(sizeof(VkPresentModeKHR) * mode_count);
vkGetPhysicalDeviceSurfacePresentModesKHR(physical_device, surface,
    &mode_count, modes);

/* Prefer FIFO on VRR displays; fall back to FIFO_RELAXED or IMMEDIATE */
VkPresentModeKHR selected = VK_PRESENT_MODE_FIFO_KHR; /* always supported */
for (uint32_t i = 0; i < mode_count; i++) {
    if (modes[i] == VK_PRESENT_MODE_MAILBOX_KHR && !vrr_active)
        selected = VK_PRESENT_MODE_MAILBOX_KHR;
}
```

### Declaring Supported Present Modes at Swapchain Creation

`VkSwapchainPresentModesCreateInfoEXT` (from `VK_EXT_swapchain_maintenance1`) allows the application to declare upfront which present modes it may switch to at runtime, enabling the driver to allocate internal resources appropriately:

```c
VkPresentModeKHR supported_modes[] = {
    VK_PRESENT_MODE_FIFO_KHR,
    VK_PRESENT_MODE_IMMEDIATE_KHR,
};

VkSwapchainPresentModesCreateInfoEXT modes_info = {
    .sType            = VK_STRUCTURE_TYPE_SWAPCHAIN_PRESENT_MODES_CREATE_INFO_EXT,
    .presentModeCount = 2,
    .pPresentModes    = supported_modes,
};

VkSwapchainCreateInfoKHR create_info = {
    /* ... */
    .pNext        = &modes_info,
    .presentMode  = VK_PRESENT_MODE_FIFO_KHR, /* initial mode */
};
```

---

## 10. Measuring and Debugging Frame Timing

### MangoHud

[MangoHud](https://github.com/flightlessmango/MangoHud) is the standard Linux gaming performance overlay. It provides real-time display of:
- FPS and frame time (ms)
- Frame time graph (bar chart showing per-frame GPU time)
- CPU and GPU usage, temperature, power draw
- VRAM usage

Enable MangoHud for any Vulkan or OpenGL game:

```bash
MANGOHUD=1 mangohud %command%

# Or via Steam launch options:
# MANGOHUD=1 %command%
```

The frame time graph is the primary diagnostic tool for VRR: on a VRR display, frame time variation is visible as bar height variation but does not produce visible stutter. On a fixed-refresh display, any bar exceeding the refresh period (e.g., 16.67 ms at 60 Hz) causes a dropped frame.

**Interpreting the MangoHud frame time graph with VRR:**
- Consistent 8 ms bars → smooth 120 Hz VRR operation.
- Occasional 22 ms bar → single-frame GPU spike; VRR display waits, presents 22 ms frame, immediately resumes next frame. No visible stutter.
- Sustained 25 ms bars → below 48 Hz VRR minimum; LFC may activate.
- Bars > 33 ms → below 30 Hz even with LFC; visible judder.

### Gamescope Performance Overlay

The Gamescope / Steam Deck overlay (accessible via the Steam button → Performance tab) shows:
- FPS (top-level number)
- Frame time graph with CPU and GPU segments
- Battery power, temperature, VRAM

This overlay uses `wp_presentation` feedback for accurate frame timing, not software clock estimates.

### Checking VRR Status with drm_info

The `drm_info` tool ([github.com/ascent12/drm_info](https://github.com/ascent12/drm_info)) dumps all DRM object properties:

```bash
# Install on Arch Linux
pacman -S drm_info

# Dump all properties
drm_info

# Filter for VRR-related properties
drm_info | grep -i vrr
```

Example output for a VRR-capable display (property names reflect actual kernel strings):
```
Connector DP-1:
  vrr_capable: 1

CRTC 0:
  VRR_ENABLED: 1
```

### Checking VRR via sysfs

The kernel's DRM sysfs interface at `/sys/class/drm/` exposes some connector attributes (status, enabled, dpms, modes, edid) but does not generally expose arbitrary atomic properties like `vrr_capable` as individual sysfs files — those are only accessible through the DRM ioctl interface (which `drm_info` uses). Use `drm_info` rather than sysfs for reliable VRR property inspection.

> **Note: needs verification** — Some downstream kernel patches or specific driver versions may add `vrr_capable` as a sysfs attribute on connectors. Check your kernel's `drivers/gpu/drm/drm_sysfs.c` if you need sysfs-based VRR detection in automation scripts; otherwise rely on `drm_info`.

### Checking VRR Status with xrandr (X11)

On X11:

```bash
xrandr --prop | grep -A 2 vrr
```

### GPU Timestamp Queries

For per-pass GPU profiling inside a frame, use `vkCmdWriteTimestamp2`:

```c
/* Before the geometry pass */
vkCmdWriteTimestamp2(cmd_buf,
    VK_PIPELINE_STAGE_2_TOP_OF_PIPE_BIT,
    query_pool, 0);

/* ... draw calls ... */

/* After the geometry pass */
vkCmdWriteTimestamp2(cmd_buf,
    VK_PIPELINE_STAGE_2_BOTTOM_OF_PIPE_BIT,
    query_pool, 1);

/* Retrieve results after fence signals */
uint64_t timestamps[2];
vkGetQueryPoolResults(device, query_pool, 0, 2,
    sizeof(timestamps), timestamps, sizeof(uint64_t),
    VK_QUERY_RESULT_64_BIT | VK_QUERY_RESULT_WAIT_BIT);

double gpu_ns = (timestamps[1] - timestamps[0])
                * timestamp_period; /* from VkPhysicalDeviceLimits */
```

This allows identifying which pass in the frame is causing occasional GPU time spikes that trigger VRR at lower refresh rates.

### Common VRR Troubleshooting

**VRR not enabling despite `vrr_capable: 1`:**

```bash
# Check kernel messages for EDID and VRR detection
dmesg | grep -i "vrr\|freesync\|adaptive"

# Check if the compositor is setting VRR_ENABLED
# (use drm_info at runtime; or check kernel VRR_ENABLED tracing)
```

**Monitor flickering at low FPS:** Common when FPS drops near or below the VRR minimum (e.g., 48 Hz). The display transitions between VRR mode and LFC, which can cause a brief blank or flicker. Solutions:
- Enable LFC if the VRR range supports it (requires 2.5:1 range minimum).
- Set a frame rate floor in Gamescope (`-r 48` minimum target).

**VRR disabling when HDR is active:** Some GPU/display combinations do not support VRR + HDR simultaneously. Check connector property list for HDR and VRR capability flags co-existing.

**NVIDIA G-Sync Compatible on Wayland:** NVIDIA's open kernel module (`nvidia-open`) is required for KMS-based VRR on Wayland with NVIDIA hardware. The proprietary `nvidia.ko` with `nvidia-drm.modeset=1` supports VRR in KMS mode but compositor support varies. Check kernel parameter `nvidia-drm.modeset=1` is set. [Source: NVIDIA KMS documentation](https://download.nvidia.com/XFree86/Linux-x86_64/595.58.03/README/kms.html)

---

## 11. wp_tearing_control_v1 — Client-Side Tearing Opt-In

### Protocol Overview

The **`wp_tearing_control_v1`** Wayland protocol, introduced in **Wayland Protocols 1.30** (released August 2023), gives clients a formal mechanism to declare that their surface can tolerate tearing in exchange for lower latency. [Source: Phoronix — Wayland Protocols 1.30 Tearing Control](https://www.phoronix.com/news/Wayland-Tearing-Control-Proto) Before this protocol existed, compositors had no way to know which surfaces could be scanned out with an async (immediate) page flip, so they defaulted to tearing-free VSync for everything. Games and drawing-tablet applications were the primary motivation: a stylus-input canvas that runs at 120 FPS has more to gain from immediate scanout than from the 8 ms additional latency imposed by waiting for VBLANK.

The protocol copyright is held by Xaver Hugl (KDE), and it is currently in the **staging** tier of wayland-protocols (`staging/tearing-control/tearing-control-v1.xml`). [Source: wayland-protocols staging — tearing-control](https://wayland.app/protocols/tearing-control-v1)

### Protocol Interfaces

The protocol defines two interfaces:

**`wp_tearing_control_manager_v1`** — a global factory object bound by the client:

```xml
<!-- staging/tearing-control/tearing-control-v1.xml (abridged) -->
<interface name="wp_tearing_control_manager_v1" version="1">
  <description summary="protocol for tearing control">
    For some use cases like games or drawing tablets it can make sense to
    reduce latency by accepting tearing with the use of asynchronous page
    flips. This global is a factory interface, allowing clients to inform
    which type of presentation the content of their surfaces is suitable for.
  </description>

  <request name="destroy" type="destructor"/>

  <request name="get_tearing_control">
    <description summary="extend surface interface for tearing control">
      Instantiate an interface extension for the given wl_surface to
      implement tearing control. If the given wl_surface already has a
      wp_tearing_control_v1 object associated, the tearing_control_exists
      protocol error is raised.
    </description>
    <arg name="id"      type="new_id" interface="wp_tearing_control_v1"/>
    <arg name="surface" type="object" interface="wl_surface"/>
  </request>

  <enum name="error">
    <entry name="tearing_control_exists" value="0"
      summary="This surface already has a tearing_control object associated"/>
  </enum>
</interface>
```

**`wp_tearing_control_v1`** — the per-surface object that carries the presentation hint:

```xml
<interface name="wp_tearing_control_v1" version="1">
  <description summary="per-surface tearing control interface"/>

  <enum name="presentation_hint">
    <entry name="vsync" value="0"
      summary="the content should be synchronized to the vertical blanking
               period — no tearing"/>
    <entry name="async" value="1"
      summary="the content is suitable for presentation with minimal
               latency — tearing is acceptable"/>
  </enum>

  <request name="set_presentation_hint">
    <description summary="set presentation hint">
      Set the presentation hint for the associated wl_surface. This state
      is double-buffered and is applied on the next wl_surface.commit.
      The compositor is free to dynamically respect or ignore this hint
      based on hardware capabilities, surface state and user preferences.
    </description>
    <arg name="hint" type="uint" enum="presentation_hint"/>
  </request>

  <request name="destroy" type="destructor">
    <description summary="destroy the tearing control interface">
      Destroy the tearing control object. The hint will be reverted to
      vsync on the next wl_surface.commit.
    </description>
  </request>
</interface>
```

[Source: Wayland Explorer — tearing-control-v1 protocol](https://wayland.app/protocols/tearing-control-v1)

### Protocol Message Sequence

A typical client interaction looks like this:

```c
/* 1. Bind the manager from the registry */
struct wp_tearing_control_manager_v1 *tearing_mgr =
    wl_registry_bind(registry, name,
                     &wp_tearing_control_manager_v1_interface, 1);

/* 2. Create a per-surface tearing control object */
struct wp_tearing_control_v1 *tearing_ctrl =
    wp_tearing_control_manager_v1_get_tearing_control(tearing_mgr, surface);

/* 3. Set the hint to async (tearing allowed) — double-buffered */
wp_tearing_control_v1_set_presentation_hint(tearing_ctrl,
    WP_TEARING_CONTROL_V1_PRESENTATION_HINT_ASYNC);

/* 4. Commit the surface — the hint takes effect after this commit */
wl_surface_commit(surface);

/* 5. To revert to vsync, set the hint back and commit again */
wp_tearing_control_v1_set_presentation_hint(tearing_ctrl,
    WP_TEARING_CONTROL_V1_PRESENTATION_HINT_VSYNC);
wl_surface_commit(surface);
```

The double-buffered nature is essential: setting the hint before `wl_surface.commit` means the compositor can atomically switch between tearing and non-tearing presentation on the same commit that updates the buffer. Note that the spec explicitly states that graphics APIs such as EGL and Vulkan manage this hint internally — clients using a Vulkan WSI should not call `set_presentation_hint` directly, as doing so alongside `VK_PRESENT_MODE_IMMEDIATE_KHR` can trigger a protocol error.

### Compositor Implementation Considerations

The hint is advisory. A compositor may silently ignore `ASYNC` and fall back to `VSYNC` if:
- The underlying KMS driver does not support atomic async page flips (`DRM_MODE_PAGE_FLIP_ASYNC` on the atomic interface). Support for async atomic flips was proposed in 2023 and landed in the AMDGPU driver; see the patch series at [lwn.net/Articles/948826/](https://lwn.net/Articles/948826/).
- The surface is not full-screen / direct-scanned out.
- The user has globally disabled tearing in compositor settings.

**wlroots** merged tearing control support in September 2023 (wlroots 0.17). When a surface requests `ASYNC` and direct scanout is active, wlroots will issue an async page flip: `DRM_MODE_PAGE_FLIP_ASYNC` or the atomic equivalent. [Source: Phoronix — wlroots merges tearing control](https://www.phoronix.com/news/wlroots-Tearing-Control-Merged)

**KWin** (Plasma 6+) supports the tearing protocol for fullscreen applications. When the `async` hint is active, KWin enables its "immediate presentation" path, which coexists with VRR: an `async`-hinted surface on a VRR display gets immediate KMS page flips; the display adapts its refresh period to the flip timing, providing both low latency *and* tear-free output — because VRR's wait-for-signal mechanism means the display begins scanning the new frame exactly when it arrives rather than tearing mid-scan.

### Interaction with VRR

The key insight is that on a **VRR-capable display**, `ASYNC` presentation with tearing control is often unnecessary. VRR already delivers immediate frames (the display waits for the GPU) without tearing. The `async` hint is most valuable on **fixed-refresh displays** where an application wants to trade tearing for the reduction in submit-to-scan latency. On VRR displays, `VSYNC` with VRR active achieves the same latency profile without tearing.

The recommended matrix is:

| Display | Hint | Result |
|---|---|---|
| Fixed refresh (60 Hz) | `VSYNC` | Tearing-free, up to 16.67 ms latency |
| Fixed refresh (60 Hz) | `ASYNC` | Potential tearing, minimal submit-to-scan latency |
| VRR (48–144 Hz) | `VSYNC` | Tearing-free, display adapts to GPU — best option |
| VRR (48–144 Hz) | `ASYNC` | Compositor may allow async flip; gain is marginal |

---

## 12. wp_fifo_v1 — Explicit FIFO Presentation Semantics

### Motivation: The Mailbox Problem on Wayland

Wayland's buffer contract has always required that every committed buffer eventually be presented at least once — a semantic known informally as "present all buffers." This rule makes `VK_PRESENT_MODE_MAILBOX_KHR` semantically incompatible with Wayland: mailbox discards frames if a newer one is ready, but Wayland must show every frame. Compositors that expose mailbox typically simulate it with FIFO, giving the same average frame rate but without the low-latency property of true mailbox. Applications that wanted minimum latency had no clean protocol solution; they used heuristics or relied on compositor-specific behavior.

The **`wp_fifo_v1`** protocol (merged into Wayland Protocols as a staging extension in 2024) replaces the old mailbox hack with an explicit surface-level barrier mechanism. [Source: Fifo protocol — Wayland Explorer](https://wayland.app/protocols/fifo-v1) Rather than suppressing frame delivery, it gives the client a way to tell the compositor "do not apply my next commit until the current frame has completed at least one display refresh cycle." This is genuine FIFO ordering at the Wayland level.

The protocol was developed with contributions from Valve (whose SDL and game pipeline needs drove much of the requirements), and KWin landed its implementation in Plasma 6.4. [Source: Phoronix forums — KWin FIFO v1 Wayland](https://www.phoronix.com/forums/forum/software/desktop-linux/1536119-kde-kwin-lands-fifo-v1-wayland-support-gnome-48-squeezed-in-xdg-toplevel-drag-v1)

### Protocol Interfaces

**`wp_fifo_manager_v1`** — a global factory bound once per client:

```xml
<!-- staging/fifo/fifo-v1.xml (abridged) -->
<interface name="wp_fifo_manager_v1" version="1">
  <description summary="protocol for fifo constraints">
    This protocol provides a way to use the completion of a display refresh
    cycle as an additional readiness constraint when a Wayland compositor
    considers applying a content update.
  </description>

  <request name="destroy" type="destructor"/>

  <request name="get_fifo">
    <description summary="bind a fifo object to a surface"/>
    <arg name="id"      type="new_id" interface="wp_fifo_v1"/>
    <arg name="surface" type="object" interface="wl_surface"/>
  </request>

  <enum name="error">
    <entry name="already_exists" value="0"
      summary="a fifo object already exists for this surface"/>
  </enum>
</interface>
```

**`wp_fifo_v1`** — per-surface interface with barrier semantics:

```xml
<interface name="wp_fifo_v1" version="1">
  <description summary="fifo interface for a surface"/>

  <request name="set_barrier">
    <description summary="set a presentation barrier">
      When the content update containing this request is applied, it sets a
      fifo_barrier condition on the surface associated with the fifo object.
      The condition is cleared immediately after the following latching
      deadline for non-tearing presentation.
      This is double-buffered state, see wl_surface.commit.
    </description>
  </request>

  <request name="wait_barrier">
    <description summary="wait for the current fifo barrier">
      Indicates that a content update is not ready to be presented, while a
      fifo_barrier condition is present on the surface. When the content
      update containing set_barrier was made active at a latching deadline,
      it will be active for at least one refresh cycle.
      This is double-buffered state, see wl_surface.commit.
    </description>
  </request>

  <request name="destroy" type="destructor"/>

  <enum name="error">
    <entry name="surface_destroyed" value="0"
      summary="the associated wl_surface was destroyed"/>
  </enum>
</interface>
```

[Source: Wayland Explorer — fifo-v1 protocol](https://wayland.app/protocols/fifo-v1)

### The FIFO Barrier Mechanism in Practice

The barrier pair (`set_barrier` + `wait_barrier`) forms a two-commit handshake:

**Commit N** — establishes the barrier:
```c
/* Commit N: render frame N, set the barrier so we know when it is scanned */
wl_surface_attach(surface, buffer_N, 0, 0);
wp_fifo_v1_set_barrier(fifo);          /* mark this commit as the barrier */
wl_surface_commit(surface);
/* Buffer N will now be presented at the next latching deadline */
```

**Commit N+1** — waits for the barrier before the compositor latches it:
```c
/* Start rendering frame N+1 in parallel ... */
render_frame_N_plus_1();

/* Commit N+1: don't apply until frame N has completed ≥ 1 refresh cycle */
wl_surface_attach(surface, buffer_N_plus_1, 0, 0);
wp_fifo_v1_wait_barrier(fifo);         /* block latching until barrier cleared */
wl_surface_commit(surface);
```

The `wait_barrier` in Commit N+1 instructs the compositor: "do not make this commit active until the fifo_barrier from Commit N has been cleared — i.e., until at least one refresh cycle has elapsed after Commit N was scanned." This is precisely the FIFO semantic: frame N+1 can only replace frame N after frame N has been shown for a full refresh.

From the client's perspective, the `wl_surface.commit` call for N+1 returns immediately. The compositor queues the commit and delays its latching. The client can submit as many commits ahead as it likes, each with `wait_barrier`, and the compositor serializes them in submission order.

### Interaction with VRR Compositors

On a VRR display, the compositor's "latching deadline" is not a fixed VBLANK — it is the moment when the KMS hardware captures the next flip request. When `wait_barrier` is set, the compositor must not latch Commit N+1 until after Commit N's fifo_barrier clears, which happens after at least one VRR refresh cycle following Commit N's scan-out. The practical consequence is:

- Frame ordering is guaranteed regardless of VRR timing.
- The compositor cannot coalesce frames N and N+1 into a single scanout (which would violate the FIFO contract).
- The VRR display's refresh period for each frame is driven by the GPU's actual completion of that frame, not by the client's submission rate.

This means `wp_fifo_v1` on a VRR display produces the ideal combination: frames are delivered in order (no `discarded` feedback events), the display runs at the GPU's natural cadence, and there is no tearing. The protocol effectively makes `VK_PRESENT_MODE_FIFO_KHR` on Wayland semantically correct for VRR.

**SDL3** (starting from late 2024) adopted `wp_fifo_v1` as the default Wayland presentation protocol for VSync-enabled windows, replacing the earlier workaround of using `wl_surface.frame` callbacks as a proxy barrier. [Source: SDL discourse — fifo-v1 as default](https://discourse.libsdl.org/t/sdl-wayland-only-require-fifo-v1-for-wayland-by-default/55974)

**Smithay** (Rust compositor toolkit) implemented the server side of `wp_fifo_v1` in its `smithay::wayland::fifo` module. [Source: Smithay fifo module](https://smithay.github.io/smithay/smithay/wayland/fifo/index.html)

---

## 13. Frame Pacing on RDNA3+ — AMD Hardware Frame Pacing

### Vulkan Extensions for Precise Frame Timing on RADV

On AMD RDNA3+ hardware using the RADV open-source Vulkan driver, frame pacing is best implemented using two Vulkan extensions that expose hardware-level presentation timing:

**`VK_KHR_present_id`** — attaches an application-managed monotonic integer to each `vkQueuePresentKHR` call:

```c
uint64_t present_id = frame_number++;  /* monotonically increasing */

VkPresentIdKHR present_id_info = {
    .sType          = VK_STRUCTURE_TYPE_PRESENT_ID_KHR,
    .swapchainCount = 1,
    .pPresentIds    = &present_id,
};

VkPresentInfoKHR present_info = {
    .sType = VK_STRUCTURE_TYPE_PRESENT_INFO_KHR,
    .pNext = &present_id_info,
    /* ... swapchain, image index, wait semaphores ... */
};

vkQueuePresentKHR(graphics_queue, &present_info);
```

**`VK_KHR_present_wait`** — blocks the CPU thread until a specific `present_id` has been displayed:

```c
/* Block until frame (present_id - 2) has been shown on screen,
   implementing a frame pacer with 2-frame lookahead */
uint64_t wait_id = present_id - 2;
VkResult res = vkWaitForPresentKHR(device, swapchain, wait_id,
                                    deadline_ns);
/* Now safe to begin rendering the next frame knowing the pipeline
   has cleared at least 2 frames' worth of backpressure */
```

Valve implemented `VK_KHR_present_wait` for RADV (and also for Intel ANV and Qualcomm TURNIP) and described it as "very useful" for frame pacing on Steam Deck workloads. Because the current Vulkan spec has limitations on when `present_wait` can safely be used, RADV exposes it as an opt-in feature through a DriConf variable:

```bash
# Enable VK_KHR_present_wait for a specific application
DRIRC_USERSPACE=1 MESA_VK_WSI_PRESENT_WAIT=true %command%

# Or via a DriConf XML entry per-application:
# <option name="vk_khr_present_wait" value="true"/>
```

[Source: Phoronix — Valve implements VK_KHR_present_wait for Mesa](https://www.phoronix.com/news/Mesa-VK_KHR_present_wait) Together, `present_id` and `present_wait` allow a Vulkan application to measure the actual GPU-to-display latency for each frame and throttle submission to stay within a target latency budget — much like the `wp_presentation` feedback mechanism at the Wayland level, but accessible from within a Vulkan application without compositor cooperation.

### AMDGPU Userspace Fence Waiting

Below the Vulkan layer, RADV ultimately synchronizes on DRM fence objects exposed by the AMDGPU kernel driver. The direct userspace interface for waiting on AMDGPU fences is the `DRM_IOCTL_AMDGPU_WAIT_FENCES` ioctl (ioctl number `DRM_COMMAND_BASE + 0x12`), defined in `include/uapi/drm/amdgpu_drm.h`:

```c
/* From include/uapi/drm/amdgpu_drm.h */
struct drm_amdgpu_wait_fences_in {
    __u64 fences;       /* pointer to array of struct drm_amdgpu_fence */
    __u32 fence_count;
    __u32 wait_all;     /* 0 = wait for any; 1 = wait for all */
    __u64 timeout_ns;   /* timeout in nanoseconds */
};

struct drm_amdgpu_wait_fences_out {
    __u32 status;           /* 0 = timeout, 1 = signalled */
    __u32 first_signaled;   /* index of the first fence that signalled */
};

union drm_amdgpu_wait_fences {
    struct drm_amdgpu_wait_fences_in  in;
    struct drm_amdgpu_wait_fences_out out;
};

#define DRM_IOCTL_AMDGPU_WAIT_FENCES \
    DRM_IOWR(DRM_COMMAND_BASE + DRM_AMDGPU_WAIT_FENCES, \
             union drm_amdgpu_wait_fences)
```

[Source: libdrm — include/drm/amdgpu_drm.h](https://github.com/grate-driver/libdrm/blob/master/include/drm/amdgpu_drm.h)

The RADV driver wraps this ioctl (accessed indirectly through the DRM syncobj infrastructure) when implementing fence-based GPU-CPU synchronization. A frame pacer built on top of this can set `timeout_ns` to a deadline before the next predicted VBLANK, implementing a bounded wait that allows the CPU to skip ahead if the GPU is late:

```c
struct drm_amdgpu_wait_fences_in args = {
    .fences     = (uint64_t)(uintptr_t)fence_array,
    .fence_count = 1,
    .wait_all   = 1,
    .timeout_ns  = predicted_vblank_ns - SAFETY_MARGIN_NS,
};
/* If the ioctl returns status=0 (timeout), the frame is late;
   submit anyway and let VRR absorb the longer refresh period */
int ret = drmIoctl(drm_fd, DRM_IOCTL_AMDGPU_WAIT_FENCES, &args);
```

### Timeline Syncobjs and Kernel-Level Fence Management

RDNA3+ supports **timeline DRM synchronization objects** (`drm_syncobj` with `DRM_SYNCOBJ_TYPE_TIMELINE`), which were added to the AMDGPU kernel driver to support `VK_KHR_timeline_semaphore`. Timeline syncobjs carry a monotonically increasing 64-bit counter rather than a binary signalled/unsignalled state. A patch series adding timeline syncobj support to the AMDGPU wait IOCTL was submitted to the amd-gfx mailing list in 2024. [Source: amd-gfx mailing list — Add wait IOCTL timeline syncobj support](https://www.mail-archive.com/amd-gfx@lists.freedesktop.org/msg113082.html)

For frame pacing, timeline semaphores allow the driver to express "GPU work for frame N is complete at timeline point N" without needing a separate binary fence per frame. The composer can wait for point N-1 before submitting frame N to maintain exactly one frame of pipelining:

```c
/* Vulkan timeline semaphore usage for frame pacing */
VkSemaphoreTypeCreateInfo timeline_type = {
    .sType         = VK_STRUCTURE_TYPE_SEMAPHORE_TYPE_CREATE_INFO,
    .semaphoreType = VK_SEMAPHORE_TYPE_TIMELINE,
    .initialValue  = 0,
};
/* ... create semaphore ... */

/* Submit frame N: signal timeline point N when GPU work is done */
uint64_t signal_value = frame_number;
VkTimelineSemaphoreSubmitInfo timeline_submit = {
    .signalSemaphoreValueCount = 1,
    .pSignalSemaphoreValues    = &signal_value,
};

/* Before submitting frame N+1, wait for frame N-1 to be presented */
uint64_t wait_value = frame_number - 1;
VkSemaphoreWaitInfo wait_info = {
    .sType          = VK_STRUCTURE_TYPE_SEMAPHORE_WAIT_INFO,
    .semaphoreCount = 1,
    .pSemaphores    = &timeline_semaphore,
    .pValues        = &wait_value,
};
vkWaitSemaphores(device, &wait_info, deadline_ns);
```

### Radeon GPU Profiler (RGP) Frame Analysis

The **Radeon GPU Profiler (RGP)** is AMD's low-level GPU profiling tool, available through AMD GPUOpen and supporting RDNA3 hardware since RGP 1.14. [Source: AMD GPUOpen — RGP 1.14 introduces RDNA3 support](https://gpuopen.com/learn/rgp_1_14/) RGP captures detailed frame traces showing command-buffer execution, pipeline barriers, memory accesses, and queue submission timing.

To analyze frame pacing behavior on RDNA3+:

1. **Capture with RGP**: Set `AMDGPU_DEBUG=rgp` (or use the Radeon Developer Panel) to enable RGP capture mode. Submit a frame trace via the `VK_AMD_rasterization_order` or RGP capture markers.
2. **Examine the Frame Summary**: The Frame Summary panel shows the distribution of GPU frame times and identifies frames that exceed the VRR maximum period.
3. **Barrier and Queue Timeline views**: Identify pipeline barriers and semaphore wait points that add latency between geometry submission and scanout readiness.
4. **Event Timing**: Per-draw-call GPU timestamps (written via `vkCmdWriteTimestamp2`) are visualized in the Event Timeline view, making it straightforward to identify which render pass is the pacing bottleneck.

> **Note: needs verification** — RGP's documentation refers to a "Frame Pacing" section in its manual (`/manuals/rgp_manual/rgp_manual-index/`), but at the time of writing the specific view name for frame-over-frame latency breakdown is not confirmed in public-facing documentation. Consult the [AMD RGP manual](https://gpuopen.com/manuals/rgp_manual/rgp_manual-index/) for the current view name and feature availability on your GPU generation.

---

## 14. VRR on OLED Displays — Burn-In Mitigation and Refresh Rate Floors

### Why OLED Requires Special VRR Treatment

Liquid Crystal Display (LCD) panels use a persistent backlight: pixel state changes cause the liquid crystals to modulate the backlight, but the backlight itself runs at a constant intensity. The display controller's refresh rate change under VRR therefore has minimal impact on apparent brightness.

**OLED (Organic Light-Emitting Diode)** panels are self-emissive: each pixel directly emits light proportional to the drive current applied during its refresh cycle. The OLED timing controller optimizes its gamma correction curves and drive current profiles for a specific reference refresh rate (commonly 120 Hz or 144 Hz). When VRR causes the refresh period to deviate from this optimized rate, the drive current profile no longer matches the panel's gamma response:

- At higher-than-reference refresh rates, the drive period is shorter; sub-pixels receive less charge and appear dimmer.
- At lower-than-reference refresh rates, the extended drive period can cause sub-pixels to reach saturation differently, shifting the displayed gamma.

The result is visible **brightness fluctuation** correlated with frame rate changes — most pronounced in dark and near-black content where human vision is most sensitive to brightness differences. Testing on WOLED panels shows gamma shifts of 11 or more RGB steps between the reference rate and 10 FPS, producing a clearly visible flicker each time the frame rate crosses a major threshold. [Source: TFTCentral — Exploring and testing OLED VRR flicker](https://tftcentral.co.uk/articles/exploring-and-testing-oled-vrr-flicker)

### The 48 Hz Minimum Refresh Rate Floor

Most OLED gaming monitors and panels (including the Steam Deck OLED's BOE display) enforce a **minimum VRR refresh rate of 48 Hz** rather than allowing the VRR range to extend arbitrarily low. Below 48 Hz, the extended blanking period required to hold the display in VRR mode becomes long enough that the OLED panel's self-refresh characteristics begin to cause visible flicker or gamma drift that manufacturers consider unacceptable for consumer products.

The choice of 48 Hz is not arbitrary: it is the lowest standard cinema frame rate (cinema is often 24 fps with 3:2 pulldown to 48 Hz), ensuring compatibility with video content while staying above the threshold where OLED gamma drift becomes visually objectionable in controlled testing. Below the VRR minimum, the display triggers **Low Framerate Compensation (LFC)**: the panel's timing controller inserts duplicate frames at a multiple of the base frame rate to stay within the VRR operating range. For a 48–144 Hz OLED panel:

| GPU frame time | Effective display refresh | Mechanism |
|---|---|---|
| 7 ms (143 FPS) | 143 Hz | VRR — normal |
| 12 ms (83 FPS) | 83 Hz | VRR — normal |
| 20 ms (50 FPS) | 50 Hz | VRR — near minimum |
| 22 ms (45 FPS) | ~90 Hz | LFC — frame doubled to stay above 48 Hz minimum |
| 30 ms (33 FPS) | ~96 Hz | LFC — frame tripled |

The LFC transition can itself cause a visible brightness jump as the display crosses from VRR mode into the LFC repetition pattern, because the effective refresh rate jumps from just below 48 Hz to 96 Hz or higher in a single step. [Source: TFTCentral — OLED VRR flicker](https://tftcentral.co.uk/articles/exploring-and-testing-oled-vrr-flicker)

### Burn-In Mitigation

OLED panels are susceptible to **permanent burn-in** when the same bright pixels are illuminated at high intensity for long periods. Gaming monitors display static HUD elements (health bars, minimaps, crosshairs) for thousands of hours, making burn-in a real concern. Panel manufacturers have developed several mitigation mechanisms that operate within and alongside VRR:

**Pixel Shift (Screen Move):** The entire displayed image is shifted by a small number of pixels (typically 1–4 px) at regular intervals, spreading the pixel wear pattern across a slightly larger area rather than concentrating it on exactly the same subpixels. Samsung OLED monitors enable pixel shift by default. The shift is small enough to be imperceptible during normal use but is visible if you watch a static edge carefully over time.

```
/* Conceptual firmware behavior — not a Linux API */
every 30 minutes:
    shift_display_origin(x += 1 px, y += 1 px)  /* wraps at ±4 px range */
```

**Static Content Dimming (Automatic Brightness Management):** When the display controller or GPU driver detects that a large portion of the screen is showing static (unchanging) content, it reduces the peak luminance of that region. ASUS implements this as "Global Dimming Control" on their ROG OLED monitors. On Linux, this typically operates in monitor firmware without OS involvement.

**Pixel Refresh Cycles:** At predetermined intervals (or when the user powers off the display), the OLED panel runs an internal maintenance cycle: each pixel is driven briefly at full brightness to discharge any residual charge buildup. Manufacturers recommend allowing these cycles to complete rather than interrupting them.

**Anti-Flicker VRR Range Restrictions:** To reduce gamma-shift flicker, ASUS introduced "OLED Anti-Flicker" technology (2024) with selectable VRR floor modes:

| ASUS Anti-Flicker mode | VRR floor | LFC available |
|---|---|---|
| Off | 48 Hz | Yes |
| Middle | 80 Hz | Yes |
| High | 140 Hz | No (static 140 Hz) |

Raising the VRR floor reduces gamma variance because the refresh period stays in a narrower range. The trade-off is that LFC is required more frequently and the display cannot exploit the full VRR range for very low FPS content. [Source: TFTCentral — Helping avoid OLED burn-in and flicker](https://tftcentral.co.uk/articles/helping-avoid-oled-burn-in-and-flicker-exploring-the-latest-asus-oled-technologies-for-2025) Similar configurable floors are offered by MSI (OLED Anti-Flicker PRO with three levels) and LG (Flicker Safe setting).

### Linux Compositor Implications

From the compositor's perspective, OLED VRR introduces one important behavioral difference from LCD VRR: **VRR should be paired with a software frame rate floor** to avoid forcing the display below its VRR minimum and triggering LFC-induced flicker. Gamescope handles this through its `-r` flag, which sets a minimum presentation rate:

```bash
# Keep the display in the 48–144 Hz VRR range;
# Gamescope will pace frames at ≥48 FPS to avoid LFC transitions
gamescope --adaptive-sync -r 48 -- %command%
```

KWin accounts for the VRR minimum refresh rate when scheduling cursor compositing (improved in Plasma 6.4) so that mouse movement does not artificially hold the display at the minimum VRR rate when the application itself is running faster.

> **Note: needs verification** — Some OLED displays with HDMI 2.1 VRR have a higher effective VRR floor than their DisplayPort counterpart (e.g., 60 Hz minimum over HDMI vs. 48 Hz over DP) due to HDMI VRR timing constraints. Consult the specific monitor's specification for interface-dependent minimums.

---

## 15. Explicit Sync Interaction with VRR

### The Problem: Unsynchronized Buffer Submission Under VRR

Under a traditional implicit-sync Wayland compositor, the compositor knows a buffer is ready for scanout only when the kernel's implicit fence on the DMA-BUF signals. Under VRR, the display starts scanning each new frame the moment the KMS page flip occurs. If the compositor submits a flip for a buffer whose GPU rendering fence has not yet signalled, the display could begin scanning a partially-rendered buffer — producing corruption or requiring the compositor to delay the flip until the fence signals, adding latency.

**`linux-drm-syncobj-v1`** (part of Wayland Protocols 1.34, landed 2024) solves this by making synchronization explicit at the protocol level. Each `wl_surface.commit` now carries an **acquire point** — a DRM syncobj timeline point that the compositor must wait for before using the buffer — and a **release point** that the compositor signals when it is done with the buffer. [Source: GNOME — Mutter lands DRM Sync Obj v1 support](https://www.phoronix.com/news/GNOME-Linux-DRM-Sync-Obj-v1)

This is particularly important for NVIDIA hardware, where implicit fence propagation across driver boundaries was a long-standing source of rendering corruption on Wayland.

### Protocol Structure

The protocol defines three interfaces:

```xml
<!-- staging/linux-drm-syncobj/linux-drm-syncobj-v1.xml (abridged) -->

<interface name="wp_linux_drm_syncobj_manager_v1" version="1">
  <request name="destroy" type="destructor"/>
  <request name="get_surface">
    <!-- Returns a wp_linux_drm_syncobj_surface_v1 for the given wl_surface -->
    <arg name="id"      type="new_id"
         interface="wp_linux_drm_syncobj_surface_v1"/>
    <arg name="surface" type="object" interface="wl_surface"/>
  </request>
  <request name="import_timeline">
    <!-- Import a DRM syncobj fd as a timeline object -->
    <arg name="id" type="new_id"
         interface="wp_linux_drm_syncobj_timeline_v1"/>
    <arg name="fd" type="fd"/>
  </request>
</interface>

<interface name="wp_linux_drm_syncobj_surface_v1" version="1">
  <request name="set_acquire_point">
    <!-- Compositor waits for this timeline point before scanning the buffer -->
    <arg name="timeline" type="object"
         interface="wp_linux_drm_syncobj_timeline_v1"/>
    <arg name="point_hi" type="uint"/>  <!-- high 32 bits of 64-bit point -->
    <arg name="point_lo" type="uint"/>  <!-- low  32 bits of 64-bit point -->
  </request>
  <request name="set_release_point">
    <!-- Compositor signals this timeline point when done with the buffer -->
    <arg name="timeline" type="object"
         interface="wp_linux_drm_syncobj_timeline_v1"/>
    <arg name="point_hi" type="uint"/>
    <arg name="point_lo" type="uint"/>
  </request>
  <request name="destroy" type="destructor"/>
</interface>
```

[Source: Wayland Explorer — linux-drm-syncobj-v1](https://wayland.app/protocols/linux-drm-syncobj-v1)

### Client Usage with Vulkan

In a Vulkan application using explicit sync, the acquire timeline point corresponds to a `VkSemaphore` (backed by a DRM syncobj) that is signalled when the GPU finishes rendering a frame:

```c
/* 1. Export the render-complete semaphore as a DRM syncobj fd */
VkSemaphoreGetFdInfoKHR get_fd_info = {
    .sType      = VK_STRUCTURE_TYPE_SEMAPHORE_GET_FD_INFO_KHR,
    .semaphore  = render_semaphore,
    .handleType = VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_OPAQUE_FD_BIT,
};
int syncobj_fd;
vkGetSemaphoreFdKHR(device, &get_fd_info, &syncobj_fd);

/* 2. Import the fd as a Wayland timeline object */
struct wp_linux_drm_syncobj_timeline_v1 *timeline =
    wp_linux_drm_syncobj_manager_v1_import_timeline(syncobj_mgr, syncobj_fd);
close(syncobj_fd);

/* 3. Set the acquire point for the next commit — point 1 on the timeline */
wp_linux_drm_syncobj_surface_v1_set_acquire_point(syncobj_surface,
    timeline, 0, 1);   /* high=0, low=1 → point value 1 */

/* 4. Set the release point — compositor signals point 2 when done */
wp_linux_drm_syncobj_surface_v1_set_release_point(syncobj_surface,
    timeline, 0, 2);

/* 5. Commit — the compositor will not scan this buffer until point 1 signals */
wl_surface_attach(surface, dmabuf_buffer, 0, 0);
wl_surface_commit(surface);
```

Both acquire and release points must be set (or both unset) for every commit that attaches a buffer, as required by the protocol. [Source: Wayland Explorer — linux-drm-syncobj-v1](https://wayland.app/protocols/linux-drm-syncobj-v1)

### VRR VBLANK Timing and Explicit Sync

Under VRR, the display adapts its refresh period to the GPU's output cadence. The key constraint is that the KMS page flip — which tells the display to begin scanning the new frame — must not be issued until the acquire point has signalled (i.e., the GPU has finished rendering the frame). The compositor must:

1. Receive the `wl_surface.commit` with the acquire point set.
2. Queue the KMS atomic commit with the `IN_FENCE_FD` plane property pointing to the DRM syncobj fence.
3. The kernel DRM driver (e.g., AMDGPU) waits for the fence before issuing the actual VBLANK flip.

The DRM plane property used to pass the acquire fence to the kernel is `IN_FENCE_FD` (a per-plane atomic property carrying a `sync_file` fd that wraps the DRM syncobj fence). The CRTC property `OUT_FENCE_PTR` receives the release fence (a userspace pointer written by the kernel with a new `sync_file` fd that signals when the flip's scan-out completes). [Source: Linux kernel DRM — drm_syncobj.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_syncobj.c)

```
/* Compositor explicit sync path under VRR (pseudocode) */

/* Receive wp_linux_drm_syncobj_surface_v1 acquire point from client */
int acquire_fence_fd = drm_syncobj_to_sync_file(acquire_syncobj, acquire_point);

/* Build atomic commit with IN_FENCE_FD on the plane */
drmModeAtomicAddProperty(req, plane_id, in_fence_fd_prop, acquire_fence_fd);

/* OUT_FENCE_PTR — kernel fills this with the release fence fd */
uint64_t out_fence_ptr = (uint64_t)(uintptr_t)&release_fence_fd;
drmModeAtomicAddProperty(req, crtc_id, out_fence_ptr_prop, out_fence_ptr);

/* VRR_ENABLED is already set; the KMS driver will hold the VBLANK
   until the acquire fence signals, then flip and adapt the refresh period */
drmModeAtomicCommit(drm_fd, req,
    DRM_MODE_ATOMIC_NONBLOCK | DRM_MODE_PAGE_FLIP_EVENT, NULL);
```

[Source: Linux kernel — DRM atomic properties](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_atomic_uapi.c)

### The VRR + Explicit Sync Contract

The interaction between explicit sync and VRR imposes a concrete timing discipline: **frames cannot be submitted to the display at arbitrary times under VRR; they must arrive before the KMS VBLANK window**. The display's VRR timing controller holds the blanking period open waiting for a flip request. Once the flip request arrives (backed by a signalled acquire fence), the controller closes the blanking window and starts scanning. The frame's arrival time — determined by when the GPU signals the acquire fence — directly sets the display's refresh period for that frame.

This creates a clean causal chain:

```
GPU renders frame N
    ↓ (GPU signals acquire fence at timeline point N)
Kernel DRM receives page-flip (IN_FENCE_FD waits for acquire fence)
    ↓ (AMDGPU DC holds VRR blanking; flip lands)
Display controller begins scanning frame N
    ↓ (CRTC signals OUT_FENCE_PTR / release fence)
Compositor releases buffer N to client (via release timeline point)
    ↓
Client renders frame N+1 into the released buffer
```

Because the explicit sync protocol makes each step in this chain deterministic, there are no implicit fence races that could cause the display to scan a partially-rendered buffer. The VRR range constraints still apply: if the GPU takes longer than the display's maximum VRR period, LFC activates. If the GPU is faster than the minimum period, the compositor should hold the flip to avoid dropping below the minimum (which would cause LFC even when unnecessary).

**Atomic async page flips with explicit sync**: a patch series (v7, submitted June 2024) to add `DRM_MODE_PAGE_FLIP_ASYNC` support to the atomic interface also allows the `IN_FENCE_FD` property to be combined with async flips. This enables tearing presentation (via `wp_tearing_control_v1` with `ASYNC` hint) to also benefit from explicit sync — the async flip fires immediately when the fence signals, with tearing still bounded to a single frame boundary. [Source: LKML — atomic async page flip with explicit sync](https://lkml.iu.edu/hypermail/linux/kernel/2406.2/02446.html)

---

## Roadmap

### Near-term (6–12 months)

- **Arm Komeda VRR support**: The Komeda DRM/KMS display driver (used in Arm Mali display processors) is receiving Adaptive-Sync and HDMI VRR enablement patches, extending kernel VRR support beyond AMD and Intel to Arm-based SoCs and mobile/embedded Linux targets. [Source: Phoronix — Arm Komeda Adding VRR Support](https://www.phoronix.com/forums/forum/linux-graphics-x-org-drivers/x-org-drm/1111410-arm-s-komeda-driver-adding-variable-refresh-rate-support)
- **Intel Adaptive Sync SDP improvements**: Intel's i915/xe driver team posted patches in early 2026 to improve Adaptive Sync Secondary Data Packet (SDP) handling for Panel Replay and Auxless ALPM modes, important for VRR over DisplayPort protocol converters and eDP panels. [Source: Phoronix — Intel Adaptive Sync SDP Patches](https://www.phoronix.com/news/Intel-Adaptive-Sync-SDP-Patches)
- **AMD HDMI VRR desktop mode (`freesync_on_desktop`)**: A 2026 patch series adds `freesync_on_desktop` logic for HDMI VRR to mitigate blanking glitches when VRR is toggled on TVs and HDMI sinks, bringing HDMI VRR closer to the seamless behaviour of DisplayPort Adaptive-Sync. [Source: LKML — drm/amd/display: freesync_on_desktop for HDMI VRR](https://lkml.iu.edu/2602.2/00869.html)
- **HDMI VRRmax=0 support**: AMD display patches add support for `VRRmax=0` in HDMI VRR signalling (meaning the upper bound is capped by the selected video mode rather than an explicit value), improving compatibility with TVs that reject VTEM packets with high BRR values. [Source: LKML — drm/amd/display: Support HDMI VRRmax=0](https://lkml.iu.edu/hypermail/linux/kernel/2602.0/04026.html)
- **COSMIC compositor VRR**: The COSMIC desktop environment (System76) is expected to implement VRR support later in 2026 or early 2027, extending adaptive sync beyond KWin and Mutter to a third major Wayland compositor. [Source: fosslinux — Wayland Linux Gaming in 2026](https://www.fosslinux.com/157640/wayland-linux-gaming-2026-desktop-comparison.htm)

### Medium-term (1–3 years)

- **`wp_fifo_v1` and `wp_tearing_control_v1` wider compositor adoption**: Both protocols are stable in wayland-protocols but implementation remains patchy across compositors as of 2026. Broader rollout in KWin, Mutter, wlroots, and COSMIC is expected, with Godot and game engine runtimes adopting `fifo_v1` for explicit FIFO semantics. [Source: Godot — fifo_v1 implementation PR](https://github.com/godotengine/godot/pull/101454) / [Source: Wayland Explorer — fifo-v1](https://wayland.app/protocols/fifo-v1)
- **Atomic async page flip stabilisation**: The `DRM_MODE_PAGE_FLIP_ASYNC` atomic interface (combined with `IN_FENCE_FD` for explicit sync) is expected to progress from experimental patchset status toward mainline inclusion, enabling tear-present with deterministic sync across all drivers. Note: needs verification on merge timeline.
- **VRR + HDR joint property management**: Ongoing work in the AMD display core and upstream DRM to allow `VRR_ENABLED` and HDR metadata properties (peak luminance, static metadata) to be updated atomically in the same commit, removing current restrictions that require separate commits for VRR and HDR state transitions. Note: needs verification on specific kernel version target.
- **Refresh rate floor kernel property**: A proposed KMS property to set a minimum VRR refresh rate floor (to suppress OLED burn-in at very low refresh rates) is under design discussion. The intent is to expose it as an atomic CRTC property alongside `VRR_ENABLED`, with panel-specific defaults from EDID. Note: needs verification on upstream proposal status.
- **Per-plane VRR hint from Vulkan**: Extensions building on `VK_EXT_swapchain_maintenance1` and `VK_EXT_present_mode_fifo_latest_ready` may expose per-swapchain VRR timing hints to the Vulkan driver, allowing Mesa/RADV and NVIDIA to integrate frame pacing policy directly in the present path rather than relying on compositor heuristics. Note: needs verification.

### Long-term

- **Hardware frame pacing in display controller**: AMD's hardware frame pacing (HFP) on RDNA3+ offloads frame interpolation and pacing to the display engine rather than the shader core. Future RDNA generations (RDNA4 and beyond) may extend HFP to handle LFC transitions, stutter suppression, and VRR floor enforcement in silicon, reducing compositor and driver involvement. Note: needs verification on RDNA4 specifics.
- **HDMI VRR parity with DisplayPort**: HDMI VRR currently lacks seamless VRR toggle (unlike DisplayPort Adaptive-Sync) due to protocol differences. Future HDMI specifications and driver work aim to achieve blanking-free VRR enable/disable on par with DisplayPort, especially important for TV-connected desktop Linux systems.
- **AI/ML-driven frame pacing prediction**: Speculative longer-term direction involves compositor-level or driver-level ML models predicting frame time variance per workload to proactively adjust VRR timing windows, reducing LFC activations and jitter on variable GPU workloads. This parallels NVIDIA's Reflex latency-reduction pipeline and AMD's Anti-Lag2 technology.
- **Kernel-level frame timing API**: A unified kernel API (beyond `drm_vblank`) surfacing per-frame GPU completion timestamps, display scanout timestamps, and VRR period measurements to userspace (compositors and profiling tools) has been discussed as a long-term goal for enabling accurate, race-free frame pacing metrics without relying on userspace-only instrumentation.

---

## 16. Integrations

- **Ch2 — KMS Atomic Modesetting**: `VRR_ENABLED` is a KMS CRTC atomic property set via `drmModeAtomicAddProperty()`. VRR state is carried in `drm_crtc_state` during atomic check and commit. The atomic modesetting path (nonblocking commits, page-flip events) is the mechanism by which compositors update VRR state alongside buffer flips.

- **Ch20 — Wayland Protocol Fundamentals**: The `wp_presentation` protocol (presentation-time) is the Wayland mechanism for delivering precise frame timing feedback to clients. The `presented` event with its `refresh` and `flags` fields is the foundation for VRR-aware frame pacing on Wayland. The `discarded` event enables clients to detect dropped frames. See §5 of this chapter for protocol details.

- **Ch22 — Production Compositors (Mutter)**: GNOME Mutter added experimental VRR support in GNOME 46 (March 2024). Mutter enables VRR via the KMS backend by setting `VRR_ENABLED` on the CRTC in its atomic commit path. Cursor scheduling improvements in GNOME 49 fix a low-FPS cursor stutter regression introduced by early VRR implementation.

- **Ch21 — wlroots and wlr-based Compositors**: wlroots exposes VRR through `WLR_OUTPUT_STATE_ADAPTIVE_SYNC_ENABLED`. The wlr-output-management protocol v4 adds an `adaptive_sync` event to `zwlr_output_head_v1` for management clients (way-displays, wlr-randr). Sway (wlroots-based) supports per-output `adaptive_sync on/off` in its config.

- **Ch24 — Vulkan on Linux**: The Vulkan present mode (`VK_PRESENT_MODE_FIFO_KHR`, `FIFO_RELAXED`, `MAILBOX`, `IMMEDIATE`) is the application-level control for how frames interact with the display pipeline. `VK_EXT_swapchain_maintenance1` enables runtime present mode switching and `VkReleaseSwapchainImagesEXT` for mailbox without waste. On VRR Wayland displays, `FIFO` is the recommended mode.

- **Ch74 — HDR and Wide Color Gamut**: VRR and HDR are combined on modern OLED displays (Steam Deck OLED, HDR gaming monitors). The KMS pipeline must set both `VRR_ENABLED` and HDR metadata properties in the same atomic commit. AMD's DC layer supports VRR + HDR simultaneously; NVIDIA requires specific driver versions for combined support.

- **Ch78 — Gamescope and Steam Deck**: Gamescope is the primary production VRR user on Linux. Its `--adaptive-sync` flag, direct scanout path, frame rate limiter integration, and `wp_presentation` based frame timing measurement are described in §7 of this chapter. Steam Deck's gaming experience relies on VRR for smooth frame delivery at variable GPU workloads.

- **Ch93 — GPU Performance Analysis**: Frame time is the fundamental GPU performance metric. MangoHud's frame time graph, `vkCmdWriteTimestamp2` GPU queries, and the Gamescope performance overlay are the primary tools for identifying VRR anomalies (spikes below VRR minimum, LFC transitions). GPU pass profiling identifies which render stage causes frame time variance.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
