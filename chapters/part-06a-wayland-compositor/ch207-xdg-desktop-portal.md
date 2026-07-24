# Chapter 207: xdg-desktop-portal — Sandboxed Desktop Integration

> **Part**: Part VI-A — Wayland Compositor Layer
> **Audience**: Systems developers building Flatpak or Snap applications, graphics application developers who need screen capture, camera, or file access from inside a sandbox, and compositor developers implementing portal backends.
> **Status**: First draft — 2026-07-20

The **xdg-desktop-portal** project solves a structural problem: a sandboxed application cannot directly call Wayland compositor APIs, open `/dev/video0`, or display a native file-chooser dialog — it lacks the ambient privilege to do any of those things. At the same time, the user expects the same camera access consent flow whether they open a sandboxed WebRTC app or a native Chrome binary. The portal answers by inserting a **privileged daemon** between sandboxed apps and the desktop: apps call D-Bus methods on the portal daemon, the daemon authenticates the caller, enforces the per-application permission ledger, optionally shows a user consent dialog, and then calls the compositor or system service on the caller's behalf — delivering results (a file descriptor, a PipeWire node, a URI) back through D-Bus.

As of mid-2026, xdg-desktop-portal **1.22.1** is the current stable release (2026-06-17). The project uses the convention that odd minor versions (1.21.x) are development series; even minor versions (1.20.x, 1.22.x) are stable. Version 1.22.x adds USB device access, ScreenCast v6 with stable `pipewire-serial` for stream identification, Screenshot target selection, and numerous security fixes.

---

## Table of Contents

