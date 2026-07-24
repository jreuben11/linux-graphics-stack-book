# Chapter 130: Wayland Protocol Extension Development

**Audiences**: Wayland compositor developers, protocol designers, and systems engineers who want to extend the Wayland ecosystem with new capabilities. This chapter assumes familiarity with the Wayland core protocol and C programming. Readers who have worked through Ch20 (Wayland Protocol Fundamentals) will find this chapter a natural next step: where Ch20 explains how to *use* existing protocols, this chapter explains how to *design and implement* new ones.

---

## Table of Contents

1. [Introduction: Wayland's Extensibility Model](#1-introduction-waylands-extensibility-model)
   - [1.1 What is a Wayland Protocol Extension?](#11-what-is-a-wayland-protocol-extension)
   - [1.2 What is wayland-scanner?](#12-what-is-wayland-scanner)
   - [1.3 What is wayland-protocols?](#13-what-is-wayland-protocols)
2. [Wayland Wire Protocol Fundamentals](#2-wayland-wire-protocol-fundamentals)
3. [Protocol XML Grammar](#3-protocol-xml-grammar)
4. [wayland-scanner: Generating C Bindings](#4-wayland-scanner-generating-c-bindings)
5. [Implementing a Server-Side Extension](#5-implementing-a-server-side-extension)
6. [Implementing a Client-Side Extension](#6-implementing-a-client-side-extension)
7. [Protocol Governance: wayland-protocols](#7-protocol-governance-wayland-protocols)
8. [Protocol Design Patterns and Anti-Patterns](#8-protocol-design-patterns-and-anti-patterns)
9. [Testing Protocol Extensions](#9-testing-protocol-extensions)
10. [Integrations](#10-integrations)

---

## 1. Introduction: Wayland's Extensibility Model

The Wayland core protocol (`wayland.xml`) is deliberately minimal. It defines only the lowest-common-denominator objects that every compositor and client must agree on: the display singleton (`wl_display`), the global registry (`wl_registry`), compositor (`wl_compositor`), surfaces (`wl_surface`), and a handful of input and output objects. Everything else—desktop shell integration, XDG toplevel windows, DRM buffer import, explicit GPU synchronisation, presentation feedback, screen capture, HDR signalling—is implemented as a *protocol extension*.

This design is not an accident. Protocol extensibility allows the ecosystem to evolve without breaking existing compositors or clients. A compositor that does not understand a new extension simply does not advertise it in the registry; a client that does not find the extension gracefully degrades. The separation of concerns also allows different vendors to innovate at their own pace while contributing their best ideas back to the shared `wayland-protocols` repository when they are ready.

This chapter covers the full lifecycle of a Wayland protocol extension:

- The wire format that all extensions share.
- The XML grammar used to describe interface semantics.
- `wayland-scanner`, the code-generation tool that turns XML into C.
- Step-by-step implementation of a server-side global and request handler.
- Step-by-step implementation of a client-side proxy and event listener.
- The `wayland-protocols` governance model: how new protocols enter staging and graduate to stable.
- Design patterns and anti-patterns distilled from the existing corpus.
- Testing tools and conformance infrastructure.

By the end, you will be able to write a new protocol from scratch, wire it into a wlroots or Weston compositor, ship a reference client, and submit it for inclusion in `wayland-protocols`.

### 1.1 What is a Wayland Protocol Extension?

A Wayland protocol extension is a formally specified, versioned contract between a compositor and one or more clients that defines new object types, requests, and events beyond those in the core `wayland.xml` protocol. Extensions are described in XML files that share a common grammar, compiled to C bindings by `wayland-scanner`, and loaded at runtime through the `wl_registry` global advertisement mechanism. Because the core protocol deliberately limits itself to the lowest-common-denominator objects—displays, surfaces, inputs, and outputs—virtually all real desktop functionality is implemented as extensions: XDG shell windows, DMA-BUF buffer import, explicit GPU synchronisation, screen capture, HDR metadata signalling, and many others.

A compositor that implements an extension registers a global for it; clients discover that global by listening to `wl_registry.global` events. If a client does not find a particular global, it gracefully degrades to a fallback code path or reports the feature unavailable. This negotiation model allows new capabilities to be deployed incrementally without breaking existing implementations. Each extension lives in its own namespace: the `wl_` prefix is reserved for the core Wayland protocol, `xdg_` for the XDG shell family, and `ext_`, `wp_`, `zwp_`, or compositor-specific prefixes for others. Extensions serve throughout this chapter as the vehicle for understanding the wire protocol, the XML grammar, the code-generation pipeline, and the governance model that allows experimental protocols to graduate to stable.

### 1.2 What is wayland-scanner?

`wayland-scanner` is the official code-generation tool that transforms a Wayland protocol XML file into C source code. It is distributed as part of the `wayland` package and is typically invoked as a build step—in Meson via `wayland_scanner.process()` or in Autotools via a `WAYLAND_SCANNER` substitution variable. Given a protocol XML file, `wayland-scanner` produces two outputs: a header (`-code=client-header` or `server-header`) that declares generated types, function prototypes, and opcode constants, and a C implementation file (`-code=private-code` or `public-code`) that defines the `wl_interface` descriptors and dispatch tables that libwayland uses to route messages to the correct handler functions.

The generated code is purely mechanical; it encodes the argument types, opcodes, and version constraints described in the XML but adds no policy or logic of its own. Protocol authors never write the dispatch infrastructure by hand: `wayland-scanner` ensures that the C types match the XML exactly, that deprecated entries are flagged, and that version guards are consistently applied. For server-side code, the generated `wl_interface` structure is what `wl_resource_create` and `wl_global_create` reference when binding object implementations to incoming requests. For client-side code, the generated proxy helpers wrap `wl_proxy_marshal_flags` so callers never touch raw opcodes directly. Understanding `wayland-scanner`'s output is essential for debugging protocol dispatch failures and for writing correct version-gated code paths (§4).

### 1.3 What is wayland-protocols?

`wayland-protocols` is the canonical upstream repository for Wayland protocol extensions intended for broad ecosystem adoption rather than remaining compositor-specific. It is hosted at `https://gitlab.freedesktop.org/wayland/wayland-protocols` and maintained under freedesktop.org governance. The repository organises protocols into three stability tiers: **stable** protocols have completed a public review process, are considered API-frozen, and carry a guarantee of backwards compatibility; **staging** protocols are under active development and may change between minor releases; and **unstable** protocols (carrying the older `zwp_` prefix) exist for historical reasons and will not receive new versions.

A protocol enters `wayland-protocols` through a merge-request process: the submitter provides the XML, a reference implementation in at least one compositor and one client, a rationale document, and test coverage. Reviewers evaluate the protocol for correctness, generality, and fit with the existing corpus before it advances from staging to stable. Protocols that remain compositor-specific—features not intended for general use—live in the compositor's own repository and are never submitted upstream. For systems engineers writing extensions, understanding the governance model determines whether to invest in a `wayland-protocols`-quality design (§7) or to ship a private protocol under a compositor-specific prefix. This chapter covers both paths.

---

## 2. Wayland Wire Protocol Fundamentals

### 2.1 Transport

Wayland uses a Unix domain socket for all communication. The socket is found at `$XDG_RUNTIME_DIR/wayland-0` (or as overridden by `$WAYLAND_DISPLAY`). The socket is `AF_UNIX / SOCK_STREAM` and carries both a byte stream of 32-bit words and out-of-band file descriptors via `SCM_RIGHTS` ancillary messages.

Because the kernel delivers ancillary data atomically with the data bytes that accompany it, Wayland can transfer file descriptors (DMA-BUF fds, shared-memory fds, keymaps, synchronisation fences) as first-class protocol arguments without a second syscall.

### 2.2 Message Layout

Every message—whether a client request or a server event—follows the same two-word header:

```
[object_id : u32][size_opcode : u32][args...]
```

- **Word 0 — object\_id**: The 32-bit ID of the object this message is addressed to (for events) or sent from (for requests).
- **Word 1 — size and opcode**: The upper 16 bits encode the total message size *in bytes*, including the 8-byte header. The lower 16 bits encode the opcode: zero-based index into the interface's request list (for requests) or event list (for events).

All values are in the native host byte order (little-endian on x86/ARM). The minimum message size is 8 bytes (header only, no arguments).

[Source: Wayland Protocol — Wire Format](https://wayland.freedesktop.org/docs/book/Protocol.html)

### 2.3 Argument Types

Every argument is padded to a 32-bit boundary. The types defined by the protocol specification are:

| Type | Wire encoding |
|------|---------------|
| `int` | 32-bit signed integer |
| `uint` | 32-bit unsigned integer |
| `fixed` | Signed 24.8 fixed-point (23-bit integer + 8-bit fraction, sign in bit 31) |
| `string` | u32 length (including NUL), UTF-8 bytes, NUL terminator, padded to 4 bytes |
| `object` | 32-bit object ID (0 = null, if `allow-null="true"`) |
| `new_id` | 32-bit object ID; preceded by interface name string + version u32 when untyped |
| `array` | u32 byte count, raw bytes, padded to 4 bytes |
| `fd` | Zero bytes on wire; FD delivered via `SCM_RIGHTS` in parallel |
| `enum` | Encoded as `uint` (or `int` for signed enums, though uncommon) |

[Source: Wayland Protocol Specification — Appendix A](https://wayland.freedesktop.org/docs/html/apa.html)

### 2.3.1 Worked Example: Decoding a `wl_compositor.create_surface` Request

This is one of the first messages a client sends after connecting. Assume the compositor's `wl_compositor` object was bound at ID 3, and the client is allocating ID 4 for the new surface. The message (little-endian, x86) is 12 bytes:

```
Bytes (hex):   03 00 00 00   0C 00 00 00   04 00 00 00
               └──────────┘  └─────┬─────┘  └──────────┘
               object_id=3   size=12,     new surface ID=4
                             opcode=0
                             (create_surface
                              is opcode 0 on
                              wl_compositor)
```

Breaking it down word by word:

- **Word 0** (`0x00000003`): Object ID 3 — the `wl_compositor` resource this request addresses.
- **Word 1** (`0x000C0000`): Upper 16 bits = `0x000C` = 12 — total message length in bytes. Lower 16 bits = `0x0000` = opcode 0 = `create_surface`.
- **Word 2** (`0x00000004`): The single `new_id` argument — client allocates ID 4 for the newly created `wl_surface`.

The server's dispatch function decodes opcode 0 as `create_surface`, extracts ID 4 from the argument, calls `wl_resource_create(client, &wl_surface_interface, version, 4)`, sets an implementation, and the new surface is live. When `WAYLAND_DEBUG=1` is set, libwayland prints this as:

```
[  0.001234] -> wl_compositor@3.create_surface(new id wl_surface@4)
```

### 2.4 Object IDs and Lifetimes

Objects are identified by 32-bit IDs within a connection. The namespace is partitioned:

- **Client-allocated IDs**: `[1, 0xFEFFFFFF]` — client sends a `new_id` argument and immediately uses the ID.
- **Server-allocated IDs**: `[0xFF000000, 0xFFFFFFFF]` — server creates the object and sends its ID in an event.
- **Null**: 0 — used only where `allow-null="true"` is specified.
- **`wl_display`**: Always object ID 1; it exists before the connection is established.

The `wl_display` object (ID 1) is the bootstrap point for every Wayland connection. It exposes two requests: `sync` (creates a `wl_callback` that fires when all prior requests have been processed—the basis for the synchronous roundtrip) and `get_registry` (creates the `wl_registry` from which globals are discovered).

On the server side, each in-flight object is represented by a `wl_resource *`. On the client side, it is represented by a `wl_proxy *`. When a destructor request fires or the client disconnects, `wl_resource_destroy` / `wl_proxy_destroy` reclaim the ID.

### 2.5 Version Negotiation

Every Wayland interface carries a version number. The compositor advertises the maximum version it supports for each global via `wl_registry.global`. The client negotiates the version it wants by passing a version number to `wl_registry_bind`. The negotiated version is `min(server_max, client_requested)` and governs which requests and events are valid for the lifetime of that object.

Version numbers are monotonically increasing and backwards-compatible: a version `N` interface is a strict superset of version `N-1`. The `since` attribute in the XML tags which version introduced each request, event, or enum entry. The `wl_resource_get_version()` and `wl_proxy_get_version()` calls let implementations guard version-gated code paths at runtime.

### 2.6 Event Loop

`libwayland-server` provides `wl_event_loop`, a `poll(2)`-based event loop. The compositor integrates its own file descriptors (DRM, input, GPU timelines) via `wl_event_loop_add_fd`. The Wayland socket itself is one such fd, and `wl_display_flush_clients` pushes queued events over the socket. On the client side, `wl_display_dispatch` reads from the socket and dispatches pending events; `wl_display_flush` drains the outgoing buffer.

---

## 3. Protocol XML Grammar

Every Wayland protocol extension is described by an XML file. The `wayland-scanner` tool (§4) parses this file and generates C source. Understanding the grammar is the foundation of protocol design.

### 3.1 Top-Level Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<protocol name="example_notification_v1">
  <copyright>
    Copyright © 2026 Example Author
    SPDX-License-Identifier: MIT
  </copyright>

  <description summary="Desktop notification protocol">
    The ext_notification_v1 protocol allows clients to request
    desktop notifications from the compositor or notification daemon.
  </description>

  <interface name="ext_notification_manager_v1" version="2">
    ...
  </interface>
</protocol>
```

The `name` attribute on `<protocol>` must be globally unique — by convention it matches the filename (minus `.xml`). Each `<interface>` has a `name` and a `version` (the highest version currently defined).

### 3.2 Requests

Requests flow from client to server. Each request has a zero-based opcode assigned by position in the XML.

```xml
<request name="show" since="1">
  <description summary="show a notification"/>
  <arg name="title"   type="string"                    summary="notification title"/>
  <arg name="body"    type="string"  allow-null="true" summary="notification body text"/>
  <arg name="timeout" type="int"                       summary="timeout in ms, -1 = never"/>
  <arg name="id"      type="new_id"
       interface="ext_notification_v1"                 summary="new notification object"/>
</request>

<request name="destroy" type="destructor" since="1">
  <description summary="destroy the manager"/>
</request>
```

`type="destructor"` tells both `wayland-scanner` and the runtime that sending this request destroys the object on both sides atomically. Omitting a destructor request is a resource-leak anti-pattern (§8).

The `since` attribute defaults to `1` if omitted. A request with `since="2"` must not be sent unless the negotiated version is ≥ 2.

### 3.3 Events

Events flow from server to client. Their opcodes are independent of requests (each list starts at zero).

```xml
<event name="activated" since="1">
  <description summary="notification was activated by the user"/>
  <arg name="notification_id" type="uint" summary="ID assigned by compositor"/>
  <arg name="action"          type="uint" enum="ext_notification_v1.action"
       summary="which action the user chose"/>
</event>

<event name="capability" since="2">
  <description summary="compositor capability flags"/>
  <arg name="flags" type="uint" enum="ext_notification_manager_v1.capability"
       summary="bitmask of supported features"/>
</event>
```

### 3.4 Enumerations

```xml
<enum name="action" since="1">
  <description summary="action taken on a notification"/>
  <entry name="default"  value="0" summary="default action (click)"/>
  <entry name="close"    value="1" summary="explicit close by user"/>
  <entry name="expire"   value="2" summary="notification timed out"/>
</enum>

<enum name="capability" bitfield="true" since="2">
  <entry name="persistence"    value="1" summary="notifications persist"/>
  <entry name="action_buttons" value="2" summary="action buttons supported"/>
</enum>

<enum name="error" since="1">
  <description summary="fatal protocol errors"/>
  <entry name="invalid_title" value="0" summary="title was empty or null"/>
  <entry name="invalid_timeout" value="1" summary="timeout value out of range"/>
</enum>
```

`bitfield="true"` tells tooling that values are OR-able flags rather than an exclusive enumeration.

### 3.5 FD and Array Arguments

```xml
<!-- Pass a shared-memory icon -->
<request name="set_icon" since="2">
  <arg name="fd"     type="fd"    summary="shared memory fd for icon data"/>
  <arg name="width"  type="uint"  summary="icon width in pixels"/>
  <arg name="height" type="uint"  summary="icon height in pixels"/>
  <arg name="stride" type="uint"  summary="bytes per row"/>
  <arg name="format" type="uint"  enum="wl_shm.format" summary="pixel format"/>
</request>
```

FDs are zero bytes on the wire; the kernel passes them out-of-band via `sendmsg`/`SCM_RIGHTS`. This is also how `zwp_linux_dmabuf_v1` passes DMA-BUF file descriptors for GPU buffer sharing.

### 3.6 The linux-dmabuf-unstable-v1 Protocol as a Case Study

The `zwp_linux_dmabuf_v1` protocol ([source](https://github.com/wayland-mirror/wayland-protocols/blob/main/unstable/linux-dmabuf/linux-dmabuf-unstable-v1.xml)) illustrates several advanced patterns:

- **Factory hierarchy**: `zwp_linux_dmabuf_v1` is a global factory that creates `zwp_linux_buffer_params_v1` via the `create_params` request. `zwp_linux_buffer_params_v1` in turn creates a `wl_buffer` via `create` or `create_immed`.
- **FD arguments**: The `add` request takes an `fd` argument for each DMA-BUF plane, plus `uint` arguments for the plane index, offset, stride, and modifier.
- **Immutable-before-commit**: All `add` calls must precede the `create` or `create_immed` call. After `create`, the params object is unusable.
- **Async vs synchronous creation**: `create` is asynchronous—it fires `created` or `failed` events. `create_immed` (since v2) is synchronous but may cause a protocol error if the import fails.
- **Feedback objects** (since v4): `get_default_feedback` and `get_surface_feedback` return `zwp_linux_dmabuf_feedback_v1` objects that stream format-modifier tables via shared memory, avoiding per-format event spam.

---

## 4. wayland-scanner: Generating C Bindings

`wayland-scanner` is the official code-generation tool shipped with `libwayland`. It reads a protocol XML file and emits C source and headers. All Wayland protocol implementations use it; understanding its output is essential for writing extension code.

### 4.1 Command-Line Usage

```bash
# Generate shared header (struct declarations, interface objects)
wayland-scanner client-header < ext-notification-v1.xml > ext-notification-v1-client.h

# Generate server-side header
wayland-scanner server-header < ext-notification-v1.xml > ext-notification-v1-server.h

# Generate glue C source (interface vtables, dispatch tables)
wayland-scanner private-code < ext-notification-v1.xml > ext-notification-v1.c
```

The older spelling `wayland-scanner code` is still valid but `private-code` is preferred for new code: it marks generated symbols with `__attribute__((visibility("hidden")))` so they don't pollute your shared library's ABI.

[Source: The Wayland Protocol Book — wayland-scanner](https://wayland-book.com/libwayland/wayland-scanner.html)

### 4.2 Generated Artifacts

From a protocol with interfaces `ext_notification_manager_v1` and `ext_notification_v1`, `wayland-scanner` generates:

**Client header** (`_client.h`):
```c
/* Proxy struct declarations */
struct ext_notification_manager_v1;
struct ext_notification_v1;

/* Interface version constants */
#define EXT_NOTIFICATION_MANAGER_V1_INTERFACE_VERSION 2

/* Request inlines — call these to send to the server */
static inline struct ext_notification_v1 *
ext_notification_manager_v1_show(
    struct ext_notification_manager_v1 *manager,
    const char *title, const char *body, int32_t timeout);

static inline void
ext_notification_manager_v1_destroy(
    struct ext_notification_manager_v1 *manager);

/* Listener struct for events */
struct ext_notification_v1_listener {
    void (*activated)(void *data,
                      struct ext_notification_v1 *notification,
                      uint32_t notification_id,
                      uint32_t action);
};

static inline int
ext_notification_v1_add_listener(
    struct ext_notification_v1 *notification,
    const struct ext_notification_v1_listener *listener,
    void *data);
```

**Server header** (`_server.h`):
```c
/* Implementation vtable — one function pointer per request */
struct ext_notification_manager_v1_interface {
    void (*show)(struct wl_client *client,
                 struct wl_resource *resource,
                 const char *title,
                 const char *body,
                 int32_t timeout,
                 uint32_t id);
    void (*destroy)(struct wl_client *client,
                    struct wl_resource *resource);
};

/* Event-sending functions — compositor calls these */
void ext_notification_v1_send_activated(
    struct wl_resource *resource,
    uint32_t notification_id,
    uint32_t action);
```

**Glue source** (`.c`):
Contains the `wl_interface` structs (message signature strings, argument type arrays), the dispatch functions that decode wire bytes and call the vtable, and the `wl_message` arrays.

### 4.3 Meson Integration

The modern way to wire `wayland-scanner` into a Meson build uses the `wayland` module:

```python
# meson.build
wl_mod = import('wayland')
wl_proto_dep = dependency('wayland-protocols')

# Find a protocol from the wayland-protocols package
xdg_shell_xml = wl_mod.find_protocol('xdg-shell')

# Generate bindings for a first-party protocol
notif_xml = files('protocol/ext-notification-v1.xml')
notif_generated = wl_mod.scan_xml(notif_xml,
    client: true,
    server: true,
    public: false,
)

# notif_generated[0] = private-code .c
# notif_generated[1] = client-header .h
# notif_generated[2] = server-header .h

compositor_src = files('src/compositor.c')
compositor = executable('compositor',
    compositor_src,
    notif_generated,
    dependencies: [dep_wayland_server, dep_wlroots],
)
```

The `wayland` Meson module (stable since Meson 1.8.0, introduced in 0.62.0) handles the `custom_target` boilerplate automatically. For older Meson versions, the equivalent manual form is:

```python
wayland_scanner = find_program('wayland-scanner')
gen_server_h = custom_target('ext-notification-v1-server.h',
    input:   'protocol/ext-notification-v1.xml',
    output:  'ext-notification-v1-server.h',
    command: [wayland_scanner, 'server-header', '@INPUT@', '@OUTPUT@'],
)
gen_code = custom_target('ext-notification-v1.c',
    input:   'protocol/ext-notification-v1.xml',
    output:  'ext-notification-v1.c',
    command: [wayland_scanner, 'private-code', '@INPUT@', '@OUTPUT@'],
)
```

[Source: Meson Wayland Module](https://mesonbuild.com/Wayland-module.html)

---

## 5. Implementing a Server-Side Extension

We build a concrete server-side implementation of the `ext_notification_manager_v1` protocol defined above. The code targets `libwayland-server` directly; a wlroots-based compositor would wrap these primitives in a `wlr_*` struct, but the underlying calls are identical.

### 5.1 Registering the Global

At compositor startup, register the global:

```c
/* notification.c — server-side implementation */
#include <wayland-server-core.h>
#include "ext-notification-v1-server.h"

struct notif_manager {
    struct wl_global *global;
    struct wl_list   notifications; /* list of active struct notif_object */
};

static void notification_manager_bind(
    struct wl_client *client,
    void             *data,
    uint32_t          version,
    uint32_t          id);

void notif_manager_init(struct wl_display *display,
                        struct notif_manager *mgr)
{
    wl_list_init(&mgr->notifications);
    mgr->global = wl_global_create(
        display,
        &ext_notification_manager_v1_interface,
        2,      /* highest version we support */
        mgr,    /* user data passed to bind callback */
        notification_manager_bind);
}
```

`wl_global_create` [Source: libwayland-server API](https://people.collabora.com/~mvlad/wayland_breathe/dir/specs/server-api.html) advertises the interface in `wl_registry`. Clients that call `wl_registry_bind` trigger `notification_manager_bind`.

### 5.2 The Bind Callback

```c
static void
manager_handle_show(struct wl_client   *client,
                    struct wl_resource *resource,
                    const char         *title,
                    const char         *body,
                    int32_t             timeout_ms,
                    uint32_t            id);

static void
manager_handle_destroy(struct wl_client   *client,
                       struct wl_resource *resource);

static const struct ext_notification_manager_v1_interface
manager_impl = {
    .show    = manager_handle_show,
    .destroy = manager_handle_destroy,
};

static void manager_resource_destroy(struct wl_resource *resource) {
    /* Called when the resource is destroyed — either by the destructor
     * request or because the client disconnected. */
    struct notif_manager *mgr = wl_resource_get_user_data(resource);
    /* cleanup per-client state here if any */
    (void)mgr;
}

static void notification_manager_bind(
    struct wl_client *client,
    void             *data,       /* the notif_manager we passed to wl_global_create */
    uint32_t          version,    /* negotiated version (≤ 2) */
    uint32_t          id)         /* client-allocated new object ID */
{
    struct notif_manager *mgr = data;
    struct wl_resource   *resource;

    resource = wl_resource_create(client,
                                  &ext_notification_manager_v1_interface,
                                  version, id);
    if (!resource) {
        wl_client_post_no_memory(client);
        return;
    }

    wl_resource_set_implementation(resource,
                                   &manager_impl,
                                   mgr,
                                   manager_resource_destroy);

    /* If version ≥ 2, send capability events immediately */
    if (version >= 2) {
        uint32_t caps =
            EXT_NOTIFICATION_MANAGER_V1_CAPABILITY_PERSISTENCE |
            EXT_NOTIFICATION_MANAGER_V1_CAPABILITY_ACTION_BUTTONS;
        /* send_capability is generated by wayland-scanner */
        ext_notification_manager_v1_send_capability(resource, caps);
    }
}
```

### 5.3 Request Handlers

```c
struct notif_object {
    struct wl_resource   *resource;
    struct notif_manager *mgr;
    uint32_t              compositor_id; /* compositor-internal ID */
    struct wl_list        link;
};

static void notif_object_destroy(struct wl_resource *resource) {
    struct notif_object *notif = wl_resource_get_user_data(resource);
    wl_list_remove(&notif->link);
    free(notif);
}

static void
manager_handle_show(struct wl_client   *client,
                    struct wl_resource *resource,
                    const char         *title,
                    const char         *body,
                    int32_t             timeout_ms,
                    uint32_t            id)
{
    struct notif_manager *mgr = wl_resource_get_user_data(resource);
    struct notif_object  *notif;

    /* Validate arguments */
    if (!title || strlen(title) == 0) {
        wl_resource_post_error(resource,
            EXT_NOTIFICATION_MANAGER_V1_ERROR_INVALID_TITLE,
            "title must not be empty");
        return;
    }

    notif = calloc(1, sizeof(*notif));
    if (!notif) {
        wl_client_post_no_memory(client);
        return;
    }

    /* Create the notification child object */
    notif->resource = wl_resource_create(client,
                                         &ext_notification_v1_interface,
                                         wl_resource_get_version(resource),
                                         id);
    if (!notif->resource) {
        free(notif);
        wl_client_post_no_memory(client);
        return;
    }
    notif->mgr = mgr;
    wl_resource_set_implementation(notif->resource,
                                   NULL, /* ext_notification_v1 has no requests */
                                   notif,
                                   notif_object_destroy);
    wl_list_insert(&mgr->notifications, &notif->link);

    /* Hand off to the system notification daemon, get an ID back */
    notif->compositor_id = compositor_show_notification(title, body, timeout_ms);
}
```

### 5.4 Sending Events

When the notification system tells us the user dismissed a notification:

```c
void notif_manager_on_activated(struct notif_manager *mgr,
                                uint32_t compositor_id,
                                uint32_t action)
{
    struct notif_object *notif;
    wl_list_for_each(notif, &mgr->notifications, link) {
        if (notif->compositor_id == compositor_id) {
            /* wayland-scanner generated this function */
            ext_notification_v1_send_activated(notif->resource,
                                               compositor_id,
                                               action);
            return;
        }
    }
}
```

`ext_notification_v1_send_activated` enqueues the event in the client's output buffer. The compositor must call `wl_display_flush_clients(display)` (usually once per frame) to actually transmit it.

### 5.5 Thread Safety

`libwayland-server`'s event loop is single-threaded by design. `wl_display_flush_clients` and all dispatch functions are not thread-safe. There is no `wl_display_post_event` — `libwayland-server` has no built-in inter-thread event queue. The correct pattern for signalling the Wayland event loop from a worker thread is:

```c
/* Worker thread signals the event loop via an eventfd */
int notify_fd = eventfd(0, EFD_CLOEXEC | EFD_NONBLOCK);

/* In event loop setup (main thread): */
wl_event_loop_add_fd(event_loop, notify_fd, WL_EVENT_READABLE,
                     on_worker_notification, state);

/* In worker thread: */
uint64_t one = 1;
write(notify_fd, &one, sizeof(one));  /* wake the event loop */
```

The `on_worker_notification` callback runs on the event loop thread where it is safe to call `ext_notification_v1_send_activated` and other libwayland functions. Never call `wl_resource_destroy` from a worker thread; schedule the destroy via the `eventfd` pattern or a thread-safe queue.

---

## 6. Implementing a Client-Side Extension

### 6.1 Discovering the Global

A Wayland client finds extensions via `wl_registry`:

```c
/* notification-client.c */
#include <wayland-client.h>
#include "ext-notification-v1-client.h"

struct client_state {
    struct wl_display              *display;
    struct wl_registry             *registry;
    struct ext_notification_manager_v1 *notif_mgr;
    uint32_t                        notif_mgr_name; /* registry name */
};

static void
registry_handle_global(void *data, struct wl_registry *registry,
                        uint32_t name, const char *interface,
                        uint32_t version)
{
    struct client_state *state = data;

    if (strcmp(interface, ext_notification_manager_v1_interface.name) == 0) {
        /* Negotiate: we want version 2, accept whatever the server offers up to 2 */
        uint32_t bind_version = version < 2 ? version : 2;
        state->notif_mgr = wl_registry_bind(registry, name,
            &ext_notification_manager_v1_interface, bind_version);
        state->notif_mgr_name = name;
    }
}

static void
registry_handle_global_remove(void *data, struct wl_registry *registry,
                               uint32_t name)
{
    struct client_state *state = data;
    if (name == state->notif_mgr_name) {
        /* Compositor removed the global at runtime */
        ext_notification_manager_v1_destroy(state->notif_mgr);
        state->notif_mgr = NULL;
    }
}

static const struct wl_registry_listener registry_listener = {
    .global        = registry_handle_global,
    .global_remove = registry_handle_global_remove,
};
```

[Source: The Wayland Protocol Book — Binding to Globals](https://wayland-book.com/registry/binding.html)

### 6.2 Setting Up an Event Listener

```c
static void
notif_handle_activated(void *data,
                        struct ext_notification_v1 *notif,
                        uint32_t notification_id,
                        uint32_t action)
{
    printf("Notification %u: action %u\n", notification_id, action);
    /* Destroy the notification proxy now that it's done */
    ext_notification_v1_destroy(notif);
}

static const struct ext_notification_v1_listener notif_listener = {
    .activated = notif_handle_activated,
};
```

### 6.3 Firing Requests

```c
int main(void)
{
    struct client_state state = {0};
    state.display = wl_display_connect(NULL);
    state.registry = wl_display_get_registry(state.display);
    wl_registry_add_listener(state.registry, &registry_listener, &state);

    /* Roundtrip: process all globals */
    wl_display_roundtrip(state.display);

    if (!state.notif_mgr) {
        fprintf(stderr, "Compositor does not support ext_notification_manager_v1\n");
        return 1;
    }

    /* Send a notification request — this creates a child proxy object */
    struct ext_notification_v1 *notif =
        ext_notification_manager_v1_show(state.notif_mgr,
                                         "Build Complete",
                                         "Your project compiled successfully.",
                                         5000 /* 5 seconds */);
    ext_notification_v1_add_listener(notif, &notif_listener, NULL);

    /* Flush outgoing requests then dispatch incoming events */
    wl_display_flush(state.display);
    while (wl_display_dispatch(state.display) != -1)
        ;   /* event loop */

    ext_notification_manager_v1_destroy(state.notif_mgr);
    wl_registry_destroy(state.registry);
    wl_display_disconnect(state.display);
    return 0;
}
```

### 6.4 Multi-Threaded Clients

Simple single-threaded clients call `wl_display_dispatch` which both reads and dispatches. Multi-threaded clients—such as a game engine with a dedicated Wayland thread—need the three-phase pattern to avoid races:

```c
/* Wayland event thread */
while (running) {
    /* 1. Announce intention to read (prevents other threads from reading) */
    while (wl_display_prepare_read(display) != 0)
        wl_display_dispatch_pending(display);

    /* 2. Flush pending requests */
    wl_display_flush(display);

    /* 3. Poll the fd */
    struct pollfd pfd = { .fd  = wl_display_get_fd(display),
                          .events = POLLIN };
    poll(&pfd, 1, -1);

    /* 4. Read events into queues (does NOT dispatch) */
    wl_display_read_events(display);

    /* 5. Dispatch main queue */
    wl_display_dispatch_pending(display);
}
```

If `poll` returns an error or the app wants to cancel the read, call `wl_display_cancel_read(display)` instead of step 4. Failing to call either `wl_display_read_events` or `wl_display_cancel_read` after a successful `prepare_read` will deadlock other threads waiting to read.

---

## 7. Protocol Governance: wayland-protocols

`wayland-protocols` ([https://gitlab.freedesktop.org/wayland/wayland-protocols](https://gitlab.freedesktop.org/wayland/wayland-protocols)) is the canonical repository for cross-compositor Wayland extensions. It is a standardisation body: protocols enter through a governed process and must demonstrate real-world interoperability before reaching stable status.

### 7.1 Repository Layout

The repository has four directories:

| Directory | Status | Usage |
|-----------|--------|-------|
| `stable/` | Final, versioned | `xdg-shell`, `linux-dmabuf-v1` (via `stable/` migration), `presentation-time` |
| `staging/` | Incubating, may change | `drm-lease-v1`, `xdg-toplevel-drag-v1`, `frog-color-management-v1` |
| `unstable/` | Legacy only | `linux-dmabuf-unstable-v1`, `xdg-output-unstable-v1` — no new additions permitted |
| `deprecated/` | Removed protocols | Protocols that were superseded and removed |

New protocols must never be placed in `unstable/`. They enter as experimental (in a MR branch), graduate to `staging/` once they meet the minimum bar, and eventually move to `stable/` after sustained cross-compositor deployment.

### 7.2 Namespacing

The namespace prefix encodes the **domain and ACK bar** required for inclusion — it does *not* directly encode the staging vs. stable distinction. A `wp_` protocol may live in `staging/` (e.g., `wp_drm_lease_device_v1`, `wp_fractional_scale_v1`); it graduates to `stable/` when it meets the implementation bar while keeping the same prefix and interface names. What *does* signal "not yet stable" is the `_vN` suffix in the filename and interface names: a staging protocol is `foo-bar-v1.xml` / `wp_foo_bar_v1`; once promoted to `stable/` those names are frozen — no suffix is dropped, the directory move alone changes the status.

[Source: wayland-protocols README](https://github.com/wayland-mirror/wayland-protocols/blob/main/README.md)

| Prefix | Domain / ACK requirement | Examples |
|--------|--------------------------|---------|
| `wl_` | Core Wayland (wayland.xml only; cannot be extended) | `wl_compositor`, `wl_surface` |
| `xdg_` | Window-management / desktop integration; 3 ACKs required | `xdg_wm_base`, `xdg_toplevel` |
| `wp_` | Generic plumbing across all desktops and OSes; 3 ACKs | `wp_viewporter`, `wp_drm_lease_device_v1` |
| `ext_` | Catch-all extension namespace; 2 ACKs | `ext_idle_notify_v1`, `ext_session_lock_v1` |
| `xx_` | Experimental / pre-staging; no ACK required | `xx_color_management_v4` |
| `zwlr_` | wlroots-specific (wlr-protocols repo, not wayland-protocols) | `zwlr_layer_shell_v1`, `zwlr_screencopy_v1` |
| `zwp_` | Unstable legacy (historical; no new additions) | `zwp_linux_dmabuf_v1`, `zwp_tablet_v2` |

A protocol developed specifically for wlroots-based compositors lives in the separate `wlr-protocols` repository ([https://gitlab.freedesktop.org/wlroots/wlr-protocols](https://gitlab.freedesktop.org/wlroots/wlr-protocols)) under the `zwlr_` namespace. If it later gains cross-compositor adoption (Mutter, KWin, Mir), the authors submit it to `wayland-protocols` under `ext_` or `wp_` as appropriate, with a redesigned interface that incorporates cross-compositor feedback.

### 7.3 The Governance Process

The governance rules are documented in `GOVERNANCE.md` ([source mirror](https://github.com/wayland-mirror/wayland-protocols/blob/main/GOVERNANCE.md)):

**Landing in `staging/`:**

1. Open a merge request with the protocol XML, a `SECURITY.md` section if the protocol has trust implications, and at least one open-source implementation (client *or* server).
2. Wait the mandatory 30-day discussion period.
3. Obtain ACKs from at least **2 member projects** (for `ext_`) or **3 member projects** (for `xdg_` / `wp_`). A single NACK from a member blocks inclusion.
4. For `wp_` and `xdg_`, at least one in-depth review from a member who is not the protocol author.
5. CI must pass: the `check-protocol.sh` script validates XML well-formedness, ensures interface names match the filename, verifies that `destroy` requests exist for all non-singleton interfaces, checks that `since` attributes are monotonically increasing, and lints description elements for completeness. Running it locally before opening your MR saves a round-trip:

```bash
# From the wayland-protocols checkout root:
./ci/check-protocol.sh staging/ext-notification-v1.xml
```

**Promoting to `stable/`:**

1. At least **3 open-source implementations** for `ext_` (1 client + 2 servers, or 2 clients + 1 server); more for `wp_`/`xdg_`.
2. Additional ACKs bringing totals to 3 (for `ext_`) or 4+ (for `wp_`/`xdg_`).
3. Another 30-day comment period.
4. No breaking changes permitted after entry into `stable/`; new functionality must be in a new version.

**Naming convention — staging vs. stable:**

All non-experimental protocols entering `wayland-protocols` carry a major-version suffix in both the filename and interface names: `ext-notification-v1.xml` / `ext_notification_manager_v1`. If a breaking change is needed before stabilisation, a separate `ext-notification-v2.xml` is created with all-new interface names (`ext_notification_manager_v2`), and the v1 file is retired to `deprecated/`. When a protocol is promoted from `staging/` to `stable/`, the interface names and filename are *not changed* — only the directory changes. This is why you see `wp_drm_lease_device_v1` (with `_v1`) in `stable/xdg-shell/`: the version suffix does not mean "unstable", it means "major protocol revision 1".

### 7.4 Case Study: `wp_drm_lease_device_v1`

The DRM lease protocol allows compositors to grant clients direct DRM master access for VR HMD outputs. As of mid-2026, it resides in `staging/drm-lease/` in `wayland-protocols`, illustrating the typical multi-year incubation path for a complex cross-compositor protocol:

1. **2018**: Keith Packard (Valve) proposed DRM leasing in the kernel. [Initial RFC](https://lists.freedesktop.org/archives/wayland-devel/2018-January/036652.html).
2. **2021**: Drew DeVault implemented `zwlr_drm_lease_device_v1` in the wlroots-specific `wlr-protocols` repository to enable VR under sway. [wlroots PR #2929](https://github.com/swaywm/wlroots/pull/2929). This gave the protocol a real-world workout without needing cross-compositor agreement.
3. **2022**: The protocol was redesigned and submitted to `wayland-protocols` as `wp_drm_lease_device_v1` in `staging/`. The redesign incorporated feedback from KWin and Mutter developers — most notably replacing the single-connector model with a multi-connector request/lease flow to support stereo and multi-display VR rigs. The `wp_` namespace was chosen from the start because the protocol targets all compositors (not just wlroots), and Valve sponsored implementations in both KWin and wlroots simultaneously.
4. **2022–present**: The protocol is implemented in wlroots, KWin, and Xwayland. It remains in `staging/` pending the additional ACKs and implementation breadth required for `wp_` promotion to `stable/`. The Phoronix announcement headline ("merged for wayland") referred to entry into `staging/`, not into `stable/`. [Source: DRM Lease Protocol | Wayland Explorer](https://wayland.app/protocols/drm-lease-v1)

The key lesson: the `zwlr_` prototype in `wlr-protocols` allowed rapid iteration without cross-compositor coordination. The formal `wayland-protocols` submission process then drove the interface improvements — particularly the multi-connector lease model — that make the protocol suitable for all compositors. The `staging/` phase is not a waiting room; it is where real-world interoperability testing happens before the interface is frozen.

---

## 8. Protocol Design Patterns and Anti-Patterns

### 8.1 Patterns

**Factory pattern.** A global object creates child objects via requests that include a `new_id` argument. The global should be a thin factory with minimal state; the richness lives in the children.

```xml
<!-- Good: factory creates children -->
<interface name="ext_foo_manager_v1" version="1">
  <request name="create_foo">
    <arg name="id" type="new_id" interface="ext_foo_v1"/>
    <arg name="flags" type="uint" enum="flags"/>
  </request>
</interface>
```

`wl_compositor.create_surface`, `xdg_wm_base.get_xdg_surface`, and `zwp_linux_dmabuf_v1.create_params` all follow this pattern.

**Capability advertisement via events.** Send capability flags to the client immediately after bind, *before* any client request that depends on them. The client must not assume capabilities until it has received the initial events (enforced by a `wl_display_roundtrip` or equivalent). See `zwp_linux_dmabuf_v1`'s `format`/`modifier` events and the `wp_presentation.clock_id` event.

**Immutable-before-commit (double-buffer).** Accumulate all state in a pending buffer; finalise atomically with a `commit`-style request. This is `wl_surface.commit`'s design. For buffer import, `zwp_linux_buffer_params_v1` uses `add` calls followed by `create`.

```xml
<!-- Accumulate then commit -->
<request name="add">    <arg name="fd" type="fd"/> ... </request>
<request name="create"> <arg name="buffer" type="new_id" interface="wl_buffer"/> </request>
```

**Version-gated features.** Use the `since` attribute on new requests and events. Guard invocations at runtime:

```c
/* Server: only send v2 events to v2+ clients */
if (wl_resource_get_version(resource) >= 2)
    ext_notification_manager_v1_send_capability(resource, caps);

/* Client: only call v2 requests if we negotiated v2 */
if (wl_proxy_get_version((struct wl_proxy *)mgr) >= 2)
    ext_notification_manager_v1_set_icon(mgr, icon_fd, w, h, stride, fmt);
```

**Tie child lifetime to parent.** When the parent resource is destroyed, the compositor should destroy all child resources that logically belong to it. Use `wl_resource_set_destructor` on child objects and maintain a `wl_list` of children in the parent's state struct.

### 8.2 Anti-Patterns

**Missing `destroy` request.** Every interface that creates a resource on the server must have a `destroy` request. Without one, resources accumulate until the client disconnects. The resource will eventually be reclaimed, but you force the compositor to hold memory indefinitely.

**String where an FD would be safer.** Passing a file path as a `string` argument creates a TOCTOU race: the compositor opens the path at a different time than the client specified it, and the path may point to a different file by then. Always pass an open FD and let the kernel's reference counting ensure the file remains valid. (See how `wl_shm` uses an FD for the shared memory pool, not a path.)

**Global state that can't be re-queried.** If you advertise initial state only at bind time, late-binding clients miss it. Either use `wl_registry.global`/`global_remove` to announce hot-plug events, or design the global to replay its state to every new client (like `wl_output` replays geometry events on every bind).

**Synchronous acknowledgment.** Requiring the client to send a request in response to an event before the compositor proceeds (a "ping-pong" that blocks the compositor) breaks the asynchronous model. `xdg_wm_base.ping`/`pong` is the only sanctioned exception, and even that is best-effort.

**Breaking version compatibility by mutating existing requests.** Adding a new argument to an existing request rather than defining a new request with `since="N"` is a hard protocol break. It changes the opcode dispatch for all existing clients. Always use `since` to add new functionality as new messages at the end of the interface.

**Forgetting `allow-null` on optional strings.** If a string argument may legitimately be absent (e.g., the notification body), mark it `allow-null="true"`. Without this, some implementations will reject a null pointer at the libwayland layer before your code runs, producing an opaque protocol error.

---

## 9. Testing Protocol Extensions

### 9.1 Wire-Level Debugging

The simplest debugging tool is `WAYLAND_DEBUG=1`. When set in a client or compositor environment, `libwayland` prints every message it sends or receives to `stderr`:

```
[1234567.123] -> wl_display@1.get_registry(new id wl_registry@2)
[1234567.456] <- wl_registry@2.global(1, "wl_compositor", 6)
[1234567.789] <- wl_registry@2.global(2, "ext_notification_manager_v1", 2)
```

You can set `WAYLAND_DEBUG=client` or `WAYLAND_DEBUG=server` to log only one side. For more structured output, `wayland-debug` ([https://github.com/wmww/wayland-debug](https://github.com/wmww/wayland-debug)) parses this output and adds filtering, colour-coding, timestamps, and GDB integration:

```bash
# Run a client with wayland-debug in pipe mode
WAYLAND_DEBUG=1 my-notification-client 2>&1 | wayland-debug -p

# Or let wayland-debug launch the client
wayland-debug -r my-notification-client
```

### 9.2 Listing Globals with `wayland-info`

`wayland-info` ([https://gitlab.freedesktop.org/wayland/wayland-utils](https://gitlab.freedesktop.org/wayland/wayland-utils)) connects to the compositor and prints every advertised global with its version:

```
$ wayland-info
interface: 'wl_compositor', version: 6, name: 1
interface: 'wl_subcompositor', version: 1, name: 2
interface: 'ext_notification_manager_v1', version: 2, name: 7
interface: 'xdg_wm_base', version: 6, name: 8
```

This confirms that your compositor is correctly advertising the extension and at the expected version.

### 9.3 wlcs — Wayland Conformance Suite

`wlcs` ([https://github.com/canonical/wlcs](https://github.com/canonical/wlcs)) is Canonical's protocol conformance test suite, used by Mir, Mutter, and wlroots CI. Unlike test suites that use a separate debug extension, `wlcs` asks the compositor under test to provide an *integration module*: a set of C function pointers that let the test harness start the compositor, create clients, move windows, and check focus state.

A `wlcs` test for our notification protocol would look like:

```cpp
// notification_test.cpp
#include <wlcs/wlcs.h>

struct NotificationTest : public wlcs::Test {
    wlcs::Client client{the_server()};
    wlcs::NotifManager mgr{client.bind<ext_notification_manager_v1>()};
};

TEST_F(NotificationTest, ShowAndReceiveActivated) {
    auto notif = mgr.show("Test", "Body", 1000);
    // Simulate user clicking the notification
    the_server().activate_notification(notif.compositor_id(), 0 /*default*/);
    client.roundtrip();
    EXPECT_EQ(notif.last_action(), EXT_NOTIFICATION_V1_ACTION_DEFAULT);
}
```

`wlcs` comes with ~300 tests covering `wl_surface`, `xdg_shell`, and core protocol invariants; protocol authors add extension-specific suites alongside their protocol submission.

### 9.4 Weston's Test Suite

Weston provides a built-in test framework accessed via the `headless` backend. Tests run as clients against a real in-process Weston instance:

```bash
# Run all Weston protocol tests
meson test -C build/ --suite protocol
```

For your extension, add a `tests/notification-test.c` that links against `libweston` and the generated client bindings. Register it in `tests/meson.build`:

```python
test('notification-protocol',
    executable('notification-test', 'notification-test.c',
        dependencies: [dep_weston, dep_wayland_client],
        link_with: libweston_test_runner),
    timeout: 60)
```

### 9.5 Testing Version Negotiation

Version negotiation is a common source of bugs. Write dedicated tests that bind at every supported version:

```c
/* Test: bind at version 1, ensure v2-only events are not sent */
static void test_bind_v1(struct wl_display *display) {
    /* Request version 1 explicitly */
    struct ext_notification_manager_v1 *mgr =
        wl_registry_bind(registry, name,
                         &ext_notification_manager_v1_interface, 1);

    /* Version 1 listener has no .capability handler */
    wl_notification_manager_v1_add_listener(mgr, &v1_listener, NULL);
    wl_display_roundtrip(display);

    /* If the compositor incorrectly sent capability (v2 event) to a v1 client,
     * libwayland would log a version mismatch warning and skip the event. */
    assert(!v1_listener_got_capability);
    ext_notification_manager_v1_destroy(mgr);
}
```

Run with `WAYLAND_DEBUG=1` and look for "error in version" messages from libwayland.

### 9.6 Security Testing

If your protocol crosses trust boundaries (e.g., it allows an untrusted client to trigger a privileged compositor action), test for:

- **Null pointer injection**: Send `allow-null="true"` arguments as null; ensure the compositor handles them gracefully rather than crashing.
- **Integer overflow in array lengths**: Send malformed `array` arguments with large length fields; ensure the compositor validates before allocating.
- **Resource exhaustion**: Have a client create thousands of objects without destroying them; ensure the compositor rate-limits or rejects once past a reasonable cap.
- **Race conditions at disconnect**: Have the client disconnect mid-sequence; ensure `wl_resource_destroy` callbacks do not double-free or access freed memory.

The `GOVERNANCE.md` requirements for `wp_` protocols include a `SECURITY.md` section specifically because compositors must consider threat models from day one.

---

## Roadmap

### Near-term (6–12 months)

- **XDG session management promotion**: `xdg-session-management-v1` entered the experimental `xx_` namespace in wayland-protocols 1.48 (April 2026) and is on track to move to `staging/` once a second compositor implementation lands; it allows clients to restore window positions and states across sessions. [Source](https://www.phoronix.com/news/Wayland-Protocols-1.48)
- **Color management protocol ecosystem roll-out**: `wp_color_management_v1` merged into `staging/` in early 2025; the near-term focus is on driving compositor support in Mutter (GNOME), KWin (KDE), and Weston so the protocol can progress to `stable/`. Weston 16 Alpha (2026) added color-pipeline DRM back-end support as a prerequisite. [Source](https://www.phoronix.com/news/Wayland-Protocols-1.47)
- **Background effects and pointer warp stabilisation**: Both `ext_background_effects_v1` and `ext_pointer_warp_v1` were added to `staging/` in wayland-protocols 1.45 (June 2025); a second production compositor implementation is required before either can graduate to `stable/`. [Source](https://www.phoronix.com/news/Wayland-Protocols-1.45)
- **Text input protocol v4**: Wayland Protocols 1.46 (December 2025) included experimental text-input refinements resolving long-standing issues with input method coordination; a concrete `text-input-v4` staging proposal is expected within the 6–12 month window. [Source](https://www.phoronix.com/forums/forum/linux-graphics-x-org-drivers/wayland-display-server/1594241-wayland-protocols-1-46-released-with-new-experimental-additions)
- **Keyboard filter protocol**: `xx-keyboard-filter-v1` (added in wayland-protocols 1.48) allows a privileged client to intercept keyboard events before they reach the focused surface — enabling use-cases like on-screen keyboards and accessibility tools — and is expected to proceed to `staging/` pending security review. [Source](https://www.phoronix.com/news/Wayland-Protocols-1.48)

### Medium-term (1–3 years)

- **Zone-based window management (`xx-zones`)**: An experimental protocol added in wayland-protocols 1.48 lets compositors expose named display zones (similar to Windows Fancy Zones) to allow clients to snap into predefined layout regions. Design discussions are ongoing to reconcile this with `xdg-toplevel` tile state. [Source](https://www.phoronix.com/news/Wayland-Protocols-1.48)
- **Display cutouts protocol (`xx-cutouts`)**: Also added experimentally in 1.48, this protocol communicates notch and camera-hole regions to clients so they can avoid obscuring UI elements — essential for tablets and embedded displays. Promotion to `staging/` requires broad adoption across form factors. [Source](https://www.phoronix.com/news/Wayland-Protocols-1.48)
- **Security context hardening (`wp_security_context_v1`)**: The existing `staging/` security context protocol (which allows a compositor to sandbox clients into named trust domains) is expected to see follow-on extensions for fine-grained capability delegation, feeding into Flatpak portal integration. Note: needs verification on exact staging timeline.
- **Explicit GPU synchronisation `stable/` promotion**: `linux-drm-syncobj-v1` has been in `staging/` since 2024 and is implemented in Mesa, wlroots, Mutter, and KWin; the path to `stable/` depends on resolving remaining edge cases around multi-GPU timeline fence import. [Source](https://wayland.app/protocols/linux-drm-syncobj-v1)
- **New input method protocol**: A major rework of input method handling announced alongside 1.45 aims to unify `text-input` and `input-method` into a single coherent protocol, an undertaking that historically has required multi-year consensus building. [Source](https://www.phoronix.com/news/Wayland-Protocols-1.45)

### Long-term

- **Wayland protocol governance automation**: The `GOVERNANCE.md` process currently relies on manual tracking of implementation counts; long-term plans include a CI-driven dashboard that automatically checks whether a staging protocol has the required two distinct compositor implementations before allowing a `stable/` promotion merge request to land. Note: needs verification on specific tooling proposals.
- **Richer capability discovery**: The current global-registry model requires clients to speculatively bind globals; proposals have circulated (on wayland-devel) for a structured capabilities manifest that a compositor could advertise up front, reducing initial roundtrips and enabling better lazy binding of optional protocols. Note: needs verification.
- **Colour pipeline compositing (`frog_color_management` successors)**: Beyond the stable `wp_color_management_v1`, compositor engineers are exploring Wayland-level signalling for scene-referred HDR compositing and gamut mapping hints that go beyond per-surface tagging — integrating tightly with future KMS colour pipeline kernel interfaces. [Source](https://www.phoronix.com/news/Wayland-CM-HDR-Merged)
- **Wayland accessibility protocol**: Long-discussed but as yet unspecified, a formal `wp_accessibility_v1` would let assistive technologies (screen readers, magnifiers) hook into the compositor's surface tree; the AT-SPI2 community and GNOME accessibility team have identified this as necessary for post-X11 accessibility parity. Note: needs verification on formal proposal status.

---

## 10. Integrations

**Ch20 — Wayland Protocol Fundamentals**: This chapter is the prerequisite — it covers the core protocol objects (`wl_surface`, `wl_compositor`, `wl_seat`) that extension protocols build upon, and the registry model by which extensions are discovered.

**Ch21 — Building Compositors with wlroots**: wlroots is the reference implementation of most `zwlr_*` extensions. The chapter shows how wlroots wraps `libwayland-server` primitives into `wlr_protocol_*` structs; the implementation patterns in §5 above are the same ones wlroots uses internally for `wlr_xdg_shell`, `wlr_layer_shell`, and `wlr_output_management`.

**Ch22 — Production Compositors (Mutter and KWin)**: Mutter and KWin are the second and third compositor implementations required for `wp_` promotion. The chapter covers how GNOME Shell and KDE Plasma implement stable protocols; protocol authors targeting the ecosystem must build consensus with those teams as part of the governance process.

**Ch46 — The Evolving Wayland Protocol Ecosystem**: Surveys the historical arc of the protocol landscape — from the original `wl_shell` (now deprecated) through `xdg-shell` becoming the canonical desktop shell, to the ongoing migration of `zwp_*` unstable protocols into `staging/` and `stable/`. Governance reforms (the introduction of the `xx_` experimental namespace, the tightening of implementation requirements) are covered in depth.

**Ch75 — Explicit GPU Synchronisation**: `linux-drm-syncobj-v1` is one of the most recent `staging/` additions at the time of writing, and it exemplifies all the advanced patterns: FD arguments for DMA-fence file descriptors, factory-style object creation, version-gated features, and cross-compositor coordination between Mesa, wlroots, and Mutter. The chapter shows the full commit/release/acquire timeline from the compositor's perspective.

**Ch116 / Ch121 — DRM Lease and VR**: `wp_drm_lease_device_v1` (whose evolution from `zwlr_drm_lease_device_v1` was the case study in §7.4) is the primary enabling protocol for VR under Wayland. Those chapters cover the DRM kernel side; this chapter covers the Wayland protocol mechanism by which the compositor grants the VR runtime a DRM lease.

**Ch132 — Wayland Security**: Protocols must consider their threat model from day one. Ch132 discusses compositor security architecture, the privilege separation enforced by the Wayland socket model, and how protocols like `ext_session_lock_v1` and `wp_security_context_v1` restrict client capabilities. The security testing notes in §9.6 above are a subset of the analysis covered there.

---

*Sources consulted for this chapter:*

- [The Wayland Protocol Book — Wire Protocol](https://wayland-book.com/protocol-design/wire-protocol.html)
- [The Wayland Protocol Book — wayland-scanner](https://wayland-book.com/libwayland/wayland-scanner.html)
- [The Wayland Protocol Book — Registering Globals (Server)](https://wayland-book.com/registry/server-side.html)
- [The Wayland Protocol Book — Binding to Globals (Client)](https://wayland-book.com/registry/binding.html)
- [The Wayland Protocol Book — Design Patterns](https://wayland-book.com/protocol-design/design-patterns.html)
- [Wayland Protocol Specification — Wire Format](https://wayland.freedesktop.org/docs/book/Protocol.html)
- [Wayland Protocol XML Message Definition Language](https://wayland.freedesktop.org/docs/book/Message_XML.html)
- [libwayland-server API Reference](https://people.collabora.com/~mvlad/wayland_breathe/dir/specs/server-api.html)
- [wayland-protocols GOVERNANCE.md (mirror)](https://github.com/wayland-mirror/wayland-protocols/blob/main/GOVERNANCE.md)
- [linux-dmabuf-unstable-v1.xml (mirror)](https://github.com/wayland-mirror/wayland-protocols/blob/main/unstable/linux-dmabuf/linux-dmabuf-unstable-v1.xml)
- [Meson Wayland Module](https://mesonbuild.com/Wayland-module.html)
- [wayland-debug](https://github.com/wmww/wayland-debug)
- [wlcs — Wayland Conformance Suite](https://github.com/canonical/wlcs)
- [DRM Lease Protocol | Wayland Explorer](https://wayland.app/protocols/drm-lease-v1)
- [wlr-protocols repository](https://github.com/swaywm/wlr-protocols)
- [Pekka Paalanen: Wayland Protocol Design — Object Lifespan](https://ppaalanen.blogspot.com/2014/07/wayland-protocol-design-object-lifespan.html)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
