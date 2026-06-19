# Chapter 151: Wayland Text Input and IME Protocols

**Target audiences**: Wayland compositor developers implementing text input support; application developers porting desktop apps to Wayland who rely on IME for CJK, Arabic, or other complex-script input; toolkit maintainers integrating GTK/Qt text input with compositors.

---

## Table of Contents

1. [Introduction](#introduction)
2. [The IME Problem on Wayland](#the-ime-problem-on-wayland)
3. [zwp_text_input_v3: The Stable Protocol](#zwp_text_input_v3-the-stable-protocol)
4. [zwp_input_method_v2: The Input Method Side](#zwp_input_method_v2-the-input-method-side)
5. [Virtual Keyboard Protocol](#virtual-keyboard-protocol)
6. [Compositors: GNOME, KDE, wlroots](#compositors-gnome-kde-wlroots)
7. [Toolkit Integration: GTK4 and Qt6](#toolkit-integration-gtk4-and-qt6)
8. [fcitx5 and IBus on Wayland](#fcitx5-and-ibus-on-wayland)
9. [XWayland Text Input Bridge](#xwayland-text-input-bridge)
10. [Integrations](#integrations)

---

## Introduction

Text input on Wayland is significantly more complex than on X11, where XIM (X Input Method) was a well-established if imperfect solution. Wayland's security model — no global input snooping — means the input method (IME) and the application communicate only through the compositor, which mediates all focus and event routing.

This chapter covers the two Wayland unstable protocols that implement text input: `zwp_text_input_v3` (the application side) and `zwp_input_method_v2` (the IME side), plus the virtual keyboard protocol for on-screen keyboards. It then traces how GTK4, Qt6, fcitx5, and IBus implement these protocols, and how XWayland bridges them back to X11 apps.

Key upstream specifications live in [wayland-protocols](https://gitlab.freedesktop.org/wayland/wayland-protocols).

---

## The IME Problem on Wayland

### Why IME Is Hard on Wayland

X11 IME worked because X11 applications and the IME could both connect to the X server and share global state (keyboard grabs, window properties). Wayland eliminates global state by design.

On Wayland, the flow is:
```
Keyboard hardware
      │ (kernel evdev)
      ▼
Compositor (wlroots / Mutter / KWin)
  │ receives all input
  ▼
  ├─→ focused application window (via wl_keyboard.key)
  └─→ input method process (via zwp_input_method_v2)
```

The compositor knows which window has focus, routes raw key events to the IME, receives composed text back from the IME, and forwards it to the application via `zwp_text_input_v3`. This is more secure (no global keyboard grab) but requires three parties to coordinate.

### Protocol Version History

| Protocol | Status | Year | Notes |
|---|---|---|---|
| `zwp_text_input_v1` | Deprecated | 2013 | KDE-originated, still in some toolkits |
| `zwp_text_input_v3` | Unstable but de-facto standard | 2018 | GTK4, Qt6, wlroots |
| `ext_text_input_v1` | Proposed stable | 2024 | Planned stable replacement |
| `zwp_input_method_v2` | Unstable | 2018 | IME side; fcitx5, ibus |

---

## zwp_text_input_v3: The Stable Protocol

### Protocol Overview

`zwp_text_input_v3` is the application-side protocol. A text-aware application (or its toolkit) creates a `zwp_text_input_v3` object to tell the compositor:
- When text focus enters/leaves an editable field
- The cursor rectangle (so IME can position its popup)
- The surrounding text (for context-aware input methods)

In return, the compositor delivers:
- `commit_string` — the text to insert
- `preedit_string` — in-progress candidate text (shown with underline)
- `delete_surrounding_text` — deletion request

### Protocol XML (key requests)

```xml
<!-- wayland-protocols/unstable/text-input/text-input-unstable-v3.xml -->
<interface name="zwp_text_input_v3" version="1">
  <!-- Sent by client to enable/disable text input for a surface -->
  <request name="enable"/>
  <request name="disable"/>

  <!-- Sent by client to set cursor rectangle (x, y, w, h in surface coords) -->
  <request name="set_cursor_rectangle">
    <arg name="x" type="int"/>
    <arg name="y" type="int"/>
    <arg name="width" type="int"/>
    <arg name="height" type="int"/>
  </request>

  <!-- Send surrounding text for context (before/after cursor) -->
  <request name="set_surrounding_text">
    <arg name="text" type="string"/>
    <arg name="cursor" type="int"/>
    <arg name="anchor" type="int"/>
  </request>

  <!-- Commit the pending state to compositor -->
  <request name="commit"/>

  <!-- Events from compositor to application: -->
  <event name="preedit_string">
    <arg name="text" type="string" allow-null="true"/>
    <arg name="cursor_begin" type="int"/>
    <arg name="cursor_end" type="int"/>
  </event>

  <event name="commit_string">
    <arg name="text" type="string" allow-null="true"/>
  </event>

  <event name="delete_surrounding_text">
    <arg name="before_length" type="uint"/>
    <arg name="after_length" type="uint"/>
  </event>

  <event name="done">
    <arg name="serial" type="uint"/>
  </event>
</interface>
```

### Application Usage Pattern

```c
/* Application text input handling (conceptual): */
struct wl_seat *seat = ...; /* from wl_registry */
struct zwp_text_input_manager_v3 *tim = ...; /* from wl_registry */

/* Create per-seat text input object: */
struct zwp_text_input_v3 *ti =
    zwp_text_input_manager_v3_get_text_input(tim, seat);

static void ti_commit_string(void *data, struct zwp_text_input_v3 *ti,
    const char *text)
{
    editor_insert_text(data, text);
}

static void ti_preedit_string(void *data, struct zwp_text_input_v3 *ti,
    const char *text, int32_t cursor_begin, int32_t cursor_end)
{
    editor_set_preedit(data, text, cursor_begin, cursor_end);
}

static const struct zwp_text_input_v3_listener ti_listener = {
    .preedit_string          = ti_preedit_string,
    .commit_string           = ti_commit_string,
    .delete_surrounding_text = ti_delete_surrounding,
    .done                    = ti_done,
};
zwp_text_input_v3_add_listener(ti, &ti_listener, editor);

/* When focus enters text field: */
zwp_text_input_v3_enable(ti);
zwp_text_input_v3_set_cursor_rectangle(ti, cursor_x, cursor_y, 1, line_height);
zwp_text_input_v3_set_surrounding_text(ti, buffer_context, cursor_offset, cursor_offset);
zwp_text_input_v3_commit(ti);
```

### The Serial / Done Dance

`zwp_text_input_v3` uses a serial-based commit scheme to handle races:
1. Application calls `commit()` → sends state to compositor
2. Compositor may deliver multiple events (`preedit_string`, `commit_string`, `delete_surrounding_text`)
3. Compositor sends `done(serial)` to end the update batch
4. Application acknowledges by including the serial in its next `commit()`

This prevents race conditions between the application updating cursor position and the compositor forwarding IME events.

---

## zwp_input_method_v2: The Input Method Side

### Protocol Role

`zwp_input_method_v2` is used by the IME process (fcitx5, IBus, an on-screen keyboard). The compositor creates a `zwp_input_method_v2` for the IME when it connects:

```xml
<!-- The compositor sends these events to the IME: -->
<event name="activate"/>
<event name="deactivate"/>
<event name="surrounding_text">
  <arg name="text" type="string"/>
  <arg name="cursor" type="uint"/>
  <arg name="anchor" type="uint"/>
</event>
<event name="text_change_cause">
  <arg name="cause" type="uint"/>
</event>
<event name="content_type">
  <arg name="hint" type="uint"/>
  <arg name="purpose" type="uint"/>
</event>
<event name="done"/>

<!-- IME sends these requests to compositor: -->
<request name="commit_string">
  <arg name="text" type="string"/>
</request>
<request name="set_preedit_string">
  <arg name="text" type="string"/>
  <arg name="cursor_begin" type="int"/>
  <arg name="cursor_end" type="int"/>
</request>
<request name="delete_surrounding_text">
  <arg name="before_length" type="uint"/>
  <arg name="after_length" type="uint"/>
</request>
<request name="commit"/>
```

### IME Popup Window

The IME needs to display a candidate selection popup. This is a special Wayland surface type:

```xml
<interface name="zwp_input_method_popup_surface_v2">
  <event name="text_input_rectangle">
    <arg name="x" type="int"/>
    <arg name="y" type="int"/>
    <arg name="width" type="int"/>
    <arg name="height" type="int"/>
  </event>
</interface>
```

The compositor reports the text cursor position to the popup surface so the IME can position itself correctly.

---

## Virtual Keyboard Protocol

On-screen keyboards use `zwp_virtual_keyboard_v1`:

```xml
<request name="keymap">
  <arg name="format" type="uint"/>
  <arg name="fd" type="fd"/>
  <arg name="size" type="uint"/>
</request>
<request name="key">
  <arg name="time" type="uint"/>
  <arg name="key" type="uint"/>
  <arg name="state" type="uint"/>
</request>
<request name="modifiers">
  <arg name="mods_depressed" type="uint"/>
  <arg name="mods_latched" type="uint"/>
  <arg name="mods_locked" type="uint"/>
  <arg name="group" type="uint"/>
</request>
```

This is privileged (requires compositor permission or a trusted layer-shell surface) to prevent keylogger abuse.

```c
/* On-screen keyboard sending a key: */
zwp_virtual_keyboard_v1_key(vk, timestamp, KEY_A, WL_KEYBOARD_KEY_STATE_PRESSED);
zwp_virtual_keyboard_v1_key(vk, timestamp + 1, KEY_A, WL_KEYBOARD_KEY_STATE_RELEASED);
```

---

## Compositors: GNOME, KDE, wlroots

### wlroots Implementation

wlroots implements both `zwp_text_input_v3` and `zwp_input_method_v2`:

```c
/* wlroots/types/wlr_text_input_v3.c */
struct wlr_text_input_v3 {
    struct wl_resource *resource;
    struct wlr_seat *seat;
    struct wlr_surface *focused_surface;
    struct {
        char *surrounding_text;
        int32_t cursor, anchor;
        struct {int32_t x, y, w, h;} cursor_rect;
        bool enabled;
    } pending, current;
    uint32_t current_serial;
    struct wl_signal enable, disable, commit, destroy;
};
```

When an application calls `commit()`, wlroots emits the `commit` signal. Compositors built on wlroots (Sway, Hyprland, labwc) handle this signal by forwarding state to any connected `zwp_input_method_v2`.

### GNOME Mutter

GNOME implements text input in Mutter's clutter layer via a D-Bus bridge to IBus — Mutter acts as both the Wayland compositor and the IBus client:

```
GNOME special: Mutter bridges IM via:
  IBus D-Bus ↔ Mutter ↔ zwp_text_input_v3
  (not zwp_input_method_v2 which is wlroots-centric)
```

### KDE Plasma

KDE Plasma (KWin) implements `zwp_text_input_v3` natively and supports both fcitx5 (via `zwp_input_method_v2`) and IBus (via D-Bus):

```bash
# KDE text input debug:
QT_LOGGING_RULES="kwin.textinput=true" kwin_wayland --xwayland
```

---

## Toolkit Integration: GTK4 and Qt6

### GTK4 Text Input

GTK4's Wayland backend (`gdk/wayland/`) implements `GdkWaylandInputContext` wrapping `zwp_text_input_v3`:

```c
/* gdk/wayland/gdkwaylandinputcontext.c (simplified) */
static void
gdk_wayland_input_context_focus_in(GtkIMContext *context,
    GdkSurface *surface)
{
    GdkWaylandInputContext *wic = GDK_WAYLAND_INPUT_CONTEXT(context);
    zwp_text_input_v3_enable(wic->text_input);
    zwp_text_input_v3_set_cursor_rectangle(wic->text_input,
        rect.x, rect.y, rect.width, rect.height);
    zwp_text_input_v3_commit(wic->text_input);
}
```

### Qt6 Text Input

Qt6's Wayland platform plugin (`qtwayland/src/client/qwaylandinputcontext.cpp`) implements `QWaylandTextInputMethod`:

```cpp
class QWaylandTextInputMethod : public QInputMethod
{
    void update(Qt::InputMethodQueries queries) override {
        if (queries & Qt::ImCursorRectangle) {
            QRect rect = inputItem()->inputMethodQuery(Qt::ImCursorRectangle).toRect();
            mTextInput->setCursorRectangle(rect.x(), rect.y(),
                                           rect.width(), rect.height());
        }
        if (queries & Qt::ImSurroundingText) {
            QString text = inputItem()->inputMethodQuery(Qt::ImSurroundingText).toString();
            int cursor = inputItem()->inputMethodQuery(Qt::ImCursorPosition).toInt();
            mTextInput->setSurroundingText(text, cursor, cursor);
        }
        mTextInput->commit();
    }
};
```

---

## fcitx5 and IBus on Wayland

### fcitx5 Native Wayland

fcitx5 natively supports `zwp_input_method_v2`:

```bash
# Install and configure:
pacman -S fcitx5 fcitx5-chinese-addons fcitx5-gtk fcitx5-qt
export INPUT_METHOD=fcitx5
export GTK_IM_MODULE=wayland   # GTK4: use Wayland protocol directly
export QT_IM_MODULE=fcitx5
export XMODIFIERS=@im=fcitx5   # XWayland fallback
```

fcitx5's Wayland module (`src/frontend/waylandim/`):

```cpp
/* fcitx5/src/frontend/waylandim/waylandimserver.cpp */
class WaylandIMInputMethodV2 : public zwp_input_method_v2 {
    void activate() override { engine_->activate(); }

    void surrounding_text(const std::string &text,
        uint32_t cursor, uint32_t anchor) override {
        engine_->setSurroundingText(text, cursor, anchor);
    }

    void key(uint32_t serial, uint32_t time, uint32_t sym, bool pressed) override {
        auto result = engine_->processKey(sym, pressed);
        if (result.commit)          im->commit_string(result.committed);
        if (result.preedit_changed) im->set_preedit_string(result.preedit, result.cursor, result.cursor);
        im->commit();
    }
};
```

### Pinyin Engine Architecture

```
User types: "n i h a o"
    │ (raw keysyms via zwp_input_method_v2)
    ▼
fcitx5 Pinyin engine
    ├─ syllable segmenter: ["ni", "hao"]
    ├─ candidate generator: [你好, 拟好, 尼耗, ...]
    └─ user dictionary / ML ranking
    │
    ▼
preedit_string("nǐhǎo", cursor)  → sent to compositor
    ↓
user selects "你好"
    ↓
commit_string("你好") → app inserts text
```

### IBus on Wayland

IBus supports Wayland via two mechanisms:
1. **GNOME integration**: IBus communicates with Mutter via D-Bus
2. **Direct `zwp_input_method_v2`** (ibus-wayland daemon, since IBus 1.5.27)

```bash
export INPUT_METHOD=ibus
ibus-daemon -drxR
IBUS_DEBUG=1 ibus-daemon -drxR 2>&1 | head -50
```

---

## XWayland Text Input Bridge

### The XWayland IME Problem

X11 apps running under XWayland use XIM. XWayland bridges:

```
X11 app (XIM client) ↔ XIM server (in XWayland) ↔ zwp_text_input_v3 ↔ Compositor ↔ IME
```

XWayland implements a built-in XIM server that forwards to `zwp_text_input_v3`:

```
X11 app sets XIC → XWayland internal XIM server
  → zwp_text_input_v3 (XWayland acts as Wayland application)
  → Compositor → IME
  → commit text back via zwp_text_input_v3
  → XWayland synthesises XIM committed text event
  → X11 app receives text
```

### XIM Filter Events

```c
/* Legacy X11 app: */
while (XNextEvent(display, &event)) {
    if (XFilterEvent(&event, None))
        continue;  /* IME consumed this event */
    switch (event.type) {
    case KeyPress:
        XLookupString(&event.xkey, buf, sizeof(buf), &keysym, &status);
        break;
    }
}
```

GTK4 apps always use the native Wayland protocol even under XWayland (`GDK_BACKEND=wayland`), bypassing the XIM bridge entirely.

---

## Integrations

- **Ch34 (Wayland Core)** — `zwp_text_input_v3` follows the same `wl_registry`/listener pattern as all Wayland extensions
- **Ch35 (Compositors: Mutter/KWin)** — GNOME and KDE implement the compositor side of `zwp_text_input_v3`; GNOME routes to IBus via D-Bus rather than `zwp_input_method_v2`
- **Ch36 (wlroots)** — wlroots `wlr_text_input_v3` and `wlr_input_method_v2` are the reference implementations used by Sway, Hyprland, labwc
- **Ch123 (XWayland)** — XWayland's XIM bridge uses `zwp_text_input_v3` as the Wayland-side transport for X11 app text input
- **Ch39 (Wayland Input)** — `wl_keyboard.key` events are routed by the compositor to the IME process via `zwp_input_method_v2` before forwarding to applications
