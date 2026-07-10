# Chapter 60c: WebRTC Server Infrastructure on Linux

**Target audiences:** Backend infrastructure engineers building real-time media platforms; streaming platform developers integrating SFU/MCU components; application developers designing multi-party video systems on Linux. This chapter assumes familiarity with the WebRTC protocol stack — SDP negotiation, ICE, DTLS-SRTP, RTP/RTCP, and GoogCC bandwidth estimation — all covered in Ch60b. The focus here is the server-side infrastructure that makes WebRTC scale beyond peer-to-peer: TURN relay, selective forwarding, recording, and production deployment on Linux.

## Table of Contents

1. [Why WebRTC Needs Server Infrastructure](#why-webrtc-needs-server-infrastructure)
2. [coturn: STUN/TURN Server](#coturn-stunturn-server)
3. [Janus WebRTC Gateway](#janus-webrtc-gateway)
4. [mediasoup: Node.js SFU with C++ Worker](#mediasoup-nodejs-sfu-with-c-worker)
5. [LiveKit: Go-Based SFU with Rooms and Tracks](#livekit-go-based-sfu-with-rooms-and-tracks)
6. [Pion: Building a Custom SFU in Go](#pion-building-a-custom-sfu-in-go)
7. [WHIP and WHEP: HTTP-Based WebRTC Signalling](#whip-and-whep-http-based-webrtc-signalling)
8. [SFU Internals: Simulcast, SVC, and Bandwidth Estimation](#sfu-internals-simulcast-svc-and-bandwidth-estimation)
9. [Recording Pipelines](#recording-pipelines)
10. [Linux Deployment and Operations](#linux-deployment-and-operations)
11. [Integrations](#integrations)

---

## Why WebRTC Needs Server Infrastructure

Peer-to-peer WebRTC works when two endpoints can reach each other and when network conditions are favorable for direct media exchange. In practice, NAT traversal fails often enough — particularly with symmetric NATs and enterprise firewalls — that a relay server is a prerequisite for any production deployment. Beyond connectivity, multi-party conferencing demands media routing: with N participants each sending to all others, a pure mesh creates O(N²) uplink streams, quickly saturating client uplinks.

**Topology choices** map onto different server roles:

- **Peer-to-peer mesh**: No server for media; each participant opens O(N−1) PeerConnections. Feasible up to roughly four participants on broadband; uplink exhaustion beyond that.
- **MCU (Multipoint Conferencing Unit)**: Server decodes all incoming streams and re-encodes a single composite video for each subscriber. Eliminates uplink scaling but burns significant server CPU and introduces encoding latency.
- **SFU (Selective Forwarding Unit)**: Server forwards RTP packets without decoding. Each publisher sends one set of streams; the SFU routes them to each subscriber. Subscriber decode happens on client devices. This is the dominant model for WebRTC conferencing at scale.
- **Cascaded SFU**: Multiple SFU nodes interconnected via `PipeTransport` or equivalent, forwarding streams between data centers to minimize latency for geographically distributed participants.

Beyond routing, servers are needed for: TURN relay (RFC 5766) when ICE fails; recording; WHIP/WHEP-based ingest from hardware encoders; transcoding for interoperability with SIP or RTSP endpoints; and analytics.

### Signalling vs. Media Planes

A key conceptual distinction in WebRTC server infrastructure is between the **signalling plane** and the **media plane**. Signalling conveys SDP offers/answers and ICE candidates; it can use any transport (WebSocket, HTTP, gRPC, MQTT) and is always application-defined. The media plane carries RTP/RTCP packets inside DTLS-SRTP tunnels; it uses UDP (or TCP as fallback) and must meet strict latency and jitter requirements.

TURN servers operate exclusively on the media plane — they relay UDP packets without understanding their content. SFUs sit at the intersection: they terminate ICE and DTLS on the media plane (receiving plaintext RTP internally) while also exposing a signalling interface for room management. MCUs go further, participating in media processing on the media plane by decoding, mixing, and re-encoding video frames.

### Choosing a Topology

The choice between SFU and MCU in practice comes down to the tradeoff between server compute cost and client device constraints. An SFU scales well on commodity server hardware but places the full decoding load on each client — a subscriber in a 10-person call must decode 9 simultaneous video streams. An MCU centralises that decode cost but requires GPU-accelerated encoding infrastructure (typically NVENC or VA-API) to keep the server-side re-encoding latency below 100 ms. Cascaded SFU topologies are used by platforms with global CDN presence: regional SFU clusters receive publisher streams and relay them to regional subscriber clusters via high-bandwidth, low-latency inter-datacenter links, while each regional cluster runs a flat SFU for local subscribers.

---

## coturn: STUN/TURN Server

[coturn](https://github.com/coturn/coturn) is the reference open-source STUN and TURN server implementing RFC 5389 (STUN), RFC 5766 (TURN), RFC 6062 (TURN-TCP), and RFC 6156 (TURN for IPv6). It is written in C and is the most widely deployed open-source TURN server in production WebRTC deployments.

### Configuration

The primary configuration file is `/etc/turnserver.conf`. A production deployment requires several categories of settings:

```ini
# /etc/turnserver.conf — annotated production configuration

# Network identity
realm=turn.example.com
listening-port=3478          # STUN/TURN UDP and TCP
tls-listening-port=5349      # TURN-TLS (coturn handles its own TLS)
listening-ip=0.0.0.0

# Relay address — the IP that allocated relay addresses come from.
# Must be a public IP reachable by WebRTC clients.
relay-ip=203.0.113.10
external-ip=203.0.113.10

# Authentication
lt-cred-mech                 # long-term credential mechanism (RFC 5389)
# Or for time-limited tokens (TURN REST API pattern):
# use-auth-secret
# static-auth-secret=your_shared_secret

# TLS certificate (Let's Encrypt or similar)
cert=/etc/letsencrypt/live/turn.example.com/fullchain.pem
pkey=/etc/letsencrypt/live/turn.example.com/privkey.pem

# Relay port range for UDP allocations
min-port=49152
max-port=65535

# Fingerprint on all STUN messages
fingerprint

# Logging
log-file=/var/log/coturn/turnserver.log
pidfile=/var/run/coturn/turnserver.pid

# Authentication backend — choose one:
# SQLite (default):
userdb=/var/lib/coturn/turndb
# PostgreSQL:
# psql-userdb="host=localhost dbname=coturn user=coturn password=secret"
# Redis:
# redis-userdb="ip=127.0.0.1 dbname=0 password=secret connect_timeout=30"
```

[Source: coturn configuration reference](https://github.com/coturn/coturn/blob/master/examples/etc/turnserver.conf)

### RFC 5766 TURN Allocation Lifecycle

TURN allocates a relay address on behalf of a WebRTC client through a four-step exchange. Understanding this lifecycle is essential for capacity planning and firewall configuration:

1. **Allocate Request**: Client sends `ALLOCATE` with `REQUESTED-TRANSPORT=UDP`. Server authenticates via `401 Unauthorized` with a nonce, then client resends with credentials. Server responds with `RELAYED-ADDRESS` (the relay endpoint assigned to this client) and `LIFETIME` (default 600 seconds).
2. **CreatePermission**: Client installs permissions for each peer IP that may send data through the relay. Without a permission, the TURN server drops packets from that peer.
3. **ChannelBind**: Optionally binds a 2-byte channel number to a peer address/port. Using channel-bound data reduces the 36-byte overhead of `SEND`/`DATA` indications down to 4 bytes, critical for high-bitrate video.
4. **Send/Data**: Client sends media via `SEND` indications (or channel-bound data); server forwards to the peer. Return traffic arrives as `DATA` indications.

Allocations are refreshed with `REFRESH` requests before `LIFETIME` expires. Clients that drop unexpectedly leave orphaned allocations until timeout.

[Source: RFC 5766, Section 2](https://www.rfc-editor.org/rfc/rfc5766)

### Linux Deployment

**systemd unit** (typically installed by the `coturn` package):

```ini
# /lib/systemd/system/coturn.service
[Unit]
Description=coTURN STUN/TURN Server
After=network.target

[Service]
User=turnserver
ExecStart=/usr/bin/turnserver -c /etc/turnserver.conf
Restart=on-failure
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

**Firewall rules** for the relay port range (using nftables):

```bash
# Allow STUN/TURN control ports
nft add rule inet filter input udp dport 3478 accept
nft add rule inet filter input tcp dport { 3478, 5349 } accept
# Allow relay UDP port range
nft add rule inet filter input udp dport 49152-65535 accept
```

### Capacity Planning

Each TURN allocation consumes approximately 4–8 KB of RAM for session state plus the kernel socket buffers. At 256 MB reserved for coturn, a single server can sustain roughly 30,000–50,000 concurrent allocations. Relay bandwidth is the binding constraint: each allocation relays media bidirectionally, so a server with 1 Gbps uplink can carry approximately 1,000 concurrent streams at 1 Mbps.

Tune kernel socket buffers for high-throughput relay:

```bash
# /etc/sysctl.d/99-coturn.conf
net.core.rmem_max = 26214400
net.core.wmem_max = 26214400
net.core.rmem_default = 1048576
net.core.wmem_default = 1048576
```

### Prometheus Metrics

coturn includes a built-in Prometheus exporter compiled in when the `prom` and `microhttpd` libraries are present (the official Docker images include this by default). Enable it in `turnserver.conf`:

```ini
prometheus                   # enable the exporter
prometheus-port=9641         # default port
prometheus-path=/metrics     # default path
```

Scrape `http://turn.example.com:9641/metrics` for counters including total allocations, active allocations, bytes relayed, and authentication failures. [Source: coturn Prometheus support](https://github.com/coturn/coturn/pull/517)

---

## Janus WebRTC Gateway

[Janus](https://github.com/meetecho/janus-gateway) is a widely deployed open-source WebRTC media server written in C. Its design principle is that the core handles all WebRTC plumbing — ICE, DTLS-SRTP, RTP/RTCP mux — while media processing logic lives in plugins that the core calls via a well-defined vtable interface. This makes Janus highly extensible without modifying core code.

[Source: Janus architecture overview](https://janus.conf.meetecho.com/docs/)

### Plugin Architecture: `janus_plugin` Vtable

Every Janus plugin implements the `janus_plugin` struct, which is a vtable of function pointers. The core calls these callbacks at the appropriate points in the session lifecycle:

```c
/* src/plugin.h — janus_plugin vtable (simplified from upstream) */
struct janus_plugin {
    /* Lifecycle */
    int  (*const init)(janus_callbacks *callback, const char *config_path);
    void (*const destroy)(void);

    /* Identity */
    int         (*const get_api_compatibility)(void);
    int         (*const get_version)(void);
    const char *(*const get_version_string)(void);
    const char *(*const get_description)(void);
    const char *(*const get_name)(void);
    const char *(*const get_package)(void);

    /* Session management */
    void (*const create_session)(janus_plugin_session *handle, int *error);
    void (*const destroy_session)(janus_plugin_session *handle, int *error);
    json_t *(*const query_session)(janus_plugin_session *handle);

    /* Signalling */
    struct janus_plugin_result *(*const handle_message)(
        janus_plugin_session *handle,
        char *transaction,
        json_t *message,
        json_t *jsep);
    json_t *(*const handle_admin_message)(json_t *message);

    /* Media events */
    void (*const setup_media)(janus_plugin_session *handle);
    void (*const hangup_media)(janus_plugin_session *handle);

    /* Incoming media — called by core per packet */
    void (*const incoming_rtp)(janus_plugin_session *handle,
                               janus_plugin_rtp *packet);
    void (*const incoming_rtcp)(janus_plugin_session *handle,
                                janus_plugin_rtcp *packet);
    void (*const incoming_data)(janus_plugin_session *handle,
                                janus_plugin_data *packet);
    void (*const data_ready)(janus_plugin_session *handle);
    void (*const slow_link)(janus_plugin_session *handle, int mindex,
                            gboolean video, gboolean uplink);
};
```

[Source: Janus plugin.h source](https://janus.conf.meetecho.com/docs/plugin_8h_source.html)

The `janus_plugin_rtp` struct wraps an RTP buffer along with metadata flags indicating whether the packet is video, and extension information. When an SFU plugin (such as VideoRoom) receives `incoming_rtp`, it forwards the packet to all subscribers by calling back into the Janus core via `janus_callbacks->relay_rtp()`.

### Key Plugins

**VideoRoom**: An SFU plugin implementing a room model with publishers and subscribers. Publishers send media; subscribers receive selected publishers' streams. The room defaults to 3 concurrent publishers (`publishers = 3` in the configuration), configurable per room at creation time — for example, 6 for a small conference or 1 for a webinar. It supports simulcast and SVC (VP9) forwarding. [Source: Janus VideoRoom documentation](https://janus.conf.meetecho.com/docs/videoroom.html)

**AudioBridge**: An MCU-style audio plugin that decodes all publishers' Opus streams, mixes them in software using Opus `opus_decoder`/`opus_encoder` chains, and re-encodes a single mix per subscriber. Suitable for audio-only conferences where subscriber count is high.

**RecordPlay**: Records a session to `.mjr` files (Janus's proprietary RTP dump container) and can play back recordings into a room.

**EchoTest**: Loopback plugin for client testing.

Transport plugins handle signalling: the WebSocket transport (`janus_websockets.c`) is most common for browser clients; an HTTP REST transport and an MQTT transport are also available.

### VideoRoom Signalling Flow

The signalling exchange uses JSON messages over WebSocket. Here is the publisher flow:

```json
// Step 1: Client attaches to the videoroom plugin
{
  "janus": "attach",
  "plugin": "janus.plugin.videoroom",
  "transaction": "abc123"
}

// Step 2: Create or join a room as publisher
{
  "janus": "message",
  "body": {
    "request": "join",
    "room": 1234,
    "ptype": "publisher",
    "display": "Alice"
  },
  "transaction": "def456"
}
// Server responds with "joined" event listing existing publishers

// Step 3: Publish — send configure with JSEP offer
{
  "janus": "message",
  "body": {
    "request": "configure",
    "audio": true,
    "video": true
  },
  "jsep": {
    "type": "offer",
    "sdp": "v=0\r\no=- ..."
  },
  "transaction": "ghi789"
}
// Server responds with JSEP answer; PeerConnection established
```

A subscriber attaches to the same plugin, sends `join` with `ptype: "subscriber"` and a list of publisher feed IDs, receives a JSEP offer from the server, and completes the negotiation by sending back its SDP answer in a `start` request.

[Source: Janus VideoRoom documentation](https://janus.conf.meetecho.com/docs/videoroom.html)

### Building Janus on Linux

Janus uses meson/ninja (preferred) or autotools:

```bash
# Install dependencies (Debian/Ubuntu)
apt install libmicrohttpd-dev libjansson-dev libssl-dev \
            libsrtp2-dev libsofia-sip-ua-dev libglib2.0-dev \
            libopus-dev libogg-dev libcurl4-openssl-dev \
            libnice-dev

# libnice provides ICE; must be >= 0.1.18 for full TURN support

git clone https://github.com/meetecho/janus-gateway.git
cd janus-gateway
meson setup build --prefix=/usr/local
ninja -C build
ninja -C build install
```

Janus runs an internal GLib `GMainLoop` event loop. Media threads are created per PeerConnection using GLib thread pools. Configuration lives in `/usr/local/etc/janus/` with per-plugin `.jcfg` files.

---

## mediasoup: Node.js SFU with C++ Worker

[mediasoup](https://mediasoup.org) is an SFU library — not a standalone server — that embeds into a Node.js (or Rust) application. Its distinctive architecture separates the control plane (Node.js) from the media plane (a compiled C++ subprocess called the Worker). [Source: mediasoup v3 API documentation](https://mediasoup.org/documentation/v3/mediasoup/api/)

### Architecture

The Node.js API layer spawns one or more Worker processes via `mediasoup.createWorker()`. Each Worker is a separate OS process containing an event loop (libuv), ICE agent, DTLS stack, and RTP forwarding tables. The Worker and the Node.js parent communicate over Unix pipes using a [FlatBuffers](https://flatbuffers.dev/)-encoded binary protocol, replacing the earlier JSON protocol that was removed in mediasoup v3.13.0. [Source: mediasoup 3.13.0 release notes](https://mediasoup.discourse.group/t/mediasoup-3-13-0-released-with-flatbuffers-and-more/5649)

The object model forms a hierarchy:

```
Worker  (C++ subprocess)
  └── Router  (RTP routing table, codec capabilities)
        ├── WebRtcTransport  (ICE + DTLS, for browser clients)
        ├── PlainTransport   (raw RTP/SRTP, for GStreamer / FFmpeg)
        ├── PipeTransport    (Router-to-Router, for cascading)
        ├── Producer         (inbound RTP track)
        └── Consumer         (outbound RTP track, one per subscriber)
```

### Node.js Code: Worker, Router, and Transports

```typescript
import * as mediasoup from 'mediasoup';
import { Worker, Router, WebRtcTransport, Producer, Consumer } from 'mediasoup/node/lib/types';

// Create a Worker — spawns the C++ subprocess
const worker: Worker = await mediasoup.createWorker({
  logLevel: 'warn',
  rtcMinPort: 40000,
  rtcMaxPort: 49999,
});

worker.on('died', () => {
  console.error('mediasoup Worker died — restarting');
  process.exit(1);
});

// Create a Router with supported codecs
const router: Router = await worker.createRouter({
  mediaCodecs: [
    { kind: 'audio', mimeType: 'audio/opus', clockRate: 48000, channels: 2 },
    { kind: 'video', mimeType: 'video/VP8',  clockRate: 90000 },
    { kind: 'video', mimeType: 'video/H264', clockRate: 90000,
      parameters: { 'packetization-mode': 1, 'profile-level-id': '42e01f' } },
  ],
});

// Transport for a browser publisher (ICE + DTLS)
const transport: WebRtcTransport = await router.createWebRtcTransport({
  listenInfos: [
    { protocol: 'udp', ip: '0.0.0.0', announcedAddress: '203.0.113.5' },
    { protocol: 'tcp', ip: '0.0.0.0', announcedAddress: '203.0.113.5' },
  ],
  enableSctp: false,
});

// Signal transport parameters to the browser client (ice/dtls/fingerprint)
const transportParams = {
  id:             transport.id,
  iceParameters:  transport.iceParameters,
  iceCandidates:  transport.iceCandidates,
  dtlsParameters: transport.dtlsParameters,
};

// Browser connects; then creates a Producer
transport.on('connect', async ({ dtlsParameters }, callback, errback) => {
  try {
    await transport.connect({ dtlsParameters });
    callback();
  } catch (err) {
    errback(err as Error);
  }
});

// Producer — created when browser calls transport.produce()
const producer: Producer = await transport.produce({
  kind: 'video',
  rtpParameters: /* from browser transport.produce() call */ {} as any,
});

// Consumer — one per subscriber per producer
const consumer: Consumer = await subscriberTransport.consume({
  producerId:    producer.id,
  rtpCapabilities: router.rtpCapabilities,
  paused: true,   // start paused; resume after client signals ready
});
await consumer.resume();
```

### PlainTransport for GStreamer/FFmpeg Ingest

`PlainTransport` exposes a raw UDP endpoint without ICE, enabling GStreamer or FFmpeg pipelines to push RTP directly:

```typescript
const plain = await router.createPlainTransport({
  listenInfo: { protocol: 'udp', ip: '127.0.0.1' },
  rtcpMux: false,
  comedia: true,   // auto-detect remote address from first incoming packet
});
// plain.tuple.localPort is the port GStreamer should target
```

```bash
# GStreamer pipeline pushing H.264 RTP to mediasoup PlainTransport
gst-launch-1.0 \
  videotestsrc ! x264enc tune=zerolatency ! rtph264pay pt=102 ! \
  udpsink host=127.0.0.1 port=40001
```

### PipeTransport for Cascading

`PipeTransport` connects two Routers — either in the same Worker or across Workers on different servers — forwarding all producers between them without re-negotiation. This is the building block for cascaded SFU deployments spanning multiple data centers.

### Simulcast in mediasoup

When a browser publishes simulcast (multiple spatial layers as separate SSRCs), mediasoup creates a single Producer that internally tracks all three layers. A `Consumer` is configured with a preferred spatial and temporal layer:

```typescript
// Consumer simulcast layer selection
await consumer.setPreferredLayers({ spatialLayer: 2, temporalLayer: 2 });

// The Worker selects which incoming SSRC to forward
consumer.on('layerschange', (layers) => {
  console.log('active layers:', layers?.spatialLayer, layers?.temporalLayer);
});
```

[Source: mediasoup simulcast documentation](https://mediasoup.org/documentation/v3/mediasoup/api/#consumer-setPreferredLayers)

---

## LiveKit: Go-Based SFU with Rooms and Tracks

[LiveKit](https://github.com/livekit/livekit) is a complete WebRTC SFU platform built in Go on top of the [Pion](https://github.com/pion/webrtc) WebRTC library. Unlike mediasoup (a library) or Janus (a framework), LiveKit ships as a self-contained server binary with a defined Room → Participant → Track object model and gRPC+JWT-based signalling protocol.

[Source: LiveKit architecture](https://docs.livekit.io/home/get-started/overview/)

### Architecture

A single `livekit-server` binary handles ICE, DTLS-SRTP, and RTP forwarding. For multi-node deployments, Redis stores distributed room state and coordinates participant migration between nodes. The Go runtime's goroutine model maps well to WebRTC's per-connection concurrency: each PeerConnection runs its own goroutine for ICE and DTLS, with RTP forwarding handled by a shared goroutine pool.

### Server Configuration

```yaml
# livekit.yaml — minimal production configuration
port: 7880              # gRPC + HTTP signalling
rtc:
  port_range_start: 50000
  port_range_end:   60000
  tcp_port: 7881          # ICE-over-TCP fallback
  use_external_ip: true   # discover public IP via STUN

redis:
  address: redis.internal:6379

turn:
  enabled: true
  udp_port: 3478
  tls_port: 5349
  domain: turn.example.com
  ttl_seconds: 300

keys:
  APIkey123: APIsecret456   # static key pair for JWT signing
```

[Source: LiveKit config-sample.yaml](https://github.com/livekit/livekit/blob/master/config-sample.yaml)

### WHIP Ingest via livekit-ingress

LiveKit's WHIP support is provided by the separate `livekit-ingress` service, not the main `livekit-server` binary. The ingress service runs alongside the SFU, receives WHIP POSTs, creates a LiveKit room participant representing the ingest source, and forwards the RTP stream into the room. An OBS instance or GStreamer `whipclientsink` can publish directly to a LiveKit room without the LiveKit client SDK. [Source: LiveKit ingress WHIP documentation](https://github.com/livekit/ingress)

The ingress service configuration specifies the WHIP listening port and the address of the upstream LiveKit server:

```yaml
# ingress.yaml
api_key: APIkey123
api_secret: APIsecret456
ws_url: wss://sfu.example.com
whip_port: 8080
```

The WHIP publishing URL takes the form `http://ingress.example.com:8080/whip/publish/{room_name}/{stream_key}`.

### LiveKit Egress: Recording

LiveKit Egress is a separate service that subscribes to room events via the LiveKit SDK and records streams. A `RoomCompositeEgressRequest` spawns a headless Chromium instance that connects to the room as a participant, captures the rendered composite via `getDisplayMedia`, and feeds the captured frames through a GStreamer pipeline:

```
Chromium (headless) → PulseAudio capture → GStreamer:
  [ appsrc video ] → [ vp8enc / x264enc ] ─┐
  [ appsrc audio ] → [ opusenc ]           ├──→ [ webmmux / mp4mux ] → [ filesink ]
```

The resulting file is uploaded to S3, Google Cloud Storage, or a local path. [Source: LiveKit Egress documentation](https://docs.livekit.io/egress/)

### Docker Compose Deployment

```yaml
# docker-compose.yml
services:
  livekit:
    image: livekit/livekit-server:latest
    command: --config /etc/livekit.yaml
    ports:
      - "7880:7880"       # signalling
      - "7881:7881/tcp"   # ICE-TCP
      - "50000-60000:50000-60000/udp"  # RTC
    volumes:
      - ./livekit.yaml:/etc/livekit.yaml
  redis:
    image: redis:7-alpine
```

---

## Pion: Building a Custom SFU in Go

[Pion](https://github.com/pion/webrtc) is a pure-Go WebRTC implementation that underlies LiveKit and many bespoke SFUs. It implements the browser WebRTC API — `PeerConnection`, `RTPSender`, `RTPReceiver`, `DataChannel` — in Go, enabling server-side WebRTC without CGo bindings to libwebrtc.

[Source: Pion webrtc package](https://pkg.go.dev/github.com/pion/webrtc/v4)

### Key Packages

- `github.com/pion/webrtc/v4` — PeerConnection API matching the W3C WebRTC spec
- `github.com/pion/interceptor` — pluggable RTP/RTCP processing chain (NACK, TWCC, stats)
- `github.com/pion/rtp` — RTP packet marshalling/unmarshalling
- `github.com/pion/rtcp` — RTCP packet types (PLI, FIR, ReceiverReport, TransportCC)

### The SFU Forwarding Loop

The core of any SFU is a loop that reads RTP from one publisher track and writes it to N subscriber tracks. In Pion, `RegisterDefaultInterceptors` wires up NACK generation and response, TWCC feedback, and statistics automatically. Custom interceptors can be added to the registry before or after calling it, but adding them separately risks duplication — prefer using the built-in registration helpers:

```go
package main

import (
    "sync"

    "github.com/pion/interceptor"
    "github.com/pion/webrtc/v4"
)

// SFU holds the set of subscriber tracks for a published track.
type SFU struct {
    mu          sync.RWMutex
    subscribers []*webrtc.TrackLocalStaticRTP
}

// NewAPI builds a Pion API with NACK, TWCC, and stats interceptors
// registered via the default interceptor set.
func NewAPI() (*webrtc.API, error) {
    m := &webrtc.MediaEngine{}
    if err := m.RegisterDefaultCodecs(); err != nil {
        return nil, err
    }

    // RegisterDefaultInterceptors adds NACK responder, NACK generator,
    // TWCC sender, and receiver stats to the registry.
    i := &interceptor.Registry{}
    if err := webrtc.RegisterDefaultInterceptors(m, i); err != nil {
        return nil, err
    }

    return webrtc.NewAPI(
        webrtc.WithMediaEngine(m),
        webrtc.WithInterceptorRegistry(i),
    ), nil
}

// ForwardRTP reads RTP packets from a publisher track and writes them
// to all subscriber tracks. Run in a dedicated goroutine per publisher.
func (s *SFU) ForwardRTP(pub *webrtc.TrackRemote) {
    for {
        // ReadRTP blocks until a packet is available.
        pkt, _, err := pub.ReadRTP()
        if err != nil {
            return // connection closed
        }

        s.mu.RLock()
        for _, sub := range s.subscribers {
            // WriteRTP copies the payload internally; safe to reuse pkt.
            if err := sub.WriteRTP(pkt); err != nil {
                continue
            }
        }
        s.mu.RUnlock()
    }
}

// AddSubscriber registers a new local track that will receive
// packets forwarded from the publisher.
func (s *SFU) AddSubscriber(track *webrtc.TrackLocalStaticRTP) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.subscribers = append(s.subscribers, track)
}
```

[Source: Pion webrtc package](https://pkg.go.dev/github.com/pion/webrtc/v4) | [Pion interceptor package](https://pkg.go.dev/github.com/pion/interceptor)

### Go's Concurrency Advantages for SFU Work

Go's goroutine scheduler makes per-connection concurrency cheap: a server handling thousands of participants runs one goroutine per connection without OS thread overhead. `sync.Pool` is useful in the application layer when the SFU needs to allocate auxiliary data structures per packet (such as per-packet timestamps for custom BWE), keeping GC pauses low. Pion's zero-CGo design means a single `go build` produces a static binary deployable with no shared library dependencies — a contrast to servers like Janus that require libnice, libsrtp2, and libjansson installed on the target system.

---

## WHIP and WHEP: HTTP-Based WebRTC Signalling

Traditional WebRTC requires a bespoke WebSocket-based signalling protocol — each implementation invents its own JSON message schema. The IETF WISH working group standardized HTTP-based alternatives that reduce signalling to a single HTTP round-trip.

### WHIP: WebRTC-HTTP Ingestion Protocol (RFC 9725)

[RFC 9725](https://www.rfc-editor.org/rfc/rfc9725.html) (published March 2025) defines WHIP, which replaces custom signalling for the ingest direction (encoder → server). The exchange is:

1. Client sends `HTTP POST` to `/whip/{endpoint}` with `Content-Type: application/sdp` and body containing the SDP offer.
2. Server responds `201 Created` with `Content-Type: application/sdp` body containing the SDP answer and a `Location` header identifying the resource URL (e.g., `/whip/session/abc123`).
3. **Trickle ICE**: Additional ICE candidates are exchanged via `HTTP PATCH` to the resource URL with `Content-Type: application/trickle-ice-sdpfrag` body.
4. To terminate: client sends `HTTP DELETE` to the resource URL.

```bash
# Minimal WHIP ingest using curl (for testing)
curl -X POST https://ingest.example.com/whip/room1 \
  -H "Content-Type: application/sdp" \
  -H "Authorization: Bearer <token>" \
  --data-binary @offer.sdp \
  -D -        # dump response headers to see Location:
```

### WHEP: WebRTC-HTTP Egress Protocol (draft)

WHEP mirrors WHIP for the egress direction (server → viewer). The HTTP exchange is identical in structure: `POST` with SDP offer, `201` with SDP answer, `PATCH` for trickle ICE, `DELETE` to stop. As of mid-2026, WHEP remains an IETF Internet-Draft (`draft-ietf-wish-whep-03`) that has not yet been published as an RFC, though it is widely implemented in production. [Source: IETF WHEP draft tracker](https://datatracker.ietf.org/doc/draft-ietf-wish-whep/)

### GStreamer WHIP/WHEP Elements

GStreamer's `gst-plugins-rs` package (the Rust plugin collection) provides WHIP client support via the `whipclientsink` element (formerly `whipsink`, which is deprecated). The `whipclientsink` element handles ICE gathering, DTLS-SRTP, and the HTTP WHIP POST internally, presenting a simple sink interface to the pipeline:

```bash
# Publish a test pattern to a WHIP endpoint using whipclientsink
gst-launch-1.0 \
  videotestsrc ! videoconvert ! vp8enc deadline=1 ! rtpvp8pay ! \
  whipclientsink whip-endpoint="https://ingest.example.com/whip/live" \
                 auth-token="Bearer mytoken"
```

For WHEP egress (playing back from a WHEP server), the equivalent element is `whepsrc` (from the same Rust plugin):

```bash
# Receive from a WHEP endpoint
gst-launch-1.0 \
  whepsrc whep-endpoint="https://stream.example.com/whep/live" ! \
  decodebin ! autovideosink
```

[Source: GStreamer whipclientsink documentation](https://gstreamer.freedesktop.org/documentation/rswebrtc/whipclientsink.html)

### OBS WHIP Output

OBS Studio 30.0 (released November 2023) added a native WHIP output, enabling sub-100 ms glass-to-glass latency streaming from OBS to any WHIP-compatible server. [Source: OBS Studio 30.0 release](https://phoronix.com/news/OBS-Studio-30-Released) Configure it under **Settings → Stream → Service: WHIP**, entering the server URL and bearer token. OBS negotiates VP8 or H.264 depending on server capability.

### OvenMediaEngine

[OvenMediaEngine](https://github.com/AirenSoft/OvenMediaEngine) (OME) is an open-source streaming server with first-class WHIP ingest and LLHLS/WebRTC egress, written in C++. It accepts WHIP POSTs, terminates ICE and DTLS-SRTP, and can simultaneously re-stream via RTMP, LLHLS, and WebRTC WHEP. For use cases requiring a public-facing low-latency streaming endpoint (OBS → OME → browser viewer) rather than a conferencing SFU, OME's simpler WHIP-in/WHEP-out architecture is easier to operate than a full SFU.

OME configuration declares applications (virtual hosts → applications → output streams):

```xml
<!-- Server.xml snippet for WHIP/WHEP -->
<Bind>
  <Providers>
    <WebRTC>
      <Signalling>
        <Interfaces>
          <Interface>
            <Port>3334</Port>    <!-- WHIP/WHEP signalling -->
            <TLSPort>3335</TLSPort>
          </Interface>
        </Interfaces>
      </Signalling>
      <IceCandidates>
        <IceCandidate>203.0.113.5:10000-10999/udp</IceCandidate>
      </IceCandidates>
    </WebRTC>
  </Providers>
</Bind>
```

The WHIP endpoint for OvenMediaEngine is `https://ome.example.com:3335/app/stream?direction=whip` and the WHEP playback endpoint is `https://ome.example.com:3335/app/stream?direction=whep`.

[Source: OvenMediaEngine WHIP documentation](https://airensoft.gitbook.io/ovenmediaengine/streaming/webrtc-publishing)

---

## SFU Internals: Simulcast, SVC, and Bandwidth Estimation

### Simulcast

Simulcast is a publishing technique where the client encoder produces three spatially-downscaled variants of the video simultaneously, each with its own SSRC. A typical configuration:

| Layer | Resolution | Bitrate | SSRC |
|-------|-----------|---------|------|
| low (r0) | 320×180 | 150 kbps | SSRC_A |
| medium (r1) | 640×360 | 500 kbps | SSRC_B |
| high (r2) | 1280×720 | 1500 kbps | SSRC_C |

The SFU receives all three layers from each publisher. For each subscriber, it selects which SSRC to forward based on the subscriber's estimated available bandwidth. Layer switching happens on keyframe boundaries to avoid visual artifacts: the SFU sends a PLI (Picture Loss Indication) RTCP message to the publisher to request a keyframe before switching the forwarded SSRC.

### SVC (Scalable Video Coding)

VP9 and AV1 implement SVC natively: a single bitstream contains temporal and spatial layer information encoded in RTP extension headers. The SFU can drop packets belonging to unwanted layers without requesting the client to send separate streams:

- **Temporal layers** (`t0`, `t1`, `t2`): reducing temporal quality lowers the frame rate. The SFU drops `t1`/`t2` packets to reduce to the `t0` base frame rate.
- **Spatial layers** (`s0`, `s1`, `s2`): different resolutions within a single stream. Dropping higher-spatial-layer packets degrades resolution.

SVC forwarding requires the SFU to parse the VP9 payload descriptor (RFC 7741) or AV1 aggregation header to identify layer membership. [Source: RFC 7741 VP9 RTP payload format](https://www.rfc-editor.org/rfc/rfc7741)

**AV1 SVC** is particularly compact: the AV1 RTP payload format carries a Dependency Descriptor (DD) RTP header extension that identifies each frame's spatial and temporal layer indices and its reference structure. The DD is defined in the AOMedia AV1 RTP specification at `https://aomediacodec.github.io/av1-rtp-spec/#dependency-descriptor-rtp-header-extension`. An SFU can forward only base-layer frames to a subscriber with limited bandwidth while forwarding all layers to a subscriber with headroom. The DD also carries `start_of_frame` and `end_of_frame` bits, enabling the SFU to drop incomplete layer packets without waiting for sequence number gaps to be resolved by the jitter buffer. [Source: AOMedia AV1 RTP Payload Format specification](https://aomediacodec.github.io/av1-rtp-spec/)

### Keyframe Strategy Under Packet Loss

Heavy packet loss degrades video quality in two phases. First, the client decoder encounters missing reference frames and either freezes or shows artefacts. Second, if loss is persistent, the decoder goes into an error-concealment loop that produces increasingly degraded output. The SFU responds by generating PLI or FIR upstream — but excessive PLI generation forces the publisher's encoder to produce keyframes frequently, which: (a) consumes substantial bitrate, temporarily crowding out audio packets; and (b) on constrained hardware encoders, can stall the encoding pipeline for tens of milliseconds. Production SFUs therefore implement rate-limiting on upstream keyframe requests — a common policy is to send at most one PLI per publisher every 500 ms per subscriber, and to coalesce requests from multiple subscribers into a single upstream PLI rather than flooding the publisher with N simultaneous requests.

### Transport-Wide Congestion Control

Google's Transport-Wide Congestion Control (TWCC, specified in `draft-holmer-rmcat-transport-wide-cc-extensions-01`) adds an RTP header extension carrying a per-packet transport sequence number. Receivers collect per-packet arrival timestamps and report them in compact RTCP feedback messages, enabling the sender's GoogCC algorithm (described in Ch60b) to estimate the path bandwidth with sub-second convergence.

[Source: TWCC IETF draft](https://datatracker.ietf.org/doc/html/draft-holmer-rmcat-transport-wide-cc-extensions-01)

In an SFU, TWCC creates a two-tier feedback loop:

1. **Publisher → SFU**: Publisher sends media; SFU collects per-packet arrival timestamps and sends TWCC feedback to the publisher so it can run GoogCC on its side and rate-adapt its encoders. The SFU acts as the receiver for the publisher's TWCC sequence numbers.
2. **SFU → Subscriber**: SFU forwards media to subscribers; each subscriber generates TWCC feedback back to the SFU. The SFU uses this feedback to estimate each subscriber's available bandwidth and decide which simulcast layer to forward — if a subscriber's bandwidth estimate drops below the medium-layer bitrate threshold, the SFU switches to forwarding the low layer.

The TWCC header extension is assigned extension ID 3 by convention in most SFU implementations (though the actual ID is negotiated in SDP). The RTCP Transport-CC feedback packet carries a run-length or status-vector encoded bitmask of received/not-received packets, plus 16-bit receive delta fields encoding inter-arrival time differences with 250 µs precision. A GoogCC implementation running on the publisher uses these deltas to compute inter-packet delay variation (jitter), from which it estimates queuing delay growth, then applies a signal filter and threshold comparator to detect congestion onset before loss occurs.

### PLI/FIR Keyframe Requests

When a new subscriber joins a room, they cannot decode from a mid-stream P-frame. The SFU sends a PLI (Picture Loss Indication, RFC 4585) upstream to the publisher, requesting an immediate keyframe. The SFU then begins forwarding to the new subscriber only after receiving the keyframe. For codecs or endpoints that do not support PLI, FIR (Full Intra Request, RFC 5104) is used instead. Production SFUs throttle keyframe requests — requesting no more than one per second per publisher — to avoid overwhelming the encoder. [Source: RFC 4585](https://www.rfc-editor.org/rfc/rfc4585)

---

## Recording Pipelines

Recording WebRTC streams server-side requires bridging the SFU's internal RTP forwarding to a muxer that produces a container file. Several approaches exist.

### Janus RecordPlay Plugin

Janus's `RecordPlay` plugin writes `.mjr` (Janus Recording) files — a binary container that prepends a JSON header and then dumps raw RTP packets with relative timestamps. After a session ends, the `janus-pp-rec` post-processing tool converts the dump to WebM or Opus:

```bash
# Convert a Janus video recording to WebM
janus-pp-rec /var/recordings/video-session1.mjr /tmp/session1-video.webm

# Convert audio recording to Opus
janus-pp-rec /var/recordings/audio-session1.mjr /tmp/session1-audio.opus

# If a muxing tool is needed to combine them:
ffmpeg -i /tmp/session1-video.webm -i /tmp/session1-audio.opus \
       -c:v copy -c:a copy /var/archive/session1.webm
```

[Source: Janus RecordPlay plugin documentation](https://janus.conf.meetecho.com/docs/recordplay.html)

### mediasoup PlainTransport Recording

mediasoup does not record internally. The recommended approach is to create a `PlainTransport` for each track to be recorded and connect it to a GStreamer or FFmpeg pipeline running locally:

```typescript
// Create a PlainTransport to receive RTP from the router
const recorder = await router.createPlainTransport({
  listenInfo: { protocol: 'udp', ip: '127.0.0.1' },
  rtcpMux: false,
});
const recorderRtpPort  = recorder.tuple.localPort;
const recorderRtcpPort = recorder.rtcpTuple!.localPort;

// Create a Consumer piping into the recorder transport
const consumer = await recorder.consume({
  producerId: videoProducer.id,
  rtpCapabilities: router.rtpCapabilities,
  paused: false,
});
// Now spawn FFmpeg to receive RTP on recorderRtpPort
```

```bash
# FFmpeg receive VP8 RTP and mux to WebM
ffmpeg \
  -protocol_whitelist file,udp,rtp \
  -i "rtp://127.0.0.1:${RTP_PORT}?rtcpport=${RTCP_PORT}" \
  -c:v copy \
  /var/recordings/session-$(date +%s).webm
```

Alternatively, a GStreamer pipeline offers more flexibility for live transcoding:

```bash
gst-launch-1.0 \
  udpsrc port=${RTP_PORT} caps="application/x-rtp,media=video,encoding-name=VP8,payload=96" ! \
  rtpjitterbuffer ! rtpvp8depay ! webmmux ! \
  filesink location=/var/recordings/session.webm
```

[Source: mediasoup PlainTransport recording guide](https://mediasoup.discourse.group/t/plaintransport-ffmpeg-recording/6512)

### LiveKit Egress Recording

As described in the LiveKit section, Egress spawns a headless Chromium instance to composite and capture the room. For a track-only recording (without composite layout), the `TrackEgressRequest` subscribes to individual tracks and records them as separate files through GStreamer pipelines within the Egress service process. [Source: LiveKit Egress documentation](https://docs.livekit.io/egress/)

### PCAP-Based Recording

As a fallback for any WebRTC server, recording the raw network traffic provides a complete packet capture that can be post-processed:

```bash
# Capture all WebRTC media on the RTC UDP port range
tcpdump -i eth0 -w /var/capture/session-$(date +%s).pcap \
        'udp portrange 50000-60000'
```

Wireshark can decrypt SRTP if the DTLS master secrets are exported (using `ssl.keylog_file` or a custom DTLS hook), then demultiplex and export individual RTP streams for each SSRC. This is primarily useful for debugging rather than production recording, as it captures all sessions simultaneously.

---

## Linux Deployment and Operations

### systemd Unit Files

Deploying coturn, Janus, and LiveKit as systemd services ensures automatic restart on failure and integration with `journalctl` logging.

```ini
# /etc/systemd/system/janus.service
[Unit]
Description=Janus WebRTC Gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=janus
ExecStart=/usr/local/bin/janus -F /usr/local/etc/janus
Restart=on-failure
RestartSec=5
LimitNOFILE=1048576
# Isolate from rest of system
ProtectSystem=strict
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/livekit.service
[Unit]
Description=LiveKit SFU Server
After=network-online.target redis.service

[Service]
Type=simple
ExecStart=/usr/local/bin/livekit-server --config /etc/livekit.yaml
Restart=on-failure
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

### Prometheus Metrics

All three major servers expose Prometheus metrics:

| Server | Mechanism | Default Endpoint |
|--------|-----------|-----------------|
| coturn | Built-in HTTP server (prom + microhttpd) | `:9641/metrics` |
| mediasoup | Node.js `worker.getResourceUsage()` + custom `/metrics` via `prom-client` | Application-defined |
| LiveKit | Built-in Prometheus endpoint | `:6789/metrics` |

Key metrics to alert on: `coturn_allocations_total`, `livekit_room_participants_total`, mediasoup Worker CPU usage via resource usage polling.

A Prometheus scrape configuration for these endpoints:

```yaml
# prometheus.yml scrape config for WebRTC infrastructure
scrape_configs:
  - job_name: 'coturn'
    static_configs:
      - targets: ['turn.example.com:9641']

  - job_name: 'livekit'
    static_configs:
      - targets: ['sfu.example.com:6789']
    metrics_path: /metrics

  - job_name: 'mediasoup'
    # mediasoup exposes metrics via a Node.js prom-client HTTP server
    static_configs:
      - targets: ['sfu.example.com:9100']
```

For mediasoup, the application must expose its own metrics. A typical implementation polls `worker.getResourceUsage()` every 5 seconds and increments counters for active Producers and Consumers:

```typescript
import { register, Gauge } from 'prom-client';

const workerCpuGauge = new Gauge({
  name: 'mediasoup_worker_cpu_percent',
  help: 'mediasoup C++ Worker CPU usage',
  labelNames: ['pid'],
});

setInterval(async () => {
  const usage = await worker.getResourceUsage();
  workerCpuGauge.set({ pid: String(worker.pid) },
    usage.ru_utime + usage.ru_stime);
}, 5000);
```

### Capacity Planning

**TURN relay**: Total relay bandwidth = Σ (bitrate per relayed stream). A TURN server with 1 Gbps uplink handles approximately 500 simultaneous 2 Mbps streams before saturating. RAM per allocation is 4–8 KB; 100,000 allocations require less than 1 GB.

**SFU CPU**: The forwarding path (copy RTP from inbound socket to N outbound sockets) is memory-bandwidth-bound, not compute-bound, when SRTP passthrough is used (no re-encryption). A rough rule of thumb is 1 CPU core per 100 simultaneously forwarded streams at 1 Mbps each. Simulcast layer switching, RTCP processing, and Transport-CC feedback generation add overhead; benchmark with realistic load before commissioning production capacity.

**Network tuning for high-throughput WebRTC**:

```bash
# /etc/sysctl.d/99-webrtc.conf

# Increase UDP socket receive buffer for TURN and SFU
net.core.rmem_max = 26214400      # 25 MB
net.core.wmem_max = 26214400
net.core.rmem_default = 1048576
net.core.wmem_default = 1048576

# Increase connection tracking table for many UDP flows
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_udp_timeout = 30
net.netfilter.nf_conntrack_udp_timeout_stream = 180
```

### ICE Candidate Priority and NAT Override

WebRTC ICE prioritizes candidates in order: `host` > `srflx` (server-reflexive, via STUN) > `relay` (via TURN). When the SFU runs in a cloud VM with a private IP and a public NAT address, it should advertise its public IP as a `host` candidate using the server's `nat1to1` configuration option (Janus: `nat-1-1` in `janus.cfg`; mediasoup: `announcedAddress` in `listenInfos`; LiveKit: `rtc.use_external_ip: true`). This prevents clients that can reach the SFU directly from unnecessarily routing through TURN.

### ICE-TCP and Firewall Traversal

When UDP is blocked by enterprise firewalls, WebRTC can fall back to ICE-over-TCP (RFC 6544). The SFU listens on a TCP port (typically 7881 for LiveKit, or configured via `tcp-port` in Janus) and includes TCP `host` candidates in the SDP offer. ICE-over-TCP carries RTP-over-TCP using the TURN framing format even for direct connections — each packet is prefixed with a 2-byte length field.

TCP fallback significantly increases per-connection CPU overhead at the SFU because each TCP connection requires its own kernel socket state and the TLS-like DTLS handshake must complete before media flows. When designing capacity for enterprise deployments where ICE-TCP is common, budget 3–5× more CPU per session compared to UDP.

An alternative is to route all traffic through TURN-TCP (coturn port 3478/TCP or 5349/TURN-TLS), which encapsulates UDP TURN messages over TCP. This keeps the media path at the TURN relay as UDP while the client-to-TURN leg uses TCP, reducing the CPU penalty at the SFU.

### Health Checks and Graceful Shutdown

SFU servers should expose an HTTP health endpoint for load balancer probes. When a node is being drained (for a rolling upgrade), it should stop accepting new ICE connections while allowing existing sessions to complete. For Janus, this can be implemented by temporarily removing the node from the load balancer target group and waiting for the active session count (`/janus/info` endpoint includes session count) to reach zero before restarting.

LiveKit handles this via a gRPC drain call: the node marks itself as unavailable in Redis, stopping new room assignments from being routed to it, while continuing to serve active rooms. After a configurable drain timeout, remaining participants are forcibly disconnected and the process exits.

### TLS Architecture Note

DTLS for media and TLS for TURN control are independent. DTLS is negotiated per PeerConnection on UDP — it cannot be terminated by nginx or any TCP reverse proxy. TURN-TLS (port 5349) is terminated by coturn itself using the certificate configured in `turnserver.conf`. Do not place an nginx SSL proxy in front of port 5349; coturn must hold the private key directly to perform the TURN-TLS handshake before any STUN/TURN messages flow.

For HTTPS signalling endpoints (Janus HTTP transport, mediasoup application HTTP server, LiveKit gRPC), nginx or a load balancer can terminate TLS normally — these are plain TCP connections. The critical distinction is: signalling TLS is on TCP and can be proxied; DTLS for media is on UDP per-PeerConnection and cannot be proxied.

---

## Integrations

- **Ch60b — WebRTC Protocol Stack**: This chapter builds directly on Ch60b's coverage of SDP offer/answer, ICE candidate gathering, DTLS-SRTP key derivation, RTP/RTCP header formats, GoogCC bandwidth estimation, and GStreamer `webrtcbin`. The server infrastructure here assumes those mechanisms; revisit Ch60b for protocol-layer details on ICE states, SRTP cipher suites, and RTCP feedback message formats.

- **Ch57 — FFmpeg**: The recording pipeline in Section 9 uses FFmpeg to receive RTP from mediasoup `PlainTransport` and mux it to MP4 or WebM. Ch57 covers FFmpeg's RTP demuxer, codec pipeline, and muxer architecture in depth, including the `rtp://` protocol handler and `rtpjitterbuffer` behaviour.

- **Ch58 — GStreamer**: GStreamer's `whipclientsink` and `whepsrc` elements (Section 7) and the RTP recording pipelines (Section 9) are built on GStreamer's plugin model covered in Ch58, including `webrtcbin`, `rtpjitterbuffer`, depayloader elements, and `webmmux`/`mp4mux`. Ch58 also covers the `webrtcbin` ICE lifecycle that `whipclientsink` wraps internally.

- **Ch38 — PipeWire Screen Capture**: PipeWire's `pipewiresrc` GStreamer element can capture a desktop portal stream and feed it into a `whipclientsink` pipeline, streaming a captured desktop into a LiveKit room or OvenMediaEngine WHIP endpoint. Ch38 covers the PipeWire stream negotiation, DMA-BUF zero-copy path, and xdg-desktop-portal integration that makes this capture path efficient.

- **Ch59 — DeepStream RTSP Ingest**: NVIDIA DeepStream pipelines (Ch59) can terminate RTSP ingest and transcode for AI inference, then forward processed video via RTP into an SFU's `PlainTransport` for WebRTC distribution. The `nvv4l2h264enc` encoder in a DeepStream pipeline can feed directly into a mediasoup or Janus `PlainTransport` endpoint without re-encoding at the SFU.

- **Ch123 — Screen Capture and Remote Desktop**: The WHIP-based streaming path for captured desktops (using PipeWire → GStreamer → `whipclientsink`) is detailed in Ch123, which covers `xdg-desktop-portal`, pipewire-capture, and the full pipeline from KMS scanout to a WebRTC room. The recording and SFU infrastructure in this chapter provides the server-side receiving end for those desktop streaming sessions.
