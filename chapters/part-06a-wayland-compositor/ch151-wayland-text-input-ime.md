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
10. [text-input-unstable-v3 Protocol Deep Dive](#text-input-unstable-v3-protocol-deep-dive)
11. [input-method-unstable-v2 Deep Dive](#input-method-unstable-v2-deep-dive)
12. [Fcitx5 Architecture on Wayland](#fcitx5-architecture-on-wayland)
13. [IBus on Wayland](#ibus-on-wayland)
14. [GTK4 and Qt6 IME Integration](#gtk4-and-qt6-ime-integration)
15. [Popup and Candidate Window Rendering](#popup-and-candidate-window-rendering)
16. [Legacy X11 XIM under XWayland](#legacy-x11-xim-under-xwayland)
17. [Emoji Input and Unicode](#emoji-input-and-unicode)
18. [Integrations](#integrations)

---

## Introduction

Text input on Wayland is significantly more complex than on X11, where XIM (X Input Method) was a well-established if imperfect solution. Wayland's security model — no global input snooping — means the input method (IME) and the application communicate only through the compositor, which mediates all focus and event routing.

This chapter covers the two Wayland unstable protocols that implement text input:

- **`zwp_text_input_v3`** — the application side
- **`zwp_input_method_v2`** — the IME side
- **virtual keyboard protocol** — for on-screen keyboards

It then traces how the following implement these protocols:

- **GTK4** — Wayland text input integration
- **Qt6** — Wayland text input integration
- **fcitx5** — native IME via `zwp_input_method_v2`
- **IBus** — IME with Wayland and D-Bus bridge support
- **XWayland** — bridges text input back to X11 apps

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

## text-input-unstable-v3 Protocol Deep Dive

Building on the overview in §3, this section dissects the full `zwp_text_input_v3` state machine, the exact semantics of each event and request, the rationale for the "unstable" designation, and the v4 work in progress.

### Full State Machine: enable → surrounding_text → content_type → commit

The application side of text input is double-buffered: every update the client wishes to make must first be staged into "pending" state, then flushed atomically with a `commit()` call. The compositor likewise batches its responses behind a `done(serial)` event. This double-buffering is what keeps the protocol race-free when cursor position and IME state change simultaneously.

The canonical lifecycle for a single focus event is:

```
[surface receives keyboard focus]
    │
    ▼
client calls enable()
    │   (resets all pending state to empty)
    ▼
client calls set_surrounding_text(text, cursor, anchor)
    │   (up to 4000 UTF-8 bytes around the cursor; cursor/anchor are byte offsets)
    ▼
client calls set_content_type(hint_bitfield, purpose_enum)
    │   (hint: completion|spellcheck|auto_capitalization|hidden_text|…)
    │   (purpose: normal|alpha|digits|number|phone|url|email|password|terminal|…)
    ▼
client calls set_cursor_rectangle(x, y, width, height)
    │   (surface-local coordinates, used to place the IME popup)
    ▼
client calls set_text_change_cause(cause)
    │   (cause: input_method=0 or other=1; tells compositor who changed surrounding text)
    ▼
client calls commit()                    ← serial counter increments
    │
    ▼
compositor receives state, notifies IME via zwp_input_method_v2
    │
    ▼                        (compositor → client, batched:)
compositor sends preedit_string(text, cursor_begin, cursor_end)
compositor sends commit_string(text)
compositor sends delete_surrounding_text(before_length, after_length)
    │
    ▼
compositor sends done(serial)
    │
    ▼
client applies events in order:
  1. Replace current preedit with new preedit (may be null)
  2. Execute delete_surrounding_text
  3. Insert commit_string at cursor
  4. Re-compute surrounding text
  5. Set new preedit at cursor position
```

[Source: wayland-protocols text-input-unstable-v3 specification, wayland.app](https://wayland.app/protocols/text-input-unstable-v3)

### The preedit_string Event and cursor_begin/cursor_end

The `preedit_string` event carries three fields:

- `text` (nullable string): The pre-edit composition text. `null` means clear preedit.
- `cursor_begin` (int): Byte offset within `text` where the preedit cursor underline begins. `-1` hides the cursor entirely.
- `cursor_end` (int): Byte offset within `text` where the preedit cursor underline ends.

When `cursor_begin == cursor_end`, the cursor is a caret (no selection highlight). When they differ, the highlighted range indicates the active composition segment — common in Japanese IME where a long kana string is divided into sub-segments, each of which can be converted independently.

```c
/* Toolkit handler for preedit_string (simplified GTK4-style): */
static void
handle_preedit_string(void *data,
                      struct zwp_text_input_v3 *ti,
                      const char *text,
                      int32_t cursor_begin,
                      int32_t cursor_end)
{
    struct EditorState *ed = data;

    /* Clear any existing preedit underline render: */
    editor_clear_preedit(ed);

    if (text) {
        /* Draw the preedit string with underline.
         * cursor_begin/cursor_end in bytes → convert to char offsets for Pango: */
        int preedit_len = strlen(text);
        PangoAttribute *attr = pango_attr_underline_new(PANGO_UNDERLINE_SINGLE);
        attr->start_index = 0;
        attr->end_index   = preedit_len;
        editor_set_preedit(ed, text, attr, cursor_begin, cursor_end);
    }
    /* Do NOT apply yet — wait for done() event */
}
```

### The commit_string Event

`commit_string` carries the final composed text that should be inserted at the cursor position, replacing any current preedit. It arrives before `done()`. When both `commit_string` and `delete_surrounding_text` arrive in the same `done()` batch, the deletion must be applied first, then the insertion.

```c
static void
handle_commit_string(void *data,
                     struct zwp_text_input_v3 *ti,
                     const char *text)
{
    struct EditorState *ed = data;
    if (text)
        ed->pending_commit = strdup(text);  /* apply on done() */
}

static void
handle_done(void *data, struct zwp_text_input_v3 *ti, uint32_t serial)
{
    struct EditorState *ed = data;
    /* Apply in the specified order: */
    if (ed->pending_delete_before || ed->pending_delete_after)
        editor_delete_surrounding(ed,
            ed->pending_delete_before, ed->pending_delete_after);
    if (ed->pending_commit) {
        editor_insert(ed, ed->pending_commit);
        free(ed->pending_commit);
        ed->pending_commit = NULL;
    }
    editor_apply_preedit(ed);
    ed->last_done_serial = serial;
}
```

### The delete_surrounding_text Request

`delete_surrounding_text` asks the application to delete `before_length` bytes before the cursor and `after_length` bytes after the cursor, measured in UTF-8 bytes, **excluding** any current preedit text. This is used by predictive engines that need to backspace over a mis-typed word before inserting the corrected version.

### Why the Protocol Is "Unstable" — and v4 in Progress

The `zwp_` prefix and "unstable" tag mean backward-incompatible changes are permitted at any time. `text-input-v3` carries this designation because of known gaps identified during the 2021 Qt Contributor Summit review:

| Missing Feature | Impact |
|---|---|
| No text direction hint | Cursor placement in RTL fields is compositor-guessed |
| No language tag | IME cannot send the BCP-47 language to enable correct glyph shaping |
| No preedit styling | Cannot specify colour, bold, or double-underline for active segment |
| No raw key-code forwarding | Terminal emulators cannot reconstruct raw keystrokes from commit_string |
| No virtual keyboard rectangle | OSK (on-screen keyboard) cannot communicate its own size to avoid overlap |

[Source: QtCS2021 — Wayland text-input-unstable-v4 protocol, Qt Wiki](https://wiki.qt.io/QtCS2021_-_Wayland_text-input-unstable-v4_protocol)

Qt has shipped an internal `qt_text_input_method_v1` protocol extension to work around these gaps, and a `text-input-unstable-v4` work-in-progress XML exists in the Qt Wayland compositor attribution files. As of 2026 the upstream `wayland-protocols` repository has not yet promoted text-input to a stable `ext_` protocol; the community expectation is that a stable `ext_text_input_v1` will be finalised when the v4 feature set is agreed upon.

---

## input-method-unstable-v2 Deep Dive

Building on the overview in §4, this section covers the compositor-side interface that IME processes implement, focusing on keyboard grab mechanics and the popup surface positioning model.

### The Full zwp_input_method_v2 Interface

The IME process connects to the compositor and creates a `zwp_input_method_v2` object via the `zwp_input_method_manager_v2` global. Only one input method per seat is permitted; a second connection causes the first to receive `unavailable` and become non-functional.

```xml
<!-- input-method-unstable-v2.xml (key interfaces excerpt) -->

<interface name="zwp_input_method_manager_v2" version="1">
  <!-- Factory: claim the input method role on a seat -->
  <request name="get_input_method">
    <arg name="seat" type="object" interface="wl_seat"/>
    <arg name="input_method" type="new_id" interface="zwp_input_method_v2"/>
  </request>
  <request name="destroy"/>
</interface>

<interface name="zwp_input_method_v2" version="1">
  <!-- Events from compositor to IME: -->
  <event name="activate"/>        <!-- text input entered a field -->
  <event name="deactivate"/>      <!-- text input left all fields -->
  <event name="surrounding_text">
    <arg name="text"   type="string"/>
    <arg name="cursor" type="uint"/>
    <arg name="anchor" type="uint"/>
  </event>
  <event name="text_change_cause">
    <arg name="cause" type="uint"/>  <!-- change_cause enum -->
  </event>
  <event name="content_type">
    <arg name="hint"    type="uint"/>  <!-- content_hint bitfield -->
    <arg name="purpose" type="uint"/>  <!-- content_purpose enum -->
  </event>
  <event name="done"/>            <!-- apply all pending events atomically -->
  <event name="unavailable"/>     <!-- another IM took the seat -->

  <!-- Requests from IME to compositor: -->
  <request name="commit_string">
    <arg name="text" type="string"/>
  </request>
  <request name="set_preedit_string">
    <arg name="text"         type="string"/>
    <arg name="cursor_begin" type="int"/>
    <arg name="cursor_end"   type="int"/>
  </request>
  <request name="delete_surrounding_text">
    <arg name="before_length" type="uint"/>
    <arg name="after_length"  type="uint"/>
  </request>
  <request name="commit">
    <arg name="serial" type="uint"/>  <!-- must echo the serial from done() -->
  </request>
  <request name="get_input_popup_surface">
    <arg name="id"      type="new_id" interface="zwp_input_popup_surface_v2"/>
    <arg name="surface" type="object" interface="wl_surface"/>
  </request>
  <request name="grab_keyboard">
    <arg name="keyboard" type="new_id" interface="zwp_input_method_keyboard_grab_v2"/>
  </request>
  <request name="destroy"/>
</interface>
```

[Source: input-method-unstable-v2 protocol, wayland.app](https://wayland.app/protocols/input-method-unstable-v2)

### grab_keyboard and zwp_input_method_keyboard_grab_v2

The `grab_keyboard` request is the mechanism by which an IME intercepts hardware key events before the focused application sees them. The compositor returns a `zwp_input_method_keyboard_grab_v2` object that delivers keyboard events exclusively to the IME:

```xml
<interface name="zwp_input_method_keyboard_grab_v2" version="1">
  <!-- Same events as wl_keyboard, but delivered only to the IME: -->
  <event name="keymap">
    <arg name="format" type="uint"/>  <!-- wl_keyboard.keymap_format enum -->
    <arg name="fd"     type="fd"/>    <!-- shared memory fd with XKB keymap -->
    <arg name="size"   type="uint"/>
  </event>
  <event name="key">
    <arg name="serial" type="uint"/>
    <arg name="time"   type="uint"/>
    <arg name="key"    type="uint"/>  <!-- raw evdev key code (KEY_A=30, etc.) -->
    <arg name="state"  type="uint"/>  <!-- WL_KEYBOARD_KEY_STATE_PRESSED/RELEASED -->
  </event>
  <event name="modifiers">
    <arg name="serial"         type="uint"/>
    <arg name="mods_depressed" type="uint"/>
    <arg name="mods_latched"   type="uint"/>
    <arg name="mods_locked"    type="uint"/>
    <arg name="group"          type="uint"/>
  </event>
  <event name="repeat_info">
    <arg name="rate"  type="int"/>  <!-- key repeat rate in Hz; 0 = no repeat -->
    <arg name="delay" type="int"/>  <!-- delay in ms before repeat starts -->
  </event>
  <request name="release"/>  <!-- IME releases the grab -->
</interface>
```

The critical design point: when the IME holds a `zwp_input_method_keyboard_grab_v2`, the compositor does **not** forward `wl_keyboard.key` events to the focused application. This gives the IME first opportunity to process each keystroke. If the IME decides it does not want to consume the key (e.g. a cursor movement key in a non-composing state), it must forward the key to the application via the `zwp_virtual_keyboard_v1` protocol.

```
Key pressed on hardware keyboard
  │
  ▼
Compositor
  │  (IME has grab)
  ├──→ zwp_input_method_keyboard_grab_v2.key(serial, time, keycode, state)
  │      → IME processes key
  │      → If consumed: IME sends commit_string / set_preedit_string / commit()
  │      → If not consumed: IME calls zwp_virtual_keyboard_v1.key(time, keycode, state)
  │                               └──→ Compositor re-delivers as wl_keyboard.key to app
  └──→ (wl_keyboard.key NOT sent to app while grab is held)
```

This architecture is far safer than X11's global keyboard grab: only one IME per seat can hold the grab, the compositor mediates all key delivery, and regular applications cannot snoop on keys destined for the IME.

---

## Fcitx5 Architecture on Wayland

### D-Bus Activation and Session Startup

Fcitx5 starts as a D-Bus service under the name `org.fcitx.Fcitx5`. On a systemd-based session the canonical startup is via XDG autostart (`~/.config/autostart/fcitx5.desktop`) or by embedding it in the compositor launch script. On GNOME, fcitx5 replaces the existing `ibus-daemon` on autostart because both claim the same IBus D-Bus protocol name.

```bash
# Verify fcitx5 owns the D-Bus name:
busctl --user list | grep fcitx
# org.fcitx.Fcitx5   <pid>   fcitx5   jreuben1   :1.42

# Required environment variables for a Wayland session:
export XMODIFIERS=@im=fcitx5      # XWayland and legacy X11 apps
# GTK_IM_MODULE should NOT be set for GTK4 — GTK4 uses text-input-v3 natively
# For GTK3 on Wayland if needed:
# export GTK_IM_MODULE=fcitx
export QT_IM_MODULES="wayland;fcitx"   # Qt 6.7+; tries wayland first, fcitx as fallback
```

[Source: Using Fcitx 5 on Wayland, fcitx-im.org](https://fcitx-im.org/wiki/Using_Fcitx_5_on_Wayland)

### The fcitx5-wayland-im Backend

The Wayland frontend of fcitx5 lives in `src/frontend/waylandim/`. It creates two server instances per Wayland display connection: `WaylandIMServer` (v1 protocol, for older compositors) and `WaylandIMServerV2` (v2 protocol, for Sway, Hyprland, KDE, etc.). The class hierarchy is:

```
WaylandIMServerBase (shared XKB state, virtual keyboard management)
  ├── WaylandIMServer        (zwp_input_method_v1)
  │     └── WaylandIMInputContextV1
  └── WaylandIMServerV2      (zwp_input_method_v2)
        └── WaylandIMInputContextV2  ← extends InputContext
```

[Source: fcitx5 src/frontend/waylandim/waylandimserverv2.cpp](https://github.com/fcitx/fcitx5/blob/9becc0f5e1a6fbde2ff874fe4291f8dbb741798f/src/frontend/waylandim/waylandimserverv2.cpp)

### Key Event Interception

When fcitx5 activates (receives the `activate` event from the compositor), it calls `grab_keyboard()` immediately. The grab connects signal handlers:

```cpp
/* fcitx5/src/frontend/waylandim/waylandimserverv2.cpp, ~lines 238–265 */
void WaylandIMInputContextV2::activate(uint32_t serial) {
    serial_ = serial;
    /* Establish hardware keyboard grab: */
    keyboardGrab_.reset(ic_->grabKeyboard());
    if (!keyboardGrab_) {
        WAYLANDIM_DEBUG() << "Failed to grab keyboard";
        return;
    }
    keyboardGrab_->keymap().connect([this](uint32_t format, int32_t fd, uint32_t size) {
        keymapCallback(format, fd, size);
    });
    keyboardGrab_->key().connect([this](uint32_t serial, uint32_t time,
                                        uint32_t key, uint32_t state) {
        keyCallback(serial, time, key, state);
    });
    keyboardGrab_->modifiers().connect([this](uint32_t serial,
        uint32_t mods_dep, uint32_t mods_lat, uint32_t mods_lock, uint32_t group) {
        modifiersCallback(serial, mods_dep, mods_lat, mods_lock, group);
    });
    keyboardGrab_->repeatInfo().connect([this](int32_t rate, int32_t delay) {
        repeatInfoCallback(rate, delay);
    });
}
```

Key events arrive in `keyCallback()`, where they are translated to XKB symbols and dispatched into the fcitx5 input engine:

```cpp
/* ~lines 366–391 */
void WaylandIMInputContextV2::keyCallback(uint32_t /*serial*/,
    uint32_t time, uint32_t key, uint32_t state)
{
    uint32_t code = key + 8;  /* evdev → X11 keycode offset */
    KeySym sym = static_cast<KeySym>(
        xkb_state_key_get_one_sym(server_->state_.get(), code));

    KeyEvent event(this,
        Key(sym, server_->modifiers_, code),
        state == WL_KEYBOARD_KEY_STATE_RELEASED,
        time);

    if (!keyEvent(event)) {
        /* Not consumed by IME — forward to application via virtual keyboard */
        vk_->key(time,
                 event.rawKey().code() - 8,
                 event.isRelease() ? WL_KEYBOARD_KEY_STATE_RELEASED
                                   : WL_KEYBOARD_KEY_STATE_PRESSED);
    }
}
```

### Preedit and Commit Delivery

When the input engine updates the preedit string (e.g. after each pinyin syllable), `updatePreeditImpl()` forwards it to the compositor via `zwp_input_method_v2`:

```cpp
/* ~lines 448–456 */
void WaylandIMInputContextV2::updatePreeditImpl() {
    auto preedit = server_->instance()->outputFilter(
        this, inputPanel().clientPreedit());
    ic_->setPreeditString(preedit.toString().data(),
                          preedit.cursor(),
                          preedit.cursor());
    ic_->commit(serial_);
    /* The compositor then forwards preedit_string to the focused application */
}
```

When the user selects a candidate (e.g. presses Space to confirm pinyin → Chinese), the engine calls `commitStringImpl()`, which sends `commit_string` followed by `commit()`. The compositor routes the committed text to the focused application's `zwp_text_input_v3.commit_string` event.

### Candidate Window Rendering

Wayland lacks a global coordinate system, making IME popup placement non-trivial. Fcitx5 uses multiple strategies depending on compositor support:

1. **`zwp_input_method_popup_surface_v2`** (preferred): The IME creates a popup surface via `get_input_popup_surface()`. The compositor positions it using the `text_input_rectangle` event (which carries the application cursor's screen rectangle). The IME renders its candidate list into this surface using its own rendering backend (fcitx5-classic-ui uses Cairo/Pango for text rendering).

2. **Client-side input panel via GTK/Qt IM module**: For Gtk4/Qt6 apps, the fcitx IM module running inside the application process creates an `xdg_popup` anchored to the cursor position and renders the candidate list there. This is the "client-side input panel" approach. [Source: fcitx5 discussions/895](https://github.com/fcitx/fcitx5/discussions/895)

3. **XWayland fallback**: If neither of the above works, fcitx5 falls back to opening an X11 window via XWayland for the candidate popup. This always has correct global coordinates but requires XWayland to be running.

---

## IBus on Wayland

### Protocol Version Support

IBus's Wayland support has evolved across versions:

| IBus version | Wayland support |
|---|---|
| ≤ 1.5.26 | GNOME only (D-Bus to Mutter) |
| 1.5.29 | `zwp_input_method_v1` (KDE Plasma, Weston) |
| 1.5.32+ | `zwp_input_method_v2` (Sway, Hyprland, COSMIC, Labwc, niri) |

[Source: IBus WaylandDesktop wiki, github.com/ibus/ibus](https://github.com/ibus/ibus/wiki/WaylandDesktop)

### GNOME Integration: D-Bus Bridge to Mutter

On GNOME, IBus does not use `zwp_input_method_v2` at all. Instead, `gnome-shell` launches `ibus-daemon` with the `--panel disable` flag and communicates with it exclusively over D-Bus. Mutter acts as the Wayland compositor and simultaneously as the IBus bus client; it bridges between the `zwp_text_input_v3` events from applications and the IBus engine D-Bus API:

```
GTK4 app                  Mutter (GNOME compositor)          ibus-daemon
  │                             │                                │
  │ zwp_text_input_v3.enable()  │                                │
  ├────────────────────────────►│                                │
  │                             │  D-Bus: IBus.InputContext      │
  │                             │  .FocusIn()                    │
  │                             ├───────────────────────────────►│
  │                             │                                │
  │                             │  D-Bus: CommitText("你好")     │
  │                             │◄───────────────────────────────┤
  │ zwp_text_input_v3           │                                │
  │ .commit_string("你好")      │                                │
  │◄────────────────────────────┤                                │
```

This architecture means that on GNOME, any IBus engine (ibus-pinyin, ibus-hangul, ibus-m17n) works regardless of whether it implements `zwp_input_method_v2` itself.

### Direct Wayland Mode (IBus 1.5.32+)

On compositors that support `zwp_input_method_v2` directly (Sway, Hyprland, KDE Plasma in non-IBus-D-Bus mode), IBus can be started with:

```bash
ibus start --type wayland   # uses zwp_input_method_v2
# or: ibus-daemon --type wayland -drxR
```

The `ibus-wayland` executable (installed to `$libexecdir/ibus-wayland`) implements `zwp_input_method_v2` and connects the IBus engine framework to the Wayland input method protocol directly. [Source: ibus/releases, github.com/ibus/ibus](https://github.com/ibus/ibus/releases)

### Known Gaps and Limitations

**Preedit styling**: The `zwp_input_method_v2` protocol dropped support for customising preedit colours that were present in the original `text-input-v1` protocol. IBus engines that previously displayed the active composition segment in a different colour lose this capability on Wayland.

**Compositor coverage**: Not all Wayland compositors support `zwp_input_method_v2`. LXQt's Miriway compositor is noted as incompatible. Users must fall back to the Labwc compositor for IBus support in that environment.

**Budgie**: Ships `QT_IM_MODULE=ibus` (an X11 convention), requiring manual override to `QT_IM_MODULES=wayland;ibus` for correct Wayland-native Qt6 operation.

**GTK4 fallback**: GTK4 prefers its built-in `text-input-v3` Wayland protocol path over any IM module. If `GTK_IM_MODULE=ibus` is set, GTK4 uses the ibus GTK4 module (`ibus-gtk4`) which communicates via IBus D-Bus rather than the Wayland text input protocol. To use the pure Wayland path in GTK4, `GTK_IM_MODULE` must be unset. [Source: Red Hat Bugzilla #2054680](https://bugzilla.redhat.com/show_bug.cgi?id=2054680)

**Qt6 < 6.7**: Older Qt6 releases only support Qt's internal `qt_text_input_method_v1` Wayland extension (supported only by KWin), plus the legacy `QT_IM_MODULE` environment variable for fallback to IM modules. Qt 6.7 introduced `QT_IM_MODULES` (plural) enabling ordered fallback: `QT_IM_MODULES="wayland;ibus"` tries the native Wayland text-input protocol first, then the IBus IM module.

---

## GTK4 and Qt6 IME Integration

### GTK4: The im-wayland Input Context

GTK4's Wayland backend implements `GdkWaylandSeat` which creates and manages a `zwp_text_input_v3` object per seat. When a `GtkText` or `GtkTextView` widget receives focus, GTK calls into the input method context chain, which ends at the Wayland input context implementation in `gdk/wayland/`.

The focus-in flow:

```c
/* gdk/wayland/ — conceptual flow when a GtkEntry gains focus */

/* 1. GTK widget signals GDK that a text input field has focus: */
gdk_wayland_device_set_focus(device, surface);

/* 2. GDK enables the text input object for this surface: */
zwp_text_input_v3_enable(seat->text_input);

/* 3. Send current field geometry so IME can position its popup: */
GdkRectangle rect = { cursor_x, cursor_y, 1, line_height };
zwp_text_input_v3_set_cursor_rectangle(seat->text_input,
    rect.x, rect.y, rect.width, rect.height);

/* 4. Send surrounding text for context (e.g. for autocorrect): */
zwp_text_input_v3_set_surrounding_text(seat->text_input,
    surrounding_buf, cursor_byte_offset, anchor_byte_offset);

/* 5. Signal the input field type: */
zwp_text_input_v3_set_content_type(seat->text_input,
    ZWP_TEXT_INPUT_V3_CONTENT_HINT_AUTO_CAPITALIZATION,
    ZWP_TEXT_INPUT_V3_CONTENT_PURPOSE_NORMAL);

/* 6. Commit the pending state: */
zwp_text_input_v3_commit(seat->text_input);
```

On `preedit_string`, GTK4 creates a Pango attribute list with an underline span covering the preedit range, and inserts it into the widget's text buffer at the cursor position as a temporary "pre-edit" run. On `commit_string`, the preedit run is replaced by the final text and the cursor advances.

GTK4 does not consult `GTK_IM_MODULE` on Wayland — the Wayland text-input path is built into GDK directly and is always active. Setting `GTK_IM_MODULE=ibus` forces the legacy ibus-gtk4 module path, which uses IBus D-Bus rather than the Wayland protocol; this is generally not recommended for native Wayland GTK4 apps.

### Qt6: QWaylandTextInputV3

Qt6's Wayland platform plugin (`qtbase/src/plugins/platforms/wayland/`) implements `QWaylandTextInputV3` which wraps `zwp_text_input_v3`. The Qt6 integration works through the `QInputMethod` abstract class, which all Qt text-accepting widgets communicate with:

```cpp
/* qtbase/src/plugins/platforms/wayland/qwaylandtextinputv3.cpp (simplified) */

void QWaylandTextInputV3::update(Qt::InputMethodQueries queries)
{
    if (queries & Qt::ImEnabled) {
        bool enabled = inputItem() && inputItem()->flags() & Qt::ItemAcceptsInputMethod;
        if (enabled != mEnabled) {
            mEnabled = enabled;
            if (enabled)
                mTextInput->enable();   /* zwp_text_input_v3_enable() */
            else
                mTextInput->disable();  /* zwp_text_input_v3_disable() */
        }
    }
    if (queries & Qt::ImCursorRectangle) {
        QRect r = inputItem()->inputMethodQuery(Qt::ImCursorRectangle).toRect();
        mTextInput->set_cursor_rectangle(r.x(), r.y(), r.width(), r.height());
    }
    if (queries & Qt::ImSurroundingText) {
        QString text = inputItem()->inputMethodQuery(Qt::ImSurroundingText).toString();
        int cursor   = inputItem()->inputMethodQuery(Qt::ImCursorPosition).toInt();
        int anchor   = inputItem()->inputMethodQuery(Qt::ImAnchorPosition).toInt();
        mTextInput->set_surrounding_text(text.toUtf8(), cursor, anchor);
    }
    mTextInput->commit();  /* zwp_text_input_v3_commit() */
}
```

[Source: Qt6 qtwayland source tree, codebrowser.dev](https://codebrowser.dev/qt6/qtbase/src/plugins/platforms/wayland/qwaylandtextinputv3.cpp.html)

On receiving `preedit_string`, Qt6 emits a `QInputMethodEvent` with a `PreeditText` attribute list. On `commit_string` it emits a `QInputMethodEvent` with `CommitString` set, which the receiving `QLineEdit` or `QPlainTextEdit` appends to the document.

### Testing CJK IME: Korean, Japanese, Chinese

**Korean (Hangul)**: Korean composing is unique in that a single syllable block (e.g. `한`) is built incrementally from consonant + vowel strokes. The preedit string shows the in-progress syllable block; pressing another consonant that would start a new syllable causes the current block to be committed and a new preedit to start. This is sometimes called "on-the-spot" composition and is well-tested with fcitx5-hangul and ibus-hangul.

**Japanese (Hiragana → Kanji)**: Japanese input uses romanised kana (romaji → hiragana) followed by kanji conversion triggered by Space. The preedit carries the full hiragana string; after Space the candidate window appears; Enter or a digit key selects the candidate and fires `commit_string`. Multiple conversion segments can be active simultaneously, each highlighted with `cursor_begin`/`cursor_end`.

**Chinese Simplified (Pinyin)**: As illustrated in §8's pinyin engine diagram, each typed syllable extends the preedit; Space triggers candidate selection; the selected Chinese characters are sent via `commit_string`. The number of characters in `commit_string` is often longer than the preedit (pinyin → CJK expansion).

```bash
# Quick test: start a text field in a GTK4 app with debug logging
GTK_DEBUG=text-input-v3 gedit &
# Check that enable/commit/preedit events are visible in stderr
```

---

## Popup and Candidate Window Rendering

### The zwp_input_popup_surface_v2 Positioning Model

The standard mechanism for IME candidate windows is `zwp_input_popup_surface_v2`. The IME creates a `wl_surface`, attaches a pixel buffer to it, and then calls `get_input_popup_surface()` to give it the input method popup surface role:

```c
/* IME candidate window lifecycle using zwp_input_method_v2 */

/* 1. Create a bare wl_surface: */
struct wl_surface *popup_surface = wl_compositor_create_surface(compositor);

/* 2. Assign it the input popup role via the active input method object: */
struct zwp_input_popup_surface_v2 *popup =
    zwp_input_method_v2_get_input_popup_surface(im, popup_surface);

/* 3. Register for the text_input_rectangle event: */
static const struct zwp_input_popup_surface_v2_listener popup_listener = {
    .text_input_rectangle = on_text_input_rectangle,
};
zwp_input_popup_surface_v2_add_listener(popup, &popup_listener, state);

/* 4. In the text_input_rectangle callback, re-position the popup: */
static void on_text_input_rectangle(void *data,
    struct zwp_input_popup_surface_v2 *popup,
    int32_t x, int32_t y, int32_t width, int32_t height)
{
    /* x,y,width,height = cursor position in compositor global coordinates */
    /* Render candidate list below this rectangle: */
    struct IMEState *s = data;
    s->popup_x = x;
    s->popup_y = y + height;  /* place popup just below cursor */
    render_candidate_list(s);
    /* Attach the rendered buffer and commit: */
    wl_surface_attach(popup_surface, s->candidate_buffer, 0, 0);
    wl_surface_commit(popup_surface);
}
```

[Source: input-method-unstable-v2, wayland.app](https://wayland.app/protocols/input-method-unstable-v2)

The key distinction from `xdg_popup`: `zwp_input_popup_surface_v2` does **not** use an `xdg_positioner`. The compositor provides the screen-space cursor rectangle via `text_input_rectangle`, and the IME applies its own offset logic. The compositor decides the ultimate position (e.g. clipping to screen edges, handling multi-monitor setups). The IME cannot move a `zwp_input_popup_surface_v2` surface directly — it must wait for `text_input_rectangle` updates.

### GPU Rendering of Candidate Lists

Each candidate entry in a CJK IME typically consists of a numbered glyph label and a CJK character string. The rendering stack for fcitx5's classic-ui:

1. **Pango layout**: The candidate text is measured and rendered using Pango with the system font. The result is a `cairo_surface_t` backed by a shared memory buffer (`wl_shm`).
2. **Cairo drawing**: Background fill, border, and candidate text are composited with Cairo onto the shared memory buffer.
3. **wl_buffer**: The `cairo_surface_t` data is wrapped in a `wl_shm_pool` / `wl_buffer` and attached to the popup surface.

For higher-performance IMEs (or when Vulkan-rendered desktops require GPU compositing), some IME frontends (e.g. fcitx5-kde's KDE integration) use Qt's rendering pipeline, which can use the GPU via `eglCreateWindowSurface` for the popup.

### xdg_popup Alternative for Client-Side Input Panel

When the IME uses the "client-side input panel" approach (see §12, Fcitx5 candidate window strategy 2), the IM module inside the application creates an `xdg_popup` surface:

```c
/* Client-side input panel: IM module inside the GTK/Qt app creates an xdg_popup */
struct xdg_positioner *pos = xdg_wm_base_create_positioner(wm_base);

/* Anchor the popup to the cursor rectangle: */
xdg_positioner_set_anchor_rect(pos, cursor_x, cursor_y, 1, cursor_height);
xdg_positioner_set_anchor(pos, XDG_POSITIONER_ANCHOR_BOTTOM_LEFT);
xdg_positioner_set_gravity(pos, XDG_POSITIONER_GRAVITY_BOTTOM_RIGHT);
xdg_positioner_set_constraint_adjustment(pos,
    XDG_POSITIONER_CONSTRAINT_ADJUSTMENT_SLIDE_X |
    XDG_POSITIONER_CONSTRAINT_ADJUSTMENT_FLIP_Y);  /* flip above cursor if no room below */
xdg_positioner_set_size(pos, candidate_width, candidate_height);

struct xdg_popup *popup = xdg_surface_get_popup(
    candidate_xdg_surface, parent_xdg_surface, pos);
```

The limitation of this approach (noted in §12) is that `xdg_popup` repositioning requires protocol support not present in GTK3 or Qt5; those toolkits cannot move a visible popup. GTK4 and Qt6 support popup repositioning via `xdg_popup.reposition()` (added in xdg-shell version 3).

---

## Legacy X11 XIM under XWayland

### The XWayland IME Architecture

X11 applications running under XWayland use the X Input Method (XIM) protocol — a 1990s-era protocol that predates Wayland by two decades. XWayland implements a built-in XIM server that bridges to the Wayland compositor's `zwp_text_input_v3`:

```
X11 application (Xlib/XCB + XIM client)
    │  XIM protocol over X11 connection
    ▼
XWayland's built-in XIM server
    │  (XWayland acts as both X11 server and Wayland client)
    │
    ▼
zwp_text_input_v3 (XWayland → Wayland compositor)
    │
    ▼
Wayland compositor → zwp_input_method_v2 → fcitx5 / IBus
    │
    ▼
commit_string / preedit_string → zwp_text_input_v3 → XWayland
    │
    ▼
XWayland synthesises XIM committed text event (XKeyEvent with XLookupString)
    │
    ▼
X11 application receives composed text
```

### XMODIFIERS and Startup

For an X11 application to reach the XIM server inside XWayland, the `XMODIFIERS` environment variable must be set before the application launches:

```bash
export XMODIFIERS=@im=fcitx5   # or @im=ibus, @im=none
```

XWayland reads `XMODIFIERS` at startup to determine which XIM server name to advertise on the X11 `_XIM_SERVERS` root window property. When the X11 application calls `XOpenIM()`, Xlib resolves the server name from `XMODIFIERS` and connects to XWayland's built-in XIM server at that name.

```c
/* Legacy X11 application: XIM setup */
Display *dpy = XOpenDisplay(NULL);
XIM xim = XOpenIM(dpy, NULL, NULL, NULL);  /* connects to XWayland's XIM server */
XIC xic = XCreateIC(xim,
    XNInputStyle, XIMPreeditNothing | XIMStatusNothing,  /* over-the-spot style */
    XNClientWindow, window,
    NULL);

/* Event loop: filter every event through the XIM before processing */
while (XNextEvent(dpy, &event)) {
    if (XFilterEvent(&event, None))
        continue;   /* XIM consumed this event (e.g. a composing keystroke) */
    if (event.type == KeyPress) {
        char buf[32];
        KeySym sym;
        Status status;
        int len = Xutf8LookupString(xic, &event.xkey, buf, sizeof(buf), &sym, &status);
        if (status == XLookupChars || status == XLookupBoth)
            /* buf contains the composed UTF-8 text */
            application_insert_text(buf, len);
    }
}
```

### Why Some X11 Apps Get IME and Others Don't

Several conditions must all be met for XIM to work under XWayland:

1. **`XMODIFIERS` set before app launch**: If the variable is absent, `XOpenIM()` returns `NULL` and the app falls back to direct key translation with no IME.
2. **XIM input style**: The app must request `XIMPreeditCallbacks` or `XIMPreeditNothing` (over-the-spot). Apps requesting `XIMPreeditPosition` (on-the-spot within the window) may work but candidate window placement can be incorrect.
3. **`XFilterEvent` call in the event loop**: Apps that do not call `XFilterEvent()` will receive raw keystrokes rather than composed text; the XIM state machine breaks.
4. **XWayland's text-input bridge**: The compositor must support `zwp_text_input_v3`. On compositors that don't (rare in 2026), XWayland cannot forward XIM events to the IME.

GTK3 under XWayland can be configured to use either the native GTK XIM module (which calls `XOpenIM`) or the `im-ibus` / `im-fcitx5` GTK modules. GTK4 bypasses XIM entirely by connecting directly to the Wayland compositor via `GDK_BACKEND=wayland` — even when the window appears as an X11 window to XWayland, the GDK backend is still Wayland.

### XMODIFIERS=@im=fcitx5 Interaction with XWayland

With `XMODIFIERS=@im=fcitx5`, the X11 app's `XOpenIM()` connects to XWayland's XIM server, which in turn holds its own `zwp_text_input_v3` object as a Wayland client to the compositor. The compositor routes this to fcitx5 via `zwp_input_method_v2`. From fcitx5's perspective, the input context created for the XWayland bridge is identical to one created for a native Wayland app; the engine processes pinyin/hangul/etc. identically.

The practical recommendation: set `XMODIFIERS=@im=fcitx5` unconditionally in the session environment, and rely on the native text-input-v3 path for GTK4/Qt6 Wayland apps. XWayland handles legacy X11 apps transparently.

---

## Emoji Input and Unicode

### The commit_string Model for Multi-Codepoint Emoji

Emoji delivered via an IME (GNOME Characters, ibus-typing-booster, ibus-m17n emoji input) follow the same `commit_string` path as CJK text. The emoji string is simply UTF-8 encoded and sent as the `text` argument of `commit_string`. The receiving application's text widget inserts it verbatim.

Multi-codepoint emoji sequences — including ZWJ sequences — are delivered as a single atomic string in one `commit_string` event. For example, the family emoji `👨‍👩‍👧` is the sequence U+1F468 MAN + U+200D ZWJ + U+1F469 WOMAN + U+200D ZWJ + U+1F467 GIRL, encoded as:

```
UTF-8 bytes: F0 9F 91 A8  E2 80 8D  F0 9F 91 A9  E2 80 8D  F0 9F 91 A7
             (👨 man)      (ZWJ)     (👩 woman)    (ZWJ)     (👧 girl)
```

This 18-byte sequence arrives in a single `commit_string` event. The application's text buffer stores it as a single logical cluster if the text engine (Pango, HarfBuzz) supports ZWJ emoji shaping — which all modern Linux text stacks do via HarfBuzz's emoji cluster detection.

```c
/* Compositor side: routing commit_string from IME to app (conceptual) */
static void
input_method_commit_string(struct wlr_input_method_v2 *im,
                            const char *text)
{
    /* Forward verbatim to the focused application's text-input object */
    struct wlr_text_input_v3 *ti = focused_text_input(im->seat);
    if (ti)
        zwp_text_input_v3_send_commit_string(ti->resource, text);
    /* The 'done' event will follow to apply the change */
}
```

### Emoji Pickers and GNOME Characters

GNOME Characters (`gnome-characters`) is not an IME; it delivers emoji by writing to the clipboard and then simulating a paste (Ctrl+V key events via `zwp_virtual_keyboard_v1`). This works regardless of whether the target application supports `zwp_text_input_v3`.

ibus-typing-booster and ibus-m17n's emoji input mode use the standard `commit_string` path. The user types a search string or Unicode code point; the IME's candidate window shows matching emoji; the user selects one; `commit_string` delivers the UTF-8 emoji string.

### Skin Tone Modifiers and Flag Sequences

Skin tone modifiers (U+1F3FB through U+1F3FF) are delivered as part of the `commit_string` when the emoji picker combines the base person emoji with a modifier, e.g. `👋🏽` = U+1F44B WAVING HAND + U+1F3FD MEDIUM SKIN TONE. Flag sequences use regional indicator symbols (U+1F1E0–U+1F1FF) in pairs, e.g. 🇩🇪 = U+1F1E9 + U+1F1EA. All of these are passed atomically in `commit_string`; the text widget never sees the individual codepoints arrive separately.

### Keycap Sequences

Keycap sequences (e.g. `1️⃣` = U+0031 DIGIT ONE + U+FE0F VARIATION SELECTOR-16 + U+20E3 COMBINING ENCLOSING KEYCAP) can cause issues if an application splits insertion by character rather than by grapheme cluster. Well-behaved text widgets use HarfBuzz's `hb_unicode_funcs_t` or ICU's `BreakIterator` to iterate by grapheme cluster, ensuring keycap sequences are treated as a unit. The Wayland compositor and `zwp_text_input_v3` are agnostic to this — they pass bytes, not semantics.

---

## Roadmap

### Near-term (6–12 months)

- **Stabilisation of `ext_text_input_v1`**: The long-running effort to promote `zwp_text_input_v3` from the `unstable` namespace into a stable `ext_text_input_v1` protocol remains the highest-priority item on the wayland-protocols text-input track. Several blocking issues around cursor-rectangle semantics and surrounding-text limits need resolution before the protocol can graduate. [Source](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/merge_requests?label_name%5B%5D=text-input)
- **Preedit text style hints in `zwp_text_input_v3`**: MR !234 (merged July 2023) added preedit hint flags; follow-on work to propagate these hints through GTK4's `GtkIMContext` and Qt6's `QInputMethodEvent` into application drawing code is ongoing in both toolkits. [Source](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/merge_requests/234)
- **IBus migration to `zwp_input_method_v2`**: IBus historically used `input-method-unstable-v1`; upstream work to port it to v2 (which wlroots compositors expose) is in progress and required for full Sway/Hyprland compatibility. Note: needs verification of current merge status.
- **Qt 6.8+ text-input-v3 fixes**: Qt 6.8.2 and later ship stability fixes for the text-input-v3 client path; distributions shipping Qt 6.7.x are encouraged to backport these patches for correct preedit rendering. [Source](https://github.com/pop-os/cosmic-session/issues/185)
- **COSMIC desktop IME support**: The COSMIC desktop (System76) identified in 2025 that its compositor ships versions of IBus/Fcitx5 too old for Wayland's `zwp_input_method_v2`; packaging updates targeting IBus ≥ 1.5.32 and Fcitx5 ≥ 5.1.12 are planned for upcoming releases. [Source](https://github.com/pop-os/cosmic-session/issues/185)

### Medium-term (1–3 years)

- **`ext_input_method_v1` stable protocol**: Once `ext_text_input_v1` is stable, a corresponding stable replacement for `zwp_input_method_v2` is expected to follow through the same wayland-protocols process, removing the `zwp`/unstable designation and giving compositors a stable ABI to target. Note: needs verification of current proposal status.
- **Candidate-window positioning protocol**: There is no Wayland protocol for the IME to request a compositor-managed popup surface at the cursor rectangle; today each IME draws its own unmanaged popup. A dedicated protocol (analogous to `xdg_popup` but for IME candidate windows) has been discussed on the wayland-devel mailing list as a way to allow compositors to enforce correct stacking, scaling, and screen-edge avoidance. [Source](https://dorotac.eu/posts/input_method/)
- **Unified text-input across Mutter and KWin**: GNOME's compositor (Mutter) routes text input through D-Bus to IBus rather than implementing `zwp_input_method_v2`, creating a behavioural divergence from wlroots compositors. Medium-term alignment — either Mutter adopting `zwp_input_method_v2` or a D-Bus bridge standardisation — is being discussed in GNOME issue trackers. Note: needs verification of specific issue links.
- **Wayland text-input v4 (Qt proposal)**: Qt engineers proposed `text-input-unstable-v4` at QtCS 2021 to address deficiencies in v3 (content-purpose, input-panel visibility). The proposal has not yet been adopted in wayland-protocols but influences the design of `ext_text_input_v1`. [Source](https://wiki.qt.io/QtCS2021_-_Wayland_text-input-unstable-v4_protocol)
- **Improved emoji and Unicode 16 support**: As Unicode 16 introduces new emoji families and ZWJ sequences, IME frameworks (ibus-typing-booster, fcitx5-chinese-addons) will need updated emoji databases; the `commit_string` transport is already sufficient — the work is entirely in the IME engines rather than the Wayland protocol layer.

### Long-term

- **Deprecation of XIM under XWayland**: As Wayland adoption completes in major distributions (Ubuntu 26.04 LTS ships without an Xorg session by default), the XWayland XIM bridge will see reduced investment; the long-term trajectory is for legacy X11 apps to either be ported or replaced, making the XIM compatibility layer progressively less critical. [Source](https://github.com/yazelin/ubuntu-26.04-setup/blob/main/notes/wayland-vs-xorg.md)
- **AI-assisted input methods**: Next-generation IME frameworks may integrate local large-language-model inference for context-aware completion and transliteration. The `set_surrounding_text` request in `zwp_text_input_v3` already provides the context window needed; the bottleneck is IME engine capability rather than protocol expressiveness.
- **Accessibility and switch-access integration**: Long-term, the virtual keyboard protocol (`zwp_virtual_keyboard_v1`) and the text-input protocol may be joined by a higher-level accessibility input protocol that allows switch-access devices and eye-tracking systems to drive text composition — a gap identified by the Wayland accessibility working group. Note: needs verification of working group formation and scope.

---

## Integrations

- **Ch34 (Wayland Core)** — `zwp_text_input_v3` follows the same `wl_registry`/listener pattern as all Wayland extensions
- **Ch35 (Compositors: Mutter/KWin)** — GNOME and KDE implement the compositor side of `zwp_text_input_v3`; GNOME routes to IBus via D-Bus rather than `zwp_input_method_v2`
- **Ch36 (wlroots)** — wlroots `wlr_text_input_v3` and `wlr_input_method_v2` are the reference implementations used by Sway, Hyprland, labwc
- **Ch123 (XWayland)** — XWayland's XIM bridge uses `zwp_text_input_v3` as the Wayland-side transport for X11 app text input
- **Ch39 (Wayland Input)** — `wl_keyboard.key` events are routed by the compositor to the IME process via `zwp_input_method_v2` before forwarding to applications

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
