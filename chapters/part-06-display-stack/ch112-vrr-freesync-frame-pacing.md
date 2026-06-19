# Chapter 112: Variable Refresh Rate — FreeSync, G-Sync, and Frame Pacing

**Target audiences:** Game developers targeting Linux and Steam Deck, compositor authors integrating VRR into display pipelines, display engineers validating adaptive sync hardware, and application developers who need to understand present mode selection, frame pacing, and the interaction between Vulkan swapchains and kernel KMS properties.

---

## Table of Contents

1. [The Tearing and Stuttering Problem](#1-the-tearing-and-stuttering-problem)
2. [Display Refresh Fundamentals](#2-display-refresh-fundamentals)
3. [VRR Technologies: FreeSync and G-Sync](#3-vrr-technologies-freesync-and-g-sync)
4. [KMS VRR Support](#4-kms-vrr-support)
5. [Wayland Presentation Feedback](#5-wayland-presentation-feedback)
6. [Frame Pacing Strategies](#6-frame-pacing-strategies)
7. [Gamescope VRR Implementation](#7-gamescope-vrr-implementation)
8. [Mutter and KWin VRR](#8-mutter-and-kwin-vrr)
9. [Vulkan Swapchain and VRR](#9-vulkan-swapchain-and-vrr)
10. [Measuring and Debugging Frame Timing](#10-measuring-and-debugging-frame-timing)
11. [Integrations](#11-integrations)

---

## 1. The Tearing and Stuttering Problem

Every frame a GPU renders must eventually appear on a display. The fundamental challenge is that GPU rendering time is variable — it depends on scene complexity, driver overhead, thermal throttling, and hundreds of other factors — while traditional displays refresh at a fixed, immutable rate. This mismatch between variable GPU output and fixed display cadence is the root cause of two of the most visible quality problems in real-time graphics: *tearing* and *stuttering*.

**Tearing** occurs when the GPU writes a new frame into the framebuffer while the display controller is in the middle of scanning out the previous frame. The result is a horizontal discontinuity where the top portion of the screen shows the new frame and the bottom shows the old one. This is especially visible during fast camera pans in games or when scrolling long pages.

**Stuttering** occurs when the GPU cannot sustain the display's fixed refresh rate. At 60 Hz, the display expects a new frame every 16.67 ms. If the GPU takes 20 ms to render a frame, it misses the VBLANK window. The display must then show the previous frame for another full refresh period — 33.33 ms total — while the next frame completes. The sudden doubling of frame duration is perceived as a stutter even if the average frame rate is 50 FPS.

The traditional fix for tearing is **VSync** (vertical synchronization): the application or compositor delays buffer swap until the display reaches its VBLANK interval — the brief period between scan-out cycles when the display is not updating pixel rows. VSync eliminates tearing because the display never reads from a framebuffer that is being written. However, VSync does not solve stuttering; it merely converts the problem. A frame that takes 17 ms (just over one 60 Hz period) must now wait for the *next* VBLANK, producing the same 33.33 ms presentation latency. VSync also introduces up to one full frame of input latency because the application must queue the swap before VBLANK occurs.

**Variable Refresh Rate (VRR)** resolves both problems simultaneously. Rather than the display driving scan-out at a fixed cadence, a VRR-capable display waits for a signal from the GPU before beginning the next refresh cycle. The display adapts its refresh period to match the GPU's actual output timing. If the GPU delivers a frame in 8 ms, the display refreshes in 8 ms; if the next frame takes 12 ms, the next refresh period is 12 ms. Tearing cannot occur because the display only starts scanning a frame after the GPU signals it is complete. Stutter cannot occur because there is no fixed period for a frame to "miss." The GPU and display are effectively locked together by a handshake rather than an external clock.

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

## 11. Integrations

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
