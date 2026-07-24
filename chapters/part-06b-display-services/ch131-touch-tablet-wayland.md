# Chapter 131: Touch, Stylus, and Tablet Input on Wayland

This chapter targets:

- **Wayland compositor developers** — building tablet and touchscreen protocol support
- **creative application developers** — working with GIMP, Krita, and Inkscape on Wayland
- **embedded touchscreen engineers** — integrating capacitive or resistive touchscreens into Wayland-based products
- **input stack engineers** — diagnosing or extending the pipeline from kernel driver to application event

The chapter traces the full input chain — hardware → kernel HID driver → evdev → libinput → Wayland compositor → Wayland protocol → application — for both multi-touch surfaces and professional drawing tablets with stylus, eraser, and physical pad controls.

---

## Table of Contents

1. [Introduction — The Touch and Tablet Input Chain](#1-introduction--the-touch-and-tablet-input-chain)
   - [1.1 What is evdev?](#11-what-is-evdev)
   - [1.2 What is libinput?](#12-what-is-libinput)
   - [1.3 What is libwacom?](#13-what-is-libwacom)
   - [1.4 What are the Wayland Tablet and Touch Protocols?](#14-what-are-the-wayland-tablet-and-touch-protocols)
2. [Kernel-Level Touch and Tablet Input](#2-kernel-level-touch-and-tablet-input)
   - [HID Subsystem and Tablet Drivers](#21-hid-subsystem-and-tablet-drivers)
   - [evdev Event Codes for Tablets](#22-evdev-event-codes-for-tablets)
   - [Multi-Touch Protocol A and B](#23-multi-touch-protocol-a-and-b)
   - [Inspecting Raw Events](#24-inspecting-raw-events)
3. [libinput Tablet and Touch Handling](#3-libinput-tablet-and-touch-handling)
   - [Touch Event API](#31-touch-event-api)
   - [Tablet Tool Events and Axes](#32-tablet-tool-events-and-axes)
   - [Tablet Pad Events](#33-tablet-pad-events)
   - [Touch Arbitration and Pressure Offset Detection](#34-touch-arbitration-and-pressure-offset-detection)
4. [libwacom: Device Database and Capabilities](#4-libwacom-device-database-and-capabilities)
   - [Device Database Structure](#41-device-database-structure)
   - [C API Overview](#42-c-api-overview)
   - [Stylus Database](#43-stylus-database)
   - [How Compositors Use libwacom](#44-how-compositors-use-libwacom)
5. [wl_touch: Core Wayland Touch Protocol](#5-wl_touch-core-wayland-touch-protocol)
   - [Obtaining a wl_touch Object](#51-obtaining-a-wl_touch-object)
   - [Touch Events](#52-touch-events)
   - [Frame Batching and Cancellation](#53-frame-batching-and-cancellation)
   - [Shape and Orientation Extensions (v6)](#54-shape-and-orientation-extensions-v6)
6. [zwp_tablet_manager_v2: Professional Stylus Protocol](#6-zwp_tablet_manager_v2-professional-stylus-protocol)
   - [Protocol Architecture](#61-protocol-architecture)
   - [zwp_tablet_tool_v2 Events and Enumerations](#62-zwp_tablet_tool_v2-events-and-enumerations)
   - [zwp_tablet_pad_v2: Rings, Strips, and Dials](#63-zwp_tablet_pad_v2-rings-strips-and-dials)
   - [Mode Switching](#64-mode-switching)
7. [Compositor Implementation: wlroots Tablet Support](#7-compositor-implementation-wlroots-tablet-support)
   - [Tablet Manager and Device Registration](#71-tablet-manager-and-device-registration)
   - [Delivering Axis and Proximity Events](#72-delivering-axis-and-proximity-events)
   - [Cursor Handling and Coordinate Mapping](#73-cursor-handling-and-coordinate-mapping)
8. [Creative Application Integration](#8-creative-application-integration)
   - [GTK4 and GDK Wayland Device Model](#81-gtk4-and-gdk-wayland-device-model)
   - [GIMP Tablet Handling](#82-gimp-tablet-handling)
   - [Krita and the Qt Wayland Tablet Plugin](#83-krita-and-the-qt-wayland-tablet-plugin)
   - [Inkscape on GTK4 Wayland](#84-inkscape-on-gtk4-wayland)
9. [Calibration, Mapping, and Multi-Monitor Considerations](#9-calibration-mapping-and-multi-monitor-considerations)
10. [Integrations](#10-integrations)

---

## 1. Introduction — The Touch and Tablet Input Chain

Touch screens, professional drawing tablets (Wacom, Huion, XP-Pen), and combined pen displays are first-class input devices on modern Linux. They share the lower layers of the input stack with mice and keyboards but diverge above libinput, where dedicated Wayland extension protocols expose their richer analogue axes: pressure (0–65535), tilt (±degrees), barrel rotation, distance from the surface, and specialised physical controls such as rings and strips on tablet pads.

The full input chain for a stylus event is:

```
Drawing tablet (USB/Bluetooth/I2C)
    → Linux HID driver (wacom.ko / hid-uclogic.ko)
    → Kernel input subsystem (drivers/input/)
    → evdev node (/dev/input/eventN)
    → libinput (compositor calls libinput_dispatch)
    → Wayland compositor (wlroots / Mutter / KWin)
    → zwp_tablet_tool_v2 Wayland protocol
    → Application (GTK4 / Qt 6 / SDL2)
```

For multi-touch surfaces the path is identical but the Wayland layer uses `wl_touch` rather than the tablet extension.

This chapter covers each layer in depth: the Linux kernel HID and evdev models, libinput's tablet and touch event API, libwacom for device capability querying, the `wl_touch` core protocol for touch surfaces, the `zwp_tablet_manager_v2` extension for professional stylus input, wlroots compositor implementation, and how creative applications (GIMP, Krita, Inkscape) consume tablet events on Wayland.

### 1.1 What is evdev?

evdev (event device) is the kernel interface through which userspace reads raw input events from hardware. Every keyboard, mouse, touchscreen, stylus, and drawing tablet presents itself as a character device node under `/dev/input/eventN`. Reads from this node return a stream of `struct input_event` records, each carrying a timestamp, an event type (`EV_KEY`, `EV_ABS`, `EV_SYN`, and others), a code identifying the specific axis or button, and an integer value. Pen tablets use absolute axes (`EV_ABS`) for position and pressure; touchscreens use a dedicated set of `ABS_MT_*` multi-touch axes to report simultaneous contacts. The event codes and their semantics are defined in `include/uapi/linux/input-event-codes.h` and documented at the [kernel event-codes page](https://docs.kernel.org/input/event-codes.html). Because evdev exposes raw, hardware-specific values — axis ranges that differ between device models, pressure scales that vary from 1024 to 8192 levels — userspace libraries, primarily libinput, normalise these into device-independent units and apply per-device calibration before compositors and applications see them. On a Wayland system, only libinput opens `/dev/input/eventN` directly; applications never touch the evdev node.

### 1.2 What is libinput?

libinput is the userspace input handling library used by virtually all Wayland compositors on Linux. It sits between the kernel evdev layer and the Wayland compositor, reading raw `struct input_event` records via `libevdev`, normalising axis values, and emitting high-level semantic events that compositors consume through `libinput_dispatch()`. For drawing tablets, libinput recognises three device capability flags — `LIBINPUT_DEVICE_CAP_TABLET_TOOL`, `LIBINPUT_DEVICE_CAP_TABLET_PAD`, and `LIBINPUT_DEVICE_CAP_TOUCH` — and routes events through dedicated handler paths that produce pressure values normalised to 0.0–1.0, tilt angles in degrees, and tool-type enumerations regardless of the underlying hardware model. libinput also implements touch arbitration, which suppresses accidental palm contacts when a stylus is in proximity, and handles pressure-offset calibration for pen-on-display tablets. The library is developed on freedesktop.org GitLab at [https://gitlab.freedesktop.org/libinput/libinput](https://gitlab.freedesktop.org/libinput/libinput) and exposes a stable C API declared in `<libinput.h>`. The X.Org `xf86-input-libinput` driver routes the same library output into the X input stack, making libinput the single normalisation layer for both X11 and Wayland sessions.

### 1.3 What is libwacom?

libwacom is a device database and query library that stores the hardware characteristics of Wacom, Huion, XP-Pen, and other drawing tablets in structured `.tablet` text files. Each database entry records vendor and product IDs, the physical dimensions of the active surface, the number and layout of express keys and ring controls on the tablet pad, the list of supported stylus serial numbers, and screen-mapping metadata for integrated pen displays. Compositors and configuration tools query this database at device hotplug time using the C API in `<libwacom/libwacom.h>` to determine how many pad button modes are valid, how rings and strips map to rotary or linear controls, and whether the device requires absolute-to-screen coordinate mapping. Without libwacom, compositors would need device-specific code for each tablet model; the database approach allows generic protocol implementations to adapt to hundreds of distinct devices. libwacom is maintained at [https://github.com/linuxwacom/libwacom](https://github.com/linuxwacom/libwacom) and is a build dependency of both libinput and all compositors that implement `zwp_tablet_manager_v2`.

### 1.4 What are the Wayland Tablet and Touch Protocols?

Wayland separates touch and stylus input into two distinct protocol namespaces. Touch surfaces use `wl_touch`, the core Wayland protocol object obtained from a `wl_seat`; it delivers per-contact down, up, motion, and cancel events identified by integer slot IDs, with optional shape and orientation extensions added in Wayland core version 6. Professional drawing tablets are handled by the `zwp_tablet_manager_v2` extension, defined in `wayland-protocols` under `unstable/tablet/tablet-unstable-v2.xml`. This extension exposes tablets as `zwp_tablet_v2` objects, individual tools (pen, eraser, airbrush, mouse, lens cursor) as `zwp_tablet_tool_v2` objects with normalised axes for pressure, tilt, rotation, distance, and slider, and the physical pad as `zwp_tablet_pad_v2` with rings, strips, and button-mode groups. Compositors implement these protocols by forwarding libinput tablet and touch events to the Wayland objects bound by each client. Applications reach this data through toolkit abstractions — GTK4 `GdkDeviceTool`, Qt `QTabletEvent` — rather than calling the Wayland protocol objects directly.

---

## 2. Kernel-Level Touch and Tablet Input

### 2.1 HID Subsystem and Tablet Drivers

Linux represents human interface devices through the **HID subsystem** in `drivers/hid/`. HID descriptors — XML-like binary structures embedded in the device firmware — declare the axes, buttons, and feature reports a device exposes; the HID core parses them and routes data to appropriate handler modules.

**Wacom tablets** are served by the `wacom.ko` kernel module, built from two source files:

- `drivers/hid/wacom_wac.c` — protocol decoding and event generation: converts Wacom-proprietary HID reports into standard `input_event` records
- `drivers/hid/wacom_sys.c` — device lifecycle, sysfs attribute groups, and hotplug management

The corresponding header is `drivers/hid/wacom.h`. The module is selected by `CONFIG_HID_WACOM` and handles USB, Bluetooth, and I2C-connected Wacom devices with a single unified driver since kernel 3.17. Earlier kernels had a legacy Bluetooth-only `hid-wacom.ko` module (`drivers/hid/hid-wacom.c`), which is now obsolete. The out-of-tree `linuxwacom/input-wacom` project backports current Wacom support to older kernels. [Source: Linux kernel Makefile](https://github.com/torvalds/linux/blob/master/drivers/hid/Makefile)

**Huion, XP-Pen, and UC-Logic tablets** are handled by `hid-uclogic.ko`, built from three files:

```
hid-uclogic-core.o    — main event loop and module entry
hid-uclogic-rdesc.o   — HID report descriptor fixups
hid-uclogic-params.o  — per-device parameter tables
```

The Huion-specific `hid-huion.c` that existed in older kernels has been folded into `hid-uclogic`. Selecting `CONFIG_HID_UCLOGIC` builds in support for the full family. [Source: Linux kernel HID Makefile](https://github.com/torvalds/linux/blob/master/drivers/hid/Makefile)

**Generic touch screens** on embedded devices typically use `hid-multitouch.ko` (`drivers/hid/hid-multitouch.c`), selected by `CONFIG_HID_MULTITOUCH`. For I2C-connected panels the kernel uses `i2c-hid` (`drivers/hid/i2c-hid/`) to bridge the I2C transport into the standard HID stack.

udev classifies input devices by querying their evdev capability bitmasks and setting properties such as:

- `ID_INPUT_TOUCHSCREEN=1` — capacitive or resistive touch panel
- `ID_INPUT_TABLET=1` — drawing tablet with stylus
- `ID_INPUT_TABLET_PAD=1` — auxiliary physical pad on a drawing tablet

These properties are used by libinput to route events to the correct handler path. To inspect them:

```bash
udevadm info /dev/input/event5
# Look for: ID_INPUT_TABLET, ID_INPUT_TABLET_PAD, DEVNAME, ID_VENDOR, ID_MODEL
```

### 2.2 evdev Event Codes for Tablets

All tablet input reaches userspace as a stream of `struct input_event` records through `/dev/input/eventN`. The kernel input subsystem defines event types and codes in `include/uapi/linux/input-event-codes.h`.

**Proximity and tool identification:**

| Event | Meaning |
|-------|---------|
| `BTN_TOOL_PEN` | Stylus entered or left proximity; value 1 = in range, 0 = out |
| `BTN_TOOL_RUBBER` | Eraser end of a dual-ended stylus |
| `BTN_TOOL_BRUSH` | Brush tool reported by some devices |
| `BTN_TOOL_PENCIL` | Pencil tool |
| `BTN_TOOL_AIRBRUSH` | Airbrush tool |
| `BTN_TOOL_MOUSE` | Mouse-like tool (e.g. Wacom Art Pen in mouse mode) |
| `BTN_TOOL_LENS` | Lens cursor |
| `BTN_TOUCH` | Tip in physical contact with surface (value 1 = touching) |
| `BTN_STYLUS` | First side button on the stylus barrel |
| `BTN_STYLUS2` | Second side button on the stylus barrel |

**Analogue axes for the stylus:**

| Event | Range | Meaning |
|-------|-------|---------|
| `ABS_X` | 0–max | Absolute X position on the tablet surface |
| `ABS_Y` | 0–max | Absolute Y position |
| `ABS_PRESSURE` | 0–max | Tip pressure (varies per device: 1024, 2048, 8192 levels) |
| `ABS_DISTANCE` | 0–max | Distance from surface when hovering |
| `ABS_TILT_X` | −60 to +60° | Tilt angle along X axis, positive = pen top tilted right |
| `ABS_TILT_Y` | −60 to +60° | Tilt angle along Y axis, positive = pen top tilted toward user |
| `ABS_WHEEL` | 0–1023 | Barrel rotation (Art Pen, airbrush thumbwheel) |
| `ABS_THROTTLE` | −1023 to +1023 | Airbrush finger wheel |

The axis capability ranges are exposed to userspace through `libevdev_get_abs_info(dev, ABS_PRESSURE, &info)`, which returns a `struct input_absinfo` containing `minimum`, `maximum`, `resolution`, and `fuzz`. [Source: Linux kernel event codes documentation](https://docs.kernel.org/input/event-codes.html)

A well-behaved tablet driver resets all tool-specific axes to zero when `BTN_TOOL_*` transitions to 0 (proximity out), ensuring applications do not see stale values when a different tool enters proximity.

### 2.3 Multi-Touch Protocol A and B

Touch screens use a distinct set of `ABS_MT_*` axes to report multiple simultaneous contact points. The kernel defines two protocol variants: [Source: Linux kernel multi-touch protocol documentation](https://docs.kernel.org/input/multi-touch-protocol.html)

**Type A (deprecated, stateless):** Drivers separate contact packets with `input_mt_sync()`, which generates `SYN_MT_REPORT` events. All contacts are re-reported in full each frame with no tracking across frames. No tracking IDs are assigned. This protocol is now obsolete; most modern drivers use Type B.

**Type B (current, slot-based):** Drivers call `input_mt_slot(dev, slot_id)` before each contact's data, emitting `ABS_MT_SLOT` to select the active slot. A non-negative `ABS_MT_TRACKING_ID` marks a live contact; `−1` removes it. Only changed values need to be emitted within a slot. The frame ends with `SYN_REPORT`.

Key `ABS_MT_*` event codes for touch:

| Code | Meaning |
|------|---------|
| `ABS_MT_SLOT` | Selects the active contact slot (0 to N−1) |
| `ABS_MT_TRACKING_ID` | Per-contact lifetime ID; −1 = slot empty |
| `ABS_MT_POSITION_X` / `_Y` | Contact centre coordinates |
| `ABS_MT_TOUCH_MAJOR` / `_MINOR` | Semi-axes of the contact ellipse (mm) |
| `ABS_MT_PRESSURE` | Contact force |
| `ABS_MT_DISTANCE` | Hover distance above surface |
| `ABS_MT_TOOL_TYPE` | Tool type (MT_TOOL_FINGER, MT_TOOL_PEN, MT_TOOL_PALM) |
| `ABS_MT_ORIENTATION` | Ellipse rotation angle |

Pen input on a tablet that also supports touch (such as Wacom Cintiq) is reported via the single-touch `ABS_X`/`ABS_PRESSURE` axes on a separate evdev node, not via `ABS_MT_*`.

### 2.4 Inspecting Raw Events

Two tools are essential for diagnosing kernel-level tablet behaviour:

```bash
# Stream all raw events from a tablet or touchscreen
evtest /dev/input/event5

# List device capabilities and current axis ranges
evtest --query /dev/input/event5 ABS ABS_PRESSURE

# Inspect udev metadata
udevadm info --attribute-walk /dev/input/event5 | grep -E 'ID_INPUT|NAME'

# Debug libinput events (higher-level)
libinput debug-events --device /dev/input/event5
```

`evtest` output for a stylus entering proximity looks like:

```
Event: time 1719843200.123, type 4 (EV_MSC), code 40 (MSC_SERIAL), value 100000000
Event: time 1719843200.123, type 1 (EV_KEY), code 320 (BTN_TOOL_PEN), value 1
Event: time 1719843200.123, type 3 (EV_ABS), code 0 (ABS_X), value 8192
Event: time 1719843200.123, type 3 (EV_ABS), code 1 (ABS_Y), value 6144
Event: time 1719843200.123, type 3 (EV_ABS), code 25 (ABS_DISTANCE), value 30
Event: time 1719843200.123, type 0 (EV_SYN), code 0 (SYN_REPORT), value 0
```

---

## 3. libinput Tablet and Touch Handling

libinput is the userspace input library used by virtually all Wayland compositors. [Source: libinput GitLab](https://gitlab.freedesktop.org/libinput/libinput) It reads raw evdev events, applies device-specific calibration and quirks, and emits normalised semantic events that compositors consume via `libinput_dispatch()`. Device capabilities for tablets are declared through:

- `LIBINPUT_DEVICE_CAP_TABLET_TOOL` — the device reports stylus/eraser events
- `LIBINPUT_DEVICE_CAP_TABLET_PAD` — the device reports pad buttons, rings, strips
- `LIBINPUT_DEVICE_CAP_TOUCH` — the device reports multi-touch contacts

### 3.1 Touch Event API

Touch events are delivered as one of five event types. The central data accessor is `libinput_event_touch *t = libinput_event_get_touch_event(event)`.

| Event type | Meaning |
|-----------|---------|
| `LIBINPUT_EVENT_TOUCH_DOWN` | A contact appeared (finger touched) |
| `LIBINPUT_EVENT_TOUCH_MOTION` | A live contact moved |
| `LIBINPUT_EVENT_TOUCH_UP` | A contact was lifted |
| `LIBINPUT_EVENT_TOUCH_CANCEL` | Compositor cancelled the touch sequence |
| `LIBINPUT_EVENT_TOUCH_FRAME` | End of a synchronised event batch |

Key accessor functions:

```c
/* Slot: 0-indexed per-device contact ID. Returns -1 for single-touch devices. */
int32_t slot = libinput_event_touch_get_slot(t);

/* Seat-wide unique ID across all active contacts on all devices in the seat */
int32_t seat_slot = libinput_event_touch_get_seat_slot(t);

/* Device-space coordinates in mm from the top-left corner of the touch surface */
double x_mm = libinput_event_touch_get_x(t);
double y_mm = libinput_event_touch_get_y(t);

/* Screen-space coordinates: map mm to output pixel space */
double x_px = libinput_event_touch_get_x_transformed(t, output_width_px);
double y_px = libinput_event_touch_get_y_transformed(t, output_height_px);
```

`libinput_event_touch_get_x_transformed()` performs an affine mapping from device physical dimensions to output pixel dimensions, accounting for any calibration matrix applied with `LIBINPUT_CALIBRATION_MATRIX`. [Source: libinput touch event API](https://wayland.freedesktop.org/libinput/doc/latest/api/group__event__touch.html)

The `LIBINPUT_EVENT_TOUCH_FRAME` event carries no coordinates; it signals that all contacts in the current hardware scan interval have been reported and the compositor should now process the accumulated set as an atomic unit.

### 3.2 Tablet Tool Events and Axes

Tablet tool events arrive as one of four types, with the data accessor `libinput_event_tablet_tool *tt = libinput_event_get_tablet_tool_event(event)`:

| Event type | Meaning |
|-----------|---------|
| `LIBINPUT_EVENT_TABLET_TOOL_PROXIMITY` | Tool entered or left sensor range |
| `LIBINPUT_EVENT_TABLET_TOOL_TIP` | Tip made or broke physical contact |
| `LIBINPUT_EVENT_TABLET_TOOL_AXIS` | One or more analogue axes changed value |
| `LIBINPUT_EVENT_TABLET_TOOL_BUTTON` | A stylus button was pressed or released |

**Tool type query:**

```c
struct libinput_tablet_tool *tool =
    libinput_event_tablet_tool_get_tool(tt);

enum libinput_tablet_tool_type type =
    libinput_tablet_tool_get_type(tool);
/* Values: LIBINPUT_TABLET_TOOL_TYPE_PEN, _ERASER, _BRUSH,
           _PENCIL, _AIRBRUSH, _MOUSE, _LENS */
```

**Capability queries (call before reading axes):**

```c
/* Returns non-zero if this specific tool supports the axis */
libinput_tablet_tool_has_pressure(tool);
libinput_tablet_tool_has_tilt(tool);
libinput_tablet_tool_has_rotation(tool);
libinput_tablet_tool_has_distance(tool);
libinput_tablet_tool_has_slider(tool);
libinput_tablet_tool_has_wheel(tool);
```

**Axis value accessors (valid on AXIS and TIP events):**

```c
/* Position in device-space mm; use get_x_transformed() for screen pixels */
double x = libinput_event_tablet_tool_get_x(tt);
double y = libinput_event_tablet_tool_get_y(tt);

/* Deltas since last event, resolution-normalised so 45° diagonal → (N, N) */
double dx = libinput_event_tablet_tool_get_dx(tt);
double dy = libinput_event_tablet_tool_get_dy(tt);

/* Pressure: 0.0 (no contact) to 1.0 (maximum force). libinput applies a
   device-specific tip threshold so that light resting contact does not
   falsely trigger a tip event. On worn tools the minimum physical pressure
   may produce a nonzero value; libinput detects and compensates for this
   "pressure offset" automatically. */
double pressure = libinput_event_tablet_tool_get_pressure(tt);

/* Tilt: signed degrees. +X = top tilted toward right edge of tablet.
   +Y = top tilted toward the far edge (away from user). */
double tilt_x = libinput_event_tablet_tool_get_tilt_x(tt);
double tilt_y = libinput_event_tablet_tool_get_tilt_y(tt);

/* Rotation: pen barrel rotation in degrees (0–360), e.g. Art Pen */
double rotation = libinput_event_tablet_tool_get_rotation(tt);

/* Distance: normalised 0.0–1.0 hover distance above surface */
double distance = libinput_event_tablet_tool_get_distance(tt);

/* Airbrush slider position: 0.0–1.0 */
double slider = libinput_event_tablet_tool_get_slider_position(tt);
```

**Proximity handling:**

```c
if (type == LIBINPUT_EVENT_TABLET_TOOL_PROXIMITY) {
    enum libinput_tablet_tool_proximity_state state =
        libinput_event_tablet_tool_get_proximity_state(tt);
    if (state == LIBINPUT_TABLET_TOOL_PROXIMITY_STATE_IN) {
        /* Tool entered: allocate per-tool state */
        libinput_tablet_tool_ref(tool); /* hold a reference */
    } else {
        libinput_tablet_tool_unref(tool);
    }
}
```

Unique tools (those where `libinput_tablet_tool_is_unique()` returns true — typically Wacom Intuos-series) can be identified by a 64-bit hardware serial number via `libinput_tablet_tool_get_serial()`. This allows compositors and applications to persist per-tool settings (pressure curves, button bindings) across proximity cycles and across different tablets.

**Tip vs. application input:**

The documentation strongly advises that most applications should ignore pressure data until they receive a `LIBINPUT_EVENT_TABLET_TOOL_TIP` event with tip state `LIBINPUT_TABLET_TOOL_TIP_DOWN`. Pressure during hovering is unreliable and device-specific. [Source: libinput tablet support documentation](https://wayland.freedesktop.org/libinput/doc/1.27.1/tablet-support.html)

**Pressure range configuration (libinput ≥ 1.26):**

```c
/* Allow the user or compositor to restrict the usable pressure range,
   e.g. to compensate for a worn-down nib that never reaches full pressure */
libinput_tablet_tool_config_pressure_range_set(tool, 0.05, 0.95);
```

### 3.3 Tablet Pad Events

Tablet pads — physical buttons, touch rings, and touch strips on devices like the Wacom Intuos Pro — report through `LIBINPUT_DEVICE_CAP_TABLET_PAD`.

| Event type | Meaning |
|-----------|---------|
| `LIBINPUT_EVENT_TABLET_PAD_BUTTON` | Physical button pressed or released |
| `LIBINPUT_EVENT_TABLET_PAD_RING` | Touch ring position changed |
| `LIBINPUT_EVENT_TABLET_PAD_STRIP` | Touch strip position changed |

```c
struct libinput_event_tablet_pad *p =
    libinput_event_get_tablet_pad_event(event);

/* Button: 0-indexed, sequential across all buttons on the pad */
uint32_t button_num = libinput_event_tablet_pad_get_button_number(p);
enum libinput_button_state state =
    libinput_event_tablet_pad_get_button_state(p);

/* Ring position: 0.0–360.0 degrees, or -1.0 if contact ended */
double ring_pos = libinput_event_tablet_pad_get_ring_position(p);

/* Strip position: 0.0–1.0 from top to bottom, or -1.0 if contact ended */
double strip_pos = libinput_event_tablet_pad_get_strip_position(p);

/* Current mode of the pad group that contains this button/ring/strip */
uint32_t mode = libinput_event_tablet_pad_get_mode(p);
```

Pad events are delivered independently of tool proximity; a user can press an express key while the pen is nowhere near the tablet.

### 3.4 Touch Arbitration and Pressure Offset Detection

libinput implements **touch arbitration** on combination tablet+touch devices: touch events on the tablet surface are suppressed while a stylus is in proximity. This prevents accidental palm or finger contact from interfering with precision drawing. The compositor does not need to implement this logic; libinput handles it transparently by monitoring `BTN_TOOL_*` proximity state and muting the associated touch device.

**Pressure offset detection** handles worn or damaged stylus nibs. When a tool reports non-zero pressure while hovering (i.e., before `BTN_TOUCH` fires), libinput detects this as a pressure offset, records it, and subtracts it from all subsequent pressure readings, remapping the remaining range to 0.0–1.0. This is transparent to both the compositor and the application.

---

## 4. libwacom: Device Database and Capabilities

libwacom ([https://github.com/linuxwacom/libwacom](https://github.com/linuxwacom/libwacom)) is a tablet description library maintained by the Linux Wacom Project. It provides a machine-readable database of tablet models, their physical dimensions, button layouts, stylus capability lists, and integration flags (e.g., whether a tablet has a built-in display). libinput, GNOME Settings, and wlroots all link against it.

### 4.1 Device Database Structure

The database lives in `/usr/share/libwacom/`. Each supported tablet has a `.tablet` file:

```ini
# /usr/share/libwacom/wacom-intuos-pro-m.tablet
[Device]
Name=Wacom Intuos Pro M
DeviceMatch=usb:056a:0357
Class=Intuos5
Width=8.7
Height=5.8
IntegrationFlags=
Layout=intuos-pro-m.svg

[Features]
Stylus=true
Touch=true
Ring=true
Ring2=false
NumStrips=0
NumButtons=8
```

Stylus properties are stored in separate `.stylus` files under the same directory. Since libwacom 2.14, the API supports styli from vendors other than Wacom.

### 4.2 C API Overview

```c
#include <libwacom/libwacom.h>

/* Load the tablet database */
WacomDeviceDatabase *db = libwacom_database_new();

/* Look up a device by its evdev node */
WacomError *err = libwacom_error_new();
WacomDevice *device = libwacom_new_from_path(
    db, "/dev/input/event5", WFALLBACK_NONE, err);

if (!device) {
    fprintf(stderr, "Unknown tablet: %s\n",
            libwacom_error_get_message(err));
}

/* Query basic properties */
const char *name     = libwacom_get_name(device);
int         width_mm = libwacom_get_width(device);   /* physical width in mm */
int         height_mm= libwacom_get_height(device);
int         nbuttons = libwacom_get_num_buttons(device);
bool        has_sty  = libwacom_has_stylus(device);
bool        has_touch= libwacom_has_touch(device);
bool        has_ring = libwacom_has_ring(device);

/* Integration flags: is this a pen display (screen + tablet combined)? */
WacomIntegrationFlags flags = libwacom_get_integration_flags(device);
if (flags & WACOM_DEVICE_INTEGRATED_DISPLAY) {
    /* Pen display: e.g. Wacom Cintiq, Huion Kamvas */
}

libwacom_destroy(device);
libwacom_database_destroy(db);
libwacom_error_free(&err);
```

The `libwacom-list-local-devices` CLI tool lists all recognised devices currently connected to the system:

```bash
libwacom-list-local-devices
# Output:
# Wacom Intuos Pro M (USB, 056a:0357) @ /dev/input/event5
```

### 4.3 Stylus Database

Each `WacomDevice` exposes its associated styli through a separate API introduced in libwacom 2.14:

```c
/* Get list of stylus IDs for this tablet */
const int *stylus_ids = libwacom_get_styli(device, &nstyli);

for (int i = 0; i < nstyli; i++) {
    const WacomStylus *stylus =
        libwacom_stylus_get_for_id(db, stylus_ids[i]);

    const char *sname = libwacom_stylus_get_name(stylus);
    /* Axes bitmask: WACOM_STYLUS_AXIS_PRESSURE, _TILT, _ROTATION, _WHEEL */
    WacomAxisTypeFlags axes = libwacom_stylus_get_axes(stylus);
    int nbuttons = libwacom_stylus_get_num_buttons(stylus);
    bool has_eraser = libwacom_stylus_has_eraser(stylus);
}
```

### 4.4 How Compositors Use libwacom

**GNOME Settings** (`gnome-control-center`) reads libwacom to populate the Wacom tablet configuration panel: it discovers button counts, determines whether the device has a touchring, and shows a diagram of the physical pad layout rendered from the SVG file referenced in the `.tablet` file.

**wlroots** calls libwacom during tablet pad initialisation to query the number of mode groups and their button/ring/strip membership. This information drives the `zwp_tablet_pad_group_v2` object hierarchy that the compositor advertises to clients. Pads on Wacom Intuos Pro devices have two independently-modeswitchable groups, which libwacom encodes in the `.tablet` file.

**libinput** itself uses libwacom to determine whether left-handed mode should rotate the tablet 180°, to apply correct pressure tip thresholds, and to configure the evdev calibration matrix for pen displays.

---

## 5. wl_touch: Core Wayland Touch Protocol

`wl_touch` is part of the Wayland core protocol (`wayland.xml`). It lives under `wl_seat` and delivers multi-touch events for touchscreens in surface-local coordinates.

### 5.1 Obtaining a wl_touch Object

```c
/* After binding wl_seat and receiving wl_seat.capabilities */
if (capabilities & WL_SEAT_CAPABILITY_TOUCH) {
    struct wl_touch *touch = wl_seat_get_touch(seat);
    wl_touch_add_listener(touch, &touch_listener, NULL);
}
```

A `wl_touch` is always associated with the `wl_seat` that created it. Touch events are delivered in the coordinate space of the `wl_surface` that received the initial `touch.down` event for that contact ID.

### 5.2 Touch Events

The `wl_touch` interface (protocol version 1 through 9) exposes the following events:

```
wl_touch.down(serial, time, surface, id, x, y)
```
- `serial` — opaque serial for input grabs
- `time` — millisecond timestamp
- `surface` — the `wl_surface` the contact landed on
- `id` — touch point ID (integer, unique among currently active contacts)
- `x`, `y` — `wl_fixed` coordinates in surface-local space

```
wl_touch.up(serial, time, id)
```
Finger was lifted. The `id` is retired and may be reused for a future contact.

```
wl_touch.motion(time, id, x, y)
```
An active contact moved. The `surface` is implicit (same as the one from `down`).

```
wl_touch.frame()
```
All contacts in the current hardware scan have been reported. Clients must accumulate all `down`/`motion`/`up` events within the same frame and process them atomically. See Section 5.3.

```
wl_touch.cancel()
```
The compositor cancels the entire touch sequence. All active contact IDs are invalidated. This is sent when the compositor needs the touch sequence for its own gesture recognition (e.g. workspace switch swipe). Clients should discard any in-progress gestures.

### 5.3 Frame Batching and Cancellation

The `frame` event is mandatory for correct multi-touch handling. Hardware often reports all contact updates within one interrupt cycle; the compositor emits these as a stream of `down`/`motion` events followed by a single `frame`. An application that processes events immediately without waiting for `frame` may act on partial state — for example, seeing finger A move before finger B has been updated, making two-finger zoom calculations incorrect.

A correct event loop accumulates pending state:

```c
static void touch_down(void *data, struct wl_touch *wl_touch,
    uint32_t serial, uint32_t time, struct wl_surface *surface,
    int32_t id, wl_fixed_t x, wl_fixed_t y)
{
    pending_contacts[id].x = wl_fixed_to_double(x);
    pending_contacts[id].y = wl_fixed_to_double(y);
    pending_contacts[id].active = true;
}

static void touch_frame(void *data, struct wl_touch *wl_touch)
{
    /* Only here do we dispatch to gesture recognisers */
    process_contacts(pending_contacts);
}

static void touch_cancel(void *data, struct wl_touch *wl_touch)
{
    memset(pending_contacts, 0, sizeof(pending_contacts));
    cancel_all_gestures();
}
```

### 5.4 Shape and Orientation Extensions (v6)

Protocol version 6 added two events that provide richer finger contact information for hardware capable of reporting them:

```
wl_touch.shape(id, major, minor)
```
Approximates the touch contact as an ellipse. `major` is the length of the longer semi-axis; `minor` is the shorter. Both are in surface-local units. A circular fingertip reports `major == minor`; a finger laid flat reports a larger `major`. This event is sent before the `frame` event within the same scan cycle as the `down` or `motion` event it accompanies.

```
wl_touch.orientation(id, orientation)
```
Reports the clockwise angle (in degrees, −180 to +180) of the contact ellipse's major axis relative to the positive surface Y-axis (pointing downward). A value of 0° means the major axis is aligned with the Y-axis (finger pointing down). The event is sent alongside `shape` when both are available.

These events are optional; compositors only send them if the underlying hardware and driver support the `ABS_MT_TOUCH_MAJOR`, `ABS_MT_TOUCH_MINOR`, and `ABS_MT_ORIENTATION` kernel axes.

**wlroots compositor delivery:**

In wlroots, touch events flow from `wlr_input_device` → `wlr_touch` → `wlr_seat_touch_send_down()`. The seat implementation iterates all `wl_touch` resource objects for the focused client and calls `wl_resource_post_event()` with the appropriate opcode.

---

## 6. zwp_tablet_manager_v2: Professional Stylus Protocol

The tablet protocol provides the Wayland extension for professional drawing tablet input. It is defined in `unstable/tablet/tablet-unstable-v2.xml` in the wayland-protocols repository — declared stable in February 2024 but retaining its `unstable/` path and `zwp_` prefix for backwards compatibility with existing compositors and clients. [Source: wayland-protocols GitLab](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/unstable/tablet/tablet-unstable-v2.xml)

### 6.1 Protocol Architecture

The object hierarchy follows a strict containment structure:

```
wl_registry
  └─ zwp_tablet_manager_v2           (compositor global singleton)
       └─ zwp_tablet_seat_v2         (one per wl_seat)
            ├─ zwp_tablet_v2         (one per physical tablet device)
            ├─ zwp_tablet_tool_v2    (one per physical tool in proximity)
            └─ zwp_tablet_pad_v2     (one per physical pad)
                  └─ zwp_tablet_pad_group_v2   (one per mode group)
                        ├─ zwp_tablet_pad_ring_v2
                        ├─ zwp_tablet_pad_strip_v2
                        └─ zwp_tablet_pad_dial_v2  (v2 addition)
```

The `zwp_tablet_seat_v2` emits three announcement events:
- `tablet_added(new_id<zwp_tablet_v2>)` — a tablet became available
- `tool_added(new_id<zwp_tablet_tool_v2>)` — a tool entered the sensor for the first time (per unique hardware serial)
- `pad_added(new_id<zwp_tablet_pad_v2>)` — a pad became available

**Tool uniqueness:** the compositor reuses the same `zwp_tablet_tool_v2` object across proximity cycles for the same physical tool (identified by its hardware serial number). Clients should not destroy the object when they receive `proximity_out`; they should keep it to receive future `proximity_in` events on the same tool.

Binding sequence:

```c
/* In wl_registry.global callback */
if (strcmp(interface, "zwp_tablet_manager_v2") == 0) {
    tablet_manager = wl_registry_bind(registry, name,
        &zwp_tablet_manager_v2_interface, 1);
    tablet_seat = zwp_tablet_manager_v2_get_tablet_seat(
        tablet_manager, seat);
    zwp_tablet_seat_v2_add_listener(tablet_seat,
        &tablet_seat_listener, NULL);
}
```

### 6.2 zwp_tablet_tool_v2 Events and Enumerations

A `zwp_tablet_tool_v2` object first announces its static properties through a sequence of events, terminated by `done`. These are sent once at tool creation:

| Event | Description |
|-------|-------------|
| `type(tool_type)` | Tool classification (pen, eraser, brush, etc.) |
| `hardware_serial(hi, lo)` | 64-bit hardware serial (unique pen identification) |
| `hardware_id_wacom(hi, lo)` | Wacom-format tool ID for model identification |
| `capability(capability)` | One event per axis capability (repeatable) |
| `done()` | Static description complete |

**Tool type enumeration (`zwp_tablet_tool_v2.type`):**

| Value | Hex | Meaning |
|-------|-----|---------|
| `type_pen` | 0x140 | Standard drawing stylus |
| `type_eraser` | 0x141 | Eraser end |
| `type_brush` | 0x142 | Brush tool |
| `type_pencil` | 0x143 | Pencil tool |
| `type_airbrush` | 0x144 | Airbrush with thumbwheel |
| `type_finger` | 0x145 | Finger (on touch+pen hybrid) |
| `type_mouse` | 0x146 | Tablet mouse (puck) |
| `type_lens` | 0x147 | Lens cursor |

The hex values correspond directly to the Linux kernel `BTN_TOOL_*` event codes.

**Capability enumeration (`zwp_tablet_tool_v2.capability`):**

| Value | Meaning |
|-------|---------|
| 1 | `tilt` — tilt_x / tilt_y axes available |
| 2 | `pressure` — pressure axis available |
| 3 | `distance` — distance axis available |
| 4 | `rotation` — barrel rotation available |
| 5 | `slider` — airbrush slider available |
| 6 | `wheel` — scroll wheel available |

**Dynamic events (sent during tool use):**

```
proximity_in(serial, tablet, surface)   tool entered hover range of a surface
proximity_out()                         tool left the sensor range entirely
down(serial)                            tip touched the surface
up()                                    tip lifted
motion(x, y)                            position changed (wl_fixed, surface-local)
pressure(value)                         pressure 0–65535
distance(distance)                      hover distance 0–65535
tilt(tilt_x, tilt_y)                   tilt in degrees (wl_fixed, signed)
rotation(degrees)                       barrel rotation 0–360 (wl_fixed)
slider(position)                        slider −65535 to +65535
wheel(degrees, clicks)                  wheel delta
button(serial, button, state)           side button press/release
frame(time)                             end of event batch (mandatory)
```

The `frame(time)` event is the counterpart to `wl_touch.frame()` — all analogue axis events for a single hardware scan cycle are delivered before the `frame`, and clients should accumulate them before updating rendering state.

**Tool set_cursor request:**

```c
/* Replace the default tablet cursor for this tool */
zwp_tablet_tool_v2_set_cursor(tool, serial,
    cursor_surface, hotspot_x, hotspot_y);
/* Passing NULL surface hides the cursor */
```

This allows applications to show a crosshair or custom brush cursor instead of the default pointer while the stylus is in proximity.

### 6.3 zwp_tablet_pad_v2: Rings, Strips, and Dials

Tablet pads (physical button decks, e.g. Wacom ExpressKey Remote, Intuos Pro side controls) are represented by `zwp_tablet_pad_v2`. On initialisation the compositor sends:

1. One `group(new_id<zwp_tablet_pad_group_v2>)` event per mode group
2. `path(path)` — the evdev node path for this pad
3. `buttons(count)` — total number of physical buttons
4. `done()` — static description complete

Each `zwp_tablet_pad_group_v2` in turn announces:
- `buttons(array)` — which button indices belong to this group
- `ring(new_id<zwp_tablet_pad_ring_v2>)` — for each touch ring in the group
- `strip(new_id<zwp_tablet_pad_strip_v2>)` — for each touch strip
- `modes(count)` — number of switchable modes in this group
- `done()` — group description complete

**Pad events during use:**

```
zwp_tablet_pad_v2.button(time, button, state)    button pressed/released
zwp_tablet_pad_v2.enter(serial, tablet, surface) pad gained focus on a surface
zwp_tablet_pad_v2.leave(serial, surface)         pad lost focus
```

**Ring events:**
```
zwp_tablet_pad_ring_v2.source(source)   source: finger (value 1)
zwp_tablet_pad_ring_v2.angle(degrees)   position 0.0–360.0 (wl_fixed)
zwp_tablet_pad_ring_v2.stop()          contact ended
zwp_tablet_pad_ring_v2.frame(time)     end of ring event batch
```

**Strip events:**
```
zwp_tablet_pad_strip_v2.source(source)     source: finger (value 1)
zwp_tablet_pad_strip_v2.position(position) 0–65535 (top to bottom)
zwp_tablet_pad_strip_v2.stop()             contact ended
zwp_tablet_pad_strip_v2.frame(time)        end of strip batch
```

**Dial events (v2 addition):**
```
zwp_tablet_pad_dial_v2.delta(value120)  rotation delta in fractions of 120 per detent
zwp_tablet_pad_dial_v2.frame(time)      end of dial event batch
```

Dials are present on newer Wacom devices (e.g. Wacom Pro Pen 3 display). The `value120` encoding follows the high-resolution wheel convention: one full mechanical detent produces a `value120` of 120, while high-resolution scroll hardware can emit sub-detent values. This matches the encoding used by `wp_pointer_axis_value120` in the core Wayland pointer protocol.

### 6.4 Mode Switching

Mode switching allows a single physical pad group to present multiple independent button binding sets. For example, a four-button group in three-mode configuration gives effectively twelve bindings.

```
zwp_tablet_pad_group_v2.mode_switch(time, serial, mode)
```

The `serial` identifies the specific button press that triggered the mode switch (the compositor may use this to grant interactive popups). The `mode` value is 0-indexed.

To provide user feedback, clients (or compositors' shell components) call:

```c
/* Set a human-readable label for what this button does in the current mode */
zwp_tablet_pad_v2_set_feedback(pad, button_index,
    "Brush Size", mode_switch_serial);

/* Similarly for rings: */
zwp_tablet_pad_ring_v2_set_feedback(ring,
    "Zoom", mode_switch_serial);
```

The compositor can display these strings in an on-screen overlay when the mode changes.

---

## 7. Compositor Implementation: wlroots Tablet Support

wlroots (the Wayland compositor library) implements the tablet protocol in `types/wlr_tablet_v2.c`. [Source: wlroots GitLab](https://gitlab.freedesktop.org/wlroots/wlroots/-/blob/master/types/wlr_tablet_v2.c)

### 7.1 Tablet Manager and Device Registration

```c
#include <wlr/types/wlr_tablet_v2.h>

/* Create the zwp_tablet_manager_v2 global */
struct wlr_tablet_manager_v2 *tablet_manager =
    wlr_tablet_v2_create(server->wl_display);

/* When a tablet input device is attached to a seat */
struct wlr_tablet_v2_tablet *tablet =
    wlr_tablet_create(tablet_manager, seat, input_device);

/* When a pad device is attached */
struct wlr_tablet_v2_tablet_pad *pad =
    wlr_tablet_pad_create(tablet_manager, seat, input_device);
```

The `wlr_tablet_tool` struct lives in the backend layer:

```c
/* From include/wlr/types/wlr_tablet_tool.h */
struct wlr_tablet_tool {
    enum wlr_tablet_tool_type type;
    uint64_t hardware_serial;
    uint64_t hardware_wacom;
    bool tilt;
    bool pressure;
    bool distance;
    bool rotation;
    bool slider;
    bool wheel;
    struct {
        struct wl_signal destroy;
    } events;
    void *data;
};
```

Tool objects are created by the backend when a tool enters proximity and destroyed when the backend removes them; the compositor-level `wlr_tablet_v2_tablet_tool` wraps them for protocol delivery.

### 7.2 Delivering Axis and Proximity Events

When libinput delivers a `LIBINPUT_EVENT_TABLET_TOOL_PROXIMITY` event, the compositor calls:

```c
/* Tool entering proximity of a surface */
wlr_tablet_v2_tablet_tool_notify_proximity_in(
    v2_tool,    /* wlr_tablet_v2_tablet_tool */
    v2_tablet,  /* wlr_tablet_v2_tablet */
    surface);   /* wlr_surface the cursor is over */
```

For axis updates (motion, pressure, tilt), wlroots uses a single batched call:

```c
wlr_tablet_v2_tablet_tool_notify_motion(v2_tool, x, y);
wlr_tablet_v2_tablet_tool_notify_pressure(v2_tool, pressure);
wlr_tablet_v2_tablet_tool_notify_tilt(v2_tool, tilt_x, tilt_y);
/* ... other axes ... */
/* Then close the frame: */
wlr_tablet_v2_tablet_tool_notify_frame(v2_tool, time_ms);
```

Each `notify_*` call posts the corresponding `zwp_tablet_tool_v2` event to the focused client's Wayland socket.

For pad events:

```c
wlr_tablet_v2_tablet_pad_notify_enter(v2_pad, v2_tablet, surface);
wlr_tablet_v2_tablet_pad_notify_button(v2_pad, button, time,
    WLR_BUTTON_PRESSED);
wlr_tablet_v2_tablet_pad_notify_ring(v2_pad, ring_index, position,
    is_finger, time);
wlr_tablet_v2_tablet_pad_notify_mode(v2_pad, group_index, mode,
    time, serial);
```

### 7.3 Cursor Handling and Coordinate Mapping

While a stylus is in proximity, the compositor must:

1. **Hide the pointer cursor** — the stylus position is tracked independently.
2. **Apply the tool cursor** set by the client via `zwp_tablet_tool_v2.set_cursor`, or fall back to a default crosshair.
3. **Map tablet coordinates to screen pixels** — the tablet's physical coordinate space (reported in libinput device-space units) must be transformed to screen output pixels. For standalone tablets this is typically a full-screen mapping: `x_screen = x_tablet / tablet_width_px * output_width`. For pen displays the tablet surface maps 1:1 to the display, but the compositor must apply the output's scale factor and position within the global output layout.
4. **Find the focused surface** — the compositor performs a surface hit-test at the mapped screen position and sends `proximity_in` with the correct `wl_surface`.

For pen displays (integrated display + tablet), `out-of-bounds` coordinates may legitimately exceed the tablet active area when the pen is at an extreme edge. libinput does not clamp these; the compositor should clamp to the output rectangle.

**Input grabs** work identically to pointer grabs: when a client initiates an `xdg_surface` move or resize using the `serial` from a `down` or `button` event, the compositor locks tablet events to that client for the duration of the interactive operation.

---

## 8. Creative Application Integration

### 8.1 GTK4 and GDK Wayland Device Model

GDK's Wayland backend consumes `zwp_tablet_tool_v2` and `wl_touch` events and presents them to applications through the `GdkDevice` / `GdkEvent` API. The device model differs substantially between GTK3 and GTK4.

**GTK3 device model (used by GIMP 3.0):** GDK3 tracks a master/slave device pair for each physical input device. Each `zwp_tablet_tool_v2` in proximity creates a `GdkDevice` of type `GDK_DEVICE_TYPE_SLAVE` (the physical tool) paired with a `GDK_DEVICE_TYPE_MASTER` logical device. The `GdkDeviceManager` (deprecated in GTK4) enumerates all devices on a `GdkDisplay`. Tablet axis values arrive on `GdkEvent` records and are retrieved with:

```c
/* GTK3 axis API — used by GIMP 3.0 */
double pressure = 0.0, tilt_x = 0.0, tilt_y = 0.0;
gdk_event_get_axis(event, GDK_AXIS_PRESSURE, &pressure);
gdk_event_get_axis(event, GDK_AXIS_XTILT,    &tilt_x);
gdk_event_get_axis(event, GDK_AXIS_YTILT,    &tilt_y);
```

**GTK4 device model (used by Inkscape and future applications):** GTK4 removed `GdkDeviceManager` and the master/slave terminology. The seat-centric model uses `GdkSeat` to enumerate devices:

```c
/* GTK4: get physical devices from the seat */
GdkDisplay *display = gdk_display_get_default();
GdkSeat    *seat    = gdk_display_get_default_seat(display);
GList *devices = gdk_seat_get_devices(seat,
    GDK_SEAT_CAPABILITY_TABLET_STYLUS);
for (GList *l = devices; l; l = l->next) {
    GdkDevice *dev = l->data;
    GdkInputSource src = gdk_device_get_source(dev);
    /* GDK_SOURCE_PEN, GDK_SOURCE_ERASER, GDK_SOURCE_TABLET_PAD, etc. */
}
g_list_free(devices);
```

GTK4 also exposes tool identity through `GdkDeviceTool` (obtained via `gdk_event_get_device_tool()`), which carries a `GdkDeviceToolType` (`GDK_DEVICE_TOOL_TYPE_PEN`, `_ERASER`, `_BRUSH`, `_PENCIL`, `_AIRBRUSH`, `_MOUSE`, `_LENS`) and a hardware serial number matching the `zwp_tablet_tool_v2.hardware_serial` event.

The GDK Wayland source implementing tablet event translation is `gdk/wayland/gdkdevice-wayland.c`. [Source: GTK source](https://github.com/GNOME/gtk/blob/main/gdk/wayland/gdkdevice-wayland.c)

Touch events (`wl_touch`) are delivered via `GdkEventTouch` (GTK3) or through gesture controllers (GTK4) with types `GDK_TOUCH_BEGIN`, `GDK_TOUCH_UPDATE`, `GDK_TOUCH_END`, and `GDK_TOUCH_CANCEL`. The touch sequence identifier maps from the Wayland `id` (the touch point ID).

### 8.2 GIMP Tablet Handling

GIMP (≥ 2.99 / 3.0) targets GTK3 (with GTK4 migration underway) and receives tablet events through GDK's device axis API described above. On Wayland, GIMP 3.0 added proper hotplug support, receiving `GdkDevice` add/remove signals when tools enter and leave proximity.

The dynamics system maps raw axis values to brush parameters through `GimpDynamics` objects. Each dynamics output (size, opacity, hardness, spacing, etc.) can be independently bound to pressure, tilt, wheel, or random inputs. The pressure curve maps the 0.0–1.0 GDK axis value through a configurable spline, allowing artists to tune response for their specific tablet and drawing style. The dynamics pipeline is executed in `app/paint/gimpdynamics.c`.

For tablet configuration, GIMP reads `GDK_AXIS_PRESSURE` and `GDK_AXIS_XTILT`/`GDK_AXIS_YTILT` per-event. Device-specific pressure curve calibration is stored in the GIMP preferences file (`~/.config/GIMP/2.10/devicerc` or the equivalent 3.0 path). On Wayland this is equivalent to X11 operation since GDK abstracts the protocol.

> Note: The author's inspection of GIMP 3.0 internals for Wayland-specific tablet code paths was limited by access to the source at time of writing. Specific function names should be verified against the GIMP 3.0 git repository at `https://gitlab.gnome.org/GNOME/gimp`.

### 8.3 Krita and the Qt Wayland Tablet Plugin

Krita 6.0 (released 2026) is built on Qt 6 and runs natively on Wayland without XWayland. Qt's Wayland tablet support is implemented in the `qtwayland` module as `QWaylandTabletManagerV2`, which translates `zwp_tablet_tool_v2` events into Qt's `QTabletEvent` objects. [Source: Qt Wayland tablet commit](https://github.com/qt/qtwayland/commit/222455cd643c128fa9730c9c527a7fdcadd0acfe)

Qt 6.8 delivered substantial improvements to the Wayland tablet plugin, including:
- **Cursor support** — the plugin now calls `zwp_tablet_tool_v2_set_cursor()` to honour Qt cursor shapes for tablet input
- **Serial tracking** — the `serial` from `down` and `button` events is preserved, enabling `xdg_toplevel.move` (window dragging with stylus) and drag-and-drop to work correctly
- **Keyboard modifier forwarding** — modifier state (Ctrl, Shift, Alt) is now included in `QTabletEvent`, enabling modifier-sensitive operations like constraining strokes to angles
- **Client-side decoration handling** — stylus input on window decorations is correctly forwarded as synthetic mouse events

[Source: Qt Wayland Tablet Improvements blog](https://nicolasfella.de/posts/qt-wayland-tablet-improvements/)

Applications receive `QTabletEvent` in their widget's `tabletEvent(QTabletEvent *event)` override:

```cpp
void MyCanvas::tabletEvent(QTabletEvent *event) {
    if (event->type() == QEvent::TabletMove) {
        double pressure = event->pressure();    // 0.0–1.0
        double tilt_x  = event->xTilt();       // degrees
        double tilt_y  = event->yTilt();        // degrees
        double rotation = event->rotation();    // degrees
        QPointF pos     = event->position();    // widget-local coords
        updateBrush(pos, pressure, tilt_x, tilt_y, rotation);
    }
    event->accept();
}
```

At the application level, `QTabletEvent` values feed Krita's brush dynamics system: pressure typically drives stroke width and opacity, tilt drives brush angle, and rotation drives barrel orientation for tools that support it. The specific internal class chain (`KisTabletEvent`, `KisToolFreehandHelper`, `KisPaintOp` subclasses) is documented in Krita's developer wiki and source tree (`libs/brush/`, `plugins/paintops/`) — exact class names should be verified against the Krita 6 source at `https://invent.kde.org/graphics/krita`.

To verify the platform:

```cpp
if (qGuiApp->platformName() == QLatin1String("wayland")) {
    // Running natively on Wayland; tablet events via zwp_tablet_tool_v2
} else if (qGuiApp->platformName() == QLatin1String("xcb")) {
    // Running on X11 or XWayland; tablet events via XInput2
}
```

### 8.4 Inkscape on GTK4 Wayland

Inkscape's migration to GTK4 brings native Wayland tablet support through GDK4's unified device model. GTK4 exposes pad buttons through `GdkDevicePad` and reports stylus type through `GdkDeviceToolType` (values: `GDK_DEVICE_TOOL_TYPE_PEN`, `_ERASER`, `_BRUSH`, `_PENCIL`, `_AIRBRUSH`, `_MOUSE`, `_LENS`).

Inkscape's calligraphy tool consumes tilt and pressure to control stroke width and nib angle. Pressure data feeds into the `PRESSURE_TOOL` attribute stored in the generated SVG path's style attribute, allowing re-rendering at different pressure levels. Tilt angle is used to compute the calligraphy pen orientation, simulating the effect of a flat-nibbed calligraphy pen held at an angle.

---

## 9. Calibration, Mapping, and Multi-Monitor Considerations

**Tablet area mapping** is the relationship between the physical drawing surface of the tablet and the screen region it controls. By default libinput maps the entire tablet area to the entire compositor output space. For pen displays (integrated screen + digitiser), the physical drawing area already matches the screen precisely, so no remapping is needed beyond applying the output scale factor.

For standalone tablets with multiple monitors:

- The compositor must pick an output to assign the tablet to. If no assignment is made, the tablet's coordinate space is stretched across the entire virtual desktop (the union of all outputs), which degrades precision on large setups.
- `GNOME Settings → Wacom` provides a UI to map the tablet to a specific screen using `gnome-control-center wacom`. Internally Mutter stores this mapping per device in its settings store and applies a transform matrix to libinput coordinates.
- On KDE Plasma, `System Settings → Input Devices → Drawing Tablet` provides equivalent per-output mapping.

**HiDPI and fractional scaling:** Wayland surface coordinates are always in logical pixels (before scale factor application). A tablet coordinate mapped to a 2× scaled output must be divided by 2 to yield the correct logical pixel position for `zwp_tablet_tool_v2.motion`. libinput's `get_x_transformed()` returns screen pixels; the compositor divides by `wl_output.scale` or the fractional scale factor to obtain logical pixels.

**Area restriction** (available in libinput ≥ 1.24):

```c
/* Restrict the active tablet area to a rectangle within the physical surface,
   useful for matching a specific aspect ratio or mapping to a screen region */
struct libinput_config_area area = {
    .x1 = 0.2, .y1 = 0.0,   /* 20% from left */
    .x2 = 0.8, .y2 = 1.0,   /* 80% from left */
};
libinput_device_config_area_set_rectangle(device, &area);
```

**Rotation for convertible laptops:** When a convertible laptop rotates to portrait orientation, the compositor should rotate the touch and stylus coordinate system to match. libinput provides:

```c
/* Rotate the input coordinate system 90° clockwise to match
   the display rotation */
libinput_device_config_rotation_set_angle(device, 90);
```

`iio-sensor-proxy` reads the accelerometer and notifies the compositor via D-Bus when the physical orientation changes; the compositor then updates the libinput rotation configuration for all touch and tablet devices.

**Touch screen calibration:** Some resistive touch screens require an affine calibration matrix to correct for manufacturing tolerances. libinput applies the matrix from the `LIBINPUT_CALIBRATION_MATRIX` udev property:

```
# /etc/udev/rules.d/99-touchscreen-cal.rules
ENV{LIBINPUT_CALIBRATION_MATRIX}="1.0 0.0 0.0 0.0 1.0 0.0"
```

The six values encode the top two rows of a 3×3 affine transform in row-major order.

**Input remapping:** For remapping tablet pad buttons beyond what compositor settings panels offer, `input-remapper` ([https://github.com/sezanzeb/input-remapper](https://github.com/sezanzeb/input-remapper)) provides a userspace daemon that reads from evdev and synthesises new events via `uinput`, allowing arbitrary key combinations or macros to be assigned to express keys.

**xsetwacom legacy note:** On X11, `xsetwacom` (from the `xf86-input-wacom` X11 driver) was the standard tool for tablet configuration including area mapping, button remapping, and pressure curve adjustment. On Wayland none of these X11 mechanisms apply; configuration must go through the compositor's native APIs (Wayland protocols, D-Bus, or the compositor's own IPC) or through libinput configuration as described above.

---

## Roadmap

### Near-term (6–12 months)

- **libinput Lua plugin system stabilisation (libinput ≥ 1.30):** The public plugin API, shipped in libinput 1.30 with Lua as the scripting language, enables compositor vendors and OEM partners to ship device-specific tablet quirks and pressure-curve overrides as loadable plugins rather than upstream patches. Further stabilisation of the plugin ABI and documentation of tablet-specific hooks are expected in the 1.31–1.32 cycle. [Source: libinput 1.30 Lua Plugins](https://wayland.freedesktop.org/libinput/doc/1.30.1/lua-plugins.html)
- **Tablet pad relative dials in KWin and Sway:** GNOME Mutter merged support for `zwp_tablet_pad_v2` relative dial events in 2025, targeting GNOME 49. KWin and wlroots-based compositors (Sway, Hyprland) are expected to follow with equivalent dial support once the libinput and libwacom changes propagate to stable distributions. [Source: Phoronix — GNOME Mutter Tablet Pad Dials](https://www.phoronix.com/news/GNOME-Mutter-Tablet-Pad-Dials)
- **Linux 6.12+ Wacom driver improvements:** Linux 6.12 extended the `wacom.ko` HID driver with high-resolution wheel scrolling, relative-motion touch rings, and dual touch-ring support for applicable hardware. Follow-on patches for newer Wacom Pro Pen 3 devices and additional Huion / XP-Pen HID quirks are in flight on the linux-input mailing list. [Source: Phoronix — Linux 6.12 HID](https://www.phoronix.com/news/Linux-6.12-HID)
- **libinput area restriction API adoption:** The `libinput_device_config_area_set_rectangle()` API (libinput ≥ 1.24) for restricting the active tablet drawing area is not yet surfaced in GNOME Control Center or KDE System Settings. Compositor UI work to expose per-output area mapping through this API is planned for both desktops. Note: needs verification for specific release targets.
- **DIGImend out-of-tree driver upstreaming:** The DIGImend project ([https://github.com/DIGImend/digimend-kernel-drivers](https://github.com/DIGImend/digimend-kernel-drivers)) maintains `hid-uclogic` extensions for Huion and XP-Pen tablets not yet covered upstream. Ongoing collaboration with the kernel HID maintainers aims to upstream remaining device tables into `hid-uclogic.ko` proper.

### Medium-term (1–3 years)

- **`zwp_tablet_manager_v3` protocol revision:** The `tablet-unstable-v2` protocol has been stable in practice for years but never formally promoted to `wp_tablet_v1` stable. A v3 revision is discussed in the wayland-protocols issue tracker to address outstanding gaps: richer tool capability advertisement (e.g., distinguishing airbrush from felt-pen beyond the current `type` enum), per-tool pressure curve negotiation, and high-resolution touch coordinates aligned with the `wl_pointer` high-resolution motion work. Note: needs verification for specific MR numbers.
- **Wayland touch protocol high-resolution coordinates:** The `wl_touch.motion` event currently reports surface-local coordinates in the same fixed-point 24.8 format as `wl_pointer`. A proposed extension to deliver sub-pixel touch coordinates (matching what multitouch Protocol B provides in evdev) is under informal discussion in the Wayland community, relevant for high-DPI stylus displays. [Source: wayland-protocols cgit log](https://cgit.freedesktop.org/wayland/wayland-protocols/log/)
- **libwacom Bluetooth LE tablet support:** Many modern Wacom and Huion tablets connect over Bluetooth LE rather than classic Bluetooth or USB. The libwacom device database and the kernel `wacom.ko` Bluetooth path need updates for newer BLE tablet models. libwacom maintainers have noted BLE quirk entries as a medium-term priority. Note: needs verification.
- **Pressure calibration protocol:** Currently pressure curves are applied in userspace (per-application or via `input-remapper`). A Wayland protocol extension for compositor-level pressure curve configuration — analogous to xsetwacom's `PressureCurve` parameter — has been informally discussed to allow system-wide consistent pressure handling across GTK, Qt, and SDL applications. Note: needs verification.
- **wlroots tablet v2 stable promotion:** The `wlr_tablet_v2` implementation in wlroots remains backed by the `unstable` protocol namespace. As the wayland-protocols governance process moves toward formally stabilising `tablet-v2`, wlroots will update its implementation to track the stable interface names, requiring compositor API changes in Sway and downstream compositors.

### Long-term

- **Unified input extension framework:** The Linux input stack currently has separate kernel-to-userspace paths for tablets (`zwp_tablet_manager_v2`), touch (`wl_touch`), gestures (`zwp_pointer_gestures_v1`), and gamepads. A longer-term architectural goal discussed in the Wayland community is a unified `wp_input_v1` extension that exposes all extended input device capabilities through a consistent object model, reducing per-device-class protocol proliferation. Note: speculative, no active proposal.
- **In-kernel tablet calibration via BPF:** The Linux HID subsystem gained BPF-based event rewriting (`hid-bpf`) in kernel 6.3. Long-term, device-specific calibration matrices and axis remapping for tablets could be applied at the kernel BPF layer, eliminating the need for userspace quirk databases like libwacom for low-level axis correction. Note: speculative.
- **AI-assisted pressure and tilt prediction:** Low-latency stylus input on touch displays suffers from prediction lag at the tip. Research prototypes have demonstrated neural-network-based stylus trajectory prediction integrated at the compositor level. Whether this lands as a standard libinput feature or a compositor-specific extension remains an open question.
- **Cross-platform tablet protocol convergence:** As ChromeOS and Android also run Wayland compositors (Exo, Arc++), there is potential for convergence between the Linux `zwp_tablet_manager_v2` protocol and Android's `MotionEvent` stylus model. This could enable a single cross-platform API surface for stylus-aware creative applications targeting multiple platforms. Note: speculative.

---

## 10. Integrations

This chapter connects to the following chapters in the book:

**Ch54 — The Linux Input Stack** describes the complete kernel-to-compositor input pipeline that this chapter extends for tablet and touch-specific semantics. libinput's core event loop, device lifecycle management, and the evdev kernel ring buffer are covered there; this chapter builds on that foundation with tablet-specific event types and axes.

**Ch20 — Wayland Protocol Fundamentals** covers `wl_seat` and the core seat capability model; `wl_touch` is part of this core and is introduced alongside `wl_pointer` and `wl_keyboard` there. This chapter extends that treatment with the full event API and frame-batching semantics.

**Ch21 — Building Compositors with wlroots** covers wlroots architecture and the `wlr_seat` abstraction. The `wlr_tablet_v2` types and notification functions described in Section 7 are part of the wlroots protocol implementation layer explained in that chapter.

**Ch22 — Production Compositors** covers Mutter (GNOME), KWin (KDE Plasma), and Sway in depth. Mutter's Wacom settings panel integration, KWin's tablet configuration UI, and Sway's tablet button binding for floating window resize are compositor-specific tablet features built on the protocol layer described here.

**Ch39 — Qt and GTK on Wayland** covers the toolkit Wayland backends at the level of window management, buffer submission, and input handling. The GDK Wayland device model (Section 8.1) and the Qt Wayland tablet plugin (Section 8.3) are introduced there; this chapter provides the tablet-specific deep dive.

**Ch130 — Wayland Protocol Extension Development** uses `zwp_tablet_manager_v2` as a case study in the design of Wayland extension protocols: how tool identity is maintained across proximity cycles, why the `frame` event is necessary for atomic multi-axis updates, how mode switching is designed to support compositor-side feedback overlays, and the rationale for tool type enumeration values mirroring kernel `BTN_TOOL_*` codes.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
