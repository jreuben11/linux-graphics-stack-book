# Chapter 39i: Desktop Framework Comparisons

**Audiences**: Systems developers choosing between GTK, Qt, or COSMIC as a target platform;
graphics application developers who need to understand how object systems, rendering architectures,
and compositor implementations relate to each other; kernel developers who encounter the overlap
between kernel object management and userspace toolkit patterns.

---

## Table of Contents

- [Overview](#overview)
- [1. Object Systems: GObject vs QObject vs kernel kobject](#1-object-systems-gobject-vs-qobject-vs-kernel-kobject)
  - [Origins and Purpose](#origins-and-purpose)
  - [Object Identity and Type Registration](#object-identity-and-type-registration)
  - [Ownership and Lifetime](#ownership-and-lifetime)
  - [Signals and Event Notification](#signals-and-event-notification)
  - [Properties and Attributes](#properties-and-attributes)
  - [Comparison Summary](#comparison-summary)
  - [Design Philosophy and Tradeoffs](#design-philosophy-and-tradeoffs)
- [2. Desktop Environments: COSMIC vs GNOME vs KDE vs elementary](#2-desktop-environments-cosmic-vs-gnome-vs-kde-vs-elementary)
  - [Architecture](#architecture)
  - [Display Stack](#display-stack)
  - [Accessibility](#accessibility)
  - [Portals and Sandboxing](#portals-and-sandboxing)
  - [Customisation and Theming](#customisation-and-theming)
  - [Platform and Ecosystem](#platform-and-ecosystem)
  - [Summary](#summary)
  - [Application Development Model: QML+C++ vs GJS-over-C vs Rust](#application-development-model-qmlc-vs-gjs-over-c-vs-rust)
  - [Rust UI Frameworks: iced/libcosmic vs gtk-rs vs cxx-qt](#rust-ui-frameworks-icedlibcosmic-vs-gtk-rs-vs-cxx-qt)
- [Integrations](#integrations)

---

## Overview

This chapter collects cross-framework comparisons that would fragment any single-toolkit chapter.
Section 1 examines the three C/C++ object systems that appear throughout the Linux graphics stack,
analysing where they agree and where they diverge:

- **GObject** (GTK)
- **QObject** (Qt)
- `struct kobject` (kernel)

Section 2 compares the four major Linux desktop environments as integrated stacks,
covering display pipelines, accessibility, portals, theming, and the application development model.

---

## 1. Object Systems: GObject vs QObject vs kernel kobject

Three major C/C++ systems in the Linux stack solve similar problems — object identity, lifetime
management, and event notification — but at different layers and with strikingly different
approaches. Understanding where they agree and where they diverge is useful both for reading kernel
and toolkit source and for choosing the right abstraction when writing a new subsystem.

### Origins and Purpose

| | GObject | QObject | `struct kobject` |
|---|---|---|---|
| Layer | Userspace — GTK/GNOME toolkit | Userspace — Qt toolkit | Kernel — device model / sysfs |
| Language | C | C++ | C |
| Problem solved | OOP, signals, properties for C | Reflection, signals/slots, properties for C++ | Refcounting, sysfs representation, hotplug events |
| Code generation | None — runtime registration only | `moc` generates `moc_*.cpp` at build time | None |

### Object Identity and Type Registration

**GObject** registers types at runtime via `g_type_register_static()` — typically invoked once per process by `get_type()` on first use. The type system lives entirely in the GLib heap; no compiler sees it. A `GType` is a `gulong` identifier handed out at registration time. `G_DEFINE_TYPE` (and its variants) generates the `get_type()` function and the `instance_init`/`class_init` boilerplate. Because everything is runtime, the same C library is introspectable from Python, Rust, Lua, or JavaScript without recompilation — only a `.gir` file is needed (Ch39c §9.10).

**QObject** splits the work: the programmer writes `Q_OBJECT` in the class declaration and `moc` generates a `moc_MyClass.cpp` that defines a `QMetaObject` — a static, compile-time table of methods, properties, signals, enums, and class hierarchy. `qobject_cast<T*>(obj)` performs a safe downcast by walking the `QMetaObject` chain; it is O(depth) and type-checked at compile time via the function-pointer cast. Because the metadata is compiled in, it is not accessible across language boundaries without a binding generator (e.g., Shiboken for Python/Qt or cxx-qt for Rust).

**kobject** has no type system. `struct kobject` is a plain C struct embedded in a larger kernel object (e.g., `struct device`, `struct gendisk`). The embedding type is recovered via `container_of(ptr, struct device, kobj)` — a macro that subtracts the field's offset from the pointer. There is no runtime check; the programmer must know the containing type statically. `struct kobj_type` supplies per-type function pointers (`release`, `sysfs_ops`, `default_groups`) but is attached to the kobject by pointer rather than inheritance.

### Ownership and Lifetime

**GObject** uses **reference counting** as its primary ownership model. Every GObject has an atomic integer refcount. `g_object_ref()` increments, `g_object_unref()` decrements; when the count reaches zero, `dispose()` runs first (the place to drop references to other objects and disconnect signals), then `finalize()` frees memory. The two-phase teardown prevents use-after-free caused by signal callbacks firing during destruction. `GInitiallyUnowned` adds a **floating reference**: newly constructed objects start with a "floating" ref that `g_object_ref_sink()` converts to a normal ref, allowing `column.append(widget)` to transfer ownership without an explicit unref.

**QObject** uses **parent/child tree ownership** as its primary model. Passing a parent to a QObject constructor transfers ownership; `delete parent` recursively deletes all children in declaration order. This works well for widget trees where a `QWidget` owns its layout and the layout owns its items. When shared or cross-tree ownership is needed, `QSharedPointer<T>` and `QWeakPointer<T>` provide reference-counted smart pointers. The base `QObject` has no refcount — it is owned either by its parent or by the stack/heap directly.

**kobject** uses `kref` — a struct wrapping an `atomic_t` — for reference counting. `kobject_get()` increments the refcount; `kobject_put()` decrements it and, when zero, calls `kobj_type.release(kobj)`. The `release` callback is responsible for freeing the containing structure (via `container_of`). There is no parent/child ownership in the C++ sense: `kobject.parent` builds the sysfs directory hierarchy but does not transfer ownership; a child kobject does not hold a reference to its parent's memory.

```c
/* kernel kobject embed-and-release pattern */
struct my_device {
    struct kobject kobj;     /* must be first for container_of */
    int            value;
};

static void my_device_release(struct kobject *kobj)
{
    struct my_device *dev = container_of(kobj, struct my_device, kobj);
    kfree(dev);              /* free the enclosing struct */
}

static const struct kobj_type my_ktype = {
    .release   = my_device_release,
    .sysfs_ops = &kobj_sysfs_ops,
};
```

### Signals and Event Notification

**GObject signals** are fully dynamic. A signal is registered with `g_signal_new()`, specifying its name, parameter types as `GType`s, and a default handler (`class_closure`). Connections are made with `g_signal_connect(object, "signal-name", G_CALLBACK(handler), user_data)` — a string-based lookup that returns a `gulong` handler id for later disconnection. Signal emission calls the default handler then all connected handlers in connection order. Because connections hold a reference to the callback and user_data, disconnecting in `dispose()` before `finalize()` is critical to avoid dangling pointer callbacks.

**QObject signals and slots** are type-safe when connected via the function-pointer syntax introduced in Qt 5:

```cpp
connect(button, &QPushButton::clicked,
        this,   &MyWindow::onButtonClicked);
// Compile error if parameter types don't match — caught by the C++ compiler.
```

The older string-based `SIGNAL()`/`SLOT()` macros still work but defer type checking to runtime. Qt signals can cross thread boundaries: if sender and receiver live in different threads, Qt queues the signal as a `QMetaCallEvent` on the receiver's event loop — thread safety for free. Qt also supports **lambda slots**:

```cpp
connect(timer, &QTimer::timeout, this, [this]{ tick(); });
```

**Kernel notification** uses several independent mechanisms rather than a unified signal system:

- **Notifier chains** (`atomic_notifier_call_chain`, `blocking_notifier_call_chain`): a linked list of callbacks registered with `register_xxx_notifier()`; the caller invokes all registered callbacks synchronously or asynchronously.
- **sysfs uevents**: when a kobject is added, removed, or changed, `kobject_uevent(kobj, KOBJ_ADD)` broadcasts a netlink message that udev/systemd-udevd receives, triggering hotplug rules.
- **Completions**: `wait_for_completion()` / `complete()` — single-shot synchronisation between kernel threads.
- **Work queues**: deferred execution via `schedule_work()` / `queue_work()`.

There is no equivalent to GObject's per-object signal connection table or Qt's per-object slot registration.

### Properties and Attributes

**GObject properties** are declared as `GParamSpec` descriptors registered with `g_object_class_install_properties()`. A widget subclass overrides `set_property` and `get_property` vtable slots in its `GObjectClass`. Every property change that calls `g_object_notify()` fires a `notify::property-name` signal. This is the basis for `GBinding` (one-way property synchronisation) and Blueprint `bind` expressions (Ch39c §13.3).

**QObject properties** are declared with the `Q_PROPERTY` macro, which lists getter, setter, and `NOTIFY` signal. `moc` generates the read/write dispatch in `QMetaObject::property()`. Properties are accessible by name as `QVariant` via `QObject::property("name")`, enabling scripting and QML binding. QML's property binding engine (`QQmlBinding`) re-evaluates a binding expression whenever any `NOTIFY` signal it depends on fires.

**kobject attributes** are kernel files in `/sys`. An attribute is a `struct attribute` (name + permissions) paired with `show(kobj, attr, buf)` and `store(kobj, attr, buf, count)` callbacks in `struct sysfs_ops`. Reading `/sys/bus/pci/devices/0000:00:02.0/vendor` calls the `show` callback for the `vendor` attribute. There is no change notification within the kernel — userspace polls or uses inotify on sysfs files, or the driver calls `sysfs_notify()` to wake inotify waiters.

### Comparison Summary

| Dimension | GObject | QObject | `kobject` |
|-----------|---------|---------|-----------|
| Code generation | None; runtime `g_type_register_static()` | `moc` → `QMetaObject` at compile time | None |
| Ownership model | Reference counting; floating ref for widgets | Parent/child tree; `QSharedPointer` for shared | `kref` atomic refcount; `release()` callback |
| Teardown | Two-phase: `dispose()` then `finalize()` | C++ destructor; children deleted by parent | `kobj_type.release()` at refcount zero |
| Inheritance | Single via struct embedding; `GTypeInterface` | Single from `QObject`; no diamond | None — composition via struct embedding |
| Signal/event system | Dynamic by name; `GClosure`; `gulong` handler id | Typed function-pointer connect; thread-safe queuing; lambda slots | Notifier chains; sysfs uevents; completions; work queues |
| Properties | `GParamSpec` + `get/set_property` vtable + `notify::` | `Q_PROPERTY` + getter/setter/NOTIFY; `QVariant` dynamic access | `struct attribute` + `show()`/`store()` sysfs files |
| Type downcast | `G_TYPE_CHECK_INSTANCE_CAST`; GIR for bindings | `qobject_cast<T*>()` via `QMetaObject` | `container_of()` macro; no runtime check |
| Language bindings | GIR → Python, Rust, JS, Lua without recompile | Requires binding generator (Shiboken, cxx-qt) | N/A — kernel-only |
| Sysfs / hotplug | None | None | Core feature: kobject = sysfs directory + uevent |

The conceptual through-line is **reference counting for lifetime** (all three), **embedding for composition** (GObject struct-in-struct, kobject in device), and **separate notification mechanisms** (GObject signals vs Qt signals-and-slots vs kernel notifier chains). The most practically significant difference for application developers is the code-generation split: GObject's runtime-only approach enables dynamic language bindings at the cost of compile-time type safety on signal connections, while QObject's `moc`-generated metadata enables compile-time checking at the cost of a build step and the absence of automatic cross-language binding.

### Design Philosophy and Tradeoffs

Each system reflects the constraints and goals of its context. Understanding why each was designed as it was explains the bugs each makes easy to write and the capabilities each makes easy to add.

**GObject: maximum portability and bindability, at the cost of type safety**

GObject's core design decision is to do everything at runtime in C. This was the right choice for GTK in the mid-1990s: C was the common denominator for the GNOME platform, C++ compilers were inconsistent, and language bindings (Python, Perl, Java) were a priority from the beginning. The tradeoff is that the C type system cannot enforce correctness at the GObject API boundary. `g_signal_connect(obj, "clicked", G_CALLBACK(handler), data)` compiles without error regardless of whether `handler`'s parameter types match the `clicked` signal's actual parameters — the mismatch is a runtime crash or silent misbehaviour. Property names are strings; a typo is silent until `g_object_get` returns `NULL`. GObject's verbosity is also a consequence of its C heritage: expressing a single-level subclass requires `G_DEFINE_TYPE`, an instance struct, a class struct, `instance_init`, `class_init`, overridden vtable slots, and `GParamSpec` descriptors — tens of lines for what Rust or Python achieves in three.

The two-phase teardown (`dispose`/`finalize`) is an elegant solution to a subtle problem. Reference cycles — two objects each holding a ref to the other — are the most common GObject memory bug. `dispose()` is the place to break cycles: drop refs to other objects, disconnect signal handlers (which themselves hold refs). After `dispose()`, the object is logically dead but its memory is still valid. `finalize()` frees the memory. Separating the two phases means that a signal callback firing during teardown sees a dead-but-valid object rather than freed memory. The discipline required is real: forgetting to disconnect a signal in `dispose()` is a use-after-free waiting to happen.

The floating reference (`GInitiallyUnowned`) is ergonomic for widget construction. Without it, `gtk_box_append(box, gtk_button_new())` would leak the button: `gtk_button_new()` returns a ref, `gtk_box_append` takes ownership, and the caller's ref is never released. With floating refs, the button starts with a floating ref that `gtk_box_append` sinks — the caller's logical "I created this" intention is consumed by the ownership transfer with no explicit `g_object_unref` needed.

**QObject: compile-time safety and cross-thread transparency, at the cost of C++ purity**

Qt's `moc` is a deliberate departure from standard C++. In 1991 when Qt was designed, C++ had no reflection, no standard introspection, and template metaprogramming was not yet practical. Qt needed signals and slots that were type-safe, cross-thread transparent, and usable from scripting languages. The choices were: use macros (error-prone), use virtual dispatch (no introspection), or use a code generator. They chose the code generator and have paid the maintenance cost ever since.

`moc`'s main practical limitations:
- **Templates and `Q_OBJECT` don't mix.** A class template cannot itself use `Q_OBJECT`; only a fully specialised concrete class can. Workarounds exist but are ugly.
- **Build system coupling.** Every build system must run `moc` before compiling. CMake's `automoc` handles this, but it adds latency and occasional dependency-graph confusion.
- **Multiple inheritance.** `QObject` must be first in a multiple-inheritance list. Having two `QObject`-derived bases in one class is not supported — a design constraint that occasionally forces awkward composition patterns.

The parent/child ownership tree elegantly handles the 90% case — a widget tree where the window owns everything — but breaks down at the edges. A `QObject` that outlives its parent (stored in a `QVector`, returned from a factory, passed across a thread boundary) requires either `setParent(nullptr)` or a `QSharedPointer`. Forgetting either results in a double-delete (parent deletes the child, then the `QSharedPointer` also releases it) or a dangling pointer (parent deletes the child, QSharedPointer now points to freed memory). The Qt documentation addresses this extensively because it is a genuine frequent mistake.

Cross-thread signal delivery is Qt's most elegant feature: `emit signal()` in a sender thread automatically becomes a `QMetaCallEvent` queued on the receiver's `QEventLoop` when sender and receiver live in different threads. The programmer declares thread affinity (`QObject::moveToThread()`), not locking discipline. This is substantially easier to reason about than manual mutex usage for the common producer/consumer pattern.

**kobject: zero-overhead bookkeeping, at the cost of safety guarantees**

kobject's design is shaped by one constraint: it runs in kernel context. That means no `malloc` failure handling beyond returning `-ENOMEM`, no C++ exceptions, no garbage collector, and extreme caution around locking because some code paths run in atomic context (interrupt handlers, RCU read-side sections) where sleeping is forbidden.

`container_of` is zero-overhead — it compiles to a single pointer subtraction with no indirection — but it is also the system's main footgun. The macro trusts the programmer that the field name and containing type are correct. There is no runtime tag, no type identifier, and no check: `container_of(ptr, struct wrong_type, kobj)` compiles silently and produces undefined behaviour at runtime. This is acceptable in kernel code because the call sites are controlled and reviewed, but it means that new kobject-embedded subsystems require careful auditing.

The `release` callback discipline is the kobject equivalent of GObject's `dispose`/`finalize` pair. The rules are:
1. Every kobject that is initialized with `kobject_init()` must eventually call `kobject_put()` exactly once on every reference acquired with `kobject_get()`.
2. The `release` callback must free the enclosing structure — not just the kobject field — because the kobject is embedded, not heap-allocated independently.
3. `release` may not be called from atomic context because `kfree` can sleep.
Violating rule 2 leaks the enclosing struct while freeing nothing (the kobject field is embedded, so `kfree(kobj)` would free the wrong address). Violating rule 3 causes a `might_sleep()` assertion in debug kernels.

The sysfs attribute model is deliberately minimal: a file, a `show` function, a `store` function. This simplicity is a feature — a new driver attribute is ten lines — but it limits expressiveness. Attributes have a 4 KB page limit per read (the kernel's page size), making them unsuitable for large data. Complex subsystems work around this with binary attributes (`bin_attribute`) or by using debugfs instead.

**The moc decision in retrospect**

`moc` was a pragmatic solution to a 1991 problem. C++26 reflection will make it technically unnecessary — a sufficiently advanced C++26 compiler could generate `QMetaObject` data from class declarations without a separate tool. The Qt project is aware of this; experimental work on a moc-replacement using C++20 concepts and C++26 reflection has been prototyped. GObject took the opposite path (no codegen, maximum runtime flexibility) and was rewarded with automatic language bindings — a capability Qt has never matched without a dedicated binding generator (Shiboken, cxx-qt). With hindsight, both were correct given their goals: Qt prioritised C++ developer ergonomics and compile-time correctness; GObject prioritised ecosystem reach across languages.

kobject made no such tradeoff because it was never trying to be a general-purpose OOP system. Its design is the correct one for its domain: a minimal refcount + sysfs-directory primitive that composes into the device model. Adding GObject-style signals or QObject-style metadata to kobject would add overhead to every device object in the kernel for capabilities that kernel code doesn't need.

| | GObject | QObject | kobject |
|---|---|---|---|
| **Primary design goal** | OOP + language bindings for C | Type-safe signals + reflection for C++ | Refcount + sysfs for kernel C |
| **Worst common bug** | Signal/slot type mismatch (silent runtime) | Ownership confusion (double-free / dangling ptr) | `container_of` wrong type (UB); release from atomic context |
| **Most elegant feature** | GIR → automatic N-language bindings | Cross-thread signal queuing is transparent | `kref` + `container_of` = zero-overhead lifetime for any struct |
| **Biggest design tax** | Boilerplate for subclassing; no compile-time signal safety | `moc` build step; template+Q_OBJECT incompatibility | No type safety; 4 KB sysfs attribute limit |
| **Future direction** | gtk-rs / GIR-Rust reduces boilerplate; AccessKit bridges a11y | C++26 reflection may eventually replace moc | Stable; no planned redesign |

---

## 2. Desktop Environments: COSMIC vs GNOME vs KDE vs elementary

### Architecture

| Dimension | COSMIC | GNOME | KDE Plasma | elementary OS (Pantheon) |
|-----------|--------|-------|------------|--------------------------|
| Implementation language | Rust | C (Shell in JS) | C++ | Vala / C |
| Compositor | cosmic-comp (Smithay) | Mutter | KWin | Gala (on Mutter) |
| Compositor renderer | GLES (`GlowRenderer`) | GL / Vulkan (GSK) | GL / Vulkan (KWin OpenGL/Vulkan) | GL |
| Widget toolkit | libcosmic (on iced) | GTK4 + libadwaita | Qt 6 + Kirigami | GTK4 (Granite) |
| App GPU rendering | wgpu → Vulkan / GLES | GSK → Vulkan (Wayland) / GL | Qt RHI → Vulkan / GL | GSK → GL |
| Config system | cosmic-config (RON files) | GSettings / dconf (binary) | KConfig (INI files) | GSettings / dconf |
| IPC / shell protocol | D-Bus + Wayland ext. | D-Bus + GNOME-specific ext. | D-Bus + plasma-wayland-protocols | D-Bus + pantheon-wayland |
| Build system for apps | Cargo + Meson | Meson (Blueprint, GResource) | CMake + ECM | Meson / CMake |
| Preferred app language | Rust | C / Rust (gtk-rs) / Python / JS | C++ / QML / Rust | Vala / C |

### Display Stack

| Dimension | COSMIC | GNOME | KDE Plasma | elementary OS (Pantheon) |
|-----------|--------|-------|------------|--------------------------|
| HDR support | Planned; not yet in Epoch 1 | Stable — HDR switch (GNOME 48), brightness in Quick Settings (49), HDR screen sharing (50) | Stable and mature — simultaneous HDR + ICC profiles since Plasma 6.7 | Not supported |
| VRR (variable refresh rate) | Basic support in cosmic-comp | Stable since GNOME 50 (experimental from GNOME 46) | Stable since Plasma 6.3; per-monitor toggle | Limited |
| Fractional scaling | Supported (Wayland) | Stable since GNOME 50; improved Xwayland fractional scaling | Mature on Wayland; per-output in Display Settings | Limited |
| Color management | Planned (Smithay protocol in progress) | `wp-color-management-v2` (GNOME 50); wide-gamut SDR-native mode | ICC profiles + per-display KCM calibration (Plasma 6.7) | Not supported |
| Multi-monitor | Supported; arrangement via COSMIC Settings | Full support; per-monitor scale; mirror / extend | Full support via KScreen; per-monitor refresh / scale / rotation | Basic support |
| Direct scan-out | Via Smithay DRM backend | Via Mutter DRM backend | Via KWin DRM backend | Via Gala/Mutter |

### Accessibility

| Dimension | COSMIC | GNOME | KDE Plasma | elementary OS (Pantheon) |
|-----------|--------|-------|------------|--------------------------|
| Accessibility framework | AT-SPI2 via AccessKit (libcosmic) | AT-SPI2 (GTK4 native + AccessKit on Win/macOS) | AT-SPI2 via Qt Accessibility | AT-SPI2 (GTK4) |
| Screen reader | Early — Orca-compatible via AT-SPI2; enabled by default in Setup since 1.0.2 | Orca (mature; redesigned in GNOME 50 with auto language switching, new prefs UI) | Basic AT-SPI2 compatibility; Orca works but integration is less complete | Basic |
| Keyboard navigation | Partial; improving in 1.0.x | Mature — full GTK4 focus model; accessible roles on all built-ins | Mature — Qt accessibility with full keyboard navigation | Moderate |
| Reduced motion | Partial | System-wide option since GNOME 50 | Configurable per-effect in KWin | Not exposed system-wide |

### Portals and Sandboxing

| Dimension | COSMIC | GNOME | KDE Plasma | elementary OS (Pantheon) |
|-----------|--------|-------|------------|--------------------------|
| xdg-portal backend | xdg-desktop-portal-cosmic | xdg-desktop-portal-gnome (comprehensive) | xdg-desktop-portal-kde (comprehensive) | xdg-desktop-portal-pantheon (basic) |
| Flatpak integration | COSMIC Store (Flatpak-first); portal support in progress | GNOME Software + Flathub; mature portal coverage | Discover (PackageKit + Flatpak + Snap); mature portal coverage | AppCenter (curated Flatpak); basic portals |
| App store / software center | COSMIC Store | GNOME Software | Discover | AppCenter |
| Sandboxed file access | FileChooser portal (basic) | FileChooser + Documents portal (mature) | FileChooser portal (mature) | FileChooser portal |
| Remote desktop / screen sharing | Not yet shipped; planned | gnome-remote-desktop (RDP + VNC; PipeWire-backed); hardware-accel in GNOME 50 | KRdp (RDP server); KRfb (VNC); PipeWire screencast | Not supported |

### Customisation and Theming

| Dimension | COSMIC | GNOME | KDE Plasma | elementary OS (Pantheon) |
|-----------|--------|-------|------------|--------------------------|
| Theming system | libcosmic themes (RON); accent colors; dark/light variants | libadwaita CSS + accent colors (GNOME 47+); GTK3 theming intentionally restricted | Plasma Themes + KColorScheme + Breeze; per-widget CSS in QML | elementary stylesheet; Granite-based accent |
| Custom app themes | libcosmic theme API (Rust) | libadwaita CSS custom properties; very limited third-party override | Full Qt stylesheet + QSS override possible | elementary stylesheet fork |
| Icon themes | Cosmic Icons (XDG icon theme) | Adwaita (symbolic-first) | Breeze (symbolic + full-color) | elementary Icons |
| Font configuration | COSMIC Settings font panel | GNOME Tweaks / Settings font panel | System Settings → Fonts + KFontConfig | elementary Settings |
| Shell extensions / plugins | COSMIC Applets (separate Rust processes) | GNOME Shell JS extensions (in-process; EGO review) | Plasmoids (QML in-process) + KWin scripting | Limited (Switchboard plugs only) |

### Platform and Ecosystem

| Dimension | COSMIC | GNOME | KDE Plasma | elementary OS (Pantheon) |
|-----------|--------|-------|------------|--------------------------|
| Session protocol | Wayland only | Wayland (X11 session removed in GNOME 50) | Wayland + X11 (X11 session removed in Plasma 6.8) | Wayland + X11 |
| XWayland | Supported | Supported | Supported | Supported |
| Mobile / convergent | Not targeting mobile | Phosh (separate GTK/wlroots shell; ~monthly cadence) | Plasma Mobile (active; tracks Plasma releases) | Not targeting mobile |
| Release cadence | Rolling point releases on Epoch 1 | 6-month (March / September; named releases) | 4-month (~3 releases/year; numbered) | Infrequent major releases |
| API stability | Unstable (pre-stable API commitment) | Stable (GTK4 / libadwaita LGPLv2.1) | Stable (Qt / KF6 LGPLv3 + GPLv2) | Stable (Granite LGPL) |
| Primary sponsor | System76 | GNOME Foundation | KDE e.V. | elementary, Inc. |
| Governance | System76-led; community PRs welcome | Board + Release Team committee; FOSS | Board of KDE e.V.; meritocracy | elementary, Inc.-led |

### Summary

The table reveals four distinct engineering philosophies. **KDE Plasma** offers the deepest configurability and the most mature display stack (HDR, VRR, ICC, multi-monitor all stable) at the cost of a large C++/QML codebase. **GNOME** prioritises coherence and Flatpak-first application distribution, with a rapidly maturing Wayland pipeline (wp-color-management-v2, VRR, hardware-accelerated remote desktop all landing in GNOME 49–50) and the strongest accessibility story. **COSMIC** is the only desktop written entirely in Rust, making memory safety a first-class architectural property; it is also the only one where the compositor renderer (GLES) and the application renderer (wgpu/Vulkan) are architecturally distinct stacks, and the only one with auto-tiling as a built-in mode. Its display stack (HDR, color management) and portal coverage trail GNOME and KDE while the 1.0.x series matures. **elementary OS** targets the most curated and opinionated experience, but its display stack and portal coverage are the least developed of the four.

From a graphics-stack perspective, GNOME and KDE are the reference targets for Wayland protocol adoption — new protocols (color management, explicit sync, input capture) typically land in Mutter or KWin first. COSMIC is a useful stress-test for Smithay maturity and Rust-based compositor development. elementary is a useful reference for a GTK4 app running under a Mutter-derived compositor with minimal extensions.

### Application Development Model: QML+C++ vs GJS-over-C vs Rust

The three major desktops represent three distinct engineering philosophies for how application logic and UI description relate to each other.

#### KDE: C++ core, QML UI layer

The architecture separates application logic (C++) from declarative UI (QML). QML is a JavaScript-syntax language compiled by the Qt QML engine into a scene graph of `QQuickItem` subtypes; the engine uses a JIT-compiled V4 JavaScript runtime, making property bindings and animations fast. The C++ ↔ QML boundary is handled by `Q_PROPERTY` / `Q_INVOKABLE` macros, which the Meta-Object Compiler (`moc`) processes at build time to generate the signal/slot and property-binding machinery.

**Strengths.** QML is purpose-built for declarative UI — property bindings, state machines, and transitions are first-class language features. C++ carries zero FFI overhead for performance-critical code. Qt RHI abstracts Vulkan, Metal, and Direct3D without the application touching GPU code. The same codebase compiles on Linux, Windows, macOS, Android, and iOS. Tooling is mature: Qt Creator, `qmllint`, `qmlformat`, and the QML language server all understand the type system.

**Weaknesses.** QML's JavaScript layer is dynamically typed — `qmllint` catches many errors but compile-time type safety is weaker than Blueprint's GIR-validated schema. The Qt licensing split (LGPLv3 / GPLv2 vs. commercial) creates friction for proprietary applications. `moc` is a pre-compilation step that adds build complexity; it is being incrementally replaced by C++26 reflection. The QML runtime adds approximately 10 MB of baseline memory overhead.

#### GNOME: C GObject core, multiple language bindings via GIR

GTK widgets and the GObject type system are implemented in C. The GObject Introspection (GIR) pipeline — `g-ir-scanner` → `.gir` XML → `g-ir-compiler` → `.typelib` — automatically exposes every annotated GObject type to any language that has a GIR binding. GJS (Mozilla SpiderMonkey embedding) is used in GNOME Shell itself; `gtk-rs` (Rust) is now production-quality and used in core GNOME applications; PyGObject (Python) remains the easiest entry point. Application UI is described in Blueprint (`.blp`) files that compile to GtkBuilder XML, with signal handlers wired via GObject Introspection.

**Strengths.** GIR means the full GTK/GNOME API is available in any binding language without hand-written glue — a library author writes once in C and gets JS, Python, Rust, Lua, and Haskell bindings automatically. GTK4 + libadwaita provides the GNOME platform's native look-and-feel with no configuration. Blueprint adds compile-time type checking for property names and signal signatures. `gtk-rs` macros make Rust-based GNOME apps ergonomic and memory-safe. Flatpak integration and Flathub distribution are the most mature of the four desktops.

**Weaknesses.** GJS/SpiderMonkey is a browser JavaScript engine bolted onto a C object system — the mismatch shows in error messages, reference-cycle debugging, and the requirement to manually disconnect GObject signals to avoid leaks. The underlying GObject type system (`G_DEFINE_TYPE`, `g_signal_connect`, manual reference counting) is verbose C from the late 1990s; even `gtk-rs` inherits its structural complexity. Blueprint handles only the widget tree — bindings between application state and UI state still require GObject property machinery or manual signal connections, unlike QML's built-in reactive bindings.

#### COSMIC: Rust end to end

COSMIC collapses the language boundary entirely: both the compositor (`cosmic-comp`) and applications (`libcosmic`) are written in Rust. The iced runtime provides the reactive Elm-style update loop natively in Rust — no separate scripting layer, no FFI to C, no GIR pipeline. The `wgpu` renderer compiles to SPIR-V for Vulkan at build time. Type safety extends from application logic through widget state into GPU shader uniforms.

**Strengths.** Memory safety is guaranteed by the Rust compiler across the entire stack — compositor, toolkit, and applications — with no garbage-collector pause, no runtime overhead, and no `unsafe` escape hatch required for ordinary widget code. This is a qualitative difference from both GTK (where C code can misuse reference counts, double-free signal handlers, or corrupt GValue containers) and Qt (where C++ permits the same classes of bug, and QML adds a garbage-collected JS heap alongside the C++ object graph). The entire COSMIC stack — from DRM buffer allocation in `cosmic-comp` through wgpu command encoding to widget layout in libcosmic — is free-of-data-races by construction: the borrow checker enforces the invariants the compositor relies on without runtime locking.

The single-language architecture also means there is no legacy API surface to carry. GTK inherits thirty years of GObject conventions — `GtkWidget` has over two hundred virtual methods accumulated across API generations, `GdkPixbuf` persists in parallel to `GdkTexture`, and the GLib main loop predates `async`/`await` by two decades. Qt carries similar weight: `QWidget` coexists with `QQuickItem`, `QObject::connect` has four incompatible syntaxes across Qt versions, and `moc` exists entirely because C++ lacked reflection when Qt was designed in 1991. COSMIC starts from a clean design point: iced's `Widget` trait has a handful of methods, the `Application` trait has three, and the entire widget model fits comfortably in a single document. There is no deprecated API to accidentally reach for, no compatibility shim to route around, and no C ABI to stabilise across years.

The iced Elm model (a `Message` enum, an `update` function, a `view` function) is explicit and testable: the full application state is a Rust struct, every state transition is a pure function, and the view is a deterministic function of state. This makes unit testing of application logic straightforward without a running compositor. `cargo` handles the entire build graph, including wgpu SPIR-V shader compilation, without a separate preprocessing step like `moc` or `g-ir-scanner`.

**Weaknesses.** The API is pre-stable (breaking changes occur between releases). The application ecosystem is small compared to GNOME's Flathub catalogue or KDE's mature app suite. GIR-style automatic multi-language binding does not exist — a non-Rust application cannot easily use libcosmic. Accessibility coverage trails GTK4's AT-SPI2 integration.

#### Practical Guidance

| If you need… | Recommended choice |
|---|---|
| Cross-platform (Linux + Windows + Android) from one codebase | KDE / Qt |
| GNOME platform look-and-feel; Flatpak-first distribution | GNOME / GTK4 |
| Rust end-to-end; no C FFI boundary | COSMIC / libcosmic |
| Richest declarative animation and reactive binding model | KDE / QML |
| Strongest accessibility on Linux, most mature a11y tooling | GNOME / GTK4 |
| Most mature HDR + VRR display pipeline (mid-2026) | KDE (stable Plasma 6.7) |
| Most mature Flatpak portal coverage | GNOME |

From the graphics-stack perspective that anchors this book, all three converge at the same point: Qt RHI, GSK, and wgpu each emit Vulkan commands that travel through the same Mesa drivers, the same kernel DRM subsystem, and the same KMS atomic commit path to the display. The compositor differences — Mutter, KWin, cosmic-comp — are the subject of Part IV (Ch20–Ch22). The toolkit rendering differences — Qt RHI, GSK GskGpuRenderer, iced wgpu — map directly to the Vulkan and GL chapters in Part III.

### Rust UI Frameworks: iced/libcosmic vs gtk-rs vs cxx-qt

A growing number of Linux desktop applications are written in Rust, but "written in Rust" spans three very different architectural positions. Each framework's Rust code sits at a different layer of the stack, inherits a different legacy surface, and makes different tradeoffs.

#### Framework Overview

| Dimension | iced / libcosmic | gtk-rs | cxx-qt |
|-----------|-----------------|--------|--------|
| What is Rust replacing? | Everything — no C or C++ in the stack | The application layer above GTK4 (C stays) | The business-logic layer above Qt (C++ stays) |
| Rendering engine | wgpu (native Rust → Vulkan/GLES/Metal/DX12) | GSK (C, part of GTK4) | Qt RHI (C++, part of Qt 6) |
| Widget model | Functional / immutable; Elm Message+update+view | GObject subclassing via `glib::subclass` macros; `CompositeTemplate` + Blueprint | QObject subclassing in Rust via `#[cxx_qt::bridge]`; QML frontend |
| UI description language | Rust view function (returns `Element` tree) | Blueprint `.blp` → GtkBuilder XML; or programmatic GTK construction | QML (declarative) backed by Rust QObject properties |
| Type system bridging | Pure Rust traits | GIR → GObject reference-counted pointers wrapped in `glib::Object<T>` smart pointers | `cxx` bridge: Rust struct ↔ Q_PROPERTY; `UniquePtr<QObject>` / `Pin<&mut T>` |
| Memory model | Rust ownership throughout; no GC | GObject refcounting wrapped by Rust (`clone()` increments refcount); some `unsafe` in bindings | Rust ownership for Rust data; Qt parent/child ownership for QObject tree; cxx handles ABI boundary |
| Accessibility | AccessKit → AT-SPI2 (Linux), native (Win/macOS) | AT-SPI2 via GTK4 (mature; best on Linux) | AT-SPI2 via Qt Accessibility (moderate) |
| Platform reach | Linux (Wayland/X11), Windows, macOS, Web (WASM) | Linux (primary), Windows, macOS | Linux, Windows, macOS, Android, iOS (wherever Qt runs) |
| Build tooling | `cargo` only; SPIR-V shaders compiled via `wgpu`/`naga` | `cargo` + Meson (Blueprint, GResource); `g-ir-scanner` for any new C libraries | `cargo` + CMake; `moc` still required for the Qt side |
| Maturity | Pre-1.0 (iced); COSMIC Epoch 1 (libcosmic) | Production — used in GNOME core apps (Papers, Loupe, Decoder) | Production — used in KDE apps (KDE Connect, NeoChat, Merkuro) |

#### iced / libcosmic

iced is a pure-Rust reactive UI library with no C or C++ in its stack. The `Widget` trait is a Rust trait; the renderer is `iced_wgpu`, which calls wgpu directly; state management follows the Elm architecture — the entire application state lives in a single Rust struct, every user interaction is a `Message` value, and `update` is a pure function that produces a new state (see Ch39e §1 for the full architecture). libcosmic extends iced with the COSMIC widget set and theming system (Ch39f §3–§5).

The absence of any C boundary means the borrow checker's guarantees extend all the way from widget layout to GPU command encoding. There are no `Arc<Mutex<_>>` wrappers around widget state to placate a foreign type system, no `unsafe` blocks to call GTK or Qt functions, no GObject refcount cycles to audit. The tradeoff is breadth: iced's widget catalogue is smaller than GTK4's or Qt's, the API is pre-1.0 (breaking changes between releases), and there is no GIR-style automatic binding to expose iced widgets to Python or JavaScript.

#### gtk-rs

gtk-rs is the official Rust binding to GTK4, GLib, GIO, GDK4, and libadwaita, generated from GIR type definitions via the `gir` tool. All GTK4 widgets, GObject signals, and GLib async primitives are available in Rust with idiomatic wrappers.

The binding wraps GObject pointers in `glib::Object<T>` smart pointers that increment/decrement the GObject refcount on `clone`/`drop`. This is safe for ordinary use, but the model is reference-counting rather than Rust ownership — a signal closure that captures a widget clone can keep the widget alive past its logical lifetime, producing a reference cycle. The `glib::subclass` module lets Rust code define new GObject types with proper `impl ObjectImpl`, `impl WidgetImpl`, etc. implementations; `#[derive(CompositeTemplate)]` wires Blueprint-defined template children to Rust struct fields at compile time.

For GNOME-target applications, gtk-rs is the production choice: libadwaita-rs exposes `AdwApplicationWindow`, `AdwPreferencesDialog`, `AdwNavigationSplitView`, and the full Adwaita widget set; Blueprint provides compile-time property and signal validation; `cargo` and Meson cooperate cleanly. The rendering path is GSK → Vulkan (on Wayland) — the same path as a C GTK4 application.

The inherited complexity is the C GObject type system itself. Even with Rust macros smoothing the surface, the concepts underneath — `GType` registration, property `ParamSpec` descriptors, signal marshalling, `GValue` variant containers, `dispose`/`finalize` split — all originate in a C design from the 1990s. A gtk-rs application developer must understand GObject ownership and signal connection semantics that have no parallel in idiomatic Rust.

#### cxx-qt

cxx-qt, built on the `cxx` C++/Rust interop library, allows Rust structs to be exposed as `QObject` subclasses consumable from QML. The Rust side defines a struct annotated with `#[cxx_qt::bridge]`; the macro generates the `moc`-compatible C++ header and the cxx bridge glue. Q_PROPERTY values stored in the Rust struct are readable and writable from QML; Rust methods are invokable via `Q_INVOKABLE`; Qt signals can be emitted from Rust and connected to QML slots.

```rust
#[cxx_qt::bridge]
mod ffi {
    #[cxx_qt::qobject]
    #[derive(Default)]
    struct Counter {
        #[qproperty]
        count: i32,
    }

    impl qobject::Counter {
        #[qinvokable]
        pub fn increment(self: Pin<&mut Self>) {
            let new = self.count() + 1;
            self.set_count(new);
        }
    }
}
```

The QML frontend consumes `Counter` as a normal `QObject` with a `count` property and an `increment()` invokable — the Rust implementation is invisible to QML. This makes cxx-qt the natural path for KDE teams that want to harden existing QML applications with Rust business logic without rewriting the UI layer.

The tradeoff is that Qt remains a C++ dependency: `moc` still runs, CMake is still required alongside `cargo`, and the cxx bridge introduces `UniquePtr<T>` and `Pin<&mut T>` pointer types at the boundary that have no equivalent in ordinary Rust code. Memory safety holds within the Rust side; the Qt object graph on the C++ side follows Qt's parent/child ownership model, which is managed manually.

#### Choosing Between the Three

| If you are building… | Recommended framework |
|----------------------|-----------------------|
| A COSMIC desktop application | iced / libcosmic |
| A GNOME / Flatpak application targeting Flathub | gtk-rs + libadwaita-rs |
| A KDE application with existing QML UI | cxx-qt |
| A cross-platform app (Linux + Windows + Android) in Rust | cxx-qt (via Qt's platform reach) or iced (via winit) |
| A greenfield app with no desktop integration requirements | iced (smallest dependency surface) |
| Strongest a11y on Linux today | gtk-rs (GTK4 AT-SPI2 maturity) |
| Full Rust ownership semantics with no C/C++ boundary | iced / libcosmic |
| Largest available widget catalogue | gtk-rs (full GTK4 + libadwaita) |

All three rendering paths ultimately converge on the same Mesa Vulkan driver: iced via `wgpu` → `VkCommandBuffer`; gtk-rs via GSK's Vulkan renderer → `VkCommandBuffer`; cxx-qt via Qt RHI → `VkCommandBuffer`. The differentiation is entirely above the Vulkan API boundary.

---

## Roadmap

### Near-term (6-12 months)
- **The Rust toolkit bindings keep shipping incremental releases rather than architectural changes.** cxx-qt continues frequent point releases (v0.9.x added QML image-provider bindings and fixed silent constructor failures) and gtk4-rs continues its own regular release cadence — both narrow API-coverage gaps without changing their fundamental architecture (Rust-behind-`QObject` vs Rust-behind-`GObject`, per the §2 Rust framework comparison table). [Source](https://github.com/KDAB/cxx-qt/releases/tag/v0.9.0) [Source](https://github.com/gtk-rs/gtk4-rs)
- **AccessKit's AT-SPI2 bridge continues incremental hardening.** The `accesskit_unix` crate that underlies libcosmic's and iced's accessibility path (§2 Accessibility table) ships regular point releases alongside the Windows and macOS backends, narrowing — but not yet closing — the accessibility-maturity gap with GTK4's native AT-SPI2 integration and Qt's Accessibility module. [Source](https://github.com/AccessKit/accesskit/releases)
- **Wayland color management remains a staging protocol.** `color-management-v1` is still listed under "Staging" rather than stable in the wayland-protocols repository, which is the structural reason §2's Display Stack table shows GNOME and KDE Plasma with mature, shipped color management while COSMIC's Smithay implementation and elementary's Pantheon are still catching up. [Source](https://wayland.app/protocols/color-management-v1)

### Medium-term (1-3 years)
- **C++26 static reflection (P2996) is now formally part of the C++26 working draft**, giving Qt a standards-based mechanism that could eventually reduce its dependence on the `moc` code-generation step discussed in the Design Philosophy comparison (§1). Adoption inside shipping Qt tooling will lag well behind compiler support, since Qt must keep working across the range of C++ standard versions its supported compilers implement — there is no published Qt commitment to a moc-removal date. [Source](https://github.com/cplusplus/papers/issues/1668)
- **The kernel's Rust device-model abstractions (`kernel::device`) now sit alongside `kobject`/`kref` rather than replacing them.** The in-tree `kernel::device` module wraps `struct device` — the kobject-embedding pattern described in §1 — in a Rust-safe API for driver authors, giving the kernel a fourth object-lifetime idiom (safe Rust ownership over the same `kref`-counted C struct) without altering the underlying kobject/sysfs mechanics. [Source](https://rust.docs.kernel.org/kernel/device/index.html)
- **Portal and Wayland-protocol coverage across the four desktops keeps narrowing rather than widening.** As staging protocols such as color management graduate to stable and land in Smithay, wlroots, Mutter, and KWin, the display-stack and portal-coverage gaps that separate COSMIC and elementary from GNOME and KDE in §2's tables are the kind of gap that closes with protocol maturity rather than one that is architectural.

### Long-term
- **`kobject`'s C ABI is likely to persist as the substrate that Rust abstractions wrap rather than replace.** No kernel roadmap commits to restructuring the device model, and the sysfs/uevent ABI that userspace (udev, systemd) depends on makes a breaking redesign unlikely regardless of how much driver code eventually moves to Rust — the comparison in §1 should remain valid as a description of the kernel's own object system for the foreseeable future.
- **The GObject/QObject architectural split — runtime introspection versus compile-time code generation — is likely to persist even if C++26 reflection reduces `moc`'s technical necessity**, because the split originates in each project's founding priorities (GNOME's multi-language binding goal versus Qt's compile-time type safety goal), not in a C++ tooling limitation that reflection alone removes.
- **No signal points toward the four desktop environments converging on a shared toolkit or object system.** libcosmic/Rust, GTK4/GObject, Qt6+QML/`QObject`, and Pantheon/Granite represent four independent, actively maintained application-development investments; unifying them would mean rewriting one desktop's entire application ecosystem onto another's object system, a cost none of the four projects' public roadmaps suggest they are pursuing.

## Integrations

- **GObject type system (Ch39c §9)**: the GObject details in §1 — `G_DEFINE_TYPE`, instance/class struct layout, signal marshalling, `GParamSpec` — are defined and exercised there; §1 here focuses on the cross-system comparison.
- **Qt meta-object system (Ch39a §2)**: `QMetaObject`, `Q_PROPERTY`, `moc` output, and `QObject` ownership semantics are covered in detail there; §1 here compares them to GObject and kobject.
- **kernel kobject and device model (Ch1, Ch2)**: kobject is the fundamental building block of the Linux device model and sysfs hierarchy; §1 here places it in comparison with userspace object systems.
- **COSMIC compositor and libcosmic (Ch39f)**: cosmic-comp, the Smithay-based compositor, and libcosmic's full Application API, theming, and applet system are the subject of Ch39f; §2 here compares COSMIC as a desktop to GNOME and KDE.
- **GNOME Shell and Mutter (Ch39d)**: Mutter's compositor rendering path, GNOME Shell extensions, and the GJS application model are covered there; §2 here compares the desktop as a platform.
- **KWin and KDE Frameworks (Ch39b)**: KWin's scene graph, KDE Frameworks widget ecosystem, and plasma-wayland-protocols are in Ch39b; §2's KDE column summarises the display stack state.
- **iced architecture (Ch39e)**: the Elm architecture, widget trait, `iced_wgpu` rendering pipeline, and Wayland integration are in Ch39e §1–§4; the Rust framework comparison in §2 cross-links there.
- **Wayland compositor comparisons (Ch20–Ch22)**: Mutter, KWin, and cosmic-comp are each compositor implementations of the stack described in Part IV; §2's Display Stack table references their respective DRM backend paths.
- **xdg-desktop-portal (Ch23)**: the portal backends listed in §2's Portals table are the subject of Ch23.
- **Flatpak (Ch111)**: the Flatpak integration column in §2 maps to the packaging and sandboxing paths described in Ch111.
