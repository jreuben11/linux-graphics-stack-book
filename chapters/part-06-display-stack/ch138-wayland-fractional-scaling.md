# Chapter 138: Wayland Fractional Scaling and HiDPI

**Target audiences**: Wayland compositor developers implementing fractional scaling, application developers targeting HiDPI displays, toolkit maintainers integrating `wp_fractional_scale_v1`, and systems engineers managing mixed-DPI multi-monitor setups.

---

## Table of Contents

1. [Introduction](#introduction)
2. [The HiDPI Problem](#the-hidpi-problem)
3. [wl_output.scale: Integer Scaling Baseline](#wl_outputscale-integer-scaling-baseline)
4. [wp_fractional_scale_v1: The Protocol](#wp_fractional_scale_v1-the-protocol)
5. [wp_fractional_scale_v1 Deep Dive](#wp_fractional_scale_v1-deep-dive)
6. [Buffer Scaling Math and Viewport Protocol Sequence](#buffer-scaling-math-and-viewport-protocol-sequence)
7. [Compositor Implementation](#compositor-implementation)
8. [Compositor Implementation Internals](#compositor-implementation-internals)
9. [Font and Text Rendering Under Fractional Scale](#font-and-text-rendering-under-fractional-scale)
10. [Toolkit Integration](#toolkit-integration)
11. [Mixed-DPI Multi-Monitor Setups](#mixed-dpi-multi-monitor-setups)
12. [XWayland Fractional Scaling — Deep Dive](#xwayland-fractional-scaling--deep-dive)
13. [XWayland and X11 Comparison](#xwayland-and-x11-comparison)
14. [GPU Implications of Fractional Scaling](#gpu-implications-of-fractional-scaling)
15. [Cursor, Icons, and Subpixel Rendering](#cursor-icons-and-subpixel-rendering)
16. [Integrations](#integrations)

---

## Introduction

High-density displays — 4K at 15 inches (≈293 PPI), 5K at 27 inches (≈218 PPI), or 2560×1600 at 14 inches (≈214 PPI) — need UI scaling to keep text and controls at comfortable sizes. At 1× (no scaling), interface elements become unreadably small. At 2× (pixel doubling), a 4K 15-inch display has only 1920×1080 logical pixels, wasting half its native resolution on 14–16 inch laptops.

Fractional scaling at 1.25×, 1.5×, or 1.75× threads the needle: comfortable UI element sizes with more logical pixels than 2× integer scaling. Wayland's original `wl_output.scale` supports only integers. The `wp_fractional_scale_v1` protocol (introduced in wayland-protocols 1.31, 2022; stabilised 2023) provides a standardised mechanism for compositors to inform surfaces of their preferred fractional scale, and for clients to render appropriately.

This chapter covers the full HiDPI story on Wayland: the underlying problem, the integer scaling baseline, the `wp_fractional_scale_v1` protocol mechanics, compositor and toolkit implementations, and the practical challenges of mixed-DPI multi-monitor setups.

[wayland-protocols fractional-scale](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/fractional-scale/fractional-scale-v1.xml)

---

## The HiDPI Problem

### Physical Pixel Density

Display PPI (pixels per inch) determines how large UI elements appear at 1:1 pixel mapping:

| Display | Resolution | Size | PPI |
|---|---|---|---|
| FHD laptop | 1920×1080 | 15.6" | 141 |
| QHD laptop | 2560×1440 | 14" | 210 |
| 4K laptop | 3840×2160 | 15.6" | 282 |
| 4K desktop | 3840×2160 | 27" | 163 |
| 5K desktop | 5120×2880 | 27" | 218 |

At 1× on a 4K/15.6" display, a 16pt font renders ≈4mm tall — marginal legibility for most adults. At 2×, the same font is ≈8mm (comfortable), but logical resolution drops to 1920×1080.

### Device Pixel Ratio

The **device pixel ratio** (DPR) is the ratio of physical pixels to logical (CSS) pixels. A DPR of 1.5 means each logical pixel spans 1.5×1.5 physical pixels. At DPR 1.5 on a 3840×2160 display, the logical resolution is 2560×1440 — the sweet spot for a 14-inch QHD display.

### The Pre-Fractional Workarounds

Before `wp_fractional_scale_v1`, applications had to choose between:

1. **Integer scale=1** — everything too small
2. **Integer scale=2** — less logical space, pixel-perfect
3. **Render at 2×, display scaled to logical** — "fake" 1.5× by rendering at scale=2 then compositing to a smaller viewport. Blurry if the compositor downsamples, and wastes GPU memory.
4. **Application-side scaling** — GTK at `GDK_SCALE=2` then `GDK_DPI_SCALE=0.75` (ugly hack)

---

## wl_output.scale: Integer Scaling Baseline

### Protocol Mechanics

`wl_output.scale(factor)` (added in Wayland 1.9) sends the compositor's preferred integer scale for an output to clients. Clients use `wl_surface.set_buffer_scale(scale)` to inform the compositor that their buffer is rendered at the given scale.

With `scale=2`:
- Client renders a 3840×2160 buffer for a logical 1920×1080 window
- Compositor maps this to the display at 1:1 physical pixel ratio
- Text and UI elements appear physically the same size as on an FHD display

### Environment Variable Integration

```bash
# GTK3/GTK4: set the integer scale
GDK_SCALE=2 gnome-terminal

# Qt5/Qt6
QT_SCALE_FACTOR=2 dolphin

# Force a specific DPI (X11-style, works in some XWayland contexts):
export GDK_DPI_SCALE=1.0  # with GDK_SCALE=2 for exact 2× scaling
```

### Limitations

Integer scaling is pixel-perfect and has zero blur, but is too coarse:

- `scale=1` is too small on HiDPI laptops
- `scale=2` wastes logical space (4K/15" → 1920×1080 logical)
- `scale=3` is useful for phones (no Linux desktop targets this)

The need for 1.5×, 1.25×, 1.75× is fundamental for comfortable desktop usage on HiDPI panels under 24 inches.

---

## wp_fractional_scale_v1: The Protocol

### Overview

`wp_fractional_scale_v1` is a Wayland staging protocol that allows the compositor to communicate a **fractional preferred scale** to a specific surface, and allows the client to render at the appropriate resolution with viewport scaling to achieve correct compositing.

[Protocol XML](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/fractional-scale/fractional-scale-v1.xml)

### Protocol Flow

```xml
<!-- XML excerpt from fractional-scale-v1.xml -->
<interface name="wp_fractional_scale_manager_v1" version="1">
  <request name="get_fractional_scale">
    <arg name="id" type="new_id" interface="wp_fractional_scale_v1"/>
    <arg name="surface" type="object" interface="wl_surface"/>
  </request>
</interface>

<interface name="wp_fractional_scale_v1" version="1">
  <event name="preferred_scale">
    <!-- scale = (desired scale × 120), e.g. 1.5× → 180 -->
    <arg name="scale" type="uint"/>
  </event>
  <request name="destroy" type="destructor"/>
</interface>
```

The `scale` value is a **fixed-point number with denominator 120**:
- 1.0× → scale = 120
- 1.25× → scale = 150
- 1.5× → scale = 180
- 1.75× → scale = 210
- 2.0× → scale = 240

### Client-Side Implementation

The client must cooperate with `wp_viewport` (from the `viewporter` protocol) to inform the compositor of the logical size:

```c
/* 1. Create fractional scale object */
struct wp_fractional_scale_v1 *frac_scale =
    wp_fractional_scale_manager_v1_get_fractional_scale(
        frac_scale_manager, surface);

wp_fractional_scale_v1_add_listener(frac_scale,
    &frac_scale_listener, window);

/* 2. Create viewport */
struct wp_viewport *viewport =
    wp_viewporter_get_viewport(viewporter, surface);

/* 3. In preferred_scale event handler: */
static void on_preferred_scale(void *data,
    struct wp_fractional_scale_v1 *fs, uint32_t scale_120)
{
    Window *w = data;
    double scale = scale_120 / 120.0;  // e.g. 1.5

    /* Buffer must be ceil(logical × scale) */
    int buf_w = (int)ceil(w->logical_w * scale);
    int buf_h = (int)ceil(w->logical_h * scale);

    /* IMPORTANT: wl_surface.set_buffer_scale MUST stay at 1 */
    wl_surface_set_buffer_scale(surface, 1);

    /* Viewport destination = logical size */
    wp_viewport_set_destination(viewport, w->logical_w, w->logical_h);

    /* Render buffer at buf_w × buf_h and attach */
    render_at_size(w, buf_w, buf_h, scale);
}
```

**Critical constraint**: `wl_surface.set_buffer_scale` must remain 1 when using `wp_fractional_scale_v1`. The viewport's `set_destination` is what tells the compositor the logical (surface-coordinate) size; the buffer's extra resolution provides the high-quality source for the compositor's downsampling.

### Why ceil() Not round()

Using `ceil()` guarantees the buffer is always at least as large as needed. If the logical size is 1000 pixels and scale is 1.5, `ceil(1500)=1500`. If logical is 1001 and scale is 1.5, `ceil(1501.5)=1502`. Using `round()` can produce a buffer 1 pixel too small, causing compositor artefacts at surface edges.

---

## Compositor Implementation

### GNOME Mutter

Mutter experimental fractional scaling support appeared in GNOME 43 (2022), with the feature becoming production-quality in GNOME 47 (2024). Mutter's implementation:

- `MetaWaylandFractionalScaleManager` registers the `wp_fractional_scale_manager_v1` global
- `MetaWaylandSurface` tracks `effective_scale` which may be fractional
- When a window moves between outputs with different scales, Mutter re-sends `preferred_scale`
- **Old path** (pre-47): Mutter rendered everything at 2× then downsampled to fractional — blurry. Enabled via `gsettings set org.gnome.mutter experimental-features "['scale-monitor-framebuffer']"`
- **New path** (47+): Mutter uses native fractional rendering via `wp_fractional_scale_v1`

```bash
# Enable in older GNOME:
gsettings set org.gnome.mutter experimental-features "['scale-monitor-framebuffer']"

# Set scale in GNOME Settings or via gnome-randr-rust:
gnome-randr modify eDP-1 --scale 1.5
```

### KWin (KDE Plasma 6)

KWin implements fractional scaling natively since Plasma 6 (2024):

- `KWaylandServer::FractionalScaleManagerV1Interface` in `src/wayland/fractional_scale_v1.cpp`
- `KWin::Output::scale()` returns a `qreal` fractional value
- Compositor renders surfaces at native buffer resolution and composites

```bash
# Set output scale in KDE:
kscreen-doctor output.eDP-1.scale.1.5
# Or via Plasma Settings → Display Configuration
```

### wlroots 0.17+

wlroots 0.17 introduced `wp_fractional_scale_v1` support:

```c
// wlroots compositor code (conceptual):
struct wlr_fractional_scale_manager_v1 *mgr =
    wlr_fractional_scale_manager_v1_create(server->wl_display, 1);

// When output scale changes or surface enters output:
wlr_fractional_scale_v1_notify_scale(surface, output->scale);
```

Sway (based on wlroots) propagates scale from `wlr_output` to surfaces:

```bash
# sway config:
output eDP-1 scale 1.5
```

### Hyprland

Hyprland supports per-output fractional scale:

```bash
# hyprland.conf:
monitor = eDP-1, 2560x1600@165, 0x0, 1.5
monitor = DP-1, 2560x1440@144, 2560x0, 1.0

# Or at runtime:
hyprctl keyword monitor eDP-1,2560x1600,0x0,1.5
```

Hyprland uses its own compositor rendering path with Vulkan blitter and handles fractional scale via `wp_fractional_scale_v1`.

### Floor vs Ceil: The Compositor Side

Compositors face a related question: when a window's logical size is fractional in physical pixels, should they floor or ceil the buffer allocation? The consensus from wlroots and Mutter is:

- **Logical → physical**: `ceil(logical × scale)` for buffer requests to clients
- **Physical → logical**: `floor(physical / scale)` for mouse coordinate transforms (avoid overshooting)

---

## Toolkit Integration

### GTK4

GTK4's Wayland backend (`gdk/wayland/gdkwaylandsurface.c`) binds `wp_fractional_scale_v1` automatically when available:

```c
// In GdkWaylandSurface:
static void gdk_wayland_surface_handle_preferred_scale_changed(
    GdkWaylandSurface *surface, double scale)
{
    surface->scale = scale;
    gdk_surface_set_scale(GDK_SURFACE(surface), scale);
    // Triggers re-layout and re-render at new scale
}
```

Cairo surfaces in GTK4 use `cairo_surface_set_device_scale(cr, scale, scale)` so that all drawing operations are automatically scaled. Pango text rendering uses FreeType with fractional device units for sub-pixel correct glyph positioning.

```python
# Python/GTK4: no explicit fractional scale code needed
# GTK4 handles wp_fractional_scale_v1 transparently
import gi
gi.require_version('Gtk', '4.0')
from gi.repository import Gtk
# window.get_scale_factor() returns integer (floor), but internal rendering is fractional
```

### Qt6

Qt6's Wayland QPA (`src/plugins/platforms/wayland/`) implements fractional scaling:

```cpp
// QWaylandWindow handles wp_fractional_scale_v1:
void QWaylandWindow::setPreferredScale(qreal scale) {
    m_scale = scale;
    // Triggers QHighDpiScaling::factor() update
    QWindowSystemInterface::handleWindowDevicePixelRatioChanged(window());
}
```

```bash
# Qt6 environment variables (may be needed for older Qt5 apps):
QT_SCALE_FACTOR=1.5      # explicit scale
QT_AUTO_SCREEN_SCALE_FACTOR=1  # auto-detect (Qt5 only, Qt6 uses Wayland)
```

### Electron/Chromium

Electron (and Chrome/Chromium) respect `wp_fractional_scale_v1` when using the Wayland backend:

```bash
# Enable Wayland native (required for fractional scale):
ELECTRON_OZONE_PLATFORM_HINT=auto myapp
# Or:
myapp --ozone-platform=wayland

# Force DPR (overrides Wayland-negotiated scale):
myapp --force-device-scale-factor=1.5
```

In Electron/Chrome, `window.devicePixelRatio` reflects the negotiated fractional scale when running natively on Wayland.

### SDL3

SDL3 provides fractional scale support via:

```c
float density = SDL_GetWindowPixelDensity(window);
// density is the device pixel ratio (e.g. 1.5)
int physical_w, physical_h;
SDL_GetWindowSizeInPixels(window, &physical_w, &physical_h);
// Render at physical_w × physical_h
```

```bash
# SDL3 Wayland hint:
SDL_HINT_VIDEO_WAYLAND_SCALE_TO_DISPLAY=1
```

### Firefox

Firefox uses `wp_fractional_scale_v1` natively when `MOZ_ENABLE_WAYLAND=1`:

```bash
MOZ_ENABLE_WAYLAND=1 firefox
# Or set in /etc/environment or ~/.profile for permanent effect
```

Firefox's WebRender GPU compositor passes the fractional DPR to CSS, so `window.devicePixelRatio` in web content reflects 1.5, 1.25, etc.

---

## Mixed-DPI Multi-Monitor Setups

### The Common Scenario

A developer docking station: 4K/14" laptop panel (1.5× = 2560×1600 logical) + 1080p/24" external monitor (1.0× = 1920×1080 logical). Managing surfaces that span or move between these outputs requires the compositor to re-negotiate scale.

### Surface Output Tracking

Wayland surfaces track which outputs they are on:

- `wl_surface.enter(output)` — surface has entered this output
- `wl_surface.leave(output)` — surface has left this output

The compositor sends `wp_fractional_scale_v1.preferred_scale` with the **effective** scale for the surface (typically the scale of the output containing the majority of the surface area).

### Configuration Tools

```bash
# wlr-randr (wlroots-based compositors):
wlr-randr --output eDP-1 --scale 1.5
wlr-randr --output DP-1 --scale 1.0
wlr-randr  # list current state

# kscreen-doctor (KDE):
kscreen-doctor output.eDP-1.scale.1.5 output.DP-1.scale.1.0

# Hyprland:
hyprctl monitors  # list monitors with scale
hyprctl keyword monitor DP-1,1920x1080,2560x0,1.0
```

### Cursor Scaling Across Outputs

Cursors must match the output scale. Wayland handles this via the `wl_pointer.set_cursor` request: the client provides a cursor surface with the appropriate resolution, and `wl_surface_set_buffer_scale` (integer) or `wp_fractional_scale_v1` on the cursor surface. Most compositors handle cursor scaling automatically from the cursor theme.

```bash
# Set cursor size appropriately for HiDPI:
XCURSOR_SIZE=48 XCURSOR_THEME=Adwaita sway

# Or in sway config:
seat seat0 xcursor_theme Adwaita 48
```

The `hyprcursor` format stores multiple scale variants in a single cursor theme package, solving the blur problem for fractional scales.

### "Blurry on Move" Problem

When a window moves from a 1.5× output to a 1.0× output, the compositor sends a new `preferred_scale`. Until the client re-renders at the new scale and commits, the compositor must stretch or shrink the existing buffer — causing temporary blur. This is an unavoidable transient effect; compositors may hide it with a fade or simply accept the momentary blur.

### wl_surface.preferred_buffer_scale

Wayland 1.22 (2023) added `wl_surface.preferred_buffer_scale(factor)` as a simpler integer-only alternative signal. This allows clients to receive an integer scale hint even without `wp_fractional_scale_v1`, improving compatibility for clients that only handle integer scaling.

---

## XWayland and X11 Comparison

### X11 HiDPI

X11 uses a global DPI setting:

```bash
# Set DPI in Xresources:
echo "Xft.dpi: 192" >> ~/.Xresources
xrdb -merge ~/.Xresources

# Or via xrandr:
xrandr --dpi 192

# For 1.5× effective DPI on 1920×1080 screen at 96 DPI:
xrandr --dpi 144  # 96 × 1.5

# GTK on X11:
GDK_SCALE=2 GDK_DPI_SCALE=0.75  # effective 1.5× on 2× integer scale
```

X11 HiDPI is screen-wide; per-output scaling requires multiple X screens (Zaphod mode) which has poor multi-monitor support.

### XWayland and Fractional Scale

XWayland bridges X11 applications to Wayland compositors. XWayland renders at an integer scale:

```bash
# XWayland DPI hint:
Xwayland -dpi 144  # 144 DPI for 1.5× effective scale
```

XWayland does **not** implement `wp_fractional_scale_v1` for X11 clients — it sets a screen DPI and integer scale, then the compositor scales the XWayland surface. This means X11 apps appear blurry on Wayland with fractional scaling, since XWayland is just another Wayland client receiving the scale hint, but individual X11 apps cannot respond to per-surface scale changes.

X11 applications running natively (not via XWayland) can use `Xft.dpi` for DPI-aware rendering, but lack per-output scaling. The only real solution for perfect HiDPI on Wayland is native Wayland clients using `wp_fractional_scale_v1`.

### Checking Protocol Support

```bash
# List Wayland globals (check for fractional scale):
wayland-info | grep fractional

# Wayland debug log:
WAYLAND_DEBUG=1 gtk4-demo 2>&1 | grep fractional
```

---

## Cursor, Icons, and Subpixel Rendering

### Cursor Themes at Fractional Scale

X cursor themes store cursors at fixed pixel sizes (typically 16, 24, 32, 48, 64 px). At 1.5× scale, a 24px cursor is upscaled to 36px logical — blurry if nearest-neighbour, slightly blurry if bilinear.

Best practice:
```bash
# Use a cursor size divisible by scale for pixel-perfect results:
# For 1.5× scale: use 48px cursor (48/1.5 = 32 logical → still fine)
XCURSOR_SIZE=48

# hyprcursor stores SVG-rendered variants at many sizes:
hyprctl setcursor Bibata-Modern-Classic 32
```

### Icon Scaling

GTK4 loads icons from `hicolor` theme at `icon_size × scale` pixels. For a 1.5× scale requesting a 48×48 icon, GTK4 looks for a 72×72 PNG or renders the SVG at 72×72. Most modern icon themes include SVG icons that render correctly at any scale.

### Subpixel Rendering and Fractional Scale

At integer scales, LCD subpixel rendering (ClearType-style) works perfectly: each physical pixel maps cleanly to one logical pixel. At fractional scales, the mapping is non-integer, making subpixel positioning more critical.

FreeType 2 with HarfBuzz:
- `FT_LOAD_TARGET_LCD` for RGB subpixel rendering
- Fractional advance widths via `FT_LOAD_NO_HINTING` for smooth text at arbitrary scale
- Pango uses `pango_cairo_create_layout()` which passes fractional device units to FreeType

At 1.5× scale, text rendering is slightly softer than at 2× but significantly better than 1× upscaling. The practical difference at typical reading distances on high-PPI displays is minimal.

### Font Configuration

```bash
# ~/.config/fontconfig/fonts.conf for HiDPI:
cat > ~/.config/fontconfig/fonts.conf << 'EOF'
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>
  <match target="font">
    <edit name="antialias" mode="assign"><bool>true</bool></edit>
    <edit name="hinting" mode="assign"><bool>true</bool></edit>
    <edit name="hintstyle" mode="assign"><const>hintslight</const></edit>
    <edit name="rgba" mode="assign"><const>rgb</const></edit>
    <edit name="lcdfilter" mode="assign"><const>lcddefault</const></edit>
  </match>
</fontconfig>
EOF
```

```bash
# GNOME font antialiasing setting:
gsettings set org.gnome.settings-daemon.plugins.xsettings antialiasing rgba
gsettings set org.gnome.settings-daemon.plugins.xsettings hinting slight
```

---

## wp_fractional_scale_v1 Deep Dive

### Full Protocol XML

The canonical protocol XML lives at
[`staging/fractional-scale/fractional-scale-v1.xml`](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/fractional-scale/fractional-scale-v1.xml)
in the `wayland-protocols` repository. The full text of both interfaces is reproduced below with all description elements retained:

```xml
<!-- staging/fractional-scale/fractional-scale-v1.xml (MIT, © 2022 Kenny Levinsen) -->
<protocol name="fractional_scale_v1">

  <interface name="wp_fractional_scale_manager_v1" version="1">
    <description summary="fractional surface scale information">
      A global interface for requesting surfaces to be presented at a
      fractional scale factor.
      A client can use this interface to request that the compositor
      present its surface at a particular scale factor. The compositor
      will attempt to present the surface at the requested scale, but
      it may present the surface at a different scale if the request
      cannot be satisfied.
    </description>

    <request name="destroy" type="destructor">
      <description summary="unbind the manager">
        Informs the server that the client will not be using this
        protocol object anymore. This does not affect any other objects,
        wp_fractional_scale_v1 objects included.
      </description>
    </request>

    <enum name="error">
      <entry name="fractional_scale_exists" value="0"
        summary="the surface already has a fractional_scale object associated"/>
    </enum>

    <request name="get_fractional_scale">
      <description summary="bind fractional surface scale to a surface">
        Create an add-on object for the the wl_surface to let the compositor
        request fractional scales. If the given wl_surface already has a
        wp_fractional_scale_v1 object associated, the fractional_scale_exists
        protocol error is raised.
      </description>
      <arg name="id" type="new_id" interface="wp_fractional_scale_v1"/>
      <arg name="surface" type="object" interface="wl_surface"/>
    </request>
  </interface>

  <interface name="wp_fractional_scale_v1" version="1">
    <description summary="fractional scale interface to a wl_surface">
      An additional interface to a wl_surface object which allows the
      compositor to inform the client of the preferred scale.
    </description>

    <request name="destroy" type="destructor">
      <description summary="remove scale information for surface">
        Removes the fractional scale object and the preferred_scale events
        from the surface. The preferred scale of the surface will no longer
        be changed by the compositor.
      </description>
    </request>

    <event name="preferred_scale">
      <description summary="preferred scale for the surface">
        Notification of a new preferred scale for this surface that the
        compositor suggests that the client should use.

        The sent scale is the numerator of a fraction with a denominator of
        120.
      </description>
      <!-- numerator of a fraction with denominator 120 -->
      <arg name="scale" type="uint"/>
    </event>
  </interface>

</protocol>
```

[Source: wayland-protocols `fractional-scale-v1.xml`](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/fractional-scale/fractional-scale-v1.xml)

### The preferred_scale Event in Detail

The `preferred_scale` event carries a single `uint` argument named `scale`. There is no separate numerator/denominator pair on the wire; the denominator is **always 120**, fixed by the protocol specification. The client divides the received integer by 120 to obtain the real-valued scale factor:

| Received `scale` | Computed factor | Common name |
|---|---|---|
| 96 | 0.8× | (unusual sub-unity) |
| 120 | 1.0× | 1× |
| 150 | 1.25× | 125% |
| 180 | 1.5× | 150% |
| 210 | 1.75× | 175% |
| 240 | 2.0× | 200% |
| 360 | 3.0× | 300% (phone/tablet) |

The choice of 120 as denominator is deliberate: 120 = 2³ × 3 × 5, so it is evenly divisible by 1, 2, 3, 4, 5, 6, 8, 10, 12, 15, 20, 24, 30, 40, 60, and 120. This covers all common scale fractions (1.25 = 150/120, 1.333… ≈ 160/120 ≈ 4/3, 1.5 = 180/120, 1.75 = 210/120, 2.5 = 300/120) with integers, and avoids floating-point representation errors on the wire. [Source: protocol description](https://wayland.app/protocols/fractional-scale-v1)

### Comparison with wl_output.scale

`wl_output.scale` (Wayland 1.9, 2015) communicates a **global integer** scale for an output to all clients. `wp_fractional_scale_v1.preferred_scale` (wayland-protocols 1.31, 2022) communicates a **per-surface fractional** scale to exactly one surface object.

| Property | `wl_output.scale` | `wp_fractional_scale_v1` |
|---|---|---|
| Granularity | Integer only (1, 2, 3…) | Any fraction with /120 denominator |
| Scope | Per-output, broadcast to all clients | Per-surface, unicast |
| Client cooperation | `wl_surface.set_buffer_scale(N)` | `wp_viewport.set_destination()` + buffer at fractional size |
| Stability | Core protocol, stable | Staging protocol (wayland-protocols) |
| Year stabilised | 2015 | 2023 |
| Blur at 1.5×? | Yes — compositor must rescale 2× buffer | No — client renders at exact physical size |

The old integer path can approximate 1.5× by having the client render at 2× and relying on the compositor to downsample. This introduces one extra blit/downscale in the GPU render loop and produces slightly soft edges at subpixel boundaries. The `wp_fractional_scale_v1` path eliminates both the wasted pixels and the blitting overhead.

A surface that supports `wp_fractional_scale_v1` must **not** also set `wl_surface.set_buffer_scale` to anything other than 1. Mixing the two signals is undefined behaviour and compositors are not required to handle it gracefully.

---

## Buffer Scaling Math and Viewport Protocol Sequence

### The Three Coordinate Spaces

When `wp_fractional_scale_v1` is in use, a surface operates in three distinct coordinate spaces:

1. **Logical (surface) coordinates** — the coordinate space used for Wayland input events (`wl_pointer`, `wl_touch`), window geometry, and `xdg_surface.set_window_geometry`. Unit: logical pixels.
2. **Physical (buffer) coordinates** — the size of the `wl_buffer` attached to the surface. Unit: physical device pixels.
3. **Compositor coordinates** — the output's physical pixel grid onto which the compositor maps the surface. Unit: physical device pixels.

The relationship is:

```
physical_size = ceil(logical_size × fractional_scale / 120)
```

More precisely (using the received `scale_uint` value):

```c
/* scale_uint is the raw value from preferred_scale, e.g. 180 for 1.5× */
int buf_w = (int)ceil((double)logical_w * scale_uint / 120.0);
int buf_h = (int)ceil((double)logical_h * scale_uint / 120.0);
```

`ceil()` is mandatory. Using `floor()` can produce a buffer one pixel too small at certain sizes, causing the compositor to see a mismatch between the viewport destination and buffer size. Using `round()` has a 50% chance of being one pixel short at half-integer boundary sizes. The protocol authors specify ceil-based rounding in the reference implementation.

### wp_viewport Request Signatures

`wp_viewport` is defined in
[`stable/viewporter/viewporter.xml`](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/stable/viewporter/viewporter.xml).
The two requests used with fractional scaling are:

```c
/* From stable/viewporter/viewporter.xml */

/* set_source(x, y, width, height: wl_fixed_t)
 *
 * Defines which portion of the buffer to display.
 * Coordinates are in post-transform, post-(wl_buffer_scale) buffer space.
 * Pass -1.0 for all four values to unset (display entire buffer).
 * Raises bad_value if negative x/y or zero/negative dimensions.
 * Raises out_of_buffer if area exceeds buffer boundaries.
 */
wp_viewport_set_source(viewport,
    wl_fixed_from_double(0.0),          /* x = 0 */
    wl_fixed_from_double(0.0),          /* y = 0 */
    wl_fixed_from_double(buf_w),        /* width  = physical buffer width  */
    wl_fixed_from_double(buf_h));       /* height = physical buffer height */

/* set_destination(width, height: int)
 *
 * Sets the displayed surface dimensions in surface-local (logical) coordinates.
 * The source rectangle is scaled to fill these dimensions exactly.
 * Pass -1, -1 to unset.
 * Raises bad_value if zero or negative (except the -1,-1 unset case).
 */
wp_viewport_set_destination(viewport, logical_w, logical_h);
```

`set_source` takes `wl_fixed_t` (24.8 fixed-point) arguments, allowing sub-pixel source crops. `set_destination` takes plain `int` arguments, enforcing integer logical dimensions. [Source: viewporter.xml](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/stable/viewporter/viewporter.xml)

### Complete Protocol Sequence

The exact order of Wayland requests matters because `wl_surface.commit` atomically applies the pending state. The canonical sequence for a resize or scale-change event is:

```c
/* ---- Step 1: Bind globals (done once at startup) ---- */
struct wp_fractional_scale_manager_v1 *frac_mgr = /* bind from wl_registry */;
struct wp_viewporter *viewporter = /* bind from wl_registry */;

/* ---- Step 2: Create per-surface objects (done once per surface) ---- */
struct wp_fractional_scale_v1 *frac_scale =
    wp_fractional_scale_manager_v1_get_fractional_scale(frac_mgr, surface);
wp_fractional_scale_v1_add_listener(frac_scale, &frac_scale_listener, window);

struct wp_viewport *viewport =
    wp_viewporter_get_viewport(viewporter, surface);

/* Keep wl_buffer_scale at 1 — never call wl_surface_set_buffer_scale(surface, N>1) */
wl_surface_set_buffer_scale(surface, 1);

/* ---- Step 3: In preferred_scale handler ---- */
static void handle_preferred_scale(void *data,
    struct wp_fractional_scale_v1 *fs, uint32_t scale_uint)
{
    Window *w = data;
    w->scale_uint = scale_uint;                      /* e.g. 180 */
    double scale = scale_uint / 120.0;               /* e.g. 1.5 */

    /* Compute buffer dimensions */
    w->buf_w = (int)ceil(w->logical_w * scale);
    w->buf_h = (int)ceil(w->logical_h * scale);

    /* Allocate or resize the backing buffer (e.g. via wl_shm or EGL) */
    resize_buffer(w, w->buf_w, w->buf_h);

    /* --- The following three requests form an atomic unit committed together --- */

    /* a) Set viewport source = entire buffer */
    wp_viewport_set_source(viewport,
        wl_fixed_from_double(0.0), wl_fixed_from_double(0.0),
        wl_fixed_from_double(w->buf_w), wl_fixed_from_double(w->buf_h));

    /* b) Set viewport destination = logical size */
    wp_viewport_set_destination(viewport, w->logical_w, w->logical_h);

    /* c) Attach and damage the buffer */
    wl_surface_attach(surface, w->buffer, 0, 0);
    wl_surface_damage_buffer(surface, 0, 0, w->buf_w, w->buf_h);

    /* d) Commit: compositor now sees buffer + viewport atomically */
    wl_surface_commit(surface);
}
```

Key ordering rules:
- `wp_viewport_set_destination` must be committed **before or together with** the `wl_surface_attach` that delivers the correctly-sized buffer — both changes are applied at the same `wl_surface.commit`.
- `wl_surface_damage_buffer` (not `wl_surface_damage`) must be used when the buffer size changes, because `damage_buffer` coordinates are in buffer-pixel space, avoiding a logical→physical confusion.
- Do not call `wl_surface_set_buffer_scale` with values other than 1 when using `wp_viewport`.

### The set_source Optimisation

For a surface that fills its entire buffer with content (the common case), calling `wp_viewport_set_source` with the full buffer extents is technically redundant — an unset source rectangle implies the entire buffer. However, explicitly setting it makes the intent clear and avoids ambiguity if the buffer is later resized without changing the viewport object. Many toolkit implementations skip `set_source` and only call `set_destination`, which is correct.

---

## Compositor Implementation Internals

This section expands on the overview in [Compositor Implementation](#compositor-implementation) with source-level details and damage tracking specifics.

### wlroots: wlr_fractional_scale_v1 API

wlroots 0.17 (released 2023-08) added `wp_fractional_scale_v1` support. The public API is declared in
`include/wlr/types/wlr_fractional_scale_v1.h`:

```c
/* include/wlr/types/wlr_fractional_scale_v1.h (wlroots 0.17+) */

struct wlr_fractional_scale_manager_v1 {
    struct wl_global *global;
    struct wl_list resources; /* list of wl_resource */
    /* private: */
    struct wl_listener display_destroy;
};

/**
 * Create the wp_fractional_scale_manager_v1 global.
 * Call once during compositor initialisation.
 */
struct wlr_fractional_scale_manager_v1 *
wlr_fractional_scale_manager_v1_create(struct wl_display *display,
    uint32_t version);

/**
 * Notify a wlr_surface's fractional scale object of a new preferred scale.
 * scale is expressed as a double (e.g. 1.5); wlroots multiplies by 120
 * internally before sending preferred_scale to the client.
 */
void wlr_fractional_scale_v1_notify_scale(struct wlr_surface *surface,
    double scale);
```

A wlroots-based compositor calls `wlr_fractional_scale_v1_notify_scale` whenever a surface enters a new output, an output changes scale, or a surface moves such that a different output's scale is now dominant:

```c
/* Compositor code: handle output scale change or surface enter/leave */
static void output_scale_changed(struct wlr_output *output, void *data) {
    Server *server = data;
    struct wlr_scene_output *scene_output =
        wlr_scene_get_scene_output(server->scene, output);

    /* Iterate surfaces visible on this output */
    struct wlr_scene_node *node;
    wl_list_for_each(node, &server->scene->tree.children, link) {
        struct wlr_surface *surface = /* extract from scene node */;
        if (surface) {
            /* wlroots picks the dominant output's scale automatically
               when you call notify_scale; for surfaces spanning two
               outputs the compositor chooses by coverage area */
            wlr_fractional_scale_v1_notify_scale(surface, output->scale);
        }
    }
}
```

[Source: wlroots fractional scale MR, swaywm/wlroots](https://github.com/swaywm/wlroots/pull/2064)

### wlroots Scene Graph: Filter Mode Selection

When the wlroots scene graph rasterises a `wlr_scene_buffer` at a fractional scale, it must choose a texture filter. The render-pass API in `include/wlr/render/pass.h` defines:

```c
/* include/wlr/render/pass.h */
enum wlr_scale_filter_mode {
    WLR_SCALE_FILTER_BILINEAR, /* bilinear texture filtering (default) */
    WLR_SCALE_FILTER_NEAREST,  /* nearest texture filtering             */
};

struct wlr_render_texture_options {
    struct wlr_texture *texture;
    struct wlr_fbox src_box;         /* source rectangle in texture coordinates */
    struct wlr_box dst_box;          /* destination rectangle in buffer pixels   */
    float alpha;
    const pixman_region32_t *clip;
    enum wl_output_transform transform;
    enum wlr_scale_filter_mode filter_mode;  /* <-- the relevant field */
    enum wlr_render_blend_mode blend_mode;
    /* ... */
};
```

[Source: wlroots documentation for `wlr/render/pass.h`](https://kennylevinsen.pages.freedesktop.org/wlroots/wlr/render/pass.h.html)

The default is `WLR_SCALE_FILTER_BILINEAR`, producing smooth (but potentially slightly soft) edges. Per-buffer filter overrides let compositors choose nearest-neighbour for pixel-art content or integer-scaled games. Sway 1.10 ported opacity and filter-mode support to the scene graph, enabling the `interpolation_mode` sway config option to propagate per-buffer filter hints. [Source: Sway 1.10 release](https://github.com/swaywm/sway/releases/tag/1.10)

### wlroots Damage Tracking Under Fractional Scale

Damage tracking is the mechanism by which a compositor avoids repainting unchanged parts of the screen. Under integer scaling, damage rectangles in logical pixels expand to physical pixels by simple multiplication. Under fractional scaling, this expansion is not exact, and rounding errors can leave undamaged strips at surface edges. wlroots addresses this with `wlr_region_scale()` and `wlr_region_expand()`:

```c
/* Conceptual: convert surface-space damage to output-space damage */
pixman_region32_t output_damage;
pixman_region32_init(&output_damage);

/* 1. Scale the damage region by the output scale */
wlr_region_scale(&output_damage, &surface_damage, output->scale);

/* 2. Expand by the fractional overshoot: ceil(scale) - scale
      e.g. for 1.5×: ceil(1.5) - 1.5 = 0.5, expand by 1 pixel */
int expand = (int)ceil(output->scale) - (int)floor(output->scale);
if (expand > 0) {
    wlr_region_expand(&output_damage, &output_damage, expand);
}
```

The `wlr_region_scale` function rounds all region coordinates so the scaled region is at least as large as the original — ensuring no unrepainted pixels. [Source: wlroots damage tracking PR](https://github.com/swaywm/wlroots/pull/3117)

A related complication discovered by River compositor users: at 1.25× scale, the same window width can produce different physical pixel widths depending on horizontal position, because `round((offset + width) × scale) − round(offset × scale)` is position-dependent. The practical fix is to use `ceil(width × scale)` for buffer sizes (client-side) and always expand damage by at least one extra physical pixel (compositor-side). [Source: River issue #953](https://codeberg.org/river/river/issues/953)

### GNOME Mutter: MetaWaylandFractionalScale

Mutter's implementation lives in `src/wayland/meta-wayland-fractional-scale.{c,h}`. The architecture:

- `MetaWaylandFractionalScaleManager` wraps the `wp_fractional_scale_manager_v1` global.
- Each `MetaWaylandSurface` carries a `MetaWaylandFractionalScale *frac_scale` pointer, created lazily when a client calls `get_fractional_scale`.
- `MetaWindow` knows which `MetaLogicalMonitor` it is on. When the window moves to a monitor with a different scale, `meta_wayland_surface_notify_geometry_changed` triggers `meta_wayland_fractional_scale_update`, which calls `wp_fractional_scale_v1_send_preferred_scale` with the new value.

The `xwayland-native-scaling` experimental feature (GNOME 46+) extends this: when enabled, Mutter tells XWayland to render its virtual screen at `ceil(scale) ×` logical size, then presents the resulting XWayland surface buffer through `wp_fractional_scale_v1` like any Wayland surface:

```bash
# Enable both fractional scaling and XWayland native scaling in GNOME:
gsettings set org.gnome.mutter experimental-features \
  "['scale-monitor-framebuffer', 'xwayland-native-scaling']"
```

[Source: GNOME Discourse on fractional scaling](https://discourse.gnome.org/t/full-explanation-of-current-hidpi-fractional-and-integer-scaling-support-in-wayland/14225)

### KDE KWin: Integer vs. Fractional Mode by Content Type

KWin (Plasma 6) registers `FractionalScaleManagerV1Interface` from `src/wayland/fractional_scale_v1.cpp`. KWin's compositor render loop in `scene/workspacescene.cpp` selects scaling behaviour based on **content type**:

- **Native Wayland windows with `wp_fractional_scale_v1` support**: compositor composites the client-provided buffer at its declared viewport destination with bilinear filtering. No re-scaling by the compositor.
- **Non-supporting Wayland windows** (older apps): KWin scales the buffer from the integer-scale size with bilinear filtering, accepting some blur.
- **XWayland surfaces** (when KDE's "apply scaling themselves" mode is off): compositor upscales the 1× XWayland buffer. This is the blurry legacy path.
- **Video surfaces** (identified by `wp_content_type_v1` with `TYPE_VIDEO`): KWin may switch to integer modes or use nearest-neighbour to avoid chroma smearing.

KWin exposes the Lanczos upscaler (`KWIN_FORCE_LANCZOS=1`) for high-quality upscaling of non-native surfaces, at higher GPU cost. For a 1.5× scale, Lanczos produces visibly better results than bilinear for fine-detail UI elements rendered at 1× source, but cannot recover information that was never rendered. The correct solution remains native `wp_fractional_scale_v1` support in the client.

The upcoming `xx-fractional-scale-v2` protocol (in development as of 2025–2026) aims to improve on `v1` by allowing clients and compositors to communicate in **unscaled physical-pixel coordinates** from the start, eliminating the lossy logical→physical rounding that causes the rounding-error artefacts described above. [Source: xx-fractional-scale-v2 Phoronix coverage](https://phoronix.com/news/xx-fractional-scale-v2-MR-KWin)

---

## Font and Text Rendering Under Fractional Scale

### Why Fractional Scaling Historically Produced Blurry Text

On integer-scale displays (1× or 2×), every glyph pixel maps exactly to one or four physical pixels. Hinting algorithms move glyph stem widths and baseline positions to align with the physical pixel grid, making stems one or two pixels wide — crisp. At 1.5×, a 1-logical-pixel stem occupies 1.5 physical pixels; hinting can round it to 1 or 2 physical pixels but cannot land it perfectly on both edges simultaneously.

The pre-`wp_fractional_scale_v1` workaround made this worse: compositors asked clients to render at integer 2× scale, then downsampled to 1.5× on the compositor's GPU. The downsampling interpolated between the 2×-hinted pixel grid and the actual 1.5× grid, producing blurred stems. Since the glyph was hinted for the wrong pixel density, the blur was compounded by hinting artefacts.

With `wp_fractional_scale_v1`, the client renders at the exact physical pixel density (e.g. 180/120 = 1.5×). Glyph rasterisation at 1.5× physical pixels still faces the half-pixel challenge, but it does so without an additional compositor downscale. The text softness is inherent to 1.5× density, not artificially introduced. [Source: GTK blog](https://blog.gtk.org/2024/03/07/on-fractional-scales-fonts-and-hinting/)

### The Cairo / Pango / FreeType Pipeline

Text rendering in GTK-based applications flows through three layers:

1. **Pango** — layout engine: computes glyph sequences, line breaks, clusters. Emits glyph runs with positions.
2. **Cairo** — 2D drawing API: receives the glyph run from Pango via `pango_cairo_show_layout()`. Calls FreeType to rasterise each glyph at the device scale.
3. **FreeType** — rasteriser: loads the glyph outline from the font file, applies hinting, and rasterises to a bitmap at the requested pixel size.

The device scale is communicated to Cairo via:

```c
/* After receiving preferred_scale = 180 (1.5×): */
cairo_surface_t *target = /* your rendering target */;

/* Tell Cairo that 1 logical unit = 1.5 physical pixels */
cairo_surface_set_device_scale(target, 1.5, 1.5);

/* All subsequent Cairo operations (including text) will be
   rasterised at 1.5× without the caller needing to scale coordinates */
cairo_t *cr = cairo_create(target);
```

FreeType is invoked by Cairo with the physical pixel size derived from the device scale × the font point size × the output DPI. Cairo passes FreeType load flags derived from the Cairo font options:

```c
/* Cairo font options for fractional-scale HiDPI */
cairo_font_options_t *opts = cairo_font_options_create();
cairo_font_options_set_antialias(opts, CAIRO_ANTIALIAS_SUBPIXEL);
cairo_font_options_set_hint_style(opts, CAIRO_HINT_STYLE_SLIGHT);
cairo_font_options_set_subpixel_order(opts, CAIRO_SUBPIXEL_ORDER_RGB);
cairo_set_font_options(cr, opts);
```

`CAIRO_HINT_STYLE_SLIGHT` maps to `FT_LOAD_TARGET_LIGHT` in FreeType, which applies only vertical hinting (locking baselines and stem heights to full pixels) while allowing horizontal subpixel positioning (fractional x-advance). This is the mode recommended for fractional-scale HiDPI because:
- Vertical hinting makes horizontal text baselines crisp (a 1px baseline doesn't straddle physical pixels).
- Horizontal subpixel positioning allows glyph spacing to use the full 1/64th-pixel precision that FreeType tracks internally, giving the renderer maximum freedom to choose horizontal positions without grid rounding.

### fontconfig DPI Hints

fontconfig does not control rasterisation directly, but it adjusts the **requested DPI** fed to FreeType. If the system DPI is declared as 96 but the display is at 1.5× physical scale, the correct DPI to request is 144 (= 96 × 1.5). This ensures that a "12pt" font is rasterised at the physical size corresponding to 12 points at 144 DPI, not 96 DPI.

```xml
<!-- ~/.config/fontconfig/fonts.conf — HiDPI DPI hint -->
<fontconfig>
  <match target="font">
    <edit name="dpi" mode="assign"><double>144</double></edit>
    <edit name="hintstyle" mode="assign"><const>hintslight</const></edit>
    <edit name="antialias" mode="assign"><bool>true</bool></edit>
    <edit name="rgba" mode="assign"><const>rgb</const></edit>
    <edit name="lcdfilter" mode="assign"><const>lcddefault</const></edit>
  </match>
</fontconfig>
```

In practice, GTK4 and Qt6 compute DPI from the `wp_fractional_scale_v1` preferred scale and pass it directly to Pango's font map rather than relying on fontconfig's DPI entry. The fontconfig hint is most relevant for applications that call FreeType directly or use older font stacks. [Source: ArchWiki HiDPI](https://wiki.archlinux.org/title/HiDPI)

### GTK4's GTK 4.14 Improvement

Prior to GTK 4.14, GTK's NGL/GL renderer rasterised text glyphs at the CSS pixel size (logical coordinates) and then relied on device scale transformation to magnify them. At 1.5×, this meant glyphs were sampled at a 2/3-integer device pixel grid, producing visible inter-glyph bleed and softness.

GTK 4.14 (2024) introduced a new glyph rendering path in the NGL renderer where:

1. Glyph positions are computed in **device pixel space** (physical pixels), not logical pixel space.
2. Y-axis hinting aligns glyph baselines to device pixel boundaries.
3. X-axis positions use subpixel precision (fractional device pixels for horizontal spacing).

The net result is sharper text at 1.25×, 1.5×, and 1.75× on compositors that properly support `wp_fractional_scale_v1`. [Source: GTK Development Blog](https://blog.gtk.org/2024/03/07/on-fractional-scales-fonts-and-hinting/)

### Qt6 Font Rendering

Qt6's text pipeline uses HarfBuzz for shaping, FreeType for rasterisation, and Qt's own glyph cache. The `QFont::pixelSize()` is computed as `pointSize × physicalDPI / 72`, where `physicalDPI` takes the fractional scale into account:

```cpp
// Qt6 / QWaylandWindow: updating DPI after preferred_scale
void QWaylandWindow::updateScale(qreal scale) {
    m_scale = scale;
    // Triggers QHighDpiScaling, which adjusts devicePixelRatio
    // QFont objects deriving from this window get correct pixel sizes
    QWindowSystemInterface::handleWindowDevicePixelRatioChanged(
        window(), scale);
}
```

Qt's `QRasterPaintEngine` passes `FT_LOAD_TARGET_LCD` when LCD subpixel antialiasing is active and `FT_LOAD_TARGET_LIGHT` otherwise, matching Cairo's behaviour for fractional scales.

---

## XWayland Fractional Scaling — Deep Dive

This section expands the overview in [XWayland and X11 Comparison](#xwayland-and-x11-comparison) with implementation mechanics and known limitations.

### How XWayland Presents a Logical Screen to X11 Apps

XWayland is itself a Wayland client. It creates a `wl_surface` for the root window, which the compositor treats like any other window. X11 applications running inside XWayland see a virtual X screen whose size is determined by XWayland's view of the display geometry.

Without fractional scaling support, XWayland always runs its virtual screen at the **logical** resolution (e.g. 1920×1080 at 1.5× scale on a 2880×1620 panel), because XWayland cannot relay `wp_fractional_scale_v1` events to individual X11 clients. When the compositor rescales XWayland's surface to fill the physical pixels, all X11 windows appear blurry.

### The -dpi Flag and Screen DPI

XWayland accepts a `-dpi N` command-line flag that sets the `Xft.dpi` resource in the X server's initial resource database:

```bash
Xwayland -dpi 144  # 96 × 1.5 = 144 DPI for 1.5× effective scale
```

This does **not** cause XWayland to render at a higher resolution. It only advertises a DPI value to X11 clients via `DisplayWidthMM` / `DisplayHeightMM` (from which clients compute DPI) and the `Xft.dpi` X resource. Applications that query DPI (such as `pango_cairo_context_get_resolution()`) will scale their fonts appropriately, but the actual pixel canvas remains at logical size.

> **Note: needs verification** — Some older documentation references a `-hidpi` flag. As of XWayland 23.x and 24.x, no `-hidpi` flag appears in `Xwayland --help` output. The correct flag is `-dpi`. Verify with `man Xwayland` on your system.

### Server-Side Scaling Path for X11 Apps

When a compositor implements **server-side scaling** for XWayland, it presents XWayland with a virtual screen that is larger than the logical size — typically `ceil(scale)` times the logical size. For 1.5× scale on a 1920×1080 logical display:

- Without server-side scaling: XWayland screen = 1920×1080, compositor upscales 1.5× (blurry).
- With server-side scaling (GNOME `xwayland-native-scaling`): XWayland screen = 3840×2160 (= 1920×1080 × 2), X11 apps render at 2×, compositor downscales from 2× to 1.5× (one step of mild softening).

GNOME Mutter implements this by coordinating two coordinate spaces for X11 clients:
- **Stage coordinates** — the surface-local (logical) coordinate space that Wayland clients use.
- **Protocol coordinates** — the X11 coordinate space, set to `stage × ceil(scale)` so that X11 apps see a HiDPI screen.

```bash
# Enable server-side XWayland scaling in GNOME (GNOME 46+):
gsettings set org.gnome.mutter experimental-features \
  "['scale-monitor-framebuffer', 'xwayland-native-scaling']"
```

[Source: GNOME Mutter XWayland fractional scaling development](https://phoronix.com/news/GNOME-XWayland-Frac-Scaling)

KDE Plasma's approach differs: **System Settings → Display → Legacy Applications (XWayland)** offers two modes:
1. **Apply scaling themselves** — KDE tells XWayland the full physical resolution; DPI-aware X11 apps scale correctly; DPI-unaware apps appear tiny.
2. **Scale by compositor** — KDE upscales the 1× XWayland surface; all X11 apps look approximately right but blurry.

### Cursor Warping Under Fractional Scale

X11 applications frequently use `XWarpPointer` to teleport the cursor — used by games (camera lock), IDEs (keeping focus inside a widget), and older dialog boxes (snapping focus to a button). On Wayland, absolute pointer warping is not possible for security reasons; only relative motion is supported.

XWayland emulates pointer warping by:
1. Locking the pointer via `zwp_relative_pointer_manager_v1` and `zwp_pointer_constraints_v1`.
2. Reporting to the X11 application that the warp succeeded (lying about pointer coordinates).
3. Using relative motion events to move the pointer to the target position.

Under fractional scaling, warp emulation has an additional complication: the conversion between X11 pixel coordinates (which are at the virtual screen's scale) and Wayland surface coordinates (logical pixels) must account for the scale factor. A misaligned coordinate transform in this path causes clicks and input events to land at visually incorrect positions — the well-known "click-through" bug observed in early GNOME fractional scaling patches. [Source: XWayland pointer warp emulation](https://lists.x.org/archives/xorg-devel/2016-April/049351.html) [Source: XWayland pointer coordinate fix](https://www.webpronews.com/xwayland-patch-fixes-pointer-coordinate-glitches-in-x11-on-wayland/)

### Pointer Precision Limitations

Because XWayland's virtual screen is at an integer scale (either 1× logical, or 2× for server-side scaling), pointer coordinates reported to X11 clients are always integers in that integer-scale space. At 1.5× compositor scale, this means:

- Wayland delivers pointer coordinates at sub-logical-pixel precision (wl_fixed_t, 1/256 logical pixel).
- XWayland quantises these to the nearest integer in X11 coordinate space, losing sub-pixel precision.
- For 1.5× server-side scaling (screen at 2×), the quantisation error is ±0.5 × (1/2) = ±0.25 logical pixels — acceptable for most UI.
- For 1× server-side scaling (no upscale), the quantisation error is ±0.5 logical pixels — significant for precision tasks like vector drawing or competitive gaming.

Native Wayland clients do not have this limitation: `wl_pointer.motion` delivers `wl_fixed_t` coordinates, and the application can handle them with full sub-pixel precision.

---

## GPU Implications of Fractional Scaling

### How the GPU Compositor Samples Surface Buffers

In a GPU-composited Wayland session (wlroots with wlr_gles2_renderer or wlr_vulkan_renderer; Mutter with Cogl/GL; KWin with its OpenGL scene), each surface buffer is uploaded to the GPU as a texture. The compositor's render pass samples this texture and writes to the framebuffer at the physical pixel grid of the output.

For a surface at 1.5× scale:
- **Buffer** (texture): `ceil(logical_w × 1.5) × ceil(logical_h × 1.5)` physical pixels.
- **Viewport destination** (on-screen): `logical_w × logical_h` logical pixels = `logical_w × 1.5 × logical_h × 1.5` physical pixels.
- **Sampling ratio**: texture texels map 1:1 to output pixels — **no resampling occurs** for a correctly-authored `wp_fractional_scale_v1` surface.

This is the key insight: when a native client renders at the exact physical pixel size dictated by `ceil(logical × scale / 120)`, the compositor maps each texture texel to exactly one framebuffer pixel. No bilinear filtering is needed and no resolution is lost. [Source: fractional-scale-v1 protocol wayland.app](https://wayland.app/protocols/fractional-scale-v1)

### When Resampling Occurs

Resampling is unavoidable in three situations:

1. **Non-supporting clients** — the application renders at an integer scale (1× or 2×) and the compositor must rescale. At 1.5× output scale with a 1× client buffer, the compositor upscales by 1.5×, which requires bilinear or higher-quality filtering.

2. **Window movement to a different output** — while the client re-renders at the new scale, the compositor must continue displaying the old buffer, scaled to the new output's coordinate space. The transient duration is typically 1–3 frames.

3. **Surfaces spanning two outputs with different scales** — the same buffer must appear on two outputs simultaneously. The compositor renders it at the scale dominant on each output, applying a different sampling transform per output. This is unavoidable and always involves at least one resampled output.

### Filter Selection: Nearest vs. Bilinear

The wlroots scene graph (via `wlr_render_texture_options.filter_mode`) offers:

```c
/* wlroots render pass: per-surface filter selection */
struct wlr_render_texture_options opts = {
    .texture = surface_texture,
    .src_box = { 0, 0, buf_w, buf_h },
    .dst_box = { x, y, logical_w, logical_h },

    /* For pixel-art content or games that want integer scaling: */
    .filter_mode = WLR_SCALE_FILTER_NEAREST,

    /* For standard UI surfaces (default): */
    /* .filter_mode = WLR_SCALE_FILTER_BILINEAR, */
};
wlr_render_pass_add_texture(pass, &opts);
```

[Source: wlroots render pass header](https://kennylevinsen.pages.freedesktop.org/wlroots/wlr/render/pass.h.html)

Practical guidance:
- **Bilinear** (default): smoothly interpolates between source texels. Produces no pixel-border artefacts at fractional scales. Appropriate for vector-rendered UI, text, photos.
- **Nearest-neighbour**: maps each destination pixel to the nearest source texel with no blending. Preserves hard pixel edges — correct for pixel-art games, retro emulators. Produces staircase artefacts on non-integer scale factors.
- **Lanczos** (KWin option, not in wlroots): a higher-quality reconstruction filter that reduces ringing compared to simple bilinear. Higher computational cost; KWin applies it selectively for upscaling legacy surfaces.

Trilinear filtering (bilinear + mipmapping) is not commonly used for compositor surface scaling because surfaces are displayed at essentially one scale per frame; the mipmap's multi-level benefit applies only when a surface is displayed at a range of scales simultaneously (which occurs in 3D engines, not 2D compositors).

### Performance Impact in the Compositor Render Loop

The GPU cost of fractional scaling depends on which path is taken:

| Scenario | GPU cost relative to 1× |
|---|---|
| Native `wp_fractional_scale_v1` client, 1:1 texel mapping | 1.0× (zero additional cost) |
| 1× client upscaled to 1.5× with bilinear | ~1.5× (more output pixels to fill) |
| 2× client downscaled to 1.5× with bilinear | ~1.5× (more source pixels to read) |
| Overscale path: 2× render then compositor downscale to 1.5× | ~1.78× (2×2 / 1.5×1.5 pixel overhead) |

The overscale path (old GNOME pre-`wp_fractional_scale_v1`) was particularly expensive: requesting a client to render at 2× generates a buffer with 4× the pixel count of a 1× buffer, even though only 2.25× pixel count (1.5²) is needed on screen. Pixel shaders execute once per source pixel, so the wasted work is proportional to (2/1.5)² ≈ 1.78×. [Source: Shuhao Wu fractional scaling blog](https://shuhaowu.com/blog/2025/01-fractional-scaling.html)

For Vulkan-based compositors (Hyprland's wlroots fork, experimental KWin Vulkan path), surface buffers are sampled via `VkSampler` with `VK_FILTER_LINEAR` or `VK_FILTER_NEAREST`. The Vulkan path enables additional optimisations:
- **VK_FILTER_CUBIC_IMG** (where supported): higher-quality bicubic filter for legacy surface upscaling.
- **Explicit GPU timeline synchronisation** (`VkSemaphore` timelines): eliminates CPU-GPU round-trips for buffer ready/release, reducing frame latency on multi-GPU setups.

> **Note: needs verification** — Hyprland's Vulkan renderer uses `VK_FILTER_LINEAR` by default for fractional-scale compositing. The exact `VkSamplerCreateInfo` configuration should be confirmed against the Hyprland source at `src/render/OpenGL.cpp` or equivalent Vulkan source.

---

## Roadmap

### Near-term (6–12 months)

- **`xx-fractional-scale-v2` stabilisation** — The experimental `xx-fractional-scale-v2` protocol (revived from Xaver Hugl's 2022 `wp-fractional-scale-v2` proposal) aims to replace integer logical coordinate space with unscaled physical-pixel coordinates, eliminating subpixel rounding gaps between windows and panels at non-integer scales. KWin MR !9023 by Vlad Zahorodnii shipped in KDE Plasma 6.7 (June 2026); the protocol is expected to move from `xx-` staging to a stable `wp-` designation once multiple compositors adopt it. [Source: Phoronix — xx-fractional-scale-v2](https://www.phoronix.com/news/xx-fractional-scale-v2-MR-KWin) [Source: KWin MR !9023](https://invent.kde.org/plasma/kwin/-/merge_requests/9023)
- **GNOME Mutter adoption of `xx-fractional-scale-v2`** — Mutter currently uses an overscale path (render at 2×, compositor downscale to 1.5×) for GNOME on Wayland. Adoption of the new protocol coordinate model is expected to land once KDE's implementation proves the semantics. Note: needs verification of current Mutter MR status.
- **SDL3 and GLFW native support** — SDL3 already shipped `wp-fractional-scale-v1` support; GLFW PR #2215 implements `wp-fractional-scale-v2`/`xx-fractional-scale-v2` for correct per-window DPR delivery to game clients without requiring app-side hacks. [Source: GLFW PR #2215](https://github.com/glfw/glfw/pull/2215)
- **XWayland rootful fractional scaling** — XWayland's rootful mode landed HiDPI/fractional scaling support recently; improvements to the XWayland bridge for fractional scale events in windowed X11 apps running under Wayland compositors are ongoing. [Source: Phoronix XWayland rootful HiDPI](https://www.phoronix.com/forums/forum/linux-graphics-x-org-drivers/wayland-display-server/1451190-xwayland-rootful-lands-hidpi-fractional-scaling-support)

### Medium-term (1–3 years)

- **`xx-fractional-scale-v2` as the new baseline** — As `xx-fractional-scale-v2` proves stable across KWin, Mutter, and wlroots, toolkit maintainers (GTK5, Qt 7) are expected to deprecate `wl_output.scale` integer path and `wp_fractional_scale_v1` in favour of the new coordinate model. This would allow logical coordinate space to be fully replaced by physical-pixel geometry in the protocol. Note: needs verification of toolkit roadmap commitments.
- **Per-output colour and scaling metadata unification** — Ongoing work on the `xx-color-management-v4` and `wp-content-type-v1` protocols may be joined by a unified output-capability advertisement protocol that bundles fractional scale, HDR peak luminance, and colour primaries in one round-trip, reducing the number of protocol objects applications must negotiate at startup.
- **Cursor protocol improvements** — The `wp_cursor_shape_v1` protocol delivers named cursors but does not yet carry fractional-scale metadata; a follow-on protocol or extension is expected to allow compositors to request cursor bitmaps at the exact physical pixel size for a given surface scale, eliminating cursor blur at non-integer DPR. Note: needs verification of upstream proposal status.
- **Improved multi-output spanning** — Applications that span two outputs with different fractional scales (e.g., a 1.5× laptop panel and a 1.0× external monitor) currently receive only the scale of the output where the window is majority-resident. A protocol extension enabling per-output scale delivery for spanning surfaces is discussed but not yet formalised. Note: needs verification.

### Long-term

- **Physical-pixel-native Wayland coordinate model** — The long-term architectural direction suggested by `xx-fractional-scale-v2` is a shift away from logical-pixel protocol coordinates entirely: surfaces, subsurfaces, and input events could all be expressed in physical pixels with compositor-side transform metadata, eliminating the rounding errors inherent in fractional logical coordinates. This would be a breaking protocol change requiring a `wl_compositor` version bump. Note: speculative; no formal proposal exists as of mid-2026.
- **Compositor-side AI upscaling for legacy clients** — GPU vendors (NVIDIA DLSS, AMD FSR, Intel XeSS) have upscaling techniques usable in display pipelines. Future compositors may optionally apply learned super-resolution to 1× legacy client buffers before compositing to a HiDPI framebuffer, improving visual quality over bilinear upscaling without requiring client-side changes. Note: speculative direction, not yet proposed in any compositor project.
- **Unified HiDPI handling across Wayland and kernel DRM** — Kernel-level DRM scaling (via `DRM_CLIENT_CAP_ATOMIC` and plane scaling properties) currently operates independently of Wayland protocol scale factors. A future unified model might allow the compositor to offload fractional scale transforms to hardware plane scalers on supported GPUs, reducing GPU shader load for scaling legacy surfaces. Note: speculative; depends on hardware plane capabilities and driver maturity.

---

## Integrations

- **Ch20 (Wayland core)** — `wl_output.scale` protocol; surface-output enter/leave events that trigger scale re-negotiation
- **Ch21 (wlroots)** — `wlr_fractional_scale_v1_notify_scale` API; `wlr_output.scale` field; sway configuration
- **Ch22 (Compositors)** — GNOME Mutter experimental scale path; KWin Plasma 6 native fractional scaling; Hyprland `monitor` scale parameter
- **Ch39 (GTK/Qt)** — `GdkWaylandSurface` fractional scale handler; `QWaylandWindow::setPreferredScale`; device pixel ratio APIs
- **Ch105 (Font Rendering)** — HiDPI pipeline in FreeType/Pango; subpixel rendering at fractional DPR
- **Ch130 (Protocol Development)** — `wp_fractional_scale_v1` as an example of a staging protocol lifecycle: from experimental staging → stable; protocol XML grammar
