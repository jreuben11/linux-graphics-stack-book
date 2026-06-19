# Chapter 138: Wayland Fractional Scaling and HiDPI

**Target audiences**: Wayland compositor developers implementing fractional scaling, application developers targeting HiDPI displays, toolkit maintainers integrating `wp_fractional_scale_v1`, and systems engineers managing mixed-DPI multi-monitor setups.

---

## Table of Contents

1. [Introduction](#introduction)
2. [The HiDPI Problem](#the-hidpi-problem)
3. [wl_output.scale: Integer Scaling Baseline](#wl_outputscale-integer-scaling-baseline)
4. [wp_fractional_scale_v1: The Protocol](#wp_fractional_scale_v1-the-protocol)
5. [Compositor Implementation](#compositor-implementation)
6. [Toolkit Integration](#toolkit-integration)
7. [Mixed-DPI Multi-Monitor Setups](#mixed-dpi-multi-monitor-setups)
8. [XWayland and X11 Comparison](#xwayland-and-x11-comparison)
9. [Cursor, Icons, and Subpixel Rendering](#cursor-icons-and-subpixel-rendering)
10. [Integrations](#integrations)

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

## Integrations

- **Ch20 (Wayland core)** — `wl_output.scale` protocol; surface-output enter/leave events that trigger scale re-negotiation
- **Ch21 (wlroots)** — `wlr_fractional_scale_v1_notify_scale` API; `wlr_output.scale` field; sway configuration
- **Ch22 (Compositors)** — GNOME Mutter experimental scale path; KWin Plasma 6 native fractional scaling; Hyprland `monitor` scale parameter
- **Ch39 (GTK/Qt)** — `GdkWaylandSurface` fractional scale handler; `QWaylandWindow::setPreferredScale`; device pixel ratio APIs
- **Ch105 (Font Rendering)** — HiDPI pipeline in FreeType/Pango; subpixel rendering at fractional DPR
- **Ch130 (Protocol Development)** — `wp_fractional_scale_v1` as an example of a staging protocol lifecycle: from experimental staging → stable; protocol XML grammar