- [1. Architecture: The Three-Party Model](#1-architecture-the-three-party-model)
- [2. Caller Identification: From D-Bus Sender to AppID](#2-caller-identification-from-d-bus-sender-to-appid)
- [3. The Request/Session Pattern](#3-the-requestsession-pattern)
- [4. Backend Discovery and Configuration](#4-backend-discovery-and-configuration)
- [5. The ScreenCast Portal — Deep Dive](#5-the-screencast-portal--deep-dive)
- [6. The Camera Portal](#6-the-camera-portal)
- [7. The RemoteDesktop Portal and libei](#7-the-remotedesktop-portal-and-libei)
- [8. File Access: FileChooser, OpenURI, and the Document Store](#8-file-access-filechooser-openuri-and-the-document-store)
- [9. GlobalShortcuts and InputCapture](#9-globalshortcuts-and-inputcapture)
- [10. Desktop Integration Portals](#10-desktop-integration-portals)
- [11. The Permission Store](#11-the-permission-store)
- [12. wp_security_context_v1 — Compositor-Side Sandbox Tagging](#12-wp_security_context_v1--compositor-side-sandbox-tagging)
- [13. libportal — The C Convenience Library](#13-libportal--the-c-convenience-library)
- [14. PipeWire Integration: OpenPipeWireRemote](#14-pipewire-integration-openpipewireremote)
- [15. Backend Implementations](#15-backend-implementations)
- [16. Security Model and CVEs](#16-security-model-and-cves)
- [17. Roadmap](#17-roadmap)
- [Integrations](#integrations)
- [References](#references)

---

## 1. Architecture: The Three-Party Model

Every portal interaction involves three parties:

```
┌───────────────────┐   D-Bus session bus    ┌──────────────────────────────┐
│  Sandboxed App    │── method calls ───────►│  xdg-desktop-portal daemon   │
│  (Flatpak / Snap  │◄─ Response signals ───│  frontend                    │
│   / host app)     │                        │  bus: org.freedesktop.       │
└───────────────────┘                        │       portal.Desktop         │
                                             │  path: /org/freedesktop/     │
                                             │        portal/desktop         │
                                             └──────────────────────────────┘
                                                          │
                                             D-Bus (impl interfaces) + Wayland
                                                          │
                                             ┌──────────────────────────────┐
                                             │  Portal backend              │
                                             │  (xdg-desktop-portal-gnome   │
                                             │   xdg-desktop-portal-kde     │
                                             │   xdg-desktop-portal-wlr     │
                                             │   xdg-desktop-portal-        │
                                             │   hyprland, etc.)            │
                                             └──────────────────────────────┘
                                                          │
                                             native compositor APIs / dialogs
```

**The app** speaks only to `org.freedesktop.portal.Desktop` on the session D-Bus bus. It never learns which compositor is running or which backend backend implements the request. Its sandbox may restrict direct compositor access entirely.

**The portal daemon** (the *frontend*) is a privileged process. It authenticates the caller, checks and updates the permission store, and delegates to a backend. The frontend code lives in `src/` in the portal source tree; the D-Bus interfaces it exposes are defined as XML in `data/`.

**The portal backend** is a per-desktop-environment daemon that implements `org.freedesktop.impl.portal.*` interfaces. It uses compositor-native APIs — `org.gnome.Mutter.ScreenCast`, KWin's screen-sharing D-Bus, or the `wlr-screencopy` Wayland protocol — to actually perform the operation. The frontend calls the backend; neither the app nor the backend calls the other directly.

Two auxiliary services live on separate D-Bus bus names:

| Service | Bus name | Object path | Purpose |
|---|---|---|---|
| Document portal | `org.freedesktop.portal.Documents` | `/org/freedesktop/portal/documents` | FUSE filesystem exposing per-app file grants |
| Permission store | `org.freedesktop.impl.portal.PermissionStore` | `/org/freedesktop/impl/portal/PermissionStore` | Per-app, per-resource permission database |

---

## 2. Caller Identification: From D-Bus Sender to AppID

Before any portal method runs, the daemon must identify *who* is calling — specifically, whether the caller is a sandboxed Flatpak, a Snap, or an unsandboxed host application. The code for this lives in `shared/xdp-app-info.c` and its sandbox-specific siblings.

### Step 1: Get the Caller's PID

```c
// Call org.freedesktop.DBus.GetConnectionCredentials on the D-Bus sender
// Returns a{sv} with:
//   "ProcessID"  (u)  — PID
//   "ProcessFD"  (h)  — pidfd (Linux 5.3+, avoids PID reuse races)
reply = g_dbus_connection_call_with_unix_fd_list_sync(
    connection, DBUS_DBUS_NAME, DBUS_DBUS_PATH, DBUS_DBUS_IFACE,
    "GetConnectionCredentials",
    g_variant_new("(s)", sender), ...);

// Fallback (older dbus-daemon):
// org.freedesktop.DBus.GetConnectionUnixProcessID → (u) pid
```

The daemon prefers the `pidfd`-based approach to avoid TOCTOU races where the PID is valid at lookup time but refers to a different process by the time `/proc/<pid>/` is read.

### Step 2: Probe the Sandbox Type

The daemon tries each sandbox detector in order:

**Flatpak** (`xdp-app-info-flatpak.c`): Opens `/proc/<pid>/root/.flatpak-info`. This is a `GKeyFile` written by `bubblewrap` into the sandbox namespace. If present:
- `[Application] name` → AppID (e.g., `org.gnome.Eog`)
- `[Instance] instance-id` → unique instance identifier
- `[Context] shared` → shared subsystems (network, etc.)

Flatpak apps route D-Bus through `xdg-dbus-proxy`, so the D-Bus sender PID may be the proxy's PID, not the app's. The portal code traverses from the proxy PID to the real bubblewrap PID via the instance-id.

**Snap** (`xdp-app-info-snap.c`): Reads snap-specific process attributes from `/proc/<pid>/attr/`.

**linyaps** (`xdp-app-info-linyaps.c`): A Chinese container format added in 1.20.3.

**Host app** (`xdp-app-info-host.c`): Falls through to this if no sandbox is detected. AppID is the empty string. Most portals apply more permissive checks for host apps, or use the `org.freedesktop.host.portal.Registry` interface for self-registration.

### Step 3: Host App Registration

Unsandboxed applications that want to participate in portal permission tracking can self-register:

```c
// Must be called BEFORE any other portal method on the connection:
g_dbus_connection_call(bus,
    "org.freedesktop.portal.Desktop",
    "/org/freedesktop/portal/desktop",
    "org.freedesktop.host.portal.Registry",
    "Register",
    g_variant_new("(sa{sv})", "com.example.myapp", &empty_opts),
    NULL, G_DBUS_CALL_FLAGS_NONE, -1, NULL, NULL, NULL);
```

`Register` may only be called once per connection. If called after any other portal method, it returns an error. Unregistered host apps are identified by empty AppID and bypass most per-app permission checks.

The resolved `XdpAppInfo` is cached keyed by D-Bus unique name. A `NameOwnerChanged` signal triggers cache cleanup when the client disconnects.

---

## 3. The Request/Session Pattern

### Request Objects

Portal methods that require user interaction follow an async pattern: the method call returns a handle path immediately (synchronous), then emits a `Response` signal on that object when the interaction concludes. This avoids the 25-second D-Bus method call timeout.

**Handle path format**:
```
/org/freedesktop/portal/desktop/request/SENDER/TOKEN
```
Where `SENDER` = unique D-Bus name with leading `:` removed and all `.` replaced by `_`. For `:1.234` → `_1_234`. `TOKEN` = caller-provided string from `handle_token` option.

**Critical**: subscribe to the `Response` signal *before* calling the method, using the pre-computed handle path:

```c
// 1. Generate a unique token
char *token = g_strdup_printf("myapp%u", g_random_int());

// 2. Compute the expected handle path (do this before the call)
const char *unique_name = g_dbus_connection_get_unique_name(bus);
char *sender = g_strdup(unique_name + 1);       // strip leading ':'
for (char *p = sender; *p; p++)
    if (*p == '.') *p = '_';                    // replace '.' with '_'
char *handle = g_strdup_printf(
    "/org/freedesktop/portal/desktop/request/%s/%s", sender, token);

// 3. Subscribe BEFORE the method call
g_dbus_connection_signal_subscribe(bus,
    "org.freedesktop.portal.Desktop",
    "org.freedesktop.portal.Request", "Response",
    handle,    /* object path filter — eliminates false positives */
    NULL, G_DBUS_SIGNAL_FLAGS_NO_MATCH_RULE,
    on_response, user_data, NULL);

// 4. Call the portal method
GVariantBuilder opts;
g_variant_builder_init(&opts, G_VARIANT_TYPE_VARDICT);
g_variant_builder_add(&opts, "{sv}", "handle_token",
                       g_variant_new_string(token));
g_dbus_connection_call(bus,
    "org.freedesktop.portal.Desktop",
    "/org/freedesktop/portal/desktop",
    "org.freedesktop.portal.ScreenCast",
    "CreateSession",
    g_variant_new("(a{sv})", &opts), ...);

// 5. If the returned handle differs from expected, update the subscription
```

The `Response` signal carries:
```c
<signal name="Response">
  <arg type="u" name="response"/>    // 0=success, 1=user cancelled, 2=other error
  <arg type="a{sv}" name="results"/> // interface-specific result dict
</signal>
```

To cancel a pending request, call `org.freedesktop.portal.Request.Close()` on the handle object. No `Response` signal is emitted after `Close`.

### Session Objects

Long-lived multi-step flows — screen capture, remote desktop, global shortcuts, location, input capture — use *sessions*. A session persists until explicitly closed or until the client D-Bus connection is lost.

**Session handle path format**:
```
/org/freedesktop/portal/desktop/session/SENDER/SESSION_TOKEN
```
The `SESSION_TOKEN` comes from the `session_handle_token` option in the `CreateSession` call.

```xml
<interface name="org.freedesktop.portal.Session">
  <method name="Close"/>
  <signal name="Closed">
    <arg type="a{sv}" name="details"/>
  </signal>
  <property name="version" type="u" access="read"/>
</interface>
```

A client vanishing from D-Bus (process exit, crash) causes the daemon to auto-close all sessions belonging to that connection.

---

## 4. Backend Discovery and Configuration

### Portal Files

Each backend installs a `.portal` file at:
```
{datadir}/xdg-desktop-portal/portals/
```
typically `/usr/share/xdg-desktop-portal/portals/`. The file declares which D-Bus name to activate and which `org.freedesktop.impl.portal.*` interfaces it implements:

```ini
[portal]
DBusName=org.freedesktop.impl.portal.desktop.gnome
Interfaces=org.freedesktop.impl.portal.ScreenCast;org.freedesktop.impl.portal.RemoteDesktop;org.freedesktop.impl.portal.FileChooser;...
UseIn=gnome
```

The `UseIn` key is deprecated but still present for compatibility. Backend selection is now governed by `portals.conf`.

### portals.conf

The portal daemon searches for a configuration file in these locations, highest precedence first:
1. `$XDG_CONFIG_HOME/` (`~/.config/`)
2. `$XDG_CONFIG_DIRS/` (`/etc/xdg/`)
3. `$XDG_DATA_HOME/` (`~/.local/share/`)
4. `$XDG_DATA_DIRS/` (`/usr/local/share/`, `/usr/share/`)

Within each location, it first tries `DESKTOP-portals.conf` for each colon-separated entry in `$XDG_CURRENT_DESKTOP` (lowercased), then `portals.conf`.

File format:
```ini
[preferred]
# default= applies to all unspecified interfaces
default=gnome;gtk;

# Override specific interfaces:
org.freedesktop.impl.portal.ScreenCast=gnome
org.freedesktop.impl.portal.Secret=gnome-keyring;
org.freedesktop.impl.portal.Access=none   # disable this interface
```

Values are semicolon-separated backend names (the `DBusName` short suffix), `none` to disable, or `*` for the first alphabetical match. Multiple backends in a list are tried in order; the first that can be D-Bus activated wins.

The daemon D-Bus-activates backend services on demand; they need not be running at portal startup.

---

## 5. The ScreenCast Portal — Deep Dive

The ScreenCast portal (`org.freedesktop.portal.ScreenCast`, version 6) is the most complex portal in the project. It delivers compositor screen-capture frames through a PipeWire graph node, with full user consent, source selection, cursor-mode control, and session persistence.

### Interface Properties

```c
// Read-only properties (query before use):
// AvailableSourceTypes (u) — bitmask:
//   1 = MONITOR   share existing monitors
//   2 = WINDOW    share application windows
//   4 = VIRTUAL   create a new virtual monitor  [added v3]

// AvailableCursorModes (u) — bitmask:  [added v2]
//   1 = Hidden    cursor not included in stream
//   2 = Embedded  cursor composited into stream pixel data
//   4 = Metadata  cursor delivered as PipeWire stream metadata

// version (u)
```

### Full Call Sequence

A screen-capture session requires four sequential D-Bus round-trips:

**Step 1 — CreateSession**: establish the session object.

```c
// handle_token and session_handle_token must be unique per call
GVariantBuilder opts;
g_variant_builder_init(&opts, G_VARIANT_TYPE_VARDICT);
g_variant_builder_add(&opts, "{sv}", "handle_token",
                       g_variant_new_string("myapp_req_1"));
g_variant_builder_add(&opts, "{sv}", "session_handle_token",
                       g_variant_new_string("myapp_sess_1"));

// Response result: session_handle (s) — note: typed (s) for compat, not (o)
```

**Step 2 — SelectSources**: user selects what to share. The compositor backend shows its source-picker UI.

```c
g_variant_builder_add(&opts, "{sv}", "types",
                       g_variant_new_uint32(1));       // MONITOR
g_variant_builder_add(&opts, "{sv}", "multiple",
                       g_variant_new_boolean(FALSE));  // one source only
g_variant_builder_add(&opts, "{sv}", "cursor_mode",
                       g_variant_new_uint32(2));       // Embedded
// Restore a previous selection (v4+):
g_variant_builder_add(&opts, "{sv}", "persist_mode",
                       g_variant_new_uint32(1));       // 0=no,1=app lifetime,2=permanent
g_variant_builder_add(&opts, "{sv}", "restore_token",
                       g_variant_new_string(saved_token));  // NULL if no prior session

// Response: no result keys (the actual selection happens in Start)
```

`persist_mode` controls whether the user's source selection is saved:
- `0`: no persistence — user picks every time
- `1`: persists for the application lifetime (token valid until app exits)
- `2`: persists until explicitly revoked by the user

**Step 3 — Start**: show consent UI (if needed), open the sharing session, get stream node IDs.

```c
// parent_window: ""                   no parent
//                "x11:<hex-XID>"      X11 window
//                "wayland:<handle>"   xdg-foreign exported handle

// Response result (ScreenCast v6):
// streams          (a(ua{sv}))    array of (node_id, props)
// restore_token    (s)            single-use; save for next session [v4+]
//
// Stream props dict (v6):
//   id                (s)    stable opaque id across restored sessions [v4]
//   position          ((ii)) x,y in compositor logical coordinates (monitors)
//   size              ((ii)) width,height in compositor logical coordinates
//   source_type       (u)    1=MONITOR, 2=WINDOW, 4=VIRTUAL  [v3]
//   mapping_id        (s)    matches libei absolute device regions  [v5]
//   pipewire-serial   (t)    monotonic uint64 object.serial  [v6]
//
// IMPORTANT: node_id (u) in the (ua{sv}) tuple is DEPRECATED in v6.
// Use pipewire-serial with PW_KEY_TARGET_OBJECT for stream connection.

g_autoptr(GVariant) streams =
    g_variant_lookup_value(results, "streams", G_VARIANT_TYPE("a(ua{sv})"));
GVariantIter iter;
guint32 node_id;
GVariant *props;
g_variant_iter_init(&iter, streams);
while (g_variant_iter_next(&iter, "(u@a{sv})", &node_id, &props)) {
    guint64 serial;
    gboolean has_serial = g_variant_lookup(props, "pipewire-serial", "t", &serial);
    // prefer serial over node_id
    g_variant_unref(props);
}
```

**Step 4 — OpenPipeWireRemote**: obtain a PipeWire fd restricted to this session's nodes.

```c
// Must use the unix-fd-list variant — the fd is returned out-of-band
g_dbus_connection_call_with_unix_fd_list(bus,
    "org.freedesktop.portal.Desktop",
    "/org/freedesktop/portal/desktop",
    "org.freedesktop.portal.ScreenCast",
    "OpenPipeWireRemote",
    g_variant_new("(oa{sv})", session_handle, &empty_opts),
    G_VARIANT_TYPE("(h)"),
    G_DBUS_CALL_FLAGS_NONE, -1, NULL, NULL,
    on_open_pipewire_remote_done, NULL);

static void
on_open_pipewire_remote_done(GObject *source, GAsyncResult *res, gpointer data)
{
    g_autoptr(GUnixFDList) fd_list = NULL;
    g_autoptr(GVariant) ret =
        g_dbus_connection_call_with_unix_fd_list_finish(
            G_DBUS_CONNECTION(source), &fd_list, res, NULL);

    gint idx;
    g_variant_get(ret, "(h)", &idx);
    int pw_fd = g_unix_fd_list_get(fd_list, idx, NULL);
    // pw_fd ownership transfers to pw_context_connect_fd — do NOT close separately
}
```

Full PipeWire connection from the fd is covered in §14.

---

## 6. The Camera Portal

The Camera portal (`org.freedesktop.portal.Camera`, version 1) follows a simpler two-step flow: request permission, then get a PipeWire fd that shows only camera nodes.

```c
// Step 1: Request camera permission (shows system dialog if not yet granted)
// Permission stored in table "camera", id "camera"
// options: handle_token (s)
// Response result: no keys (success = permission granted)

// Step 2: Open PipeWire remote (only camera nodes visible)
// Must be called AFTER AccessCamera succeeded on this connection
// options: no supported keys
// Returns: fd h  (use with pw_context_connect_fd)
```

A property `IsCameraPresent (b)` allows checking whether any camera device is available before requesting access. The permission state is one of `"yes"`, `"no"`, or `"ask"` in the permission store. For `"ask"`, the daemon calls the backend's `org.freedesktop.impl.portal.Access` interface to display a modal consent dialog.

---

## 7. The RemoteDesktop Portal and libei

The RemoteDesktop portal (`org.freedesktop.portal.RemoteDesktop`, version 2) enables injecting keyboard, pointer, and touch events into a remote desktop session — used by VNC/RDP backends, KVM-over-IP tools, and accessibility infrastructure. In version 2 it gained session persistence, clipboard integration, and a switch to the **libei** (Emulated Input) event injection system.

### Session Setup

```c
// CreateSession → SelectDevices → Start (same Request pattern as ScreenCast)

// SelectDevices options:
//   types (u)  bitmask: 1=KEYBOARD, 2=POINTER, 4=TOUCHSCREEN
//   restore_token / persist_mode  [v2]

// Start Response:
//   devices (u)           bitmask of granted device types
//   clipboard_enabled (b) clipboard capture enabled  [v2]
//   streams (a(ua{sv}))   same format as ScreenCast.Start (shares the stream)
//   restore_token (s)     [v2]
```

RemoteDesktop sessions can include a ScreenCast stream to provide the video feed alongside the input injection, creating a full remote-desktop session.

### Input Injection: Notify* Methods (v1, Deprecated)

Version 1 provided individual methods for each event type:

```c
// Relative pointer motion (logical stream coordinates):
NotifyPointerMotion(session o, options a{sv}, dx d, dy d)

// Absolute pointer position:
NotifyPointerMotionAbsolute(session o, options a{sv}, stream u, x d, y d)

// Mouse button (Linux evdev button code: BTN_LEFT=0x110, BTN_RIGHT=0x111, …):
NotifyPointerButton(session o, options a{sv}, button i, state u)
//   state: 0=Released, 1=Pressed

// Scroll:
NotifyPointerAxis(session o, options a{sv}, dx d, dy d)
//   options: finish (b)  — last event in series
NotifyPointerAxisDiscrete(session o, options a{sv}, axis u, steps i)
//   axis: 0=Vertical, 1=Horizontal

// Keyboard (evdev keycode):
NotifyKeyboardKeycode(session o, options a{sv}, keycode i, state u)
// Keyboard (X11 keysym):
NotifyKeyboardKeysym(session o, options a{sv}, keysym i, state u)

// Touch:
NotifyTouchDown(session o, options a{sv}, stream u, slot u, x d, y d)
NotifyTouchMotion(session o, options a{sv}, stream u, slot u, x d, y d)
NotifyTouchUp(session o, options a{sv}, slot u)
```

### Input Injection: ConnectToEIS (v2)

Version 2 adds `ConnectToEIS`, which returns a file descriptor for the **libei** (Emulated Input Subsystem) sender context. This replaces the `Notify*` methods for production implementations:

```c
// Obtain the libei fd:
// Returns: fd h  (ei_setup_backend_fd socket)
ConnectToEIS(session o, options a{sv}) → fd h

// In application code (libei API):
struct ei *ei = ei_new_sender(NULL);
ei_setup_backend_fd(ei, pw_fd);

struct ei_seat *seat = ei_get_seat(ei, "default");
struct ei_device *ptr = ei_seat_new_device(seat);
ei_device_configure_name(ptr, "Remote Pointer");
// ... add capabilities, commit, send events
```

Once `ConnectToEIS` is called, the `Notify*` D-Bus methods return an error; the libei path is exclusively used. The `mapping_id` stream property from ScreenCast matches libei's absolute device regions, enabling coordinate-correct absolute positioning.

---

## 8. File Access: FileChooser, OpenURI, and the Document Store

### FileChooser

`org.freedesktop.portal.FileChooser` (version 4) presents the compositor's native file-chooser dialog and returns `file://` URIs. The key methods:

```c
// Open files:
OpenFile(parent_window s, title s, options a{sv}) → handle o
// options:
//   handle_token      (s)
//   accept_label      (s)   button label (underline-mnemonic with _)
//   modal             (b)   default true
//   multiple          (b)   allow multiple selection
//   directory         (b)   select directories  [v3]
//   filters           (a(sa(us)))  filter list
//   current_filter    ((sa(us)))   pre-selected filter
//   choices           (a(ssa(ss)s)) extra comboboxes/checkboxes
//   current_folder    (ay)  byte string (null-terminated) of path
//
// Response results:
//   uris             (as)       file:// URIs
//   choices          (a(ss))    selected choice values
//   current_filter   ((sa(us))) applied filter

// Save file:
SaveFile(parent_window s, title s, options a{sv}) → handle o
// adds: current_name (s), current_folder (ay), current_file (ay)

// Save multiple files to a directory:
SaveFiles(parent_window s, title s, options a{sv}) → handle o
// adds: files (aay — array of filename byte strings)
```

**Filter format** — `a(sa(us))`: an array of (label, [(type, pattern),...]) tuples. Type `0` = glob pattern; type `1` = MIME type (wildcard subtype supported):

```c
GVariant *filters = g_variant_new_parsed(
    "[('Images', [(uint32 0, '*.png'), (uint32 0, '*.jpg'), (uint32 1, 'image/*')]),"
    " ('All files', [(uint32 0, '*')])]");
```

**Choices format** — `a(ssa(ss)s)`: (id, label, [(value_id, value_label),...], default). An empty choices array indicates a boolean checkbox:

```c
GVariant *choices = g_variant_new_parsed(
    "[('overwrite', 'Overwrite?', [], 'false'),"
    " ('encoding', 'Encoding', [('utf8', 'UTF-8'), ('latin1', 'Latin-1')], 'utf8')]");
// Response choices: [('overwrite', 'true'), ('encoding', 'utf8')]
```

### OpenURI

`org.freedesktop.portal.OpenURI` (version 5) opens URIs and files through the user's default handler:

```c
// Open a URI (http://, mailto:, etc.):
OpenURI(parent_window s, uri s, options a{sv}) → handle o
//   options: writable (b), ask (b) [show chooser always, v3],
//            activation_token (s) [XDG activation, v4]

// Open a file by fd:
OpenFile(parent_window s, fd h, options a{sv}) → handle o  [v2]

// Open a directory in the file manager:
OpenDirectory(parent_window s, fd h, options a{sv}) → handle o  [v3]

// Check if a URI scheme has a registered handler:
SchemeSupported(scheme s, options a{sv}) → supported b  [v5]
```

### The Document Store

`org.freedesktop.portal.Documents` (version 5) provides a **FUSE filesystem** at `/run/user/$UID/doc/` that maps opaque document IDs to real host files. A sandboxed app reads `/run/user/$UID/doc/<id>/filename.ext` and sees exactly the file the portal granted — no broader filesystem access.

```c
// Export a file (get document id):
Add(o_path_fd h, reuse_existing b, persistent b) → id s

// Export with explicit permissions and flags:
AddFull(o_path_fds ah, flags u, app_id s, permissions as) → ids as, extra a{sv}
//   flags:  1=reuse_existing, 2=persistent, 4=as-needed, 8=export_directory [v4]
//   permissions: ["read","write","grant-permissions","delete"]

// Get the host path for document IDs (useful for host apps):
GetHostPaths(ids as) → paths a{say}  [v5]

// Manage grants:
GrantPermissions(doc_id s, app_id s, permissions as)
RevokePermissions(doc_id s, app_id s, permissions as)
Delete(doc_id s)

// Query:
GetMountPoint() → path ay
Lookup(filename s) → id s
Info(doc_id s) → path s, apps a{sas}
List(app_id s) → docs a{sas}
```

The FUSE filesystem translates accesses at `/run/user/$UID/doc/<id>/` into `openat(real_host_fd, "filename")` calls, using the fd the document portal holds. Sandboxed apps see a restricted view; the host can call `GetHostPaths` to retrieve the real absolute path.

---

## 9. GlobalShortcuts and InputCapture

### GlobalShortcuts

`org.freedesktop.portal.GlobalShortcuts` (version 2) allows sandboxed apps to register system-wide keyboard shortcuts that remain active even when the app is not focused:

```c
// Create session, then bind shortcuts:
BindShortcuts(
    session_handle o,
    shortcuts a(sa{sv}),   // [(id, {description, preferred_trigger}), ...]
    parent_window s,
    options a{sv}) → handle o

// shortcuts format:
// [("screenshot", {
//     "description": "Take a screenshot",
//     "preferred_trigger": "Ctrl+Print"  // XDG shortcut format
//   }), ...]

// Response: same format, with trigger_description added by compositor

// List registered shortcuts:
ListShortcuts(session_handle o, options a{sv}) → handle o

// Open compositor shortcut configuration UI [v2]:
ConfigureShortcuts(session_handle o, parent_window s, options a{sv})
// No Response signal

// Signals (emitted on the Session object):
// Activated(session_handle o, shortcut_id s, timestamp t, options a{sv})
//   options may include: activation_token (s)
// Deactivated(session_handle o, shortcut_id s, timestamp t, options a{sv})
// ShortcutsChanged(session_handle o, shortcuts a(sa{sv}))
```

On GNOME, this is implemented via `org.gnome.GlobalShortcutsProvider`. On Hyprland, via `xdg-desktop-portal-hyprland`'s `GlobalShortcuts` implementation.

### InputCapture

`org.freedesktop.portal.InputCapture` (version 2) captures all pointer and keyboard input for a session, used by remote-desktop receivers, KVM-switch software, and accessibility tools. Version 2 introduces `CreateSession2` (replacing the deprecated `CreateSession`):

```c
// Create session:
CreateSession2(options a{sv}) → results a{sv}
// options: session_handle_token (s), capabilities (u)
// capabilities: 1=KEYBOARD, 2=POINTER, 4=TOUCHSCREEN

// Get capture zones (display geometry):
GetZones(session_handle o, options a{sv}) → handle o
// Response: zones a(uuii) — (width, height, x_offset, y_offset); zone_set u

// Set pointer barriers (lines that trigger capture):
SetPointerBarriers(session_handle o, options a{sv},
                    barriers aa{sv}, zone_set u) → handle o
// barriers: [{barrier_id (u), position (iiii) as x1,y1,x2,y2}, ...]
// Response: failed_barriers au  — barriers that could not be applied

// Enable/disable:
Enable(session_handle o, options a{sv})
Disable(session_handle o, options a{sv})
Release(session_handle o, options a{sv})  // release captured pointer to compositor

// Get libei fd for event injection:
ConnectToEIS(session_handle o, options a{sv}) → fd h

// Signals:
// Activated   — capture begins (pointer crossed a barrier)
// Deactivated — capture ends
// Disabled    — portal stopped capture
// ZonesChanged — display geometry changed
```

InputCapture and RemoteDesktop both use `ConnectToEIS` as the injection path in v2, so both ultimately feed events through the libei kernel subsystem.

---

## 10. Desktop Integration Portals

Beyond the media and input portals, xdg-desktop-portal exposes a large family of desktop integration interfaces. The following summarises the most frequently used in graphics and multimedia applications.

### Settings

`org.freedesktop.portal.Settings` (version 2) exposes desktop theme settings:

```c
// Read multiple namespaces at once:
ReadAll(namespaces as) → value a{sa{sv}}
// namespaces supports trailing globs: "org.freedesktop.*"

// Read a single key (v2+, replaces deprecated Read):
ReadOne(namespace s, key s) → value v

// Signal:
SettingChanged(namespace s, key s, value v)
```

**Standardised keys** under `org.freedesktop.appearance`:

| Key | Type | Values |
|---|---|---|
| `color-scheme` | `u` | 0=no preference, 1=dark, 2=light |
| `accent-color` | `(ddd)` | sRGB triple [0.0, 1.0] |
| `contrast` | `u` | 0=normal, 1=higher |
| `reduced-motion` | `u` | 0=no preference, 1=reduced [added 1.21] |

These are the authoritative values for applications that want to respect the user's system theme, contrast, and animation preferences without depending on toolkit-specific APIs.

### Notification

`org.freedesktop.portal.Notification` (version 2):

```c
AddNotification(id s, notification a{sv})
// notification vardict:
//   title            (s)
//   body             (s)
//   markup-body      (s)  XML: <b>, <i>, <a href="">   [v2]
//   icon             (v)  ("themed", as) | ("file-descriptor", h)
//   sound            (v)  ("default","") | ("silent","") | ("file-descriptor",h) [v2]
//   priority         (s)  "low"|"normal"|"high"|"urgent"
//   default-action   (s)  exported action name
//   buttons          (aa{sv})  [{label,action,target,purpose}]
//   display-hint     (as) ["transient","tray","persistent","hide-on-lockscreen"] [v2]
//   category         (s)  "im.received"|"call.incoming"|"alarm.ringing"  [v2]

RemoveNotification(id s)

// Signal:
ActionInvoked(id s, action s, parameter av)
// parameter: [target?, platform_data?, reply_text?] in order
```

### Background

`org.freedesktop.portal.Background` (version 2) requests permission to continue running after the user closes the app window:

```c
RequestBackground(parent_window s, options a{sv}) → handle o
// options: reason (s), autostart (b), commandline (as), dbus-activatable (b)
// Response: background (b), autostart (b)

SetStatus(options a{sv})  [v2]
// options: message (s) ≤96 chars — shown in background-app monitor UI
```

### Inhibit

`org.freedesktop.portal.Inhibit` (version 3) prevents logout, user-switch, suspend, or idle:

```c
Inhibit(window s, flags u, options a{sv}) → handle o
// flags: 1=Logout, 2=UserSwitch, 4=Suspend, 8=Idle
// Release: call Request.Close() on the returned handle

// Monitor session state [v2+]:
CreateMonitor(window s, options a{sv}) → handle o  // creates session
// Signal on session: StateChanged(session o, state a{sv})
//   state: screensaver-active (b), session-state (u: 1=Running,2=QueryEnd,3=Ending)

// When session-state=2 (QueryEnd), must call within 1 second:
QueryEndResponse(session_handle o)  [v3]
```

### Secret

`org.freedesktop.portal.Secret` (version 1) provides a stable per-application master secret:

```c
RetrieveSecret(fd h, options a{sv}) → handle o
// fd: writable fd — portal writes secret bytes into it
// options: handle_token (s), token (s) for reproducibility
// Returns a unique stable secret keyed by AppID in the user's keyring
// Apply KDF (HKDF, Argon2) before using as encryption key
```

### System Monitoring Portals

Three lightweight monitoring interfaces require no user interaction:

```c
// Network connectivity:
// org.freedesktop.portal.NetworkMonitor (v3)
GetStatus() → a{sv}   // available(b), metered(b), connectivity(u: 1-4)
CanReach(hostname s, port u) → b
Signal: changed()

// Memory pressure:
// org.freedesktop.portal.MemoryMonitor (v1)
Signal: LowMemoryWarning(level y)  // 0=lowest, 255=highest

// Power profile:
// org.freedesktop.portal.PowerProfileMonitor (v1)
Property: power-saver-enabled (b)
```

### Trash and Realtime

```c
// Safe file deletion (org.freedesktop.portal.Trash v1):
TrashFile(fd h) → result u  // fd opened r/w; result 0=failed, 1=success

// Realtime scheduling (org.freedesktop.portal.Realtime v1):
// PID-namespace-aware proxy for org.freedesktop.RealtimeKit1
MakeThreadRealtimeWithPID(process t, thread t, priority u)
MakeThreadHighPriorityWithPID(process t, thread t, priority i)
```

---

## 11. The Permission Store

### D-Bus Interface

`org.freedesktop.impl.portal.PermissionStore` (version 2) is the daemon that persists per-application consent decisions. Applications never call it directly; the portal frontend queries it during method dispatch.

```c
// Query permission:
GetPermission(table s, id s, app s) → permissions as
// returns e.g. ["yes"], ["no"], ["ask"]

// Set permission:
SetPermission(table s, create b, id s, app s, permissions as)
Set(table s, create b, id s, app_permissions a{sas}, data v)

// Enumerate:
List(table s) → ids as
Lookup(table s, id s) → permissions a{sas}, data v

// Remove:
DeletePermission(table s, id s, app s)  [v2]
Delete(table s, id s)

// Signal:
Changed(table s, id s, deleted b, data v, permissions a{sas})
```

### On-Disk Format

The permission store uses **GVariant-serialized binary files** (not SQLite, not plaintext) stored at:
```
~/.local/share/flatpak/db/<table-name>
```

Known tables and their semantics:

| Table | Object ID | Permission values |
|---|---|---|
| `camera` | `"camera"` | `["yes"]`, `["no"]`, `["ask"]` |
| `devices` | `"speakers"`, `"microphone"`, `"camera"` | `["yes"]`, `["no"]`, `["ask"]` |
| `documents` | document UUID | `["read","write","grant-permissions","delete"]` |
| `notifications` | `"notification"` | `["yes"]`, `["no"]` |
| `location` | `"location"` | accuracy level + timestamp |
| `background` | `"background"` | `["yes"]`, `["no"]`, `["ask"]` |
| `desktop-used-apps` | MIME type | handler-id + use count |

The GVariant schema for a table entry is `(a{sas}v)` — a map from AppID to string array of permissions, plus optional extra GVariant data whose format is table-specific. For `documents`, the extra data is `(ayttu)` containing the path byte string, `st_dev`, `st_ino`, and flags.

### Inspecting Permissions

```bash
# View raw database (requires flatpak-info or direct read):
ls ~/.local/share/flatpak/db/

# Read with the D-Bus interface (gdbus example):
gdbus call --session \
    --dest org.freedesktop.impl.portal.PermissionStore \
    --object-path /org/freedesktop/impl/portal/PermissionStore \
    --method org.freedesktop.impl.portal.PermissionStore.GetPermission \
    camera camera org.gnome.Cheese
# → (['yes'],)

# List all apps with camera permission:
gdbus call --session \
    --dest org.freedesktop.impl.portal.PermissionStore \
    --object-path /org/freedesktop/impl/portal/PermissionStore \
    --method org.freedesktop.impl.portal.PermissionStore.Lookup \
    camera camera
# → ({'org.gnome.Cheese': ['yes'], 'org.mozilla.firefox': ['ask']}, <@mv nothing>)
```

---

## 12. wp_security_context_v1 — Compositor-Side Sandbox Tagging

The `wp_security_context_v1` Wayland protocol (staging, version 1) allows a sandbox engine — Flatpak, Snap — to tag all Wayland connections from sandboxed clients at the compositor level. This enables the compositor to enforce protocol restrictions without the portal daemon being the sole enforcement point.

### Protocol Usage

The sandbox engine (Flatpak's bwrap launcher, not the app itself) calls:

```c
// Manager interface: wp_security_context_manager_v1
// Method: create_listener(listen_fd h, close_fd h) → context wp_security_context_v1
//
//   listen_fd: bound+listening Unix socket; compositor accepts sandbox clients on it
//   close_fd:  when this fd becomes readable, compositor stops accepting new connections

// Context interface: wp_security_context_v1
// All setters may only be called once each (error: already_set if repeated):
wp_security_context_v1_set_sandbox_engine(context, "org.flatpak");
wp_security_context_v1_set_app_id(context, "org.gnome.Eog");
wp_security_context_v1_set_instance_id(context, instance_id_string);

// Atomically register all settings. After commit(), only destroy() is valid:
wp_security_context_v1_commit(context);
```

### Compositor Enforcement

Connections arriving on `listen_fd` are tagged with the security context metadata. The compositor uses the tag to:

- **Restrict advertised globals**: sandboxed clients may not see `wp_security_context_manager_v1` (preventing privilege escalation), `zxdg_exporter_v2`, or other privileged protocols.
- **Enable per-application protocol policy**: KWin (Plasma 6) uses the `app_id` tag to apply per-application Wayland protocol restrictions, which can be configured via GUI or policy files.
- **Identify sandbox engine**: `sandbox_engine = "org.flatpak"` vs `"io.snapcraft"` etc., allowing compositor policy to distinguish confinement levels.

The protocol explicitly forbids exposing `wp_security_context_manager_v1` to clients that are themselves already sandboxed — checked by the compositor at bind time using the connection's security context state.

### Integration with the Portal

The portal daemon sets up the security context before the sandboxed app connects to the compositor Wayland socket. The app receives a restricted socket (the `listen_fd` side); all its Wayland requests arrive on it. The compositor sees the security context metadata and enforces restrictions transparently, without the portal daemon needing to proxy every Wayland request.

---

## 13. libportal — The C Convenience Library

**libportal** (package `libportal-1`, version 0.10.1, LGPL-3.0) wraps the raw D-Bus pattern described in §3 and §5 into GObject/GIO-style async functions with automatic handle-path computation, subscription management, and type-safe result extraction. It is the recommended API for new applications.

### Core Types

```c
#include <libportal/portal.h>
#include <libportal/remote.h>

// Main portal object — one per application:
XdpPortal *portal = xdp_portal_new();         // aborts if D-Bus unavailable
XdpPortal *portal = xdp_portal_initable_new(GError **error);  // returns NULL on failure

// Window identifier for dialog parenting:
XdpParent *parent = xdp_parent_new_gtk(GTK_WINDOW(window));
xdp_parent_free(parent);
```

### Key Enumerations

```c
typedef enum {
    XDP_OUTPUT_NONE    = 0,
    XDP_OUTPUT_MONITOR = 1 << 0,
    XDP_OUTPUT_WINDOW  = 1 << 1,
    XDP_OUTPUT_VIRTUAL = 1 << 2,
} XdpOutputType;

typedef enum {
    XDP_CURSOR_MODE_HIDDEN   = 1 << 0,
    XDP_CURSOR_MODE_EMBEDDED = 1 << 1,
    XDP_CURSOR_MODE_METADATA = 1 << 2,
} XdpCursorMode;

typedef enum {
    XDP_PERSIST_MODE_NONE,       // 0
    XDP_PERSIST_MODE_TRANSIENT,  // 1 — app lifetime
    XDP_PERSIST_MODE_PERSISTENT, // 2 — until revoked
} XdpPersistMode;

typedef enum {
    XDP_SESSION_INITIAL,
    XDP_SESSION_ACTIVE,
    XDP_SESSION_CLOSED,
} XdpSessionState;
```

### ScreenCast Session (libportal)

```c
// Create session:
xdp_portal_create_screencast_session(portal,
    XDP_OUTPUT_MONITOR | XDP_OUTPUT_WINDOW,
    XDP_SCREENCAST_FLAG_NONE,
    XDP_CURSOR_MODE_EMBEDDED,
    XDP_PERSIST_MODE_TRANSIENT,
    NULL,          // restore_token (NULL = no prior session)
    NULL,          // cancellable
    on_session_created, user_data);

static void
on_session_created(GObject *src, GAsyncResult *res, gpointer data)
{
    GError *err = NULL;
    XdpSession *session =
        xdp_portal_create_screencast_session_finish(XDP_PORTAL(src), res, &err);
    if (!session) { g_warning("%s", err->message); return; }

    XdpParent *parent = xdp_parent_new_gtk(GTK_WINDOW(main_win));
    xdp_session_start(session, parent, NULL, on_session_started, NULL);
    xdp_parent_free(parent);
}

static void
on_session_started(GObject *src, GAsyncResult *res, gpointer data)
{
    GError *err = NULL;
    if (!xdp_session_start_finish(XDP_SESSION(src), res, &err)) return;

    int pw_fd = xdp_session_open_pipewire_remote(session); // §14
    GVariant *streams = xdp_session_get_streams(session);  // a(ua{sv})
    char *token = xdp_session_get_restore_token(session);  // save for next run
}

// Camera:
gboolean present = xdp_portal_is_camera_present(portal);
xdp_portal_access_camera(portal, parent, XDP_CAMERA_FLAG_NONE,
                          NULL, on_camera_granted, NULL);
int cam_pw_fd = xdp_portal_open_pipewire_remote_for_camera(portal);

// File chooser:
xdp_portal_open_file(portal, parent, "Open Image",
                      filters, current_filter, choices,
                      XDP_OPEN_FILE_FLAG_NONE,
                      NULL, on_file_opened, NULL);
GVariant *result = xdp_portal_open_file_finish(portal, res, &err);
// result contains: uris (as), choices (a(ss)), current_filter
```

---

## 14. PipeWire Integration: OpenPipeWireRemote

Both the ScreenCast and Camera portals return a Unix socket fd via `OpenPipeWireRemote`. This fd is a restricted PipeWire remote — only the session's permitted nodes are visible to the connecting client.

### Connecting with PipeWire

```c
#include <pipewire/pipewire.h>

// The fd is a Unix socket already connected to the PipeWire daemon.
// pw_context_connect_fd() takes ownership — do NOT close the fd separately.
struct pw_main_loop *loop = pw_main_loop_new(NULL);
struct pw_context  *ctx  = pw_context_new(pw_main_loop_get_loop(loop), NULL, 0);
struct pw_core     *core = pw_context_connect_fd(ctx, pw_fd, NULL, 0);
// pw_fd is now owned by core; closing core releases the socket
```

### Identifying the Correct Node (v6 API)

In ScreenCast interface v6, `node_id (u)` in the streams array is deprecated. Node IDs can be reused after monitor hotplug or suspend cycles. The `pipewire-serial (t)` property is a monotonically increasing 64-bit integer (`object.serial`) that is never reused:

```c
// From the Start Response (§5):
guint64 serial;
g_variant_lookup(stream_props, "pipewire-serial", "t", &serial);

// Convert to string for PW_KEY_TARGET_OBJECT:
char serial_str[32];
snprintf(serial_str, sizeof(serial_str), "%" PRIu64, serial);

// Create a PipeWire stream targeting by serial:
struct pw_properties *props = pw_properties_new(
    PW_KEY_MEDIA_TYPE,     "Video",
    PW_KEY_MEDIA_CATEGORY, "Capture",
    PW_KEY_MEDIA_ROLE,     "Screen",
    PW_KEY_TARGET_OBJECT,  serial_str,  // stable, never reused
    NULL);

struct pw_stream *stream = pw_stream_new(core, "screencast", props);
```

### WirePlumber Permissions

The portal daemon connects to the PipeWire daemon with the property `PW_KEY_ACCESS = "xdg-desktop-portal"`. WirePlumber (the PipeWire session manager) observes this and restricts the view of the remote connection:

- For ScreenCast: only the stream nodes created for this session are visible.
- For Camera: only nodes where `PW_KEY_MEDIA_CLASS` includes `Video/Source` (camera), gated by the permission granted through the Camera portal.

This enforcement is double-layered: the portal daemon only opens a PipeWire remote after the user has granted permission; WirePlumber further restricts what the client can see through that remote.

---

## 15. Backend Implementations

### xdg-desktop-portal-gnome

**Bus**: `org.freedesktop.impl.portal.desktop.gnome`  
**UseIn**: `gnome`  
**Interfaces**: Access, Account, AppChooser, Background, Clipboard, DynamicLauncher, FileChooser, GlobalShortcuts, InputCapture, Lockdown, Notification, Print, RemoteDesktop, ScreenCast, Screenshot, Settings, USB, Wallpaper

GNOME's backend delegates the heaviest interfaces to Mutter via internal D-Bus:
- **ScreenCast / RemoteDesktop**: calls `org.gnome.Mutter.ScreenCast` and `org.gnome.Mutter.RemoteDesktop` D-Bus interfaces on the running GNOME Shell / Mutter instance. Mutter is the Wayland compositor (Ch22) and holds the compositor-side frame data.
- **GlobalShortcuts**: via `org.gnome.GlobalShortcutsProvider`.
- **InputCapture**: via `org.gnome.Mutter.InputCapture`.
- **FileChooser**: GTK4 `GtkFileChooserDialog` shown as a system dialog.
- **Settings**: reads GSettings schemas (`org.gnome.desktop.interface`, etc.) and maps them to the standardised `org.freedesktop.appearance` keys.
- **Account**: via `org.freedesktop.Accounts` (AccountsService daemon).

### xdg-desktop-portal-kde

**Bus**: `org.freedesktop.impl.portal.desktop.kde`  
**UseIn**: `KDE`  
**Interfaces**: Access, Account, AppChooser, Background, Clipboard, Email, FileChooser, GlobalShortcuts, Inhibit, InputCapture, Print, RemoteDesktop, ScreenCast, Screenshot, Settings, DynamicLauncher, USB, Wallpaper

KDE's backend uses Qt/KF6 dialogs (QFileDialog, etc.) and integrates with KWin — the KDE Wayland compositor — for screen sharing. KWallet provides the Secret interface as a separate daemon. The backend provides an `org.freedesktop.impl.portal.Lockdown` implementation for enterprise policy (disable camera, disable printing, etc.).

### xdg-desktop-portal-wlr

**Bus**: `org.freedesktop.impl.portal.desktop.wlr`  
**UseIn**: `wlroots;sway;Wayfire;river;phosh;Hyprland`  
**Interfaces**: ScreenCast, Screenshot only

The minimal backend for wlroots-based compositors implements only the two compositor-dependent interfaces. Applications on wlroots desktops rely on `xdg-desktop-portal-gtk` for FileChooser and other interfaces, configured via `portals.conf`:

```ini
# ~/.config/sway-portals.conf for Sway
[preferred]
default=gtk
org.freedesktop.impl.portal.ScreenCast=wlr
org.freedesktop.impl.portal.Screenshot=wlr
```

The backend uses the `wlr-screencopy` protocol (a wlroots-proprietary extension) or `ext-image-copy-capture-v1` (the new standard) to capture frames from the compositor, then injects them into PipeWire as a source node.

### xdg-desktop-portal-hyprland

**Bus**: `org.freedesktop.impl.portal.desktop.hyprland`  
**UseIn**: `wlroots;Hyprland;sway;Wayfire;river`  
**Interfaces**: ScreenCast, Screenshot, GlobalShortcuts, InputCapture

A superset of the wlr backend: adds GlobalShortcuts and InputCapture. Uses Hyprland's native IPC for the source-picker UI (window chooser, monitor chooser), giving a more integrated UX than the separate `slurp`/`wofi` tools used by xdg-desktop-portal-wlr.

---

## 16. Security Model and CVEs

### The Security Boundary

The portal does not eliminate the attack surface; it reshapes it. The key properties:

- **Caller authentication** is based on filesystem inspection of `/proc/<pid>/root/.flatpak-info`. A race exists between PID lookup and file read; the daemon mitigates this using `pidfd` (Linux 5.3+) — the fd holds a reference to the process namespace regardless of PID reuse.

- **The Document FUSE filesystem** exposes only granted files. An app that receives a document grant can only read/write that file through `/run/user/$UID/doc/<id>/`, not the host path. This makes the grant portable across mount namespaces.

- **Camera / ScreenCast permissions** in the permission store are per-AppID. An app that spoofs its AppID (which requires escaping the Flatpak sandbox) can impersonate another. Sandbox integrity is a precondition of permission-store trust.

- **The GPU boundary**: the portal does not restrict GPU compute. A Flatpak app with `--device=dri` in its manifest has full access to the render node. There is no per-application GPU isolation below the DRM render node. (Ch111)

### Recent CVEs (xdg-desktop-portal 1.22.1, 2026-06-17)

**GHSA-c5cf-79w8-pvfh** — `FileTransfer.RetrieveFiles` used a predictable key for `StartTransfer`. An attacker with D-Bus access could predict the key and retrieve files intended for another application. Fixed by using cryptographically random keys.

**GHSA-cm83-2936-gxjm** — `FileChooser.SaveFiles` did not validate that the returned path was inside the user-chosen directory. A malicious backend could cause an arbitrary write. Fixed by validating the output path against the selected directory.

**App ID validation in Document Portal** — document grants were not correctly validated against the calling application's AppID, potentially allowing one app to access another's grants. Fixed in 1.22.0.

**PipeWire realtime module** — xdg-desktop-portal was loading PipeWire's realtime module (which calls `rtkit`), which could cause deadlocks in certain situations. Disabled in 1.22.1.

### Debugging

```bash
# Enable portal daemon verbose logging:
G_MESSAGES_DEBUG=all /usr/libexec/xdg-desktop-portal 2>&1 | grep -i portal

# Check which backend is active:
busctl list | grep impl.portal
# org.freedesktop.impl.portal.desktop.gnome
# org.freedesktop.impl.portal.PermissionStore

# Inspect the portals.conf resolution:
ls /usr/share/xdg-desktop-portal/portals/
cat ~/.config/gnome-portals.conf   # user override

# Watch D-Bus traffic:
dbus-monitor "sender=org.freedesktop.portal.Desktop" 2>&1 | head -100

# Inspect permissions:
gdbus call --session \
    --dest org.freedesktop.impl.portal.PermissionStore \
    --object-path /org/freedesktop/impl/portal/PermissionStore \
    --method org.freedesktop.impl.portal.PermissionStore.List \
    camera
```

---

## 17. Roadmap

This section tracks active development work and open proposals as of mid-2026. Status labels: **open PR** (merged to `main` not yet released), **draft PR** (work in progress, not ready for review), **discussion** (no code yet), **merged** (shipped in a release).

### 17.1 Versioning Scheme — major.minor

The current odd/even minor-version convention (odd = unstable, even = stable) will be dropped. PR [#2042](https://github.com/flatpak/xdg-desktop-portal/pull/2042) proposes renaming to a `YY.N` calendar-version scheme; the release after 1.22 would be `23.0`. The rationale is that GNOME OS nightly already ships from `main`, distros ship "unstable" releases anyway, and the convention adds friction without delivering isolation. **Status: open PR** (2026-06-23).

### 17.2 Interface Versioning RFC

Currently, breaking changes to a portal interface require a new `version` property increment and careful backwards-compat shims. PR [#1964](https://github.com/flatpak/xdg-desktop-portal/pull/1964) proposes major-version suffixes on D-Bus interface names (e.g. `org.freedesktop.portal.Camera2`) paired with an `active-revision` property carrying minor-version semantics. This would allow old and new interface versions to coexist during transition periods without a flag day. **Status: open RFC PR** (2026-04-10); maintainer consensus not yet reached.

### 17.3 ScreenCast — Audio Stream Support

PR [#1993](https://github.com/flatpak/xdg-desktop-portal/pull/1993) extends `org.freedesktop.portal.ScreenCast` (bumping to interface version 6) with simultaneous audio capture. `SelectSources()` gains a boolean `audio` option; `Start()` may return multiple PipeWire streams tagged with a `media_type` property (`"video"` or `"audio"`). Backends older than v6 have the option stripped transparently. The audio stream appears as a PipeWire node alongside the existing video node; the caller connects to it via the same `OpenPipeWireRemote` fd. **Status: open PR** (2026-05-04); blocked on backend implementation in xdg-desktop-portal-gnome and xdg-desktop-portal-kde.

### 17.4 WirePlumber Native Integration

The xdp daemon currently embeds its own PipeWire modules to manage stream permissions and camera node monitoring. The ongoing work moves this policy entirely into WirePlumber: a WirePlumber plugin handles camera monitoring, metadata, and cross-owner stream link permissions (`PW_PERM_L`), while xdp becomes a pure D-Bus frontend. The ScreenCast side of this migration merged during the 1.22 development cycle. Active PRs cover the metadata system and camera monitoring migration: [#2070](https://github.com/flatpak/xdg-desktop-portal/pull/2070) (draft, 2026-07-21) and [#2072](https://github.com/flatpak/xdg-desktop-portal/pull/2072) (open, 2026-07-22). Long-term this eliminates xdp's direct PipeWire manipulation entirely.

### 17.5 Async Portal Implementation with libdex

PR [#2036](https://github.com/flatpak/xdg-desktop-portal/pull/2036) adds infrastructure to implement portal handlers using libdex (the GNOME async I/O / Future library), enabling non-blocking I/O throughout the xdp daemon. PR [#1855](https://github.com/flatpak/xdg-desktop-portal/pull/1855) (draft) then ports individual portal implementations to use libdex. The motivation is eliminating GLib main-loop stalls on slow D-Bus round-trips to backends. **Status: #2036 open PR**; #1855 draft, blocked on #2036.

### 17.6 New Portal: Speech (Text-to-Speech)

PR [#1690](https://github.com/flatpak/xdg-desktop-portal/pull/1690) proposes `org.freedesktop.portal.Speech` with two methods: `GetProviders()` (list available TTS engines via libspiel) and `Synthesize()` (stream audio via a file descriptor). The portal relies on libspiel for provider discovery, avoiding reimplementation of the provider enumeration protocol. An API design question — whether to expose a flat voice list or the full provider/voice hierarchy — is still open. **Status: open PR** (2025-04-03, last updated 2026-02-07).

### 17.7 New Portal: Session Save/Restore

Draft PR [#1818](https://github.com/flatpak/xdg-desktop-portal/pull/1818) proposes a portal for applications to save and restore window state across compositor crashes and OS updates, reducing disruption from security reboots. The companion backend MR exists in gnome-session (MR#162) and a GTK client implementation is in progress (GTK MR#8980). **Status: draft PR** (2025-09-25); portal API design not yet finalised.

### 17.8 New Portal: Proxy Configuration

PR [#1828](https://github.com/flatpak/xdg-desktop-portal/pull/1828) adds a `Proxy` backend interface so GNOME and KDE backends can expose system proxy configuration to sandboxed apps. libproxy would consume this portal as its discovery mechanism, replacing the current `gsettings` / KConfig direct reads that fail inside strict sandboxes. **Status: open PR** (2025-10-05, last updated 2026-06-22).

### 17.9 New Portal: FileAccess (Runtime Path Request)

Draft PR [#1737](https://github.com/flatpak/xdg-desktop-portal/pull/1737) adds a third file-access mechanism beyond `--filesystem` manifest flags and `FileChooser`. A sandboxed app can request access to a specific known path at runtime via a yes/no user prompt; approval grants access through the document store. Target use cases: `~/.steam/steam/config/libraryfolders.vdf`, `/usr/share/applications/`. Missing: permission persistence, XDG directory handling. **Status: draft PR** (2025-06-10).

### 17.10 New Portal: Credentials (Experimental)

PR [#1889](https://github.com/flatpak/xdg-desktop-portal/pull/1889) introduces an experimental `org.freedesktop.portal.Credentials` portal for credential storage and retrieval, gated behind `XDG_DESKTOP_PORTAL_ENABLE_EXPERIMENTAL`. The interface design is not finalised: a `DiscoverCredential(services as)` method, mock backend, and unit tests are all missing. This work is related to the broader experimental-portal infrastructure being discussed in issue [#2066](https://github.com/flatpak/xdg-desktop-portal/issues/2066). **Status: open WIP PR** (2026-01-23).

### 17.11 New Portal: App-to-App ScreenCast

Draft PR [#1554](https://github.com/flatpak/xdg-desktop-portal/pull/1554) proposes a `ScreenCastProvider` mechanism allowing applications to share their own DMA-BUF surfaces to other applications without video encoding. The sharing path uses P2P D-Bus between provider and portal, with WirePlumber granting `PW_PERM_L` for cross-owner node linking. This would enable e.g. a game streaming server to share its rendered framebuffer directly to a client app without a CPU copy. **Status: draft PR** (2025-01-05); WirePlumber Lua policy script and DMA-BUF integration issues noted.

### 17.12 Settings Portal Extensions

Two additions to `org.freedesktop.portal.Settings` are in progress:

- **Cursor theme** ([PR #1539](https://github.com/flatpak/xdg-desktop-portal/pull/1539)): `cursor-theme` and `cursor-theme-size` keys, enabling sandboxed apps to load the correct cursor theme from the host without filesystem access to `~/.icons`. **Status: open PR** (2024-12-xx).
- **Button placement** ([PR #1821](https://github.com/flatpak/xdg-desktop-portal/pull/1821)): `button-placement` key exposing the GNOME/KDE window button order preference. **Status: draft PR** (2025-09-xx).

Both require backend implementations in xdg-desktop-portal-gnome and xdg-desktop-portal-kde. For reference, `accent-color` shipped in 1.17.1 and `reduced-motion` shipped in 1.21.0.

### 17.13 Transient Permission Expiry

Screen-cast and remote-desktop sessions create "transient" permissions that currently persist indefinitely in the permission store. Draft PR [#990](https://github.com/flatpak/xdg-desktop-portal/pull/990) proposes automatically discarding them after 24 hours, preventing silent re-use of a screen-share grant across reboots. Related issues: [#325](https://github.com/flatpak/xdg-desktop-portal/issues/325), [#981](https://github.com/flatpak/xdg-desktop-portal/issues/981). **Status: draft PR** (open since 2023-03-23); policy decision on timeout value not yet reached.

### 17.14 InputCapture — KDE and wlroots Backends

The `org.freedesktop.portal.InputCapture` interface (version 2, with `CreateSession2`) is implemented in the GNOME backend (via Mutter MR#2628 and the RemoteDesktop `ConnectToEIS` path). KDE Plasma support is tracked in [xdg-desktop-portal-kde issue #12](https://invent.kde.org/plasma/xdg-desktop-portal-kde/-/issues/12) — not yet implemented as of mid-2026, pending KWin-side EIS support. xdg-desktop-portal-wlr is in maintenance mode and unlikely to gain InputCapture; wlroots-based compositors are expected to implement it in their own backends (Hyprland, niri, etc.) as EIS support lands in the compositor.

### 17.15 Portals Considered but Not Started

Several commonly requested capabilities have open issues but no active implementation:

- **hidraw portal** ([#611](https://github.com/flatpak/xdg-desktop-portal/issues/611)): requires `HIDIOCREVOKE` kernel ioctl, scheduled for Linux 6.12 (Sep 2024); no portal PR yet.
- **USB serial port portal** ([#229](https://github.com/flatpak/xdg-desktop-portal/issues/229)): open since 2018, body reads "TBD".
- **MIDI portal**: no dedicated issue; software MIDI via PipeWire is not addressed by any portal; USB MIDI hardware access goes through the existing USB portal (merged 1.19.1, January 2025).
- **Background portal graceful termination** ([#975](https://github.com/flatpak/xdg-desktop-portal/issues/975)): open since 2023, no PR.
- **Password manager portal** ([discussion #2061](https://github.com/flatpak/xdg-desktop-portal/discussions/2061), July 2026): early design discussion only.

---

## Integrations

**Chapter 23 — Legacy and Sandboxed App Support** is where XWayland and the original portal overview appear. This chapter supersedes the portal coverage in Ch23, which should be read for XWayland context and the broader sandboxing landscape.

**Chapter 38 — PipeWire** is the downstream consumer of both `ScreenCast.OpenPipeWireRemote` and `Camera.OpenPipeWireRemote`. The PipeWire node model (`pw_context_connect_fd`, stream connection, `PW_KEY_TARGET_OBJECT` with `pipewire-serial`) is covered in depth there. WirePlumber's permission enforcement for portal-opened PipeWire remotes is implemented in the WirePlumber scripts discussed in that chapter.

**Chapter 46 — Staging Wayland Protocols** covers `wp_security_context_v1` in the context of Mutter 50.x and KWin 6.7.x implementation timelines, and the `ext-image-copy-capture-v1` protocol that backends are migrating to from `wlr-screencopy` for compositor-independent screen capture.

**Chapter 96 — libcamera** describes the Camera portal's upstream: libcamera's pipeline handlers produce DMA-BUF frames that PipeWire's camera adapter exposes as nodes. The `Camera.OpenPipeWireRemote` fd exposes exactly those nodes to the sandboxed caller.

**Chapter 111 — Flatpak Graphics** covers GPU access in the Flatpak sandbox: the `--device=dri` manifest entry, Mesa GL extension mounting, and Vulkan ICD discovery inside the namespace. That chapter explains the security boundary (render node grant = full GPU access) that the portal does not close.

**Chapter 20 — Wayland Protocol Fundamentals** covers the `xdg-foreign` protocol used to generate `parent_window` handles (`"wayland:<handle>"`) for modal dialog parenting, and the `xdg-activation-v1` protocol used for `activation_token` parameters in OpenURI and Email.

**Chapter 22 — Production Compositors** covers Mutter (GNOME) and KWin (KDE) — the compositors that xdg-desktop-portal-gnome and xdg-desktop-portal-kde call via `org.gnome.Mutter.ScreenCast` and KWin's equivalents. The compositor's screencopy implementation is what the portal frontend ultimately exercises.

**Chapter 206 — SDL3** covers `SDL_ShowOpenFileDialog`, which calls `org.freedesktop.portal.FileChooser` on Linux, and `SDL_OpenCamera`, which calls `org.freedesktop.portal.Camera`. SDL3 also uses `org.freedesktop.portal.OpenURI` for `SDL_OpenURL`.

---

## References

- [flatpak/xdg-desktop-portal — GitHub](https://github.com/flatpak/xdg-desktop-portal) — Source repository; XML D-Bus interface definitions in `data/`
- [xdg-desktop-portal API Documentation](https://flatpak.github.io/xdg-desktop-portal/docs/) — Official interface reference
- [flatpak/libportal — GitHub](https://github.com/flatpak/libportal) — C convenience library wrapping the D-Bus pattern
- [xdg-desktop-portal 1.22.1 Release](https://github.com/flatpak/xdg-desktop-portal/releases/tag/1.22.1) — Security fixes, ScreenCast v6 stable
- [xdg-desktop-portal-gnome — GNOME GitLab](https://gitlab.gnome.org/GNOME/xdg-desktop-portal-gnome) — GNOME backend; Mutter integration
- [xdg-desktop-portal-kde — KDE Invent](https://invent.kde.org/plasma/xdg-desktop-portal-kde) — KDE backend; KWin screen sharing
- [xdg-desktop-portal-wlr — GitHub](https://github.com/emersion/xdg-desktop-portal-wlr) — wlroots backend; wlr-screencopy and ext-image-copy-capture integration
- [xdg-desktop-portal-hyprland — GitHub](https://github.com/hyprwm/xdg-desktop-portal-hyprland) — Hyprland backend; GlobalShortcuts + InputCapture additions
- [wp_security_context_v1 — wayland.app](https://wayland.app/protocols/security-context-v1) — Protocol specification and rationale
- [PipeWire — pipewire.pages.freedesktop.org](https://pipewire.pages.freedesktop.org/pipewire/) — PipeWire API reference; pw_context_connect_fd, PW_KEY_TARGET_OBJECT
- [libei — freedesktop GitLab](https://gitlab.freedesktop.org/libinput/libei) — Emulated Input library used by RemoteDesktop ConnectToEIS and InputCapture
- [GDBus manual — GNOME Developer Docs](https://docs.gtk.org/gio/class.DBusConnection.html) — G_DBUS signal subscription, unix fd lists
- [ScreenCast interface v6 changes — portal changelog](https://github.com/flatpak/xdg-desktop-portal/blob/main/CHANGELOG.md) — pipewire-serial replaces node_id
- [Permission Store wiki — GitHub wiki](https://github.com/flatpak/xdg-desktop-portal/wiki/The-Permission-Store) — GVariant on-disk format and table schema
- [GHSA-c5cf-79w8-pvfh](https://github.com/flatpak/xdg-desktop-portal/security/advisories/GHSA-c5cf-79w8-pvfh) — FileTransfer predictable key CVE (1.22.1)
- [GHSA-cm83-2936-gxjm](https://github.com/flatpak/xdg-desktop-portal/security/advisories/GHSA-cm83-2936-gxjm) — FileChooser.SaveFiles arbitrary write CVE (1.22.1)
