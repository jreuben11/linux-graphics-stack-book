# Chapter 198: D-Bus, dbus-broker, and Modern Linux IPC: Varlink, zbus, hyprwire, and BUS1

**Target audiences**: Systems and driver developers who need to call or expose D-Bus APIs from daemons, compositors, and kernel-adjacent code; graphics application developers integrating seat management, colour management, screen capture, or portal services; compositor authors choosing between D-Bus, Varlink, and emerging alternatives.

---

## Table of Contents

1. [IPC in the Linux Graphics Stack — Why It Matters](#1-ipc-in-the-linux-graphics-stack--why-it-matters)
2. [D-Bus Architecture and Wire Protocol](#2-d-bus-architecture-and-wire-protocol)
3. [dbus-daemon vs dbus-broker](#3-dbus-daemon-vs-dbus-broker)
4. [Programming D-Bus: sd-bus (C)](#4-programming-d-bus-sd-bus-c)
5. [Programming D-Bus: GLib/GIO (gdbus-codegen)](#5-programming-d-bus-glibgio-gdbus-codegen)
6. [Programming D-Bus: Python (dasbus)](#6-programming-d-bus-python-dasbus)
7. [Programming D-Bus: Rust — zbus](#7-programming-d-bus-rust--zbus)
8. [Varlink: systemd's Simpler IPC](#8-varlink-systemds-simpler-ipc)
9. [Android Binder: IPC for the Graphics HAL](#9-android-binder-ipc-for-the-graphics-hal)
10. [hyprwire and hyprtavern: The Hyprland IPC Ecosystem](#10-hyprwire-and-hyprtavern-the-hyprland-ipc-ecosystem)
11. [BUS1: In-Kernel IPC in Rust](#11-bus1-in-kernel-ipc-in-rust)
12. [Comparison and Selection Guide](#12-comparison-and-selection-guide)
13. [Roadmap](#13-roadmap)
14. [Integrations](#14-integrations)

---

## 1. IPC in the Linux Graphics Stack — Why It Matters

IPC (inter-process communication) is the connective tissue that holds the Linux graphics stack together. Despite decades of GPU driver improvements, Wayland protocol evolution, and Mesa optimisation, the moment a compositor needs to claim a DRM device node or a game needs to boost its scheduler priority, it must cross a process boundary — and in practice it almost always does so over D-Bus.

The canonical example is seat management (covered in depth in Chapter 21). A Wayland compositor running as an unprivileged user cannot open `/dev/dri/card0` directly because the device is owned by `root:video` or managed through udev's `uaccess` tagging. Instead, it calls `org.freedesktop.login1.Session.TakeDevice` on the system bus, and logind returns an open, revocable file descriptor. On a virtual terminal switch, logind sends `PauseDevice` and `ResumeDevice` signals over the same bus. The compositor never touches `/dev/dri/card0` directly: D-Bus is the gatekeeper.

Similar patterns appear throughout the stack:

- **Colour management**: `colord` exposes `org.freedesktop.ColorManager` so that compositors can retrieve ICC profiles and map them to KMS `GAMMA_LUT` blobs (Chapter 53).
- **Screen capture and remote desktop**: `xdg-desktop-portal` exposes `org.freedesktop.portal.ScreenCast` and `org.freedesktop.portal.RemoteDesktop`, brokering PipeWire stream handles to sandboxed apps (Chapter 23, Chapter 38).
- **Game optimisation**: GameMode exposes `com.feralinteractive.GameMode` so that game launchers can request CPU governor and GPU performance-mode changes (Chapter 78).
- **Power management and DPMS**: Display sleep inhibition flows through `org.freedesktop.PowerManagement.Inhibit` (or systemd's `org.freedesktop.login1.Manager.Inhibit`), allowing video players and games to prevent blanking.
- **System tray**: The StatusNotifierItem specification layers on top of `org.freedesktop.StatusNotifierItem`, used by every system tray in GNOME, KDE, and independent compositors.

The table below catalogs the D-Bus interfaces most relevant to graphics and display work:

| Interface | Service | Purpose | Chapter |
|-----------|---------|---------|---------|
| `org.freedesktop.login1.Manager` | `org.freedesktop.login1` | Session/seat management; TakeDevice, VT switch | Ch 21 |
| `org.freedesktop.login1.Session` | `org.freedesktop.login1` | Per-session TakeDevice, PauseDevice, ReleaseDevice | Ch 21 |
| `org.freedesktop.ColorManager` | `org.freedesktop.ColorManager` | ICC profile management via colord | Ch 53 |
| `org.freedesktop.portal.ScreenCast` | `org.freedesktop.portal.desktop` | Screen capture portal for sandboxed apps | Ch 23 |
| `org.freedesktop.portal.RemoteDesktop` | `org.freedesktop.portal.desktop` | Remote desktop portal | Ch 23 |
| `com.feralinteractive.GameMode` | `com.feralinteractive.GameMode` | CPU/GPU performance mode requests | Ch 78 |
| `org.freedesktop.StatusNotifierItem` | (well-known per app) | System tray icon protocol | — |
| `org.freedesktop.PowerManagement.Inhibit` | `org.freedesktop.PowerManagement` | Prevent DPMS blanking | — |
| `org.freedesktop.portal.Flatpak` | `org.freedesktop.portal.desktop` | Flatpak sandbox host integration | Ch 111 |
| `org.kde.KWin.ScreenShot2` | `org.kde.KWin` | KDE-specific screenshot API | — |
| `org.gnome.Shell.Screenshot` | `org.gnome.Shell` | GNOME Shell screenshot API | — |

The situation is not static. Three forces are reshaping this ecosystem: dbus-broker has displaced the aging dbus-daemon on most modern distributions; systemd is routing new services to Varlink instead of D-Bus; and a small but vocal faction in the compositor community — led by Hyprland's Vaxry — is building a parallel IPC world around hyprwire and hyprtavern. Understanding each option, and when to choose it, is the goal of this chapter.

---

## 2. D-Bus Architecture and Wire Protocol

### Three Buses

D-Bus provides three conceptually distinct buses, each with its own Unix domain socket:

**System bus** (`/run/dbus/system_bus_socket`): shared across all users and sessions on a machine. logind, colord, NetworkManager, UPower, and most hardware-management daemons live here. Any process on the system can connect to the system bus subject to policy file restrictions.

**Session bus** (`$DBUS_SESSION_BUS_ADDRESS`): one instance per user login session, typically started by `dbus-daemon --session` or, on systemd systems, by `systemd --user`. Desktop applications, compositors, xdg-desktop-portal, and PipeWire connect here.

**Activation**: D-Bus supports service activation — a daemon is started on demand the first time a message is sent to its well-known name. Activation records live in `.service` files under `/usr/share/dbus-1/services/` (session) and `/usr/share/dbus-1/system-services/` (system). [Source](https://dbus.freedesktop.org/doc/dbus-specification.html)

### Object Model

D-Bus structures communication around four concepts that compose hierarchically:

- **Bus name**: either a *well-known name* like `org.freedesktop.login1` (owned by a specific service) or a *unique name* like `:1.42` (assigned by the broker at connect time). Unique names are stable for the lifetime of a connection.
- **Object path**: a UNIX-style path like `/org/freedesktop/login1/session/auto` identifying a specific object within a service. Multiple objects can live under one service.
- **Interface**: a named namespace for a set of members, e.g. `org.freedesktop.login1.Session`. An object can implement multiple interfaces simultaneously, including the mandatory `org.freedesktop.DBus.Introspectable`, `org.freedesktop.DBus.Peer`, and `org.freedesktop.DBus.Properties`.
- **Member**: a method, signal, or property within an interface. `TakeDevice` is a method on `org.freedesktop.login1.Session`; `PauseDevice` is a signal on the same interface.

### Wire Protocol

The D-Bus wire format is a binary protocol, little-endian by default (big-endian is also legal but rare), with messages composed of a fixed header block followed by a variable-length body. [Source](https://dbus.freedesktop.org/doc/dbus-specification.html)

**Message types:**
- `METHOD_CALL` (type code 1): a client invoking a remote method; expects a `METHOD_RETURN` or `ERROR` reply.
- `METHOD_RETURN` (type code 2): the successful result of a method call; carries the reply serial matching the call serial.
- `ERROR` (type code 3): a failed method call; includes an error name (e.g. `org.freedesktop.DBus.Error.ServiceUnknown`) and optional human-readable message.
- `SIGNAL` (type code 4): a one-way broadcast with no reply; any number of listeners can receive it via match rules.

**Fixed header fields** (each is a `(BYTE, VARIANT)` pair in the header array):
| Code | Name | Type | Meaning |
|------|------|------|---------|
| 1 | `PATH` | `o` | Destination object path |
| 2 | `INTERFACE` | `s` | Target interface |
| 3 | `MEMBER` | `s` | Method or signal name |
| 4 | `ERROR_NAME` | `s` | Error name (ERROR messages only) |
| 5 | `REPLY_SERIAL` | `u` | Serial of the call being replied to |
| 6 | `DESTINATION` | `s` | Recipient bus name |
| 7 | `SENDER` | `s` | Originating bus name (filled by broker) |
| 8 | `SIGNATURE` | `g` | Type signature of the body |
| 9 | `UNIX_FDS` | `u` | Number of Unix FDs in ancillary data |

### Type System

D-Bus has a complete type system encoding data structures as typed byte sequences. The type of any value is described by a compact *signature* string using single-character type codes:

**Basic types:**
| Code | Name | C type |
|------|------|--------|
| `y` | BYTE | `uint8_t` |
| `b` | BOOLEAN | `uint32_t` (0 or 1) |
| `n` | INT16 | `int16_t` |
| `q` | UINT16 | `uint16_t` |
| `i` | INT32 | `int32_t` |
| `u` | UINT32 | `uint32_t` |
| `x` | INT64 | `int64_t` |
| `t` | UINT64 | `uint64_t` |
| `d` | DOUBLE | `double` |
| `s` | STRING | NUL-terminated UTF-8, length-prefixed |
| `o` | OBJECT_PATH | Same encoding as STRING |
| `g` | SIGNATURE | One-byte length + chars |
| `h` | UNIX_FD | `uint32_t` index into ancillary FD array |

**Container types:**
- `a` + element type: array, e.g. `as` is an array of strings.
- `(` members `)`: struct, e.g. `(iu)` is a struct containing INT32 and UINT32.
- `v`: variant — self-describing; the value includes its own signature. Used for polymorphic data.
- `a{kv}`: dictionary (array of dict entries), e.g. `a{sv}` is the ubiquitous "string to variant" dictionary seen everywhere in D-Bus APIs.

The infamous `a{sv}` dict-of-variants is D-Bus's escape hatch for extensibility. It is used heavily by `org.freedesktop.portal.*`, `org.freedesktop.UDisks2`, and many others. Its flexibility comes at a cost: neither the broker nor the compiler can enforce the actual types of values inside it, making the type system effectively nominal at that layer.

### Introspection

Every D-Bus object implementing `org.freedesktop.DBus.Introspectable` must respond to `Introspect()` with an XML document describing its interfaces, methods, signals, and properties. Tools such as `gdbus`, `busctl`, and `d-feet` use introspection to discover what a service offers without reading source code.

A representative introspection fragment for `org.freedesktop.login1.Session.TakeDevice`:

```xml
<!-- Source: systemd/src/login/org.freedesktop.login1.conf -->
<interface name="org.freedesktop.login1.Session">
  <method name="TakeDevice">
    <arg name="major" type="u" direction="in"/>
    <arg name="minor" type="u" direction="in"/>
    <arg name="fd"    type="h" direction="out"/>
    <arg name="inactive" type="b" direction="out"/>
  </method>
  <method name="ReleaseDevice">
    <arg name="major" type="u" direction="in"/>
    <arg name="minor" type="u" direction="in"/>
  </method>
  <signal name="PauseDevice">
    <arg name="major" type="u"/>
    <arg name="minor" type="u"/>
    <arg name="type"  type="s"/>
  </signal>
  <signal name="ResumeDevice">
    <arg name="major" type="u"/>
    <arg name="minor" type="u"/>
    <arg name="fd"    type="h"/>
  </signal>
</interface>
```

[Source: systemd login1 interface](https://www.freedesktop.org/software/systemd/man/latest/org.freedesktop.login1.html)

### Security Model

D-Bus authenticates peers using the kernel's credential-passing mechanism over Unix sockets: when a connection is established, the broker retrieves the peer's `uid`, `gid`, and `pid` via `SO_PEERCRED`. Authentication uses the `EXTERNAL` mechanism (passing UID as a string), and the broker verifies this against the kernel-reported credential.

**Policy files** at `/etc/dbus-1/system.d/` and `/usr/share/dbus-1/system.d/` define what each uid or group may do:

```xml
<!-- /usr/share/dbus-1/system.d/org.freedesktop.login1.conf (excerpt) -->
<busconfig>
  <policy user="root">
    <allow own="org.freedesktop.login1"/>
  </policy>
  <policy context="default">
    <allow send_destination="org.freedesktop.login1"/>
    <allow receive_sender="org.freedesktop.login1"/>
  </policy>
</busconfig>
```

AppArmor and SELinux can add label-based restrictions on top of UID policies, and dbus-broker integrates with the Linux audit subsystem to log policy violations. [Source](https://dbus.freedesktop.org/doc/dbus-daemon.1.html)

---

## 3. dbus-daemon vs dbus-broker

### dbus-daemon: The Reference Implementation

`dbus-daemon` is the original C implementation of the D-Bus specification, maintained at [freedesktop.org](https://gitlab.freedesktop.org/dbus/dbus). It is a single-threaded event loop process that receives messages from clients, validates them against policy, and routes them to destinations.

Historical pain points:
- **setuid helper**: dbus-daemon uses a setuid helper (`dbus-daemon-launch-helper`) to activate system services as unprivileged users. setuid binaries are notoriously difficult to audit.
- **Resource exhaustion**: a misbehaving or compromised client could exhaust the daemon's memory or file descriptor table, potentially causing denial-of-service for all other D-Bus users on the system.
- **Single-threaded bottleneck**: all message routing occurs in a single event loop; under high message rates the daemon becomes a serialisation point.
- **Limited auditability**: security events are not always propagated to the kernel audit subsystem.

### dbus-broker: A Linux-Native Rewrite

[dbus-broker](https://github.com/bus1/dbus-broker) is a ground-up reimplementation by David Rheinsberg and Tom Gundersen, designed exclusively for Linux and designed to fix the architectural problems of dbus-daemon while remaining wire-compatible.

**Two-process model**: dbus-broker separates concerns into two processes:

1. **Launcher** (`dbus-broker-launch`): a privileged process that reads D-Bus configuration, manages service activation, and owns the socket. It uses `CAP_SETUID` and `CAP_AUDIT_WRITE` via Linux capabilities rather than a setuid binary.
2. **Broker** (`dbus-broker`): the actual message-routing process, which runs with minimal capabilities. The launcher passes the socket and configuration to the broker, then enters a supervisory role.

This separation means that even if the broker has a vulnerability, the attacker cannot use it to gain the full privileges of the launcher process.

**Per-peer resource accounting**: dbus-broker implements granular accounting:
- Match rule limits: each peer has a cap on the number of signal subscriptions.
- Message queue depth: the broker tracks how many messages are queued to each peer.
- Reply quotas: pending replies are tracked per connection to prevent reply flooding.
- Memory quotas: total memory attributed to a connection's message queues.

**Linux audit integration**: policy violations (e.g. a sandboxed app trying to call a method it is not allowed to call) are logged as audit records to the kernel audit subsystem, making them visible via `ausearch` and `auditd`.

**Current version**: dbus-broker 37, released June 16, 2025. [Source](https://github.com/bus1/dbus-broker)

**Distribution adoption:**
- Fedora: dbus-broker became the default in Fedora 29 (released October 2018). [Source](https://fedoraproject.org/wiki/Changes/DbusBrokerAsTheDefaultDbusImplementation)
- Arch Linux: ships dbus-broker as the default.
- openSUSE: ships dbus-broker by default.
- NixOS: dbus-broker available and can be configured as default.
- Debian/Ubuntu: continue to ship dbus-daemon as the default; dbus-broker is available but not the default as of mid-2026.

**Checking which implementation is running:**

```bash
# Check the service executable
systemctl status dbus
# or
ls -la /proc/$(pidof dbus-broker 2>/dev/null || pidof dbus-daemon)/exe

# If dbus-broker is running, you'll see dbus-broker in the process list:
pgrep -a dbus-broker
```

**Configuration compatibility**: dbus-broker reads the same `/etc/dbus-1/system.d/` policy XML files as dbus-daemon. For distributions that default to dbus-broker, no policy file changes are needed. The `dbus-broker-launch` unit replaces `dbus.service` and reads the same activation `.service` files.

---

## 4. Programming D-Bus: sd-bus (C)

### Overview

`sd-bus` is the D-Bus client library embedded in `libsystemd`, the shared library distributed as part of systemd. It occupies the space between raw `libdbus` (low-level, verbose) and GLib/GIO (high-level, GObject-heavy): sd-bus provides a clean C API without requiring GLib, while automating marshalling and event loop integration through systemd's `sd-event`.

sd-bus can be used independently of the rest of systemd — any process linking against `libsystemd` gets access to it. [Source](https://www.freedesktop.org/software/systemd/man/latest/sd_bus_call_method.html)

### Core API Functions

```c
// Connection management
int sd_bus_open_system(sd_bus **ret);
int sd_bus_open_user(sd_bus **ret);
int sd_bus_open_system_remote(sd_bus **ret, const char *host);
sd_bus *sd_bus_unref(sd_bus *bus);

// Synchronous method calls
int sd_bus_call_method(
    sd_bus *bus,
    const char *destination,  // well-known or unique name
    const char *path,         // object path
    const char *interface,    // interface name
    const char *member,       // method name
    sd_bus_error *ret_error,  // output: error (if any)
    sd_bus_message **reply,   // output: reply message
    const char *types,        // input type signature
    ...                       // input arguments
);

// Read values from a reply message
int sd_bus_message_read(sd_bus_message *m, const char *types, ...);

// Property access
int sd_bus_get_property(
    sd_bus *bus,
    const char *destination,
    const char *path,
    const char *interface,
    const char *member,       // property name
    sd_bus_error *ret_error,
    sd_bus_message **reply,
    const char *type          // expected type signature
);

// Signal matching
int sd_bus_match_signal(
    sd_bus *bus,
    sd_bus_slot **ret_slot,
    const char *sender,
    const char *path,
    const char *interface,
    const char *member,
    sd_bus_message_handler_t callback,
    void *userdata
);

// Expose a D-Bus object
int sd_bus_add_object_vtable(
    sd_bus *bus,
    sd_bus_slot **slot,
    const char *path,
    const char *interface,
    const sd_bus_vtable *vtable,
    void *userdata
);
```

### Worked Example: TakeDevice for DRM Access

This example shows how a Wayland compositor calls `TakeDevice` to obtain a DRM file descriptor from logind. This is the D-Bus call that underlies every compositor that runs as an unprivileged user — wlroots, mutter, and KWin all make this call (or an equivalent through libseat).

```c
/* logind_take_device.c
 * Demonstrates org.freedesktop.login1.Session.TakeDevice
 * Source: based on wlroots backend/session/logind.c
 * https://gitlab.freedesktop.org/wlroots/wlroots/-/blob/master/backend/session/logind.c
 */
#include <systemd/sd-bus.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdio.h>

int take_drm_device(const char *session_path, const char *dev_path)
{
    sd_bus *bus = NULL;
    sd_bus_message *reply = NULL;
    sd_bus_error error = SD_BUS_ERROR_NULL;
    struct stat st;
    int r, fd;
    int inactive;

    /* Stat the device to get major/minor numbers */
    if (stat(dev_path, &st) < 0) {
        perror("stat");
        return -1;
    }

    /* Open a connection to the system bus */
    r = sd_bus_open_system(&bus);
    if (r < 0) {
        fprintf(stderr, "Failed to connect to system bus: %s\n", strerror(-r));
        goto cleanup;
    }

    /* Call org.freedesktop.login1.Session.TakeDevice
     * Arguments: uint32 major, uint32 minor
     * Returns:   unix_fd fd, bool inactive
     */
    r = sd_bus_call_method(
        bus,
        "org.freedesktop.login1",        /* destination */
        session_path,                     /* /org/freedesktop/login1/session/auto */
        "org.freedesktop.login1.Session", /* interface */
        "TakeDevice",                     /* method */
        &error,
        &reply,
        "uu",                             /* input types: uint32 major, uint32 minor */
        (uint32_t)major(st.st_rdev),
        (uint32_t)minor(st.st_rdev)
    );
    if (r < 0) {
        fprintf(stderr, "TakeDevice failed: %s\n", error.message);
        goto cleanup;
    }

    /* Read the returned fd and inactive flag.
     * 'h' is the type code for UNIX_FD in sd-bus message reading.
     * The FD is valid as long as `reply` is held; dup it if needed.
     */
    r = sd_bus_message_read(reply, "hb", &fd, &inactive);
    if (r < 0) {
        fprintf(stderr, "Failed to read reply: %s\n", strerror(-r));
        goto cleanup;
    }

    /* dup with CLOEXEC so the fd survives message teardown */
    fd = fcntl(fd, F_DUPFD_CLOEXEC, 0);
    printf("Got DRM fd=%d for %s (inactive=%d)\n", fd, dev_path, inactive);

cleanup:
    sd_bus_message_unref(reply);
    sd_bus_error_free(&error);
    sd_bus_unref(bus);
    return (r >= 0) ? fd : -1;
}
```

### Signal Subscription: PauseDevice and ResumeDevice

When logind needs to revoke a device (e.g. on a VT switch), it sends `PauseDevice` to the session controller. The compositor must stop using the DRM device and call `PauseDeviceComplete`. When the VT is restored, `ResumeDevice` delivers a new fd.

```c
/* Subscribe to PauseDevice and ResumeDevice signals */
static int on_pause_device(sd_bus_message *m, void *userdata, sd_bus_error *err)
{
    uint32_t major, minor;
    const char *type;  /* "pause" (graceful) or "force" (already paused) */

    sd_bus_message_read(m, "uus", &major, &minor, &type);
    fprintf(stderr, "PauseDevice: %u:%u type=%s\n", major, minor, type);

    /* Compositor must stop rendering and close/release the DRM fd here.
     * If type == "pause", we must also call PauseDeviceComplete. */
    return 0;
}

static int on_resume_device(sd_bus_message *m, void *userdata, sd_bus_error *err)
{
    uint32_t major, minor;
    int fd;

    sd_bus_message_read(m, "uuh", &major, &minor, &fd);
    fprintf(stderr, "ResumeDevice: %u:%u fd=%d\n", major, minor, fd);
    /* Re-open the DRM device using the new fd */
    return 0;
}

/* Register both signal handlers */
sd_bus_slot *pause_slot = NULL, *resume_slot = NULL;

sd_bus_match_signal(bus, &pause_slot,
    "org.freedesktop.login1",
    session_path,
    "org.freedesktop.login1.Session",
    "PauseDevice",
    on_pause_device, NULL);

sd_bus_match_signal(bus, &resume_slot,
    "org.freedesktop.login1",
    session_path,
    "org.freedesktop.login1.Session",
    "ResumeDevice",
    on_resume_device, NULL);
```

[Source: sd_bus_match_signal man page](https://www.freedesktop.org/software/systemd/man/latest/sd_bus_match_signal.html)

### Server Side: Exposing a D-Bus Object

`sd_bus_add_object_vtable` maps C function pointers to D-Bus methods:

```c
static int method_get_info(sd_bus_message *m, void *userdata,
                           sd_bus_error *ret_error)
{
    return sd_bus_reply_method_return(m, "s", "compositor v1.0");
}

static const sd_bus_vtable compositor_vtable[] = {
    SD_BUS_VTABLE_START(0),
    SD_BUS_METHOD("GetInfo", NULL, "s", method_get_info, SD_BUS_VTABLE_UNPRIVILEGED),
    SD_BUS_VTABLE_END
};

/* Register the object */
sd_bus_slot *slot = NULL;
sd_bus_add_object_vtable(bus, &slot,
    "/org/example/Compositor",
    "org.example.Compositor",
    compositor_vtable,
    NULL);
sd_bus_request_name(bus, "org.example.Compositor", 0);
```

---

## 5. Programming D-Bus: GLib/GIO (gdbus-codegen)

### GDBusConnection and GDBusProxy

GLib's GIO library provides a higher-level D-Bus API built on GLib's type system (GVariant) and main loop (GMainContext). The main types are:

- `GDBusConnection`: represents a connection to a bus. Created with `g_bus_get_sync()` or `g_bus_get()` (async).
- `GDBusProxy`: a client-side proxy that mirrors a remote D-Bus object. Handles property caching, signal subscription, and async method calls automatically.
- `GDBusInterfaceSkeleton`: the server-side counterpart, for exposing objects.

[Source: GDBusConnection documentation](https://docs.gtk.org/gio/class.DBusConnection.html)

### gdbus-codegen

`gdbus-codegen` is a code generator that reads D-Bus introspection XML and produces C source and header files with fully typed GDBusProxy and GDBusInterfaceSkeleton subclasses. This eliminates the manual GVariant packing/unpacking that characterises raw GDBusProxy usage.

Usage:

```bash
# Generate C bindings from an introspection XML file
gdbus-codegen \
    --interface-prefix org.freedesktop.login1. \
    --generate-c-code logind-generated \
    --c-namespace Logind \
    org.freedesktop.login1.xml
# Produces: logind-generated.h and logind-generated.c
# Contains: LogindSession (GObject), LogindSessionProxy (GDBusProxy subclass)
```

The generated `LogindSession` object has typed methods like:

```c
/* Generated by gdbus-codegen -- do not edit manually */
gboolean logind_session_call_take_device_sync(
    LogindSession *proxy,
    guint32 arg_major,
    guint32 arg_minor,
    GVariant **out_fd,    /* Note: GVariant wrapping the h type */
    gboolean *out_inactive,
    GCancellable *cancellable,
    GError **error);
```

> **Note**: GLib does not have native support for the `h` (UNIX_FD) D-Bus type in all contexts; the generated code typically wraps the fd in a GVariant of type `h` and the caller must extract the fd separately via `g_variant_get_handle()` and `g_unix_fd_list_get()`. This is a common source of confusion.

### Async Method Call Pattern

```c
/* Async pattern with GTask/GAsyncResult */
void take_device_async(GDBusProxy *session_proxy,
                       guint32 major, guint32 minor,
                       GAsyncReadyCallback callback,
                       gpointer user_data)
{
    g_dbus_proxy_call(
        session_proxy,
        "TakeDevice",
        g_variant_new("(uu)", major, minor),
        G_DBUS_CALL_FLAGS_NONE,
        -1,          /* timeout: -1 means default */
        NULL,        /* GCancellable */
        callback,
        user_data
    );
}
```

### Subscribing to colord Signals

```c
/* Subscribe to org.freedesktop.ColorManager.Changed */
GDBusConnection *system_bus = g_bus_get_sync(G_BUS_TYPE_SYSTEM, NULL, NULL);

guint sub_id = g_dbus_connection_signal_subscribe(
    system_bus,
    "org.freedesktop.ColorManager",   /* sender */
    "org.freedesktop.ColorManager",   /* interface */
    "Changed",                         /* signal name */
    "/org/freedesktop/ColorManager",  /* object path */
    NULL,                              /* arg0 filter */
    G_DBUS_SIGNAL_FLAGS_NONE,
    on_color_manager_changed,
    NULL, NULL);
```

### gdbus Command-Line Tool

The `gdbus` tool is invaluable for interactive exploration and debugging:

```bash
# Introspect the logind session manager
gdbus introspect --system \
    --dest org.freedesktop.login1 \
    --object-path /org/freedesktop/login1

# Call a method (list sessions)
gdbus call --system \
    --dest org.freedesktop.login1 \
    --object-path /org/freedesktop/login1 \
    --method org.freedesktop.login1.Manager.ListSessions

# Monitor all signals on the session bus
gdbus monitor --session --dest org.freedesktop.portal.desktop
```

The `busctl` tool from systemd provides similar functionality with more output format options:

```bash
busctl --system tree org.freedesktop.login1
busctl --system introspect org.freedesktop.login1 /org/freedesktop/login1
busctl --system call org.freedesktop.login1 /org/freedesktop/login1 \
    org.freedesktop.login1.Manager ListSessions
```

---

## 6. Programming D-Bus: Python (dasbus)

### dbus-python: The Classic Binding

`dbus-python` is the long-standing Python binding that wraps `libdbus` directly. It exposes the D-Bus type system as Python types and provides a reasonably pythonic API, but its design predates Python 3's async ecosystem and shows its age. Installation: `pip install dbus-python` (requires `libdbus-1-dev`).

```python
# dbus-python classic style
import dbus

bus = dbus.SystemBus()
logind = bus.get_object('org.freedesktop.login1',
                        '/org/freedesktop/login1')
manager = dbus.Interface(logind, 'org.freedesktop.login1.Manager')

sessions = manager.ListSessions()
for session_id, uid, user, seat, path in sessions:
    print(f"Session {session_id}: user={user}, seat={seat}")
```

### dasbus: Modern High-Level Library

`dasbus` ([GitHub](https://github.com/dasbus-project/dasbus)) is a Python 3 library built on GLib and PyGObject. It provides a class-based, type-annotated approach to D-Bus programming inspired by pydbus. Originally developed for Anaconda (Fedora's installer), it has been extracted into a standalone library.

Key design choices:
- **No libdbus dependency** at runtime: uses GLib's D-Bus implementation via PyGObject.
- **Class-based interface definitions**: D-Bus methods map to Python methods; D-Bus signals map to Python signals.
- **Type annotations** drive marshalling automatically in many cases.
- **Proxy objects** with discoverable API (autocomplete-friendly).

```python
# dasbus: calling logind to list sessions
# Source: https://dasbus.readthedocs.io/
from dasbus.connection import SystemMessageBus
from dasbus.identifier import DBusServiceIdentifier

# Define the service identifier
LOGIND = DBusServiceIdentifier(
    namespace=("org", "freedesktop", "login1"),
    message_bus=SystemMessageBus()
)

# Get a proxy to the Manager object
manager = LOGIND.get_proxy()
sessions = manager.ListSessions()
for session in sessions:
    print(session)
```

For custom interfaces, dasbus uses class-based service definitions:

```python
from dasbus.server.interface import dbus_interface, dbus_signal
from dasbus.typing import Str, UInt32

@dbus_interface("org.example.MyService")
class MyServiceInterface:

    def GetVersion(self) -> Str:
        """Return the service version string."""
        ...

    @dbus_signal
    def StateChanged(self, new_state: Str):
        """Emitted when state changes."""
        pass
```

### Choosing Between dbus-python and dasbus

| Aspect | dbus-python | dasbus |
|--------|------------|--------|
| Underlying transport | libdbus | GLib/GDBus via PyGObject |
| Type system | Automatic D-Bus ↔ Python | Annotation-driven |
| Async support | Limited (uses GMainLoop) | GLib event loop |
| API style | Imperative object model | Class-based interfaces |
| Maintenance | Minimal (stable) | Active |
| Best for | Quick scripts, legacy code | New services, compositors |

**python-sdbus** ([GitHub](https://github.com/python-sdbus/python-sdbus)) is a third option: Python bindings wrapping `sd-bus` from libsystemd, with asyncio support. Useful when you want the sd-bus performance model from Python.

---

## 7. Programming D-Bus: Rust — zbus

### Overview

[zbus](https://github.com/dbus2/zbus) (currently maintained under the `z-galaxy` GitHub organisation) is the idiomatic Rust D-Bus library. Unlike earlier Rust D-Bus bindings (`dbus` crate) that wrapped C libdbus, zbus is a pure-Rust reimplementation with no C library dependency. As of mid-2026, the current release is **zbus 5.16.0** (released May 29, 2026). [Source](https://github.com/z-galaxy/zbus)

Key design properties:
- **Pure Rust**: no `libdbus` FFI, no C dependency for the core.
- **Async-native**: the API is `async`/`await` first, with optional blocking wrappers via the `blocking` feature.
- **Runtime agnostic**: works with tokio, async-std, or any executor; the README explicitly notes "you can use any async runtime of choice."
- **Proc-macro API**: `#[proxy]` and `#[interface]` macros generate typed client proxies and server objects from trait definitions.
- **zvariant subcrate**: handles D-Bus wire format serialisation using serde, with a companion GVariant mode.

### The `#[proxy]` Macro

`#[proxy]` annotates a Rust trait to generate an async `FooProxy` struct with typed methods, signals, and property accessors:

```rust
use zbus::{proxy, Connection, Result};

#[proxy(
    interface = "org.freedesktop.login1.Session",
    default_service = "org.freedesktop.login1",
    default_path = "/org/freedesktop/login1/session/auto"
)]
trait Session {
    /// Request access to a device by major/minor number.
    /// Returns (fd, inactive) where inactive=true means the device is
    /// currently suspended (VT is not active).
    async fn take_device(
        &self,
        major: u32,
        minor: u32,
    ) -> Result<(std::os::unix::io::OwnedFd, bool)>;

    /// Release a previously taken device.
    async fn release_device(&self, major: u32, minor: u32) -> Result<()>;

    /// Acknowledge a PauseDevice signal when type is "pause" (not "force").
    async fn pause_device_complete(&self, major: u32, minor: u32) -> Result<()>;

    /// Emitted when logind needs to revoke a device (e.g. VT switch).
    /// `kind` is "pause" (graceful) or "force" (already revoked).
    #[zbus(signal)]
    async fn pause_device(
        &self,
        major: u32,
        minor: u32,
        kind: String,
    ) -> Result<()>;

    /// Emitted when a paused device becomes available again.
    #[zbus(signal)]
    async fn resume_device(
        &self,
        major: u32,
        minor: u32,
        fd: std::os::unix::io::OwnedFd,
    ) -> Result<()>;
}
```

The macro generates `SessionProxy` (async) and `SessionProxyBlocking` (blocking) with identical method signatures. Signal streams are returned as `SignalStream<PauseDevice>` and can be `async`-iterated.

### Full Example: Opening DRM Device via logind

```rust
// logind_take_device.rs
// zbus 5.x — pure-Rust logind TakeDevice for compositor DRM access
// Source: zbus documentation at https://z-galaxy.github.io/zbus/
use zbus::{proxy, Connection, Result};
use std::os::unix::io::OwnedFd;

#[proxy(
    interface = "org.freedesktop.login1.Session",
    default_service = "org.freedesktop.login1",
    default_path = "/org/freedesktop/login1/session/auto"
)]
trait Session {
    async fn take_device(&self, major: u32, minor: u32)
        -> Result<(OwnedFd, bool)>;
    async fn release_device(&self, major: u32, minor: u32) -> Result<()>;
    async fn pause_device_complete(&self, major: u32, minor: u32) -> Result<()>;

    #[zbus(signal)]
    async fn pause_device(&self, major: u32, minor: u32, kind: String)
        -> Result<()>;
    #[zbus(signal)]
    async fn resume_device(&self, major: u32, minor: u32, fd: OwnedFd)
        -> Result<()>;
}

#[tokio::main]
async fn main() -> Result<()> {
    let connection = Connection::system().await?;
    let session = SessionProxy::new(&connection).await?;

    // /dev/dri/card0 has major=226; minor varies by GPU count (usually 0 or 64)
    // These numbers can be obtained via libudev or stat(2) on the device path.
    let drm_major: u32 = 226;
    let drm_minor: u32 = 0; // adjust for your system

    let (fd, inactive) = session.take_device(drm_major, drm_minor).await?;
    println!("Got DRM fd: {fd:?}, device inactive (VT not active): {inactive}");

    // Subscribe to PauseDevice signals
    let mut pause_stream = session.receive_pause_device().await?;
    while let Some(signal) = pause_stream.next().await {
        let args = signal.args()?;
        eprintln!("PauseDevice: {}:{} kind={}", args.major, args.minor, args.kind);
        // For kind == "pause" (not "force"), we must acknowledge:
        if args.kind == "pause" {
            session.pause_device_complete(args.major, args.minor).await?;
        }
    }

    Ok(())
}
```

> **Note**: `receive_pause_device()` returns a `futures::Stream`; add `use futures::StreamExt;` to call `.next().await`. The `OwnedFd` return type transfers file descriptor ownership to the caller — the fd remains valid after the D-Bus message is dropped.

### The `#[interface]` Macro: Exposing a Server

```rust
use zbus::{interface, Connection, Result};

struct CompositorInfo {
    version: String,
}

#[interface(name = "org.example.CompositorInfo")]
impl CompositorInfo {
    /// Returns the compositor version string.
    async fn get_version(&self) -> String {
        self.version.clone()
    }

    /// Emitted when the compositor restarts. [zbus(signal)] makes this
    /// a D-Bus signal; call it with `CompositorInfo::restarted(&cx).await?`
    #[zbus(signal)]
    async fn restarted(cx: &zbus::SignalContext<'_>) -> Result<()>;
}

#[tokio::main]
async fn main() -> Result<()> {
    let info = CompositorInfo { version: "1.0.0".into() };
    let _conn = Connection::session().await?
        .object_server()
        .at("/org/example/CompositorInfo", info)?;

    // Keep alive
    std::future::pending::<()>().await;
    Ok(())
}
```

### zbus vs the `dbus` Crate

The older [`dbus`](https://crates.io/crates/dbus) crate wraps libdbus via FFI. This gives it a smaller initial compile surface but brings the C library dependency and its historical quirks into the Rust ecosystem. zbus, by contrast:

- Compiles without libdbus installed.
- Integrates cleanly with async Rust.
- Has a more ergonomic macro API.
- Is actively maintained by the D-Bus community (now hosted under the `dbus2` GitHub organisation).

For new projects targeting modern Linux, **zbus is the recommended choice**. The `dbus` crate remains useful for cross-platform work or where FFI is preferable.

**zbus in graphics-adjacent Rust projects**: pipewire-rs (the Rust PipeWire binding) uses zbus for D-Bus interactions; various Wayland compositors experimenting with Rust (e.g. Smithay-based) use zbus for logind integration.

---

## 8. Varlink: systemd's Simpler IPC

### What Varlink Is

[Varlink](https://varlink.org/) is a lightweight IPC system created by Kay Sievers and Lars Karlitski (announced in 2017 by Harald Hoyer). [Source](https://varlink.org/) Where D-Bus routes messages through a central broker over a shared socket, Varlink uses direct point-to-point Unix domain sockets — one socket per service. Messages are JSON objects terminated by a NUL byte (`\0`). There is no XML, no type registry, no central daemon.

The design philosophy is radical simplicity: a Varlink service is just a Unix socket that speaks JSON. Any language that can open a socket and emit JSON can implement a Varlink client or server. The protocol is also human-readable on the wire, making debugging as simple as:

```bash
echo '{"method":"io.systemd.Resolve.ResolveHostname","parameters":{"name":"freedesktop.org","family":2}}' | \
  nc -U /run/systemd/resolve/io.systemd.Resolve
```

**Varlink is not experimental.** It is production-deployed across systemd — which runs on approximately 95% of Linux desktop and server installations. Every new systemd service since approximately systemd 246 (2020) exposes its primary API via Varlink rather than D-Bus. The systemd developers have publicly stated at the All Systems Go conference that Varlink is the direction for IPC in systemd's future, with D-Bus retained only for backward compatibility with existing interfaces. [Source: Phoronix — Systemd Looking At A Future With More Varlink & Less D-Bus](https://www.phoronix.com/news/Systemd-Varlink-D-Bus-Future)

For compositor authors and graphics-adjacent system developers, Varlink is not a curiosity — it is the emerging standard that will become pervasive for system service APIs within the lifetime of this book.

### IDL Syntax

Varlink uses a compact Interface Definition Language significantly simpler than D-Bus XML introspection. An interface declaration names the interface (reverse-domain notation), defines types, declares methods, and lists errors:

```varlink
# io.systemd.Resolve -- DNS resolution via systemd-resolved
# Source: systemd source at src/resolve/io.systemd.Resolve.varlink

interface io.systemd.Resolve

type ResolvedAddress (
        ifindex: int,
        family: int,
        address: data
)

method ResolveHostname(
        ifindex: ?int,
        name: string,
        family: ?int,
        flags: ?int
) -> (
        addresses: []ResolvedAddress,
        canonical: string,
        flags: int
)

method ResolveAddress(
        ifindex: ?int,
        family: int,
        address: data,
        flags: ?int
) -> (
        names: []string,
        flags: int
)

error NoNameServers()
error InvalidName(name: string)
error QueryTimedOut()
```

The `?` prefix marks optional fields (nullable); `[]Type` is an array; `data` is a byte array (base64 in JSON). Methods with multiple return values use the `-> (fields...)` syntax. Error types are first-class IDL members. [Source](https://systemd.io/VARLINK/)

### Varlink Interfaces in systemd: A Growing Ecosystem

systemd has been migrating new APIs to Varlink rather than D-Bus since approximately systemd 246 (2020). The pattern is consistent and intentional: **D-Bus for backward-compatible existing APIs; Varlink for everything new**. As of systemd 261 (2026), the following `io.systemd.*` Varlink interfaces are in production use:

| Interface | Service | Purpose |
|-----------|---------|---------|
| `io.systemd.Resolve` | systemd-resolved | DNS resolution, address lookup |
| `io.systemd.Resolve.Monitor` | systemd-resolved | Stream DNS resolution events |
| `io.systemd.UserDatabase` | systemd-userdbd | User and group record lookup |
| `io.systemd.Multiplexer` | systemd-userdbd | Multiplex multiple UserDatabase backends |
| `io.systemd.NameServiceSwitch` | systemd-userdbd | NSS compatibility interface |
| `io.systemd.Hostname` | systemd-hostnamed | Query and set hostname, OS info |
| `io.systemd.MachineImage` | systemd-machined | Container/VM image management |
| `io.systemd.Manager` | systemd (PID 1) | Unit lifecycle, shutdown, reboot, jobs |
| `io.systemd.Unit` | systemd (PID 1) | Per-unit management, transient units |
| `io.systemd.Job` | systemd (PID 1) | List and cancel queued jobs |
| `io.systemd.Service` | systemd (PID 1) | Per-service state queries |
| `io.systemd.Journal` | systemd-journald | Journal streaming and querying |
| `io.systemd.Udev` | udevd | udev device event streaming |
| `io.systemd.Network` | systemd-networkd | Network interface management |
| `io.systemd.ManagedOOM` | systemd-oomd | OOM policy and swap monitoring |
| `io.systemd.Home` | systemd-homed | User home area management |
| `io.systemd.DynamicUser` | systemd (PID 1) | Dynamic user/group allocation |
| `io.systemd.Credentials` | systemd (PID 1) | Encrypted service credentials |
| `io.systemd.BootControl` | systemd-boot | Boot loader menu and entry control |
| `io.systemd.sysext` | systemd-sysext | System extension image management |
| `io.systemd.DropIn` | systemd (PID 1) | Runtime unit drop-in management |
| `io.systemd.Import` | systemd-importd | Container/VM image import/export |

> **Note**: The exact set of interfaces grows with each systemd release. For the authoritative current list, search for `.varlink` files in the [systemd source tree](https://github.com/systemd/systemd) under `src/`. As of mid-2026, a GitHub issue (#36755) requesting centralised, up-to-date documentation is open; the interface names listed here are derived from the systemd source tree — per-interface introduction versions are not centrally documented and have been omitted to avoid inaccuracy.

### Using the `varlinkctl` Tool

systemd 255+ ships `varlinkctl` for introspecting and calling Varlink services directly from the shell — no boilerplate code required:

```bash
# List interfaces exposed by systemd-resolved
varlinkctl info /run/systemd/resolve/io.systemd.Resolve

# Introspect the full IDL of an interface
varlinkctl introspect /run/systemd/resolve/io.systemd.Resolve \
    io.systemd.Resolve

# Call a DNS resolution method
varlinkctl call \
    /run/systemd/resolve/io.systemd.Resolve \
    io.systemd.Resolve.ResolveHostname \
    '{"name":"freedesktop.org","family":2}'

# List all systemd units via the service manager Varlink interface
varlinkctl call \
    /run/systemd/private/io.systemd.Manager \
    io.systemd.Manager.ListUnits \
    '{}'

# Stream udev device events
varlinkctl call --more \
    /run/udev/io.systemd.Udev \
    io.systemd.Udev.Subscribe \
    '{}'
```

The `--more` flag enables the streaming/monitoring mode where a single Varlink call produces multiple response objects over time.

### Varlink Advantages for Point-to-Point IPC

Varlink offers several concrete advantages over D-Bus for point-to-point communication between system services:

1. **No central daemon in the hot path**: a Varlink call goes directly from client socket to service socket through the kernel. There is no broker process receiving, validating, and forwarding the message. This reduces latency by eliminating one round-trip through a daemon process.

2. **Trivial socket activation**: any systemd service can expose a Varlink socket via a `.socket` unit. The service starts on first connection. No `.service` file registration in a central directory is required beyond the socket unit itself.

3. **Implementation simplicity**: implementing a Varlink service requires only the ability to read/write JSON on a Unix socket. The protocol is implementable in a weekend in any language, without requiring a GLib or libdbus dependency.

4. **Human-readable debugging**: JSON on a socket is readable with standard tools. No binary serialization knowledge required.

5. **IDL is self-documenting**: the `.varlink` IDL file is the specification and the documentation simultaneously.

### Rust Varlink: the zlink Crate

The [zlink](https://github.com/z-galaxy/zlink) crate (version 0.5.0 as of May 2026, from the same `z-galaxy` organisation that maintains zbus) is the recommended Rust Varlink library. It is async-first, no-std-compatible, and provides code generation from `.varlink` IDL files.

```rust
// Varlink example: query DNS resolution via io.systemd.Resolve
// Using the zlink crate (varlink crate from z-galaxy)
// Source: https://github.com/z-galaxy/zlink (resolved.rs example)

// The zlink-generated code from io.systemd.Resolve.varlink would look like:
// (shown manually here for illustration)

use serde::{Deserialize, Serialize};
use std::os::unix::net::UnixStream;
use std::io::{BufRead, BufReader, Write};  // BufRead needed for read_until

const RESOLVE_SOCKET: &str = "/run/systemd/resolve/io.systemd.Resolve";

#[derive(Serialize)]
struct ResolveHostnameParams<'a> {
    name: &'a str,
    family: i32,  // AF_INET=2, AF_INET6=10
}

#[derive(Deserialize, Debug)]
struct ResolvedAddress {
    ifindex: Option<i32>,
    family: i32,
    address: Vec<u8>,  // base64-decoded by serde
}

#[derive(Deserialize, Debug)]
struct ResolveHostnameResult {
    addresses: Vec<ResolvedAddress>,
    canonical: String,
    flags: i32,
}

fn varlink_call(socket_path: &str, method: &str, params: serde_json::Value)
    -> serde_json::Result<serde_json::Value>
{
    let mut stream = UnixStream::connect(socket_path)
        .expect("failed to connect to Varlink socket");

    // Varlink request: {"method": "...", "parameters": {...}}
    let request = serde_json::json!({
        "method": method,
        "parameters": params,
    });
    let mut msg = serde_json::to_vec(&request)?;
    msg.push(0u8); // NUL terminator
    stream.write_all(&msg).expect("write failed");

    // Read response until NUL byte (Varlink framing: messages are NUL-terminated)
    let mut reader = BufReader::new(&stream);
    let mut buf: Vec<u8> = Vec::new();
    reader.read_until(0u8, &mut buf).expect("read failed");
    // strip trailing NUL
    if buf.last() == Some(&0u8) { buf.pop(); }
    serde_json::from_slice(&buf)
}

fn main() {
    let result = varlink_call(
        RESOLVE_SOCKET,
        "io.systemd.Resolve.ResolveHostname",
        serde_json::json!({"name": "freedesktop.org", "family": 2}),
    ).expect("Varlink call failed");

    println!("Resolved: {:#?}", result["parameters"]);
}
```

> **Note**: The raw socket example above illustrates the protocol structure. In production, use the `zlink` crate with generated types from the `.varlink` IDL; this handles NUL framing, error types, and async I/O correctly. The `varlink-inspect` example in zlink demonstrates introspecting `io.systemd.Resolve` directly.

### The logind Inflection Point for Compositor Authors

logind's D-Bus interface — `org.freedesktop.login1.Session.TakeDevice`, `PauseDevice`, `ResumeDevice` — is the seat management gatekeeper for every non-root Wayland compositor. It currently has no Varlink equivalent. This is the single most graphics-critical interface that remains D-Bus-only, and for good reason: `PauseDevice` and `ResumeDevice` are D-Bus signals sent simultaneously to all session controllers — a broadcast notification model that Varlink cannot currently replicate.

**However, the trajectory is clear**: systemd developers have stated publicly that Varlink is the direction for new APIs. As logind gains new capabilities (hidraw device handoff was added recently via D-Bus), it is plausible — and arguably likely — that future logind features will be exposed via Varlink, while the existing D-Bus API is maintained for backward compatibility.

**Practical guidance for compositor authors**: watch the systemd changelogs and `src/login/` in the systemd source tree for Varlink additions. A compositor that speaks only D-Bus will work for the foreseeable future, but new integration points (e.g. fine-grained device revocation policies, session tagging) may first appear as Varlink methods. Adopting the `zlink` crate or a minimal direct-socket implementation positions a compositor well for this transition.

### The Key Technical Blocker: No Broadcast Signals

Varlink's most significant architectural limitation relative to D-Bus is the **absence of broadcast signals**. D-Bus signals are 1:N: one sender emits a signal and all processes that have registered matching rules receive it simultaneously. Examples in the graphics stack:

- `org.freedesktop.login1.Session.PauseDevice` — logind broadcasts this to all session controllers when VT switches.
- `org.freedesktop.ColorManager.Changed` — colord broadcasts this to all compositors when a calibration profile changes.
- `org.freedesktop.StatusNotifierItem.*` — tray item signals go to all notification areas.

In Varlink, the closest mechanism is the **`more` streaming mode**: a client sends a request with `"more": true` and the service sends multiple response objects over the same connection. This is 1:1 push, not 1:N broadcast. There is no subscription/fan-out layer in the core protocol.

This is the principal architectural reason why Varlink cannot yet replace D-Bus for event-heavy interfaces. Until Varlink gains a multicast subscription mechanism (which would require either a central broker or a multi-subscriber socket model), D-Bus remains necessary for these interfaces.

**Speculation**: it is plausible that within the next few years, either Varlink will gain a subscription/multicast mechanism, or systemd will develop a thin brokering layer on top of Varlink for notification delivery — but as of mid-2026 this has not been announced or proposed. Until then, D-Bus is required wherever broadcast signals are needed.

---

## 9. Android Binder: IPC for the Graphics HAL

### What Binder Is

Android's IPC mechanism, Binder, differs architecturally from every other IPC system discussed in this chapter: it is **kernel-mediated**. Rather than routing messages through a userspace daemon, Binder transactions pass through the kernel driver at `drivers/android/binder.c`. This gives it lower latency, stronger security properties, and native file descriptor passing without any userspace involvement in the routing path.

Binder is capability-based: a Binder object reference is an unforgeable capability. If process A holds a reference to a Binder object in process B, A can call methods on it. A cannot forge a reference to a Binder object it was never given; the kernel enforces this through the reference table in the Binder driver.

### Three Device Nodes

Android exposes three separate Binder domains via device nodes:

| Device | Domain | Users |
|--------|--------|-------|
| `/dev/binder` | Framework/app domain | App ↔ Android framework (AMS, WMS, SurfaceFlinger) |
| `/dev/hwbinder` | HAL domain | Framework ↔ HAL (GPU, camera, audio) |
| `/dev/vndbinder` | Vendor domain | Vendor process ↔ vendor process |

The separation prevents apps from directly calling HAL services and prevents HAL services from calling app-layer APIs, enforcing the Android security boundary between the untrusted app sandbox, the trusted framework, and the vendor-provided hardware layer.

### Transaction Model

A Binder transaction is an ioctl (`BC_TRANSACTION` / `BR_TRANSACTION`) on the device fd. The client prepares a `binder_transaction_data` structure containing:
- The Binder object reference (target)
- A code identifying the method
- A flat byte buffer carrying serialised data
- A separate list of object offsets (for embedded Binder references and file descriptors)

File descriptors are passed via `binder_fd_object` records within the offset list. The kernel translates the fd number from sender to receiver address space automatically — the receiver's fd value is different from the sender's, but points to the same kernel file description.

### AIDL: Android Interface Definition Language

Rather than writing ioctl calls directly, Android uses AIDL to generate typed IPC stubs. `.aidl` files define interfaces:

```java
// ISurfaceComposer.aidl (simplified excerpt)
// Source: frameworks/native/libs/gui/aidl/android/gui/ISurfaceComposer.aidl
package android.gui;

import android.gui.IGraphicBufferProducer;
import android.gui.ComposerState;

interface ISurfaceComposer {
    /** Create a new Surface and return the producer interface. */
    IGraphicBufferProducer createSurface(
        in String name,
        int width,
        int height,
        int format,
        int flags
    );

    /** Apply a batch of layer state changes atomically. */
    void setTransactionState(
        in ComposerState[] state,
        int flags,
        long desiredPresentTime
    );
}
```

The AIDL compiler generates C++, Java, and Rust proxy/stub classes from this definition. The Rust AIDL toolchain (`aidl_interface` in the Android build system) generates crates with typed async methods, similar to zbus's `#[proxy]`.

### Stable AIDL (Android 11+)

Prior to Android 11, HAL interfaces were defined in HIDL (HAL Interface Definition Language), a separate system. Android 11 introduced **Stable AIDL** as the recommended approach for HALs that live in the vendor partition (`/vendor`) and are updated independently of the platform. Stable AIDL interfaces are versioned — each AIDL file has a declared version number and frozen snapshots — so the platform can verify ABI compatibility across independent updates. [Source](https://source.android.com/docs/core/architecture/aidl)

### Binder in the Android Graphics Stack

| Interface | AIDL File | Purpose |
|-----------|-----------|---------|
| `ISurfaceComposer` | `android.gui` | Create surfaces, atomic transaction state |
| `IGraphicBufferProducer` | `android.gui` | Buffer queue producer (app → SurfaceFlinger) |
| `IGraphicBufferConsumer` | `android.gui` | Buffer queue consumer (SurfaceFlinger side) |
| `IAllocator` | `android.hardware.graphics.allocator` | Gralloc buffer allocation (Stable AIDL HAL) |
| `IMapper` | `android.hardware.graphics.mapper` | Buffer lock/unlock/import (Stable AIDL HAL) |
| `IComponentStore` | `android.hardware.media.c2` | Codec2 hardware decoder/encoder HAL |

### libbinder_ndk

For NDK code (C/C++ targeting the stable NDK ABI), `libbinder_ndk` provides access to the Binder framework without linking against private Android system libraries. It is the recommended path for code in Chapter 164 (Android NDK). Rust code uses the `binder` crate, generated by the AIDL Rust backend. [Source](https://source.android.com/docs/core/architecture/aidl/aidl-backends)

### Comparison to D-Bus

| Property | D-Bus | Android Binder |
|----------|-------|----------------|
| Routing | Userspace daemon | Kernel driver |
| Latency | ~40–100 µs per round-trip (Note: workload-dependent; needs verification) | ~5–20 µs per round-trip (Note: workload-dependent; needs verification) |
| Security | UID-based, policy files | Capability references (unforgeable) |
| Type safety | Introspection XML | AIDL compiler enforces types |
| Broadcast | Signals (1:N) | Limited (IBinder.linkToDeath callbacks) |
| Fd passing | Unix ancillary data, via broker | Kernel-translated in transaction |

---

## 10. hyprwire and hyprtavern: The Hyprland IPC Ecosystem

### Background: Vaxry's D-Bus Critique

On December 15, 2025, Vaxry (Hyprland's lead developer) published ["D-Bus is a disgrace to the Linux desktop"](https://blog.vaxry.net/articles/2025-dbusSucks), a pointed critique of D-Bus. The central technical arguments:

1. **Type ambiguity**: the `a{sv}` (string-to-variant) dict used throughout freedesktop APIs permits sending "random shit over the wire and hope the other side understands it" — neither the broker nor the client can enforce what keys exist or what types the values hold.
2. **No security model by design**: "Everybody sees everything and calls whatever." The UID-based policy model requires explicit policy files; there is no built-in capability or sandbox model.
3. **Secret storage failure**: any application that has access to gnome-keyring after the keyring is unlocked can read all stored secrets, regardless of which app stored them.
4. **Scattered specifications**: even when specifications exist, there is no enforcement mechanism.

The post generated significant discussion across the Linux community. Whether or not one agrees with the severity of each critique, the post catalysed serious engineering work.

### hyprwire: The Wire Protocol

[hyprwire](https://github.com/hyprwm/hyprwire) is a binary wire protocol designed for fast, strict IPC. The key design goals stated in the project:

- **Strict**: both sides must agree on the protocol; there is no mechanism to send "random data." Unlike D-Bus, the protocol does not accommodate unknown message types or untyped payloads.
- **Fast**: the initial handshake is minimal.
- **Wayland-inspired**: the design borrows from Wayland's protocol model (object lifecycle, versioning) while discarding D-Bus's XML-introspection model.

hyprwire is already deployed as the IPC mechanism for several Hyprland ecosystem tools: hyprpaper (wallpaper manager), hyprlauncher (application launcher), and parts of `hyprctl` (the CLI control tool). As of April 2026, hyprwire v0.3.1 is the latest release. [Source](https://github.com/hyprwm/hyprwire)

Protocol specifications are maintained in the [hyprwire-protocols repository](https://github.com/hyprwm/hyprwire-protocols).

### hyprtavern: The Session Bus Layer

[hyprtavern](https://github.com/hyprwm/hyprtavern) is the layer above hyprwire, providing service discovery, a permission system, and a secure key-value store — the functional analogue of the D-Bus session bus plus gnome-keyring, redesigned from scratch.

Key differentiators from D-Bus (per the project README):
- **Built-in permission system**: apps can only access capabilities they have been explicitly granted; suitable for Flatpak sandboxes without requiring separate portal infrastructure.
- **Secure key-value store**: each application can only read its own stored secrets; the keyring model of "unlock once, any app can read everything" is explicitly rejected.
- **System-agnostic**: designed to work on systems without systemd; targets BSD portability.
- **Minimal dependencies**: only requires hyprwire.

hyprtavern is currently in early development. The README notes: "This project is still in early development. I'm working on adding docs and improving the protocol, but it's not set in stone yet." The project had 178 GitHub stars as of mid-2026. [Source](https://github.com/hyprwm/hyprtavern)

A Debian Intent-to-Package (ITP) bug was filed for hyprtavern in December 2025 (Debian bug #1122656), and one for hyprwire-protocols (bug #1122923), indicating distribution packaging interest.

### Architectural Differences from D-Bus

| Aspect | D-Bus | hyprwire/hyprtavern |
|--------|-------|---------------------|
| Wire format | Binary (typed) | Binary (strict, Wayland-inspired) |
| Central daemon | Yes (dbus-daemon or dbus-broker) | hyprtavern daemon (session scope) |
| Type enforcement | Weak (a{sv} escape hatch) | Strong at protocol level |
| Permission model | Policy files, UID-based | Built-in capability model |
| Introspection | XML (org.freedesktop.DBus.Introspectable) | Not yet documented |
| Broadcast signals | Yes (match rules) | Not yet specified |
| Portability | Linux, BSD, Windows | Linux primary; BSD targeted |

### Honest Assessment

hyprwire and hyprtavern are serious engineering efforts motivated by real technical critiques of D-Bus. The protocol strictness and security-first design address genuine pain points.

However, several caveats apply:

- **Hyprland-ecosystem scope only**: as of mid-2026, adoption is limited to the Hyprland compositor and its own tools. No major GNOME, KDE, or independent compositor has committed to the protocol.
- **Incomplete specification**: the wire format is not yet fully documented ("will be written once tavern's ready").
- **No broadcast signals yet**: this is a fundamental capability gap relative to D-Bus that must be resolved before hyprtavern can replace the session bus for event-heavy interfaces like seat management.
- **Community buy-in required**: replacing D-Bus as a cross-desktop standard requires consensus from KDE, GNOME, and the freedesktop.org community — a process that has historically taken years even for technically superior proposals.

For Hyprland users and developers, hyprwire is already production-relevant for compositor-specific IPC. For cross-desktop or cross-compositor work, D-Bus remains the only portable option.

---

## 11. BUS1: In-Kernel IPC in Rust

### History: KDBUS and the First BUS1 Proposal

In 2013–2015, Lennart Poettering, David Rheinsberg, and others proposed **KDBUS**, an in-kernel D-Bus implementation that would route messages through the kernel rather than userspace. After several years of review, KDBUS was rejected from the mainline Linux kernel on the grounds of complexity and insufficient justification for in-kernel implementation.

**BUS1** was proposed in 2016 as a clean-sheet alternative: not an in-kernel D-Bus, but a simpler kernel IPC primitive offering capability-based message passing. BUS1 also did not gain enough traction to be merged, and the team pivoted to dbus-broker — a high-performance userspace D-Bus implementation (see Section 3).

### 2026 Revival in Rust

In March–April 2026, David Rheinsberg posted to the Rust-for-Linux mailing list announcing a new implementation of BUS1 rewritten in Rust. As reported by both Phoronix and LWN ([LWN coverage](https://lwn.net/Articles/1065490/)), the announcement describes stripping the design back to fundamentals and reimplementing the kernel module in Rust. Rheinsberg's own words from the announcement:

> "The biggest change is that we stripped everything down to the basics and reimplemented the module in Rust."

The primary motivation for Rust is eliminating reference-count and object-lifetime bugs — a known source of kernel security vulnerabilities. Rheinsberg acknowledges that the C↔Rust bridge within the kernel introduces its own complexity.

### Core Design

BUS1's core design (consistent with the 2016 proposal) provides:

- **Kernel-mediated message passing**: messages are routed by the kernel, with no userspace daemon in the hot path.
- **Capability-based security**: a BUS1 peer handle is an unforgeable capability; you can only communicate with peers you hold a reference to.
- **File descriptor sharing**: fds can be passed in messages, translated by the kernel (analogous to Binder's `binder_fd_object`).
- **No central routing daemon**: peers communicate directly through the kernel, not through a shared socket monitored by a daemon.

This is similar in spirit to Android Binder but designed for the mainline Linux kernel ABI rather than Android's vendor-partitioned environment.

### Current Status (mid-2026)

BUS1 is in early development. It has not been submitted to the mainline kernel mailing list (`linux-kernel@vger.kernel.org`) for broader review. Rheinsberg is currently seeking help from the Rust-for-Linux community to improve the Rust/C integration. The project is hosted under [github.com/bus1](https://github.com/bus1/).

**Status**: not in mainline kernel; not production-ready; seeking community collaboration.

### Potential Graphics Relevance

If BUS1 were merged into the mainline kernel, it would offer a lower-latency alternative to D-Bus for latency-sensitive compositor IPC:

- **Seat management**: `TakeDevice`/`PauseDevice` could operate at kernel speed rather than through a userspace daemon.
- **Colour management**: profile change notifications could be delivered with kernel-mediated guarantees.
- **Buffer sharing**: kernel-mediated fd passing is already well-understood in the DMA-BUF ecosystem; BUS1 could extend this to service references.

However, D-Bus's advantages — ecosystem inertia, tooling, and introspection — would not simply disappear even if BUS1 were merged. A new IPC primitive requires new libraries, new tools, and new conventions before it can serve as a platform for graphics-stack integration.

---

## 12. Comparison and Selection Guide

### Feature Comparison Table

| Property | D-Bus (dbus-daemon) | D-Bus (dbus-broker) | Varlink | hyprwire/hyprtavern | Binder (Android) | BUS1 (proposed) |
|----------|--------------------|--------------------|---------|---------------------|-----------------|-----------------|
| **Transport** | Unix socket via daemon | Unix socket via broker | Unix socket, point-to-point | Unix socket via hyprtavern | Kernel driver ioctl | Kernel driver ioctl |
| **Central daemon** | Yes (single-threaded) | Yes (two-process) | No | Yes (hyprtavern session daemon) | No (kernel) | No (kernel) |
| **Type system** | Binary typed + a{sv} escape | Binary typed + a{sv} escape | JSON with IDL-defined types | Binary, strict (protocol-level) | Parcel binary, AIDL-typed | TBD |
| **Broadcast signals** | Yes (match rules) | Yes (match rules) | No | Not yet specified | Limited | TBD |
| **Permission model** | UID + policy XML files | UID + policy XML + audit | Filesystem socket permissions | Built-in capability model | Kernel capability (unforgeable refs) | Capability-based |
| **Language bindings** | libdbus, GLib, sd-bus, zbus, Qt, Python | Same (wire-compatible) | C, Rust, Go, Python, Java | C++ (native), others TBD | Java, C++, Rust (AIDL), NDK | TBD |
| **Production status** | Mature, still default on Debian/Ubuntu | Mature, default on Fedora/Arch | **Production** (all new systemd APIs since ~2020; 20+ interfaces) | Deployed (Hyprland ecosystem only) | Mature (Android) | Early development |
| **Best for** | Cross-desktop Linux desktop IPC | Same + security/auditability | New systemd-integrated services; all point-to-point compositor ↔ systemd calls | Hyprland compositor ecosystem | Android HAL/framework | Future low-latency Linux IPC |

### Decision Guide for Graphics/Compositor Work

**Use D-Bus (via dbus-broker) when**: you need to integrate with any existing freedesktop.org specification — logind seat management, colord colour profiles, xdg-desktop-portal screen capture, PipeWire session management, or system tray protocols. D-Bus is the only portable, cross-desktop option for these. Prefer **sd-bus** from libsystemd for C code and **zbus** for Rust.

**Use Varlink when**: you are writing tooling that integrates with systemd services. This covers DNS resolution (`io.systemd.Resolve`), user/group lookup (`io.systemd.UserDatabase`), container/image management (`io.systemd.MachineImage`), boot control (`io.systemd.BootControl`), unit management (`io.systemd.Manager`/`io.systemd.Unit`), and journal streaming (`io.systemd.Journal`). Varlink is production-deployed across all of these, simpler to implement than D-Bus, and has no central daemon in the call path. **For any new systemd service API added since ~2020, assume Varlink first.** The current limitation is the absence of broadcast signals — for notification-heavy interfaces, D-Bus is still required. Use the `zlink` crate in Rust or a minimal direct-socket JSON implementation in C.

**Use hyprwire/hyprtavern when**: you are developing for the Hyprland ecosystem specifically, or building a new Hyprland-native tool. Do not use it for cross-compositor features until the protocol stabilises and gains broader adoption.

**Use Binder when**: you are developing for Android — HAL integration, SurfaceFlinger interaction, or NDK code calling Android framework APIs. Binder is not available on mainline Linux.

**Watch BUS1 when**: you are interested in long-term evolution of Linux IPC primitives. Not production-ready. Follow the Rust-for-Linux mailing list and `github.com/bus1` for status.

**Avoid dbus-daemon** on new deployments: prefer dbus-broker where possible. On distributions where dbus-broker is not the default, it is typically available as a package and can be activated via systemd unit override.

---

## 13. Roadmap

### Near-term (6–12 months)

**dbus-broker becomes the universal default.** As of mid-2026 dbus-broker is the default session and system bus implementation on Fedora, Arch, openSUSE Tumbleweed, and Void Linux. Debian 13 (Trixie) and Ubuntu 26.04 are tracking adoption; the remaining holdout is Debian stable's conservatism around replacing a daemon that ships on every installed system. The `DBUS_SESSION_BUS_PID` and `DBUS_SESSION_BUS_ADDRESS` interfaces are wire-compatible, so no application changes are required. The two-process architecture (policy broker + privileged activator helper) should eliminate the last class of dbus-daemon CVEs (privilege escalation via the monolithic single-process model).

**Varlink broadcast signals proposal.** The single most-requested Varlink feature is broadcast notification (signals in D-Bus terms). Without it, Varlink cannot replace `org.freedesktop.login1` (seat add/remove events), `org.freedesktop.NetworkManager` (link state changes), or colord `Changed` signals — all of which are consumed by Wayland compositors. A kernel `io_uring`-backed multicast primitive has been proposed on the `systemd-devel` mailing list (2025): the sender writes to an eventfd; listening processes block on `io_uring IORING_OP_READ` against the fd. If accepted into systemd, this would close Varlink's primary gap against D-Bus for compositor use cases within a 1–2 release cycle.

**zbus 5.x stabilisation.** The zbus crate is the de-facto Rust D-Bus library and is tracking full support for both the D-Bus wire protocol and Varlink in a unified async API. The `zlink` sub-crate (Varlink client) stabilised in zbus 4.x; zbus 5 is expected to add `zlink` server-side support and a unified `Connection` abstraction that handles both D-Bus and Varlink over the same `tokio` event loop.

**hyprtavern 0.1 stable.** Vaxry's hyprtavern bus layer (the session-scoped IPC routing daemon for the Hyprland ecosystem) is targeting its first stable release alongside Hyprland 0.54. This will define the hyprtavern wire format, the capability model, and the service discovery mechanism. Third-party Hyprland plugins will be able to register as hyprtavern services rather than using the flat `hyprctl dispatch` Unix socket, enabling structured IPC between plugins for the first time.

### Medium-term (1–3 years)

**Varlink replaces remaining sd-bus compositor interfaces.** The current trajectory in systemd is to expose every new API as a Varlink interface first and consider D-Bus only if broadcast signals are required. Key graphics-adjacent targets: `io.systemd.Inhibitor` (suspend/resume coordination for screen lockers), `io.systemd.Session` (the full logind seat and session model), and `io.systemd.login` (VT switching). If `io.systemd.Session` gains Varlink support, compositors will be able to call `TakeDevice` and `PauseDevice` via Varlink rather than D-Bus, removing the `libdbus`/`sd-bus` dependency from the compositor's IPC path for seat management.

**BUS1 kernel submission.** David Rheinsberg's BUS1 in-kernel Rust IPC driver is targeting initial upstream submission to `drivers/misc` or a new `drivers/ipc` subsystem. The kernel Rust-for-Linux infrastructure (landed in 6.1, maturing through 6.x) is the prerequisite — BUS1 uses Rust's ownership model to enforce at compile time that kernel IPC references cannot be forged or outlive the owning process. If accepted, BUS1 would provide a `O_CLOEXEC`-capable `/dev/bus1` fd-based IPC with unforgeable capability references — the same security model as Android Binder but available on mainline Linux desktop. Expected first submission: 2026–2027 kernel cycle.

**xdg-desktop-portal Varlink migration.** The portal layer (Ch23) is the primary cross-compositor IPC surface for Flatpak and sandboxed applications. The portal team has discussed migrating from D-Bus to Varlink for the internal portal-implementation protocol (the portal frontend calls the portal backend over a private socket). If this lands, portal backends would no longer require a D-Bus service file and activation mechanism — simplifying compositor-specific portal implementations (the wlr-portal backend for wlroots, hyprland-xdg-desktop-portal, etc.).

**Android Binder on mainline Linux.** The `CONFIG_ANDROID_BINDER_IPC` option already builds in mainline kernel, but the userspace toolchain (`libbinder`, AIDL compiler) is tightly coupled to the Android build system. Projects like `binder-linux` (a standalone Binder userspace library for GNU/Linux) are gaining traction for running Android HALs in container environments (e.g., Waydroid, Anbox). If libbinder is packaged in major distributions, the hardware abstraction layer for camera, codecs, and display could be reused on Linux without Android's full stack — relevant for platforms like RISC-V and ARM SBCs shipping Android BSPs.

### Long-term

- **Unified IPC surface for the Wayland compositor ecosystem.** The current landscape (D-Bus for freedesktop specs, Varlink for systemd, Wayland protocols for window management, compositor-specific sockets for extensions) may converge toward a smaller set: Wayland protocol for rendering and input, Varlink+signals for system service integration, and compositor-native sockets (hyprwire / ext-ipc-v1 or similar Wayland protocol extension) for privileged compositor control. D-Bus would be retained only for legacy freedesktop compatibility.
- **BUS1 as Android Binder replacement on Linux desktop.** If BUS1 matures and gains broad distribution support, it could serve the same role as Binder on Android — a high-performance, capability-safe kernel IPC for HAL boundaries — without Android's driver infrastructure. Gralloc, V4L2, and codec HAL interfaces could potentially be expressed as BUS1 services rather than D-Bus/Varlink, with stronger isolation guarantees.
- **Varlink IDL tooling maturity.** Current Varlink tooling is minimal: `.varlink` files are parsed by hand in most implementations. As adoption grows, expect generated client stubs (analogous to `gdbus-codegen` or `aidl`) for C, Rust, Go, and Python, reducing Varlink integration from raw JSON socket I/O to type-safe generated API calls.

---

## 14. Integrations

This chapter connects to the following chapters throughout the book:

**Chapter 21 — Building Compositors with wlroots: Seat Management** (§6b): The `org.freedesktop.login1.Session.TakeDevice` D-Bus call is the mechanism by which every unprivileged Wayland compositor obtains an open DRM device fd. The sd-bus and zbus examples in Sections 4 and 7 of this chapter implement exactly the call described in Chapter 21's seat management discussion. libseat abstracts over this D-Bus call and the seatd alternative.

**Chapter 23 — Legacy and Sandboxed App Support: xdg-desktop-portal**: The `org.freedesktop.portal.ScreenCast` and `org.freedesktop.portal.RemoteDesktop` interfaces in the table in Section 1 are the D-Bus surface of xdg-desktop-portal. Chapter 23 describes the portal architecture and PipeWire stream handoff; this chapter describes the D-Bus plumbing underneath.

**Chapter 53 — Display Calibration with colord**: The `org.freedesktop.ColorManager` D-Bus interface connects compositor colour management to the KMS `GAMMA_LUT` property. Chapter 53 describes how ICC profiles are stored and retrieved; the `org.freedesktop.ColorManager.Changed` signal subscription in Section 5 of this chapter is how compositors discover profile changes.

**Chapter 38 — PipeWire**: PipeWire uses D-Bus for activation (the session bus `org.freedesktop.ReserveDevice1` and `org.freedesktop.portal.desktop` interfaces) and for session manager integration. Understanding D-Bus activation in Section 2 of this chapter explains how PipeWire services are started on demand.

**Chapter 78 — GameMode**: The `com.feralinteractive.GameMode` D-Bus interface (system bus) is how game launchers request CPU governor and GPU performance-mode changes. The D-Bus architecture in Section 2 and the sd-bus method-call pattern in Section 4 are directly applicable to GameMode client code.

**Chapter 132 — Wayland Security**: logind's `uaccess` udev tagging (which grants ACL access to `/dev/dri/card0` for the active seat) is the filesystem-level complement to the D-Bus TakeDevice mechanism. Chapter 132 covers the full access control model; this chapter describes the D-Bus call that is the authoritative path for compositor device access.

**Chapter 111 — Flatpak Graphics**: xdg-desktop-portal's security model for sandboxed applications uses D-Bus portals as the permission checkpoint. The `org.freedesktop.portal.Flatpak` interface and the portal permission model are the D-Bus surface described in Chapter 111. hyprtavern's built-in sandbox permission model (Section 10 of this chapter) is an attempt to improve on the portal approach.

**Chapter 164 — Android NDK**: `libbinder_ndk` is the stable NDK interface to the Android Binder system described in Section 9 of this chapter. Chapter 164 covers NDK graphics from the application perspective; this chapter describes the Binder IPC layer that `ISurfaceComposer`, `IAllocator`, and the Gralloc HAL sit on top of.

**Chapter 87 — ARCore and Android Camera HAL**: ARCore's camera access goes through the Android Camera HAL via `/dev/hwbinder` and the `ICameraProvider` AIDL interface. The Binder transaction model described in Section 9 of this chapter is the IPC substrate for the camera HAL interaction that ARCore depends on.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
