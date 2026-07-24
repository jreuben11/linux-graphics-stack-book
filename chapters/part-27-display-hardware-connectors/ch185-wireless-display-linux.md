# Chapter 185: Wireless Display Technologies on Linux

**Target audiences:** Linux desktop developers implementing wireless display support; network engineers building presentation infrastructure; system administrators deploying wireless display solutions in enterprise and education environments; multimedia developers constructing low-latency display streaming pipelines using GStreamer, PipeWire, and VA-API.

---

## Table of Contents

1. [Wireless Display Paradigms and Technical Trade-offs](#wireless-display-paradigms)
   - [1.1 What is Miracast (Wi-Fi Display)?](#11-what-is-miracast-wi-fi-display)
   - [1.2 What is Wi-Fi Direct?](#12-what-is-wi-fi-direct)
   - [1.3 What is WiGig (IEEE 802.11ad/802.11ay)?](#13-what-is-wigig-ieee-80211ad80211ay)
2. [Miracast / Wi-Fi Display Standard](#miracast-wifi-display)
   - [Wi-Fi Direct P2P Group Formation](#wifi-direct-p2p)
   - [RTSP Control Plane: M1–M16 Message Exchange](#rtsp-control-plane)
   - [Data Plane: MPEG-TS/RTP Transport](#data-plane)
3. [MiracleCast: Open-Source Miracast on Linux](#miraclecast)
   - [Architecture and Components](#miraclecast-architecture)
   - [wpa_supplicant P2P D-Bus API](#wpa-supplicant-p2p-dbus)
   - [GStreamer Encode Pipeline](#miraclecast-gstreamer)
4. [GNOME Network Displays](#gnome-network-displays)
5. [wpa_supplicant P2P In Depth](#wpa-supplicant-p2p-depth)
   - [GO Negotiation and Intent Values](#go-negotiation)
   - [Persistent Groups and IP Assignment](#persistent-groups)
   - [Scripting P2P via wpa_ctrl](#scripting-p2p)
6. [WiGig: IEEE 802.11ad and 802.11ay](#wigig-80211ad)
   - [60 GHz Band Characteristics](#60ghz-characteristics)
   - [wil6210 Kernel Driver](#wil6210-driver)
   - [Linux WiGig Display Status](#wigig-display-status)
7. [Network Display via VNC and RDP](#vnc-rdp)
   - [wayvnc and Wayland VNC](#wayvnc)
   - [Weston RDP Backend and xrdp](#weston-rdp)
   - [Latency Comparison Table](#latency-comparison)
8. [Chromecast Sender from Linux](#chromecast-linux)
   - [CASTV2 Protocol and mDNS Discovery](#castv2)
   - [mkchromecast Screen Mirroring](#mkchromecast)
9. [WebRTC-Based Wireless Display](#webrtc-wireless)
   - [XDG ScreenCast Portal and PipeWire](#xdg-screencast)
   - [GStreamer webrtcbin Pipeline](#gstreamer-webrtcbin)
10. [VA-API Hardware Encode for Low-Latency Wireless Display](#vaapi-encode)
    - [Zero-Copy DMA-BUF Capture Pipeline](#dmabuf-pipeline)
    - [Latency Budget Analysis](#latency-budget)
11. [Integrations](#integrations)

---

## Wireless Display Paradigms and Technical Trade-offs {#wireless-display-paradigms}

Wireless display on Linux divides into three structurally distinct paradigms, each with different kernel, userspace, and network requirements.

**Wi-Fi Direct / Miracast (peer-to-peer, no AP required):** The Wi-Fi Alliance's Wi-Fi Display (WFD) specification — marketed as Miracast — layers an RTSP control channel and an RTP/MPEG-TS media stream over an IEEE 802.11 peer-to-peer link. No access point is needed; one device acts as a "soft AP" Group Owner. The primary advantage is infrastructure independence, making it attractive for presentation rooms and consumer displays. Linux support exists through the MiracleCast project and GNOME Network Displays, though driver and kernel integration remains more complex than on Windows.

**WiGig / 60 GHz proximity display (IEEE 802.11ad and 802.11ay):** At 60 GHz, WiGig achieves multi-gigabit throughput (up to 7 Gbps on 802.11ad, over 40 Gbps on 802.11ay with MIMO) with sub-millisecond latency at the driver level, enabling lossless or near-lossless 4K display extension within approximately 10 metres of line-of-sight range. Beamforming is mandatory to compensate for high free-space path loss. Linux kernel support for 802.11ad hardware exists through the `wil6210` driver, though WiGig Display Extension (WDE) userspace software is immature.

**Network display streaming over LAN or internet (VNC, RDP, WebRTC, Chromecast):** Traditional remote-display protocols (VNC with RFB, Microsoft RDP) and newer streaming approaches (Google Cast / CASTV2, WebRTC with PipeWire screen capture) operate over any IP network and benefit from unlimited range, but incur higher latency due to encoding, network transport, and decode stacks.

### Trade-off Summary

| Paradigm | Typical Latency | Max Bandwidth | Range | Linux Maturity |
|---|---|---|---|---|
| Miracast (Wi-Fi Direct) | 50–100 ms | ~25 Mbps | 30 m | Partial (MiracleCast, GNOME ND) |
| WiGig 802.11ad (60 GHz) | <5 ms (driver) | 7 Gbps | 10 m LOS | Basic (wil6210 driver; WDE userspace absent) |
| WiGig 802.11ay | <1 ms (driver) | 40+ Gbps | 10 m LOS | Very limited |
| VNC over LAN | 20–50 ms | 100 Mbps | Unlimited | Mature (wayvnc, x11vnc) |
| RDP (RemoteFX) | ~20 ms | 100 Mbps | Unlimited | Good (weston-rdp, GNOME RDP, KRdp) |
| Chromecast mirror | 200–500 ms | 10–50 Mbps | Unlimited (LAN) | Functional (mkchromecast) |
| WebRTC (PipeWire) | 50–150 ms | Variable | Unlimited | Growing (GStreamer webrtcbin) |

Power consumption is an additional dimension: Miracast activates the 2.4/5 GHz radio continuously under P2P group ownership, WiGig 60 GHz chips are power-hungry during beamforming training, and network streaming delegates power cost to existing infrastructure radios.

### 1.1 What is Miracast (Wi-Fi Display)?

Miracast is the Wi-Fi Alliance's certification program and brand name for the Wi-Fi Display (WFD) standard, which enables a source device — such as a laptop or smartphone — to mirror or extend its screen to a sink device — such as a smart TV or wireless display adapter — without requiring an existing Wi-Fi access point. The Wi-Fi Display Technical Specification (currently version 2.1) defines both the radio connection method and the application-level streaming protocol.

At the protocol level, Miracast combines two distinct layers. The radio link is provided by Wi-Fi Direct (IEEE 802.11 P2P), which creates a peer-to-peer 802.11 network directly between source and sink. The application layer uses RTSP for session control (a handshake sequence of M1–M16 messages that negotiates codecs, resolutions, and transport parameters) and RTP over UDP for media delivery. The source encodes the display framebuffer as H.264 video multiplexed in an MPEG-TS container, streamed to the sink for decode and render. Mandatory support covers H.264 Baseline, Main, and High Profile at up to 1920×1080p60 and LPCM audio; AAC and Dolby Digital are optional extensions.

On Linux, Miracast has no native desktop integration comparable to Windows or Android. Support is provided by userspace projects: MiracleCast drives wpa_supplicant for the P2P radio layer and constructs GStreamer pipelines for encode and decode, while GNOME Network Displays provides a higher-level UI. A practical constraint is that a Miracast implementation must take exclusive control of the wireless interface, conflicting with NetworkManager's own wpa_supplicant instance. This chapter covers both the protocol mechanics and the Linux-specific software stack in detail.

### 1.2 What is Wi-Fi Direct?

Wi-Fi Direct is an IEEE 802.11 extension that allows two Wi-Fi devices to form a direct peer-to-peer connection without a traditional infrastructure access point. One device takes the Group Owner (GO) role and operates a software-defined access point; the other joins as a P2P client. Both devices use standard 802.11 frame formats, so no proprietary hardware is required beyond driver support for the GO state machine and the P2P management frame exchange.

Connection setup proceeds in three phases: device discovery (probe requests and responses on social channels 1, 6, and 11 to locate P2P-capable peers), GO negotiation (an exchange of intent values from 0 to 15 that determines which device becomes the GO, with a random tie-break bit), and WPS provisioning (PBC push-button or PIN-based credential exchange to authenticate the client). After provisioning, the GO runs a DHCP server on the `192.168.49.0/24` subnet and assigns the client an address. Miracast mandates that the WFD source act as or negotiate toward the GO role so that it controls the group channel.

Linux kernel support for Wi-Fi Direct is implemented in the `cfg80211` subsystem and exposed through `nl80211` netlink messages. Drivers must implement the P2P-relevant `cfg80211_ops` callbacks — including `start_p2p_device`, `remain_on_channel`, and `mgmt_frame_tx` — to participate in P2P management frame exchange. The wpa_supplicant daemon manages the P2P state machine and exposes control through the D-Bus interface `fi.w1.wpa_supplicant1.Interface.P2PDevice`. MiracleCast and GNOME Network Displays both drive wpa_supplicant through this interface to form and tear down Wi-Fi Direct groups.

### 1.3 What is WiGig (IEEE 802.11ad/802.11ay)?

WiGig is a family of IEEE 802.11 standards that operate in the 60 GHz millimeter-wave band, providing extremely high throughput at short range. IEEE 802.11ad (standardized in 2012) achieves up to approximately 7 Gbps physical-layer data rates using 2.16 GHz-wide channels in the 57–66 GHz band and a single spatial stream with OFDM and SC-PHY modulation modes. Its successor, IEEE 802.11ay, extends throughput beyond 40 Gbps through channel bonding (up to four 2.16 GHz channels aggregated) and MIMO, targeting uncompressed 4K and 8K display streaming.

The 60 GHz band has two properties critical to wireless display use. High available bandwidth enables uncompressed or near-lossless video transmission, eliminating the encode and decode latency that limits Miracast to roughly 50–100 ms end-to-end. The short propagation range (approximately 10 metres line-of-sight) and inability to penetrate walls confine signal to a docking or presentation area. These same characteristics impose a mandatory constraint: the high free-space path loss at 60 GHz requires beamforming, where transmitter and receiver electronically steer antenna arrays toward each other. Beam training and tracking must run continuously to maintain link quality as devices move.

In the Linux kernel, WiGig hardware is supported through the `wil6210` driver located at `drivers/net/wireless/ath/wil6210/`, which covers Qualcomm Atheros ath10k 60 GHz chipsets. The driver integrates with `cfg80211` and exposes the interface through `nl80211`. However, the WiGig Display Extension (WDE) application-layer software — analogous to MiracleCast in the Miracast ecosystem — is absent from mainline Linux distributions, leaving WiGig display as a hardware-capable but software-incomplete feature on Linux.

---

## Miracast / Wi-Fi Display Standard {#miracast-wifi-display}

The Wi-Fi Alliance's Wi-Fi Display (WFD) specification, version 2.1, defines both the peer-to-peer radio connection and the application-level streaming protocol. [Source: Wi-Fi CERTIFIED Miracast Technical Overview](https://tools.barco.com/kb-downloads/4814/Wi-Fi_CERTIFIED_Miracast_Technical_Overview_20170725.pdf)

### Wi-Fi Direct P2P Group Formation {#wifi-direct-p2p}

Miracast runs atop **Wi-Fi Direct** (IEEE 802.11 P2P), which allows two Wi-Fi stations to form an infrastructure-less network. One device becomes the **Group Owner (GO)** and runs a soft access point; the other joins as a P2P client. The GO negotiation determines which device takes the GO role and occurs in three frames: GO Negotiation Request, Response, and Confirmation.

The **GO Intent** value (0–15) signals a device's preference to become the GO. A device with intent 15 strongly prefers the GO role; intent 0 strongly prefers the client role. If both devices advertise the same intent value, a tie-break bit (set randomly per session) decides. The chosen GO activates an AP BSS on a negotiated channel (typically 2412 MHz / channel 1 for 2.4 GHz or a 5 GHz channel), assigns itself an IP from the `192.168.49.0/24` subnet, and starts a DHCP server for the P2P client.

In the **autonomous GO** mode, a device pre-empts negotiation and immediately raises a group. This is useful when the source always wants to be the GO (e.g., a laptop acting as a Miracast source to a TV).

**WPS provisioning** follows group formation to authenticate the P2P client. Two methods are common:

- **PBC (Push-Button Configuration):** User presses a button on both devices within a two-minute window.
- **PIN:** One device displays a PIN that the user enters on the other.

After WPS, the P2P client receives a DHCP lease from the GO. The IP addresses are typically in `192.168.49.0/24` with the GO at `192.168.49.1`. [Source: wpa_supplicant P2P documentation](https://w1.fi/wpa_supplicant/devel/p2p.html)

### RTSP Control Plane: M1–M16 Message Exchange {#rtsp-control-plane}

Once the IP link is established, the WFD source opens a TCP connection to port 7236 on the WFD sink and begins an RTSP-based capability negotiation sequence. The messages are numbered M1 through M16 and follow RTSP semantics (OPTIONS, GET_PARAMETER, SET_PARAMETER, SETUP, PLAY, PAUSE, TEARDOWN).

The key exchange sequence:

```
Source → Sink:  M1  OPTIONS rtsp://sink-ip:7236/wfd1.0 RTSP/1.0
                    (CSeq: 1)
                    (Require: org.wfa.wfd1.0)

Sink → Source:  M1  Response 200 OK
                    (Public: org.wfa.wfd1.0, GET_PARAMETER, SET_PARAMETER, SETUP, PLAY, PAUSE, TEARDOWN)

Sink → Source:  M2  OPTIONS rtsp://source-ip:7236/wfd1.0 RTSP/1.0
                    (Require: org.wfa.wfd1.0)

Source → Sink:  M2  Response 200 OK

Source → Sink:  M3  GET_PARAMETER rtsp://sink-ip:7236/wfd1.0 RTSP/1.0
                    Body:
                    wfd_client_rtp_ports
                    wfd_video_formats
                    wfd_audio_codecs
                    wfd_content_protection

Sink → Source:  M3  Response 200 OK
                    Body:
                    wfd_client_rtp_ports: RTP/AVP/UDP;unicast 1028 0 mode=play
                    wfd_video_formats: 00 00 02 02 0001DEFF 053C7FFF 00000FFF 00 0000 0000 00 none none
                    wfd_audio_codecs: AAC 00000001 00, LPCM 00000002 00
                    wfd_content_protection: none

Source → Sink:  M4  SET_PARAMETER rtsp://sink-ip:7236/wfd1.0 RTSP/1.0
                    Body: (negotiated parameters — resolution, framerate, codec)
                    wfd_video_formats: 00 00 02 02 00000020 00000000 00000000 00 0000 0000 00 none none
                    wfd_audio_codecs: LPCM 00000002 00
                    wfd_presentation_URL: rtsp://source-ip:7236/wfd1.0/streamid=0

Sink → Source:  M5  Response 200 OK (trigger: SETUP)

Sink → Source:  M6  SETUP rtsp://source-ip:7236/wfd1.0/streamid=0 RTSP/1.0
                    Transport: RTP/AVP/UDP;unicast;client_port=1028-1029

Source → Sink:  M6  Response 200 OK
                    Session: 1234567890;timeout=60

Sink → Source:  M7  PLAY rtsp://source-ip:7236/wfd1.0/streamid=0 RTSP/1.0
                    Session: 1234567890

Source → Sink:  M7  Response 200 OK
                    (RTP stream begins on UDP port 1028)
```

The `wfd_video_formats` parameter is a bitmask-encoded field specifying supported resolutions and frame rates. The encoding is defined in the WFD specification and covers CEA resolutions (up to 1920×1080p60), VESA modes, and HH (handheld) modes. [Source: Miracast HandWiki overview](https://handwiki.org/wiki/Miracast)

Post-session control messages include:

- **M8 PLAY** / **M9 PAUSE** — pause/resume the stream
- **M12** — standby mode
- **M13** — IDR frame refresh request (sink requests an Intra frame to recover from packet loss)
- **M14/M15** — UIBC (User Input Back Channel) selection and enable/disable

The **User Input Back Channel (UIBC)** allows the WFD sink (e.g., a smart TV) to send touch and HID input events back to the WFD source (e.g., a laptop), enabling interactive use of the mirrored screen. UIBC events are sent over a separate TCP connection.

### Data Plane: MPEG-TS/RTP Transport {#data-plane}

The media stream is carried as **H.264 (AVC) video** multiplexed in an **MPEG-2 Transport Stream (MPEG-TS)** container, packetised over **RTP** and sent as **UDP datagrams** to the sink's negotiated port (typically 1028).

Mandatory codec parameters:
- **Video:** H.264 Baseline, Main, or High Profile; up to 1920×1080p60; constant bit rate.
- **Audio:** LPCM 16-bit 48 kHz stereo (mandatory). AAC and AC3 are optional.

The RTP payload type for MPEG-TS is 33 (static assignment per [RFC 2250](https://www.rfc-editor.org/rfc/rfc2250)). Each RTP packet carries one or more 188-byte MPEG-TS packets. The typical encoded bitrate is 5–25 Mbps depending on resolution and negotiated quality.

Miracast resolution/frame rate table (subset):

| CEA ID | Resolution | Frame Rate | H.264 Level |
|---|---|---|---|
| 1 | 640×480 | 60p | 3.1 |
| 5 | 1280×720 | 30p | 3.2 |
| 7 | 1280×720 | 60p | 4.0 |
| 14 | 1920×1080 | 30p | 4.1 |
| 16 | 1920×1080 | 60p | 4.2 |

---

## MiracleCast: Open-Source Miracast on Linux {#miraclecast}

### Architecture and Components {#miraclecast-architecture}

[MiracleCast](https://github.com/albfan/miraclecast) (also actively maintained at [AtesComp/miraclecast](https://github.com/AtesComp/miraclecast)) is the primary open-source Miracast implementation for Linux. It targets both the source (sender) and sink (receiver) roles and depends on systemd ≥ 213, GStreamer, wpa_supplicant, and glib.

Core daemons and utilities:

- **`miracle-wifid`**: The Wi-Fi management daemon. Spawns and supervises a wpa_supplicant instance with a custom configuration that enables P2P support (`p2p_disabled=0`). Exposes a D-Bus API for P2P group management and forwards wpa_supplicant events to higher-level components.
- **`miracle-sinkctl`**: Interactive command-line tool to control the sink side. Listens for incoming WFD sessions, performs RTSP negotiation, and hands off decoded video to a display backend.
- **`miracle-gst`**: GStreamer helper used for media encode (source) and decode (sink). Constructs and launches GStreamer pipelines for H.264 encode with RTP packetisation (source side) or RTP depacetisation and video render (sink side).
- **`miracle-userctl`**: Scans for WFD peers and initiates connections.

The daemon topology follows a layered architecture:

```
miracle-userctl / miracle-sinkctl
        │ (D-Bus)
miracle-wifid
        │ (wpa_ctrl socket)
wpa_supplicant (P2P mode)
        │ (nl80211 / cfg80211)
kernel 802.11 P2P driver
```

A key operational constraint is that MiracleCast must take exclusive control of the wireless interface because wpa_supplicant cannot run twice on the same interface. This conflicts with NetworkManager's own wpa_supplicant instance. The workaround is to stop NetworkManager before starting MiracleCast, or to use a separate dedicated wireless interface. [Source: MiracleCast issue #415](https://github.com/albfan/miraclecast/issues/415)

**wpa.conf** required by MiracleCast (excerpt from [res/wpa.conf](https://github.com/albfan/miraclecast/blob/master/res/wpa.conf)):

```ini
# File: res/wpa.conf
ctrl_interface=/run/wpa_supplicant
ctrl_interface_group=0
update_config=1
device_name=MiracleCast
device_type=1-0050F204-1
p2p_disabled=0
```

The `device_type` field is the WPS primary device type in the format `category-OUI-subcategory`. `1-0050F204-1` identifies the device as a Computer → PC, which Miracast sinks use to display appropriate icons.

### wpa_supplicant P2P D-Bus API {#wpa-supplicant-p2p-dbus}

MiracleCast communicates with wpa_supplicant over the D-Bus interface `fi.w1.wpa_supplicant1`. The P2P-specific interface is `fi.w1.wpa_supplicant1.Interface.P2PDevice`, accessible on the path of the managed interface (e.g., `/fi/w1/wpa_supplicant1/Interfaces/0`). [Source: wpa_supplicant D-Bus API](https://w1.fi/wpa_supplicant/devel/dbus.html)

Key methods:

```
fi.w1.wpa_supplicant1.Interface.P2PDevice.Find(
    a{sv} args)
    → void
    # args: {"Timeout": uint32, "RequestedDeviceTypes": aay}
    # Alternates between P2P Search and Listen states

fi.w1.wpa_supplicant1.Interface.P2PDevice.StopFind()
    → void

fi.w1.wpa_supplicant1.Interface.P2PDevice.Connect(
    a{sv} args)
    → s pin
    # Required args: "peer" (object path), "wps_method" (string: "pbc"|"pin"|"display"|"keypad")
    # Optional: "persistent" (bool), "join" (bool), "frequency" (int32), "go_intent" (int32)

fi.w1.wpa_supplicant1.Interface.P2PDevice.GroupAdd(
    a{sv} args)
    → void
    # args: {"persistent": bool, "frequency": int32}
    # Starts autonomous GO without negotiation
```

Key signals emitted by `fi.w1.wpa_supplicant1.Interface.P2PDevice`:

```
DeviceFound(o peer_object)      # New P2P peer discovered
DeviceLost(o peer_object)       # Peer no longer visible
GroupStarted(a{sv} properties)  # Group formed; contains "interface_object", "role" ("GO"|"client"), "group_object"
GroupFinished(a{sv} properties) # Group dissolved
GONegotiationSuccess(a{sv})     # GO Negotiation completed successfully
GONegotiationFailure(a{sv})     # GO Negotiation failed
```

The `P2PDeviceConfig` property controls the local device's GO Intent:

```python
# Python example: read current P2P device configuration
import dbus
bus = dbus.SystemBus()
wpa = bus.get_object("fi.w1.wpa_supplicant1", "/fi/w1/wpa_supplicant1")
iface_path = wpa.GetInterface("wlan0",
    dbus_interface="fi.w1.wpa_supplicant1")
p2p_iface = bus.get_object("fi.w1.wpa_supplicant1", iface_path)
config = p2p_iface.Get(
    "fi.w1.wpa_supplicant1.Interface.P2PDevice",
    "P2PDeviceConfig",
    dbus_interface="org.freedesktop.DBus.Properties")
print(f"GO Intent: {config['GOIntent']}")   # 0–15
```

### GStreamer Encode Pipeline {#miraclecast-gstreamer}

On the source side, `miracle-gst` captures the compositor output and encodes it for streaming. A representative pipeline:

```bash
# GStreamer source pipeline for Miracast (miracle-gst internal)
gst-launch-1.0 -v \
  ximagesrc use-damage=0 ! \
  video/x-raw,framerate=30/1 ! \
  videoconvert ! \
  video/x-raw,format=I420 ! \
  vaapih264enc rate-control=cbr bitrate=5000 \
               cabac=true dct8x8=true \
               keyframe-period=15 ! \
  video/x-h264,profile=high,stream-format=byte-stream ! \
  mpegtsmux name=mux ! \
  rtpmp2tpay mtu=1316 ! \
  udpsink host=192.168.49.X port=1028 sync=false async=false
```

The `mtu=1316` setting ensures each RTP packet carries exactly 7 MPEG-TS packets (7 × 188 = 1316), the standard packing for MPEG-TS over RTP per [RFC 2250](https://www.rfc-editor.org/rfc/rfc2250). For Wayland compositors, `pipewiresrc` replaces `ximagesrc`; see [Section 10](#vaapi-encode).

Sink-side decode pipeline:

```bash
# GStreamer sink pipeline
gst-launch-1.0 -v \
  udpsrc port=1028 caps="application/x-rtp,media=video,clock-rate=90000,encoding-name=MP2T" ! \
  rtpmp2tdepay ! \
  tsdemux ! \
  h264parse ! \
  vaapih264dec ! \
  vaapisink
```

---

## GNOME Network Displays {#gnome-network-displays}

[GNOME Network Displays](https://gitlab.gnome.org/GNOME/gnome-network-displays) (also mirrored at [benzea/gnome-network-displays](https://github.com/benzea/gnome-network-displays)) is a GTK4 application that provides a polished GNOME front-end for wireless display casting. Version 0.91 added support for:

- **Miracast over Infrastructure (MICE):** Miracast transported over a conventional Wi-Fi access point rather than Wi-Fi Direct, eliminating the need for P2P group formation at the cost of requiring both devices on the same AP.
- **Chromecast:** Direct casting to Google Cast devices discovered via Avahi (mDNS).
- **Virtual screen:** Cast a virtual (headless) screen to avoid disrupting the user's physical session. [Source: Phoronix GNOME Network Displays 0.91](https://www.phoronix.com/news/GNOME-Network-Displays-0.91)

**Provider plugin architecture:** gnome-network-displays implements a provider abstraction with two concrete backends:

1. **`GndMiracastWFDProvider`** (Wi-Fi Display / Miracast): Uses wpa_supplicant for P2P group formation, then constructs an RTSP session and a GStreamer encode pipeline.
2. **`GndChromecastProvider`** (Google Cast): Uses Avahi for `_googlecast._tcp` mDNS discovery and pychromecast-compatible CASTV2 protocol messages for session control.

**GStreamer codec selection:** gnome-network-displays selects the H.264 encoder through environment variable overrides to accommodate different hardware:

```bash
# Use VA-API hardware encoder (Intel/AMD/NVIDIA)
NETWORK_DISPLAYS_H264_ENC=vaapih264enc gnome-network-displays

# Use software x264 fallback
NETWORK_DISPLAYS_H264_ENC=x264enc gnome-network-displays

# Use hardware AAC encoder
NETWORK_DISPLAYS_AAC_ENC=faac gnome-network-displays
```

The application uses **PipeWire** for screen capture via the XDG ScreenCast portal (see [Section 9](#webrtc-wireless)), which allows it to work on both X11 and Wayland sessions without compositor-specific code.

---

## wpa_supplicant P2P In Depth {#wpa-supplicant-p2p-depth}

### GO Negotiation and Intent Values {#go-negotiation}

P2P device discovery uses **Probe Request** frames that carry a **P2P Information Element (IE)** in the 802.11 management frame body. The P2P IE (OUI `50:6F:9A`, OUI type 0x09) encodes the device's capabilities, device name, primary device type, and supported channels. Peer devices in Listen state respond with Probe Responses carrying their own P2P IE.

The **GO Negotiation** exchange consists of three 802.11 Action frames (P2P Public Action frame category 0x04, OUI type P2P):

1. **GO Negotiation Request** — carries the initiator's GO Intent, configuration timeout, listen channel, P2P capability, and WPS configuration method.
2. **GO Negotiation Response** — the peer's GO Intent and capabilities.
3. **GO Negotiation Confirmation** — finalises the chosen channel and Operating Channel attribute.

If both intents are equal, the device whose tie-break bit is 1 becomes the GO. Intent 15 + tie-break bit set always wins the GO role. Intent 0 always loses unless the peer also offers intent 0, in which case the tie-break bit decides.

**Typical wpa_cli workflow for source (laptop):**

```bash
# Bring up the interface and start P2P discovery
wpa_cli -i wlan0 p2p_find

# List discovered peers
wpa_cli -i wlan0 p2p_peers
# Output: aa:bb:cc:dd:ee:ff

# Connect using PBC; the local device prefers GO role (go_intent=15)
wpa_cli -i wlan0 p2p_connect aa:bb:cc:dd:ee:ff pbc go_intent=15

# Alternatively, start an autonomous P2P group on 2.4 GHz channel 6
wpa_cli -i wlan0 p2p_group_add freq=2437

# List active groups
wpa_cli -i wlan0 p2p_group_status
```

wpa_supplicant creates a new virtual interface for the P2P group (typically `p2p-wlan0-0` or `p2p-dev-wlan0`) and starts a DHCP server (or client) on it automatically when the group forms.

### Persistent Groups and IP Assignment {#persistent-groups}

**Persistent groups** store the group credentials (SSID, passphrase, device addresses) in wpa_supplicant's configuration file after the first connection. On subsequent connections between the same two devices, the P2P GO Negotiation is replaced by an **Invitation Request/Response** exchange, which is faster (one round trip instead of three). The persistent group is identified by a network block in `wpa_supplicant.conf` with `mode=3` (P2P-GO) or `mode=4` (P2P-client).

IP address assignment within P2P groups:

- The GO allocates an IP from `192.168.49.0/24`, typically `192.168.49.1` for itself.
- Clients receive DHCP leases in the `192.168.49.x` range.
- wpa_supplicant can optionally perform IP address allocation internally (without an external DHCP daemon) using the `ip_addr_go` and `ip_addr_start`/`ip_addr_end` configuration options.

### Scripting P2P via wpa_ctrl {#scripting-p2p}

Applications that need programmatic P2P control without a D-Bus dependency can use the `wpa_ctrl` C library, which communicates with wpa_supplicant through a Unix domain control socket. [Source: wpa_supplicant control interface](https://w1.fi/wpa_supplicant/devel/ctrl_iface_page.html)

```c
/* Example: P2P peer discovery loop via wpa_ctrl
 * File: example_p2p_discover.c
 * Compile: gcc -o p2p_discover example_p2p_discover.c -lwpa_client
 */
#include <stdio.h>
#include <string.h>
#include "wpa_ctrl.h"

#define CTRL_SOCKET "/run/wpa_supplicant/wlan0"
#define BUF_SIZE    4096

int main(void) {
    struct wpa_ctrl *ctrl;
    char buf[BUF_SIZE];
    size_t len;
    int ret;

    /* Open control socket to wpa_supplicant */
    ctrl = wpa_ctrl_open(CTRL_SOCKET);
    if (!ctrl) {
        perror("wpa_ctrl_open");
        return 1;
    }

    /* Attach to receive unsolicited events */
    if (wpa_ctrl_attach(ctrl) != 0) {
        fprintf(stderr, "wpa_ctrl_attach failed\n");
        wpa_ctrl_close(ctrl);
        return 1;
    }

    /* Start P2P discovery */
    len = BUF_SIZE;
    ret = wpa_ctrl_request(ctrl, "P2P_FIND", 8, buf, &len, NULL);
    if (ret == 0) {
        buf[len] = '\0';
        printf("P2P_FIND: %s\n", buf);   /* Expect "OK" */
    }

    /* Poll for P2P-DEVICE-FOUND events */
    printf("Listening for P2P peers (Ctrl-C to stop)...\n");
    while (1) {
        if (wpa_ctrl_pending(ctrl) > 0) {
            len = BUF_SIZE;
            if (wpa_ctrl_recv(ctrl, buf, &len) == 0) {
                buf[len] = '\0';
                /* Event format: <3>P2P-DEVICE-FOUND aa:bb:cc:dd:ee:ff ... */
                if (strstr(buf, "P2P-DEVICE-FOUND"))
                    printf("Peer discovered: %s\n", buf);
                else if (strstr(buf, "P2P-GROUP-STARTED"))
                    printf("Group formed: %s\n", buf);
            }
        }
    }

    wpa_ctrl_detach(ctrl);
    wpa_ctrl_close(ctrl);
    return 0;
}
```

The `wpa_ctrl.h` and `wpa_client` library are distributed with the wpa_supplicant source (`src/common/wpa_ctrl.h`). Distributions package it separately (e.g., `libwpa-client-dev` on Debian/Ubuntu). [Source: wpa_supplicant source code](https://w1.fi/wpa_supplicant/devel/ctrl_iface_page.html)

---

## WiGig: IEEE 802.11ad and 802.11ay {#wigig-80211ad}

### 60 GHz Band Characteristics {#60ghz-characteristics}

The **60 GHz millimetre-wave (mmWave) band** (57–71 GHz globally, subject to regional spectrum allocation) offers dramatically higher channel bandwidth than 2.4 or 5 GHz Wi-Fi:

| Standard | Max PHY Rate | Max Net Throughput | Release |
|---|---|---|---|
| IEEE 802.11ad | 6.75 Gbps | ~4 Gbps usable | 2012 |
| IEEE 802.11ay | 40+ Gbps (4×4 MIMO) | ~20 Gbps usable | 2021 |

The physics impose stringent constraints:

- **Free-space path loss:** At 60 GHz, path loss is ~28 dB higher than at 2.4 GHz for the same distance. Effective range is limited to approximately 10 metres indoors under line-of-sight (LOS) conditions; walls and human bodies cause 10–40 dB additional attenuation, making non-LOS operation unreliable.
- **Beamforming (BF) required:** 802.11ad mandates BF training (Sector-Level Sweep, SLS) to identify the best transmit and receive antenna sectors. BF training adds latency at association and after link interruptions.
- **Channel plan:** 802.11ad defines four 2.16 GHz-wide channels (channels 1–4 at 58.32, 60.48, 62.64, and 64.80 GHz). Channel 2 (60.48 GHz) is globally available; others depend on regional regulations.
- **GCMP cipher:** The 60 GHz band's high data rate necessitates GCMP (Galois/Counter Mode Protocol) rather than the TKIP/CCMP used at lower frequencies.

These characteristics make WiGig ideal for **dock-to-laptop display extension** (replacing DisplayLink USB adapters and wired docks) where sub-5 ms latency and lossless display at 4K/60 Hz are achievable, but impractical for room-scale wireless presentation.

### wil6210 Kernel Driver {#wil6210-driver}

The Linux kernel includes the **`wil6210`** driver at `drivers/net/wireless/ath/wil6210/` for Qualcomm's WiGig chipsets. [Source: Linux kernel wil6210](https://github.com/torvalds/linux/blob/master/drivers/net/wireless/ath/wil6210/wil6210.h) [Source: wil6210 documentation](https://wireless.docs.kernel.org/en/latest/en/users/drivers/wil6210.html)

Supported hardware:

| PCIe ID | Chip Codename | Notes |
|---|---|---|
| `1ae9:0310` | Sparrow (Wilocity/QCA9005) | Mainstream 802.11ad; DW1601 card |
| `17cb:1201` | Talyn (Qualcomm QCA6435) | 802.11ay-capable, higher MIMO |

The driver uses the **cfg80211** framework (not mac80211, because the hardware handles most 802.11 MAC functions in firmware). Control commands are issued to the firmware via **WMI (Wireless Management Interface)** messages written to the BAR0 PCIe memory-mapped register space.

Key driver Kconfig option:

```kconfig
# In menuconfig: Device Drivers → Network device support → Wireless LAN → Atheros/AR9xxx
CONFIG_WIL6210=m
CONFIG_WIL6210_ISR_COR=y    # Clear-on-Read interrupt mode
```

The driver requires firmware blobs:
- `wil6210.brd` — board data file (calibration)
- `wil6210.fw` — main firmware image

These are distributed through `linux-firmware` and must be present in `/lib/firmware/` before the driver will initialize the hardware. The card will not function without the firmware. [Source: wil6210 archived documentation](https://cateee.net/lkddb/web-lkddb/WIL6210.html)

Supported 802.11ad channels in the kernel driver (channels 1–3 in the Linux 3.8+ driver; channel 4 added later):

```
Channel 1: 58320 MHz
Channel 2: 60480 MHz
Channel 3: 62640 MHz
```

Basic station mode and AP mode (up to 8 clients) are functional. The driver supports sniffer mode for 60 GHz frame capture, useful for debugging WiGig association failures.

### Linux WiGig Display Status {#wigig-display-status}

Despite hardware support through wil6210, **WiGig Display Extension (WDE)** userspace software is absent from the mainstream Linux ecosystem as of 2026. The WDE specification (a Miracast-like streaming layer over 802.11ad) requires:

1. A WiGig display controller on the sink device (e.g., a monitor with integrated WiGig receiver).
2. A WDE daemon to manage BF training, capability negotiation, and display stream setup.
3. Integration with the Linux DRM/KMS subsystem to present the incoming display as a virtual output.

None of these components exist in a production-ready form for Linux. The practical path for near-lossless wireless display on Linux today is therefore USB-C/Thunderbolt docking (wired) or the emerging DisplayPort-over-USB4 standard, rather than WiGig.

> **Note: needs verification.** Reports from Qualcomm and Intel WiGig SDK documentation suggest WDE was prototyped on Windows; Linux WDE support has not been publicly confirmed and may require proprietary firmware extensions not available in the open-source wil6210 driver.

---

## Network Display via VNC and RDP {#vnc-rdp}

### wayvnc and Wayland VNC {#wayvnc}

[wayvnc](https://github.com/any1/wayvnc) is the canonical VNC server for **wlroots-based Wayland compositors** (Sway, river, Hyprland, labwc, and others). It attaches to the running compositor session via the `wlr-export-dmabuf-v1` or `wlr-screencopy-v1` Wayland protocols, creates virtual input devices through `wlr-virtual-pointer-v1` and `zwp-virtual-keyboard-manager-v1`, and exposes the session over the RFB protocol. [Source: wayvnc manpage](https://manpages.ubuntu.com/manpages/noble/man1/wayvnc.1.html)

wayvnc does not use PipeWire for screen capture — it reads frames directly from the compositor via Wayland protocols, avoiding the XDG portal layer. This provides lower overhead but limits wayvnc to wlroots compositors specifically; it does not work on GNOME (Mutter) or KDE Plasma (KWin) sessions without modifications.

```bash
# Start wayvnc on all interfaces, port 5900
wayvnc 0.0.0.0 5900

# With TLS encryption (recommended over untrusted networks)
wayvnc --tls-cert=/etc/wayvnc/cert.pem \
       --tls-key=/etc/wayvnc/key.pem \
       0.0.0.0 5900

# Headless (no physical display): run a headless wlroots compositor
sway --config /etc/sway/headless.config &
wayvnc 0.0.0.0 5900
```

For GNOME sessions, GNOME Remote Desktop (`gnome-remote-desktop`) exposes both RDP and VNC interfaces using Mutter's screen capture APIs. For Wayland sessions that don't use wlroots, the XDG ScreenCast portal is the standardised capture path (see [Section 9](#webrtc-wireless)).

**On X11** sessions, x11vnc remains the standard:

```bash
# x11vnc: attach to the running X11 session on display :0
x11vnc -display :0 -auth guess -rfbport 5900 -noxdamage -forever
```

The `-noxdamage` flag disables XDamage tracking (which can cause corruption in some GPU configurations) and falls back to periodic full-frame capture.

Modern VNC implementations (TigerVNC, TurboVNC, libvncserver) support **JPEG** and **H.264 encoding** extensions (Tight encoding type 7, and the H.264 RFB extension in some implementations), significantly reducing bandwidth versus the baseline raw pixel encoding. TurboVNC uses libjpeg-turbo for SIMD-accelerated JPEG encoding.

### Weston RDP Backend and xrdp {#weston-rdp}

The **Weston RDP backend** (`rdp_backend.c` in the Weston source) provides an RDP compositor backend using the **FreeRDP** library for session management. Rather than rendering to a physical display, Weston renders to a GBM/EGL surface and streams the framebuffer to connected RDP clients. [Source: FOSDEM 2023 RDP on Wayland](https://archive.fosdem.org/2023/schedule/event/rdp_wayland/)

To start Weston with the RDP backend:

```bash
weston --backend=rdp-backend.so \
       --port=3389 \
       --cert=/etc/weston/tls.crt \
       --key=/etc/weston/tls.key
```

RDP clients connect using xfreerdp (for X11 host) or wlfreerdp (for Wayland host):

```bash
xfreerdp /v:192.168.1.100:3389 /w:1920 /h:1080 /bpp:32 /gfx:AVC444 /rfx
```

The `/gfx:AVC444` flag enables **RemoteFX AVC444 mode** (H.264 codec), which significantly improves bandwidth efficiency over raw pixel modes. FreeRDP on Linux supports hardware-accelerated H.264 decode via VA-API when the `vaapi` plugin is active.

**GNOME Remote Desktop** (backed by gnome-remote-desktop v46+) supports RDP natively for GNOME Wayland sessions, using PipeWire screen capture internally. **KRdp** provides the equivalent for KDE Plasma 6, exposing the Plasma session via RDP with hardware-accelerated encode.

**xrdp** connects incoming RDP sessions to existing Xorg or Weston desktop sessions through X11 sockets or Weston RDP. Its architecture separates the RDP frontend (`xrdp` daemon, port 3389) from session management (`xrdp-sesman`), allowing it to multiplex multiple user sessions.

### Latency Comparison Table {#latency-comparison}

| Protocol | Transport | Codec | Typical Latency | Notes |
|---|---|---|---|---|
| VNC (raw) | TCP/LAN | None (raw pixels) | 50–100 ms | Bandwidth-heavy; acceptable on Gbit LAN |
| VNC (Tight/JPEG) | TCP/LAN | JPEG | 30–50 ms | TigerVNC default |
| RDP (RemoteFX) | TCP/LAN | H.264 AVC444 | 15–25 ms | Best RDP latency |
| Miracast (WFD) | UDP/Wi-Fi Direct | H.264/MPEG-TS | 50–100 ms | No AP needed |
| Chromecast mirror | TCP+UDP/LAN | H.264 | 200–500 ms | High encode+decode chain |
| WebRTC (GStreamer) | DTLS-SRTP/LAN | VP8/H.264 | 50–150 ms | NACK recovery adds jitter |
| WiGig (802.11ad) | UDP/60 GHz | Uncompressed or H.264 | <5 ms | Driver level; WDE not in Linux |

---

## Chromecast Sender from Linux {#chromecast-linux}

### CASTV2 Protocol and mDNS Discovery {#castv2}

Google Chromecast devices advertise themselves on the local network via **mDNS** (multicast DNS, RFC 6762) as `_googlecast._tcp.local`. Each device broadcasts a DNS-SD TXT record containing:

- `id=` — 32-character unique device identifier
- `fn=` — friendly name (e.g., "Living Room TV")
- `md=` — model name
- `ve=` — protocol version
- `ca=` — capabilities bitmask (audio, video, multizone)

A Linux sender discovers Chromecast devices by subscribing to the `_googlecast._tcp.local` mDNS service using **Avahi** (the Linux mDNS/DNS-SD implementation) or the Python `zeroconf` library. [Source: gcast PROTOCOL.md](https://github.com/dylanmckay/gcast/blob/master/PROTOCOL.md)

Once the device IP and port (default 8009) are known, the sender opens a **TLS connection** (the Chromecast device presents a self-signed certificate). All further communication uses the **CASTV2 protocol**: each message is a 4-byte big-endian length prefix followed by a serialised **protobuf** message defined by the `cast.channel.CastMessage` type.

```protobuf
// cast_channel.proto (Google Cast SDK)
message CastMessage {
  required ProtocolVersion protocol_version = 1;
  required string source_id = 2;      // e.g., "sender-0"
  required string destination_id = 3; // e.g., "receiver-0" or transport_id
  required string namespace = 4;      // e.g., "urn:x-cast:com.google.cast.tp.connection"
  required PayloadType payload_type = 5;
  optional string payload_utf8 = 6;
  optional bytes payload_binary = 7;
}
```

Key CASTV2 namespaces used for screen mirroring:

| Namespace | Purpose |
|---|---|
| `urn:x-cast:com.google.cast.tp.heartbeat` | PING/PONG keepalive |
| `urn:x-cast:com.google.cast.tp.connection` | CONNECT / CLOSE |
| `urn:x-cast:com.google.cast.receiver` | Launch/stop apps, get/set volume |
| `urn:x-cast:com.google.cast.media` | LOAD, PLAY, PAUSE, SEEK for media |
| `urn:x-cast:com.google.cast.webrtc` | WebRTC-based screen mirroring (newer Cast) |

The sender instructs the receiver to launch a Cast application (identified by an App ID; the screen mirror app ID is `0F5096E8` for the Backdrop/Mirroring receiver). The receiver then exposes a WebRTC endpoint, and the sender establishes a WebRTC PeerConnection to stream the encoded display. [Source: pychromecast](https://github.com/home-assistant-libs/pychromecast)

### mkchromecast Screen Mirroring {#mkchromecast}

[mkchromecast](https://github.com/muammar/mkchromecast) is a Python application that casts audio and video from a Linux or macOS desktop to Google Cast devices. For screen mirroring, it uses ffmpeg to capture the display and encode it as a media stream, then starts a local HTTP server and instructs the Chromecast to load the stream URL.

```bash
# Install mkchromecast and dependencies
pip install mkchromecast pychromecast

# Mirror the screen (requires ffmpeg, Chromecast on same LAN)
mkchromecast --encoder-backend ffmpeg \
             --video \
             --video-codec h264 \
             --resolution 1920x1080 \
             --fps 30 \
             -n "Living Room TV"
```

Internally, mkchromecast runs an ffmpeg pipeline similar to:

```bash
# X11 screen capture and encode (mkchromecast internal ffmpeg command)
ffmpeg -f x11grab -framerate 30 -video_size 1920x1080 -i :0.0 \
       -c:v libx264 -preset ultrafast -tune zerolatency \
       -b:v 4000k -maxrate 4000k -bufsize 8000k \
       -pix_fmt yuv420p \
       -f mp4 -movflags frag_keyframe+empty_moov \
       http://localhost:5000/stream.mp4
```

For Wayland sessions, the `x11grab` input device must be replaced with a PipeWire-based capture (e.g., using `ffmpeg -f lavfi -i pipewire=...` or a PipeWire → GStreamer → ffmpeg pipeline). This is an area of active development.

The end-to-end latency in mkchromecast mirror mode is 200–500 ms, dominated by:

1. Display capture interval (33 ms at 30 fps)
2. H.264 software encode (30–80 ms with `ultrafast` preset)
3. HTTP server buffering (fragmented MP4 requires at least one fragment)
4. Network transfer to Chromecast (10–50 ms on LAN)
5. Chromecast receive buffer and decode (50–100 ms)

This latency is unsuitable for interactive use but adequate for presentation mirroring.

**pychromecast** provides the Python API for CASTV2 session management:

```python
import pychromecast

# Discover all Chromecast devices on the LAN
chromecasts, browser = pychromecast.get_chromecasts()
cast = next(c for c in chromecasts if c.name == "Living Room TV")
cast.wait()

# Launch the default media receiver
mc = cast.media_controller
mc.play_media("http://192.168.1.x:5000/stream.mp4",
              content_type="video/mp4")
mc.block_until_active()
print(f"Status: {mc.status}")
```

---

## WebRTC-Based Wireless Display {#webrtc-wireless}

### XDG ScreenCast Portal and PipeWire {#xdg-screencast}

The **XDG Desktop Portal** (`xdg-desktop-portal`) provides a compositor-agnostic D-Bus API for screen capture that works across GNOME, KDE Plasma, wlroots, and other Wayland compositors. The `org.freedesktop.portal.ScreenCast` interface (version 6 as of 2025) defines the following call sequence: [Source: xdg-desktop-portal ScreenCast portal](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.ScreenCast.html)

```
Application                          xdg-desktop-portal
     │                                      │
     │  CreateSession({handle_token: ...})  │
     │ ─────────────────────────────────►  │
     │  ◄─ Response(session_handle)         │
     │                                      │
     │  SelectSources(session, {           │
     │    types: MONITOR|WINDOW|VIRTUAL,   │
     │    multiple: false,                 │
     │    cursor_mode: EMBEDDED            │
     │  })                                 │
     │ ─────────────────────────────────►  │
     │  (compositor shows picker dialog)   │
     │                                      │
     │  Start(session, parent_window, {})  │
     │ ─────────────────────────────────►  │
     │  ◄─ Response(streams=[             │
     │       {pipewire-serial: 42,        │
     │        position: (0,0),            │
     │        size: (1920,1080),          │
     │        source_type: MONITOR}])     │
     │                                      │
     │  OpenPipeWireRemote(session, {})   │
     │ ─────────────────────────────────►  │
     │  ◄─ fd (PipeWire remote fd)        │
     │                                      │
     │  pw_context_connect_fd(fd)         │
     │  pw_stream_connect(node_id=42)     │
     │  (consume PipeWire video frames)   │
```

The `pipewire-serial` field (64-bit monotonically increasing, introduced in version 6) is preferred over the legacy `node_id` for stable stream identification across reconnects. Applications use `pw_context_connect_fd()` with the returned file descriptor to join the PipeWire graph, then connect a `pw_stream` to the node identified by `pipewire-serial`.

The `pipewiresrc` GStreamer element provides a convenient wrapper:

```bash
# Connect GStreamer to an XDG portal screen capture stream
gst-launch-1.0 pipewiresrc path=42 \
  ! video/x-raw,width=1920,height=1080,framerate=60/1 \
  ! videoconvert \
  ! vp8enc deadline=1 \
  ! rtpvp8pay pt=96 \
  ! udpsink host=192.168.1.100 port=5004
```

The `path` parameter accepts either the numeric PipeWire node ID or the node's `object.serial` value. PipeWire 1.4.8+ is required for reliable timestamp propagation; earlier versions suffer from buffer timestamp mismatches that cause frames to be dropped. [Source: GStreamer discourse - PipeWire to WebRTC](https://discourse.gstreamer.org/t/send-screencapture-data-from-pipewire-to-webrtc/5426)

### GStreamer webrtcbin Pipeline {#gstreamer-webrtcbin}

GStreamer's `webrtcbin` element implements a full WebRTC peer connection, handling ICE candidate gathering, DTLS-SRTP negotiation, and RTCP feedback. A complete pipeline for WebRTC-based wireless display:

```bash
# Sender: PipeWire screen capture → H.264 encode → WebRTC
gst-launch-1.0 -v \
  pipewiresrc path=42 \
  ! video/x-raw,width=1920,height=1080,framerate=30/1 \
  ! videoconvert \
  ! vaapih264enc rate-control=cbr bitrate=8000 \
                  keyframe-period=60 \
                  quality-level=5 \
  ! rtph264pay config-interval=1 pt=96 \
  ! application/x-rtp,media=video,encoding-name=H264,payload=96 \
  ! webrtcbin name=webrtc \
              stun-server=stun://stun.l.google.com:19302 \
              bundle-policy=max-bundle
```

The `config-interval=1` property on `rtph264pay` embeds SPS/PPS parameter sets in every keyframe RTP packet, ensuring the receiver can decode after any packet loss event that drops the initial keyframe.

**WebRTC NACK/RTCP feedback loop:** WebRTC's RTCP NACK mechanism allows the receiver to request retransmission of lost RTP packets. GStreamer's `webrtcbin` participates in this loop; when a NACK is received, the sender's `rtph264pay`/`rtprtxsend` elements retransmit the requested packet. For wireless display where packet loss on Wi-Fi is common (~1–5%), NACK recovery is essential to avoid visible corruption or decoder stalls.

For AV1 codec (higher efficiency, increasing hardware support):

```bash
# AV1 encode with hardware VA-API (Intel Gen12+ / AMD RDNA2+)
gst-launch-1.0 \
  pipewiresrc path=42 \
  ! videoconvert \
  ! vaav1enc rate-control=cbr bitrate=5000 \
  ! rtpav1pay pt=97 \
  ! webrtcbin name=webrtc stun-server=stun://stun.l.google.com:19302
```

The `webrtcsink` composite element (introduced in GStreamer 1.22) further simplifies signalling by bundling ICE, SDP, and a built-in signalling server:

```bash
gst-launch-1.0 \
  pipewiresrc path=42 \
  ! webrtcsink run-signalling-server=true run-web-server=true
```

This starts a built-in WHIP/WHEP-compatible HTTP signalling server on port 8080 and a WebSocket signalling server on port 8443, enabling receivers to connect via a web browser without custom signalling infrastructure.

---

## VA-API Hardware Encode for Low-Latency Wireless Display {#vaapi-encode}

### Zero-Copy DMA-BUF Capture Pipeline {#dmabuf-pipeline}

The highest-performance wireless display pipeline on Linux avoids any CPU copy of pixel data between the compositor output and the encoded bitstream. This is achieved by combining:

1. **KMS/DRM framebuffer export as DMA-BUF:** The compositor (or a screen capture backend) exports the current scanout buffer as a DMA-BUF file descriptor using `DRM_IOCTL_PRIME_HANDLE_TO_FD`.
2. **VA-API surface import from DMA-BUF:** `vaCreateSurfaces()` with `VASurfaceAttribMemoryType = VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2` and a `VADRMPRIMESurfaceDescriptor` imports the DMA-BUF as a VA surface without any data copy.
3. **Hardware H.264 encode:** `vaBeginPicture()`, `vaRenderPicture()`, `vaEndPicture()`, `vaSyncSurface()`, `vaMapBuffer()` produce the compressed H.264 NAL unit bitstream.
4. **RTP packetisation and UDP send:** The bitstream is sliced into RTP packets and sent to the sink.

[Source: Intel memory sharing with media](https://www.intel.com/content/www/us/en/docs/oneapi/optimization-guide-gpu/2024-1/memory-sharing-with-media.html)

In GStreamer, this zero-copy path is realised when:
- `pipewiresrc` provides buffers with `memory:DMABuf` capability negotiated.
- `vaapih264enc` accepts DMA-BUF input buffers directly via `GstDmaBufMemory`.

```bash
# Full zero-copy pipeline: PipeWire DMA-BUF → VA-API encode → RTP/UDP
gst-launch-1.0 -v \
  pipewiresrc path=42 \
              fd=3 \
  ! video/x-raw(memory:DMABuf),width=1920,height=1080,framerate=60/1,format=NV12 \
  ! vaapih264enc rate-control=cbr \
                  bitrate=20000 \
                  cabac=true \
                  dct8x8=true \
                  keyframe-period=120 \
                  quality-level=6 \
                  max-bframes=0 \
  ! video/x-h264,profile=high \
  ! rtph264pay config-interval=1 mtu=1316 \
  ! udpsink host=SINK_IP port=5004 sync=false async=false
```

The `max-bframes=0` property eliminates B-frames, which require frame reordering and add 1–2 frame latency. Combined with `max-bframes=0`, the `keyframe-period` controls the I-frame interval; shorter periods (30–60 frames) improve resilience to packet loss at the cost of slightly higher bitrate.

**`vaapih264enc` key properties for wireless display:**

| Property | Recommended Value | Effect |
|---|---|---|
| `rate-control` | `cbr` | Constant bit rate; predictable bandwidth |
| `bitrate` | 5000–20000 (kbps) | Trade quality vs bandwidth |
| `keyframe-period` | 30–120 | I-frame every 0.5–2 seconds at 60 fps |
| `max-bframes` | 0 | Eliminate B-frame delay |
| `cabac` | true | Better compression vs CAVLC |
| `quality-level` | 4–7 | Speed/quality tradeoff (higher = faster) |

[Source: GStreamer vaapih264enc documentation](https://gstreamer.freedesktop.org/documentation/vaapi/vaapih264enc.html)

### Latency Budget Analysis {#latency-budget}

For interactive wireless display (target: <100 ms end-to-end), the latency budget across the pipeline is:

| Stage | VA-API HW Encode | x264 Software (`ultrafast`) |
|---|---|---|
| Framebuffer capture (1 frame at 60 fps) | 16.7 ms | 16.7 ms |
| H.264 encode | **5–10 ms** | 30–80 ms |
| RTP packetisation + kernel UDP send | 0.5–1 ms | 0.5–1 ms |
| Wi-Fi transmission (802.11n/ac) | 2–10 ms | 2–10 ms |
| Decode (vaapih264dec) | 3–8 ms | 3–8 ms |
| Display scan-out | 8.3 ms (120 Hz) | 8.3 ms |
| **Total (typical)** | **~36–46 ms** | **~61–116 ms** |

The encode step is the dominant variable. VA-API hardware encode on Intel Gen9+ GPUs (via `i965` or `iHD` drivers) achieves 5–10 ms encode latency at 1080p60 with CBR rate control, compared to 30–80 ms for `x264` with `ultrafast`+`zerolatency` tuning. This places VA-API pipelines well within the 100 ms threshold below which latency becomes imperceptible for most presentation use cases. [Source: GStreamer low-latency telephony, GstConf 2023](https://indico.freedesktop.org/event/5/contributions/253/attachments/120/178/Low_Latency_Video_Telephony_using_gstreamer_gstconf2023_27sept.pdf)

The critical configuration choices for lowest encode latency:
- Disable B-frames (`max-bframes=0`): eliminates 1–2 frame reorder delay.
- Use CBR rate control: avoids VBR lookahead buffering.
- Set `quality-level` to 6–7 (fastest): accept some quality loss for speed.
- Use `slice-type=I` for `keyframe-period=1` when maximum resilience to packet loss is needed (at the cost of bitrate).

For the x264 software fallback used when VA-API is unavailable:

```bash
# x264 software encode with zero-latency tuning
gst-launch-1.0 \
  pipewiresrc path=42 \
  ! videoconvert \
  ! x264enc tune=zerolatency \
             speed-preset=ultrafast \
             bitrate=8000 \
             key-int-max=60 \
             bframes=0 \
             byte-stream=true \
  ! rtph264pay config-interval=1 \
  ! udpsink host=SINK_IP port=5004 sync=false
```

The `tune=zerolatency` option in x264 sets `b_vbv_delay_compensation=1`, disables lookahead, and enables slice-level encoding, minimising encoder-internal buffering at the cost of ~10–20% coding efficiency.

---

## Integrations {#integrations}

**Ch22 — Wayland Compositors:** The screen capture source for all Miracast, WebRTC, and network display pipelines described in this chapter originates at the Wayland compositor. Compositors export framebuffers via the `wlr-export-dmabuf-v1` protocol (wlroots) or the XDG ScreenCast portal (GNOME/KDE), both of which deliver DMA-BUF handles usable by the VA-API zero-copy encode pipeline.

**Ch26 — VA-API Hardware Video Encode:** The low-latency wireless display pipeline relies on VA-API for H.264 and AV1 hardware encode. Chapter 26 details the VA-API encode API (context creation, picture parameters, rate control), buffer lifecycle, and driver-specific capabilities (Intel iHD, AMD Radeon SI, NVIDIA VDPAU-VA bridge).

**Ch38 — PipeWire Video Pipeline:** PipeWire underpins the XDG ScreenCast portal transport. Chapter 38 covers the PipeWire graph model, `pw_stream` API, buffer negotiation for DMA-BUF, and the PipeWire SPA (Simple Plugin API) used by GStreamer's `pipewiresrc` element.

**Ch57 — FFmpeg:** FFmpeg provides an alternative encode backend for Miracast and Chromecast pipelines (particularly mkchromecast). Chapter 57 covers FFmpeg's VAAPI encoder (`h264_vaapi`, `av1_vaapi`), x11grab/Wayland capture inputs, and low-latency pipeline options.

**Ch58 — GStreamer:** GStreamer is the primary pipeline framework for wireless display streaming on Linux. Chapter 58 details element negotiation, buffer lifecycle, the `appsrc`/`appsink` escape hatches, and debugging techniques applicable to the Miracast and WebRTC pipelines here.

**Ch79 — Remote Display and Screen Casting (LAN Focus):** Chapter 79 covers LAN-based display protocols (VNC, RDP, SPICE) in depth, including multi-monitor support and clipboard synchronisation. This chapter focuses specifically on wireless transport standards (Miracast, WiGig, Chromecast) and the Linux software stacks that implement them.

**Ch123 — Screen Capture and Remote Desktop:** Chapter 123 details the portal-based screen capture architecture, permission model, and compositing layer interactions, which form the upstream input to wireless display pipelines.

**Ch146 — WebCodecs and Browser Hardware Acceleration:** On the receiver side of WebRTC-based wireless display, the browser's WebCodecs API (exposed via `VideoDecoder`) can consume the incoming H.264 or AV1 stream with hardware acceleration. Chapter 146 covers `VideoDecoder`, buffer queue management, and the GPU decode path on Linux.

---

## Roadmap

### Near-term (6–12 months)
- **GNOME Network Displays Miracast-over-Infrastructure (MICE) stabilisation:** The 0.91 release introduced MICE support; ongoing work targets reliable multi-monitor virtual-screen casting and improved codec negotiation, with patches under review on the GNOME GitLab tracker.
- **PipeWire 1.4.x timestamp fixes for wireless display:** PipeWire maintainers are merging buffer-timestamp propagation improvements needed for sub-frame latency accuracy in `pipewiresrc`-based Miracast and WebRTC pipelines, addressing drop issues that appear at 60+ fps capture rates.
- **GStreamer `webrtcsink` WHIP/WHEP integration maturation:** GStreamer 1.24/1.26 development cycles are stabilising the `webrtcsink` composite element's built-in signalling server and adding `webrtcsrc` for the receiver path, enabling turnkey browser-based wireless display without external signalling infrastructure.
- **KRdp hardware-accelerated encode (KDE Plasma 6.x):** KDE's KRdp project is adding VA-API H.264 and HEVC encode support via GStreamer's VA plugin, reducing CPU load for RDP-based Plasma session casting on AMD and Intel GPUs.

### Medium-term (1–3 years)
- **Miracast Wi-Fi 6 / Wi-Fi 7 profiles:** The Wi-Fi Alliance is drafting WFD 3.0 specifications that extend Miracast to Wi-Fi 6 (802.11ax) multi-link operation and Wi-Fi 7 (802.11be), targeting <20 ms end-to-end latency and 4K/120 Hz support; Linux open-source implementations will need corresponding wpa_supplicant and GStreamer pipeline updates.
- **AV1 hardware encode adoption in wireless display pipelines:** As Intel Meteor Lake, AMD RDNA3+, and NVIDIA Ada hardware AV1 encoders become mainstream, wireless display tools (GNOME Network Displays, mkchromecast replacements, WebRTC paths) are expected to default to AV1 for equivalent quality at 30–40% lower bitrate versus H.264.
- **WiGig 802.11ay dock-to-laptop display revival:** Intel and Qualcomm have signalled renewed interest in WiGig docking for laptop form factors; a coherent Linux WDE stack would require a userspace WiGig display daemon and DRM virtual connector integration, likely modelled on the DisplayPort Tunneling architecture used for USB4.
- **xdg-desktop-portal v7 multi-stream and cursor improvements:** Planned portal revisions aim to expose multiple simultaneous capture streams per session and reliable cursor position metadata, closing gaps that currently require compositor-specific workarounds in Miracast and RDP backends.

### Long-term
- **DisplayPort Tunnelling over Wi-Fi 7 (native lossless display):** Wi-Fi 7's deterministic latency (OFDMA scheduling, multi-link diversity) makes native DisplayPort tunnelling over wireless feasible, potentially replacing MPEG-TS/H.264 streaming with a direct pixel-transport path analogous to USB4 Alt Mode — eliminating encode/decode latency entirely for short-range scenarios.
- **Unified wireless display abstraction in PipeWire:** The PipeWire project's long-term vision includes a device-graph model that could abstract wired (DRM/KMS) and wireless (Miracast, WebRTC, RDP) outputs as first-class nodes, allowing compositors to treat a Miracast sink identically to a physical connector without per-compositor display protocol integration.
- **Converged open-source WiGig/Wi-Fi 7 stack with WDE support:** As 802.11ay silicon matures and kernel cfg80211/mac80211 gains multi-link management, a community WiGig Display Extension daemon could emerge, targeting dock-to-laptop and monitor-to-laptop use cases with sub-5 ms latency, completing the unfinished work noted in the `wil6210` driver's current WDE-absent status.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
