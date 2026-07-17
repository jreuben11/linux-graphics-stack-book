# Chapter 175: Linux Compositor Accessibility — AT-SPI2, Screen Readers, and the Wayland Accessibility Gap

**Part VI — The Display Stack**

**Audiences**:
- **Compositor developers** — adding accessibility support to wlroots-based or custom compositors
- **TUI and terminal application developers** — who need to understand how terminal content is (or is not) exposed to screen readers
- **Platform engineers at GNOME and KDE** — navigating the transition from X11-based accessibility to Wayland-native approaches

The chapter assumes familiarity with Wayland's core security model (Chapter 20) and basic D-Bus concepts; knowledge of GTK or Qt internals is helpful but not required.

---

## Table of Contents

1. [Introduction: Accessibility and the Graphics Stack](#1-introduction-accessibility-and-the-graphics-stack)
2. [AT-SPI2: The Assistive Technology Service Provider Interface](#2-at-spi2-the-assistive-technology-service-provider-interface)
3. [Toolkit Integration: GTK4 and Qt6](#3-toolkit-integration-gtk4-and-qt6)
4. [The Compositor Layer: X11 History and the Wayland Gap](#4-the-compositor-layer-x11-history-and-the-wayland-gap)
5. [Orca Screen Reader](#5-orca-screen-reader)
6. [GNOME Magnifier and Zoom](#6-gnome-magnifier-and-zoom)
7. [Terminal Accessibility](#7-terminal-accessibility)
8. [Wayland Accessibility Protocol Proposals](#8-wayland-accessibility-protocol-proposals)
9. [Developer Guide: Adding AT-SPI2 to a Custom Application](#9-developer-guide-adding-at-spi2-to-a-custom-application)
10. [Integrations](#10-integrations)

---

## 1. Introduction: Accessibility and the Graphics Stack

A screen reader does not just read text files — it reads *the current state of the display* as seen by the user. For a blind user running a GTK application, Orca needs to know that the focused element is a push button labelled "OK", that it belongs to a dialog titled "Save File", and that pressing Space will activate it. For a low-vision user, a magnifier needs real-time access to the compositor's output to scale it by a factor of four without degrading into pixel blur. For a deaf-blind user, a braille display needs the same text that a screen reader would speak, delivered in milliseconds.

Each of these requirements reaches down into the graphics stack. The accessibility service provider needs to intercept the same keyboard events the application received, read the widget tree that the toolkit manages, receive notifications when that tree changes, and — for magnification — observe or clone the compositor's framebuffer output. This is why accessibility is not simply a GUI toolkit concern: it touches kernel input routing, compositor protocol design, D-Bus bus architecture, and GPU rendering pipelines simultaneously.

The Linux accessibility landscape in 2026 sits at an inflection point. The X11 era provided a working but deeply insecure model: screen readers could intercept all keystrokes and read all window properties with no permission controls. The Wayland era provides security-by-default isolation that inadvertently broke many accessibility tools. The community is now rebuilding the accessibility stack on Wayland-native foundations, with genuine progress but persistent gaps.

This chapter explains the full picture: how the X11 model worked, why Wayland breaks it, what AT-SPI2 provides as a toolkit-side bridge, and what the community is doing to close the remaining gaps.

---

## 2. AT-SPI2: The Assistive Technology Service Provider Interface

### 2.1 History and Overview

AT-SPI (Assistive Technology Service Provider Interface) was originally developed by Sun Microsystems for the GNOME 2 desktop and implemented over CORBA. After Sun's acquisition by Oracle and the subsequent withdrawal of commercial Linux accessibility investment, the CORBA underpinning was replaced with D-Bus, producing **AT-SPI2** — the current standard. AT-SPI2 is implemented in the `at-spi2-core` package and is the de facto standard for accessibility on Linux and other free desktops. [Source](https://www.freedesktop.org/wiki/Accessibility/AT-SPI2/)

The key insight of AT-SPI2 is to decouple the accessibility layer from the display server. Instead of routing through X11 window properties, AT-SPI2 exposes the widget tree as a tree of D-Bus objects on a dedicated **accessibility bus**. Screen readers and other assistive technologies query this bus directly; the compositor or display server is not involved except to route keyboard events.

### 2.2 The Accessibility Bus

AT-SPI2 operates on a separate D-Bus bus instance — not the session bus — to prevent the extremely chatty accessibility event traffic from saturating other clients on the session bus. [Source](https://github.com/GNOME/at-spi2-core/blob/main/bus/README.md)

The infrastructure is managed by `at-spi-bus-launcher`, a small daemon that:

1. Registers the name `org.a11y.Bus` on the session bus and exposes a single object at `/org/a11y/bus`.
2. Provides the `GetAddress` method on that object so that applications can discover the accessibility bus address on demand.
3. Manages the lifecycle of a separate `dbus-daemon` (or `dbus-broker`) instance that serves as the accessibility bus itself.
4. If a screen reader is enabled via GSettings, starts the accessibility bus unconditionally at session startup (not on demand) so that the screen reader can connect immediately.

An application that wants to register itself with the accessibility infrastructure calls `org.a11y.Bus.GetAddress` on the session bus, connects to the returned address, and then contacts the registry.

### 2.3 The Registry Daemon

The accessibility bus hosts `at-spi-registryd`, which claims the well-known name `org.a11y.atspi.Registry` on the accessibility bus and occupies the object path `/org/a11y/atspi/registry`. The registry serves two purposes:

- **Application registration**: Applications call `Embed` on the `org.a11y.atspi.Socket` interface of the registry's root object to announce themselves. The registry assigns the application an integer ID by setting the `Id` property on the application's root object.
- **Event brokering**: Assistive technologies call `RegisterEvent` on the `org.a11y.atspi.Registry` interface to subscribe to event classes. The registry notifies applications of which event types ATs are interested in, so applications can avoid emitting events nobody is listening for. [Source](https://gnome.pages.gitlab.gnome.org/at-spi2-core/devel-docs/doc-org.a11y.atspi.Registry.html)

### 2.4 Core D-Bus Interfaces

The formal interface definitions live in `at-spi2-core/xml/` as D-Bus introspection XML files. [Source](https://gitlab.gnome.org/GNOME/at-spi2-core) The complete set of interfaces under the `org.a11y.atspi` namespace includes:

| Interface | Purpose |
|---|---|
| `org.a11y.atspi.Accessible` | Core object: name, description, role, state, children |
| `org.a11y.atspi.Component` | Screen coordinates, bounding box, hit-test |
| `org.a11y.atspi.Text` | Text content, cursor position, selection, character attributes |
| `org.a11y.atspi.Action` | Named actions (e.g. "click", "expand") |
| `org.a11y.atspi.Selection` | Multi-selection in lists and trees |
| `org.a11y.atspi.Value` | Numeric value (sliders, progress bars) |
| `org.a11y.atspi.EditableText` | Text insertion and deletion |
| `org.a11y.atspi.Image` | Image description and bounding box |
| `org.a11y.atspi.Hypertext` | Hyperlink navigation |
| `org.a11y.atspi.Table` | Row/column navigation in grids |
| `org.a11y.atspi.Application` | Per-application metadata |
| `org.a11y.atspi.Socket` | Application registration with registry |
| `org.a11y.atspi.Cache` | Bulk transfer of the accessible tree |
| `org.a11y.atspi.Registry` | Event subscription management |
| `org.a11y.atspi.DeviceEventController` | Legacy keyboard/mouse event registration |

The **`org.a11y.atspi.Accessible`** interface is the foundation of the object model. Its key methods and properties are: [Source](https://gnome.pages.gitlab.gnome.org/at-spi2-core/devel-docs/doc-org.a11y.atspi.Accessible.html)

```xml
<!-- Simplified representation of the org.a11y.atspi.Accessible interface.
     The canonical XML is at at-spi2-core/xml/Accessible.xml in the GNOME
     at-spi2-core repository: https://gitlab.gnome.org/GNOME/at-spi2-core
     Type signatures and method names are drawn from the devel-docs reference. -->
<interface name="org.a11y.atspi.Accessible">
  <property name="Name"        type="s"  access="read"/>
  <property name="Description" type="s"  access="read"/>
  <property name="Parent"      type="(so)" access="read"/>
  <property name="ChildCount"  type="i"  access="read"/>
  <property name="Locale"      type="s"  access="read"/>
  <property name="AccessibleId" type="s" access="read"/>

  <method name="GetChildAtIndex">
    <arg name="index"  type="i" direction="in"/>
    <arg name="child"  type="(so)" direction="out"/>
  </method>
  <method name="GetChildren">
    <arg name="children" type="a(so)" direction="out"/>
  </method>
  <method name="GetIndexInParent">
    <arg name="index" type="i" direction="out"/>
  </method>
  <method name="GetRole">
    <arg name="role" type="u" direction="out"/>
  </method>
  <method name="GetState">
    <arg name="states" type="au" direction="out"/>
  </method>
  <method name="GetInterfaces">
    <arg name="interfaces" type="as" direction="out"/>
  </method>
  <method name="GetApplication">
    <arg name="application" type="(so)" direction="out"/>
  </method>
</interface>
```

Each method returning `(so)` returns a D-Bus bus name and object path tuple, which together uniquely identify an accessible object on the accessibility bus. The caller must then issue a separate D-Bus call to that object to retrieve its properties.

### 2.5 The Cache Interface

Retrieving the accessible tree object-by-object produces substantial D-Bus round-trip overhead. The `org.a11y.atspi.Cache` interface addresses this with a bulk transfer mechanism. [Source](https://gnome.pages.gitlab.gnome.org/at-spi2-core/devel-docs/doc-org.a11y.atspi.Cache.html)

`GetItems` returns an array where each element contains the full metadata for one accessible object: its D-Bus identity `(so)`, its application reference, its parent reference, its index in parent, its child count, the list of interfaces it implements, its name, its role, its description, and its current state flags — all in a single round-trip. As the tree changes, the `AddAccessible` and `RemoveAccessible` signals deliver incremental updates.

### 2.6 The Event System

ATs subscribe to events via `RegisterEvent` on the `org.a11y.atspi.Registry` interface. The event type argument is a colon-separated string of the form `EventClass:major_type:minor_type:detail`, where only the `EventClass` is required — partial strings subscribe to all sub-events within that class. For example:

```
"object:"              # all object events
"object:state-changed" # state changes only
"object:text-changed"  # text mutations only
"focus:"               # all focus events
```

When a toolkit widget changes state (e.g., a button gains keyboard focus), the toolkit emits a D-Bus signal on the accessibility bus on the interface corresponding to the event class. The principal event interfaces are:

- `org.a11y.atspi.Event.Object` — `StateChanged`, `ChildrenChanged`, `TextChanged`, `PropertyChange`, `VisibleDataChanged`
- `org.a11y.atspi.Event.Focus` — `Focus`
- `org.a11y.atspi.Event.Window` — `Activate`, `Deactivate`, `Create`, `Destroy`
- `org.a11y.atspi.Event.Keyboard` — keyboard event notifications (legacy path)

This event-driven architecture means that Orca does not poll the accessible tree — it receives push notifications when anything changes, and fetches details on demand. [Source](https://gnome.pages.gitlab.gnome.org/at-spi2-core/devel-docs/architecture.html)

---

## 3. Toolkit Integration: GTK4 and Qt6

### 3.1 GTK4

GTK4 underwent a significant rewrite of its accessibility implementation compared to GTK3. Under GTK3, the flow was: GTK widget → ATK (abstract GObject interfaces) → `atk-bridge` (adapter library) → D-Bus. GTK4 removes the ATK intermediary entirely and emits AT-SPI2 D-Bus messages directly. [Source](https://blog.gtk.org/2020/10/21/accessibility-in-gtk-4/)

The GTK4 accessibility model is built on three concepts aligned with WAI-ARIA: [Source](https://docs.gtk.org/gtk4/section-accessibility.html)

**Roles** define the semantic type of a widget. Roles are immutable after widget creation — a button is always a button. Examples include `GTK_ACCESSIBLE_ROLE_BUTTON`, `GTK_ACCESSIBLE_ROLE_TEXTBOX`, `GTK_ACCESSIBLE_ROLE_CHECKBOX`, `GTK_ACCESSIBLE_ROLE_SLIDER`, and dozens more. Every role carries an implicit promise about keyboard behaviour — assigning a role commits the widget to the interaction patterns ATs expect.

**States** are transient widget conditions: `GTK_ACCESSIBLE_STATE_FOCUSED`, `GTK_ACCESSIBLE_STATE_CHECKED`, `GTK_ACCESSIBLE_STATE_DISABLED`, `GTK_ACCESSIBLE_STATE_EXPANDED`. State changes fire the `StateChanged` AT-SPI2 signal automatically.

**Properties** and **Relations** provide additional semantic metadata: `GTK_ACCESSIBLE_PROPERTY_LABEL`, `GTK_ACCESSIBLE_PROPERTY_PLACEHOLDER`, `GTK_ACCESSIBLE_RELATION_LABELLED_BY` (mapping a widget to the label that describes it), `GTK_ACCESSIBLE_RELATION_CONTROLS` (associating a slider with the value it controls).

The platform connection layer is `GtkATContext`, an abstract class where each platform supported by GTK provides a subclass. On Linux and BSD, `GtkAtSpiContext` (implemented in `gtk/a11y/gtkatspicontext.c` [Source](https://gitlab.gnome.org/GNOME/gtk/-/blob/main/gtk/a11y/gtkatspicontext.c)) is responsible for registering the widget's D-Bus object on the accessibility bus and emitting AT-SPI2 signals in response to `GtkAccessible` state changes. A dedicated test backend also exists for CI purposes. Prior to GTK 4.18, there were no production AT context backends for macOS or Windows — those platforms received no GTK accessibility support.

For complex text widgets, GTK 4.14 introduced the public `GtkAccessibleText` interface, which out-of-tree widgets can implement to expose formatted text, caret position, selection bounds, and character attributes (font, size, colour) to ATs. [Source](https://blog.gtk.org/2024/03/08/accessibility-improvements-in-gtk-4-14/)

GTK 4.18 introduced a new **AccessKit** backend as an additional `GtkATContext` subclass. AccessKit is a cross-platform accessibility abstraction library; the new backend translates GTK's `GtkAccessible` model into AccessKit's tree and from there into native platform APIs on macOS and Windows — providing real accessibility output on those platforms for the first time. On Linux, the AccessKit backend provides the hook point for Newton's experimental Wayland push-model protocol (see Section 8); the existing `GtkAtSpiContext` AT-SPI2 backend remains the production path. [Source](https://blogs.gnome.org/tbernard/2025/04/11/gnome-stf-2024/)

A simple GTK4 custom widget making itself accessible:

```c
/* my_button.c — custom GTK4 widget with accessible role */
static void my_button_class_init(MyButtonClass *klass)
{
    GtkWidgetClass *widget_class = GTK_WIDGET_CLASS(klass);

    /* Set accessible role once for the class — immutable per instance */
    gtk_widget_class_set_accessible_role(widget_class,
                                         GTK_ACCESSIBLE_ROLE_BUTTON);
}

static void my_button_set_label(MyButton *self, const char *label)
{
    g_free(self->label);
    self->label = g_strdup(label);

    /* Notify AT-SPI2 of the property change */
    gtk_accessible_update_property(GTK_ACCESSIBLE(self),
        GTK_ACCESSIBLE_PROPERTY_LABEL, label,
        -1);
}

static void my_button_set_sensitive(MyButton *self, gboolean sensitive)
{
    /* State changes propagate automatically from GtkWidget:sensitive */
    gtk_widget_set_sensitive(GTK_WIDGET(self), sensitive);
}
```

### 3.2 Qt6

Qt6 provides AT-SPI2 accessibility on Linux through a plugin architecture. The relevant source is `src/gui/accessible/linux/qspiaccessiblebridge.cpp` in `qtbase`. [Source](https://code.qt.io/cgit/qt/qtbase.git/log/src/gui/accessible/linux/qspiaccessiblebridge.cpp)

The Qt accessibility abstraction layer centres on `QAccessible` and `QAccessibleInterface`. Standard Qt widgets (buttons, text editors, list views) implement `QAccessibleInterface` automatically. Custom widgets must provide a subclass:

```cpp
// MyWidget accessible interface implementation
class MyWidgetAccessible : public QAccessibleWidget
{
public:
    MyWidgetAccessible(MyWidget *widget)
        : QAccessibleWidget(widget, QAccessible::Button) {}

    QString text(QAccessible::Text type) const override {
        if (type == QAccessible::Name)
            return static_cast<MyWidget*>(object())->label();
        return QAccessibleWidget::text(type);
    }

    QAccessible::State state() const override {
        QAccessible::State s = QAccessibleWidget::state();
        s.disabled = !object()->isEnabled();
        return s;
    }
};

// Register the factory
QAccessible::installFactory([](const QString &classname, QObject *obj) {
    if (classname == QLatin1String("MyWidget"))
        return static_cast<QAccessibleInterface*>(
            new MyWidgetAccessible(qobject_cast<MyWidget*>(obj)));
    return static_cast<QAccessibleInterface*>(nullptr);
});
```

The `QSpiAccessibleBridge` plugin translates `QAccessibleInterface` calls into AT-SPI2 D-Bus messages on the accessibility bus, implementing the same object model as GTK4's `GtkAtSpiContext`. Qt applications therefore appear as first-class citizens in the AT-SPI2 object tree alongside GTK applications.

KDE Plasma's accessibility integration uses this same Qt AT-SPI2 bridge. The `qtatspi` plugin is built as part of `qtbase` (not as a separate package) when the `accessibility` Meson/CMake feature is enabled. [Source](https://community.kde.org/Accessibility/qt-atspi)

---

## 4. The Compositor Layer: X11 History and the Wayland Gap

### 4.1 X11: Global Access as a Feature

The X11 accessibility model rested on two capabilities that Wayland deliberately removed:

**Global keyboard interception**: Under X11, a screen reader could register a passive keyboard grab using `XGrabKey` on the root window, causing the X server to deliver copies of all keyboard events to the screen reader regardless of which application had focus. This allowed Orca to intercept its command keys (Caps Lock, Insert) from anywhere in the session.

**Cross-window property inspection**: An X11 client could call `XQueryTree` on the root window to enumerate all windows in the session, then call `XGetWindowProperty` to read any window's properties. The `_NET_WM_NAME` and related EWMH properties exposed window titles. Combined with AT-SPI 1.x (CORBA-based), this gave screen readers the ability to build an accessible tree from any running application, even one that did not explicitly support AT-SPI, by falling back to window-level inspection.

These capabilities were not permissions — they were simply part of the X11 protocol. Any connected client had them. The security implications (keyloggers, screen scrapers, window spies) were accepted because X11 was designed in a network-transparent era when all clients on a display were assumed to be mutually trusted.

### 4.2 Wayland: Security by Design, Accessibility by Accident

Wayland's isolation model (described in detail in Chapter 20) removes both capabilities structurally:

- There is no root window, no global window namespace, and no mechanism for one client to enumerate another's surfaces.
- Keyboard events are delivered only to the client that owns the focused surface. There is no mechanism for a client to receive keyboard events destined for another client's surface.
- The `keyboard-shortcuts-inhibit-unstable-v1` protocol allows a client to request that the compositor stop consuming its own shortcut keys and forward them to the client's surface instead [Source](https://wayland.app/protocols/keyboard-shortcuts-inhibit-unstable-v1) — but this only works while that surface has focus, and the compositor retains the right to override the inhibition. This is usable by full-screen game engines and terminal emulators, but not by a screen reader that needs global key interception regardless of focus.

The practical consequence was severe: GTK4's accessibility rewrite in 2020 removed the legacy ATK intermediary, which had been the last source of keyboard events that reached Orca on a Wayland session. After this change, Orca on a Wayland session could still read the AT-SPI2 tree (because AT-SPI2 is toolkit-side and does not require compositor cooperation), but it could not intercept its own command keys — Caps Lock, Insert, and function keys — making it impossible to stop speech synthesis, navigate headings, or open Orca's settings panel. [Source](https://lwn.net/Articles/1025127/)

The additional gap on Wayland compared to X11 is:
- **No global keyboard hooks**: Screen reader command keys cannot be intercepted globally.
- **No synthesized mouse events cross-client**: Sending a mouse event to a different application's window is not possible without compositor cooperation (it requires `zwp_virtual_keyboard_v1` or similar, which compositors may or may not expose to accessibility clients).
- **No screen-coordinate normalisation**: AT-SPI2 returns widget bounding boxes in surface-local coordinates; the compositor has the global coordinates, but there is no standard protocol for an AT to query the compositor for a surface's position. Newton's prototype work found that this prevents coordinate-based "explore by touch" features from working. [Source](https://blogs.gnome.org/a11y/2024/06/18/update-on-newton-the-wayland-native-accessibility-project/)

### 4.3 The Interim Solution: GNOME 48 Keyboard Interception

The immediate fix for the keyboard gap landed in **GNOME 48** (released March 2025). Lukáš Tyrychtr (Red Hat) adapted Matt Campbell's Newton prototype work into a standalone, pragmatic patch: a new D-Bus interface that Mutter exposes specifically for screen readers, allowing them to register keyboard shortcuts that the compositor will deliver to them regardless of focus. [Source](https://lwn.net/Articles/1025127/)

The same D-Bus interface was added to **KWin in KDE 6.4**, extending the fix to Plasma Wayland sessions. The interface uses hardcoded D-Bus service name checks to limit access to designated assistive technology processes, preventing arbitrary applications from registering global keyboard hooks through this mechanism.

This is not a general Wayland protocol — it is a compositor-specific D-Bus interface. GNOME and KDE implement the same interface name, which gives Orca portability across both desktops, but it does not help compositors outside GNOME and KDE (sway, river, Hyprland, etc.).

### 4.4 xdg-desktop-portal and the Accessibility Shortcuts Proposal

The `xdg-desktop-portal` framework provides compositor-independent access to privileged operations via D-Bus interfaces that portal backends implement per-compositor. As of 2026, there is no merged accessibility portal interface, but an open proposal (`org.freedesktop.portal.AT.Shortcuts`) was filed in June 2023 and remains under active discussion. [Source](https://github.com/flatpak/xdg-desktop-portal/issues/1046)

The proposed interface would:
- Present a one-time user-consent prompt when an AT requests shortcut interception.
- Support flexible key binding (including single-key and multi-modifier combinations like `Insert + a`).
- Allow bulk shortcut map updates via `{sa(sa{sv})}` D-Bus structures.
- Give AT shortcuts higher priority than general global shortcuts.
- Allow system administrators to configure a default AT that bypasses the consent prompt.

The proposal is inspired by Android's accessibility permissions model. Its advantage over the GNOME/KDE-specific D-Bus interface is that it would work on any compositor that implements xdg-desktop-portal backends. **Note: needs verification** as to whether any compositor has begun implementing this portal.

---

## 5. Orca Screen Reader

### 5.1 Architecture

Orca is a Python daemon distributed by the GNOME project. It subscribes to AT-SPI2 D-Bus events on the accessibility bus and translates the accessible object tree into speech and braille output. [Source](https://wiki.debian.org/Accessibility/Orca)

The high-level architecture is:

```
AT-SPI2 D-Bus events
        │
        ▼
  pyatspi2 / libatspi   ← Python bindings for at-spi2-core's C library
        │
        ▼
     Orca core
    (Python daemon)
        │
        ├─────────────────────────────────►  Speech Dispatcher
        │                                    │
        │                                    ▼
        │                             eSpeak-NG / Festival
        │                                  / Piper TTS
        │
        └─────────────────────────────────►  BrlAPI / BRLTTY
                                             │
                                             ▼
                                       Braille display
```

**Speech output**: Orca does not drive the speech synthesizer directly. It communicates with `speech-dispatcher`, a middleware daemon that abstracts over multiple TTS engines (eSpeak-NG, Festival, Piper, and others). This allows users to switch TTS engines without reconfiguring Orca. [Source](https://grokipedia.com/page/orca_assistive_technology)

**Braille output**: Orca uses `BrlAPI`, the client library for `BRLTTY`, which manages communication with physical braille display hardware. The `liblouis` library provides contracted braille translation (converting English text to grade-2 contracted braille). Both `brlapi` and `liblouis` are optional dependencies.

### 5.2 How Orca Navigates the Accessible Tree

When the user presses an Orca navigation command (e.g., "read next heading"), Orca queries the AT-SPI2 accessible tree:

```python
# Simplified Orca navigation logic (illustrative)
import pyatspi

def find_next_heading(current_node):
    """Walk the accessible tree forward, looking for heading role."""
    desktop = pyatspi.Registry.getDesktop(0)
    # Navigate forward through the flat accessible tree
    node = current_node
    while node:
        node = next_node_in_tree(node)
        if node and node.getRole() == pyatspi.ROLE_HEADING:
            return node
    return None

def speak_node(node, orca_state):
    name = node.name
    role_name = node.getRoleName()
    # Check for GtkAccessible text content
    if node.queryInterface("org.a11y.atspi.Text"):
        text_iface = node.queryText()
        content = text_iface.getText(0, text_iface.characterCount)
        orca_state.speech.speak(f"{role_name}: {content}")
    else:
        orca_state.speech.speak(f"{name} {role_name}")
```

`pyatspi` wraps `libatspi`, the C library in `at-spi2-core`. Recent work (2023 onward) aims to remove the `pyatspi2` binding layer and have Orca call `libatspi` directly via GObject introspection, reducing the number of context switches between Orca and `dbus-daemon`. [Source](https://gnome.pages.gitlab.gnome.org/at-spi2-core/devel-docs/architecture.html)

### 5.3 Orca on Wayland: Current State (2026)

As of GNOME 48 / Orca 48:

**Working**: Reading GTK4 application content via AT-SPI2 (Nautilus, GNOME Text Editor, Fractal, Podcasts). Flat review (reading a screen line-by-line). Word and character navigation. Orca command keys (Caps Lock, Insert combinations) on GNOME and KDE Plasma Wayland sessions via the new compositor D-Bus keyboard interception interface.

**Still missing or limited**:
- Synthesized mouse event injection into other windows (requires compositor cooperation not yet standardised).
- Screen coordinate normalisation for explore-by-touch.
- Full GNOME Shell UI accessibility (gnome-shell uses its own custom Clutter widget toolkit, not GTK4; it is not yet integrated with Newton's AccessKit-based path). GNOME Shell still uses the legacy AT-SPI2 path for its own widgets.
- Orca on compositors that have not implemented the keyboard interception D-Bus interface (non-GNOME, non-KDE compositors).

---

## 6. GNOME Magnifier and Zoom

### 6.1 Architecture: Compositor-Side Scaling

The GNOME Shell magnifier is implemented as a JavaScript component (`magnifier.js`) inside the GNOME Shell compositor process itself. This is the correct architectural choice for Wayland: because GNOME Shell (running atop Mutter) is the compositor, it has direct access to the composited scene graph. It does not need to capture a framebuffer snapshot and then re-render it — it can clone and scale the scene graph's texture output directly. [Source](https://wiki.gnome.org/Projects/GnomeShell/Magnification)

The magnifier treats desktop magnification as a compositing effect. Mutter uses a Clutter-based scene graph where all windows are actors. The magnifier creates a zoomed viewport actor that mirrors the region of interest, applies an affine scale transform, and presents it to the monitor. The GPU performs the scaling during composition — this is the same path used by any other compositor scaling operation (like HiDPI scaling), and it avoids a CPU copy or a separate screen-capture pipeline.

The magnification factor, mouse tracking mode, and display position (full-screen, split-screen, or lens) are all configurable via GSettings under the `org.gnome.desktop.a11y.magnifier` schema.

### 6.2 Power Implications

Keeping the magnifier active prevents the display from entering low-power refresh states. The Mutter compositor must re-composite the display on every frame (or at least whenever the cursor moves), because the magnified viewport must follow the pointer. On integrated graphics, this is typically manageable, but the magnifier does prevent runtime GPU power gating and will measurably increase power draw compared to an idle compositor.

An interaction between the magnifier and screen capture tools exists: because the magnifier operates on the compositor's scene graph, screen recorders that capture via the compositor (using the xdg-desktop-portal ScreenCast interface and PipeWire) may capture the magnified view rather than the logical desktop, depending on implementation. [Source](https://gitlab.gnome.org/GNOME/gnome-shell/-/issues/4882) **Note: needs verification** of the exact compositor code path that determines whether a recording captures pre- or post-magnification output.

### 6.3 KDE Zoom

KDE Plasma's zoom functionality is similarly implemented as a KWin compositor effect, operating on the scene graph rather than requiring a separate framebuffer capture. KDE Plasma 6.5 relocated the zoom and invert effects from the Desktop Effects section into the main Accessibility settings page, making them easier to discover. [Source](https://blogs.kde.org/2025/06/14/this-week-in-plasma-wayland-pip-and-accessibility/)

---

## 7. Terminal Accessibility

### 7.1 VTE: The Accessible Terminal Widget

`libvte` (Virtual Terminal Emulator) is a GTK widget used by GNOME Terminal, GNOME Console (kgx), Tilix, Terminator, and other GTK-based terminal emulators. VTE exposes terminal content via `GtkAccessibleText`, the GTK4 interface for complex text content.

Through AT-SPI2, VTE exposes the terminal screen buffer as an `org.a11y.atspi.Text` object. Orca can perform:

- **Flat review**: Reading the terminal screen line by line.
- **Live monitoring**: Terminal output is announced as it arrives, via the `TextChanged` AT-SPI2 signal.
- **Typing echo**: Characters typed in the terminal are echoed to speech.
- **Caret tracking**: The terminal cursor position is reflected as the accessible text caret.

The limitation is fundamental to how terminals work: the `org.a11y.atspi.Text` interface exposes raw character sequences. A terminal running `htop` or `ranger` looks to Orca like a flat text buffer full of box-drawing characters and ANSI colour codes — there is no semantic structure (no "this cell-drawing character is a window border, not content"). This leads directly to Section 7.3.

However, the GTK4 AccessKit backend in GTK 4.18 does not yet support out-of-tree text widgets that implement `GtkAccessibleText` (such as VTE's GTK4 widget). This means that VTE-based terminals like GNOME Console do not yet work with Newton's experimental push-model path and remain on the legacy AT-SPI2 pull-model. [Source](https://github.com/ghostty-org/ghostty/discussions/2351)

### 7.2 Ghostty

Ghostty is a GPU-accelerated terminal emulator that renders using a Metal/OpenGL/Vulkan backend. Because it renders directly to a GPU surface, no GTK widget tree contains the text content — the characters live only in GPU buffers. This means standard AT-SPI2 cannot observe the terminal content; there is no `GtkAccessibleText` object to query.

A community contributor (alex19EP) has developed a proof-of-concept that implements `GtkAccessibleText` for Ghostty's GTK4 window, exposing the terminal buffer via GTK4's native accessibility interfaces. This enables:

- Orca flat review navigation (line, word, character granularity).
- Live announcement of new output.
- Braille display support.
- Per-character text attributes (foreground/background colours for colour-based navigation). [Source](https://github.com/ghostty-org/ghostty/discussions/2351)

As of mid-2026, this work is a pull request under review; it is not yet in a stable Ghostty release. The Ghostty maintainers have indicated that macOS accessibility (via `NSAccessibility`) is the primary initial target, with Linux following.

### 7.3 Alacritty

Alacritty is a GPU-rendered terminal that historically provided essentially no AT-SPI2 accessibility support on Linux. Issue #3855 in the Alacritty repository tracks a feature request for a "text selection mode" for keyboard-based navigation, filed by a user with a neuromuscular disorder who cannot use mouse-based copy/paste. As of mid-2026, this remains open with no committed implementation timeline. [Source](https://github.com/alacritty/alacritty/issues/3855) **Note: needs verification** of current status.

### 7.4 WezTerm

WezTerm's issue #913 tracks accessibility support. The planned approach is to introduce an explicit "read cursor" (independent from the visual viewport) and use `speech-dispatcher` as the speech output backend. As of mid-2026, this issue remains open; no AT-SPI2 implementation has been merged. [Source](https://github.com/wezterm/wezterm/issues/913)

### 7.5 TUI Application Accessibility: The Semantic Gap

When a screen reader reads a terminal running a TUI application (ncurses `vim`, ratatui `broot`, etc.), it sees the terminal's raw text buffer via AT-SPI2. This creates a fundamental semantic gap:

**"Terminal content is readable"** means: Orca can read the characters on screen using flat review. A user can navigate line by line and hear the raw text, including box-drawing characters and labels.

**"TUI is navigable"** means: the screen reader can understand the structure — "this is a menu", "this is a list of items", "pressing Down Arrow moves to the next item". This semantic understanding does not exist for TUI applications via the terminal's AT-SPI2 interface, because the terminal has no knowledge of the TUI framework's widget model.

Some TUI applications work reasonably well in practice because their output is naturally linear text (e.g., a pager like `less`). Others (multi-pane editors, complex dashboards) are essentially unusable with Orca's flat-review mode because the visual layout encodes information that the raw text stream does not.

The only path to full TUI navigability via AT-SPI2 would be for the TUI framework itself to directly implement AT-SPI2 — emitting accessible objects for each "widget" in the TUI. No mainstream TUI framework (ncurses, ratatui, crossterm) currently does this. This is an area where the terminal accessibility story is fundamentally limited by the terminal's role as a dumb text renderer.

---

## 8. Wayland Accessibility Protocol Proposals

### 8.1 The Wayland Protocols Issue Tracker

Issue #65 on the `wayland/wayland-protocols` GitLab tracks the request for a native Wayland accessibility protocol. The issue was filed noting that embedded systems may lack D-Bus, that sandboxed clients may not be able to connect to the accessibility bus, and that a Wayland-native protocol would allow the compositor to enforce that only surfaces with focus can use accessibility features. [Source](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/issues/65) As of mid-2026, no merged `ext-accessibility-v1` protocol exists in the `wayland-protocols` repository.

The obstacles to a standard Wayland accessibility protocol are genuine:

- **Security model**: Who can connect to the accessibility channel? The current AT-SPI2 approach uses D-Bus authorization to restrict access to designated assistive technology processes. A Wayland protocol would need to replicate this, likely through compositor policy (similar to how `wp_security_context_v1` tags sandbox contexts).
- **Performance model**: The AT-SPI2 pull model generates heavy D-Bus traffic. A Wayland protocol could use shared memory or file descriptors to pass the accessibility tree more efficiently — but this requires careful design to avoid becoming a new side channel.
- **Cross-toolkit standardisation**: The protocol must work with GTK, Qt, Flutter, Electron, and web browsers simultaneously. Each represents a different accessibility abstraction.

### 8.2 Newton: The Push-Model Experiment

The Newton project, developed by Matt Campbell with GNOME STF (Sovereign Tech Fund) funding in 2024, represents the most concrete attempt at a Wayland-native accessibility architecture. [Source](https://blogs.gnome.org/a11y/2024/06/18/update-on-newton-the-wayland-native-accessibility-project/) [Source](https://blogs.gnome.org/tbernard/2025/04/11/gnome-stf-2024/)

Newton's architecture has three layers:

**Layer 1 — Wayland protocol (toolkit → compositor)**: Applications push their accessibility tree to Mutter via a Wayland protocol extension. Critically, this synchronises the accessibility state with the visual frame: the accessibility tree update is delivered at the same time as the rendered frame, so the screen reader never sees stale coordinates or widget states. The serialisation format is currently JSON, though this is not finalised.

**Layer 2 — D-Bus protocol (compositor → screen reader)**: Orca communicates with Mutter via a D-Bus protocol. This is intentionally D-Bus rather than Wayland, because D-Bus provides the authorization controls needed to limit which process can act as the screen reader. The D-Bus protocol includes the keyboard interception functionality that landed in GNOME 48.

**Layer 3 — AccessKit (cross-platform abstraction)**: AccessKit is a cross-platform accessibility tree abstraction library. Matt Campbell added AccessKit support to GTK 4.18, which gives GTK applications accessibility on Windows and macOS (via native platform APIs) as a side effect, and provides the integration point for Newton's Wayland protocol on Linux.

What Newton achieves vs. what remains:

| Capability | AT-SPI2 (current) | Newton (experimental) |
|---|---|---|
| GTK4 app content reading | Yes | Yes |
| Keyboard command interception (GNOME/KDE) | Yes (GNOME 48+) | Yes |
| Frame-synchronised accessible tree | No (pull, asynchronous) | Yes |
| Flatpak without a11y exception | No | Yes |
| Screen coordinates from compositor | No | Partial (surface-local only) |
| Touch exploration / eye tracking | No | No |
| GNOME Shell itself accessible | Partial | No (not yet integrated) |
| Cross-desktop standard | N/A | No (GNOME-specific prototype) |

Newton's Wayland protocol is "not yet rigorously defined" and the D-Bus protocol is "ad-hoc". A cross-desktop standardisation conversation has not yet begun as of mid-2026. [Source](https://blogs.gnome.org/tbernard/2025/04/11/gnome-stf-2024/)

### 8.3 GNOME's Practical Strategy

GNOME's working strategy, independent of Newton, is to push all accessibility responsibility to the toolkit layer and avoid needing compositor-level extensions beyond the keyboard interception D-Bus interface. The reasoning: if every GTK4 widget is accessible via AT-SPI2, and if the compositor only needs to forward keyboard events to the screen reader, then Mutter can remain largely uninvolved in the accessibility stack. This is the minimum viable product that makes Orca functional on Wayland — and it shipped in GNOME 48.

This strategy works well for GNOME's own application stack but does not help Electron apps, Firefox, and other toolkits that have their own AT-SPI2 implementations and their own gaps.

---

## 9. Developer Guide: Adding AT-SPI2 to a Custom Application

### 9.1 GTK4: Accessible Custom Widgets

For GTK4 custom widgets, the steps are:

1. **Set the accessible role** in your widget class init:

```c
gtk_widget_class_set_accessible_role(klass, GTK_ACCESSIBLE_ROLE_LIST_ITEM);
```

2. **Update properties** when labels or descriptions change:

```c
gtk_accessible_update_property(GTK_ACCESSIBLE(self),
    GTK_ACCESSIBLE_PROPERTY_LABEL,       "Item name",
    GTK_ACCESSIBLE_PROPERTY_DESCRIPTION, "Long description of item",
    -1);
```

3. **Update states** when selection or enablement changes:

```c
gtk_accessible_update_state(GTK_ACCESSIBLE(self),
    GTK_ACCESSIBLE_STATE_SELECTED, TRUE,
    -1);
```

4. **Establish relations** for label-widget pairs:

```c
/* mywidget is described by mylabel */
gtk_accessible_update_relation(GTK_ACCESSIBLE(mywidget),
    GTK_ACCESSIBLE_RELATION_LABELLED_BY,
    mylabel, NULL,
    -1);
```

5. **Implement `GtkAccessibleText`** for custom text-rendering widgets (GTK 4.14+).

The key vfuncs to implement, with their real signatures from the GTK4 API
([Source](https://docs.gtk.org/gtk4/iface.AccessibleText.html)):

```c
/* get_contents: return text bytes in [start, end) character range */
static GBytes *
my_text_widget_get_contents(GtkAccessibleText *self,
                             unsigned int       start,
                             unsigned int       end)
{
    MyTextWidget *w = MY_TEXT_WIDGET(self);
    /* Slice the UTF-8 buffer and return as GBytes */
    const char *buf = w->text_buffer;
    gsize len = end - start;          /* character count, not byte count */
    return g_bytes_new(buf + start, len);
}

/* get_caret_position: return current caret offset in characters */
static unsigned int
my_text_widget_get_caret_position(GtkAccessibleText *self)
{
    return MY_TEXT_WIDGET(self)->caret_pos;
}

/* Wire up the interface */
static void
my_text_widget_accessible_text_init(GtkAccessibleTextInterface *iface)
{
    iface->get_contents        = my_text_widget_get_contents;
    iface->get_caret_position  = my_text_widget_get_caret_position;
    /* implement get_selection, get_attributes, get_extents as needed */
}

G_IMPLEMENT_INTERFACE(GTK_TYPE_ACCESSIBLE_TEXT,
                      my_text_widget_accessible_text_init)
```

When the caret moves or text changes, notify the AT-SPI2 infrastructure via:

```c
gtk_accessible_text_update_caret_position(GTK_ACCESSIBLE_TEXT(self));
gtk_accessible_text_update_contents(GTK_ACCESSIBLE_TEXT(self),
                                    GTK_ACCESSIBLE_TEXT_CONTENT_CHANGE_INSERT,
                                    start, end);
```

### 9.2 Qt6: Custom Widget Interface

```cpp
// Implement QAccessibleInterface for a custom widget
class MyListAccessible : public QAccessibleObject
{
public:
    MyListAccessible(MyList *list) : QAccessibleObject(list), m_list(list) {}

    QAccessible::Role role() const override {
        return QAccessible::List;
    }

    QString text(QAccessible::Text type) const override {
        if (type == QAccessible::Name)
            return m_list->title();
        return {};
    }

    int childCount() const override {
        return m_list->itemCount();
    }

    QAccessibleInterface *child(int index) const override {
        return new MyListItemAccessible(m_list->itemAt(index));
    }

    QRect rect() const override {
        return m_list->mapToGlobal(m_list->rect());
    }

private:
    MyList *m_list;
};
```

### 9.3 Testing Accessibility

**Accerciser**: The primary AT-SPI2 inspection tool. Launch it with `accerciser`, then select any running application in the left panel to see its accessible tree. The Interface Viewer plugin shows which AT-SPI2 interfaces each node implements and lets you call methods directly. The Event Monitor plugin displays `StateChanged`, `ChildrenChanged`, `TextChanged`, and other events in real time as the application runs. [Source](https://help.gnome.org/users/accerciser/stable/introduction.html.en)

**D-Bus monitoring on the accessibility bus**: Standard `dbus-monitor` operates on the session bus and will not see AT-SPI2 events. To monitor the accessibility bus specifically, you must obtain its address and pass it explicitly:

```bash
# Get the accessibility bus address
A11Y_BUS=$(gdbus call \
    --session \
    --dest org.a11y.Bus \
    --object-path /org/a11y/bus \
    --method org.a11y.Bus.GetAddress | sed "s/('//;s/',)//")

# Monitor events on the accessibility bus
dbus-monitor --address "$A11Y_BUS"
```

**Orca debug mode**: Running `orca --debug` writes a verbose log of every AT-SPI2 event processed, every speech utterance generated, and every D-Bus call made. This log is invaluable for understanding why Orca is or is not reading a particular widget correctly.

**Test from Orca's perspective**: Enable Orca via GNOME Settings → Accessibility → Screen Reader, then navigate your application using Orca keyboard commands. The flat review commands (Orca + semicolon to read by line, Orca + comma for character, Orca + period for word) will reveal exactly what Orca sees.

**Automated testing**: `pyatspi2` can drive an application's accessible tree programmatically:

```python
import pyatspi
import time

def find_button(desktop, name):
    for app in desktop:
        for i in range(app.childCount):
            child = app.getChildAtIndex(i)
            if (child.getRole() == pyatspi.ROLE_PUSH_BUTTON
                    and child.name == name):
                return child
    return None

desktop = pyatspi.Registry.getDesktop(0)
time.sleep(1)  # let applications register

button = find_button(desktop, "OK")
if button:
    action = button.queryAction()
    action.doAction(0)  # "click"
```

This pattern is also used by `dogtail`, a higher-level Python test framework built on pyatspi2.

---

## Roadmap

### Near-term (6–12 months)

- **Newton Wayland protocol stabilisation**: GNOME's Newton project — funded through the Sovereign Tech Fund — has a working prototype using a new Wayland protocol to push per-surface accessibility trees from applications to Mutter, with Orca communicating back via a new D-Bus interface. The immediate goal is to stabilise these interfaces and ship them in a GNOME release so that Orca on Wayland no longer requires DE-specific hacks. [Source](https://blogs.gnome.org/a11y/2024/06/18/update-on-newton-the-wayland-native-accessibility-project/)
- **GNOME DE-specific keyboard interception consolidation**: The ad-hoc D-Bus keyboard interception API that landed in GNOME 48 and KDE Plasma 6.4 is expected to be superseded or wrapped by the Newton protocol's key-intercept mechanism, reducing the number of parallel interception paths screen readers must handle. [Source](https://blogs.gnome.org/tbernard/2025/04/11/gnome-stf-2024/)
- **COSMIC desktop AT-SPI extension**: System76's COSMIC compositor has already published `cosmic-atspi-unstable-v1`, a Wayland protocol extension for accessibility that mirrors Newton's push-based model. Further iteration on this protocol is expected as COSMIC matures toward a stable release. [Source](https://wayland.app/protocols/cosmic-atspi-unstable-v1)
- **AccessKit AT-SPI2 Rust adapter**: The AccessKit project's Rust-based AT-SPI2 platform adapter — which uses AccessKit's Chromium-derived node schema — is being extended to cover more role types and to integrate with the Newton-style per-surface tree model. This would let non-GTK Rust applications (including terminal emulators like Alacritty and Ghostty) expose accessibility trees without depending on GTK internals. [Source](https://gnome.pages.gitlab.gnome.org/at-spi2-core/devel-docs/new-protocol.html)
- **xdg-desktop-portal AT shortcuts portal**: A pending proposal at `flatpak/xdg-desktop-portal` to add `org.freedesktop.portal.AT.Shortcuts` would give sandboxed AT clients a portal-mediated path to keyboard interception, bypassing the need for DE-specific hacks. Near-term work focuses on reaching agreement on the API surface. [Source](https://github.com/flatpak/xdg-desktop-portal/issues/1046)

### Medium-term (1–3 years)

- **Newton plug-and-socket for web content**: The current Newton architecture assumes one accessibility tree per Wayland surface, but lacks an equivalent to AT-SPI2's plug-and-socket mechanism that allows out-of-process subtrees (such as a web rendering engine's DOM accessibility tree) to be grafted into a parent surface's tree. Design work on this sub-tree delegation model is identified as a prerequisite for full browser accessibility under Newton. Note: needs verification on current design discussions.
- **Standardised Wayland accessibility protocol**: The GNOME-specific and COSMIC-specific Wayland protocol extensions are expected to feed into a Wayland protocols repository proposal for a cross-compositor accessibility extension, analogous to how `xdg-output` was extracted from GNOME-specific origins. [Source](https://github.com/splondike/wayland-accessibility-notes/blob/main/README.md)
- **Terminal emulator AT-SPI2 adoption**: GPU-accelerated terminals (Ghostty, WezTerm, Alacritty) are tracking upstream AT-SPI2 and AccessKit developments; their respective issue trackers contain open requests for AT-SPI2 support. Medium-term adoption depends on AccessKit's Rust adapter reaching sufficient API stability to remove the dependency on GTK widget scaffolding. Note: needs verification on specific timelines.
- **Flatpak and sandboxed application accessibility**: Currently, applications running inside Flatpak sandboxes cannot reach the accessibility bus without special portal support. A medium-term goal is to extend the XDG desktop portal so that sandboxed apps can register their AT-SPI2 trees via the portal, with the portal acting as an untrusted relay. This is a prerequisite for accessibility in the broader containerised-app ecosystem.

### Long-term

- **Newton as the default Linux accessibility protocol**: The long-term architectural goal is for Newton (or a standardised descendant) to replace AT-SPI2 as the primary accessibility mechanism on Wayland desktops, with AT-SPI2 retained only as a compatibility shim for legacy applications. This transition would require GTK, Qt, Electron, and web engines all shipping Newton-native providers. [Source](https://gnome.pages.gitlab.gnome.org/at-spi2-core/devel-docs/new-protocol.html)
- **Unified key-intercept surface in the kernel input stack**: The current per-compositor, per-DE key interception models (GNOME D-Bus interface, Newton D-Bus protocol, COSMIC Wayland extension) may eventually converge on a kernel-level or compositor-protocol-level mechanism that lets screen readers register global key intercepts through a well-defined, security-audited interface, rather than relying on compositor-specific IPC. Note: needs verification on whether any kernel-level proposals exist.
- **Real-time magnification via compositor capture protocol**: Long-term, the GNOME Shell magnifier's privileged framebuffer access is expected to be replaced by a general compositor-level screen capture protocol that grants magnifier tools access to the composited output at full frame rate without requiring shell-plugin privileges. This would allow third-party magnifiers to run as regular Wayland clients. Note: needs verification on specific protocol proposals.

---

## 10. Integrations

**Chapter 20 (Wayland Protocol Fundamentals)**: The security model described there — no global window enumeration, keyboard events delivered only to the focused surface — is precisely the architectural decision that creates the accessibility gap described in Section 4. The `keyboard-shortcuts-inhibit-unstable-v1` protocol discussed in Chapter 20 is examined in Section 4.2 here for its (in)ability to help screen readers.

**Chapter 21 (Building Compositors with wlroots)**: Compositor authors using wlroots should note that neither the GNOME-specific keyboard interception D-Bus interface nor Newton's Wayland protocol is part of wlroots. A wlroots-based compositor that wants to support Orca must implement the same D-Bus interface that Mutter and KWin expose, or wait for a standardised approach. The `zwlr_input_inhibit_manager_v1` wlroots protocol provides global input inhibition but is designed for lockscreens, not for per-shortcut screen-reader access.

**Chapter 22 (Production Compositors: Mutter and KWin)**: Mutter's GNOME 48 keyboard interception interface and KWin's KDE 6.4 implementation are the specific compositor-side changes that unblock Orca on Wayland today. Chapter 22 covers the compositor architecture that makes these D-Bus interfaces feasible to implement.

**Chapter 39 (Qt and GTK GPU Rendering)**: The `GtkATContext` and `QSpiAccessibleBridge` implementations discussed in Section 3 exist within the same toolkit rendering pipeline described in Chapter 39. Understanding how GTK4's GL/Vulkan backend renders widgets helps explain why accessibility for GPU-rendered content (Ghostty, Alacritty) is harder: the pixel content is on the GPU, not in a retained widget object model.

**Chapter 132 (Wayland Security)**: The same security boundaries analysed in Chapter 132 — no cross-client pixel access, no global keyboard grabs — are the root cause of the accessibility gap examined here. The two chapters together illustrate the fundamental tension between security and observability in Wayland's design.

---

*Sources consulted for this chapter:*

- [AT-SPI2 specification — freedesktop.org](https://www.freedesktop.org/wiki/Accessibility/AT-SPI2/)
- [at-spi2-core architecture documentation](https://gnome.pages.gitlab.gnome.org/at-spi2-core/devel-docs/architecture.html)
- [org.a11y.atspi.Accessible interface reference](https://gnome.pages.gitlab.gnome.org/at-spi2-core/devel-docs/doc-org.a11y.atspi.Accessible.html)
- [org.a11y.atspi.Cache interface reference](https://gnome.pages.gitlab.gnome.org/at-spi2-core/devel-docs/doc-org.a11y.atspi.Cache.html)
- [org.a11y.atspi.Registry interface reference](https://gnome.pages.gitlab.gnome.org/at-spi2-core/devel-docs/doc-org.a11y.atspi.Registry.html)
- [org.a11y.atspi.Socket interface reference](https://gnome.pages.gitlab.gnome.org/at-spi2-core/devel-docs/doc-org.a11y.atspi.Socket.html)
- [at-spi2-core bus README — GNOME/at-spi2-core](https://github.com/GNOME/at-spi2-core/blob/main/bus/README.md)
- [Accessibility in GTK 4 — GTK Development Blog](https://blog.gtk.org/2020/10/21/accessibility-in-gtk-4/)
- [GTK 4.14 accessibility improvements — GTK Development Blog](https://blog.gtk.org/2024/03/08/accessibility-improvements-in-gtk-4-14/)
- [GTK4 accessibility section — GNOME developer docs](https://docs.gtk.org/gtk4/section-accessibility.html)
- [Accessibility in Wayland — LWN.net](https://lwn.net/Articles/980811/)
- [Enhancing screen-reader functionality in modern GNOME — LWN.net](https://lwn.net/Articles/1025127/)
- [Update on Newton, the Wayland-native accessibility project — GNOME Accessibility blog](https://blogs.gnome.org/a11y/2024/06/18/update-on-newton-the-wayland-native-accessibility-project/)
- [GNOME STF 2024 Project Report (Newton outcomes)](https://blogs.gnome.org/tbernard/2025/04/11/gnome-stf-2024/)
- [Wayland accessibility notes — splondike/wayland-accessibility-notes](https://github.com/splondike/wayland-accessibility-notes)
- [keyboard-shortcuts-inhibit-unstable-v1 — Wayland Explorer](https://wayland.app/protocols/keyboard-shortcuts-inhibit-unstable-v1)
- [org.freedesktop.portal.AT.Shortcuts proposal — flatpak/xdg-desktop-portal#1046](https://github.com/flatpak/xdg-desktop-portal/issues/1046)
- [Qt AT-SPI2 bridge — KDE Community Wiki](https://community.kde.org/Accessibility/qt-atspi)
- [Qt qtbase SPI bridge source](https://code.qt.io/cgit/qt/qtbase.git/log/src/gui/accessible/linux/qspiaccessiblebridge.cpp)
- [Accerciser introduction — GNOME Help](https://help.gnome.org/users/accerciser/stable/introduction.html.en)
- [Orca — Debian Wiki](https://wiki.debian.org/Accessibility/Orca)
- [Ghostty screen reader discussion #2351](https://github.com/ghostty-org/ghostty/discussions/2351)
- [WezTerm accessibility issue #913](https://github.com/wezterm/wezterm/issues/913)
- [GNOME Shell magnifier — GNOME Wiki](https://wiki.gnome.org/Projects/GnomeShell/Magnification)
- [KDE Plasma Wayland and accessibility — KDE Blogs](https://blogs.kde.org/2025/06/14/this-week-in-plasma-wayland-pip-and-accessibility/)
- [Wayland protocols accessibility issue #65 — gitlab.freedesktop.org](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/issues/65)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
