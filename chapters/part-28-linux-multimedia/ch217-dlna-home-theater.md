# Chapter 217: DLNA and Home Theater: GUPnP, Rygel, Kodi, and Network Media Streaming

**Part XXVIII — Linux Multimedia**

**Audiences**: Application developers building home media servers or DLNA-compatible clients on Linux; systems developers integrating network streaming with V4L2 capture, PipeWire audio, and hardware video decode.

---

## Scope

This chapter walks through the full Linux home-theater networking stack: the UPnP/DLNA protocol layers, the GLib-based GUPnP/GSsdp frameworks used by native Linux software, Rygel as the GNOME-ecosystem DLNA media server, Jellyfin as the dominant open-source self-hosted server, Kodi as the flagship Linux media center, and the reverse-engineered AirPlay 2 and Google Cast protocols. It also covers the Avahi mDNS daemon that underpins service discovery, network and firewall requirements, transcoding pipeline design, and a comparative survey of the major Linux media server options.

---

## Table of Contents

1. [UPnP/DLNA Protocol Stack](#1-upnpdlna-protocol-stack)
2. [GUPnP: GLib-Based UPnP Framework](#2-gupnp-glib-based-upnp-framework)
3. [Rygel Media Server](#3-rygel-media-server)
4. [Jellyfin: Open-Source Self-Hosted Media Server](#4-jellyfin-open-source-self-hosted-media-server)
5. [Kodi: The Linux Media Center](#5-kodi-the-linux-media-center)
6. [AirPlay 2 on Linux: UxPlay](#6-airplay-2-on-linux-uxplay)
7. [Chromecast and Google Cast](#7-chromecast-and-google-cast)
8. [Avahi: mDNS/DNS-SD for Linux](#8-avahi-mdnsdns-sd-for-linux)
9. [Firewall and Network Considerations](#9-firewall-and-network-considerations)
10. [Transcoding Pipeline Design](#10-transcoding-pipeline-design)
11. [Security Concerns](#11-security-concerns)
12. [Comparison: Plex, Jellyfin, Emby, and Rygel](#12-comparison-plex-jellyfin-emby-and-rygel)
13. [Integrations](#13-integrations)

---

## 1. UPnP/DLNA Protocol Stack

UPnP (Universal Plug and Play) is a suite of networking protocols that enable devices to advertise services, discover peers, and exchange control messages without manual configuration. DLNA (Digital Living Network Alliance) is a certification layer on top of UPnP AV that specifies mandatory media format profiles, HTTP delivery requirements, and device class interactions. Both rest on four layered protocols. [Source](https://en.wikipedia.org/wiki/DLNA)

### 1.1 SSDP — Discovery

The Simple Service Discovery Protocol carries device announcements and queries over UDP multicast. Devices join the `239.255.255.250` group and listen on port 1900. A control point searching for devices sends an `M-SEARCH` request to the multicast group:

```http
M-SEARCH * HTTP/1.1
HOST: 239.255.255.250:1900
MAN: "ssdp:discover"
MX: 3
ST: urn:schemas-upnp-org:device:MediaServer:1
```

Devices respond with unicast UDP containing their location URL (pointing to an XML device description) and a `USN` (Unique Service Name). Devices announce their presence unprompted via `NOTIFY ssdp:alive` on startup and `NOTIFY ssdp:byebye` on shutdown. TTL for SSDP packets is typically 4. [Source](https://gitlab.gnome.org/GNOME/gssdp)

### 1.2 Device Description and SOAP Control

After a control point resolves a device URL, it fetches the device description XML via HTTP GET. This XML lists embedded services, each referencing a Service Control Protocol Description (SCPD) XML that enumerates actions and state variables. Control points invoke actions via SOAP: an HTTP POST with a `SOAPAction` header and an XML body wrapped in `<s:Envelope>`.

The two central UPnP AV services are:

- **ContentDirectory** (`urn:schemas-upnp-org:service:ContentDirectory:1`) — Browse and Search actions that return DIDL-Lite XML describing the server's media tree.
- **AVTransport** (`urn:schemas-upnp-org:service:AVTransport:1`) — Play, Pause, Stop, Seek, SetAVTransportURI actions that control a renderer's playback state.

> **Note:** The UPnP AV Architecture specification document has been revised through multiple versions. Some sources refer to ContentDirectory version 4 or AVTransport version 3 in specification context, but the service type version string deployed in production implementations — including GUPnP, Rygel, Kodi Platinum, and Jellyfin's DLNA plugin — uses `:1` in the `serviceType` URN. Higher version numbers in the service URN could not be verified against current upstream code and should be treated as specification revision numbers, not wire-protocol version numbers.

### 1.3 GENA — Eventing

GENA (General Event Notification Architecture) allows a control point to subscribe to state variable changes. The control point sends an HTTP `SUBSCRIBE` to the service's event URL with a `CALLBACK` header naming a local HTTP server. The device delivers state change notifications via HTTP `NOTIFY` POST to that callback, carrying XML-encoded new values. Subscriptions expire unless renewed.

### 1.4 HTTP Content Delivery

Actual media bytes are served over plain HTTP, using standard `Content-Type` and `Content-Length` headers. DLNA adds transfer-mode headers: `transferMode.dlna.org: Streaming` for AV content and `getcontentFeatures.dlna.org: 1` to request DLNA metadata in the response.

### 1.5 DLNA Profiles and Device Classes

The `protocolInfo` attribute on each DIDL-Lite resource encodes the MIME type and a DLNA profile name:

```
audio/mpeg:DLNA.ORG_PN=MP3;DLNA.ORG_OP=01;DLNA.ORG_FLAGS=01700000000000000000000000000000
video/mp4:DLNA.ORG_PN=AVC_MP4_HP_HD_AAC;DLNA.ORG_OP=01
image/jpeg:DLNA.ORG_PN=JPEG_LRG
```

`DLNA.ORG_OP=01` signals that range-based seeking is supported via HTTP `Range` headers. `DLNA.ORG_FLAGS` encodes additional transfer capabilities as a 32-bit hex field.

DLNA defines four device classes: **DMS** (Digital Media Server, exposes content), **DMR** (Digital Media Renderer, receives and renders content), **DMP** (Digital Media Player, browses and renders), and **DMC** (Digital Media Controller, pushes content from a DMS to a DMR). [Source](https://en.wikipedia.org/wiki/DLNA)

---

## 2. GUPnP: GLib-Based UPnP Framework

GUPnP is the primary UPnP framework in the GNOME/freedesktop ecosystem, built on GLib, GObject, and libsoup. It is composed of six cooperating libraries, all licensed LGPL-2.1. [Source](https://wiki.gnome.org/Projects/GUPnP)

### 2.1 GSsdp — SSDP Bus

`gssdp` (currently 1.6.x) implements the SSDP multicast bus as a GObject hierarchy. `GSSDPClient` wraps a UDP socket on `239.255.255.250:1900` and emits a `message-received` signal for each incoming packet. [Source](https://gitlab.gnome.org/GNOME/gssdp)

```c
/* gssdp/gssdp-client.h — constructors */
GSSDPClient *gssdp_client_new (GMainContext *context,
                               const char   *iface,
                               GError      **error);
GSSDPClient *gssdp_client_new_with_port (const char *iface,
                                         guint16     msearch_port,
                                         GError    **error);
```

`GSSDPResourceGroup` manages device announcements:

```c
/* Announce a device on the local network */
GSSDPResourceGroup *group = gssdp_resource_group_new (client);
gssdp_resource_group_add_resource_simple (
    group,
    "urn:schemas-upnp-org:device:MediaServer:1",   /* target */
    "uuid:my-server-uuid::urn:schemas-upnp-org:device:MediaServer:1", /* USN */
    locations);                                    /* GList of location URLs */
gssdp_resource_group_set_max_age (group, 1800);
/* DLNA spec requires minimum 120 ms delay between consecutive NOTIFY packets */
gssdp_resource_group_set_message_delay (group, 120);
gssdp_resource_group_set_available (group, TRUE);
```

`GSSDPResourceBrowser` handles discovery:

```c
GSSDPResourceBrowser *browser =
    gssdp_resource_browser_new (client,
                                "urn:schemas-upnp-org:device:MediaServer:1");
g_signal_connect (browser, "resource-available",
                  G_CALLBACK (on_resource_available), NULL);
gssdp_resource_browser_set_active (browser, TRUE);
```

### 2.2 GUPnP Core — Context, Devices, and Services

`GUPnPContext` extends `GSSDPClient` and adds HTTP server/client capabilities via libsoup, exposing `get_server()` (returns `SoupServer`) and `get_session()` (returns `SoupSession`). Since GUPnP 1.6.0, the preferred constructors are `gupnp_context_new_for_address()` and `gupnp_context_new_full()`. [Source](https://gnome.pages.gitlab.gnome.org/gupnp/docs/class.Context.html)

Creating a device:

```c
/* server-tutorial.c pattern — gupnp docs */
GUPnPContext    *context = gupnp_context_new_for_address (NULL, NULL, 0, NULL);
GUPnPRootDevice *dev     =
    gupnp_root_device_new (context, "MediaServer1.xml", ".", NULL);
gupnp_root_device_set_available (dev, TRUE);
```

Retrieving a service and handling actions:

```c
GUPnPService *service =
    GUPNP_SERVICE (
        gupnp_device_info_get_service (
            GUPNP_DEVICE_INFO (dev),
            "urn:schemas-upnp-org:service:ContentDirectory:1"));
g_signal_connect (service, "action-invoked::Browse",
                  G_CALLBACK (on_browse), NULL);
```

The `gupnp-binding-tool` code generator reads a service description XML and emits type-safe C wrappers, eliminating manual `gupnp_service_action_get()`/`gupnp_service_action_set()` calls with `GValue`. [Source](https://gnome.pages.gitlab.gnome.org/gupnp/docs/server-tutorial.html)

### 2.3 GUPnP-AV — DIDL-Lite Parsing and Generation

`gupnp-av` provides `GUPnPDIDLLiteParser` and `GUPnPDIDLLiteWriter` for the DIDL-Lite XML dialect used by ContentDirectory. `GUPnPDIDLLiteObject` (and its subclasses `GUPnPDIDLLiteItem`, `GUPnPDIDLLiteContainer`) carries typed accessors for all DIDL-Lite metadata fields. [Source](https://github.com/GNOME/gupnp-av/blob/master/libgupnp-av/gupnp-didl-lite-object.h)

```c
/* libgupnp-av/gupnp-didl-lite-object.h — selected accessors */
const char *gupnp_didl_lite_object_get_id       (GUPnPDIDLLiteObject *object);
const char *gupnp_didl_lite_object_get_title     (GUPnPDIDLLiteObject *object);
GList      *gupnp_didl_lite_object_get_resources (GUPnPDIDLLiteObject *object);
GUPnPDIDLLiteResource *
            gupnp_didl_lite_object_add_resource  (GUPnPDIDLLiteObject *object);
```

The library uses libxml2 internally and iterates `xmlNode` trees, so it never builds an in-memory DOM for the full ContentDirectory response — important for large libraries.

### 2.4 GUPnP-DLNA — Profile Matching

`gupnp-dlna` (0.13.x) sits above `gupnp-av` and provides `GUPnPDLNAProfileGuesser`, which analyses a media file's container and codec information and returns the best matching DLNA profile. [Source](https://gnome.pages.gitlab.gnome.org/gupnp-dlna/docs/gupnp-dlna/)

```c
GUPnPDLNAProfileGuesser *guesser =
    gupnp_dlna_profile_guesser_new (TRUE, FALSE);
GUPnPDLNAInformation *info =
    gupnp_dlna_profile_guesser_guess_profile_from_uri (guesser, uri, NULL);
GUPnPDLNAProfile *profile =
    gupnp_dlna_information_get_profile (info);
const char *pn   = gupnp_dlna_profile_get_name (profile);  /* e.g. "AVC_MP4_HP_HD_AAC" */
const char *mime = gupnp_dlna_profile_get_mime (profile);  /* e.g. "video/mp4" */
```

`gupnp_dlna_profile_get_extended()` returns `TRUE` for profiles that are GUPnP-DLNA extensions rather than standard DLNA profiles.

### 2.5 GUPnP Control Point — Discovering and Controlling Remote Devices

For a client that browses remote ContentDirectory services, the entry point is `GUPnPControlPoint`. It wraps a `GSSDPResourceBrowser` for device discovery and emits signals when devices arrive or depart: [Source](https://gnome.pages.gitlab.gnome.org/gupnp/docs/client-tutorial.html)

```c
/* gupnp/gupnp-control-point.h */
GUPnPControlPoint *cp =
    gupnp_control_point_new (context,
                             "urn:schemas-upnp-org:device:MediaServer:1");

g_signal_connect (cp, "device-proxy-available",
                  G_CALLBACK (on_device_available), NULL);
g_signal_connect (cp, "device-proxy-unavailable",
                  G_CALLBACK (on_device_unavailable), NULL);
gssdp_resource_browser_set_active (GSSDP_RESOURCE_BROWSER (cp), TRUE);
```

When a MediaServer appears, the callback receives a `GUPnPDeviceProxy`. From it the application retrieves a `GUPnPServiceProxy` for the ContentDirectory service and invokes Browse:

```c
static void on_device_available (GUPnPControlPoint *cp,
                                 GUPnPDeviceProxy  *dev_proxy,
                                 gpointer           user_data)
{
    GUPnPServiceProxy *cd =
        GUPNP_SERVICE_PROXY (
            gupnp_device_info_get_service (
                GUPNP_DEVICE_INFO (dev_proxy),
                "urn:schemas-upnp-org:service:ContentDirectory:1"));

    /* Invoke Browse — result arrives in browse_cb */
    gupnp_service_proxy_begin_action (
        cd, "Browse", browse_cb, NULL,
        "ObjectID",       G_TYPE_STRING,  "0",
        "BrowseFlag",     G_TYPE_STRING,  "BrowseDirectChildren",
        "Filter",         G_TYPE_STRING,  "*",
        "StartingIndex",  G_TYPE_UINT,    0,
        "RequestedCount", G_TYPE_UINT,    0,
        "SortCriteria",   G_TYPE_STRING,  "",
        NULL);
}
```

`gupnp_service_proxy_begin_action()` issues a non-blocking SOAP POST; `browse_cb` receives the action result and extracts `Result` (a DIDL-Lite XML string), `NumberReturned`, `TotalMatches`, and `UpdateID` via `gupnp_service_proxy_end_action()`. GENA event subscriptions use `gupnp_service_proxy_add_notify()` to listen for state variable changes, with the subscription maintained by GUPnP automatically issuing HTTP SUBSCRIBE/renewal requests.

### 2.6 DIDL-Lite XML Format

A ContentDirectory Browse response body contains DIDL-Lite XML inside the SOAP envelope. An abbreviated example:

```xml
<DIDL-Lite xmlns="urn:schemas-upnp-org:metadata-1-0/DIDL-Lite/"
           xmlns:dc="http://purl.org/dc/elements/1.1/"
           xmlns:upnp="urn:schemas-upnp-org:metadata-1-0/upnp/">
  <item id="42" parentID="0" restricted="1">
    <dc:title>Dark Side of the Moon</dc:title>
    <upnp:class>object.item.audioItem.musicTrack</upnp:class>
    <dc:creator>Pink Floyd</dc:creator>
    <upnp:album>The Dark Side of the Moon</upnp:album>
    <upnp:trackNumber>1</upnp:trackNumber>
    <res protocolInfo="audio/mpeg:DLNA.ORG_PN=MP3;DLNA.ORG_OP=01"
         size="8765432"
         duration="0:07:06"
         bitrate="320000">
      http://192.168.1.10:8200/content/42
    </res>
  </item>
</DIDL-Lite>
```

`gupnp-av`'s `GUPnPDIDLLiteParser` parses this XML and emits `object-available` signals, passing `GUPnPDIDLLiteObject` instances to the application. The `get_resources()` call returns a `GList` of `GUPnPDIDLLiteResource` objects, each carrying a `protocolInfo` string and the content URI.

### 2.7 GUPnP-IGD and GObject Introspection

`gupnp-igd` provides a client for UPnP IGD (Internet Gateway Device) NAT traversal — used when an application needs to open an external port on a home router. GObject introspection metadata (`.gir`/`.typelib` files) shipped with all six libraries enables Python, JavaScript, and Vala bindings without separate wrapper code. Vala has first-class support with hand-maintained VAPI files.

---

## 3. Rygel Media Server

Rygel (version 46.alpha as of mid-2026) is the GNOME-ecosystem DLNA/UPnP media server and renderer, written primarily in Vala and built on GUPnP. It functions as both a UPnP AV MediaServer (DMS) and a MediaRenderer (DMR). [Source](https://wiki.gnome.org/Projects/Rygel)

### 3.1 Library Architecture

Rygel exposes several developer libraries:

| Library | Purpose |
|---|---|
| `librygel-core` | Basic UPnP-AV infrastructure |
| `librygel-db` | SQLite helper (since 0.28) |
| `librygel-server` | MediaServer toolkit |
| `librygel-renderer` | Renderer infrastructure |
| `librygel-renderer-gst` | GStreamer-based renderer (since 0.18) |
| `librygel-ruih` | UPnP Remote UI Host server |

Core build dependencies: `gssdp`, `gupnp`, `gupnp-av`, GStreamer, GIO, libgee, libsoup, libmediaart, libxml2, Vala. Optional: GTK (UI), SQLite + gupnp-dlna (MediaExport plugin), TinySparql (Localsearch plugin). [Source](https://github.com/GNOME/rygel/blob/master/README.md)

### 3.2 Server Plugins

Rygel's server functionality is entirely plugin-driven. Each plugin provides one or more `RygelMediaContainer` hierarchies that ContentDirectory Browse/Search actions traverse. [Source](https://wiki.gnome.org/Projects/Rygel/ServerPlugins)

**MediaExport** — Recursively exports configured directories. It handles any URI that GIO/gvfs and GStreamer can open, uses `gupnp-dlna` APIs to extract and cache metadata, and stores that metadata in SQLite via `librygel-db`. It cannot run concurrently with the Localsearch plugin, since both serve the same role.

**Localsearch** — Uses TinySparql (the project formerly known as Tracker3/tinysparql) to query a continuously updated SPARQL index of media files. This avoids re-scanning directories manually — TinySparql maintains the index as filesystem changes occur. Earlier documentation referred to this plugin as "Tracker"; the current upstream name is Localsearch.

**GstLaunch** — Accepts arbitrary GStreamer pipeline description strings in `rygel.conf` and exposes each pipeline as a separate UPnP MediaServer item. Useful for exposing webcam feeds or radio streams.

**External** — Bridges `org.gnome.MediaServer2.<ApplicationName>` D-Bus interfaces into DLNA servers, enabling DVB Daemon, Rhythmbox, or any MPRIS2-compatible application to appear as a network media source.

### 3.3 Renderer Plugins

The **MPRIS** renderer plugin uses MPRIS2 D-Bus interfaces to forward AVTransport commands (play, pause, seek, stop, set URI) to media players such as Totem, Rhythmbox, or VLC. The **Playbin** renderer plugin uses GStreamer `PlayBin3` directly and is suitable for background (non-fullscreen) rendering.

### 3.4 GStreamer Media Engine and Transcoding

The GStreamer media engine (the default) provides streaming, transcoding, and time-based seeking. When a requesting client declares a `protocolInfo` list that does not match the source file's native format, Rygel constructs an on-the-fly GStreamer pipeline:

```
uridecodebin uri=file:///media/movie.mkv
  → queue
  → avenc_aac (or lamemp3enc)
  → muxer (mp4mux / mpegpsmux)
  → tcpserversink / appsink → HTTP response
```

Supported transcode output profiles include MP3 and LPCM for audio, and MPEG-TS, WMV, and H.264/MP4 for video.

The **Simple** media engine bypasses all transcoding — it passes files through unchanged via HTTP. No time-based seeking support is provided by the Simple engine; it is intended for embedded deployments where GStreamer is unavailable or resource-constrained.

### 3.5 Configuration

Rygel reads configuration in priority order: environment variables, command-line arguments, user config (`$XDG_CONFIG_HOME/rygel.conf`), system config (`/etc/rygel.conf`). The format is INI. Key environment variables: `RYGEL_IFACE`, `RYGEL_PORT`, `RYGEL_DISABLE_TRANSCODING`, `RYGEL_LOG`, `RYGEL_PLUGIN_PATH`. [Source](https://gnome.pages.gitlab.gnome.org/rygel/configuration.html)

```ini
# /etc/rygel.conf (fragment)
[general]
interface=eth0
port=8200
enable-transcoding=true
log-level=default:2

[MediaExport]
enabled=true
uris=/home/user/Music;/home/user/Videos

[Localsearch]
enabled=false
```

---

## 4. Jellyfin: Open-Source Self-Hosted Media Server

Jellyfin is a GPL-3.0 media server written in C#/.NET 8. It is the community-maintained fork of Emby that branched when Emby moved to a proprietary model. [Source](https://jellyfin.org/posts/jellyfin-release-10.9.0/)

### 4.1 DLNA Plugin Architecture

As of Jellyfin 10.9.0 (May 2024), DLNA support was removed from the core server and moved to an optional first-party plugin. The DLNA plugin is separately versioned (Version 11, May 2026) and installed from the Jellyfin plugin catalogue. [Source](https://github.com/jellyfin/jellyfin-plugin-dlna)

The plugin uses **Rssdp**, a .NET SSDP library, for low-level multicast messaging — not the GLib-based GSsdp library. The key assemblies are:

| Assembly | Role |
|---|---|
| `Jellyfin.Plugin.Dlna` | Main plugin entry point, UPnP device hosting, SSDP publishing |
| `Jellyfin.Plugin.Dlna.Model` | Device profiles, DLNA header enums |
| `Jellyfin.Plugin.Dlna.Playback` | Transcoding coordination, DLNA header parsing |
| `Rssdp` | SSDP communications (`SsdpCommunicationsServer`, `SsdpDevice`) |

`DlnaHost` initialises a `SsdpDevicePublisher` that announces the Jellyfin DMS identity on UDP port 1900. The ContentDirectory service responds to Browse/Search with DIDL-Lite XML built from the Jellyfin library database.

### 4.2 PlayTo — DLNA Renderer Control

The plugin's PlayTo subsystem enables "push to TV": `PlayToManager` discovers DLNA renderers on the local network and exposes them in the Jellyfin web UI. When a user requests playback on a remote renderer, PlayTo issues AVTransport `SetAVTransportURI` and `Play` SOAP calls to the device. `DlnaPluginConfiguration` controls whether PlayTo is enabled, the SSDP poll interval, and which Jellyfin user account is used for renderer-initiated playback (`DefaultUserId`).

### 4.3 Hardware Transcoding

Jellyfin transcodes through `jellyfin-ffmpeg`, a patched FFmpeg build with expanded hardware acceleration support. Three hardware paths are validated on Linux: [Source](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/)

- **VA-API** — Intel integrated graphics (Gen 8+) and AMD GPUs via the Mesa VA-API driver. The `jellyfin` user must belong to the `render` and `video` groups.
- **NVENC/NVDEC** — NVIDIA GPUs via the CUDA-based encode/decode paths. Requires the proprietary NVIDIA driver.
- **QSV (Intel Quick Sync Video)** — Intel CPUs/iGPUs via the oneVPL/MFX backend.

Using upstream FFmpeg rather than `jellyfin-ffmpeg` results in partial or absent hardware acceleration because the patches are not merged upstream.

### 4.4 Network Requirements

UDP port 1900 is mandatory for SSDP; it cannot be remapped per the UPnP specification. DLNA clients must be on the same subnet because multicast does not cross routers by default. Containerised deployments require `--network=host` because Docker's bridge networking blocks multicast. [Source](https://jellyfin.org/docs/general/post-install/networking/dlna/)

---

## 5. Kodi: The Linux Media Center

Kodi is a GPL-2.0 open-source media center application supporting Linux, macOS, Windows, Android, and embedded Linux (Raspberry Pi, ODROID). [Source](https://github.com/xbmc/xbmc)

### 5.1 UPnP Stack: Platinum SDK

Kodi's UPnP implementation is built on the **Platinum UPnP SDK** (Plutinosoft), bundled at `lib/libUPnP/Platinum/` in the Kodi source tree. Platinum consists of two layers: the Neptune portable C++ runtime (threads, sockets, logging) and the Platinum UPnP framework on top. This is distinct from the libupnp/pupnp library used by some other projects. [Source](https://github.com/xbmc/xbmc/tree/master/lib/libUPnP/Platinum)

### 5.2 CUPnP Architecture

`CUPnP` is a singleton (`CUPnP::GetInstance()`) that owns a `PLT_UPnP` runtime. It manages four roles, each activatable independently: [Source](https://github.com/xbmc/xbmc/blob/master/xbmc/network/upnp/UPnP.cpp)

```cpp
// xbmc/network/upnp/UPnP.cpp
void CUPnP::StartClient();     // DMC — discovers remote renderers
void CUPnP::StartServer();     // DMS — serves Kodi's library over UPnP
void CUPnP::StartRenderer();   // DMR — receives play commands from DMCs
void CUPnP::StartController(); // BrowseController for the built-in UPnP browser
```

`CMediaBrowser` extends `PLT_SyncMediaBrowser` and `PLT_MediaContainerChangesListener` — it provides synchronous browse/search against remote DMS devices and is used by Kodi's UPnP source plugin (visible in the Sources list as "UPnP devices"). `CMediaController` extends `PLT_MediaController` and its delegate interface — it discovers renderers and issues AVTransport SOAP commands.

UUIDs for Kodi's own DMS and DMR are generated once and persisted in the user profile (`upnpserver.xml`) for stable device identity across restarts.

`CUPnPCleaner` is a background thread that performs deferred cleanup of Platinum device objects after shutdown. Platinum device and control-point objects cannot be destroyed on the calling thread (they may hold references counted across the UPnP message dispatch loop), so `CUPnP` queues them to `CUPnPCleaner` which drains the queue on its own thread after a safety delay. Thread safety for callback-facing data structures (for example, the `g_UserData` list of discovered renderers) is maintained via a `CCriticalSection` mutex acquired before any modification or read.

### 5.3 Linux Rendering: DRM/GBM/GLES

Kodi 22 undertook a substantial rewrite of the Linux rendering path for embedded and desktop platforms. [Source](https://linuxiac.com/kodi-22-enters-beta-with-big-linux-rendering-changes/)

- **Atomic DRM**: GUI rendered into a GBM-backed overlay plane; decoded video frames placed in the primary DRM plane via atomic commit (`drmModeAtomicCommit`). This separates GUI and video compositing at the KMS level.
- **EGL DMA-BUF import**: For the GLES path on platforms without DRM plane split, `EGL_EXT_image_dma_buf_import` allows VAAPI-decoded frames to be imported as `EGLImage` objects without a CPU copy.
- **VAAPI decode**: HEVC 4:2:2 and 4:4:4 validated on GLES with 12-bit decoding and dithering. High-quality scalers ported to GLES.
- **HDR pipeline**: End-to-end HDR — 10-bit DRM output, HDR metadata signalling, tonemapped GUI.

### 5.4 Add-on API

Kodi's add-on system provides two surfaces:

- **Python add-ons** (`xbmcgui`, `xbmcaddon`, `xbmcplugin` modules): used for content plugins (video sources, subtitles, skins). The majority of third-party Kodi extensions are Python-based.
- **Binary add-ons (C++ ABI)**: used for PVR backends (DVB, IPTV), game emulators, and audio DSP. PVR add-ons implement the `kodi::addon::IAddonInstance` interface and communicate via a C ABI header, making them version-stable across Kodi releases.

---

## 6. AirPlay 2 on Linux: UxPlay

UxPlay is an open-source AirPlay 2 server implementation for Linux (and macOS/Windows), built on GStreamer. It implements what Apple calls the "Legacy Protocol" — the subset of AirPlay 2 used for screen mirroring, which iOS 17 and earlier continue to support. [Source](https://github.com/FDH2/UxPlay)

### 6.1 Service Announcement

UxPlay announces itself on the local network as an AirPlay receiver via mDNS, registering a `_airplay._tcp` service. Since a recent build option was introduced, UxPlay includes a minimal built-in mDNS implementation and no longer requires Avahi by default. Compiling with `-DUSE_DNS_SD=1` substitutes the system's Avahi or Bonjour DNS-SD library.

### 6.2 Audio and Video Paths

| Mode | Audio codec | Video codec |
|---|---|---|
| Screen mirror | AAC (lossy), AES-encrypted | H.264 (default) or H.265 with `-h265` |
| Audio-only | ALAC (lossless) | — |

The GStreamer pipeline for the video mirror path:

```
appsrc (H.264 RTP payload)
  → h264parse
  → decodebin         ← auto-selects hardware or software decoder
  → videoconvert (or v4l2convert for GPU accel)
  → autovideosink
```

`decodebin` rank-orders available decoders: `vah264dec` (VA-API, Intel/AMD), `nvh264dec` (NVIDIA CUDA), `v4l2h264dec` (V4L2/Raspberry Pi), then `avdec_h264` (software fallback). The `-h265` flag substitutes `h265parse` and a matching HEVC decoder.

### 6.3 Authentication and Protocol Status

UxPlay supports three authentication modes: 4-digit PIN entry, HTTP MD5 Digest authentication, and device-ID access control lists. The RAOP2 audio transport uses AES encryption; the key exchange is a reverse-engineered protocol derived from the shairplay/playfair lineage and is not publicly documented by Apple.

The implementation covers the Legacy Protocol (screen mirroring). Full AirPlay 2 multiroom audio, HomeKit integration, and newer Airplay 2 sender features are outside scope. The protocol reverse-engineering status is ongoing, and future iOS versions may drop Legacy Protocol support.

---

## 7. Chromecast and Google Cast

The Google Cast protocol (CASTV2) is used by Chromecast devices, Google Nest displays, Android TV, and Google Smart Screens. [Source](https://kyteinsky.github.io/p/chromecast-protocol/)

### 7.1 Discovery

Cast devices advertise themselves via mDNS with the service type `_googlecast._tcp`. A sender application resolves the device using Avahi (or the system mDNS resolver) to obtain the device's IP address and port 8009.

### 7.2 Control Channel

The sender opens a TCP connection to port 8009 and performs a TLS handshake. The device presents a self-signed certificate. Messages are framed as a 32-bit big-endian unsigned integer length prefix followed by a protobuf-serialised `CastMessage`: [Source](https://github.com/athombv/node-castv2)

```protobuf
// cast_channel.proto (simplified)
message CastMessage {
  required ProtocolVersion protocol_version = 1;  // CASTV2_1_0
  required string source_id      = 2;  // e.g. "sender-0"
  required string destination_id = 3;  // e.g. "receiver-0"
  required string namespace_      = 4;
  required PayloadType payload_type = 5;
  optional string  payload_utf8   = 6;
  optional bytes   payload_binary = 7;
}
```

The key namespaces and their roles:

| Namespace | Purpose |
|---|---|
| `urn:x-cast:com.google.cast.tp.connection` | Virtual connection setup |
| `urn:x-cast:com.google.cast.tp.heartbeat` | `PING`/`PONG` keepalive |
| `urn:x-cast:com.google.cast.receiver` | `LAUNCH`, `STOP`, `GET_STATUS` |
| `urn:x-cast:com.google.cast.media` | `LOAD`, `PLAY`, `PAUSE`, `SEEK`, `STOP` |
| `urn:x-cast:com.google.cast.tp.deviceauth` | Certificate authentication |

### 7.3 Media Streaming Model

The sender passes a content URL in the `LOAD` message's `contentId` field, along with `streamType` (BUFFERED or LIVE) and `contentType` (MIME type). The Cast receiver application fetches that URL independently — the sender is not a proxy or relay. This means the content URL must be reachable from the Cast device directly (on the same LAN, or publicly accessible).

### 7.4 go2rtc as a Bridge

go2rtc is an open-source RTSP/media aggregator that bridges local streams to Google Cast and Android TV. It accepts RTSP, RTMP, HTTP-FLV, and other sources, transcodes or remuxes to HLS or MP4, and serves the result on a local HTTP endpoint that the Cast device can fetch. [Source](https://go2rtc.com/)

```yaml
# go2rtc.yaml — bridge a local RTSP camera to Chromecast
streams:
  camera1: rtsp://admin:pass@192.168.1.50/stream
```

go2rtc exposes the stream at `http://<host>:1984/api/stream.mp4?src=camera1`, which is passed to the Cast `LOAD` command as the `contentId`.

---

## 8. Avahi: mDNS/DNS-SD for Linux

Avahi is the standard mDNS/DNS-SD implementation on Linux, compatible with Apple's Bonjour. It implements RFC 6762 (mDNS) and RFC 6763 (DNS-SD). The current upstream release is 0.8 ('Dobro Jutro', February 2020). [Source](https://github.com/avahi/avahi)

### 8.1 Daemon Architecture

`avahi-daemon` wraps `libavahi-core` (the full standalone mDNS stack) and exposes two interfaces:

- **D-Bus**: `org.freedesktop.Avahi` on the system bus — used by applications that need service publication or browsing at runtime.
- **Unix socket**: `/run/avahi-daemon/socket` — used by `avahi-browse`, `avahi-resolve`, and similar CLI tools.

Three client library options exist for C/C++ applications:

| Library | Use case |
|---|---|
| `libavahi-client` | C wrapper over D-Bus (recommended for most apps) |
| `libavahi-gobject` | GObject wrapper for GNOME applications |
| `libavahi-core` | Embedded stack for appliances without a running daemon |

### 8.2 Service Publication C API

```c
// avahi-client/publish.h — publish a service
AvahiClient *client = avahi_client_new (
    avahi_threaded_poll_get (threaded_poll),
    AVAHI_CLIENT_NO_FAIL,   /* tolerate daemon not running at startup */
    client_callback, NULL, &error);

AvahiEntryGroup *group = avahi_entry_group_new (client, entry_group_callback, NULL);

avahi_entry_group_add_service (
    group,
    AVAHI_IF_UNSPEC, AVAHI_PROTO_UNSPEC,
    0,                          /* flags */
    "My AirPlay Receiver",      /* service name */
    "_airplay._tcp",            /* service type */
    NULL, NULL,                 /* domain, host (use defaults) */
    7000,                       /* port */
    "pk=...", "srcvers=220.1",  /* TXT records (variadic, NULL-terminated) */
    NULL);
avahi_entry_group_commit (group);
```

For collision resolution, `avahi_alternative_service_name()` generates a new name (appending a counter) that can be re-used in a retry. [Source](https://avahi.org/doxygen/html/)

### 8.3 Service Browsing C API

```c
AvahiServiceBrowser *browser =
    avahi_service_browser_new (
        client,
        AVAHI_IF_UNSPEC, AVAHI_PROTO_UNSPEC,
        "_googlecast._tcp",   /* service type */
        NULL,                 /* domain — NULL uses default */
        0,                    /* flags */
        browse_callback,
        NULL);
```

When `browse_callback` fires with `AVAHI_BROWSER_NEW`, call `avahi_service_resolver_new()` to resolve the service to a hostname and port. Server states must be checked: register services only in `AVAHI_SERVER_RUNNING` state; withdraw them on `AVAHI_SERVER_COLLISION` or `AVAHI_SERVER_REGISTERING`.

### 8.4 Static Services

For long-running daemons, Avahi supports static service registration via XML files in `/etc/avahi/services/`:

```xml
<!-- /etc/avahi/services/rygel.service -->
<?xml version="1.0" standalone='no'?>
<!DOCTYPE service-group SYSTEM "avahi-service.dtd">
<service-group>
  <name>Rygel Media Server</name>
  <service>
    <type>_upnp._tcp</type>
    <port>8200</port>
  </service>
</service-group>
```

GUPnP uses the system mDNS resolver (Avahi or Bonjour) for resource location on platforms where available, falling back to its own multicast socket.

---

## 9. Firewall and Network Considerations

### 9.1 DLNA Multicast Requirements

DLNA/UPnP requires UDP port 1900 open for SSDP multicast to `239.255.255.250`. The following `iptables` rules allow SSDP from the local LAN while rejecting ingress from other interfaces:

```bash
# Allow SSDP multicast in
iptables -A INPUT -i eth0 -d 239.255.255.250 -p udp --dport 1900 -j ACCEPT
# Allow SSDP unicast responses
iptables -A INPUT -i eth0 -p udp --sport 1900 -j ACCEPT
# Allow HTTP content delivery (Rygel default port is ephemeral; adjust as needed)
iptables -A INPUT -i eth0 -p tcp --dport 8200 -m state --state NEW,ESTABLISHED -j ACCEPT
```

DLNA multicast does not cross IP routers. All devices participating in the same DLNA network must be on the same Layer 2 subnet, or a multicast proxy (such as `udproxy` or `avahi-daemon --no-rlimits`) must bridge subnets.

### 9.2 Docker Networking

Docker bridge networking (`--network=bridge`) isolates containers from multicast by default, which silently breaks DLNA discovery. The correct mode for containerised DLNA servers (Jellyfin, Rygel) is `--network=host`. This is documented explicitly in Jellyfin's DLNA guide.

### 9.3 UPnP IGD — NAT Traversal

For applications that need to open external ports on a home router, `miniupnpc` provides a compact ANSI C client for the UPnP Internet Gateway Device (IGD) protocol. Current miniupnpd is 2.3.10 (March 2026). [Source](https://github.com/miniupnp/miniupnp)

```c
// miniupnpc/miniupnpc.h — typical IGD workflow
struct UPNPDev *devlist = upnpDiscover (2000, NULL, NULL, 0, 0, 2, &error);

struct UPNPUrls urls;
struct IGDdatas data;
char externalIP[40];
UPNP_GetValidIGD (devlist, &urls, &data, localIP, sizeof localIP);
UPNP_GetExternalIPAddress (urls.controlURL, data.first.servicetype, externalIP);

UPNP_AddPortMapping (
    urls.controlURL, data.first.servicetype,
    "8200",       /* external port */
    "8200",       /* internal port */
    localIP,      /* internal client */
    "Rygel",      /* description */
    "TCP",        /* protocol */
    NULL,         /* remote host — NULL = any */
    "0");         /* lease duration — 0 = indefinite */
```

`minissdpd` can run alongside `miniupnpc` to cache SSDP responses and reduce multicast traffic on networks with many UPnP devices.

---

## 10. Transcoding Pipeline Design

### 10.1 Format Negotiation

When a DLNA control point sends a Browse request, the server returns a `<res>` element for each available representation of a media item. Each `<res>` carries a `protocolInfo` attribute describing what the client would receive if it fetched that URI:

```xml
<res protocolInfo="video/mp4:DLNA.ORG_PN=AVC_MP4_HP_HD_AAC;DLNA.ORG_OP=01"
     size="1234567890"
     duration="1:32:15">
  http://192.168.1.10:8200/content/12345?transcode=0
</res>
<res protocolInfo="video/mpeg:DLNA.ORG_PN=MPEG_PS_NTSC;DLNA.ORG_OP=01"
     duration="1:32:15">
  http://192.168.1.10:8200/content/12345?transcode=mpeg_ps
</res>
```

The client fetches its preferred resource URI. If it chooses a transcoded URI, the server must spin up a GStreamer (or FFmpeg) pipeline before the first byte of the HTTP response body arrives.

### 10.2 GStreamer Autoplug Transcoding

Rygel's GStreamer engine uses `uridecodebin` for source decoding and builds the encode/mux chain based on the requested DLNA profile. GStreamer's `autoplug-continue` / `autoplug-select` signals allow the application to influence element selection (for example, to prefer hardware decoders):

```
uridecodebin uri="file:///media/source.mkv"
  → queue
  → [VA-API decode path: vaapidecodebin, or software: avdec_h264]
  → videoconvert
  → x264enc  bitrate=4000  key-int-max=30
  → mp4mux  streamable=true
  → appsink   ← bytes fed into HTTP response
```

For audio-only transcode to MP3:

```
uridecodebin → audioconvert → audioresample → lamemp3enc → filesink (or appsink)
```

### 10.3 Buffer Management and Time-Shift

For time-based seeking (`DLNA.ORG_OP=01`), the server must handle HTTP `Range` requests. For a live or buffered stream this requires either:

1. **Seekable GStreamer sink**: The pipeline writes to a temporary file; incoming Range requests map to `fseek()` on that file. Rygel uses this for its time-shift buffer.
2. **Ring buffer**: A circular memory buffer holds the last N seconds of transcoded output; Range requests within the buffer window are served from memory; older requests return 416.

The Simple engine omits this entirely — seekable delivery requires the full GStreamer engine.

### 10.4 DLNA Transfer Mode Headers

Clients signal their intent via HTTP request headers. The server should inspect and honour:

```
getcontentFeatures.dlna.org: 1        → include protocolInfo in response
transferMode.dlna.org: Streaming      → live AV content
transferMode.dlna.org: Background     → album art, thumbnails
transferMode.dlna.org: Interactive    → on-demand file transfer
```

Responses for AV content should include `Content-Type`, `Content-Length` (if known), and the `contentFeatures.dlna.org` header repeating the `protocolInfo` flags.

---

## 11. Security Concerns

### 11.1 UPnP Attack Surface

UPnP has no built-in authentication or access control. Any device on the local network can invoke ContentDirectory Browse, AVTransport Play, or — critically — IGD port-mapping actions. The IGD attack surface has historically been severe: malicious software on the LAN could open arbitrary external ports on a home router by sending UPnP SOAP calls to the gateway.

Known CVEs in miniupnpc's IGD XML parser: [Source](https://app.opencve.io/cve/?vendor=miniupnp_project)

| CVE | Affected versions | Vector |
|---|---|---|
| CVE-2015-6031 | Pre-1.9.20150917 | Buffer overflow in `igddescparse.c` — oversized XML element name |
| CVE-2017-8798 | — | Heap corruption via negative `chunksize` in `realloc` path |
| CVE-2017-1000494 | Pre-2.0 | Uninitialized stack variable in `NameValueParserEndElt` |

All three are triggered during the initial network discovery phase when parsing XML responses from UPnP servers, meaning a malicious server on the LAN can exploit a vulnerable UPnP client application.

### 11.2 DLNA Server Isolation

Recommended isolation practices:

- **VLAN segmentation**: Place DLNA servers and media-rendering devices on a separate VLAN from general-purpose workstations. Multicast between VLANs requires a controlled proxy.
- **Docker**: While `--network=host` is required for DLNA discovery, the container filesystem isolation limits damage from a compromised media server.
- **Disable IGD clients** on servers that do not need external port forwarding. Rygel and Jellyfin use `gupnp-igd` and miniupnpc respectively for optional IGD functionality — both can be disabled in configuration.

### 11.3 Jellyfin Authentication Model

Jellyfin's core API uses token-based authentication (API keys or username/password issuing a bearer token). The DLNA plugin assigns a `DefaultUserId` for renderer-initiated playback, which grants that user's library permissions to any device that connects without authentication. Administrators should create a dedicated restricted DLNA user rather than using an admin account as `DefaultUserId`.

### 11.4 AirPlay / UxPlay Security

UxPlay implements PIN-based pairing (4-digit PIN displayed on screen), HTTP MD5 Digest authentication, and device-ID access control lists. The RAOP2 AES transport key exchange is reverse-engineered and not independently verified. For a secure deployment, device-ID allow-listing combined with PIN pairing provides reasonable defence against rogue senders on the LAN.

---

## 12. Comparison: Plex, Jellyfin, Emby, and Rygel

| | **Rygel** | **Jellyfin** | **Emby** | **Plex** |
|---|---|---|---|---|
| **License** | LGPL/GPL | GPL-3.0 | Proprietary (since ~2018) | Proprietary |
| **Language / runtime** | Vala / GLib | C# / .NET 8 | C# / .NET | C# / .NET |
| **Web UI** | None | Full web app | Full web app | Full web app |
| **DLNA** | Native (core) | Optional plugin (10.9+) | Included | Included |
| **Hardware transcode** | Via GStreamer (VAAPI) | VA-API, NVENC, QSV (jellyfin-ffmpeg) | Premium subscription | Plex Pass subscription |
| **Transcoding engine** | GStreamer | jellyfin-ffmpeg | FFmpeg | FFmpeg |
| **Client apps** | Any DLNA renderer | Free apps (Android, iOS, web, Kodi plugin) | Free + Emby Premiere | Free + Plex Pass |
| **Telemetry** | None | None | Limited | Phones home to Plex servers |
| **Memory footprint** | Very low (daemon) | Moderate (.NET runtime) | Moderate | High |
| **GNOME/freedesktop integration** | Native (GUPnP, GStreamer, Tracker) | None | None | None |

**Key differentiators:**

- **Rygel** is the only server with native GUPnP/GSsdp integration, making it the natural choice when building on the GNOME or freedesktop stack, or when resource constraints prohibit a .NET runtime. It lacks a web UI and has a smaller client ecosystem.
- **Jellyfin** offers the broadest open-source feature set — hardware transcode, free clients, an active plugin ecosystem — without subscription requirements. DLNA is opt-in since 10.9. It performs well on single-board computers when hardware transcode is available.
- **Emby** started open-source and forked into Jellyfin; its free tier is now limited, with hardware transcoding locked behind Emby Premiere.
- **Plex** has the broadest client support (game consoles, smart TVs), but central authentication, phone-home behaviour, and hardware transcoding behind Plex Pass make it unsuitable for privacy-focused self-hosted deployments.

DLNA compliance testing is conducted by the DLNA organisation's Hapi test suite; as of this writing, Rygel has historically been the only Linux server to publish formal compliance test results. Jellyfin's DLNA plugin targets practical interoperability with common renderers (Kodi, smart TVs, AVR receivers) rather than formal certification.

---

## 13. Integrations

- **Chapter 38 — PipeWire**: When Kodi or UxPlay renders audio through PipeWire, the DMR audio output path terminates in a PipeWire sink node. Rygel's Playbin renderer plugin can be directed to output audio to a PipeWire sink for routing flexibility.
- **Chapter 215 — mpv as a DLNA Renderer Client**: mpv can act as a DLNA renderer (DMR) using the `mpv-mpris` plugin together with Rygel's MPRIS renderer bridge, allowing a DMC to push content from a DMS to an mpv instance on the same machine.
- **Chapter 218 — SRT/RTSP Bridges from Broadcast to Home Network**: SRT or RTSP streams originating from broadcast capture equipment can be bridged into DLNA/UPnP via Rygel's GstLaunch plugin (wrapping the stream in a GStreamer pipeline) or via go2rtc serving the stream as an HLS endpoint consumable by Chromecast.
- **Chapter 60 — V4L2 TV Capture Cards as Rygel/Kodi Sources**: V4L2-captured streams from DVB or HDMI capture cards can be exposed as DLNA MediaServer items through Rygel's GstLaunch plugin (`v4l2src → tsdemux → queue → tcpserversink`) or ingested into Kodi's PVR subsystem via a binary PVR add-on that wraps the V4L2 device.
