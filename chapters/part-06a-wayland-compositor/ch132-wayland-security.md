# Chapter 132: Wayland Security

**Part VI — The Display Stack**

**Audiences:**

- **Security engineers** — need a rigorous threat-model analysis of the Wayland stack
- **Wayland compositor authors** — must decide which protocols to expose and with what access controls
- **Application sandboxing engineers** — need to understand what Flatpak's `--socket=wayland` permission actually grants
- Anyone evaluating Wayland against X11 from a security perspective

The chapter assumes familiarity with core Wayland concepts (surfaces, seats, the object model); see Chapter 20 for that background.

---

## Table of Contents

1. [Introduction: The Security Contract Shift](#1-introduction-the-security-contract-shift)
2. [X11 Security Model and Its Fundamental Problems](#2-x11-security-model-and-its-fundamental-problems)
3. [Wayland's Isolation Model](#3-waylands-isolation-model)
4. [Attack Surfaces Remaining in Wayland](#4-attack-surfaces-remaining-in-wayland)
5. [xdg-desktop-portal as Permission Gatekeeper](#5-xdg-desktop-portal-as-permission-gatekeeper)
6. [Sandboxed Apps and Wayland: Flatpak Security](#6-sandboxed-apps-and-wayland-flatpak-security)
7. [Virtual Keyboard and Input Injection Security](#7-virtual-keyboard-and-input-injection-security)
8. [DRM Render Node Access Control](#8-drm-render-node-access-control)
9. [Clipboard and Drag-and-Drop Security](#9-clipboard-and-drag-and-drop-security)
10. [Practical Hardening Recommendations](#10-practical-hardening-recommendations)
11. [Integrations](#integrations)

---

## 1. Introduction: The Security Contract Shift

The Linux graphics stack's security story changed fundamentally when compositors migrated from X11 to Wayland. This is not a minor protocol update — it is a restructuring of trust boundaries. Under X11, every client that can connect to the display server is implicitly trusted with extraordinary capabilities: reading any other window's pixels, injecting synthetic keyboard and mouse events into any focused window, intercepting global keyboard state, and capturing the entire screen without any permission check. Any malicious or compromised application on an X11 session can trivially become a keylogger, a clipboard thief, or a screen recorder with no special privileges beyond a connection to the display socket.

Wayland's design principle is the opposite: clients receive only what they need. The compositor is the sole arbiter of what a client may see and do. A client receives `wl_keyboard` events only for surfaces it owns. It cannot issue requests that observe another client's pixels. It cannot grab the keyboard globally. This isolation is not implemented as a policy add-on or a kernel access-control mechanism — it is structural: the Wayland protocol simply does not contain the requests that would be needed to perform those cross-client operations.

This chapter provides a rigorous analysis of:

- What X11 allowed (and why it was catastrophic for isolation)
- What the Wayland core protocol guarantees, and the implementation mechanisms behind those guarantees
- The attack surfaces that remain — screencopy, virtual keyboard injection, XWayland, DMA-BUF leakage, clipboard managers, and the `$XDG_RUNTIME_DIR` socket itself
- How `xdg-desktop-portal` fills the permission-gatekeeper role for sandboxed applications
- How Flatpak's bubblewrap sandbox interacts with compositor-enforced Wayland access control
- How `wp_security_context_v1` (a staging protocol in wayland-protocols, shipping in version 1.48 as of April 2026) enables compositors to enforce ACLs based on sandbox identity
- Practical hardening steps for compositor authors, application packagers, and system administrators

---

## 2. X11 Security Model and Its Fundamental Problems

### 2.1 The Shared Display Server Model

The core of X11's security problem is architectural. All clients connect to a single Xorg process (or XCB/Xlib to the same socket), which acts as an omniscient display server. The server maintains a single flat namespace of windows — identified by 32-bit `Window` XIDs — and every client can reference every other client's windows by XID. There is no per-client isolation of the window tree. The server will execute requests against any window from any client that knows the XID, and XID enumeration is trivial: `XQueryTree` walks the entire tree from the root window and returns all children recursively.

### 2.2 Pixel Capture: XGetImage and XShmGetImage

Any X11 client can call `XGetImage` on any window, including the root window (which covers the entire display), and receive the pixels:

```c
/* Capture the entire display — no permission check, no user prompt */
XImage *img = XGetImage(display,
                        DefaultRootWindow(display),
                        0, 0,
                        DisplayWidth(display, screen),
                        DisplayHeight(display, screen),
                        AllPlanes, ZPixmap);
```

The `XShmGetImage` variant uses the MIT-SHM shared-memory extension to avoid a copy, making screen capture essentially free. Any application that connects to the display can do this. There is no authentication step, no capability, no ACL. The only requirement is that the caller has a connection to the X server — which means every application on a logged-in session.

### 2.3 Input Injection: XSendEvent and the XTEST Extension

**XSendEvent** allows any client to deliver synthetic events to any window:

```c
/* Inject a keypress to any window — no permission required */
XKeyEvent ev = {0};
ev.type        = KeyPress;
ev.display     = display;
ev.window      = target_window;
ev.keycode     = XKeysymToKeycode(display, XK_Return);
ev.state       = 0;
ev.time        = CurrentTime;
XSendEvent(display, target_window, True, KeyPressMask, (XEvent *)&ev);
```

The **XTEST** extension goes further: `XTestFakeKeyEvent` injects raw key events into the input stream itself, bypassing window targeting entirely. Any application can simulate arbitrary keystrokes that will reach whichever window currently holds focus. This is used by accessibility tools, automated testing, and — trivially — by keyloggers running in reverse: inject credentials into a password field, or inject commands into a terminal.

### 2.4 Global Keyboard Grab: XGrabKeyboard and XGrabKey

`XGrabKeyboard` allows an application to intercept *all* keyboard input, routing it away from every other window. `XGrabKey` registers for a specific key combination globally — the mechanism behind global media keys in X11 desktops. A malicious application can use `XGrabKey` to listen to any specific key sequence across all applications. Combined with `XQueryKeymap`, which returns the current state of all 256 keycodes, a keylogger can poll the entire keyboard in a tight loop:

```c
/* X11 keylogger in ~10 lines — no special privileges required */
char keys_pressed[32];
while (1) {
    XQueryKeymap(display, keys_pressed);
    /* Inspect keys_pressed bitmask for any key state change */
    usleep(10000); /* 100Hz polling */
}
```

This is not a theoretical attack. It is trivially reproducible. Any application that runs in an X11 session with `DISPLAY` set can implement a complete keylogger with no special permissions. Browser passwords, sudo passphrases, encryption keys typed at the keyboard — all are accessible to any co-resident application.

### 2.5 Clipboard Theft

The X11 clipboard (CLIPBOARD selection) and primary selection are accessible to any client. `XGetSelectionOwner` identifies the current owner, and `XConvertSelection` requests the data. No focus requirement exists for reading; an application running entirely in the background can poll the clipboard on every `SelectionNotify` event and log every copy operation the user performs.

### 2.6 X11 Access Control Mechanisms: Insufficient

X11 provides two nominal access-control mechanisms:

**xauth MIT-MAGIC-COOKIE**: The compositor places a random cookie in `~/.Xauthority`. Clients must present this cookie when connecting. This prevents remote attackers from connecting to an exposed TCP X server, but provides no intra-session isolation. Every application running as the same user has the same cookie and the same privileges.

**xhost**: IP-based access control for the TCP listener. Completely irrelevant on modern systems where X11 runs over a Unix domain socket.

The X11 `SECURITY` extension (RFC from 1996) defines "untrusted" clients with restricted capabilities — they cannot read pixmaps of trusted clients, cannot grab the keyboard, etc. However, [no modern toolkit or application runtime requests "untrusted" mode by default](https://www.osnews.com/story/142962/the-x11-security-extension-from-the-1990s/), and the extension is essentially unimplemented in production.

### 2.7 Recent CVEs Confirming the Problem

The X.Org server continues to receive security patches for fundamental architecture flaws. In early 2025, CVE-2025-62229, CVE-2025-62230, and CVE-2025-62231 disclosed use-after-free and overflow vulnerabilities in Xorg dating back to X11R6 — some over 20 years old. In 2024, multiple vulnerabilities were patched in XWayland itself: heap buffer overreads in `ProcXIGetSelectedEvents` and `ProcXIPassiveGrabDevice`, plus a use-after-free in `ProcRenderAddGlyphs` ([XOrg Security Advisories](https://www.x.org/wiki/Development/Security/)). These are not isolated bugs; they reflect the systemic complexity of a codebase that must implement a security model never designed for process isolation.

---

## 3. Wayland's Isolation Model

### 3.1 The Structural Security Guarantee

Wayland's security model is not implemented as a set of policy checks layered over a shared protocol — it is structural. The Wayland core protocol (`wayland.xml` in libwayland) simply does not contain requests that would allow cross-client observation. There is no `get_window_pixels` request, no `inject_key_event` request, no `grab_keyboard_globally` request. The compositor processes every request in the context of the client that sent it, and `wl_resource` objects are per-client: object ID 5 in client A's context is entirely separate from object ID 5 in client B's context. One client cannot reference another client's resources at all.

This is the foundational difference. On X11, the server enforces what clients *may not* do (a small, incomplete deny-list). On Wayland, clients can only do what the protocol *allows* (a small, carefully considered allow-list).

### 3.2 Input Isolation: wl_keyboard and wl_pointer Events

The compositor routes `wl_keyboard.key`, `wl_keyboard.modifiers`, and `wl_keyboard.enter` events only to the client whose surface currently holds keyboard focus. This routing is determined entirely by the compositor's internal focus tracking. There is no protocol mechanism by which client A can learn which key is pressed when client B has focus. The `wl_keyboard` object bound to client A only ever receives events for client A's surfaces.

Similarly, `wl_pointer.motion`, `wl_pointer.button`, and `wl_pointer.enter` are scoped to the client whose surface the pointer is currently over. A client that does not have pointer focus receives no pointer events at all.

There is no `wl_keyboard_grab` or `wl_pointer_grab` request in the core Wayland protocol. Popup grabs (`xdg_popup.grab`) are a constrained variant: the compositor delivers events to the popup client, and if a click lands outside the popup's surface, the compositor dismisses the popup and ends the grab. This is under complete compositor control and does not expose raw input to arbitrary clients.

### 3.3 Pixel Isolation: No XGetImage Equivalent

The core Wayland protocol contains no mechanism for reading another client's surface pixels. A client receives a `wl_buffer` to write into (its own render target), and the compositor reads from that buffer when compositing — but the compositor never sends pixels back to clients. The only way a client can observe screen content is through extension protocols explicitly designed for that purpose (see Section 4), which compositors can choose not to expose or to restrict.

### 3.4 Object Ownership and Per-Client Resource Isolation

The Wayland server maintains a per-client resource map. When client A creates a `wl_surface` (object ID 5), that surface is registered in client A's resource table. Client B has no knowledge of this object. Client B cannot issue requests against it. The integer IDs are local to each connection — they are not global identifiers visible across clients.

This is enforced by `libwayland-server`'s resource lookup mechanism. Every incoming request is dispatched via `wl_resource_from_link` in the context of the sending client's `wl_client`. If client B sends a request for an object ID that does not exist in client B's resource table, the compositor sends a `wl_display.error` event with code `WL_DISPLAY_ERROR_INVALID_OBJECT` and disconnects the client.

```c
/* libwayland-server: resource lookup is always client-scoped */
struct wl_resource *
wl_client_get_object(struct wl_client *client, uint32_t id)
{
    struct wl_map *objects = &client->objects;
    return wl_map_lookup(objects, id);
    /* Returns NULL if ID is not in THIS client's map */
}
```
[Source: libwayland-server, `src/` tree](https://gitlab.freedesktop.org/wayland/wayland/-/tree/main/src)

### 3.5 Unix Socket Authentication: XDG_RUNTIME_DIR

The Wayland compositor creates its socket at `$XDG_RUNTIME_DIR/wayland-0` (or a compositor-chosen name, advertised via `$WAYLAND_DISPLAY`). `$XDG_RUNTIME_DIR` is created by `systemd-logind` (or pam_runtime on other systems) with mode `0700` and ownership set to the logged-in user:

```bash
$ stat $XDG_RUNTIME_DIR
  File: /run/user/1000
  Size: 320        Blocks: 0
Access: (0700/drwx------)  Uid: ( 1000/  jreuben1)  Gid: ( 1000/  jreuben1)
```

This means the socket is inaccessible to other local users. The compositor additionally verifies the connecting process via `getsockopt(SO_PEERCRED)` on the accepted connection, which returns the `ucred` struct (uid, gid, pid) of the connecting process. This lets the compositor verify that the connection came from the session owner — and is the mechanism that `wp_security_context_v1` builds upon for sandbox identity verification.

### 3.6 No Shared Memory by Default

Each Wayland client allocates its own shared-memory pool via `wl_shm_create_pool`, which creates an anonymous memfd and shares it with the compositor via `SCM_RIGHTS` ancillary data. The compositor maps this pool read-only for compositing. No other client can access this pool. There is no global shared framebuffer as existed with X11's shared pixmaps.

### 3.7 Surface-to-Surface Privacy: Popup Hierarchy

`xdg_popup` surfaces must declare a parent relationship — either to an `xdg_toplevel` or another `xdg_popup` — at creation time. The compositor enforces this hierarchy: a popup without a valid parent is rejected. This prevents a client from creating a surface that appears to belong to another client's window hierarchy, mitigating UI spoofing attacks.

---

## 4. Attack Surfaces Remaining in Wayland

Despite the strong baseline isolation, Wayland is not a perfect security boundary. The following attack surfaces are real and require compositor-level or system-level mitigations.

### 4.1 Screen Capture: zwlr_screencopy_manager_v1 and ext-image-copy-capture-v1

**This is the most significant remaining attack surface in Wayland.** The wlroots ecosystem defines `zwlr_screencopy_manager_v1` ([protocol XML](https://github.com/swaywm/wlroots/blob/master/protocol/wlr-screencopy-unstable-v1.xml)) as a privileged protocol that allows a client to request a copy of any output's framebuffer:

```c
/* Client creates a screencopy frame — no user prompt in bare wlroots */
struct zwlr_screencopy_frame_v1 *frame =
    zwlr_screencopy_manager_v1_capture_output(screencopy_manager,
                                              0 /* overlay_cursor */,
                                              output);
```

The problem: **the protocol itself contains no ACL mechanism**. When a compositor like Sway exposes `zwlr_screencopy_manager_v1` globally (i.e., advertises it in the `wl_registry`), *any* client that binds to it can capture the entire screen. Tools like `grim` and `wf-recorder` rely on this — but so can malicious applications.

As of 2024–2025, this is a confirmed open issue in the wlroots/Hyprland/Sway ecosystems. A Flatpak application with `--socket=wayland` can invoke `zwlr_screencopy_manager_v1` directly, bypassing the `xdg-desktop-portal` entirely. [Hyprland issue #4432](https://github.com/hyprwm/Hyprland/issues/4432) documents this precisely: "any graphical program that has access to the Wayland socket can just invoke the screencopy protocol directly." The issue was closed as "not planned" — compositors must implement their own policy.

The newer `ext-image-copy-capture-v1` (wayland-protocols staging) has the same structural property: the compositor decides which clients can bind the manager interface. GNOME's Mutter and KDE's KWin restrict screen capture to clients that have gone through the portal; wlroots-based compositors typically do not.

**Mitigation**: Compositors must not advertise `zwlr_screencopy_manager_v1` or `ext_image_copy_capture_manager_v1` to arbitrary clients. Only the portal backend process (e.g., `xdg-desktop-portal-wlr`) should be granted access, identified via socket credentials or `wp_security_context_v1` metadata.

### 4.2 Input Injection: Virtual Keyboard and Input Method Protocols

The `zwp_virtual_keyboard_manager_v1` protocol ([wlroots protocol XML](https://github.com/swaywm/wlroots/blob/master/protocol/virtual-keyboard-unstable-v1.xml)) allows a client to create a virtual keyboard that injects keystrokes into the compositor's input stream:

```c
struct zwp_virtual_keyboard_v1 *vkbd =
    zwp_virtual_keyboard_manager_v1_create_virtual_keyboard(
        vk_manager, seat);
zwp_virtual_keyboard_v1_key(vkbd, timestamp, KEY_ENTER, WL_KEYBOARD_KEY_STATE_PRESSED);
```

The protocol's own description notes: "If the compositor enables a keyboard to perform arbitrary actions, it should present an error when an untrusted client requests a new keyboard." An `unauthorized` error (value 0) is defined. However, the protocol places all enforcement responsibility on the compositor with no built-in ACL — compositors must implement their own authorization logic.

A malicious application that successfully binds `zwp_virtual_keyboard_manager_v1` can inject arbitrary keystrokes. When a password dialog has focus, this allows credential theft through injection (type into a password manager, trigger autofill, exfiltrate the result) or simple disruptive attacks.

Legitimate consumers of this protocol include `ibus-daemon`, `fcitx5` (input method engines), and remote desktop servers. Compositors should restrict access to these specific, well-known processes.

### 4.3 XWayland: The X11 Security Island

XWayland runs a full X server process to support legacy X11 applications, and that X server grants all XWayland clients the full X11 privilege set. Two applications running via XWayland are isolated from Wayland-native applications (the X11 apps cannot read Wayland surfaces) but are **not** isolated from each other. An XWayland application can:

- Use `XGetImage` to capture any other XWayland window's pixels
- Use `XQueryKeymap` to poll keyboard state when any XWayland app has focus
- Use `XSendEvent` to inject events into any other XWayland window
- Use `XGrabKey` to intercept specific key sequences globally within the XWayland session

This is the most commonly underestimated security gap in current Linux desktops. As long as XWayland is present, the security properties of the session degrade to X11 security for all legacy applications. GNOME 50 (released March 2026) dropped the X11 session entirely — users can no longer log into a GNOME session via X11 even if Xorg is installed. XWayland itself is retained for running legacy X11 applications inside the Wayland session, but eliminating the X11 session means GNOME Shell itself no longer runs under X11, dramatically shrinking the attack surface. The remaining XWayland island only affects legacy applications that cannot or have not been ported to Wayland.

### 4.4 DMA-BUF File Descriptor Leakage

DMA-BUF FDs are unforgeable kernel file descriptors, but they are transferable via Unix socket ancillary data (`SCM_RIGHTS`). If a compositor or a poorly-implemented Wayland service passes a DMA-BUF FD representing a surface's backing storage to a client that should not receive it, that client can `mmap` the FD and read the pixel data. The Linux kernel's `O_CLOEXEC` flag prevents FD inheritance across `exec`, but does not prevent deliberate FD passing.

Compositors that implement import (`zwp_linux_dmabuf_v1`) and export (`zwlr_export_dmabuf_manager_v1`) should audit all paths where DMA-BUF FDs transit client boundaries.

### 4.5 Wayland Socket Exposure

If `$XDG_RUNTIME_DIR` is misconfigured (wrong permissions, accessible to other users), the Wayland socket becomes accessible to untrusted processes. This bypasses all compositor-level access control entirely. Standard `systemd-logind` configuration prevents this, but manual configurations or embedded systems may deviate.

```bash
# Check XDG_RUNTIME_DIR permissions — should be 0700
stat -c "%a %U %G" $XDG_RUNTIME_DIR
# Expected output: 700 <username> <groupname>
```

### 4.6 Clipboard Manager Access

Clipboard managers by design receive all clipboard content. The `ext-data-control-v1` protocol (the production successor to the deprecated `zwlr-data-control-v1`) grants a "privileged client" control over all seat selections. A clipboard manager binding `ext_data_control_manager_v1` receives `data_offer` events for every clipboard change, regardless of which application performed the copy. The compositor must restrict which clients can bind this manager.

---

## 5. xdg-desktop-portal as Permission Gatekeeper

### 5.1 Architecture Overview

`xdg-desktop-portal` ([GitHub](https://github.com/flatpak/xdg-desktop-portal)) is a D-Bus service that mediates sensitive desktop operations for sandboxed applications. It runs as a user daemon (`xdg-desktop-portal.service` under `systemd --user`) and provides a stable D-Bus API that sandboxed apps call instead of accessing hardware or compositor protocols directly.

The architecture has two layers:

- **Frontend** (`xdg-desktop-portal`): The stable D-Bus API surface that sandboxed applications call. Handles request routing, permission store lookup, and user interaction flow.
- **Backend** (`xdg-desktop-portal-gnome`, `xdg-desktop-portal-kde`, `xdg-desktop-portal-wlr`, etc.): Desktop-environment-specific implementations of `org.freedesktop.impl.portal.*` interfaces. The backend talks directly to the compositor (e.g., via PipeWire for screencasting, via compositor Wayland protocols for screen selection).

### 5.2 Key Portal Interfaces

| Frontend Interface | Purpose | User Prompt |
|---|---|---|
| `org.freedesktop.portal.Screenshot` | Capture a screenshot | Yes — always |
| `org.freedesktop.portal.ScreenCast` | Capture a video stream | Yes — source selection |
| `org.freedesktop.portal.RemoteDesktop` | Remote control (input injection) | Yes — always |
| `org.freedesktop.portal.Camera` | Access camera devices | Yes — first time |
| `org.freedesktop.portal.Clipboard` | Read/write clipboard | Compositor-dependent |
| `org.freedesktop.portal.FileChooser` | Open/save file dialogs | Yes — always (user picks) |
| `org.freedesktop.portal.Notification` | Send desktop notifications | Policy-based |

The `ScreenCast` portal ([documentation](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.impl.portal.ScreenCast.html)) is particularly important for security. A sandboxed application calls `CreateSession()`, `SelectSources()`, and `Start()` via D-Bus, and the compositor backend presents a picker UI asking the user to select which monitor or window to share. Only the selected source is streamed to the application via PipeWire.

### 5.3 The Permission Store

Granted permissions are persisted in a SQLite-backed D-Bus service: `org.freedesktop.impl.portal.PermissionStore`. The database lives at `~/.local/share/xdg-desktop-portal/` and maps `(table, id, app_id) → permission_flags`.

```bash
# List all stored portal permissions
gdbus call --session \
  --dest org.freedesktop.impl.portal.PermissionStore \
  --object-path /org/freedesktop/impl/portal/PermissionStore \
  --method org.freedesktop.impl.portal.PermissionStore.List \
  "screencast"

# Using flatpak tooling (for Flatpak app permissions)
flatpak permission-list
```

The `ScreenCast` portal supports three persistence modes via the `persist_mode` parameter:

- `0` — Non-persistent: user must re-approve every session
- `1` — Application-lifetime: persists until the app exits
- `2` — Indefinite: stored in the permission store; future calls can use a `restore_token` to bypass the picker UI

The `restore_token` mechanism is a security-relevant design: once a user grants a persistent screencast permission, the application can start screen capture in future sessions without showing a prompt. Revoking this requires explicitly deleting the stored permission:

```bash
# Revoke a stored screencast permission for a specific app
gdbus call --session \
  --dest org.freedesktop.impl.portal.PermissionStore \
  --object-path /org/freedesktop/impl/portal/PermissionStore \
  --method org.freedesktop.impl.portal.PermissionStore.DeletePermission \
  "screencast" "MONITOR_ID" "com.example.app"
```

### 5.4 AppID Identification

The portal identifies the calling application via its `app_id`, which the portal frontend resolves from the D-Bus connection's process credentials (`SO_PEERCRED` → pid → `/proc/<pid>/cgroup` → Flatpak `app_id`). For Flatpak applications, this is the application identifier from the manifest (e.g., `org.videolan.VLC`). For non-sandboxed apps, the ID is derived from process information and may be less reliable.

The frontend validates the app_id before consulting the permission store. An app cannot claim an arbitrary app_id — the portal derives it from the kernel-verified process identity.

### 5.5 Portal as TOCTOU Mitigation

A subtle security property of the portal architecture is that the portal itself opens the resource (file, camera FD, PipeWire node), not the sandboxed application. This eliminates a class of TOCTOU (time-of-check-time-of-use) attacks where an application might substitute a different resource between the user's approval and the access. The portal opens the PipeWire node and hands the FD to the application, ensuring the object handed over matches what the user approved.

---

## 6. Sandboxed Apps and Wayland: Flatpak Security

### 6.1 Bubblewrap and Namespace Isolation

Flatpak uses `bubblewrap` (version 0.11+ in recent releases) to create containerized application environments using Linux namespaces:

- **User namespace**: Maps container UID/GID to host UIDs
- **Mount namespace**: Restricts filesystem visibility to the sandbox tree
- **Network namespace**: Controlled by `--share=network` / `--unshare=network`
- **PID namespace**: Isolates the process tree
- **IPC namespace**: Controlled by `--share=ipc` / `--unshare=ipc`

Flatpak permissions are declared in the application manifest and managed via:

```bash
# Inspect app permissions
flatpak info --show-permissions com.example.MyApp

# Override permissions (user-scoped)
flatpak override --user --unshare=ipc com.example.MyApp
flatpak override --user --nodevice=all com.example.MyApp
```

### 6.2 --socket=wayland: What It Actually Grants

The `--socket=wayland` Flatpak permission grants the sandboxed application access to `$XDG_RUNTIME_DIR/wayland-0`. This is a **single permission that grants access to the full Wayland compositor API**, including any privileged extension protocols the compositor exposes globally.

This means that on a wlroots-based compositor that exposes `zwlr_screencopy_manager_v1` without restriction, a Flatpak app with `--socket=wayland` can capture the screen without any portal interaction. GNOME and KDE compositors (Mutter and KWin) restrict these protocols more aggressively, but the permission model is ultimately compositor-dependent.

Flatpak 1.16 added `--socket=inherit-wayland-socket` as an alternative that passes the parent process's existing Wayland socket rather than `wayland-0`, enabling better integration with compositors that implement `wp_security_context_v1`.

### 6.3 wp_security_context_v1 Integration

Flatpak has merged support for `wp_security_context_v1` ([wayland-protocols staging](https://wayland.app/protocols/security-context-v1), [Phoronix coverage of Flatpak integration](https://www.phoronix.com/news/Flatpak-Wayland-Security-Ctx)). The protocol allows Flatpak's launcher to create a new Wayland socket for the sandboxed app and attach security metadata to all connections coming from inside the sandbox:

```c
/* Compositor side: create a security context for the sandbox listener */
struct wp_security_context_v1 *ctx =
    wp_security_context_manager_v1_create_listener(ctx_manager,
        listen_fd, close_fd);

wp_security_context_v1_set_sandbox_engine(ctx, "org.flatpak");
wp_security_context_v1_set_app_id(ctx, "org.videolan.VLC");
wp_security_context_v1_set_instance_id(ctx, "12345");
wp_security_context_v1_commit(ctx);
```

The compositor now knows that all connections accepted on `listen_fd` originate from within the `org.videolan.VLC` Flatpak sandbox. The compositor can use this metadata to:

- Restrict `zwlr_screencopy_manager_v1` to connections without a security context (i.e., only the compositor's own portal backend)
- Enforce `xdg_activation_v1` token requirements per app_id
- Apply per-application policies to privileged protocols

Critically, the protocol forbids nested security contexts — a sandboxed app cannot create its own security context to escalate privileges:

```
/* From wp_security_context_v1 protocol spec */
/* Compositors should reject nesting:
 * "nested security contexts are dangerous because they can
 *  potentially allow privilege escalation of a sandboxed client." */
error: nested = 1
```
[Source: wayland.app/protocols/security-context-v1](https://wayland.app/protocols/security-context-v1)

### 6.4 Wayland Privileged Protocols in Flatpak

Applications that legitimately need GPU access for compute or rendering should use `--device=dri` (grants `/dev/dri/renderD*` access) rather than `--device=all` (grants all devices including `/dev/input/*`). For applications that have no GPU rendering requirement, omit device access entirely:

```bash
# Grant only render node access (no display master, no /dev/input)
flatpak override --user --device=dri com.example.GpuApp

# Revoke all device access (display access remains via Wayland socket)
flatpak override --user --nodevice=all com.example.App
```

### 6.5 zypak: Chrome Sandbox in Flatpak

Chromium-based browsers in Flatpak use `zypak` as a replacement for `chrome-sandbox`. Zypak uses Flatpak's own bubblewrap machinery to create process sandboxes for renderer processes, rather than requiring a setuid binary. This keeps the renderer isolation chain entirely within the Flatpak security model, avoiding the need for elevated privileges at the sandbox boundary.

---

## 7. Virtual Keyboard and Input Injection Security

### 7.1 The zwp_virtual_keyboard_v1 Threat

The `zwp_virtual_keyboard_manager_v1` protocol is used legitimately by:

- **Input method engines**: `ibus-daemon`, `fcitx5` — convert CJK/complex script input to key events
- **Remote desktop servers**: `wayvnc`, `wlroots-based RDP` — inject input from remote clients
- **Accessibility tools**: on-screen keyboards, switch access software
- **Testing frameworks**: automated UI testing tools

But the protocol provides a raw capability: any client bound to `zwp_virtual_keyboard_manager_v1` can inject arbitrary key events into whichever surface holds keyboard focus. A malicious app that gains this binding can phish credentials by waiting for a password manager to have focus and then injecting "reveal password" keyboard shortcuts, or can exfiltrate data by simulating clipboard paste into a controlled surface.

### 7.2 Compositor-Level Enforcement

The protocol XML explicitly requires compositor enforcement:

```xml
<!-- From virtual-keyboard-unstable-v1.xml -->
<enum name="error">
  <entry name="unauthorized" value="0"
    summary="the compositor denied access to create a virtual keyboard"/>
</enum>
<request name="create_virtual_keyboard">
  <description summary="create a new virtual keyboard">
    Creates a new virtual keyboard associated to a seat.
    If the compositor enables a keyboard to perform arbitrary actions,
    it should present an error when an untrusted client requests a new keyboard.
  </description>
  <arg name="seat" type="object" interface="wl_seat"/>
  <arg name="id" type="new_id" interface="zwp_virtual_keyboard_v1"/>
</request>
```
[Source: wlroots protocol/virtual-keyboard-unstable-v1.xml](https://github.com/swaywm/wlroots/blob/master/protocol/virtual-keyboard-unstable-v1.xml)

Compositors implementing this protocol should:

1. Check if the connecting client is a known, trusted process (input method engine, remote desktop server)
2. Use `wp_security_context_v1` metadata if available to identify Flatpak-sandboxed apps and deny them access
3. Potentially prompt the user for approval before allowing a new virtual keyboard

### 7.3 Input Method vs Virtual Keyboard

`zwp_input_method_v2` is a more structured alternative for input method engines. It establishes a handshake with the compositor's text input infrastructure (`zwp_text_input_v3`), limiting the scope of key injection to active text input fields and requiring the compositor to explicitly route events. This is preferable to raw virtual keyboard access for IME use cases.

### 7.4 RemoteDesktop Portal as the Controlled Path

For remote desktop applications, the correct approach is the `org.freedesktop.portal.RemoteDesktop` portal ([documentation](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.RemoteDesktop.html)). The portal:

1. Presents a user consent dialog before enabling remote control
2. Limits input injection to the PipeWire session that the user approved
3. Allows the user to terminate the remote desktop session at any time
4. Logs the access for auditability

Applications using the RemoteDesktop portal do not need direct access to `zwp_virtual_keyboard_manager_v1` — the portal backend handles the compositor interaction.

---

## 8. DRM Render Node Access Control

### 8.1 Render Node vs Master Node

DRM exposes two node types in `/dev/dri/`:

- **Card nodes** (`/dev/dri/card0`): Full DRM access including modesetting (`DRM_IOCTL_SET_MASTER`). Required for display configuration. Mode: `crw-rw----` (root:video group). Requires KMS master status.
- **Render nodes** (`/dev/dri/renderD128`): GPU compute and rendering without display access. Mode: `crw-rw----` (root:render group). No KMS master required.

The render/master split is a deliberate security boundary. Applications that need GPU acceleration for rendering (3D, video decode, compute) only need render node access. Only the Wayland compositor (and VT console) needs KMS master.

### 8.2 systemd-logind and udev ACLs

`systemd-logind` uses `udev`'s `uaccess` tagging mechanism to dynamically grant the active session user access to GPU devices without requiring group membership:

```bash
# udev rule (typically in /lib/udev/rules.d/70-uaccess.rules)
SUBSYSTEM=="drm", KERNEL=="renderD*", TAG+="uaccess"
```

When a user logs in, `logind` applies a POSIX ACL granting that user `rw` access to all `uaccess`-tagged devices in their seat. When the session ends or the VT switches, the ACL is removed. This means:

- The `render` group is a fallback for non-session users (system daemons, SSH sessions)
- Interactive users get dynamic access without needing group membership
- Multi-seat setups correctly assign devices to the active seat

```bash
# Verify ACL on render node for active session
getfacl /dev/dri/renderD128
# user:jreuben1:rw-  (if active session)
```

### 8.3 DRM Master and VT Session Security

`DRM_IOCTL_SET_MASTER` requires the calling process to hold the active VT or have `CAP_SYS_ADMIN`. `logind`'s `TakeControl` D-Bus method is how Wayland compositors legally acquire DRM master:

```c
/* Compositor acquires DRM master via logind, not direct ioctl */
sd_bus_call_method(bus,
    "org.freedesktop.login1",
    session_path,
    "org.freedesktop.login1.Session",
    "TakeControl",
    &error, &reply, "b", false);
```
[Source: wlroots DRM backend](https://gitlab.freedesktop.org/wlroots/wlroots/-/tree/master/backend/drm)

When VT focus switches, `logind` sends `PauseDevice` to the previous compositor and `ResumeDevice` to the new one, atomically transferring DRM master between sessions. A compositor that loses DRM master via VT switch cannot perform KMS commits until it regains master.

### 8.4 Container and Namespace GPU Access

Containers (Docker, Podman) that need GPU acceleration should use render nodes only:

```bash
# Correct: grant only render node access
docker run --device=/dev/dri/renderD128 ...

# Risky: grant all DRM access including card nodes
docker run --device=/dev/dri/ ...
```

With just `/dev/dri/renderD128`, a container can perform GPU rendering and compute but cannot interact with display management. The `render` group GID must be mapped into the container's user namespace:

```bash
docker run --device=/dev/dri/renderD128 \
           --group-add $(getent group render | cut -d: -f3) \
           my-gpu-image
```

### 8.5 O_CLOEXEC and FD Hygiene

Render node file descriptors should always be opened with `O_CLOEXEC` to prevent inheritance across `exec`:

```c
int fd = open("/dev/dri/renderD128", O_RDWR | O_CLOEXEC);
```

DMA-BUF FDs created from render node operations should similarly be created with `O_CLOEXEC` wherever the kernel API supports it (e.g., `DMA_BUF_IOCTL_EXPORT_SYNC_FILE` with `O_CLOEXEC`). In a compositor context, FD hygiene prevents renderer processes from inheriting display-management FDs across the compositor's process spawning.

---

## 9. Clipboard and Drag-and-Drop Security

### 9.1 The wl_data_device Focus Requirement

Wayland's clipboard model (`wl_data_device`) requires the receiving client to have keyboard focus at the moment a selection is offered:

```
/* From wayland.xml */
/* wl_data_device.selection event:
 * "The client with keyboard focus will receive selection events."
 * "This event is also sent to a client immediately before it receives
 *  keyboard focus." */
```

This means a background application — one whose surface does not have keyboard focus — never receives `wl_data_device.selection` events. It cannot learn that a copy was performed, cannot access the clipboard contents, and cannot register for clipboard change notifications. This is a fundamental improvement over X11's clipboard model, where any background application can receive `SelectionNotify` events and read clipboard content.

The mechanism works as follows:

1. Application A copies text: it calls `wl_data_source.offer(mime_type)` and `wl_data_device.set_selection(source, serial)`.
2. The compositor stores the source association with the seat's keyboard focus.
3. When application B gains keyboard focus, the compositor sends `wl_data_device.selection` with a `wl_data_offer` representing A's data source.
4. B calls `wl_data_offer.receive(mime_type, pipe_fd)` to actually transfer the data.

A background application never reaches step 3 and therefore never learns the clipboard content.

### 9.2 Primary Selection: zwp_primary_selection_device_v1

The primary selection (middle-click paste, X11 `PRIMARY` selection equivalent) follows the same focus-based model via `zwp_primary_selection_device_v1`. A client only receives primary selection offers when it gains keyboard focus, and can only set the primary selection when it has a valid input event serial.

### 9.3 Clipboard Manager Protocols: ext-data-control-v1

Clipboard managers — applications that retain clipboard history after the source app exits — require a privileged protocol. The production protocol is `ext-data-control-v1` ([Wayland Explorer](https://wayland.app/protocols/ext-data-control-v1)), which supersedes the deprecated `zwlr-data-control-v1`.

The `ext_data_control_manager_v1` interface documentation states explicitly: *"a privileged client"*. When a client binds `ext_data_control_manager_v1`, it receives `data_offer` events for every clipboard change on the seat — bypassing the focus requirement. This is intentional for clipboard managers, but it means **the compositor must restrict which clients can bind this interface**.

```c
/* ext-data-control-v1: clipboard manager receives all selections */
/* Interface grants: set_selection(), primary_selection(), and receives
 * data_offer() events for all clipboard changes — no focus required */
static const struct ext_data_control_manager_v1_interface manager_impl = {
    .create_data_source = handle_create_data_source,
    .get_data_device    = handle_get_data_device,
    .destroy            = handle_destroy,
};
```

If a malicious application can bind `ext_data_control_manager_v1`, it has full clipboard monitoring capability — functionally equivalent to an X11 clipboard logger. Compositors should restrict this binding to known clipboard manager processes, ideally identified via `wp_security_context_v1`.

### 9.4 DnD: wl_data_device Drag-and-Drop

Drag-and-drop follows a similar model: only the surface currently under the pointer during a DnD operation receives `wl_data_device.enter`, `wl_data_device.motion`, and `wl_data_device.drop` events. A background application cannot snoop on DnD operations in progress. The compositor routes events based on pointer position, which it fully controls.

MIME type filtering provides an additional layer: a receiving application declares which MIME types it accepts via `wl_data_offer.accept`, and the compositor/source uses this to determine if a drop is valid. A malicious application cannot register for all MIME types on a drop that wasn't directed at it.

---

## 10. Practical Hardening Recommendations

### 10.1 Compositor Level

**Restrict screencopy to portal backends:**

```c
/* Compositor: only advertise zwlr_screencopy_manager_v1 to the portal backend */
/* Identify portal backend via wp_security_context_v1 metadata or SO_PEERCRED */
static void
bind_screencopy_manager(struct wl_client *client, void *data,
                        uint32_t version, uint32_t id)
{
    struct wl_client_security_context *ctx = get_security_context(client);
    if (ctx && !is_portal_backend(ctx)) {
        wl_client_post_implementation_error(client,
            "screencopy not available to sandboxed clients");
        return;
    }
    /* Proceed with binding */
}
```

**Restrict virtual keyboard to trusted processes:**

Compositors should check whether a client requesting `zwp_virtual_keyboard_manager_v1` is a known IME process or has `wp_security_context_v1` metadata indicating it is sandboxed (and therefore should be denied).

**Implement wp_security_context_v1:**

Compositors should implement `wp_security_context_manager_v1` to receive Flatpak sandbox identity metadata. This metadata should be consulted before granting access to any privileged protocol (`zwlr_screencopy_manager_v1`, `zwp_virtual_keyboard_manager_v1`, `ext_data_control_manager_v1`, `zwlr_export_dmabuf_manager_v1`).

```bash
# Verify compositor supports security context protocol
wayland-info | grep security_context
# Expected: wp_security_context_manager_v1 (version 1)
```

**Restrict DRM lease creation:**

`wp_drm_lease_device_v1` allows clients to lease DRM connectors for direct-mode VR rendering. Only VR runtime processes (Monado) should be permitted to create leases. Compositors should identify the VR runtime via `SO_PEERCRED` verification.

### 10.2 XDG_RUNTIME_DIR Hardening

```bash
# Verify runtime dir has correct permissions
stat -c "%a" $XDG_RUNTIME_DIR
# Expected: 700

# If incorrect, fix (systemd-logind normally sets this correctly)
chmod 700 $XDG_RUNTIME_DIR

# Verify socket permissions
stat -c "%a" $XDG_RUNTIME_DIR/wayland-0
# Expected: 777 (socket) — permissions on the socket itself
#           are controlled by the directory's mode
```

For systemd-managed sessions, `systemd-logind` creates `XDG_RUNTIME_DIR` with mode 0700 automatically. Non-systemd systems using `pam_rundir` or manual configuration must be audited.

### 10.3 Application Level (Flatpak)

Use the principle of least privilege for Wayland socket and device access:

```bash
# Check current permissions for an app
flatpak info --show-permissions com.example.App

# Restrict device access to render nodes only
flatpak override --user --nodevice=all com.example.App
flatpak override --user --device=dri com.example.App

# Prevent IPC namespace sharing (stops shared-memory snooping)
flatpak override --user --unshare=ipc com.example.App

# Use portals instead of direct Wayland protocols
# Verify app uses portal D-Bus interfaces, not direct zwlr_ protocols
strace -e trace=sendmsg -p <app_pid> 2>&1 | grep wayland
```

Use xdg-desktop-portal for screen capture rather than direct screencopy:

```c
/* Application should call portal, not zwlr_screencopy directly */
/* D-Bus call to org.freedesktop.portal.ScreenCast */
GDBusConnection *bus = g_bus_get_sync(G_BUS_TYPE_SESSION, NULL, NULL);
GVariant *result = g_dbus_connection_call_sync(bus,
    "org.freedesktop.portal.Desktop",
    "/org/freedesktop/portal/desktop",
    "org.freedesktop.portal.ScreenCast",
    "CreateSession",
    g_variant_new("(a{sv})", options),
    G_VARIANT_TYPE("(o)"),
    G_DBUS_CALL_FLAGS_NONE, -1, NULL, NULL);
```

### 10.4 System Level

```bash
# SELinux: deny Wayland socket access to specific domains
# (policy depends on distribution, example for custom domain)
# allow unconfined_t user_tmpfs_t:sock_file { read write };
# deny myapp_t user_tmpfs_t:sock_file { read write };

# Verify logind is managing GPU device ACLs
loginctl show-session $(loginctl | grep $(whoami) | awk '{print $1}') \
  -p Remote,State,Active

# List device ACLs granted by logind for current session
for dev in /dev/dri/*; do
    echo "=== $dev ==="; getfacl "$dev" 2>/dev/null
done

# Restrict access to wayland socket in container environments
# Pass only the specific socket, not the entire runtime dir
# docker run -v $XDG_RUNTIME_DIR/wayland-0:/tmp/wayland-0 ...
```

### 10.5 Monitoring and Auditing

```bash
# Monitor which processes are connecting to the Wayland socket
inotifywait -m $XDG_RUNTIME_DIR -e OPEN 2>/dev/null | grep wayland

# Check which apps have screen capture permissions in portal store
flatpak permission-list screencast

# Identify processes using the virtual keyboard protocol
# (compositor-dependent logging required)
journalctl --user -u compositor.service | grep "virtual keyboard"

# Audit DMA-BUF FD passing in compositor logs
# wlroots-based compositors can enable debug logging:
WLROOTS_LOG_LEVEL=debug sway 2>&1 | grep -i dmabuf
```

### 10.6 Future Protocol: wp_security_context_v1 Adoption Roadmap

`wp_security_context_v1` is a staging protocol in wayland-protocols (as of version 1.48, April 2026) and is already implemented in Flatpak and KDE Plasma 6.3+. The expected hardening roadmap:

1. **Flatpak** creates a dedicated Wayland socket for each sandboxed app with security context metadata attached.
2. **Compositor** (Mutter, KWin, Sway) receives connection requests on that socket and tags all resources from those connections with `{sandbox_engine, app_id, instance_id}`.
3. Compositor's global advertisment code checks the security context tag before advertising privileged protocols.
4. **Portal backends** connect via the main compositor socket (no security context) and are therefore trusted to receive screencopy, DRM lease, and other privileged protocol access.

This chain closes the gap where Flatpak apps with `--socket=wayland` could bypass portal-mediated permission checks by directly binding privileged extension protocols.

---

## Roadmap

### Near-term (6–12 months)

- **`wp_security_context_v1` broader sandbox adoption**: Firejail and bubblejail are actively tracking implementation of `wp_security_context_v1` to restrict sandboxed clients from accessing privileged protocols such as `zwlr_screencopy_manager_v1` without going through the portal. Firejail issue [#5883](https://github.com/netblue30/firejail/issues/5883) and bubblejail issue [#118](https://github.com/igo95862/bubblejail/issues/118) track this work; the protocol shipped in wayland-protocols 1.32 but adoption by non-Flatpak sandboxes is still in progress.
- **KDE Plasma 6.8 X11 session removal (October 2026)**: KDE Plasma 6.8 is planned to drop the X11 login session entirely, with XWayland retained only for running legacy X11 applications inside the Wayland session. [Source](https://9to5linux.com/kde-plasma-6-8-desktop-environment-to-drop-the-x11-session-and-go-wayland-only) This eliminates the largest remaining attack surface from X11's broken isolation model for the majority of KDE users; 95% of Plasma 6.6 users already run Wayland. [Source](https://www.phoronix.com/news/KDE-Plasma-Wayland-Ex-X11)
- **Clipboard access via input-capture portal**: Work is ongoing to add clipboard support to the `org.freedesktop.portal.InputCapture` interface as an alternative to the closed-as-not-planned dedicated clipboard portal ([xdg-desktop-portal#743](https://github.com/flatpak/xdg-desktop-portal/issues/743)), allowing clipboard synchronisation for remote-desktop scenarios under portal mediation rather than raw Wayland protocol access. Note: needs verification of current merge status.
- **xdg-desktop-portal-generic availability**: A standalone generic portal backend ([xdg-desktop-portal-generic](https://github.com/lamco-admin/xdg-desktop-portal-generic)) implementing ScreenCast v6 is maturing to serve compositors that lack a native portal backend, reducing the gap where those compositors expose privileged screencopy without user consent. [Source](https://lamco.ai/open-source/xdg-desktop-portal-generic/)
- **Hyprland and other independent compositors adopting security-context ACLs**: Hyprland tracks screencopy ACL enforcement in issue [#4432](https://github.com/hyprwm/Hyprland/issues/4432); the expectation is that `wp_security_context_v1`-aware gatekeeping for `zwlr_screencopy_manager_v1` and `ext-image-copy-capture-v1` will land in major independent compositors within the next release cycle. Note: needs verification of exact merge timeline.

### Medium-term (1–3 years)

- **Formal clipboard isolation protocol in wayland-protocols**: The current `wl_data_device` clipboard mechanism has no cross-client isolation; clipboard managers require privileged access (`zwlr_data_control_manager_v1` on wlroots) with no user-visible permission gate. A portal-mediated clipboard protocol analogous to `org.freedesktop.portal.ScreenCast` is a long-discussed design goal; a formal draft in wayland-protocols staging is expected once the input-capture portal clipboard work establishes design precedent. Note: needs verification — no merged RFC exists as of June 2026.
- **XWayland privilege reduction**: As compositors drop native X11 sessions, XWayland itself becomes the residual X11 attack surface. Ongoing work to run XWayland in a more restricted security context — giving it only the DRM render node and the compositor's Wayland socket rather than full GPU device access — is tracked in the Xorg and Mutter/KWin issue trackers. [Source](https://www.heise.de/en/news/KDE-Desktop-says-goodbye-to-X11-mode-and-fully-commits-to-Wayland-11094400.html) Note: specific patchset references need verification.
- **Snap sandbox integration with `wp_security_context_v1`**: Canonical's Snap confinement system has not yet adopted `wp_security_context_v1` (as of June 2026); medium-term roadmap discussion in the Snap and Ubuntu security teams is expected to follow Flatpak's implementation as the protocol stabilises in more compositors. Note: needs verification of Canonical's official roadmap.
- **Compositor-enforced DMA-BUF export policy**: Currently, any client that successfully imports a DMA-BUF handle can retain a reference and re-export it without compositor knowledge. A design discussion for export-tracking or scoped DMA-BUF handles at the compositor level is an open research area in the freedesktop security community. Note: no merged proposal exists as of June 2026; this is a known gap flagged in threat-model discussions.
- **KDE Plasma X11 end-of-life support window closes (early 2027)**: KDE commits to supporting the X11 session in Plasma 6.7 until early 2027, after which enterprise users are redirected to LTS distributions such as AlmaLinux 9 (support until 2032). [Source](https://blogs.kde.org/2025/11/26/going-all-in-on-a-wayland-future/) This represents the effective end of mainstream upstream X11 security maintenance.

### Long-term

- **Capability-based Wayland object model**: The current Wayland object model uses opaque numeric IDs within a per-client namespace, which prevents cross-client object forgery. A longer-term architectural direction sometimes discussed is making capabilities explicit and inspectable — allowing compositors to reason about capability delegation (e.g., a trusted portal delegating a scoped screencopy capability to an untrusted consumer) at the protocol level rather than through out-of-band D-Bus policy. Note: speculative; no protocol draft exists.
- **Mandatory access control integration at the compositor layer**: LSM (Linux Security Module) hooks do not currently extend into the Wayland protocol layer; a compositor receives no LSM-mediated policy about which clients should be permitted to bind which globals. Long-term integration of SELinux or AppArmor policy evaluation into compositor global advertisement — analogous to how LSM mediates `open(2)` — is a direction discussed in security-focused compositor communities. Note: speculative; no kernel or compositor RFC as of June 2026.
- **Unified sandbox identity across the Linux desktop**: `wp_security_context_v1` establishes `{sandbox_engine, app_id, instance_id}` as a compositor-visible sandbox identity. Long-term, convergence of this identity with systemd's `app.slice` unit structure, `xdg-desktop-portal` permission store, and PipeWire session metadata into a single kernel-attested identity would allow coherent, auditable capability grants across all privileged desktop interfaces. Note: speculative architectural direction discussed in freedesktop circles; no cross-project RFC as of June 2026.
- **Post-quantum cryptography for Wayland socket authentication**: The current Wayland socket model relies on filesystem permissions (`$XDG_RUNTIME_DIR`, mode 0700) for authentication — no cryptographic handshake occurs. As quantum-resistant cryptography becomes standard, there is long-term interest in extending Wayland's connection setup with an authenticated key exchange, particularly for use cases where the compositor runs remotely (e.g., cloud gaming, Wayland-over-network via Waynergy or similar). Note: highly speculative; no concrete proposal exists.

---

## 11. Integrations

This chapter connects to the following chapters in the book:

**Chapter 20 (Wayland Protocol Fundamentals)**: The core Wayland object model and isolation guarantees described here (per-client resource IDs, `wl_keyboard` event routing, `wl_shm` pool isolation) are derived from the fundamental Wayland design. Section 10 of Chapter 20 covers the security model at the protocol level; this chapter extends that analysis with threat modeling, attack surfaces, and hardening practice.

**Chapter 21 (Building Compositors with wlroots)**: wlroots implements both secure protocols (core Wayland surfaces, seat management) and potentially insecure ones (`zwlr_screencopy_manager_v1`, `zwp_virtual_keyboard_manager_v1`). Chapter 21 explains how compositors are built on wlroots; this chapter explains the security decisions those compositor authors must make about which protocols to expose to which clients.

**Chapter 22 (Production Compositors: Mutter and KWin)**: GNOME's Mutter and KDE's KWin implement compositor-level ACLs for screencopy and input injection that are significantly more restrictive than bare wlroots. Chapter 22 covers their architecture; this chapter explains the security policies they implement — particularly their integration with `xdg-desktop-portal-gnome` and `xdg-desktop-portal-kde` as trusted screencopy backends.

**Chapter 38 (PipeWire)**: Screen casting goes through PipeWire as the transport layer between the compositor's capture output and the application's consumer. The portal (covered in Section 5 of this chapter) creates PipeWire nodes and streams; the application consumes PipeWire streams via `libpipewire`. Chapter 38 covers PipeWire's graph model and session management.

**Chapter 111 (Flatpak)**: Flatpak is the primary production consumer of the portal permission model described in Section 5 and 6. Chapter 111 covers Flatpak's sandbox model in depth; this chapter explains how that sandbox model intersects with the Wayland compositor's access control. The `wp_security_context_v1` integration between Flatpak and compositors is the critical security mechanism connecting these chapters.

**Chapter 123 (Screen Capture and Remote Desktop)**: Screen capture via `zwlr_screencopy_manager_v1` and `ext-image-copy-capture-v1`, and the `org.freedesktop.portal.ScreenCast` and `org.freedesktop.portal.RemoteDesktop` portals, are the primary sensitive operations requiring the portal mediation described here. Chapter 123 covers the technical pipeline; this chapter covers the permission model that governs when that pipeline is accessible.

**Chapter 130 (Protocol Development)**: New Wayland protocols must incorporate security considerations from design. The lessons from `zwlr_screencopy_manager_v1`'s lack of ACLs and `zwp_virtual_keyboard_manager_v1`'s pure-compositor enforcement model demonstrate that security cannot be an afterthought. Chapter 130 discusses the wayland-protocols review process; this chapter provides the security analysis that should inform that process.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
