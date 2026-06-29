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
8. [Programming D-Bus: Qt (QDBus)](#8-programming-d-bus-qt-qdbus)
9. [Varlink: systemd's Simpler IPC](#9-varlink-systemds-simpler-ipc)
10. [Android Binder: IPC for the Graphics HAL](#10-android-binder-ipc-for-the-graphics-hal)
11. [hyprwire and hyprtavern: The Hyprland IPC Ecosystem](#11-hyprwire-and-hyprtavern-the-hyprland-ipc-ecosystem)
12. [BUS1: In-Kernel IPC in Rust](#12-bus1-in-kernel-ipc-in-rust)
13. [PipeWire Native Protocol](#13-pipewire-native-protocol)
14. [eBPF and io_uring: Modern IPC Acceleration](#14-ebpf-and-io_uring-modern-ipc-acceleration)
15. [D-Bus vs Wayland Protocol: When to Use Which](#15-d-bus-vs-wayland-protocol-when-to-use-which)
16. [Testing D-Bus and IPC Code](#16-testing-d-bus-and-ipc-code)
17. [Comparison and Selection Guide](#17-comparison-and-selection-guide)
18. [libzmq and the Jupyter Kernel Protocol](#18-libzmq-and-the-jupyter-kernel-protocol)
19. [Roadmap](#19-roadmap)
20. [Integrations](#20-integrations)

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

### Linux IPC Primitives: The Transport Substrate

Every IPC bus mechanism discussed in this chapter is built on top of lower-level Linux kernel IPC primitives. Understanding the primitives clarifies why each bus mechanism makes the design choices it does, and which primitive is optimal for a given throughput, latency, or semantic requirement.

| Mechanism | Kernel object | Max throughput | Typical latency | fd passing | Named | Persistence | Primary use in graphics stack |
|---|---|---|---|---|---|---|---|
| **Anonymous pipe** | `pipe2(O_CLOEXEC)` | 4–6 GB/s | 1–5 µs | No | No | Lifetime of fd pair | ffmpeg → Sixel encoder, shell pipelines |
| **Named pipe (FIFO)** | `mkfifo` / VFS inode | 4–6 GB/s | 1–5 µs | No | Yes (`/tmp/foo`) | Until unlinked | Rare; replaced by Unix sockets |
| **POSIX shared memory** | `shm_open` / `memfd_create` | Memory bus (~50 GB/s) | <1 µs (after setup) | Via fd | `/dev/shm/` name or anonymous | Until `shm_unlink` / fd GC | `wl_shm` pixel buffers, PipeWire memfds, GBM bo dmabuf |
| **socketpair** | Two connected `AF_UNIX SOCK_STREAM` sockets | 3–5 GB/s | 2–8 µs | Yes (SCM_RIGHTS) | No | Lifetime of both fds | D-Bus daemon↔client, Wayland display fd, zbus `Connection::pair()` tests |
| **Unix domain socket** | `AF_UNIX` bound to VFS path | 3–5 GB/s | 5–20 µs | Yes (SCM_RIGHTS) | Yes (`/run/...`) | Server holds bind | Wayland compositor socket, D-Bus bus socket, Varlink service sockets |
| **memfd + splice / io_uring** | `memfd_create` + `IORING_OP_SPLICE` | ~Memory bus | <1 µs (zero-copy) | Via fd | No | Until fd closed | DMA-BUF passing, PipeWire zero-copy, Ghostty PTY I/O |
| **Kernel Binder (`/dev/binder`)** | `drivers/android/binder.c` ioctl | 1–2 GB/s | 5–20 µs | Yes (native) | Via kernel ref table | Kernel ref counting | Android SurfaceFlinger, HAL transactions |
| **BUS1 (`/dev/bus1`, proposed)** | Rust kernel driver, ioctl | TBD (similar to Binder) | ~5–15 µs (estimated) | Yes (native) | Via kernel cap refs | Kernel ref counting | Future: Linux HAL isolation, compositor privilege separation |

**Throughput figures** are approximations for same-machine, same-NUMA-node transfers at message sizes of 4–64 kB on a modern x86_64 system. Real-world figures depend on message size, CPU frequency, kernel version, and whether the data is already in cache. For small messages (< 256 bytes), latency dominates throughput; for large transfers (> 1 MB), `memfd`/`splice` zero-copy paths become essential.

**How bus mechanisms relate to primitives:**

- **D-Bus and Varlink** run over **Unix domain sockets**. Every D-Bus method call is a `write()` of a serialised message to a Unix socket, routed by the broker (dbus-broker) using `sendmsg()` with `SCM_RIGHTS` for fd passing. The broker adds service discovery, naming, policy enforcement, and signal fan-out on top of the raw socket transport.
- **Wayland** uses a **Unix domain socket** (`/run/user/$UID/wayland-0`) for control messages (protocol events and requests), but pixel data moves through **shared memory** (`wl_shm`) or **DMA-BUF fds** (via `linux-dmabuf-v1`) — not through the Wayland socket itself.
- **PipeWire** uses a **Unix domain socket** for its native protocol (session management, graph topology) and **memfds** or **DMA-BUF fds** for the actual media data — the same split as Wayland.
- **Android Binder** bypasses the userspace socket layer entirely: transactions go through a kernel ioctl on `/dev/binder`, and the kernel copies (or maps) the data directly between process address spaces.
- **Anonymous pipes** appear in the graphics stack primarily as the stdin/stdout of external tools: a terminal emulator piping Sixel-encoded image data from `ffmpeg` or `img2sixel` reads it from the subprocess's stdout pipe. They are not used for service IPC.
- **memfd + io_uring splice** is the emerging zero-copy path for high-throughput IPC: data written into a `memfd` by one process can be `splice()`d into another's socket buffer without any userspace copy — or, with io_uring, enqueued as an `IORING_OP_SPLICE` submission without blocking. PipeWire uses this for low-latency audio buffer handoff; Ghostty uses io_uring for PTY I/O.

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

### Debugging with busctl

`busctl` is the systemd-provided D-Bus inspection and call tool, and the recommended replacement for the older `dbus-monitor` and `dbus-send` utilities. It is included in the `systemd` package on all major distributions.

**List all services on the session bus:**
```bash
busctl --user list
```
Output shows the unique name (`:1.42`), the well-known name (`org.freedesktop.portal.desktop`), PID, process name, and whether the service is activatable (i.e., has a `.service` file but is not yet running).

**Introspect an object's interfaces and methods:**
```bash
busctl --user introspect org.freedesktop.login1 /org/freedesktop/login1/session/auto
```
This calls `org.freedesktop.DBus.Introspectable.Introspect` and prints a human-readable table of interfaces, methods (with argument types), signals, and properties — including the direction (`→` for in, `←` for out).

**Call a method:**
```bash
# Ask logind for the active session path (no args, returns object path)
busctl --system call org.freedesktop.login1 \
    /org/freedesktop/login1 \
    org.freedesktop.login1.Manager \
    GetSessionByPID u $$

# Open a DRM device (major=226, minor=0 = /dev/dri/card0; read-only=false)
busctl --system call org.freedesktop.login1 \
    /org/freedesktop/login1/session/auto \
    org.freedesktop.login1.Session \
    TakeDevice uu 226 0
```
The argument format uses D-Bus type codes inline: `u` = uint32, `s` = string, `b` = boolean, `h` = Unix fd (returned, not passed). `busctl` unmarshals the reply and prints it as a typed value tree.

**Monitor all messages in real time:**
```bash
busctl --user monitor
# Filter to a specific service:
busctl --user monitor org.freedesktop.portal.desktop
```
This is the replacement for `dbus-monitor`; it uses the kernel-level socket monitor (`SO_ATTACH_FILTER`) to capture messages without requiring a match rule subscription, so it works even for point-to-point replies.

**Read a property:**
```bash
busctl --system get-property org.freedesktop.login1 \
    /org/freedesktop/login1/session/auto \
    org.freedesktop.login1.Session \
    State
# → s "active"
```

**Set a property:**
```bash
busctl --system set-property org.freedesktop.login1 \
    /org/freedesktop/login1/session/auto \
    org.freedesktop.login1.Session \
    IdleHint b false
```

**Capture to a pcap file for Wireshark analysis:**
```bash
busctl --system capture > /tmp/dbus.pcap
wireshark /tmp/dbus.pcap   # D-Bus dissector built-in since Wireshark 3.0
```

`busctl` is the primary tool for interactive D-Bus exploration and debugging when developing compositor seat management code or portal backends. For automated test assertions, `dbus-test-runner` (covered in §14) is more appropriate.

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

## 8. Programming D-Bus: Qt (QDBus)

Qt's D-Bus binding, `QDBus`, is part of the `QtDBus` module (shipped with every Qt5/Qt6 install). It is the backbone of KDE Plasma's IPC layer — every KDE application, Plasma shell component, and KWin compositor extension communicates via `QDBus`. Understanding it is essential for working with or extending any KDE-adjacent compositor or desktop integration code.

### Core Classes

| Class | Purpose |
|---|---|
| `QDBusConnection` | Represents a connection to a bus (session or system) |
| `QDBusInterface` | Proxy object for calling methods on a remote service |
| `QDBusMessage` | Raw message — used for low-level send/receive |
| `QDBusReply<T>` | Typed reply wrapper (wraps a `QDBusMessage`) |
| `QDBusAbstractInterface` | Base class for generated proxies (via `qdbusxml2cpp`) |
| `QDBusAbstractAdaptor` | Base class for server-side D-Bus object adaptor |

### Calling a Method: QDBusInterface

```cpp
#include <QDBusConnection>
#include <QDBusInterface>
#include <QDBusReply>
#include <QUnixFileDescriptor>

// Connect to the system bus and call TakeDevice on the active logind session
QDBusConnection bus = QDBusConnection::systemBus();
QDBusInterface session(
    "org.freedesktop.login1",
    "/org/freedesktop/login1/session/auto",
    "org.freedesktop.login1.Session",
    bus
);

// TakeDevice(major: u32, minor: u32) -> (fd: h, inactive: b)
QDBusReply<QDBusUnixFileDescriptor> reply =
    session.call("TakeDevice", QVariant((uint)226), QVariant((uint)0));

if (reply.isValid()) {
    int drm_fd = reply.value().fileDescriptor();
    // dup() before the QDBusUnixFileDescriptor goes out of scope
    int owned_fd = dup(drm_fd);
}
```

`QDBusUnixFileDescriptor` wraps a Unix fd received over D-Bus, handling the SCM_RIGHTS receive path. Qt handles fd lifetime: the descriptor is closed when the `QDBusUnixFileDescriptor` object is destroyed, so `dup()` it immediately if the fd needs to outlive the reply object.

### Subscribing to Signals

```cpp
// Subscribe to PauseDevice signal (sent by logind on VT switch)
QObject::connect(
    &session,
    SIGNAL(PauseDevice(uint, uint, QString)),
    this,
    SLOT(onPauseDevice(uint, uint, QString))
);

// The slot receives (major, minor, type) where type = "pause" or "force"
void MyCompositor::onPauseDevice(uint major, uint minor, const QString &type) {
    if (type == "pause") {
        drm_drop_master(drm_fd);
        session.call("PauseDeviceComplete", QVariant(major), QVariant(minor));
    }
}
```

Qt's signal/slot system maps directly onto D-Bus signals: `QDBusConnection::connect()` registers a match rule internally, and incoming signal messages are dispatched as Qt signals.

### Code Generation with qdbusxml2cpp

Qt provides `qdbusxml2cpp` to generate typed proxy and adaptor classes from D-Bus introspection XML. This is the recommended approach for stable, well-typed D-Bus interfaces:

```bash
# Generate a client proxy from logind's introspection XML
dbus-send --system --dest=org.freedesktop.login1 \
    --type=method_call --print-reply \
    /org/freedesktop/login1 \
    org.freedesktop.DBus.Introspectable.Introspect \
    2>/dev/null | grep -A9999 '<?xml' > logind_session.xml

qdbusxml2cpp -c LogindSessionInterface -p logind_session_iface logind_session.xml
# Generates: logind_session_iface.h / logind_session_iface.cpp
```

The generated `LogindSessionInterface` subclasses `QDBusAbstractInterface` and provides typed C++ methods (`takeDevice()`, `releaseDevice()`, etc.) and Qt signals (`PauseDevice`, `ResumeDevice`), eliminating all string-based method name dispatch.

### Server Side: QDBusAbstractAdaptor

```cpp
class ScreenCastAdaptor : public QDBusAbstractAdaptor {
    Q_OBJECT
    Q_CLASSINFO("D-Bus Interface", "org.freedesktop.portal.ScreenCast")
public:
    explicit ScreenCastAdaptor(QObject *parent) : QDBusAbstractAdaptor(parent) {}

Q_SIGNALS:
    void StreamsChanged();

public Q_SLOTS:
    uint CreateSession(const QVariantMap &options, QDBusMessage &message);
    uint SelectSources(const QDBusObjectPath &session_handle,
                       const QVariantMap &options, QDBusMessage &message);
    uint Start(const QDBusObjectPath &session_handle,
               const QString &parent_window,
               const QVariantMap &options, QDBusMessage &message);
};

// Register with the session bus:
QDBusConnection::sessionBus().registerObject("/org/freedesktop/portal/desktop",
                                              new ScreenCastImpl());
QDBusConnection::sessionBus().registerService("org.freedesktop.portal.desktop");
```

`QDBusAbstractAdaptor` subclasses attached to a `QObject` are automatically discovered by QtDBus. Slot signatures are introspected at runtime and exposed as D-Bus methods; `Q_SIGNALS` are exposed as D-Bus signals. The `QDBusMessage &message` parameter enables asynchronous replies (call `message.setDelayedReply(true)` and later send the reply manually — essential for portal methods that show a user-interaction dialog before responding).

### KDE Usage Patterns

KDE makes extensive use of the generated-proxy pattern. Key examples relevant to the Linux graphics stack:

- **KWin's D-Bus API** (`org.kde.KWin`): exposes window management, compositor effects, screen configuration, and screenshot APIs. Tools like `kwin_wayland --replace`, Spectacle (screenshot app), and KScreen all use QDBus to call into KWin.
- **KScreen/KRandR** (`org.kde.KScreen`): screen layout persistence and monitor hot-plug handling via D-Bus.
- **KWin Wayland scripts**: KWin's scripting API is exposed as a D-Bus interface, allowing external tools to register window rules or trigger effects dynamically.
- **Plasma Wayland session**: the `org.kde.KSplash`, `org.kde.KWinFT` (KWin with KFramework Theming), and `org.kde.plasmashell` services are all QDBus-based.

[Source: Qt QDBus documentation](https://doc.qt.io/qt-6/qtdbus-index.html)

---

## 9. Varlink: systemd's Simpler IPC

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

### Varlink in C: Direct Socket Implementation

For C compositor code, there is no widely-adopted Varlink C library (the original `libvarlink` is unmaintained). The protocol is simple enough to implement directly with POSIX sockets — the cost is writing the JSON serialiser/deserialiser, for which most C projects already have a library (`jansson`, `json-c`, or a lightweight custom parser).

```c
#include <sys/socket.h>
#include <sys/un.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>

/* Varlink: send a JSON call, receive a NUL-terminated JSON reply */
static int varlink_call(const char *socket_path,
                        const char *json_call,
                        char *reply_buf, size_t reply_max)
{
    struct sockaddr_un addr = { .sun_family = AF_UNIX };
    strncpy(addr.sun_path, socket_path, sizeof(addr.sun_path) - 1);

    int fd = socket(AF_UNIX, SOCK_STREAM | SOCK_CLOEXEC, 0);
    if (fd < 0) return -1;
    if (connect(fd, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        close(fd); return -1;
    }

    /* Varlink framing: message body + NUL byte terminator */
    size_t len = strlen(json_call);
    if (write(fd, json_call, len) != (ssize_t)len ||
        write(fd, "\0", 1) != 1) {
        close(fd); return -1;
    }

    /* Read until NUL byte */
    size_t n = 0;
    char c;
    while (n < reply_max - 1) {
        ssize_t r = read(fd, &c, 1);
        if (r <= 0) break;
        if (c == '\0') break;
        reply_buf[n++] = c;
    }
    reply_buf[n] = '\0';
    close(fd);
    return (int)n;
}

int main(void) {
    /* io.systemd.Resolve.ResolveHostname: resolve "localhost" A record */
    const char *call =
        "{\"method\":\"io.systemd.Resolve.ResolveHostname\","
        "\"parameters\":{\"name\":\"localhost\",\"family\":2}}";

    char reply[4096];
    int n = varlink_call("/run/systemd/resolve/io.systemd.Resolve",
                         call, reply, sizeof(reply));
    if (n > 0)
        printf("Reply: %s\n", reply);
    return 0;
}
```

Key points for C Varlink implementations:
- **NUL terminator**: the message boundary is a single `\0` byte appended after the JSON body. `read_until('\0')` is the framing rule — do not use line-oriented I/O.
- **`more` mode**: for streaming replies (multi-value responses), set `"more": true` in the request. The service sends multiple NUL-terminated JSON objects; the final reply omits `"continues": true`.
- **Error replies**: the service sends `{"error": "io.systemd.Resolve.NoNameServers", "parameters": {...}}` on failure — check for the `"error"` key before reading `"parameters"`.
- **Socket discovery**: Varlink service sockets live under `/run/systemd/` (system services) or `$XDG_RUNTIME_DIR/` (user services). For user services, resolve via `$DBUS_SESSION_BUS_ADDRESS` or `systemd-userdb` discovery.

For production C code, wrapping the above into a small `varlink_ctx` with epoll/`io_uring` support (for async calls without blocking the compositor event loop) is the recommended approach.

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

## 10. Android Binder: IPC for the Graphics HAL

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

### libbinder_ndk: C/C++ NDK Programming

For NDK code (C/C++ targeting the stable NDK ABI), `libbinder_ndk` provides access to the Binder framework without linking against private Android system libraries. [Source](https://source.android.com/docs/core/architecture/aidl/aidl-backends)

The `libbinder_ndk` API works at two levels: a low-level C API (`AIBinder_*`) for raw Binder object management, and a higher-level C++ wrapper generated by the AIDL compiler (`android/binder_auto_utils.h`).

#### Low-Level: Getting a Service Reference

```cpp
#include <android/binder_manager.h>
#include <android/binder_ibinder.h>
#include <android/binder_status.h>

// Obtain the SurfaceFlinger binder reference from the service manager
// This is the /dev/binder domain (framework services)
AIBinder* sf_binder = AServiceManager_waitForService("SurfaceFlinger");
if (!sf_binder) {
    // Service not yet registered; AServiceManager_waitForService blocks
    // until available or a timeout elapses
    return;
}

// Link a death recipient — called when SurfaceFlinger crashes/restarts
AIBinder_DeathRecipient* death = AIBinder_DeathRecipient_new(
    [](void* cookie) {
        // SurfaceFlinger died — trigger reconnection logic
        reinterpret_cast<MyApp*>(cookie)->onSurfaceFlingerDied();
    }
);
AIBinder_linkToDeath(sf_binder, death, this);

// Always decrement the refcount when done
AIBinder_decStrong(sf_binder);
AIBinder_DeathRecipient_delete(death);
```

#### AIDL-Generated C++ Stubs

For production code, the AIDL compiler generates typed C++ proxy classes. Given an AIDL interface `android.hardware.graphics.allocator.IAllocator` (the Gralloc4 HAL interface), the generated `BpAllocator` class wraps the raw Binder transaction:

```cpp
#include <aidl/android/hardware/graphics/allocator/BnAllocator.h>
#include <aidl/android/hardware/graphics/allocator/BpAllocator.h>
#include <android/binder_manager.h>

using namespace aidl::android::hardware::graphics::allocator;

// Get the IAllocator HAL service — on /dev/hwbinder domain
const std::string instance = std::string(IAllocator::descriptor) + "/default";
std::shared_ptr<IAllocator> allocator =
    IAllocator::fromBinder(ndk::SpAIBinder(
        AServiceManager_waitForService(instance.c_str())));

// Allocate a Gralloc buffer (64×64 RGBA_8888, GPU-renderable + CPU-readable)
AllocationResult result;
BufferDescriptorInfo info {
    .name = "my_buffer",
    .width = 64, .height = 64, .layers = 1,
    .format = PixelFormat::RGBA_8888,
    .usage = static_cast<int64_t>(
        BufferUsage::GPU_RENDER_TARGET | BufferUsage::CPU_READ_OFTEN),
};
ndk::ScopedAStatus status = allocator->allocate(info, 1, &result);
if (!status.isOk()) {
    // status.getDescription() returns AIDL error string
    ALOGE("allocate failed: %s", status.getDescription().c_str());
    return;
}
// result.buffers[0] is a NativeHandle (HAL_PIXEL_FORMAT buffer handle)
// Pass to IMapper.importBuffer() to get a mappable handle
```

[Source: Android AIDL backends documentation](https://source.android.com/docs/core/architecture/aidl/aidl-backends)

#### Rust: The `binder` Crate

Android's build system generates Rust crates from AIDL interfaces via the `aidl_interface` Soong rule with `backend: { rust: { enabled: true } }`. The generated Rust code uses the `binder` crate:

```rust
use binder::{Strong, IBinder};
use android_hardware_graphics_allocator::aidl::android::hardware::graphics::allocator::{
    IAllocator::IAllocator,
    AllocationResult::AllocationResult,
    BufferDescriptorInfo::BufferDescriptorInfo,
};
use android_hardware_graphics_common::aidl::android::hardware::graphics::common::{
    PixelFormat::PixelFormat,
    BufferUsage::BufferUsage,
};

// Get the IAllocator service — async-capable via tokio in Android's Rust runtime
let allocator: Strong<dyn IAllocator> =
    binder::get_interface::<dyn IAllocator>("android.hardware.graphics.allocator.IAllocator/default")
        .expect("IAllocator service not found");

// Allocate a buffer — the generated proxy marshals types into a Binder parcel
let info = BufferDescriptorInfo {
    name: "rust_buffer".to_string(),
    width: 64, height: 64, layers: 1,
    format: PixelFormat::RGBA_8888,
    usage: (BufferUsage::GPU_RENDER_TARGET as i64) | (BufferUsage::CPU_READ_OFTEN as i64),
    ..Default::default()
};
let result: AllocationResult = allocator.allocate(&info, 1)
    .expect("allocate failed");
// result.buffers[0]: NativeHandle
```

The `binder` crate is part of the Android source tree (`libs/binder/rust/`) and is not published to crates.io — it is built exclusively through the Android build system. For Rust code targeting the NDK ABI outside the Android build system, `libbinder_ndk` C FFI bindings via the `bindgen`-generated `binder_ndk_sys` crate are the alternative.

#### Monitoring Binder Transactions

```bash
# Show all active Binder services (like busctl list for D-Bus)
adb shell service list | grep -E "SurfaceFlinger|allocator|mapper"

# Dump SurfaceFlinger state via its Binder interface
adb shell dumpsys SurfaceFlinger | head -50

# Trace Binder transactions in real time (requires root or userdebug build)
adb shell binder-trace start
adb shell binder-trace stop > /tmp/binder.trace

# systrace with Binder tags — shows transaction timing alongside GPU/CPU
adb shell atrace --async_start -c -b 32768 binder_driver binder_lock gfx
```

`dumpsys SurfaceFlinger` is the equivalent of `busctl introspect` — it calls the `DUMP_TRANSACTION` Binder code on the SurfaceFlinger proxy and returns a human-readable state dump including layer list, buffer queue depths, vsync timing, and GPU composition statistics.

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

## 11. hyprwire and hyprtavern: The Hyprland IPC Ecosystem

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

### hyprctl: The Stable CLI and Socket API

While the hyprwire binary protocol and hyprtavern daemon are still evolving, `hyprctl` — Hyprland's control CLI — exposes a **stable Unix socket interface** that is usable today. Its socket protocol is simple enough to drive directly from shell or C code.

**Socket path:**
```bash
# The socket lives at:
$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket.sock   # request socket
$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket2.sock  # event socket
```

**hyprctl command examples:**
```bash
# List all windows as JSON (compositor state query)
hyprctl clients -j

# Move focus to the right monitor
hyprctl dispatch focusmonitor +1

# Set an active window's opacity (useful for compositor effects)
hyprctl setprop address:0x$(hyprctl activewindow -j | jq -r '.address') alpha 0.85 lock

# Reload the compositor config without restart
hyprctl reload

# Execute a keyword change live (e.g. change gap size)
hyprctl keyword general:gaps_in 4

# Get current monitor configuration as JSON
hyprctl monitors -j

# Trigger a custom user event (for hyprwire plugin communication)
hyprctl dispatch exec "notify-send 'hello from hyprctl'"
```

**Raw socket protocol (the transport beneath hyprctl):**
```bash
# hyprctl is a thin wrapper over a Unix socket; you can call it directly:
SOCK="$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket.sock"

# Send a text command and read the reply (newline-terminated)
echo -n "j/clients" | socat - UNIX-CONNECT:$SOCK

# The format is: [flags/]<command>
# 'j' flag = JSON output; without it, human-readable text is returned
echo -n "clients"   | socat - UNIX-CONNECT:$SOCK  # text
echo -n "j/clients" | socat - UNIX-CONNECT:$SOCK  # JSON
```

**Event socket: subscribing to compositor events:**
```bash
# The .socket2.sock pushes newline-delimited events continuously
SOCK2="$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket2.sock"
socat - UNIX-CONNECT:$SOCK2
# Output example:
# activewindow>>kitty,nvim
# workspace>>2
# focusedmon>>DP-1,2
# openwindow>>0x55a1b2c3,2,kitty,/home/user
# closewindow>>0x55a1b2c3
# movewindow>>0x55a1b2c3,2
```

Event names are plain strings; values are `>>` separated. This is the protocol that tools like `waybar` (the status bar) and `hyprpaper` use to react to workspace or window changes.

**C: raw socket client:**
```c
#include <sys/socket.h>
#include <sys/un.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>

int hyprctl_call(const char *cmd, char *out, size_t out_len) {
    char path[256];
    const char *sig = getenv("HYPRLAND_INSTANCE_SIGNATURE");
    const char *xdg = getenv("XDG_RUNTIME_DIR");
    snprintf(path, sizeof(path), "%s/hypr/%s/.socket.sock", xdg, sig);

    struct sockaddr_un addr = { .sun_family = AF_UNIX };
    strncpy(addr.sun_path, path, sizeof(addr.sun_path) - 1);

    int fd = socket(AF_UNIX, SOCK_STREAM, 0);
    connect(fd, (struct sockaddr *)&addr, sizeof(addr));
    write(fd, cmd, strlen(cmd));
    shutdown(fd, SHUT_WR);   /* signal end of request */

    ssize_t n = read(fd, out, out_len - 1);
    out[n > 0 ? n : 0] = '\0';
    close(fd);
    return (int)n;
}

int main(void) {
    char buf[65536];
    hyprctl_call("j/monitors", buf, sizeof(buf));
    printf("%s\n", buf);   /* JSON array of monitor objects */
    return 0;
}
```

**Rust: event loop for .socket2:**
```rust
use std::os::unix::net::UnixStream;
use std::io::{BufRead, BufReader};

fn main() {
    let sig = std::env::var("HYPRLAND_INSTANCE_SIGNATURE").unwrap();
    let xdg = std::env::var("XDG_RUNTIME_DIR").unwrap();
    let path = format!("{}/hypr/{}/.socket2.sock", xdg, sig);

    let stream = UnixStream::connect(&path).expect("Hyprland not running");
    let reader = BufReader::new(stream);

    for line in reader.lines() {
        let event = line.unwrap();
        // Events arrive as "eventname>>value"
        if let Some((name, data)) = event.split_once(">>") {
            match name {
                "workspace"     => println!("Workspace changed to: {}", data),
                "activewindow"  => println!("Active window: {}", data),
                "openwindow"    => println!("New window opened: {}", data),
                "closewindow"   => println!("Window closed: {}", data),
                "monitoradded"  => println!("Monitor added: {}", data),
                "monitorremoved"=> println!("Monitor removed: {}", data),
                _               => {}
            }
        }
    }
}
```

The `hyprland-rs` crate ([crates.io/crates/hyprland](https://crates.io/crates/hyprland)) wraps both sockets with an async Tokio API, generated event enum, and typed JSON deserialisation for all `hyprctl` output objects.

### Honest Assessment

hyprwire and hyprtavern are serious engineering efforts motivated by real technical critiques of D-Bus. The protocol strictness and security-first design address genuine pain points.

However, several caveats apply:

- **Hyprland-ecosystem scope only**: as of mid-2026, adoption is limited to the Hyprland compositor and its own tools. No major GNOME, KDE, or independent compositor has committed to the protocol.
- **Incomplete specification**: the wire format is not yet fully documented ("will be written once tavern's ready").
- **No broadcast signals yet**: this is a fundamental capability gap relative to D-Bus that must be resolved before hyprtavern can replace the session bus for event-heavy interfaces like seat management.
- **Community buy-in required**: replacing D-Bus as a cross-desktop standard requires consensus from KDE, GNOME, and the freedesktop.org community — a process that has historically taken years even for technically superior proposals.

For Hyprland users and developers, hyprwire is already production-relevant for compositor-specific IPC. For cross-desktop or cross-compositor work, D-Bus remains the only portable option.

---

## 12. BUS1: In-Kernel IPC in Rust

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

## 13. PipeWire Native Protocol

PipeWire is the universal media server on modern Linux — it handles audio (replacing PulseAudio and JACK), video capture, and screen capture. Its IPC model is distinct from both D-Bus and Wayland: PipeWire defines its own binary protocol over a Unix domain socket, with D-Bus used only for service activation and portal integration. Understanding the native protocol is essential for graphics-adjacent work: screen capture (via the `org.freedesktop.portal.ScreenCast` portal → PipeWire stream), camera sharing, and GPU-accelerated video processing all go through it.

### Architecture Overview

```
Application / Compositor
        │
        │  PipeWire native protocol (Unix socket: /run/pipewire-0)
        ▼
  pipewire (session daemon)
        │
        ├── wireplumber (session manager, policy — communicates via PW protocol)
        │
        ├── pipewire-pulse (PulseAudio compatibility — /run/user/$UID/pulse/native)
        │
        ├── pipewire-jack (JACK compatibility — libpipewire-jack intercept)
        │
        └── pipewire-v4l2 (V4L2 compatibility shim)
```

D-Bus role: `pipewire.service` (or the user session unit) is activated on demand; `wireplumber` registers `org.freedesktop.ReserveDevice1` on the session bus to prevent ALSA/JACK conflicts. Once running, all real communication uses the PipeWire native protocol.

### The Native Protocol Wire Format

PipeWire's protocol is a **binary, little-endian, type-tagged message format** sent over a Unix domain socket. It is not based on D-Bus, Varlink, or Wayland. [Source: PipeWire source, `src/pipewire/protocol-native.c`](https://gitlab.freedesktop.org/pipewire/pipewire/-/blob/master/src/pipewire/protocol-native.c)

Each message is a **Pod** (Plain Old Data) encoded with PipeWire's pod serialiser (`spa_pod`). A pod is a tagged union of:
- `SPA_TYPE_Int`, `SPA_TYPE_Long`, `SPA_TYPE_Float`, `SPA_TYPE_Double`
- `SPA_TYPE_String` (NUL-terminated)
- `SPA_TYPE_Bytes` (raw buffer)
- `SPA_TYPE_Object` (named key-value pairs, like a D-Bus `a{sv}` but typed)
- `SPA_TYPE_Struct` (ordered sequence of typed fields)
- `SPA_TYPE_Fd` (file descriptor, carried via `SCM_RIGHTS`)
- `SPA_TYPE_Choice` (value + allowed range/enum for negotiation)

```c
/* spa/include/spa/pod/pod.h — the pod header layout */
struct spa_pod {
    uint32_t size;   /* body size in bytes, NOT including this header */
    uint32_t type;   /* SPA_TYPE_* tag */
    /* body follows immediately */
};
```

Messages are framed with an 8-byte header (`id` + `opcode` + `size` + `seq`) followed by a pod body. The protocol is **object-centric**: each connection is modelled as a graph of `pw_proxy` objects (client side) and `pw_global` objects (server side), identified by uint32 IDs.

### Core Protocol Objects

| Object type | Server global | Client proxy | Purpose |
|---|---|---|---|
| `pw_core` | `pw_context` | `pw_core` | Root connection object; `sync`, `error`, `get_registry` |
| `pw_registry` | Global list | `pw_registry` | Enumerate and bind global objects |
| `pw_node` | Audio/video node | `pw_node` | A processing element (source, sink, filter) |
| `pw_port` | Node port | `pw_port` | Input or output port on a node |
| `pw_link` | Graph edge | `pw_link` | Connection between two ports |
| `pw_client` | Peer client | — | Metadata about a connected client |
| `pw_device` | Hardware device | `pw_device` | ALSA card, V4L2 device, libcamera device |
| `pw_stream` | — | `pw_stream` | High-level streaming abstraction (most apps use this) |
| `pw_filter` | — | `pw_filter` | Low-latency processing filter node |

### pw_stream: The Application-Level API

Most application code uses `pw_stream` rather than the raw protocol objects. `pw_stream` abstracts the connection, port negotiation, buffer allocation, and data callback:

```c
#include <pipewire/pipewire.h>
#include <spa/param/video/format-utils.h>

static void on_process(void *userdata)
{
    struct pw_stream *stream = userdata;
    struct pw_buffer *buf = pw_stream_dequeue_buffer(stream);
    if (!buf) return;

    struct spa_buffer *sb = buf->buffer;
    /* sb->datas[0].data  — pointer to raw frame data (CPU mapping)
       sb->datas[0].fd    — DMA-BUF fd for zero-copy GPU import (if type = MemFd/DmaBuf) */
    void *frame = sb->datas[0].data;
    uint32_t stride = sb->datas[0].chunk->stride;

    /* Process frame... */

    pw_stream_queue_buffer(stream, buf);
}

static const struct pw_stream_events stream_events = {
    PW_VERSION_STREAM_EVENTS,
    .process = on_process,
};

/* Connect a video capture stream (screen cast source) */
struct pw_main_loop *loop = pw_main_loop_new(NULL);
struct pw_context *ctx = pw_context_new(pw_main_loop_get_loop(loop), NULL, 0);
struct pw_core *core = pw_context_connect(ctx, NULL, 0);

struct pw_stream *stream = pw_stream_new(core, "my-capture",
    pw_properties_new(PW_KEY_MEDIA_TYPE, "Video",
                      PW_KEY_MEDIA_CATEGORY, "Capture",
                      PW_KEY_MEDIA_ROLE, "Screen", NULL));
pw_stream_add_listener(stream, &listener, &stream_events, stream);

/* Negotiate format: BGRA at any resolution */
uint8_t buffer[1024];
struct spa_pod_builder b = SPA_POD_BUILDER_INIT(buffer, sizeof(buffer));
const struct spa_pod *params[] = {
    spa_format_video_raw_build(&b, SPA_PARAM_EnumFormat,
        &SPA_VIDEO_INFO_RAW_INIT(.format = SPA_VIDEO_FORMAT_BGRA))
};
pw_stream_connect(stream, PW_DIRECTION_INPUT, PW_ID_ANY,
    PW_STREAM_FLAG_AUTOCONNECT | PW_STREAM_FLAG_MAP_BUFFERS,
    params, 1);

pw_main_loop_run(loop);
```

[Source: PipeWire examples, `src/examples/video-capture.c`](https://gitlab.freedesktop.org/pipewire/pipewire/-/blob/master/src/examples/video-capture.c)

### Buffer Types and Zero-Copy

PipeWire negotiates the buffer type during stream connection. The `spa_data` type field determines how the buffer is shared:

| `spa_data_type` | Backing memory | GPU import path | Zero-copy? |
|---|---|---|---|
| `SPA_DATA_MemPtr` | `malloc` in server | CPU memcpy required | No |
| `SPA_DATA_MemFd` | `memfd_create` | mmap + EGL `EGL_EXT_image_dma_buf_import` via CPU upload | Partial |
| `SPA_DATA_DmaBuf` | GBM buffer object | Direct DMA-BUF fd import → `eglCreateImageKHR(EGL_LINUX_DMA_BUF_EXT)` | **Yes** |

When the producer (e.g. a compositor's screen capture) and the consumer (e.g. a recording application) both support `SPA_DATA_DmaBuf`, the frame data never leaves GPU memory — the compositor renders to a GBM BO, PipeWire passes the DMA-BUF fd via `SCM_RIGHTS`, and the consumer imports it directly into its EGL context. This is the zero-copy screen capture path used by OBS Studio, wf-recorder, and the xdg-desktop-portal ScreenCast implementation.

### D-Bus Integration: Activation and Portal Handoff

PipeWire's D-Bus presence is minimal but critical:

1. **Service activation**: `pipewire.service` and `wireplumber.service` are systemd user units; on D-Bus-activated systems, `pipewire-pulse.service` activates when an application connects to the PulseAudio-compatible socket.
2. **Device reservation**: `wireplumber` holds `org.freedesktop.ReserveDevice1.Audio0` on the session bus, preventing JACK or `arecord` from claiming ALSA devices while PipeWire is running.
3. **Screen cast portal handoff**: when `xdg-desktop-portal-wlr` or `xdg-desktop-portal-gnome` handles a `org.freedesktop.portal.ScreenCast.Start` D-Bus call, it:
   - Creates a PipeWire stream in the compositor's context
   - Returns the PipeWire **node ID** and **fd** (the socket fd for the PipeWire connection) to the calling application via the D-Bus reply
   - The application connects to PipeWire using the provided fd and binds to the node ID — bypassing service discovery entirely

This fd-passing handoff is why screen capture works in sandboxed Flatpak applications: the Flatpak sandbox cannot connect to the PipeWire socket directly, but the portal (running outside the sandbox) connects on the app's behalf and passes the already-connected fd through the D-Bus reply.

### Wireplumber: Session Policy via Lua

`wireplumber` is the session manager responsible for routing policy (which streams connect to which devices) and policy scripting. It communicates with `pipewire` exclusively over the PipeWire native protocol — not D-Bus. Policy logic is written in **Lua scripts** loaded from `/usr/share/wireplumber/` and `~/.config/wireplumber/`.

```lua
-- wireplumber/main.lua.d/51-alsa-jumbo.lua
-- Route all audio to the headphone output when plugged in
rule = Rule {
    matches = { { { "node.name", "matches", "alsa_output.*" } } },
    apply_properties = { ["audio.channels"] = 2 }
}
rule:apply()
```

For graphics-adjacent use cases (routing a compositor's screen capture node to a recording tool's input), wireplumber policy rules determine the default connection. Compositors that implement `wp_export_dmabuf_manager` or `ext-image-capture-source-v1` feed into PipeWire nodes managed by wireplumber.

---

## 14. eBPF and io_uring: Modern IPC Acceleration

eBPF and io_uring are two Linux kernel features that fundamentally change the performance and observability characteristics of IPC-heavy applications. Neither replaces D-Bus or Varlink — they accelerate or instrument the underlying socket layer on which those protocols run.

### eBPF and D-Bus: Observability Without Overhead

**eBPF** (extended Berkeley Packet Filter) allows safe, JIT-compiled programs to run in the kernel in response to events — including socket send/receive, system calls, and tracepoints. For D-Bus and Varlink debugging, eBPF provides observability that `busctl monitor` cannot: it can trace at the `write()`/`sendmsg()` syscall level without requiring a match rule subscription on the bus.

#### Tracing D-Bus Method Calls with bpftrace

```bash
# Trace all write() calls to any socket that looks like a D-Bus message
# (D-Bus messages start with 'l' (0x6c) for little-endian or 'B' for big-endian)
bpftrace -e '
tracepoint:syscalls:sys_enter_write
/comm == "dbus-broker"/
{
    printf("pid=%d size=%d\n", pid, args->count);
}'
```

A more sophisticated approach uses the `sock:inet_sock_set_state` or `sock:unix_sock_connect` tracepoints to track when compositors connect to the D-Bus socket, then correlates `sendmsg()` calls with process names to build a per-process D-Bus traffic profile.

#### eBPF-Based D-Bus Policy Enforcement

**BPF LSM** (Linux Security Module hooks via eBPF, available since Linux 5.7) enables dynamic D-Bus-adjacent security policy. Rather than configuring static policy XML for dbus-broker, a BPF LSM program can:
- Block `connect()` to `/run/dbus/system_bus_socket` from specific cgroups or UID ranges
- Intercept `sendmsg()` on the D-Bus socket and inspect the first bytes of the message header to detect specific interface names
- Emit audit events to `bpf_ringbuf` when a sandboxed process attempts a privileged D-Bus call

This is more flexible than dbus-broker's XML policy (which can only match on service names and interface names, not on calling process cgroup or container namespace) and does not require a policy reload to take effect.

```c
/* BPF LSM: deny connect to dbus system socket from cgroup != trusted */
SEC("lsm/socket_connect")
int BPF_PROG(restrict_dbus_connect, struct socket *sock,
             struct sockaddr *address, int addrlen)
{
    struct sockaddr_un *un = (struct sockaddr_un *)address;
    if (sock->type != SOCK_STREAM) return 0;

    /* Check if connecting to the D-Bus system socket */
    const char dbus_path[] = "/run/dbus/system_bus_socket";
    char path[108];
    bpf_probe_read_kernel_str(path, sizeof(path), un->sun_path);
    if (__builtin_memcmp(path, dbus_path, sizeof(dbus_path) - 1) != 0)
        return 0;

    /* Allow only processes in the trusted cgroup */
    uint64_t cgroup_id = bpf_get_current_cgroup_id();
    if (cgroup_id != TRUSTED_CGROUP_ID)
        return -EPERM;
    return 0;
}
```

#### eBPF for Varlink Monitoring

Since Varlink uses plain Unix domain sockets with NUL-terminated JSON, eBPF can parse Varlink messages directly in the kernel. A `uprobe` on `write()` combined with a BPF map can:
- Count calls per Varlink method (by scanning for the `"method":` JSON key in the first 256 bytes of each message)
- Measure latency between `sendmsg()` and the matching `recvmsg()` on the server side
- Export per-method histograms to userspace via `bpf_ringbuf` for continuous monitoring

This level of observability is impossible with Varlink's current tooling (varlinkctl cannot monitor live traffic on an established connection).

### io_uring and IPC: Latency Reduction for Socket-Heavy Applications

**io_uring** (Linux 5.1+) is an asynchronous I/O interface based on shared-memory submission and completion rings between userspace and the kernel. For IPC-heavy applications (compositors, media servers, TUI frameworks) it reduces the per-operation overhead from a `read()`/`write()` syscall pair to a single `io_uring_enter()` that drains the entire submission queue — or, with `SQPOLL` mode, zero syscalls per operation.

#### io_uring and the Wayland Socket

A Wayland compositor's event loop typically `poll()`s or `epoll_wait()`s on the Wayland display socket fd, processes incoming requests, and `write()`s events back. With io_uring:

```c
struct io_uring ring;
io_uring_queue_init(32, &ring, 0);

/* Register the Wayland client fd to avoid repeated fd table lookups */
int fds[] = { wl_display_get_fd(display) };
io_uring_register_files(&ring, fds, 1);

/* Submit a multishot read — one SQE, continuous CQEs until fd is closed */
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read_multishot(sqe, 0 /* registered fd index */, buf, sizeof(buf), 0);
io_uring_sqe_set_flags(sqe, IOSQE_FIXED_FILE);
io_uring_submit(&ring);

/* Event loop: drain CQEs without poll() */
struct io_uring_cqe *cqe;
while (io_uring_wait_cqe(&ring, &cqe) == 0) {
    handle_wayland_data(buf, cqe->res);
    io_uring_cqe_seen(&ring, cqe);
}
```

`IORING_OP_READ_MULTISHOT` (Linux 6.1+) issues a single submission and generates a CQE for every available read, eliminating the `poll()` → `read()` → `poll()` pattern. For a compositor handling hundreds of Wayland clients, this significantly reduces syscall count under high message rates.

#### io_uring and D-Bus / Varlink Sockets

The same multishot read pattern applies to D-Bus and Varlink sockets. An sd-bus or zbus application can use io_uring to:
- Read incoming D-Bus messages without blocking the event loop thread
- Submit `write()` operations for outgoing method calls and signal emissions as SQEs, batching them into a single `io_uring_submit()` call
- Use `IORING_OP_SEND_ZC` (zero-copy send, Linux 6.0+) for large D-Bus messages (e.g. `org.freedesktop.portal.ScreenCast` returning a large PipeWire fd bundle) to avoid a userspace copy into the kernel socket buffer

For Varlink, where each call is a short NUL-terminated JSON string (~100–500 bytes), the fixed-buffer registered I/O path (`io_uring_register_buffers`) eliminates per-call buffer registration overhead:

```c
/* Pre-register a fixed buffer for Varlink call/reply cycles */
char buf[4096];
struct iovec iov = { .iov_base = buf, .iov_len = sizeof(buf) };
io_uring_register_buffers(&ring, &iov, 1);

/* Issue fixed-buffer read for NUL-terminated Varlink reply */
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read_fixed(sqe, varlink_fd, buf, sizeof(buf), 0, 0 /* buf index */);
io_uring_submit(&ring);
```

#### io_uring and PipeWire

PipeWire's event loop (`pw_loop`) uses `epoll` internally. A PipeWire application can substitute an io_uring-backed loop via the `spa_loop` abstraction. The most impactful use case is the `SPA_DATA_DmaBuf` zero-copy path:

```
Producer (compositor GBM BO render)
    │
    │  io_uring IORING_OP_SEND_ZC  ──►  PipeWire socket (passes DMA-BUF fd via SCM_RIGHTS)
    │
Consumer (OBS, wf-recorder)
    │
    │  eglCreateImageKHR(EGL_LINUX_DMA_BUF_EXT)  ──►  GPU texture (zero-copy import)
```

The splice path (`IORING_OP_SPLICE`) can pipe DMA-BUF-backed memfds between PipeWire nodes without any userspace copies, reducing the latency of the screen-capture → encoding pipeline to near-zero memory-bandwidth cost.

#### SQPOLL for Ultra-Low-Latency IPC

For latency-critical IPC paths (compositor ↔ GPU driver communication, real-time audio with PipeWire), `io_uring`'s `IORING_SETUP_SQPOLL` mode runs a kernel thread that continuously polls the submission ring without requiring any userspace `io_uring_enter()` syscall:

```c
struct io_uring_params params = {
    .flags = IORING_SETUP_SQPOLL,
    .sq_thread_idle = 2000,  /* poll for 2ms before sleeping */
};
io_uring_queue_init_params(64, &ring, &params);
```

With SQPOLL, a userspace loop that continuously submits Wayland events or PipeWire buffer notifications to the kernel operates with **zero syscall overhead** for the I/O path — only the `io_uring_get_sqe()` / `io_uring_sqe_set_*()` / ring write sequence remains, which is pure userspace memory operations.

SQPOLL requires `CAP_SYS_NICE` (or `io_uring` being allowed by seccomp policy) and is most appropriate for the compositor's own event loop, not for general-purpose IPC clients. Ghostty uses io_uring (without SQPOLL) for its PTY I/O path (Ch178); wlroots-based compositors have experimental io_uring backends for Wayland socket dispatch.

### Summary: eBPF + io_uring in the IPC Stack

| Tool | Layer | Purpose in IPC context |
|---|---|---|
| **bpftrace / eBPF tracepoints** | Kernel syscall | Observe D-Bus / Varlink traffic without match rules; latency histograms |
| **BPF LSM** | Kernel security | Dynamic socket-level policy: block connects, audit message content |
| **uprobe + BPF map** | Userspace+kernel boundary | Parse Varlink JSON methods, count per-method calls, export to userspace |
| **io_uring IORING_OP_READ_MULTISHOT** | Kernel async I/O | Reduce `poll`→`read` pairs to single SQE; ideal for Wayland / D-Bus sockets |
| **io_uring IORING_OP_SEND_ZC** | Kernel async I/O | Zero-copy send for large D-Bus replies or PipeWire fd bundles |
| **io_uring IORING_OP_SPLICE** | Kernel async I/O | Zero-copy memfd/DMA-BUF handoff between PipeWire nodes |
| **io_uring SQPOLL** | Kernel async I/O | Zero-syscall IPC for compositor main loop; real-time PipeWire audio |

---

## 15. D-Bus vs Wayland Protocol: When to Use Which

Every compositor author eventually faces a design choice: should a new integration surface be a **D-Bus interface**, a **Wayland protocol extension**, a **Varlink service**, or a **compositor-specific Unix socket**? The choice has significant consequences for portability, security, toolability, and long-term maintenance. This section provides structured guidance.

### The Core Distinction

**D-Bus and Varlink** are general-purpose IPC mechanisms. They operate independently of the Wayland connection: a client does not need a Wayland display connection to call a D-Bus or Varlink method. This makes them suitable for:
- System-level concerns (power management, colour management, seat management)
- Cross-session communication (system bus)
- Services accessed by non-GUI tools (CLI tools, daemons, sandboxed apps via portals)
- Interfaces that predate Wayland or must be compositor-neutral (e.g. logind)

**Wayland protocol extensions** are multiplexed over the Wayland Unix socket and can only be spoken by clients with a valid `wl_display` connection. They are suitable for:
- Window management (positioning, focus, decoration, fullscreen)
- Input handling (seat, keyboard, pointer, touch, tablet)
- Output configuration (resolution, rotation, colour primaries per-output)
- Surface-level operations (subsurfaces, layer surfaces, fractional scaling)
- Pixel data exchange (DMA-BUF import, shared memory buffers)

### Decision Matrix

| Requirement | D-Bus | Varlink | Wayland Extension | Compositor Socket |
|---|---|---|---|---|
| Compositor-agnostic (works on GNOME + KDE + wlroots) | ✓ | ✓ | Only if ratified in wayland-protocols | ✗ |
| Non-Wayland clients (CLI tools, daemons) | ✓ | ✓ | ✗ | ✗ |
| Broadcast notifications (1:N signals) | ✓ | ✗ (no broadcast) | ✓ (events to subscribed clients) | ✗ |
| Carries file descriptors | ✓ (`h` type) | ✗ (JSON only) | ✓ (native fd passing) | ✓ (SCM_RIGHTS) |
| Access to compositor internal state | ✗ | ✗ | ✓ | ✓ |
| Sandboxed app access (Flatpak) | Via portals | Via portals | ✗ (Wayland socket not in sandbox) | ✗ |
| Rate: messages per second | ~5k–20k/s | ~10k–50k/s | ~100k–500k/s | ~100k–500k/s |
| Requires freedesktop standardisation | Usually yes | Rarely | For cross-compositor use | Never |
| Introspectable / toolable | ✓ (busctl, d-feet) | ✓ (varlinkctl) | ✗ (wayland-info only) | ✗ |

### The Canonical Partition

In practice, the Linux desktop ecosystem has converged on a clear partition:

**Use D-Bus** for any interface from the `org.freedesktop.*` namespace — logind seat management, colord, NetworkManager, PipeWire activation, xdg-desktop-portal. These are cross-compositor contracts; no Wayland extension is appropriate because they must work without a compositor or be shared across compositors.

**Use Varlink** for new systemd-integrated services (`io.systemd.*`). If your service integrates with systemd for activation, units, or resources and does not need broadcast signals, Varlink is the right choice. Avoid D-Bus for brand-new systemd-facing APIs.

**Use a ratified Wayland extension** (from `wayland-protocols` or `wlr-protocols`) for window management, surface operations, output configuration, and input when cross-compositor compatibility is needed. The `xdg-shell`, `xdg-output`, `wlr-layer-shell`, and `ext-session-lock` protocols are the reference examples. If a Wayland extension exists for the use case, use it in preference to a D-Bus interface.

**Use a compositor-specific Wayland extension or private socket** for features that are intentionally compositor-specific — KWin's `org.kde.kwin.dpms`, Hyprland's `hyprctl` socket, sway's IPC socket. These are not intended to be portable, and trying to standardise them prematurely adds maintenance burden without adding users.

### Anti-Patterns

**Do not use D-Bus for per-frame or per-input-event signalling.** The D-Bus round-trip latency (~40–100 µs typical, workload-dependent) and message-size limitations make it unsuitable for any latency-sensitive path. Frame timing, cursor position updates, and touch tracking belong in Wayland protocol event streams.

**Do not use a Wayland extension for compositor ↔ system-service communication.** A compositor talking to logind, NetworkManager, or PipeWire uses D-Bus, not a Wayland extension — these system services have no Wayland connection.

**Do not invent a new binary IPC protocol** when D-Bus, Varlink, or Wayland extensions cover the use case. Custom binary protocols lack introspection, have no standard tooling, and create unnecessary maintenance burden. The cost of implementing a D-Bus adaptor is low; the cost of debugging a custom protocol without `busctl monitor` is high.

### fd Passing: D-Bus vs Wayland

Both D-Bus (`h` type with `SCM_RIGHTS`) and Wayland protocol use Unix domain socket fd passing, but at different abstraction levels. D-Bus fd passing is used for:
- logind `TakeDevice` returning a DRM device fd
- PipeWire returning a memfd for shared memory
- `org.freedesktop.portal.ScreenCast` returning a PipeWire stream fd

Wayland fd passing is used for:
- `wl_shm_pool` (shared memory buffer allocation)
- `linux_dmabuf_v1` DMA-BUF import
- `wl_drm` (legacy EGL buffer sharing)

The rule of thumb: if the fd is produced by a system service and consumed by a Wayland client, D-Bus carries it. If the fd is produced by the client and consumed by the compositor (or vice versa) as part of the rendering pipeline, the Wayland protocol carries it.

---

## 16. Testing D-Bus and IPC Code

Untested D-Bus code is a common source of subtle bugs — method signature mismatches, missing error handling, signal subscriptions that fire too early or not at all, and policy denials that only appear in production. This section covers the testing tools and patterns for each layer of the D-Bus and Varlink stack.

### Private dbus-daemon Session for CI

The most reliable approach for integration tests is to start a private, isolated `dbus-daemon` instance scoped to the test process tree:

```bash
#!/bin/bash
# Start a private session bus for the test run
eval $(dbus-launch --sh-syntax)
# $DBUS_SESSION_BUS_PID and $DBUS_SESSION_BUS_ADDRESS are now set
trap "kill $DBUS_SESSION_BUS_PID" EXIT

# Run the service under test
my-dbus-service &
SERVICE_PID=$!

# Run the test client
./test-client
EXIT_CODE=$?

kill $SERVICE_PID
exit $EXIT_CODE
```

`dbus-launch --sh-syntax` spawns a `dbus-daemon` process, prints `export DBUS_SESSION_BUS_ADDRESS=...` and `export DBUS_SESSION_BUS_PID=...`, and the `eval $()` pattern sets them in the current shell. All child processes inherit these variables and connect to the private bus, completely isolated from the desktop session bus.

This is the basis for most D-Bus integration tests in open-source projects including Mutter, KWin, and xdg-desktop-portal.

### dbus-test-runner

`dbus-test-runner` (from the Ubuntu `dbus-test-runner` package, also available on Fedora/Arch) wraps the `dbus-launch` pattern and adds:
- **Service activation**: services listed in `--task` arguments are started and waited for before the test task runs.
- **Bus ready waiting**: waits until the service appears on the bus before starting the test.
- **Timeout and exit code propagation**: the test binary's exit code is propagated; a configurable timeout kills all tasks.

```bash
dbus-test-runner \
    --task my-dbus-service \
    --task-name "service" \
    --wait-for "org.freedesktop.MyService" \
    --task ./test-client \
    --task-name "test"
```

### Testing with sd-bus (C)

For sd-bus code, `systemd`'s `sd_bus_open_user_machine()` with a synthesised `dbus-daemon` address enables in-process bus creation:

```c
sd_bus *bus = NULL;
/* Use the private bus address from the environment */
const char *addr = getenv("DBUS_SESSION_BUS_ADDRESS");
assert(addr);
sd_bus_new(&bus);
sd_bus_set_address(bus, addr);
sd_bus_start(bus);

/* Now call methods or register objects on the isolated bus */
```

For unit tests without a daemon, `sd-bus` supports an **"in-process bus pair"** via `sd_bus_new_pair()` (available in libsystemd 255+): two `sd_bus` objects connected via a socket pair, usable without any daemon process. This is the fastest option for testing method dispatch and signal delivery in isolation.

### Testing with zbus (Rust)

zbus provides a first-class in-process test mechanism via `zbus::connection::Builder::p2p()`:

```rust
use zbus::{connection, interface, proxy};

#[proxy(interface = "org.example.Counter", default_path = "/counter")]
trait Counter {
    fn increment(&self) -> zbus::Result<u32>;
    #[zbus(signal)]
    fn counted(&self, value: u32) -> zbus::Result<()>;
}

struct CounterImpl { count: u32 }

#[interface(name = "org.example.Counter")]
impl CounterImpl {
    async fn increment(&mut self) -> u32 {
        self.count += 1;
        CounterImplSignals::counted(self.signal_context(), self.count)
            .await.unwrap();
        self.count
    }
}

#[tokio::test]
async fn test_counter() {
    // Create a peer-to-peer connection pair with no daemon
    let (server_conn, client_conn) = zbus::Connection::pair().await.unwrap();

    // Register the service object on the server side
    let _object_server = server_conn.object_server()
        .at("/counter", CounterImpl { count: 0 }).await.unwrap();

    // Use the generated proxy on the client side
    let proxy = CounterProxy::new(&client_conn).await.unwrap();
    assert_eq!(proxy.increment().await.unwrap(), 1);
    assert_eq!(proxy.increment().await.unwrap(), 2);
}
```

`zbus::Connection::pair()` creates two endpoints connected via a `socketpair()` with no bus daemon involved. Both method calls and signals work over the pair, making it suitable for unit-testing both client proxy and server adaptor code in a single test process.

### Testing with Qt (QDBus)

Qt's D-Bus testing uses a `QDBusConnection::connectToBus()` call with a private `dbus-daemon` address, combined with Qt Test:

```cpp
class DBusTest : public QObject {
    Q_OBJECT
private Q_SLOTS:
    void initTestCase() {
        // Start private bus (set DBUS_SESSION_BUS_ADDRESS in setUp script)
        m_bus = QDBusConnection::connectToBus(
            qgetenv("DBUS_SESSION_BUS_ADDRESS"), "test-bus");
        QVERIFY(m_bus.isConnected());
        QVERIFY(m_bus.registerService("org.example.TestService"));
    }
    void testMethodCall() {
        QDBusInterface iface("org.example.TestService",
                             "/", "org.example.TestInterface", m_bus);
        QVERIFY(iface.isValid());
        QDBusReply<QString> reply = iface.call("getVersion");
        QVERIFY(reply.isValid());
        QCOMPARE(reply.value(), QString("1.0"));
    }
private:
    QDBusConnection m_bus = QDBusConnection::sessionBus();
};
```

KDE projects use `dbus-test-runner` in their CI pipelines with `QDBusConnection` pointed at the private bus address.

### Testing Varlink (C and Rust)

For Varlink, since there is no central daemon, testing is straightforward: start the service as a child process (or thread) listening on a `socketpair()` or a temporary socket path, then send JSON requests and assert on responses.

```bash
# Shell-level Varlink test using varlinkctl
SOCKET=$(mktemp -d)/test.socket
my-varlink-service --socket $SOCKET &
sleep 0.1  # wait for socket to appear

# Call a method and assert on JSON output
RESULT=$(varlinkctl call $SOCKET io.example.Service.GetStatus '{}')
echo "$RESULT" | jq -e '.status == "running"' || exit 1

kill %1
```

For Rust, the same `socketpair()` approach used for D-Bus applies: bind a `UnixListener` in one task, connect from another, and exchange NUL-terminated JSON strings directly — no external process needed.

### Monitoring in CI: busctl capture + Wireshark

For debugging flaky CI test failures involving D-Bus, the `busctl capture` → pcap workflow is invaluable:

```bash
# Record all D-Bus traffic to a pcap file during the test run
busctl capture > /tmp/test-bus.pcap &
CAPTURE_PID=$!
./run-tests
kill $CAPTURE_PID
# Upload /tmp/test-bus.pcap as a CI artifact
```

In Wireshark, the D-Bus dissector (built-in) shows every method call, reply, signal, and error with decoded type values — dramatically faster than parsing `busctl monitor` text output for multi-service interaction bugs.

---

## 17. Comparison and Selection Guide

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

### Scope Boundary: Distributed Messaging Libraries

Readers coming from backend or networked-systems backgrounds will notice that **ZeroMQ** (`libzmq`), **nanomsg**, **nng** (nanomsg next generation), and **NATS** are absent from the comparison table. This is deliberate: these libraries solve a different problem at a different layer.

**What they are**: ZeroMQ, nanomsg, and nng are message-passing libraries that provide patterns (pub/sub, request/reply, push/pull, pipeline) over multiple transports — TCP, UDP, Unix sockets, in-process. They have no service registry, no type system, no introspection, and no central daemon. NATS is a cloud-native message broker with at-least-once delivery guarantees.

**Why they are not in this chapter**: none of them appear in the Linux desktop graphics stack. `logind`, `colord`, `PipeWire`, and `xdg-desktop-portal` are all D-Bus or Varlink services; no Wayland compositor, DRM driver, Mesa component, or freedesktop.org specification uses ZeroMQ. The gap is architectural: the Linux desktop IPC ecosystem was designed around **service discovery** (named well-known bus names), **introspection** (discoverable interface contracts), and **security policy** (per-method access control) — none of which ZeroMQ provides.

**Where they DO appear in Linux graphics-adjacent contexts**:

| Tool / Framework | Transport lib | Use case |
|---|---|---|
| **NVIDIA Omniverse** | ZeroMQ | Inter-process communication between USD stage, render, and physics processes |
| **OpenFrameworks** | ZeroMQ (ofxZmq addon) | Multi-process creative coding; GPU-rendered apps communicating with data sources |
| **Jupyter / JupyterLab** | ZeroMQ | Kernel ↔ frontend protocol; GPU compute notebooks (RAPIDS, CuPy) use this path |
| **ROS2 / Nav2** | DDS (not ZMQ) | Robot sensor fusion + GPU SLAM pipelines; ZMQ sometimes used for bridge nodes |
| **VJ / media server tools** | ZeroMQ or OSC-over-ZMQ | Resolume, TouchDesigner, VDMX inter-process media control |
| **GPU render farms** | ZeroMQ or NATS | Job dispatch to distributed render nodes (Blender, Arnold, V-Ray) |
| **Display wall systems** | ZeroMQ | Multi-node frame synchronisation (SAGE2, Scalable Adaptive Graphics Environment) |
| **MLflow / Ray** | ZeroMQ / gRPC | GPU-accelerated ML training coordination |

**The key distinction**: ZeroMQ operates *above* the graphics stack at the application or tooling layer, not *within* it. A VJ tool using ZeroMQ to receive cues from a lighting console still uses Wayland, D-Bus (via logind), and PipeWire internally — ZeroMQ is the application-to-application channel, not the compositor-to-system channel. When you are building something that sits *inside* the graphics stack (a compositor, a portal backend, a DRM driver userspace component), D-Bus, Varlink, Wayland protocol, and Binder are the tools. When you are building something that sits *on top of* the graphics stack and needs to communicate with other applications or remote peers, ZeroMQ/NATS/gRPC are legitimate options.

**When a graphics application should consider ZeroMQ or nng**:
- Streaming high-throughput data between GPU processes where D-Bus's per-message overhead is prohibitive and PipeWire is not the right abstraction (e.g., custom shader parameter streaming between a control UI and a render process)
- Network-distributed rendering where frames or render tiles need to be dispatched across machines (TCP transport, not Unix socket)
- Integration with existing tooling ecosystems (Jupyter, ROS2, Omniverse) that already use ZeroMQ

In these cases, ZeroMQ/nng over an in-process `inproc://` or Unix-socket `ipc://` transport gives ~1–2 µs latency for small messages — comparable to a raw Unix socket but with pattern semantics (pub/sub, dealer/router) built in.

---

## 18. libzmq and the Jupyter Kernel Protocol

Jupyter (formerly IPython Notebook) is the most widely-deployed interactive GPU computing environment in the world. Its architecture is one of the most instructive real-world uses of ZeroMQ outside the traditional messaging-middleware domain: a language-agnostic kernel protocol built on five ZMQ sockets, enabling any language kernel (Python, Julia, Rust, C++, Haskell) to serve any frontend (JupyterLab, VS Code, Jupyter Notebook, nteract) over a standardised wire protocol. For GPU developers, this is the IPC layer beneath every RAPIDS, CuPy, PyTorch, and JAX notebook.

### Architecture: Kernel as Subprocess

The Jupyter architecture decouples the execution kernel from the user interface completely. Each kernel runs as an independent OS process; the notebook server or JupyterLab starts it via `jupyter_client.KernelManager.start_kernel()`, which spawns the kernel subprocess and passes it a **connection file** — a JSON document describing which ZMQ ports to bind.

```
JupyterLab / VS Code / nteract (frontend)
        │
        │  Five ZMQ sockets (described in connection file)
        │
┌───────▼────────────────────────────────────┐
│           Kernel Process                   │
│  (python / julia / rustkernel / xeus-cpp)  │
│                                            │
│  GPU runtime (CUDA, ROCm, oneAPI, Vulkan)  │
└────────────────────────────────────────────┘
```

A single kernel can serve multiple frontends simultaneously (JupyterLab tab + VS Code + a monitoring dashboard) because ZMQ's `PUB/SUB` and `ROUTER/DEALER` patterns support multiple simultaneous connections.

### The Five ZMQ Sockets

| Socket | ZMQ type (kernel/client) | Purpose |
|---|---|---|
| `shell` | ROUTER / DEALER | Client → kernel requests: `execute_request`, `inspect_request`, `complete_request`, `history_request` |
| `iopub` | PUB / SUB | Kernel → all frontends broadcast: `stream` (stdout/stderr), `display_data`, `execute_result`, `error`, `status` |
| `stdin` | ROUTER / DEALER | Kernel → client input requests (`input_request`); client → kernel reply (`input_reply`) |
| `control` | ROUTER / DEALER | Interrupt (`interrupt_request`) and shutdown (`shutdown_request`) — separate socket prevents blocking behind `shell` |
| `hb` (heartbeat) | REP / REQ | Kernel echoes back any bytes sent; frontend uses this to detect kernel death |

The `iopub` socket is the ZMQ broadcast mechanism in action: one `PUB` socket in the kernel, multiple `SUB` sockets in different frontends, each receiving all output independently. This is the 1:N fan-out pattern that D-Bus signals provide for desktop IPC, applied here at the process level with ZMQ.

[Source: Jupyter Messaging Protocol specification](https://jupyter-client.readthedocs.io/en/stable/messaging.html)

### Connection File Format

When the kernel manager starts a kernel, it writes a connection file to `$JUPYTER_RUNTIME_DIR/kernel-<pid>.json`:

```json
{
  "shell_port": 52817,
  "iopub_port": 52818,
  "stdin_port": 52819,
  "control_port": 52820,
  "hb_port": 52821,
  "ip": "127.0.0.1",
  "key": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "transport": "tcp",
  "signature_scheme": "hmac-sha256",
  "kernel_name": "python3"
}
```

- `transport`: `"tcp"` for network-capable operation, `"ipc"` for Unix socket (higher performance, local-only). Most kernels default to `"tcp"` on localhost.
- `key`: a shared secret for HMAC-SHA256 message signing — every message has a signature computed over its header, parent header, metadata, and content. This prevents arbitrary code injection via the ZMQ sockets.
- `ip`: always `127.0.0.1` for local kernels; remote kernels (SSH-tunneled or containerised) use the tunnel endpoint.

### Message Format

Every Jupyter message is a multi-part ZMQ message with a fixed structure:

```
[identity frames (routing, ROUTER sockets only)]
["<IDS|MSG>"]           ← delimiter
[hmac_signature]        ← hex string, HMAC-SHA256 over the next 4 frames
[header JSON]           ← msg_id, session, date, username, version, msg_type
[parent_header JSON]    ← header of the message this is a reply to (or {})
[metadata JSON]         ← optional key-value pairs
[content JSON]          ← message-type-specific payload
[buffers ...]           ← optional binary buffers (for display_data images, etc.)
```

The `<IDS|MSG>` delimiter separates routing frames (used by `ROUTER` to track which client sent the message) from the signed payload frames.

### Python: Connecting to a Running Kernel

```python
import jupyter_client
import json, time

# Start a fresh kernel (or connect to an existing one via connection file)
km = jupyter_client.KernelManager(kernel_name='python3')
km.start_kernel()
kc = km.client()
kc.start_channels()
kc.wait_for_ready(timeout=30)

# Execute code in the kernel
msg_id = kc.execute("import cupy as cp; a = cp.ones((1024,1024)); print(a.sum())")

# Collect output: drain iopub until the execution completes
while True:
    try:
        msg = kc.get_iopub_msg(timeout=10)
    except Exception:
        break
    mt = msg['msg_type']
    if mt == 'stream':
        print(f"[{msg['content']['name']}] {msg['content']['text']}", end='')
    elif mt == 'display_data':
        data = msg['content']['data']
        if 'image/png' in data:
            # Inline image — base64 PNG for matplotlib or GPU-rendered output
            import base64
            png = base64.b64decode(data['image/png'])
            open('/tmp/kernel_output.png', 'wb').write(png)
    elif mt == 'error':
        print(f"ERROR: {''.join(msg['content']['traceback'])}")
    elif mt == 'status' and msg['content']['execution_state'] == 'idle':
        break  # kernel finished executing

kc.stop_channels()
km.shutdown_kernel()
```

### Raw pyzmq: Direct Protocol Access

For tooling that needs fine-grained control (custom kernels, kernel proxies, monitoring dashboards), using `pyzmq` directly against the wire protocol is more appropriate than `jupyter_client`:

```python
import zmq, json, hmac, hashlib, uuid, datetime

def sign(key, frames):
    """Compute HMAC-SHA256 signature over message frames."""
    h = hmac.new(key.encode(), digestmod=hashlib.sha256)
    for f in frames:
        h.update(f if isinstance(f, bytes) else f.encode())
    return h.hexdigest()

def send_message(socket, key, msg_type, content, parent=None):
    session = str(uuid.uuid4())
    header = json.dumps({
        "msg_id": str(uuid.uuid4()),
        "session": session,
        "date": datetime.datetime.utcnow().isoformat() + "Z",
        "username": "tool",
        "version": "5.3",
        "msg_type": msg_type,
    }).encode()
    parent_header = json.dumps(parent or {}).encode()
    metadata = b"{}"
    content_bytes = json.dumps(content).encode()

    frames = [header, parent_header, metadata, content_bytes]
    sig = sign(key, frames).encode()
    socket.send_multipart([b"<IDS|MSG>", sig] + frames)

# Connect directly to a running kernel's shell socket
ctx = zmq.Context()
shell = ctx.socket(zmq.DEALER)
shell.connect("tcp://127.0.0.1:52817")

iopub = ctx.socket(zmq.SUB)
iopub.connect("tcp://127.0.0.1:52818")
iopub.setsockopt(zmq.SUBSCRIBE, b"")   # subscribe to all topics

KEY = "a1b2c3d4-e5f6-7890-abcd-ef1234567890"   # from connection file

# Send an execute_request
send_message(shell, KEY, "execute_request", {
    "code": "import torch; print(torch.cuda.get_device_name(0))",
    "silent": False, "store_history": True,
    "user_expressions": {}, "allow_stdin": False,
})

# Read output from iopub
poller = zmq.Poller()
poller.register(iopub, zmq.POLLIN)
while True:
    socks = dict(poller.poll(5000))
    if iopub not in socks:
        break
    frames = iopub.recv_multipart()
    # frames: [topic, <IDS|MSG>, sig, header, parent, metadata, content, ...]
    content = json.loads(frames[6])
    header  = json.loads(frames[3])
    if header['msg_type'] == 'stream':
        print(content['text'], end='')
    elif header['msg_type'] == 'status' and content['execution_state'] == 'idle':
        break
```

### Implementing a Minimal Custom Kernel

Any language or runtime can become a Jupyter kernel by implementing the five-socket protocol. The minimum viable kernel:

```python
import zmq, json, hmac, hashlib, sys

def kernel_main(connection_file):
    with open(connection_file) as f:
        cfg = json.load(f)

    ctx = zmq.Context()
    key = cfg['key']
    t   = cfg['transport']
    ip  = cfg['ip']

    def addr(port): return f"{t}://{ip}:{port}"

    shell   = ctx.socket(zmq.ROUTER); shell.bind(addr(cfg['shell_port']))
    iopub   = ctx.socket(zmq.PUB);   iopub.bind(addr(cfg['iopub_port']))
    control = ctx.socket(zmq.ROUTER); control.bind(addr(cfg['control_port']))
    stdin   = ctx.socket(zmq.ROUTER); stdin.bind(addr(cfg['stdin_port']))
    hb      = ctx.socket(zmq.REP);   hb.bind(addr(cfg['hb_port']))

    poller = zmq.Poller()
    poller.register(shell, zmq.POLLIN)
    poller.register(control, zmq.POLLIN)
    poller.register(hb, zmq.POLLIN)

    execution_count = 0

    while True:
        socks = dict(poller.poll())
        if hb in socks:
            hb.send(hb.recv())   # heartbeat echo — keeps kernel "alive"

        if shell in socks:
            frames = shell.recv_multipart()
            # Parse: [identity..., b"<IDS|MSG>", sig, header, parent, meta, content]
            delim_idx = frames.index(b"<IDS|MSG>")
            idents    = frames[:delim_idx]
            header    = json.loads(frames[delim_idx + 2])
            content   = json.loads(frames[delim_idx + 5])

            if header['msg_type'] == 'execute_request':
                execution_count += 1
                code = content['code']

                # Broadcast "busy" status
                # (full implementation would sign all outgoing messages)
                iopub.send_multipart([b"status", json.dumps({
                    "msg_type": "status",
                    "content": {"execution_state": "busy"}
                }).encode()])

                # Execute the code (here: eval in Python itself)
                try:
                    result = str(eval(compile(code, "<kernel>", "exec")))
                    iopub.send_multipart([b"execute_result", json.dumps({
                        "msg_type": "execute_result",
                        "content": {
                            "execution_count": execution_count,
                            "data": {"text/plain": result},
                            "metadata": {}
                        }
                    }).encode()])
                except Exception as e:
                    iopub.send_multipart([b"error", json.dumps({
                        "msg_type": "error",
                        "content": {"ename": type(e).__name__,
                                    "evalue": str(e), "traceback": []}
                    }).encode()])

                iopub.send_multipart([b"status", json.dumps({
                    "msg_type": "status",
                    "content": {"execution_state": "idle"}
                }).encode()])

if __name__ == '__main__':
    kernel_main(sys.argv[1])
```

Production kernel implementations use the **xeus** framework (C++, [github.com/jupyter-xeus/xeus](https://github.com/jupyter-xeus/xeus)) or **ipykernel** (Python). Xeus is the basis for xeus-python, xeus-cling (C++ via Cling REPL), and xeus-lua; it handles all ZMQ socket management, message signing, and comm (widget) protocol, leaving only the language-specific evaluation loop to implement.

### GPU Kernels: RAPIDS, CuPy, and GPU-Accelerated Notebooks

For GPU computing, several kernel variants expose GPU runtimes through the standard Jupyter protocol:

| Kernel | GPU backend | Use case |
|---|---|---|
| **ipykernel** (standard Python) | CUDA via CuPy / PyTorch / JAX / TensorFlow | General GPU ML/data science |
| **RAPIDS cuDF kernel** | CUDA (RAPIDS) | GPU-accelerated DataFrame operations (like pandas on GPU) |
| **xeus-python** | CUDA via Python ecosystem | C++-backed Python kernel; lower overhead than ipykernel |
| **IJulia** | CUDA.jl / AMDGPU.jl | Julia GPU computing notebooks |
| **IHaskell** | Accelerate / ArrayFire | Haskell GPU notebooks |
| **xeus-cling** | CUDA C++ via Cling | Interactive CUDA C++ in Jupyter |

In all cases, the GPU work happens entirely inside the kernel process — the ZMQ protocol carries only the code strings in and the text/image results out. The `display_data` message type carries binary content (base64-encoded PNG images, HTML, JSON) via the `data` dict; GPU-rendered visualisations (matplotlib with CUDA backend, RAPIDS cuDF plots, Plotly) are serialised to PNG or JSON and sent over this channel.

**Widget protocol (ipywidgets / comm)**: For interactive GPU visualisations (slider controls that update a GPU simulation in real time), the **comm** sub-protocol runs over the `shell` and `iopub` sockets:
- Frontend sends `comm_open` → kernel creates a `Comm` object (e.g., a live plot widget)
- Frontend sends `comm_msg` with slider value → kernel updates GPU array, re-renders, sends back `display_data`
- This is ZMQ request/reply over the shell socket, with iopub broadcasting the updated display to all connected frontends simultaneously

The comm protocol is what `ipywidgets`, `plotly.FigureWidget`, `bqplot`, and `ipyvolume` (GPU 3D volume rendering in the browser) all use — ZMQ's `PUB/SUB` is the transport that makes interactive multi-frontend GPU visualisation possible without any additional IPC infrastructure.

### Kernel Registration: kernel.json

Each kernel registers itself with Jupyter by placing a `kernel.json` file in a kernelspec directory (`~/.local/share/jupyter/kernels/<name>/kernel.json`):

```json
{
  "argv": ["/usr/bin/python3", "-m", "ipykernel_launcher", "-f", "{connection_file}"],
  "display_name": "Python 3 (GPU)",
  "language": "python",
  "env": {
    "CUDA_VISIBLE_DEVICES": "0",
    "LD_LIBRARY_PATH": "/usr/local/cuda/lib64"
  },
  "metadata": {
    "debugger": true
  }
}
```

`{connection_file}` is replaced at launch time by the kernel manager with the path to the JSON file it wrote. The `env` dict is merged into the kernel subprocess environment — this is how GPU kernels select a specific CUDA device or set library paths without modifying the user's shell environment.

For containerised GPU kernels (JupyterHub with Docker or Kubernetes), the `argv` invokes a container runtime instead of a local binary, but the ZMQ sockets are the same — typically tunnelled via SSH port forwarding or exposed on the container network.

---

## 19. Roadmap

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

## 20. Integrations

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
