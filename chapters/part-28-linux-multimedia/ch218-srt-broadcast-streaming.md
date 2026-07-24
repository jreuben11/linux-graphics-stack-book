# Chapter 218: Broadcast Streaming on Linux: SRT, NDI, RTSP, and Live Production

> **Part**: Part XXVIII — Linux Multimedia
>
> **Audience**: Systems engineers and multimedia developers building live streaming pipelines, broadcast infrastructure, or low-latency contribution links on Linux.
>
> **Status**: First draft — 2026-07-22

---

## Table of Contents

1. [RTSP/RTP Fundamentals](#rtsprtp-fundamentals)
   - [1.1 What is RTSP?](#11-what-is-rtsp)
   - [1.2 What is RTP?](#12-what-is-rtp)
   - [RTSP Control Plane](#rtsp-control-plane)
   - [RTP and RTCP](#rtp-and-rtcp)
   - [Interleaved TCP Mode](#interleaved-tcp-mode)
2. [SRT: Secure Reliable Transport](#srt-secure-reliable-transport)
   - [ARQ and TSBPD](#arq-and-tsbpd)
   - [Connection Modes](#connection-modes)
   - [Encryption](#encryption)
   - [libsrt API](#libsrt-api)
3. [SRT in GStreamer](#srt-in-gstreamer)
   - [srtsrc and srtsink Elements](#srtsrc-and-srtsink-elements)
   - [Latency Tuning and Statistics](#latency-tuning-and-statistics)
4. [SRT in FFmpeg](#srt-in-ffmpeg)
   - [URI Handler and Options](#uri-handler-and-options)
   - [TS-over-SRT Broadcast](#ts-over-srt-broadcast)
5. [NDI: Network Device Interface](#ndi-network-device-interface)
   - [Discovery and Transport](#discovery-and-transport)
   - [SpeedHQ Codec](#speedhq-codec)
   - [GStreamer NDI Plugin](#gstreamer-ndi-plugin)
6. [OBS Studio on Linux](#obs-studio-on-linux)
   - [libobs Scene and Source Model](#libobs-scene-and-source-model)
   - [Encoder Plugins](#encoder-plugins)
   - [Streaming Outputs](#streaming-outputs)
7. [WebRTC-HTTP Ingestion and Egress: WHIP and WHEP](#webrtc-http-ingestion-and-egress-whip-and-whep)
   - [WHIP Protocol Flow](#whip-protocol-flow)
   - [WHEP Protocol Flow](#whep-protocol-flow)
   - [GStreamer whipclientsink and whepclientsrc](#gstreamer-whipclientsink-and-whepclientsrc)
8. [mediamtx: Protocol Router](#mediamtx-protocol-router)
   - [Architecture and Configuration](#architecture-and-configuration)
   - [Authentication Hooks](#authentication-hooks)
9. [GStreamer Live Pipeline Design](#gstreamer-live-pipeline-design)
   - [rtpbin for RTP Sessions](#rtpbin-for-rtp-sessions)
   - [Clock Synchronisation and Jitter Buffer](#clock-synchronisation-and-jitter-buffer)
   - [Pipeline Visualisation](#pipeline-visualisation)
10. [FFmpeg Live Transcoding](#ffmpeg-live-transcoding)
    - [lavfi Sources and Filter Graphs](#lavfi-sources-and-filter-graphs)
    - [Hardware Decode and HLS Output](#hardware-decode-and-hls-output)
11. [Low-Latency HLS and DASH](#low-latency-hls-and-dash)
    - [LL-HLS: Parts and Blocking Reload](#ll-hls-parts-and-blocking-reload)
    - [LL-DASH and CDN Design](#ll-dash-and-cdn-design)
12. [Hardware Encode Pipeline](#hardware-encode-pipeline)
    - [VA-API HEVC via V4L2 Capture](#va-api-hevc-via-v4l2-capture)
    - [Latency Budget Breakdown](#latency-budget-breakdown)
13. [Integrations](#integrations)

---

## RTSP/RTP Fundamentals

### RTSP Control Plane

The Real Time Streaming Protocol version 2.0 is defined by [RFC 7826](https://www.rfc-editor.org/rfc/rfc7826.html), published in 2016, replacing the earlier RFC 2326. RTSP is a text-based application-layer protocol modelled on HTTP, operating by default over TCP port 554. Its role is purely a control plane: it negotiates, starts, pauses, and tears down media streams, but does not carry media bytes itself.

The core method sequence for a unicast live stream is:

```
C→S  OPTIONS rtsp://server/stream RTSP/2.0
S→C  200 OK Public: OPTIONS, DESCRIBE, SETUP, PLAY, PAUSE, TEARDOWN

C→S  DESCRIBE rtsp://server/stream RTSP/2.0
S→C  200 OK Content-Type: application/sdp
     v=0
     o=- 0 0 IN IP4 192.0.2.1
     s=Live Stream
     m=video 0 RTP/AVP 96
     a=rtpmap:96 H264/90000
     a=fmtp:96 profile-level-id=42e01f;sprop-parameter-sets=...

C→S  SETUP rtsp://server/stream/track=0 RTSP/2.0
     Transport: RTP/AVP;unicast;client_port=5004-5005
S→C  200 OK Session: abcd1234
     Transport: RTP/AVP;unicast;client_port=5004-5005;server_port=6004-6005

C→S  PLAY rtsp://server/stream RTSP/2.0 Session: abcd1234
S→C  200 OK
```

`DESCRIBE` returns an SDP body describing all media tracks with codec, clock rate, and format parameters (`fmtp`). `SETUP` carries the `Transport` header which negotiates RTP/RTCP port pairs for UDP delivery (or interleaved channel IDs for TCP delivery). `PLAY` begins media flow; `PAUSE` suspends it without closing the session; `TEARDOWN` closes it.

### RTP and RTCP

RTP ([RFC 3550](https://datatracker.ietf.org/doc/html/rfc3550)) carries the actual media datagrams. The fixed 12-byte RTP header contains:

- **V** (2 bits): version, always 2
- **P**: padding flag
- **X**: extension header flag
- **CC** (4 bits): CSRC count
- **M** (marker bit): codec-specific; for video marks the last packet of a frame
- **PT** (7 bits): payload type index into the SDP `rtpmap`
- **Sequence number** (16 bits): increments by 1 per packet for loss detection
- **Timestamp** (32 bits): media-clock units (e.g., 90 000 Hz for video)
- **SSRC** (32 bits): synchronisation source identifier

The timestamp clock is media-specific: H.264 uses a 90 kHz clock, so one video frame at 30 fps advances the timestamp by 3000. Audio codecs typically use their sample rate.

RTCP (RFC 3550 §6) is the companion control protocol, running on the port immediately above the RTP port. Two critical packet types:

- **Sender Report (SR)**: carries a 64-bit NTP wall-clock timestamp paired with the current RTP timestamp, plus sender packet and octet counts. This mapping lets receivers synchronise audio and video streams to real time.
- **Receiver Report (RR)**: carries fraction lost (8-bit fixed-point), cumulative packets lost (24-bit), extended highest sequence received, interarrival jitter (32-bit in timestamp units), last SR timestamp, and delay since last SR. Used by senders for congestion feedback.

RTCP session bandwidth is negotiated to approximately 5% of the media session bandwidth (RFC 3550 §6.2), so on a 10 Mbit/s stream RTCP consumes around 500 kbit/s.

### Interleaved TCP Mode

RFC 7826 §14 defines a multiplexing mode in which RTP and RTCP are carried inside the existing RTSP TCP connection rather than in separate UDP flows. Each media packet is framed with a 4-byte header:

```
'$' (0x24)  |  channel_id (1 byte)  |  payload_length (2 bytes, big-endian)
```

followed by the RTP or RTCP payload. Even-numbered channel IDs carry RTP; odd-numbered carry RTCP. The interleaved mode eliminates UDP firewall traversal problems and avoids NAT state management, making it the default for consumer RTSP clients (e.g., IP cameras behind NAT). The trade-off is that TCP's head-of-line blocking introduces variable delivery jitter that UDP avoids; this matters for strict real-time playback and is one reason SRT is preferred for broadcast contribution links.

### 1.1 What is RTSP?

RTSP (Real Time Streaming Protocol) is an application-layer control protocol defined by RFC 7826 (RTSP 2.0) that governs the negotiation, initiation, and teardown of real-time media delivery sessions over IP networks. It operates primarily over TCP port 554 and acts as a session-control plane: RTSP messages select tracks, allocate transport channels, and issue play or pause commands, but carry no media bytes themselves. The actual media payload is delivered by a companion protocol — typically RTP — negotiated through the SDP body returned by the RTSP DESCRIBE method.

On Linux, RTSP appears in two broad roles. First, as a consumption protocol: IP cameras and ONVIF-compliant devices expose RTSP endpoints that GStreamer's `rtspsrc` element or FFmpeg's `rtsp://` URI scheme can receive and decode. Second, as a server-side delivery mechanism: tools such as mediamtx and GStreamer's `rtspsink` publish RTSP endpoints consumed by players and hardware decoders. RTSP session state is maintained on the server, identified by a session token returned in the SETUP response, which allows a single TCP connection to control multiple parallel media tracks (video, audio, metadata). This section focuses on the RTSP 2.0 control-plane message flow, its relationship to RTP transport, and the interleaved TCP mode that enables RTSP operation across firewalls and NAT devices.

### 1.2 What is RTP?

RTP (Real-time Transport Protocol), standardised in RFC 3550, is the datagram protocol that carries media payloads — video frames, audio samples, timed metadata — from sender to receiver in real-time streaming systems. RTP operates over UDP, taking advantage of UDP's low overhead and avoiding TCP's head-of-line blocking and unbounded retransmission delays that would introduce jitter in time-sensitive streams.

Each RTP packet carries a fixed 12-byte header containing a payload type index (mapped to a codec via the SDP `rtpmap` attribute), a sequence number for loss detection, a media-clock timestamp for playout scheduling, and a synchronisation source (SSRC) identifier that scopes statistics per sender. The companion protocol RTCP (RFC 3550 §6) provides out-of-band control: Sender Reports correlate RTP timestamps with wall-clock time for audio/video synchronisation, while Receiver Reports feed back loss fraction and jitter statistics to the sender for congestion awareness.

On Linux, RTP is the universal media transport beneath RTSP, WebRTC, and GStreamer's `rtpbin` subsystem. GStreamer provides a rich set of RTP payloader and depayloader elements — `rtph264pay`, `rtph264depay`, `rtpopuspay`, and others — that packetise codec bitstreams according to the relevant RFC payload format specifications. Understanding RTP sequence numbers, timestamps, and RTCP timing is essential for diagnosing loss, jitter, and synchronisation problems in live streaming pipelines built on Linux.

---

## SRT: Secure Reliable Transport

SRT ([github.com/Haivision/srt](https://github.com/Haivision/srt)) is an open-source application-layer protocol operating over UDP sockets. Originally developed by Haivision and open-sourced in 2017, it addresses the challenge of reliable, low-latency video delivery over unpredictable IP networks while keeping latency bounded and predictable — a property neither TCP (unbounded retransmit) nor plain UDP (no recovery) provides.

### ARQ and TSBPD

SRT implements Automatic Repeat reQuest via NAK (Negative Acknowledgement) control packets. When a receiver detects a gap in sequence numbers, it sends a NAK listing the missing ranges. The sender retransmits missing packets up to the configured latency window. In live mode (`SRTO_TRANSTYPE=SRTT_LIVE`), FASTREXMIT retransmits after RTT/2 without waiting for TCP-style exponential backoff, keeping recovery fast. In file mode, LATEREXMIT uses conservative TCP-like backoff.

Time-Stamp Based Packet Delivery (TSBPD) is the core real-time delivery mechanism. Each SRT data packet carries a 32-bit timestamp in microseconds relative to the connection start time. The receiver maintains a sorted receive buffer and delivers packets to the application at the exact moment their timestamp dictates:

```
PTS_deliver = TsbPdTimeBase + TsbPdDelay + PKT_TIMESTAMP + Drift
```

where `TsbPdTimeBase` accounts for the sender/receiver clock offset measured at connection start, `TsbPdDelay` is `SRTO_RCVLATENCY`, and `Drift` is a continuously corrected clock drift compensation term. [Source](https://srtlab.github.io/srt-cookbook/protocol/tsbpd/latency.html)

The latency window serves a dual purpose: it is both the maximum time a packet can wait for retransmission and the maximum time the receiver holds packets before the deadline. When `SRTO_TLPKTDROP=true` (default in live mode), packets whose timestamp is already past the delivery deadline are silently dropped rather than delivered late, ensuring the application always sees a monotonically advancing stream. [Source](https://haivision.github.io/srt-rfc/draft-sharabayko-srt.html)

Recommended latency configuration from Haivision documentation: `SRTO_LATENCY = 4 × RTT`. The default `SRTO_RCVLATENCY` is 120 ms for live mode. At 50 ms RTT, 4 × 50 = 200 ms provides comfortable headroom for two retransmit cycles. [Source](https://doc.haivision.com/SRT/1.5.4/Haivision/latency)

### Packet Structure

SRT defines two packet types distinguished by the high bit of the first 32-bit word. [Source](https://haivision.github.io/srt-rfc/draft-sharabayko-srt.html)

**Data packets** (type bit = 0) carry the following header fields:
- Bits 0: type flag (0 = data)
- Bits 1–31: sequence number (31-bit, monotonically increasing)
- Bits 32–33: packet position flags — `FF` (first fragment), `ML` (middle), `LF` (last fragment), `SB` (single complete message)
- Bit 34: order flag (`O`): in-order delivery required
- Bits 35–36: encryption key indicator (`KK`): 00 = not encrypted, 01 = even key, 10 = odd key — allows smooth key rotation without dropping frames
- Bit 37: retransmit flag (`R`): set on retransmitted packets so receivers can distinguish original from ARQ copies
- Bits 38–63: message number (26-bit)
- Bits 64–95: timestamp (32-bit, microseconds relative to connection start) — the value that drives TSBPD delivery
- Bits 96–127: destination socket ID (32-bit) — identifies the recipient socket for multiplexing multiple SRT streams over a single UDP port

**Control packets** (type bit = 1) carry:
- Bits 1–15: control type (0x0000 = HANDSHAKE, 0x0001 = KEEPALIVE, 0x0002 = ACK, 0x0003 = NAK, 0x0004 = CONGESTION_WARNING, 0x0005 = SHUTDOWN, 0x0006 = ACKACK, 0x0007 = DROPREQ, 0x7FFF = user-defined)
- Bits 16–31: subtype (0x0000 for most; 0x0001 for KMREQ, 0x0002 for KMRSP in key material exchange)

The 1316-byte default payload size (`SRTO_PAYLOADSIZE`) is exactly 7 × 188 bytes, aligning SRT packet boundaries with MPEG-TS packet boundaries — one SRT data packet carries exactly seven TS packets with no fragmentation, which simplifies stream alignment and error concealment at the receiver.

### Connection Modes

SRT supports three connection establishment modes, all negotiated during the handshake: [Source](https://github.com/Haivision/srt/blob/master/docs/API/API.md)

- **Caller** (client): calls `srt_connect()`. The caller initiates the handshake and takes the HSREQ role.
- **Listener** (server): calls `srt_listen()` then `srt_accept()` in a loop. Each accepted connection produces a new `SRTSOCKET`.
- **Rendezvous**: both parties call `srt_rendezvous()` simultaneously, connecting to each other's IP and port. A cookie exchange during the handshake determines which side wins the caller role. Rendezvous is useful when both peers are behind NAT and neither can act as a pure listener.

### Encryption

SRT encryption uses a two-level key hierarchy: [Source](https://vajracast.com/blog/srt-encryption-aes-256/)

1. The application provides a passphrase (10–80 characters).
2. The passphrase is run through PBKDF2 with a random salt to derive the Key Encrypting Key (KEK).
3. The listener generates a random Stream Encrypting Key (SEK).
4. The KEK wraps the SEK using AES key wrap (RFC 3394).
5. The wrapped SEK is transmitted in a Key Material (KM) message during the handshake.
6. All payload data packets are encrypted with AES in Counter (CTR) mode using the SEK.

Key length is configured via `SRTO_PBKEYLEN`: 16 bytes (AES-128), 24 bytes (AES-192), or 32 bytes (AES-256). Starting from SRT v1.6.0, AES-GCM mode (providing both encryption and authentication) is available as an alternative to CTR. Keys rotate periodically at `kmrefreshrate` packets (default 16384).

The KM exchange mechanism differs between handshake versions: in **HSv4** (legacy, SRT 1.2.x and earlier), the Key Material is exchanged as a separate control message sequence (`KMREQ`/`KMRSP`) after the handshake completes. In **HSv5** (current, SRT 1.3.0+), the KM is embedded directly in a handshake extension field, completing key exchange atomically with connection establishment and eliminating the additional round trip. Connections negotiated as HSv5 also support the `streamid` field and the `SRTO_PEERLATENCY` negotiation, making HSv5 the required minimum for modern broadcast deployments.

### libsrt API

The core libsrt C API ([API-functions.md](https://github.com/Haivision/srt/blob/master/docs/API/API-functions.md)) follows a socket-like model:

```c
// srt/srt.h — core types and API

// Initialise/cleanup the SRT library
int srt_startup(void);
int srt_cleanup(void);

// Socket lifecycle
SRTSOCKET srt_create_socket(void);
int srt_bind(SRTSOCKET u, const struct sockaddr* name, int namelen);
int srt_listen(SRTSOCKET u, int backlog);
SRTSOCKET srt_accept(SRTSOCKET lsn, struct sockaddr* addr, int* addrlen);
int srt_connect(SRTSOCKET u, const struct sockaddr* name, int namelen);
int srt_close(SRTSOCKET u);

// Options  (see SRTO_* enum in srt.h)
int srt_setsockopt(SRTSOCKET u, int level, SRT_SOCKOPT optname,
                   const void* optval, int optlen);
int srt_getsockopt(SRTSOCKET u, int level, SRT_SOCKOPT optname,
                   void* optval, int* optlen);

// Data transfer
int srt_sendmsg2(SRTSOCKET u, const char* buf, int len,
                 SRT_MSGCTRL* mctrl);
int srt_recvmsg2(SRTSOCKET u, char* buf, int len,
                 SRT_MSGCTRL* mctrl);

// Event polling (non-blocking I/O)
int srt_epoll_create(void);
int srt_epoll_add_usock(int eid, SRTSOCKET u, const int* events);
int srt_epoll_uwait(int eid, SRT_EPOLL_EVENT* fdsSet,
                    int fdsSize, int64_t msTimeOut);
```

Key socket options from [API-socket-options.md](https://github.com/Haivision/srt/blob/master/docs/API/API-socket-options.md):

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `SRTO_TRANSTYPE` | enum | `SRTT_LIVE` | Live or file mode |
| `SRTO_LATENCY` | int32 ms | 120 | Sets both rcv + peer latency |
| `SRTO_RCVLATENCY` | int32 ms | 120 | Receiver TSBPD delay |
| `SRTO_PEERLATENCY` | int32 ms | 0 | Requested peer latency |
| `SRTO_PASSPHRASE` | string | — | 10–80 char passphrase |
| `SRTO_PBKEYLEN` | int | 0 | 0/16/24/32 bytes |
| `SRTO_TLPKTDROP` | bool | true | Drop late packets in live mode |
| `SRTO_NAKREPORT` | bool | true | Periodic NAK retransmission |
| `SRTO_PAYLOADSIZE` | int | 1316 | 7 × 188-byte TS packets |
| `SRTO_OHEADBW` | int | 25 | Overhead bandwidth % for ARQ |

---

## SRT in GStreamer

### srtsrc and srtsink Elements

The GStreamer SRT plugin lives in `gst-plugins-bad`. Prior to GStreamer 1.16, separate `srtclientsrc`, `srtserversrc`, `srtclientsink`, and `srtserversink` elements existed. From 1.16 onwards these were unified into `srtsrc` and `srtsink` with a `mode` property. [Source](https://gstreamer.freedesktop.org/documentation/srt/index.html)

`srtsrc` properties: [Source](https://gstreamer.freedesktop.org/documentation/srt/srtsrc.html)

- `uri` — `srt://host:port` (with caller mode) or `srt://:port` (listener mode)
- `mode` — `caller`, `listener`, or `rendezvous`
- `localaddress` / `localport` — bind address for listener/rendezvous
- `latency` — TSBPD delay in milliseconds (default 125)
- `passphrase` / `pbkeylen` — encryption parameters
- `streamid` — up to 512-character opaque ID for stream multiplexing
- `wait-for-connection` — block pipeline start until a peer connects (default true for sink, false for source in listener mode)
- `keep-listening` / `auto-reconnect` — survive caller disconnection and reconnect
- `poll-timeout` — internal epoll timeout in milliseconds (default 1000)
- `stats` — read-only `GstStructure` containing SRT statistics (packets sent/received, retransmitted, dropped, RTT, bandwidth)

The GStreamer SRT elements expose a set of GLib signals for lifecycle management on listener-mode sockets: [Source](https://gstreamer.freedesktop.org/documentation/srt/srtsrc.html)

- `caller-added` — emitted after a new connection is accepted; carries the peer's `GSocketAddress`
- `caller-connecting` — emitted before accepting a connection; handler receives the peer address and stream ID and **returns a boolean**: `TRUE` accepts, `FALSE` rejects. This is the primary hook for per-connection access control in listener mode.
- `caller-rejected` — emitted when `caller-connecting` returns `FALSE` or when an error occurs during the handshake
- `caller-removed` — emitted when an accepted caller disconnects

In multi-caller listener mode (`keep-listening=true`), `srtsink` maintains a list of active callers and delivers each outgoing buffer to all of them. The `caller-rejected` signal is useful for logging authentication failures; the `caller-removed` signal for teardown cleanup in server implementations.

A minimal GStreamer broadcast contribution pipeline (MPEG-TS over SRT):

```bash
# gst-launch-1.0 — SRT contribution from camera
gst-launch-1.0 \
  v4l2src device=/dev/video0 ! \
  videoconvert ! \
  x264enc bitrate=5000 key-int-max=60 tune=zerolatency ! \
  h264parse ! \
  mpegtsmux name=mux ! \
  srtsink uri="srt://ingest.example.com:4200" mode=caller latency=200
```

### Latency Tuning and Statistics

When integrating srtsink/srtsrc into a production pipeline, three parameters dominate latency: the SRT `latency` property (TSBPD window), the downstream GStreamer jitter buffer (`rtpjitterbuffer` `latency`), and the encoder lookahead/GOP size. Reducing encoder latency with `tune=zerolatency` removes the lookahead buffer from x264; reducing SRT latency to the minimum viable for the network path cuts the TSBPD window but risks packet drops during congestion.

The `stats` property exposes a `GstStructure` readable via `g_object_get(element, "stats", &stats, NULL)` containing fields including `packets-sent`, `packets-received`, `packets-retransmitted`, `packet-loss-rate`, `rtt-ms`, `estimated-bandwidth-mbps`. These are surfaced from the underlying `srt_bistats()` / `srt_bstats()` libsrt calls. [Source](https://www.collabora.com/news-and-blog/blog/2018/02/16/srt-in-gstreamer/)

---

## SRT in FFmpeg

### URI Handler and Options

FFmpeg exposes SRT via the `srt://` URI scheme, implemented in `libavformat/libsrt.c`. [Source](https://ffmpeg.org/ffmpeg-protocols.html#srt) A critical difference from the libsrt API: **FFmpeg SRT latency is in microseconds, not milliseconds**. Setting `latency=400000` in an FFmpeg SRT URI means 400 ms, not 400 seconds.

Options are passed as query parameters:

```bash
# General URI form
srt://IP:port?option1=val1&option2=val2

# Key options
#   mode=caller|listener|rendezvous
#   latency=<microseconds>          (FFmpeg-specific! multiply ms by 1000)
#   passphrase=<string>
#   pbkeylen=16|24|32
#   streamid=<string>
#   maxbw=<bits/s>                  (0 = use inputbw + oheadbw)
#   oheadbw=<percent>               (default 25)
#   pkt_size=1316                   (7×188 for TS alignment)
#   transtype=live|file
#   tlpktdrop=1|0
#   nakreport=1|0
```

The container must be MPEG-TS for broadcast SRT: `-f mpegts`. FFmpeg does not support RTP-over-SRT directly; TS-over-SRT is the industry standard for contribution links.

### TS-over-SRT Broadcast

A complete test source using the libavfilter `lavfi` device (`smptebars` + `sine`) to generate SMPTE colour bars with a 1 kHz tone, encode to H.264, and push TS over SRT:

```bash
# ffmpeg — synthetic SMPTE bars + tone → H.264 → MPEG-TS → SRT caller
ffmpeg \
  -f lavfi -re -i smptebars=duration=60:size=1280x720:rate=30 \
  -f lavfi -re -i sine=frequency=1000:duration=60:sample_rate=44100 \
  -pix_fmt yuv420p \
  -c:v libx264 -b:v 1000k -g 30 -keyint_min 30 \
  -profile:v baseline -preset veryfast \
  -c:a aac -b:a 128k \
  -f mpegts 'srt://127.0.0.1:4200?pkt_size=1316'
```

[Source](https://srtlab.github.io/srt-cookbook/apps/ffmpeg.html)

RTSP-to-SRT bridging (pass-through remux without re-encode):

```bash
ffmpeg -i rtsp://camera.local/stream -c copy -f mpegts \
  'srt://ingest.example.com:9000?mode=caller&latency=200000'
```

For firewall traversal in environments where neither peer can open an inbound port, rendezvous mode connects both sides simultaneously:

```bash
# srt-live-transmit — UDP source to SRT rendezvous
srt-live-transmit udp://:5000 \
  'srt://peer.example.com:6000?mode=rendezvous'
```

The bandwidth overhead reserved for ARQ recovery is governed by `oheadbw` (default 25%). At a 10 Mbit/s video bitrate, SRT reserves 2.5 Mbit/s of retransmission bandwidth headroom. Setting `maxbw=0` with an explicit `inputbw` matching the actual stream rate ensures the overhead percentage applies correctly.

---

## NDI: Network Device Interface

NDI (Network Device Interface) is a proprietary IP media transport protocol developed by NewTek (now Vizrt). It targets studio LAN environments where wiring individual SDI cables is impractical, enabling any NDI-capable device to discover and receive any other NDI source on the subnet. [Source](https://en.wikipedia.org/wiki/Network_Device_Interface)

### Discovery and Transport

NDI sources broadcast their availability as mDNS service records (`_ndi._tcp.local`) via Avahi on Linux (runtime dependencies: `libavahi-common3`, `libavahi-client3`). NDI receivers call `NDIlib_find_create_v3()` to enumerate sources on the local multicast domain. For cross-subnet operation, NDI 4.0 introduced a Discovery Server to which sources register explicitly, eliminating reliance on multicast routing.

The NDI SDK for Linux is royalty-free for standard NDI but requires a commercial licence for the Advanced SDK. The SDK loads at runtime as `libndi.so` (or `libndisenders.so` for send-only), with the path overridable via the `NDI_RUNTIME_DIR_V5` environment variable.

NDI transport is TCP by default (low-latency unicast). UDP multicast with FEC and Multi-TCP modes are available. Standard NDI video is uncompressed or lightly compressed and consumes roughly 100 Mbit/s for 1080i/50.

### SpeedHQ Codec

Standard NDI uses SpeedHQ (SHQ), an intra-only DCT codec family designed for fast software encode/decode:

- **SHQ0**: 4:2:0 chroma subsampling, roughly equivalent to JPEG quality 90
- **SHQ2**: 4:2:2 chroma subsampling
- **SHQ7**: 4:2:2 high-quality mode

SpeedHQ is included in FFmpeg as the `speedhq` codec (decode only in mainline; the encoder requires the NDI SDK). At 1080i/50 the bitrate is approximately 100 Mbit/s, which is the network budget constraint for standard NDI on Gigabit Ethernet.

NDI|HX reduces the bitrate to 8–20 Mbit/s using H.264, and NDI|HX2 achieves 1–50 Mbit/s, both with higher encode latency than standard NDI.

### GStreamer NDI Plugin

The GStreamer NDI plugin ([gst-plugin-ndi](https://github.com/teltek/gst-plugin-ndi)) is implemented in Rust by Teltek. It has been tested against NDI SDK versions 4.0, 4.1, and 5.0. Elements:

- `ndisrc`: receives video and audio from an NDI source by name
- `ndisink`: exposes a GStreamer pipeline output as an NDI source, discoverable by other devices on the network
- A device provider enables GStreamer device monitoring to discover NDI sources via the standard `GstDeviceMonitor` API

Build dependencies are `libgstreamer1.0-dev` and `libgstreamer-plugins-base1.0-dev`; GStreamer 1.18 or later is required. Because the NDI SDK is not redistributable, the plugin links against it at runtime via `dlopen` using the path from `NDI_RUNTIME_DIR_V5`.

---

## OBS Studio on Linux

### libobs Scene and Source Model

OBS Studio is structured around `libobs`, the core shared library, with all user-visible functionality implemented in dynamically loaded plugin modules (`.so` files on Linux). [Source](https://deepwiki.com/obsproject/obs-studio/2.1-libobs-core)

The fundamental data type is `obs_source_t`. Sources represent any media producer:

- Video capture: V4L2 cameras via the `v4l2-input` plugin, screen capture via the `xshm-input` or `pipewire-capture` plugin
- Scene: an `obs_source_t` of type `OBS_SOURCE_TYPE_SCENE` containing `obs_sceneitem_t` objects, each wrapping another source with a 2D transform
- Browser source: a web page rendered via CEF (Chromium Embedded Framework), composited as a GPU texture

The data flow is: sources → scene graph → GPU composition (via libobs-opengl using EGL/OpenGL on Linux) → encoder → output muxer. [Source](https://docs.obsproject.com/backend-design)

Plugin entry points follow a well-defined contract: [Source](https://deepwiki.com/obsproject/obs-studio/4-plugin-system)

```c
// obs-module.h — plugin registration entry points
bool obs_module_load(void);     // register types, return true on success
void obs_module_unload(void);   // cleanup

// Registration structures
struct obs_source_info {
    const char *id;
    enum obs_source_type type;
    uint32_t output_flags;
    // Callbacks:
    void *(*create)(obs_data_t *settings, obs_source_t *source);
    void (*destroy)(void *data);
    obs_properties_t *(*get_properties)(void *data);
    void (*video_tick)(void *data, float seconds);
    void (*video_render)(void *data, gs_effect_t *effect);
};

obs_register_source(&source_info);
obs_register_encoder(&encoder_info);
obs_register_output(&output_info);
```

### Encoder Plugins

The `obs-ffmpeg` plugin provides hardware-accelerated encoders: [Source](https://deepwiki.com/obsproject/obs-studio/4.4-video-encoders)

- `ff_vaapi_h264`, `ff_vaapi_hevc`, `ff_vaapi_av1`: VA-API encoders for Intel and AMD GPUs, wrapping FFmpeg's `h264_vaapi`, `hevc_vaapi`, `av1_vaapi` codecs
- `ff_nvenc_h264`, `ff_nvenc_hevc`: NVENC via FFmpeg for NVIDIA GPUs

OBS 30.2 added native NVENC support on Linux using CUDA shared textures, bypassing the CPU readback path that previously added a full frame of latency. The GPU composites the scene into a shared texture, which the NVENC encoder reads directly via CUDA interop, eliminating the PCIe round-trip for frame readback.

The `encoder_packet` structure carries compressed output:

```c
// libobs/obs-encoder.h
struct encoder_packet {
    uint8_t          *data;
    size_t            size;
    int64_t           pts;
    int64_t           dts;
    int32_t           timebase_num;
    int32_t           timebase_den;
    enum obs_encoder_type type;
    bool              keyframe;
};
```

### Streaming Outputs

OBS supports multiple output protocols. For SRT, OBS provides a native SRT streaming service in Settings → Stream → Service → Custom → SRT. The SRT URL format accepted by OBS follows the libsrt convention: `srt://host:port?latency=200000&passphrase=...`. [Source](https://obsproject.com/kb/srt-protocol-streaming-guide)

Output services are registered through `obs_service_info`, which ties a service descriptor to encoder-configuration callbacks:

```c
// libobs/obs-service.h — service plugin registration
struct obs_service_info {
    const char *id;                   // e.g. "rtmp_custom", "srt_custom"

    // Return the ingest URL (e.g. "srt://ingest.example.com:9000")
    const char *(*get_url)(void *data);

    // Return the stream key or passphrase
    const char *(*get_key)(void *data);

    // Return the protocol name string
    const char *(*get_protocol)(void *data);

    // Called before encoding to let the service override encoder settings
    // (e.g. force keyframe interval, set maximum bitrate for the service)
    void (*apply_encoder_settings)(void *data,
                                   obs_data_t *video_encoder_settings,
                                   obs_data_t *audio_encoder_settings);

    // Standard obs_properties_t for the settings UI
    obs_properties_t *(*get_properties)(void *data);
};

obs_register_service(&service_info);
```

[Source](https://deepwiki.com/obsproject/obs-studio/4-plugin-system)

The `apply_encoder_settings` callback is particularly useful for SRT-based services: a broadcast contribution service can enforce CBR encoding and specific keyframe intervals required by the ingest point, independent of what the user has configured in the OBS encoder panel.

HLS output is available through the `obs-outputs` plugin with the HLS muxer. RTMP output uses `rtmp_output` wrapping librtmp. For multi-destination streaming (e.g., simultaneous SRT to an encoder and RTMP to a social platform), OBS 29+ supports multiple simultaneous outputs.

---

## WebRTC-HTTP Ingestion and Egress: WHIP and WHEP

### WHIP Protocol Flow

WHIP (WebRTC-HTTP Ingestion Protocol) became [RFC 9725](https://datatracker.ietf.org/doc/rfc9725/) in March 2025. It standardises browser-to-server WebRTC ingest using plain HTTP rather than a custom WebSocket signalling protocol.

Publisher flow:

1. Publisher sends `POST /path/to/stream/whip` with `Content-Type: application/sdp` body containing an SDP offer.
2. Server responds `201 Created` with `Content-Type: application/sdp` SDP answer and a `Location:` header containing the session URL.
3. Optional trickle ICE candidates are sent as `PATCH /path/to/stream/whip/SESSION_ID` with `Content-Type: application/trickle-ice-sdpfrag`.
4. ICE connectivity checks run; DTLS handshake establishes SRTP keys.
5. Media flows over ICE/DTLS-SRTP (encrypted RTP datagrams).
6. Publisher ends the session with `DELETE /path/to/stream/whip/SESSION_ID`.

WHIP eliminates the need for custom signalling servers for browser ingest, enabling OBS (with the obs-webrtc plugin or native WHIP support added in OBS 30) to publish directly to a WHIP endpoint.

### WHEP Protocol Flow

WHEP (WebRTC-HTTP Egress Protocol) mirrors WHIP for playback. As of mid-2025 it remains in IETF draft ([draft-ietf-wish-whep-01](https://datatracker.ietf.org/doc/html/draft-ietf-wish-whep-01)):

1. Player sends `POST /path/to/stream/whep` with SDP offer.
2. Server responds `201 Created` with SDP answer + `Location:` header.
3. Media flows from server to player via ICE/DTLS-SRTP.
4. Player terminates with `DELETE`.

The WHIP→WHEP architecture allows a browser publisher (or OBS) to ingest via WHIP, with the media server converting and distributing to WHEP consumers, RTSP clients, HLS players, and SRT destinations simultaneously.

### GStreamer whipclientsink and whepclientsrc

The GStreamer `rswebrtc` Rust plugin (in `gst-plugins-rs`) provides WHIP and WHEP client elements: [Source](https://gstreamer.freedesktop.org/documentation/rswebrtc/index.html)

- `whipclientsink`: sends a media pipeline's output to a WHIP endpoint. Accepts `audio/x-raw`, `audio/x-opus`, `video/x-raw`, VP8, H.264, VP9, H.265, and AV1. Also accepts GPU memory types: CUDA, OpenGL, NVMM, D3D11, and VA (for zero-copy GPU paths).
- `whepclientsrc`: receives media from a WHEP endpoint.
- `whipserversrc` / `whepserversink`: server-side counterparts for embedding WHIP/WHEP ingestion into a GStreamer pipeline.

`webrtcsink` provides a higher-level abstraction serving a fixed set of streams to multiple simultaneous WebRTC consumers with built-in congestion control (transport-wide CC), FEC, and per-consumer sandboxed pipelines.

---

## mediamtx: Protocol Router

### Architecture and Configuration

mediamtx ([github.com/bluenviron/mediamtx](https://github.com/bluenviron/mediamtx), formerly `rtsp-simple-server`) is a zero-dependency Go binary implementing a media routing layer. It accepts publishers on any protocol and distributes to readers on any protocol, performing on-the-fly remuxing as needed. Supported protocols: RTSP/RTSPS (default ports 8554/8322), RTMP/RTMPS (1935/1936), HLS (8888), WebRTC/WHIP/WHEP (8889), SRT (8890), and Media-over-QUIC.

The central concept is a **path**: a named slot to which exactly one publisher writes and any number of readers subscribe. Configuration lives in `mediamtx.yml`:

```yaml
# mediamtx.yml — path configuration
paths:
  live/camera1:
    # Accept SRT publisher with streamid publish:live/camera1
    source: publisher
    record: yes
    recordPath: /var/recordings/%path/%Y-%m-%d_%H-%M-%S-%f
    recordFormat: fmp4

    # Start FFmpeg transcoder when stream goes live
    runOnPublish: >
      ffmpeg -i rtsp://localhost:8554/$MTX_PATH
             -c:v libx264 -preset veryfast -b:v 2M
             -f mpegts srt://localhost:8890?streamid=publish:live/camera1_hq
    runOnPublishRestart: yes

  all_others:
    source: publisher
```

[Source](https://stable-learn.com/en/mediamtx-streaming-server/)

WebRTC internals use the Pion WebRTC library for DTLS-SRTP. The WHIP endpoint URL pattern is `/{path}/whip`; WHEP is `/{path}/whep`. [Source](https://deepwiki.com/bluenviron/mediamtx/3.4-webrtc-server)

For SRT, the stream ID encodes the role and path: `publish:stream_name:user:password` for publishers; `read:stream_name:user:password` for readers. mediamtx parses this stream ID to route the SRT connection to the correct named path and enforce authentication.

### Authentication Hooks

mediamtx supports three authentication backends: [Source](https://deepwiki.com/bluenviron/mediamtx/4.2-authentication-and-authorization)

1. **Internal**: plain-text, SHA256, or Argon2id password comparison configured in `mediamtx.yml`
2. **HTTP hook**: `POST` to an external URL with a JSON body containing the connection metadata (IP, path, protocol, action, user); a 2xx response allows; 4xx denies
3. **JWT/JWKS**: Bearer token validation against a JWKS endpoint

Path-level permissions can be set as `allow` (no auth), exact username list, wildcard (`*`), or regex (`~^admin.*`). This allows a single mediamtx instance to host public read-only streams alongside authenticated contribution paths.

---

## GStreamer Live Pipeline Design

### rtpbin for RTP Sessions

`rtpbin` is the main GStreamer element for managing one or more RTP sessions. Internally it combines `rtpsession` (RTCP SR/RR generation and processing), `rtpssrcdemux` (per-SSRC demultiplexing), `rtpjitterbuffer` (ordered delivery with configurable delay), and `rtpptdemux` (per-payload-type demultiplexing). [Source](https://gstreamer.freedesktop.org/documentation/rtpmanager/rtpbin.html)

Pad naming follows the pattern `recv_rtp_sink_%u` / `recv_rtcp_sink_%u` for input sessions and `recv_rtp_src_%u_%u_%u` (session/SSRC/PT) for demuxed output. Sending uses `send_rtp_sink_%u` for unpacketised RTP input and `send_rtp_src_%u` / `send_rtcp_src_%u` for the network-facing output.

A live H.264 broadcast pipeline using individual RTP streams (rather than TS-over-SRT) with jitter buffering:

```bash
# gst-launch-1.0 — v4l2 → H.264 RTP → rtpbin → UDP output
gst-launch-1.0 \
  rtpbin name=rtpb latency=100 \
  v4l2src device=/dev/video0 ! \
    videoconvert ! \
    x264enc bitrate=3000 tune=zerolatency key-int-max=30 ! \
    rtph264pay pt=96 ! \
    rtpb.send_rtp_sink_0 \
  rtpb.send_rtp_src_0 ! \
    udpsink host=192.0.2.10 port=5004 \
  rtpb.send_rtcp_src_0 ! \
    udpsink host=192.0.2.10 port=5005 sync=false async=false
```

For the receive side with RTCP-driven clock sync:

```bash
gst-launch-1.0 \
  rtpbin name=rtpb latency=200 ntp-sync=true \
  udpsrc port=5004 ! rtpb.recv_rtp_sink_0 \
  udpsrc port=5005 ! rtpb.recv_rtcp_sink_0 \
  rtpb. ! rtph264depay ! avdec_h264 ! autovideosink
```

### Clock Synchronisation and Jitter Buffer

`rtpjitterbuffer` operates in four modes selectable via the `mode` property: `none` (passthrough), `slave` (lock to sender clock via RTCP SR), `buffer` (add a configurable delay), and `synced` (absolute synchronisation using NTP wall clock from RTCP SR). The default `latency` is 200 ms. [Source](https://gstreamer.freedesktop.org/documentation/rtpmanager/rtpjitterbuffer.html)

When `do-retransmission=true`, the jitter buffer sends a `GstRTPRetransmissionRequest` event upstream when it detects a gap, enabling RTX (RTP retransmission, RFC 4588). The `rtx-delay` property controls how long the buffer waits before declaring a packet lost and requesting retransmission (default: -1, meaning use the maximum observed jitter delay).

For multi-stream synchronisation (e.g., audio and video at different RTP timestamps), `rtpbin` with `ntp-sync=true` and `rtcp-sync=true` uses the NTP–RTP timestamp mapping from RTCP SR packets to align all streams to a common wall clock.

### Pipeline Visualisation

GStreamer can export the live pipeline graph as a Graphviz DOT file, invaluable for debugging complex pipelines: [Source](https://gstreamer.freedesktop.org/documentation/gstreamer/debugutils.html)

```bash
# Shell: export DOT files when pipeline state changes
export GST_DEBUG_DUMP_DOT_DIR=/tmp/gst_dots
mkdir -p /tmp/gst_dots

# In application code (C):
# gst_debug_bin_to_dot_file(GST_BIN(pipeline),
#                            GST_DEBUG_GRAPH_SHOW_ALL,
#                            "pipeline_playing");

# Convert to PNG
dot -Tpng -o /tmp/pipeline.png /tmp/gst_dots/pipeline_playing.dot
```

The macro `GST_DEBUG_BIN_TO_DOT_FILE(bin, details, name)` can be called at any point during pipeline execution. GStreamer 1.26 introduced `gst-dots-viewer`, a web-based real-time pipeline visualisation tool that watches the DOT directory and renders interactive pipeline graphs in a browser. [Source](https://blogs.gnome.org/tsaunier/2025/05/16/gst-dots-viewer-a-new-tool-for-gstreamer-pipeline-visualization/)

---

## FFmpeg Live Transcoding

### lavfi Sources and Filter Graphs

The `lavfi` device (`-f lavfi`) provides synthetic audio/video sources via the libavfilter graph, useful for testing pipelines without physical hardware. Key sources:

- `smptebars`: SMPTE HD colour bars with configurable size, frame rate, and duration
- `testsrc`: animated test pattern with timestamp overlay
- `sine`: pure tone at a configurable frequency
- `aevalsrc`: expression-evaluated audio waveform

Complex filter graphs (`-filter_complex`) enable multi-input processing. A scaling + overlay pipeline for a branded broadcast feed (constructed example; adapt device, logo path, and SRT target for deployment):

```bash
ffmpeg \
  -re -f v4l2 -framerate 30 -video_size 1920x1080 -i /dev/video0 \
  -loop 1 -i logo.png \
  -filter_complex \
    '[0:v]scale=1920:1080[base];
     [1:v]scale=200:100[logo];
     [base][logo]overlay=W-w-10:10[out]' \
  -map '[out]' -map 0:a \
  -c:v libx264 -preset veryfast -b:v 4M \
  -c:a aac -b:a 128k \
  -f mpegts 'srt://ingest.example.com:9000?latency=200000'
```

### Hardware Decode and HLS Output

VA-API hardware decode (`-hwaccel vaapi`) can be combined with VA-API encode to keep frames on GPU memory throughout. The VA-API FFmpeg workflow is documented at [trac.ffmpeg.org/wiki/Hardware/VAAPI](https://trac.ffmpeg.org/wiki/Hardware/VAAPI):

```bash
# ffmpeg — VA-API transcode (RTSP in → HEVC VAAPI → MPEG-TS)
ffmpeg \
  -vaapi_device /dev/dri/renderD128 \
  -hwaccel vaapi -hwaccel_output_format vaapi \
  -i rtsp://camera.local/stream \
  -vf 'scale_vaapi=format=nv12' \
  -c:v hevc_vaapi -b:v 8M -profile:v main \
  -c:a copy \
  -f mpegts 'srt://ingest.example.com:9000?latency=200000'
```

`-hwaccel_output_format vaapi` keeps decoded frames in VAAPI surface memory. `scale_vaapi` runs the format conversion and scaling on the GPU. This avoids the PCIe round-trip of reading frames to system memory for software scaling.

For HLS output with the segment muxer (`-f hls`) — constructed example based on [FFmpeg HLS muxer documentation](https://ffmpeg.org/ffmpeg-formats.html#hls-1):

```bash
ffmpeg \
  -i rtsp://camera.local/stream \
  -c:v libx264 -preset veryfast -g 60 \
  -c:a aac \
  -f hls \
  -hls_time 2 \
  -hls_segment_type fmp4 \
  -hls_flags independent_segments+delete_segments \
  -hls_list_size 10 \
  -master_pl_name master.m3u8 \
  /var/www/hls/stream_%03d.m4s
```

The `avio` callback interface (`AVIOContext` with custom `read_packet`/`write_packet` function pointers) allows FFmpeg to write output to arbitrary destinations — S3-compatible object storage, memory buffers, or custom network protocols — without intermediate disk writes.

---

## Low-Latency HLS and DASH

### LL-HLS: Parts and Blocking Reload

Standard HLS achieves 15–30 second latency because clients must wait for complete segments (typically 6–10 seconds). Low-Latency HLS (LL-HLS, introduced in Apple's HLS specification and supported by FFmpeg since version 4.4) reduces this to 2–4 seconds through three mechanisms: [Source](https://www.mux.com/blog/low-latency-hls-part-2)

1. **Partial segments (parts)**: each segment is pre-announced in the playlist as `EXT-X-PART` entries with `DURATION` < 200 ms before the parent `EXT-X-SEGMENT` is complete. Clients can request and play parts as they are produced.

2. **Blocking playlist reload**: a client appending `?_HLS_msn=N&_HLS_part=P` to the playlist URL tells the server to hold the HTTP response until part P of segment N is available. The server responds immediately when ready, eliminating polling jitter.

3. **Manifest delta updates (`_HLS_skip=YES`)**: the server returns only new manifest lines rather than the full playlist, reducing HTTP response size for high-frequency polling.

CDN requirements: HTTP/2 or HTTP/3 for request multiplexing; streaming (chunked transfer) responses forwarded immediately rather than buffered; per-part caching. [Source](https://gcore.com/blog/optimizing-hls-dash-3sec)

FFmpeg LL-HLS production:

```bash
ffmpeg \
  -re -i input.ts \
  -c copy \
  -f hls \
  -hls_time 1 \
  -hls_segment_type fmp4 \
  -hls_flags independent_segments+delete_segments+split_by_time \
  -hls_fmp4_init_filename init.mp4 \
  -master_pl_name master.m3u8 \
  /var/www/ll-hls/stream_%08d.m4s
```

Note: Full LL-HLS support (blocking reload, `EXT-X-PRELOAD-HINT`) requires a capable origin server (nginx with the `nginx-vod-module` or a purpose-built packager like `go2rtc` or mediamtx's HLS packager) rather than plain FFmpeg file output.

### LL-DASH and CDN Design

Low-Latency DASH achieves similar 2–4 second latency via CMAF chunked transfer encoding. Segments are produced as a sequence of short chunks (`availabilityTimeOffset` in the MPD), and the HTTP response for the segment URL begins streaming before the full segment is complete. The player reads chunks as they arrive.

The MPD `availabilityTimeOffset` indicates how early (in seconds before the segment's nominal availability time) the first chunk is available. Setting it to `(segment_duration - chunk_duration)` achieves theoretical minimum latency.

CDN origin design for scalable live: [Source](https://gcore.com/blog/optimizing-hls-dash-3sec)

- Origin packager (FFmpeg + custom muxer, or mediamtx) runs on a low-latency server close to the ingest point
- CDN edge nodes pull from origin via HTTP; edge caching is configured with `Cache-Control: max-age=<part_duration>` on parts and aggressive TTLs on the manifest
- Push CDN (RTMP/SRT to CDN ingest points) eliminates the origin-pull round-trip for geo-distributed origins

---

## Hardware Encode Pipeline

### VA-API HEVC via V4L2 Capture

A production hardware encode pipeline from V4L2 capture to SRT output combines three Linux kernel subsystems: V4L2 (capture), VA-API (encode), and SRT (transport). The complete FFmpeg invocation:

```bash
# ffmpeg — V4L2 capture → VA-API HEVC → MPEG-TS → SRT
ffmpeg \
  -vaapi_device /dev/dri/renderD128 \
  -f v4l2 \
    -framerate 30 \
    -input_format mjpeg \
    -video_size 1920x1080 \
  -i /dev/video0 \
  -vf 'format=nv12,hwupload' \
  -c:v hevc_vaapi \
    -b:v 8M \
    -profile:v main \
    -rc_mode CBR \
  -f mpegts \
  'srt://ingest.example.com:9000?mode=caller&latency=200000&passphrase=broadcast_key_16&pbkeylen=16&pkt_size=1316'
```

[Source](https://trac.ffmpeg.org/wiki/Hardware/VAAPI)

For cameras delivering native NV12 (rather than MJPEG), the software `format=nv12` conversion can be eliminated:

```bash
ffmpeg \
  -vaapi_device /dev/dri/renderD128 \
  -hwaccel vaapi \
  -hwaccel_output_format vaapi \
  -f v4l2 -framerate 30 -input_format nv12 -video_size 1920x1080 -i /dev/video0 \
  -vf 'scale_vaapi=format=nv12' \
  -c:v hevc_vaapi -b:v 8M \
  -f mpegts 'srt://ingest.example.com:9000?latency=200000'
```

`hevc_vaapi` has been benchmarked at roughly 50% faster encode time than `h264_vaapi` on the same hardware for equivalent quality, due to HEVC's more efficient intra prediction implementation in most VA-API drivers. [Source](https://trac.ffmpeg.org/wiki/Hardware/VAAPI)

For a GStreamer equivalent using the VA-API GStreamer plugin:

```bash
# gst-launch-1.0 — V4L2 → VA-API HEVC → TS → SRT
gst-launch-1.0 \
  v4l2src device=/dev/video0 io-mode=mmap ! \
  video/x-raw,format=NV12,width=1920,height=1080,framerate=30/1 ! \
  vaapih265enc rate-control=cbr bitrate=8000 keyframe-period=30 ! \
  h265parse ! \
  mpegtsmux ! \
  srtsink uri="srt://ingest.example.com:9000" mode=caller latency=200 \
    passphrase="broadcast_key_16" pbkeylen=16
```

### Latency Budget Breakdown

A complete glass-to-glass latency budget for V4L2 capture → VA-API HEVC encode → SRT → network → decode → display:

| Stage | Latency | Notes |
|-------|---------|-------|
| Camera sensor exposure | ~16 ms | at 60 fps; 33 ms at 30 fps |
| V4L2 capture and DMA | ~1–5 ms | MMAP or DMABUF path |
| MJPEG→NV12 software decode | ~5–15 ms | skipped for native NV12 cameras |
| VA-API HEVC encode | ~33–100 ms | 1–3 frame pipeline depth in hardware |
| MPEG-TS muxing | ~1–2 ms | one PES packet per AU |
| SRT TSBPD buffer | 120–400 ms | `SRTO_RCVLATENCY`; set to 4× RTT |
| Network transit | RTT/2 | typically 5–50 ms on WAN |
| Receiver jitter buffer | 50–100 ms | for network jitter absorption |
| Decode (software) | ~20–50 ms | HEVC software decode |
| Decode (VAAPI) | ~10–33 ms | hardware decode path |
| Display compositor | ~8–16 ms | one vsync at 60 Hz |
| **Total (software decode)** | **~260–760 ms** | depending on SRT latency |
| **Total (hardware decode)** | **~210–650 ms** | VA-API decode path |

The dominant contributor is the SRT TSBPD buffer. For contribution links with well-characterised RTT (e.g., dedicated fibre), the latency can be set aggressively: 4 × 10 ms RTT = 40 ms SRT latency reduces total contribution latency to under 200 ms. Over unpredictable WAN paths, 200–400 ms SRT latency is typical. [Source](https://doc.haivision.com/SRT/1.5.4/Haivision/latency)

The SRT bandwidth overhead (`SRTO_OHEADBW=25` by default) must be provisioned on the contribution link. For an 8 Mbit/s HEVC stream, plan for 10 Mbit/s link capacity: 8 Mbit/s video + 2 Mbit/s ARQ headroom.

---

## Integrations

**Chapter 35 (VA-API)**: The VA-API hardware encode path described in the hardware pipeline section (`hevc_vaapi`, `h264_vaapi`) maps directly to the VA-API architecture — `vaCreateContext`, `vaCreateBuffer`, `vaEndPicture` calls under the hood of the FFmpeg and GStreamer VA-API wrappers. The V4L2→VA-API zero-copy path via DRM PRIME buffer sharing avoids system memory copies.

**Chapter 38 (PipeWire)**: OBS on Linux uses PipeWire as the preferred audio capture backend for the `pipewire-capture` source. The audio path in a live broadcast pipeline — microphone → PipeWire graph → OBS audio track → AAC encode → MPEG-TS mux → SRT — depends on PipeWire's low-latency graph scheduling. FFmpeg can open PipeWire sources via the `lavfi` `pipewire` input device when `libavdevice` is compiled with PipeWire support.

**Chapter 60 (V4L2 and Media Subsystem)**: The V4L2 capture layer provides raw frames to both FFmpeg (`-f v4l2`) and GStreamer (`v4l2src`). The M2M (Memory-to-Memory) API in V4L2 exposes hardware codecs on SoCs; on platforms with V4L2-based HEVC encoders (e.g., Raspberry Pi, Rockchip), the V4L2 M2M codec can replace VA-API for the encode stage while preserving the same MPEG-TS/SRT output path.

**Chapter 213 (WebRTC / VoIP Signalling)**: WHIP and WHEP converge the WebRTC signalling model with HTTP, reducing the impedance mismatch between browser WebRTC and server infrastructure. The ICE/DTLS-SRTP media plane in WHIP/WHEP is identical to that used in VoIP (SIP + WebRTC) — the same STUN/TURN server infrastructure, the same SRTP keying via DTLS handshake, and the same RTP/RTCP control plane described in this chapter.

**Chapter 217 (SRT-to-DLNA Bridge)**: A mediamtx instance receiving a broadcast SRT contribution stream can simultaneously serve it to DLNA/UPnP-capable home devices via an HLS output consumed by a DLNA-to-HLS bridge. The mediamtx path configuration (`runOnPublish`) can launch a DLNA announcer as soon as a stream goes live, enabling seamless broadcast-to-home distribution without manual re-encoding.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
