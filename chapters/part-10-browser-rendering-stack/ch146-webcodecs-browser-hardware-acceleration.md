# Chapter 146: WebCodecs and Browser Hardware Acceleration

**Part**: X — The Browser Rendering Stack

**Primary audiences**: Browser and web platform engineers building real-time media pipelines — live streaming, video conferencing, custom codec transcoders, and frame-level WebGPU compositing. Application developers who need to understand the performance boundaries between the web platform and the Linux hardware stack beneath it: VA-API, DMA-BUF, SharedImage, and the GPU process sandbox. Systems engineers debugging hardware video acceleration on Linux Chrome who need to trace the path from a JavaScript `VideoDecoder.decode()` call down to a libva `vaEndPicture()` invocation in a kernel DRM driver.

> **Relationship to Chapter 147.** This chapter covers the **API and specification side** of browser video acceleration: what JavaScript developers call (`VideoDecoder`, `VideoFrame`, `importExternalTexture`), what the W3C WebCodecs spec promises, how decoded frames reach WebGPU and Canvas, and how to query and debug hardware support from JavaScript. It includes an overview of how Chrome maps a `decode()` call through to VA-API (§4), sufficient for a developer to understand the performance contract.
>
> **Chapter 147** covers the **browser implementation side**: how Chrome and Firefox implement VA-API hardware decode in C++ — `VaapiVideoDecoder`, `VaapiWrapper`, codec-specific delegates, `OzoneImageBacking`, the Out-of-Process Video Decode sandbox, Firefox's `FFmpegVideoDecoder` and `DMABufSurface`, and NVIDIA-specific quirks. Readers who want to understand browser internals, contribute to Chrome or Firefox media code, or debug VA-API failures at the driver level should read Chapter 147 alongside this one.

---

## Table of Contents

1. [WebCodecs Overview: The Web Platform's First Low-Latency Codec API](#1-webcodecs-overview-the-web-platforms-first-low-latency-codec-api)
   - [1.1 Background: Media Source Extensions (MSE)](#11-background-media-source-extensions-mse)
   - [1.2 Background: DASH — Dynamic Adaptive Streaming over HTTP](#12-background-dash--dynamic-adaptive-streaming-over-http)
   - [1.3 Background: WebRTC — Real-Time Peer-to-Peer Media](#13-background-webrtc--real-time-peer-to-peer-media)
   - [1.4 Background: EBML, Matroska, and WebM](#14-background-ebml-matroska-and-webm)
   - [1.5 Background: ISO BMFF and Fragmented MP4](#15-background-iso-bmff-and-fragmented-mp4)
   - [1.6 Background: HLS — HTTP Live Streaming](#16-background-hls--http-live-streaming)
   - [1.7 Background: Video and Audio Codecs](#17-background-video-and-audio-codecs)
   - [1.8 Why WebCodecs Instead of MSE or WebRTC?](#18-why-webcodecs-instead-of-mse-or-webrtc)
2. [VideoDecoder: Chunk Submission, Frame Delivery, and Lifecycle](#2-videodecoder-chunk-submission-frame-delivery-and-lifecycle)
3. [VideoEncoder: Parameters, Latency Modes, and Key Frame Control](#3-videoencoder-parameters-latency-modes-and-key-frame-control)
4. [The Hardware Acceleration Path on Linux](#4-the-hardware-acceleration-path-on-linux)
5. [VideoFrame and DMA-BUF: Zero-Copy from Decoder to Canvas](#5-videoframe-and-dmabuf-zero-copy-from-decoder-to-canvas)
6. [WebCodecs + WebGL and WebGPU Integration](#6-webcodecs--webgl-and-webgpu-integration)
7. [Codec Support Matrix on Linux](#7-codec-support-matrix-on-linux)
8. [MediaCapabilities API: Querying Hardware Support](#8-mediacapabilities-api-querying-hardware-support)
9. [WebRTC Integration: Encoded Transform and Custom Pipelines](#9-webrtc-integration-encoded-transform-and-custom-pipelines)
10. [VideoFrame Canvas Capture and Recording](#10-videoframe-canvas-capture-and-recording)
11. [Debugging WebCodecs and Hardware Acceleration](#11-debugging-webcodecs-and-hardware-acceleration)
12. [Integrations](#12-integrations)
13. [References](#13-references)

---

## 1. WebCodecs Overview: The Web Platform's First Low-Latency Codec API

For most of the web's history, video decoding was an opaque operation controlled by the browser. A developer handed an MP4 to an `<video>` element or a DASH manifest to Media Source Extensions (MSE), and the browser managed the entire decode, frame-timing, and display pipeline internally. This worked well for linear playback but was fundamentally incompatible with applications that needed to process individual frames: video conferencing clients applying AR effects frame-by-frame, broadcast tools that needed to inspect or modify H.264 NAL units before rendering, game streaming clients applying custom interpolation, or scientific visualisation tools overlaying decoded sensor data onto a WebGPU scene.

**WebCodecs** is the W3C specification that closes this gap. [Source](https://www.w3.org/TR/webcodecs/) It gives JavaScript direct, low-level access to the browser's codec machinery: not as a black-box playback pipeline but as individual primitives — decode an `EncodedVideoChunk`, receive a `VideoFrame`, inspect or transform its pixel data, render it wherever you need it. The API shipped in **Chrome 94** (September 2021) and was enabled on all desktop platforms in **Firefox 130** (September 2024). [Source](https://developer.mozilla.org/en-US/docs/Web/API/WebCodecs_API)

### 1.1 Background: Media Source Extensions (MSE)

**Media Source Extensions (MSE)** is the W3C API that gave JavaScript its first programmatic control over the browser media pipeline — not frame-level control, but segment-level control. [Source](https://www.w3.org/TR/media-source-2/) Before MSE, the only way to play video in a browser was to point an `<video src="...">` at a file and let the browser download and decode it monolithically. MSE replaced that with a streaming contract where JavaScript owns the byte flow:

```javascript
const video = document.querySelector('video');
const mediaSource = new MediaSource();
video.src = URL.createObjectURL(mediaSource);

mediaSource.addEventListener('sourceopen', async () => {
  // MIME type includes codec string for decoder pre-configuration
  const sourceBuffer = mediaSource.addSourceBuffer(
    'video/mp4; codecs="avc1.64001f,mp4a.40.2"'
  );

  // Fetch and append the initialization segment (ftyp + moov boxes)
  const init = await fetch('/stream/init.mp4').then(r => r.arrayBuffer());
  sourceBuffer.appendBuffer(init);

  sourceBuffer.addEventListener('updateend', async () => {
    // Append successive media segments (moof + mdat boxes)
    const seg = await fetch('/stream/segment-1.m4s').then(r => r.arrayBuffer());
    sourceBuffer.appendBuffer(seg);
  });
});
```

The `SourceBuffer` accepts ISO BMFF (MP4) or WebM containers and forwards the enclosed encoded data to the browser's internal decoder. The critical constraint: **decoded frames never surface to JavaScript**. They flow from the internal decoder directly into the compositor. MSE is a _feeding pipe_, not a processing API. [Source](https://www.w3.org/TR/media-source-2/#sourcebuffer)

MSE Level 2 (the current revision) added `appendEncodedChunks()`, which accepts `EncodedVideoChunk` and `EncodedAudioChunk` directly — the same objects that WebCodecs produces. This creates a compositing bridge: WebCodecs decodes, JavaScript transforms, `appendEncodedChunks()` re-feeds into the MSE pipeline for timed playback. The full round-trip avoids re-encapsulating in a container.

---

### 1.2 Background: DASH — Dynamic Adaptive Streaming over HTTP

**DASH** (Dynamic Adaptive Streaming over HTTP, ISO/IEC 23009-1) is the adaptive bitrate (ABR) streaming standard built on top of MSE. [Source](https://www.iso.org/standard/83005.html) A DASH session begins with an **MPD** (Media Presentation Description), an XML manifest that describes available quality tiers:

```xml
<MPD xmlns="urn:mpeg:dash:schema:mpd:2011" minBufferTime="PT2S" type="dynamic">
  <Period>
    <!-- Video: three quality representations -->
    <AdaptationSet mimeType="video/mp4" codecs="avc1.640028" frameRate="30">
      <Representation id="360p"  bandwidth="800000"  width="640"  height="360"
                      initialization="video/360p/init.mp4"
                      media="video/360p/seg-$Number$.m4s" startNumber="1"/>
      <Representation id="720p"  bandwidth="2500000" width="1280" height="720"
                      initialization="video/720p/init.mp4"
                      media="video/720p/seg-$Number$.m4s" startNumber="1"/>
      <Representation id="1080p" bandwidth="5000000" width="1920" height="1080"
                      initialization="video/1080p/init.mp4"
                      media="video/1080p/seg-$Number$.m4s" startNumber="1"/>
    </AdaptationSet>
    <!-- Audio -->
    <AdaptationSet mimeType="audio/mp4" codecs="mp4a.40.2">
      <Representation id="aac128" bandwidth="128000"
                      initialization="audio/init.mp4"
                      media="audio/seg-$Number$.m4s" startNumber="1"/>
    </AdaptationSet>
  </Period>
</MPD>
```

The DASH player library (dash.js, Shaka Player, hls.js for HLS) implements the **ABR algorithm**: it monitors network throughput and buffer fullness, selects the highest representation that can be fetched without rebuffering, fetches the next segment via `fetch()`, and feeds it to MSE's `SourceBuffer`. Segment boundaries are typically 2–6 seconds, allowing quality switches at each boundary without decoding glitches.

**CMAF** (Common Media Application Format, ISO/IEC 23000-19) standardised the container profile — chunked CMAF can produce sub-segment boundaries of 100–500 ms, dramatically reducing DASH live latency. Netflix, YouTube, and Disney+ all use CMAF-based DASH. [Source](https://www.iso.org/standard/71975.html)

DASH itself has no JavaScript API — it is entirely implemented in userspace libraries that use `fetch()` + MSE. The browser sees only a stream of `appendBuffer()` calls; the segment selection logic runs in the page. This makes DASH highly extensible (custom ABR, custom DRM integration via Encrypted Media Extensions) but keeps frame access equally opaque.

---

### 1.3 Background: WebRTC — Real-Time Peer-to-Peer Media

**WebRTC** (Web Real-Time Communication) is the IETF/W3C suite for browser-to-browser audio and video communication without a server in the media path. [Source](https://www.w3.org/TR/webrtc/) The core API is `RTCPeerConnection`, which manages connection negotiation, codec selection, and media transport:

```javascript
// Offer/answer SDP exchange (signalling channel is application-defined)
const pc = new RTCPeerConnection({ iceServers: [{ urls: 'stun:stun.example.com' }] });

// Add local camera track
const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
stream.getTracks().forEach(track => pc.addTrack(track, stream));

// Create SDP offer
const offer = await pc.createOffer();
await pc.setLocalDescription(offer);
// ... send offer via signalling, receive answer ...
await pc.setRemoteDescription(new RTCSessionDescription(answer));

// ICE candidate exchange
pc.onicecandidate = e => { if (e.candidate) signalling.send(e.candidate); };

// Receive remote media
pc.ontrack = e => {
  const remoteVideo = document.querySelector('#remote-video');
  remoteVideo.srcObject = e.streams[0];
};
```

The transport layer uses **SRTP** (Secure Real-time Transport Protocol) over UDP, with **DTLS** for key exchange and **ICE** (Interactive Connectivity Establishment) for NAT traversal via STUN/TURN servers. Codec negotiation happens in the **SDP** (Session Description Protocol) offer/answer: the browser proposes VP8/VP9/H.264/AV1 depending on hardware availability, and the remote peer selects from the intersection.

Like MSE, the baseline WebRTC API exposes no access to encoded frames or decoded pixels — the video track goes from the RTP receiver directly into the compositor. The `<video>` element shows the received stream, but JavaScript cannot inspect NAL units, apply GPU transforms, or reroute frames to a canvas.

**Insertable Streams / Encoded Transform** (now standardised as `RTCRtpScriptTransform`) is the WebRTC extension that opens the codec layer — and is where WebRTC and WebCodecs intersect. Covered in §9 of this chapter.

```javascript
// Encoded Transform: intercept encoded video frames on the receive path
const receiver = pc.getReceivers().find(r => r.track.kind === 'video');
const { readable, writable } = new TransformStream({
  transform(encodedFrame, controller) {
    // encodedFrame is an RTCEncodedVideoFrame — same structure as EncodedVideoChunk
    // Inspect H.264 NAL units, apply E2E encryption, watermark, etc.
    const view = new DataView(encodedFrame.data);
    console.log('NAL unit type:', view.getUint8(4) & 0x1f);
    controller.enqueue(encodedFrame);  // pass through unchanged
  }
});
receiver.transform = new RTCRtpScriptTransform(worker, { readable, writable });
```

`RTCEncodedVideoFrame` carries the same `type` (`key` / `delta`), `timestamp`, and raw `data` as `EncodedVideoChunk`. Chrome 94 shipped both APIs simultaneously, making them a matched pair: WebRTC captures the camera, Encoded Transform extracts frames, WebCodecs re-decodes them in a worker for GPU processing, and a `VideoFrame` goes to `importExternalTexture()` for WebGPU compositing. This pipeline underpins advanced video conferencing features like virtual backgrounds and real-time noise gating. [Source](https://www.w3.org/TR/webrtc-encoded-transform/)

#### WebRTC Transport Stack

The line "uses SRTP over UDP, with DTLS and ICE" in the opening paragraph covers five distinct protocols that each solve a different problem. Here is what each one does:

**RTP** (Real-time Transport Protocol; RFC 3550) is the media packet format. Every audio or video frame is chopped into one or more **RTP packets** with a 12-byte fixed header: [Source](https://www.rfc-editor.org/rfc/rfc3550)

```
RTP Header (12 bytes minimum):
  V=2 | P | X | CC (4 bits) | M | PT (7 bits) | Sequence Number (16 bits)
  Timestamp (32 bits) — media clock units (90 kHz for video, 48 kHz for audio)
  SSRC (32 bits)      — synchronisation source; identifies the sender's stream
  CSRC list (0–15 × 4 bytes) — contributing sources for mixed streams
```

`Timestamp` uses the codec's native clock: 90 kHz for video (same as MPEG-TS PTS), 48 000 Hz for Opus audio. `Sequence Number` increments per packet; gaps indicate loss. `SSRC` uniquely identifies one media stream — a call with camera + screen share has two video SSRCs. RTP carries no encryption; that is SRTP's job.

**SRTP** (Secure RTP; RFC 3711) adds confidentiality and integrity to RTP by encrypting the payload and authenticating the header + payload with a message authentication code. [Source](https://www.rfc-editor.org/rfc/rfc3711) Two cipher suites dominate WebRTC:
- `AES_CM_128_HMAC_SHA1_80` — AES-128 Counter Mode + HMAC-SHA1 with 80-bit tag (legacy)
- `AEAD_AES_128_GCM` — AES-128-GCM; authenticates in one pass; preferred in RFC 7714

The keys are never in RTP packets; they are established by DTLS.

**DTLS** (Datagram TLS; RFC 9147) is TLS 1.3 adapted for UDP — it adds sequencing and retransmission logic because UDP drops and reorders packets. [Source](https://www.rfc-editor.org/rfc/rfc9147) In WebRTC, DTLS runs over the ICE-established UDP path and performs a standard TLS handshake. At the end of the handshake, both peers extract the **SRTP key material** from the TLS `exportKeyingMaterial()` function (RFC 5705) — this is the DTLS-SRTP binding defined in RFC 5764. After that the DTLS channel carries only data channel (SCTP) traffic; media travels as SRTP.

**ICE** (Interactive Connectivity Establishment; RFC 8445) solves the NAT traversal problem: two browsers behind different NATs cannot directly address each other. [Source](https://www.rfc-editor.org/rfc/rfc8445) ICE gathers a set of **candidates** — network addresses the peer might be reachable at:
- **Host candidates**: local IP:port pairs
- **Server-reflexive (srflx) candidates**: the public IP:port seen by a STUN server
- **Relayed (relay) candidates**: a TURN server's address that will proxy traffic

**STUN** (RFC 8489) is a simple request/response protocol: the client sends a Binding Request to a public STUN server, which echoes back the client's observed public IP and port — this is the server-reflexive candidate. **TURN** (RFC 8656) is a relay: when direct connectivity fails (symmetric NAT, firewall), the TURN server allocates a relay address and forwards all media. TURN is expensive (full media bandwidth through the server) and used only as a last resort. ICE tries all candidate pairs in priority order and selects the one that succeeds.

**SDP** (Session Description Protocol; RFC 8866) is the text format used to describe what each peer can do and agree on what they will do. [Source](https://www.rfc-editor.org/rfc/rfc8866) A WebRTC SDP offer/answer contains:

```sdp
v=0
o=- 1234567890 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1          ← multiplex audio+video on one port
m=video 9 UDP/TLS/RTP/SAVPF 96 97 98
c=IN IP4 0.0.0.0
a=rtcp-mux
a=fingerprint:sha-256 AA:BB:...  ← DTLS certificate fingerprint
a=ice-ufrag:F7gI
a=ice-pwd:x9cml...
a=rtpmap:96 VP9/90000          ← RTP payload type 96 = VP9, 90 kHz clock
a=rtpmap:97 H264/90000
a=fmtp:97 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=4d0034
a=rtpmap:98 AV01/90000
a=extmap:1 urn:ietf:params:rtp-hdrext:toffset   ← RTP header extension
a=ssrc:1234 cname:s9hDwDQNjISOvji
m=audio 9 UDP/TLS/RTP/SAVPF 111
a=rtpmap:111 opus/48000/2
a=fmtp:111 minptime=10;useinbandfec=1
```

`profile-level-id=4d0034` in the `fmtp` line is the same hex triplet as the `avc1.4d0034` codec string — SDP and BMFF use the same H.264 profile/level encoding. The `fingerprint` line ties DTLS identity to SDP, preventing man-in-the-middle attacks. ICE credentials (`ice-ufrag`, `ice-pwd`) are also in the SDP, so the signalling channel that carries SDP must be secured (typically WebSocket over HTTPS).

---

### 1.4 Background: EBML, Matroska, and WebM

#### EBML (Extensible Binary Meta Language)

**EBML** is a binary serialization format designed by the Matroska team (Steve Lhomme et al.) in 2002 and standardised as **IETF RFC 8794** in 2020. [Source](https://www.rfc-editor.org/rfc/rfc8794) It is the wire format shared by both Matroska and WebM. EBML is self-describing at the structural level: every element carries its own ID, size, and data, so a reader can traverse any EBML document without a pre-shared schema — similar in spirit to XML but binary and far more compact.

Every EBML element follows this layout:

```
┌─────────────┬──────────────┬──────────────────────┐
│  ID  (VINT) │  Size (VINT) │  Data (size bytes)   │
└─────────────┴──────────────┴──────────────────────┘
```

Both ID and Size use **VINT** (Variable-Length Integer) encoding, analogous to UTF-8's width detection: the number of leading zero bits in the first byte determines how many bytes follow.

```
VINT width encoding:
  1 byte : 1xxx xxxx                        (values 0x01 – 0x7E)
  2 bytes: 01xx xxxx  xxxx xxxx             (values 0x7F – 0x3FFE)
  3 bytes: 001x xxxx  xxxx xxxx  xxxx xxxx
  4 bytes: 0001 xxxx  ...                   (used for most Matroska element IDs)

Examples:
  ID 0x1A45DFA3  →  bytes: 0x1A 0x45 0xDF 0xA3  (4-byte VINT: leading 0001)
  Size = 31      →  byte:  0x9F                  (1-byte VINT: 1001 1111 → 0x1F = 31)
  Size = unknown →  bytes: 0x01 0xFF 0xFF 0xFF 0xFF 0xFF 0xFF 0xFF  (all-ones sentinel)
```

The **unknown-size sentinel** (all data bits set to 1) is critical for streaming: the top-level `Segment` and each `Cluster` may be written with unknown size, allowing a live muxer to write elements without knowing the final duration. This is exactly how `MediaRecorder` generates WebM streams from a live camera capture.

The EBML document always begins with an **EBML Header** element (ID `0x1A45DFA3`) that carries schema metadata:

```
EBML Header
  EBMLVersion:       1
  EBMLReadVersion:   1
  EBMLMaxIDLength:   4
  EBMLMaxSizeLength: 8
  DocType:           "matroska"  (or "webm")
  DocTypeVersion:    4
  DocTypeReadVersion: 2
```

`DocType` is the only field that distinguishes a Matroska file from a WebM file at the EBML level.

---

#### Matroska (MKV)

**Matroska** is an open-source multimedia container format, first released in 2003 and standardised as **IETF RFC 9559** in 2024. [Source](https://www.rfc-editor.org/rfc/rfc9559) It uses EBML with `DocType: "matroska"` and supports virtually any codec, making it the dominant archival and remux container on Linux. File extensions: `.mkv` (video+audio), `.mka` (audio only), `.mks` (subtitles only).

Top-level Matroska element hierarchy:

```
EBML Header       (DocType="matroska", DocTypeVersion=4)
└── Segment       (may be unknown-size for live streams)
    ├── SeekHead  (byte-offset index to top-level elements for fast seeking)
    ├── Info      (duration, TimestampScale [default 1ms], title, muxing app)
    ├── Tracks
    │   └── TrackEntry × N
    │       ├── TrackType    (1=video, 2=audio, 17=subtitle)
    │       ├── CodecID      (e.g. "V_MPEGH/ISO/HEVC", "A_AAC", "A_OPUS")
    │       ├── CodecPrivate (codec init data — same role as avcC/hvcC in BMFF)
    │       └── Video / Audio  (resolution, pixel format, sample rate, channels)
    ├── Chapters  (ordered chapter list, edition entries — absent in WebM)
    ├── Tags      (key/value metadata: title, artist, date, IMDB ID, ...)
    ├── Cues      (seek index: timestamp → Cluster byte offset + track relative pos)
    ├── Attachments (fonts, cover art, ICC profiles — absent in WebM)
    └── Cluster × N   (time-anchored group of blocks, may be unknown-size)
        ├── Timestamp    (absolute cluster base time in TimestampScale units)
        ├── SimpleBlock × N  (single frame: track ref + relative timestamp + flags + data)
        └── BlockGroup × N   (block + optional BlockDuration, BlockAdditions, ReferenceBlock)
            ├── Block          (same payload as SimpleBlock but in a wrapper)
            └── BlockAdditions (HDR metadata, Dolby Vision RPU, etc.)
```

**CodecID and CodecPrivate** are the Matroska equivalents of the BMFF `stsd` box. The `CodecPrivate` element carries the codec initialisation record:

| CodecID | Codec | CodecPrivate content |
|---------|-------|----------------------|
| `V_MPEG4/ISO/AVC` | H.264 | Binary `avcC` record (SPS + PPS NAL units) |
| `V_MPEGH/ISO/HEVC` | H.265 | Binary `hvcC` record (VPS + SPS + PPS) |
| `V_AV1` | AV1 | Binary `av1C` record (Sequence Header OBU) |
| `V_VP9` | VP9 | VP9 codec features struct (optional) |
| `V_VP8` | VP8 | None required |
| `A_AAC` | AAC | AudioSpecificConfig (same as BMFF `esds`) |
| `A_OPUS` | Opus | `OpusHead` identification header (8 bytes + channels) |
| `A_VORBIS` | Vorbis | Three Vorbis header packets concatenated |
| `A_FLAC` | FLAC | `fLaC` marker + STREAMINFO metadata block |
| `A_AC3` | AC-3 | None required |

This `CodecPrivate` content maps directly to `VideoDecoderConfig.description` or `AudioDecoderConfig.description` in WebCodecs — a Matroska demuxer extracts it and passes it verbatim.

Matroska supports a much broader codec set than WebM: H.264, H.265, AV1, VP8, VP9, Theora, MPEG-2, FFV1 (lossless), RealVideo for video; AAC, AC-3, E-AC-3, DTS, TrueHD, FLAC, MP3, Opus, Vorbis for audio. It also supports ordered chapters (referencing segments in other files), complex subtitle formats (ASS/SSA, PGS, DVBSUB), and file attachments (fonts for subtitle rendering, cover art). These features are explicitly excluded from the WebM profile.

---

#### WebM: A Restricted Matroska Profile

**WebM** is a royalty-free, open-source media container format released by Google in 2010 alongside VP8 and the first public version of WebRTC. [Source](https://www.webmproject.org/about/) It is a restricted profile of Matroska with `DocType: "webm"`: it uses the same EBML wire format but constrains codec choices and removes rarely-used features to produce a simpler, browser-friendly container.

**Allowed codecs in WebM:**
| Track type | Permitted codecs |
|------------|-----------------|
| Video | VP8, VP9, AV1 |
| Audio | Vorbis, Opus |

The container hierarchy for a WebM file:

```
EBML Header          (magic bytes: 0x1A 0x45 0xDF 0xA3)
└── Segment
    ├── SeekHead     (index into top-level element positions)
    ├── Info         (duration, timecode scale, muxing app)
    ├── Tracks
    │   ├── TrackEntry (Video: codec=V_VP9, width, height, colour metadata)
    │   └── TrackEntry (Audio: codec=A_OPUS, channels, sample rate)
    ├── Cues         (keyframe seek index: timestamp → cluster offset)
    └── Cluster*     (group of frames, timecode anchor)
        └── SimpleBlock* | BlockGroup*
            └── Block (track number, relative timestamp, frame data)
```

Each `Cluster` holds a batch of `SimpleBlock` elements. A `SimpleBlock` carries the raw codec bitstream (VP9 frame, AV1 OBU sequence, Opus packet) with a track reference and a timecode relative to the cluster's base. There are no codec-specific framing bytes in the container: the block data IS the raw codec output. This matters for WebCodecs: when a JavaScript demuxer (such as the `mp4box.js`/`ebml-stream` family) parses a WebM file, each `SimpleBlock` payload maps 1:1 to one `EncodedVideoChunk`.

```javascript
// Feeding a WebM video track to WebCodecs via a userspace EBML parser
import EBMLReader from 'ebml-stream';

const reader = new EBMLReader();
const decoder = new VideoDecoder({
  output: frame => { /* render */ frame.close(); },
  error: e => console.error(e),
});

reader.on('SimpleBlock', ({ track, keyframe, timestamp, payload }) => {
  if (track === videoTrackNumber) {
    decoder.decode(new EncodedVideoChunk({
      type: keyframe ? 'key' : 'delta',
      timestamp: timestamp * 1000,   // WebM timecodes are µs; WebCodecs uses µs
      data: payload,
    }));
  }
});
```

**WebM and MediaRecorder**: when `MediaRecorder` outputs `video/webm`, it produces a live WebM stream with VP8, VP9, or AV1 video (browser-selected). Clusters are closed after each keyframe interval, making the stream seekable after receipt. MSE accepts `video/webm; codecs="vp09.00.10.08"` as a `SourceBuffer` MIME type.

**libwebm** is the reference C++ library for WebM muxing/demuxing ([github.com/webmproject/libwebm](https://github.com/webmproject/libwebm)). Chromium's media pipeline uses `webm_video_util.cc` / `webm_audio_util.cc` from this library for both playback and `MediaRecorder` output.

---

### 1.5 Background: ISO BMFF and Fragmented MP4

**ISO BMFF** (ISO Base Media File Format, ISO/IEC 14496-12) is the container standard underlying `.mp4`, `.m4a`, `.m4v`, `.mov`, and the CMAF segments used by DASH and HLS. [Source](https://www.iso.org/standard/83102.html) It uses a **box** (also called _atom_) structure where every element is:

```
┌─────────────────────────────────────────────────────────┐
│  size (4 bytes, big-endian)  │  type (4 bytes, ASCII)   │
│               data (size − 8 bytes)                     │
└─────────────────────────────────────────────────────────┘
```

Extended size: if the 4-byte size field is 1, the next 8 bytes hold a 64-bit size (used for boxes >4 GiB). Type `uuid` signals a vendor extension box.

**Standard (non-fragmented) MP4** layout:
```
ftyp  (file type + compatible brands: "isom", "mp42", "avc1", ...)
free  (padding / placeholder)
moov  (movie container — ALL metadata lives here)
│  mvhd  (duration, timescale, creation time)
│  trak  (one per track)
│  │  tkhd  (track header: width, height, duration)
│  │  mdia
│  │  │  mdhd  (media timescale)
│  │  │  hdlr  ('vide' | 'soun' | 'text')
│  │  │  minf → stbl  (sample table: stts, stss, stco, stsz, stsc, stsd)
│  │  │  └── stsd  (sample description → avcC / hvcC / av1C / vpCC boxes)
mdat  (raw encoded frame data, indexed by stbl offsets)
```

The `stsd` child box (`avcC`, `hvcC`, `av1C`, `vpCC`) carries the codec configuration record — the SPS/PPS for H.264, VPS/SPS/PPS for H.265, or Sequence Header OBU for AV1. This is what `VideoDecoder` needs as the `description` field in `VideoDecoderConfig`.

**Fragmented MP4 (fMP4)** replaces the monolithic `mdat` with repeating `moof`+`mdat` pairs:
```
ftyp  (brands include "iso5", "avc1", "dash" for DASH CMAF)
moov  (minimal: track declarations only, no sample table)
│  mvex/trex  (track extends defaults)
│
moof  ← movie fragment (one per segment boundary)
│  mfhd  (sequence number)
│  traf  (track fragment)
│     tfhd  (base data offset, default sample flags)
│     tfdt  (decode time of first sample in this fragment)
│     trun  (per-sample: duration, size, flags, composition offset)
mdat  ← raw encoded data for this fragment
moof  ← next fragment
mdat
...
```

The `tfdt` box gives absolute decode time without requiring the full `stts` sample table — MSE uses it to place each fragment on the media timeline. The `trun` box lists per-sample composition time offsets (PTS − DTS), critical for B-frame reordering in H.264 and H.265.

**CMAF** (Common Media Application Format, ISO/IEC 23000-19) is a profile of fMP4 that constrains which codec configurations and box combinations are permitted, enabling a single set of segment files to be served by both DASH and HLS manifests. A CMAF track file is a self-contained fMP4 stream; the DASH MPD and the HLS `.m3u8` playlist both reference the same `.cmfv`/`.cmfa` segments.

**Codec string mapping**: the `stsd` codec configuration box determines the codec string used in WebCodecs and MSE:

| `stsd` box | Codec string example |
|------------|---------------------|
| `avcC` | `avc1.64001f` (profile=High, constraints=0x00, level=31) |
| `hvcC` | `hvc1.1.6.L120.90` |
| `av1C` | `av01.0.08M.10` (profile=0, level=8, tier=Main, depth=10) |
| `vpCC` | `vp09.00.10.08` (profile=0, level=10, bit-depth=8) |
| `Opus` | `opus` |
| `mp4a` | `mp4a.40.2` (MPEG-4 AAC-LC) |

The codec string is what you pass to `VideoDecoderConfig.codec`, `SourceBuffer.addSourceBuffer()`, and `MediaCapabilities.decodingInfo()`. The `avcC` box content (the SPS/PPS NAL units) becomes `VideoDecoderConfig.description`.

**Ogg** is the third browser-supported container, predating both WebM and BMFF. It carries Vorbis audio and Theora video (both deprecated in favour of Opus/AV1) but Opus-in-Ogg remains in use for audio-only streams. MSE support for Ogg is limited (Firefox only); WebCodecs ignores containers entirely, so Ogg-demuxed Opus packets feed `AudioDecoder` identically to WebM-demuxed ones.

---

### 1.6 Background: HLS — HTTP Live Streaming

**HLS** (HTTP Live Streaming) is Apple's adaptive bitrate streaming protocol, defined in RFC 8216. [Source](https://www.rfc-editor.org/rfc/rfc8216) It uses a hierarchical **M3U8 playlist** format and is the dominant streaming protocol for iOS/Safari and Apple TV, while DASH dominates Android/Chrome. The two protocols have largely converged on the same CMAF segment format.

---

#### M3U8 Playlist Format

M3U8 is a plain-text playlist format derived from the extended M3U format. Every line is either a URI (pointing to a segment or a sub-playlist) or a tag beginning with `#`. Tags with `#EXT-X-` prefix are HLS-specific extensions.

**Master playlist** (`master.m3u8`) — selects among quality variants:
```m3u8
#EXTM3U
#EXT-X-VERSION:6

# Video renditions
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360,CODECS="avc1.64001f,mp4a.40.2"
360p/index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720,CODECS="avc1.64001f,mp4a.40.2"
720p/index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080,CODECS="avc1.640028,mp4a.40.2"
1080p/index.m3u8

# Audio-only rendition (for offline/low-bandwidth)
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="audio",NAME="English",DEFAULT=YES,URI="audio/index.m3u8"
```

**Variant playlist** (`720p/index.m3u8`) — lists individual segments:
```m3u8
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-TARGETDURATION:6
#EXT-X-MAP:URI="720p/init.mp4"         ← CMAF initialization segment (ftyp + moov)

#EXTINF:6.000,
720p/seg-001.m4s                        ← CMAF media segment (moof + mdat)
#EXTINF:6.000,
720p/seg-002.m4s
#EXTINF:6.000,
720p/seg-003.m4s
```

Key M3U8 tags and their semantics:

| Tag | Meaning |
|-----|---------|
| `#EXTM3U` | File type marker; must be first line |
| `#EXT-X-VERSION:N` | HLS spec version; version 7 required for fMP4 segments |
| `#EXT-X-TARGETDURATION:N` | Maximum segment duration in seconds (ceiling of all `#EXTINF` values) |
| `#EXT-X-MAP:URI=...` | Initialization segment URI; required for fMP4; absent for MPEG-TS |
| `#EXTINF:duration,title` | Duration of the following segment in seconds |
| `#EXT-X-STREAM-INF:...` | Describes a variant stream in the master playlist |
| `#EXT-X-MEDIA:TYPE=AUDIO,...` | Declares an alternate audio/subtitle rendition |
| `#EXT-X-KEY:METHOD=...,URI=...` | Encryption key for following segments |
| `#EXT-X-ENDLIST` | Marks a complete VOD playlist (absent in live streams) |
| `#EXT-X-PLAYLIST-TYPE:VOD\|EVENT` | VOD = static; EVENT = live that only appends |
| `#EXT-X-PART:DURATION=...,URI=...` | LL-HLS partial segment declaration |
| `#EXT-X-PRELOAD-HINT:TYPE=PART,...` | LL-HLS prefetch hint for next partial segment |
| `#EXT-X-SERVER-CONTROL:CAN-BLOCK-RELOAD=YES` | Enables LL-HLS blocking playlist reload |

`#EXT-X-MAP` points to the same `init.mp4` a DASH `initialization` URL references. When HLS uses fMP4 segments (version ≥ 7, `.m4s` extensions), those segment files are byte-for-byte identical to DASH CMAF segments — only the manifest format differs.

---

#### MPEG-TS: Legacy HLS Segment Format

Older HLS deployments (and still default for broadcast-to-web pipelines) use **MPEG-TS** (MPEG Transport Stream, ISO/IEC 13818-1) segments instead of fMP4. [Source](https://www.iso.org/standard/83348.html) TS is a fixed-size packet format originally designed for broadcast (DVB, ATSC) where fixed packet sizes simplify hardware multiplexers and error-correction schemes.

Every TS packet is exactly **188 bytes** — 4-byte header + up to 184 bytes of payload:

```
TS Packet (188 bytes):
┌─────────────────────────────────────────────────────────────┐
│ Byte 0: Sync byte = 0x47 (always)                           │
│ Bytes 1–2:                                                   │
│   bit 15    : Transport Error Indicator (TEI)                │
│   bit 14    : Payload Unit Start Indicator (PUSI)           │
│                 — set on first TS packet of a new PES        │
│   bit 13    : Transport Priority                             │
│   bits 12–0 : PID (13 bits, identifies the elementary stream)│
│ Byte 3:                                                      │
│   bits 7–6  : Transport Scrambling Control (00 = not scrambled)│
│   bits 5–4  : Adaptation Field Control                       │
│                 01 = payload only                            │
│                 10 = adaptation field only                   │
│                 11 = adaptation field + payload              │
│   bits 3–0  : Continuity Counter (wraps 0–15, per PID)      │
├─────────────────────────────────────────────────────────────┤
│ Adaptation Field (if AFC = 10 or 11):                        │
│   length (1 byte) + flags + optional PCR + stuffing bytes   │
│   PCR (Program Clock Reference): 33+9 bit timestamp         │
│         at 27 MHz; used for A/V synchronisation             │
├─────────────────────────────────────────────────────────────┤
│ Payload (up to 184 bytes)                                    │
└─────────────────────────────────────────────────────────────┘
```

**PIDs** (Packet Identifiers) demultiplex streams within a TS file:
- **PID 0x0000** — PAT (Program Association Table): maps program numbers to PMT PIDs
- **PID from PAT** — PMT (Program Map Table): maps elementary stream PIDs to stream types
- **Elementary stream PIDs** — carry video, audio, or subtitle data

Stream type values in the PMT determine the codec:

| Stream type | Codec |
|-------------|-------|
| `0x1B` | H.264 / AVC |
| `0x24` | H.265 / HEVC |
| `0x10` | MPEG-4 Part 2 video |
| `0x0F` | AAC (ADTS-framed) |
| `0x81` | AC-3 / Dolby Digital |
| `0x06` | Private data (may carry AAC, SCTE-35 splice points, etc.) |

**PES** (Packetized Elementary Stream) wraps the raw codec data with a timing header. When `PUSI = 1`, the TS payload starts with a PES header:

```
PES Header:
  Start code:    0x000001 (3 bytes)
  Stream ID:     0xE0 (video), 0xC0 (audio first), 0xC1 (audio second), ...
  PES Length:    2 bytes (0 = unbounded, used for video)
  Flags:         PTS_DTS_flags, scrambling, priority, alignment
  Header length: 1 byte
  PTS:           33-bit timestamp at 90 kHz clock (optional)
  DTS:           33-bit decode timestamp (optional, only when PTS ≠ DTS)
  Payload:       raw codec data (Annex-B H.264, ADTS AAC, etc.)
```

The **90 kHz PTS** is the timing unit for all MPEG-TS timestamps — 1 tick = 1/90000 seconds ≈ 11.1 µs. To convert to WebCodecs `timestamp` (microseconds): `pts_us = pts_90khz * 1000 / 90`. This conversion is performed by hls.js in `src/demux/tsdemuxer.ts` when extracting `EncodedVideoChunk` data from TS segments.

TS segments are self-contained: the PAT and PMT repeat at the start of every segment, so no initialization segment is needed. This is why `#EXT-X-MAP` is absent in MPEG-TS playlists. The trade-off is that TS segments are larger than equivalent fMP4 segments (due to padding and repeated table overhead) and limited to H.264 + AAC in practice — VP9, AV1, and HEVC are technically specifiable via private stream types but lack standardised carriage and browser support.

---

#### Low-Latency HLS (LL-HLS)

Apple extended HLS in 2019 to achieve sub-2-second live latency using three mechanisms:
- **Partial segments** (`#EXT-X-PART`): 200 ms CMAF chunks within the 6-second target duration; the server publishes them as they complete
- **Preload hints** (`#EXT-X-PRELOAD-HINT`): the playlist advertises the next partial segment's URI before it exists; the client begins the fetch immediately (HTTP/2 push or 206 range request)
- **Blocking playlist reload**: the client appends `_HLS_msn=` and `_HLS_part=` query parameters; the server holds the response until that sequence number/part is available

LL-HLS reaches 1–2 seconds end-to-end latency versus 15–30 seconds for standard HLS. CMAF chunked encoding (sub-segment boundaries in fMP4) enables the equivalent technique for DASH. Apple, AWS Elemental, and Akamai serve both simultaneously from a single CMAF origin by generating one set of `.m4s` files referenced by both the M3U8 and the DASH MPD.

---

#### HLS Encryption and DRM

**Segment encryption** (`#EXT-X-KEY`): HLS can encrypt segments with AES-128 in CBC mode. The key URI points to a 16-byte key file; the IV is derived from the segment sequence number or specified explicitly:

```m3u8
#EXT-X-KEY:METHOD=AES-128,URI="https://keys.example.com/key",IV=0x000000000000000000000000000001A3
#EXTINF:6.000,
720p/seg-004.m4s
```

**FairPlay Streaming (FPS)**: Apple's DRM for HLS, used on iOS, tvOS, macOS Safari, and Apple TV. FairPlay uses `METHOD=SAMPLE-AES` in `#EXT-X-KEY`, where individual media samples (not whole segments) are encrypted. The key exchange uses Apple's `skd://` URI scheme, resolved by the application via `AVContentKeySession` (native) or `com.apple.fps` key system in EME (Encrypted Media Extensions). On Linux, FairPlay is not available outside Safari — Chrome and Firefox use Widevine, which requires DASH or MSE-level integration.

#### DRM Ecosystem: EME, CDM, CENC, Widevine, PlayReady

DRM for streaming video in browsers is a three-layer stack. Understanding each layer explains why Widevine is "better" on Chrome, FairPlay is "better" on Safari, and PlayReady is "better" on Edge — they are not competing quality levels but platform-specific implementations of the same standard interface.

**CENC** (Common Encryption; ISO/IEC 23001-7) is the container-level standard that specifies how encrypted samples are stored in fMP4/CMAF segments. [Source](https://www.iso.org/standard/84637.html) It defines two encryption schemes:
- **`cenc`** — AES-128 Counter Mode (CTR); used by Widevine and PlayReady
- **`cbcs`** — AES-128 Cipher Block Chaining with constant IV; used by FairPlay

A CENC-encrypted fMP4 segment is identical to an unencrypted one except:
1. Each sample's data is encrypted (video NAL unit payloads, audio frame payloads)
2. A `pssh` (Protection System Specific Header) box in the `moov` atom carries DRM-system-specific data — one `pssh` per DRM system, identified by a 16-byte System ID UUID
3. The `sinf`/`schm` box in the `stsd` replaces the codec box to signal encryption

The same CMAF segment file can be decrypted by multiple DRM systems because each DRM's `pssh` is included in the `moov`. This **multi-DRM** pattern (one set of segments, multiple `pssh` boxes) is the standard for cross-platform streaming.

**EME** (Encrypted Media Extensions; W3C Recommendation) is the browser API that connects JavaScript to the DRM system. [Source](https://www.w3.org/TR/encrypted-media-2/) It does not implement DRM itself — it provides a standard interface to a platform-supplied **CDM** (Content Decryption Module). The flow:

```javascript
// 1. Detect encrypted content
video.addEventListener('encrypted', async e => {
  // e.initDataType: 'cenc' | 'keyids' | 'webm'
  // e.initData: the pssh box bytes from the segment

  // 2. Create a MediaKeys session for the DRM system
  const access = await navigator.requestMediaKeySystemAccess('com.widevine.alpha', [{
    initDataTypes: ['cenc'],
    videoCapabilities: [{ contentType: 'video/mp4; codecs="avc1.4d0034"' }],
    distinctiveIdentifier: 'required',
    persistentState: 'required',
  }]);
  const mediaKeys = await access.createMediaKeys();
  await video.setMediaKeys(mediaKeys);

  // 3. Open a key session
  const session = mediaKeys.createSession('temporary');
  session.addEventListener('message', async e => {
    // e.message: licence request blob (opaque, DRM-system-specific)
    // Send to licence server, receive licence response
    const licence = await fetch('/licence', { method: 'POST', body: e.message })
                          .then(r => r.arrayBuffer());
    await session.update(licence);  // gives the CDM the decryption key
  });
  await session.generateRequest(e.initDataType, e.initData);
});
```

The `'com.widevine.alpha'` string is the **Key System ID** — a reverse-domain string that identifies which CDM to use. Each DRM vendor registers their own:

| Key System ID | DRM System | Platforms |
|---------------|-----------|-----------|
| `com.widevine.alpha` | Widevine | Chrome, Edge, Firefox, Android |
| `com.apple.fps` | FairPlay | Safari (macOS, iOS, tvOS) |
| `com.microsoft.playready` | PlayReady | Edge, IE, Xbox, Tizen |
| `org.w3.clearkey` | ClearKey (no DRM, test only) | All browsers |

**CDM** (Content Decryption Module) is the sandboxed native binary that actually holds the decryption keys and decrypts media samples. The browser communicates with the CDM via EME; the CDM communicates with the DRM vendor's licence server. The CDM runs in a separate process (Chrome's `media/cdm/`) with restricted syscalls (seccomp-BPF) so it cannot be inspected or tampered with. On Linux, the Widevine CDM is a proprietary Google binary (`libwidevinecdm.so`) shipped with Chrome; Firefox downloads it via `mozplayernative`.

**Widevine** (Google) operates three **security levels** that differ in where the key and decode happen:
- **L1** (highest): decryption and decode happen inside a TEE (Trusted Execution Environment) or secure video path; the decrypted frame never appears in normal memory; required for 1080p+ on streaming platforms
- **L2**: decryption in TEE but decode in software; rare; transitional
- **L3** (most common on desktop Linux): decryption and decode in software inside the CDM process; sufficient for SD/HD; what Chrome on Linux typically achieves without TEE hardware

**PlayReady** (Microsoft) uses a similar security-level model (SL150/SL2000/SL3000). PlayReady 4.0 introduced CBCS support alongside the older CTR mode, enabling CMAF compatibility.

**What is "better"**: CENC/EME/CDM is the framework — it has no inherent quality difference from Widevine vs PlayReady. The practical hierarchy is:
- **Widevine L1** > **L3** for content protection strength (L3 is reversible in software)
- **Widevine** is better than **PlayReady** on Linux (PlayReady has no Linux CDM for Chrome)
- **FairPlay** is required for Safari/iOS; there is no alternative on Apple platforms
- **ClearKey** is not real DRM — it is an EME test mode with unprotected key delivery

---

#### HLS vs DASH Summary

| | HLS | DASH |
|---|---|---|
| Spec body | IETF RFC 8216 | ISO/IEC 23009-1 |
| Manifest | M3U8 (text) | MPD (XML) |
| Segments | MPEG-TS or fMP4/CMAF | fMP4/CMAF |
| Native browser | Safari | None (all use MSE libraries) |
| Live latency (standard) | ~15–30 s | ~6–10 s |
| Live latency (low) | LL-HLS ~1–2 s | Chunked CMAF ~2–4 s |
| DRM | FairPlay (`SAMPLE-AES`) | Widevine, PlayReady (CENC) |
| Timing unit | 90 kHz PTS (MPEG-TS) / `tfdt` (fMP4) | `tfdt` (fMP4) |

---

### 1.7 Background: Video and Audio Codecs

WebCodecs operates on raw encoded bitstreams rather than containers, so the codec — not the wrapper — determines the `VideoDecoderConfig.codec` string, the `description` field layout, and what hardware acceleration paths are available on Linux. This section explains every codec referenced in this chapter.

---

#### Video Codecs

**H.264 / AVC** (ISO/IEC 14496-10, ITU-T H.264; 2003) is the most widely deployed video codec in history, carried in virtually every MP4, HLS, and DASH stream. [Source](https://www.itu.int/rec/T-REC-H.264/en) Its bitstream is structured as **NAL units** (Network Abstraction Layer): SPS (Sequence Parameter Set, global configuration), PPS (Picture Parameter Set, per-slice settings), IDR slices (intra/keyframe), and non-IDR slices (inter/delta frames).

Two bitstream framing formats exist:
- **Annex B** — NAL units prefixed with a start code `0x00 0x00 0x00 0x01`; used in MPEG-TS and raw H.264 files
- **AVCC** (AVC Configuration) — length-prefixed NAL units; the `avcC` box in ISO BMFF stores the SPS+PPS, and each sample carries 4-byte length-prefixed slices

`VideoDecoderConfig` for AVCC streams must set `description` to the binary `avcC` record. For Annex B streams, `description` may be omitted. The `format` field in `VideoEncoderConfig.avc` selects output framing: `'annexb'` or `'avc'`.

Profiles encode the feature set: Baseline (no B-frames, no interlaced) → Main → High. The codec string `avc1.PPCCLL` encodes profile (`PP`), constraint flags (`CC`), and level (`LL`) as hex bytes — e.g., `avc1.4d0034` = Main Profile (0x4d), no constraints (0x00), Level 5.2 (0x34).

H.264's patent portfolio (Via LA LLC) is scheduled to expire around 2028 for most jurisdictions, making it royalty-free in practice for new deployments. Hardware decode support is universal: Intel Gen7+, AMD GCN1+, NVIDIA Fermi+, and every ARM SoC with a video engine supports H.264 VLD acceleration.

---

**VP8** (Google / On2 Technologies; 2010; IETF RFC 6386) was released alongside WebM and WebRTC as a royalty-free H.264 alternative. [Source](https://www.rfc-editor.org/rfc/rfc6386) It uses a partition-based bitstream (no NAL units), DCT-based transform with 4×4 and 8×8 prediction, and a binary arithmetic (range) coder. VP8 was the original mandatory video codec in WebRTC, specified in RFC 7742.

VP8 is now largely superseded by VP9 for new deployments. Its hardware decode support is present on some Intel (Gen6 Ivy Bridge+) and ARM SoCs but absent on AMD discrete GPUs. No mainstream Linux GPU supports VP8 hardware **encode**. In WebCodecs, `VideoDecoder` supports VP8 (`codec: 'vp8'`) on all platforms in software; hardware decode is opportunistic.

---

**VP9** (Google; 2013) is the successor to VP8, offering roughly double the compression efficiency at the same quality level. [Source](https://www.webmproject.org/vp9/) YouTube adopted VP9 as its primary streaming codec in 2014 and only began transitioning to AV1 after 2020. VP9 introduced 64×64 superblock coding units (versus VP8's 16×16 macroblock maximum), improved intra prediction, and a more sophisticated entropy coder.

VP9 defines four profiles by bit depth and chroma subsampling:
- **Profile 0**: 8-bit, 4:2:0 (most common; `vp09.00.LL.08`)
- **Profile 1**: 8-bit, 4:2:2 or 4:4:4
- **Profile 2**: 10 or 12-bit, 4:2:0 (HDR)
- **Profile 3**: 10 or 12-bit, 4:2:2 or 4:4:4

The codec string is `vp09.PP.LL.BB.CC` — profile, level (×10), bit depth, chroma subsampling. Hardware decode is available on Intel Gen9 (Skylake+), AMD RDNA1+, NVIDIA Pascal+, and Rockchip/MediaTek ARM SoCs. Hardware encode via VA-API is rare; software encode (libvpx-vp9) is standard.

---

**H.265 / HEVC** (ISO/IEC 23008-2, ITU-T H.265; 2013) achieves approximately 40% better compression than H.264 at the same perceptual quality by using larger coding tree units (up to 64×64 CTU vs H.264's 16×16 macroblock), improved intra prediction (35 directions vs 9), and sample-adaptive offset filtering. [Source](https://www.itu.int/rec/T-REC-H.265/en)

HEVC's browser support is complicated by a fragmented patent pool (MPEG LA, Via LA, Access Advance / HEVC Advance). Safari and Edge support it natively; Chrome ships it as an optional hardware-decode path gated behind `chrome://flags/#enable-hevc-for-streaming`; Firefox has no HEVC support. On Linux, Chrome enables HEVC hardware decode via VA-API when the `hvcC` entrypoint is advertised (Intel Gen9+ for 8-bit Main Profile, Gen12+ for Main10 10-bit).

The codec string is `hvc1.PP.CC.LX.FF` — profile, compatibility flags, level indicator, tier flag.

---

**AV1** (Alliance for Open Media; 2018) is the dominant royalty-free next-generation video codec, targeting 30% better efficiency than VP9 and 50% better than H.264. [Source](https://aomedia.org/av1-bitstream-and-decoding-process-specification/) Its design is radically different from the NAL-unit codecs: the bitstream is structured as **OBUs** (Open Bitstream Units) — Temporal Delimiter, Sequence Header, Frame Header, Tile Group, and Metadata OBUs. The Sequence Header OBU is what `VideoDecoderConfig.description` carries for AV1 in MP4 (via the `av1C` box).

Key technical advances: superblock sizes up to 128×128 (vs VP9's 64×64), 56 intra prediction modes, compound inter prediction, constrained directional enhancement filter (CDEF), loop restoration filter, and a non-binary arithmetic entropy coder. AV1 defines three profiles — **Main** (4:2:0, 8/10-bit), **High** (4:4:4), **Professional** (12-bit).

The codec string is `av01.P.LLT.BB[.MC.CP.TC.MX]` — profile (0–2), level (`LL`), tier (`M`ain/`H`igh), bit depth (08 or 10). Example: `av01.0.08M.10` = Main Profile, Level 4.0, Main Tier, 10-bit.

Hardware decode (Linux): Intel Tiger Lake / Gen12+ via iHD driver; AMD RDNA3 (RX 7000 series); NVIDIA Ampere (RTX 3000+). Hardware encode: Intel DG2/Arc and Meteor Lake+ via VA-API; AMD hardware encode is present but has known bugs in Chrome's delegate (see §7.2). Software encode is via SVT-AV1 (fastest) or libaom (reference, slowest).

---

#### Audio Codecs

**AAC** (Advanced Audio Coding; ISO/IEC 13818-7; 1997) is the default audio codec for MP4, DASH, and HLS. It superseded MP3 for most streaming applications. The most common profile is **AAC-LC** (Low Complexity), codec string `mp4a.40.2`. **HE-AAC** (High Efficiency, using Spectral Band Replication) is `mp4a.40.5` and targets lower bitrates (24–64 kbps). **HE-AACv2** adds Parametric Stereo (`mp4a.40.29`). `AudioDecoder` accepts AAC in ADTS framing (with sync word `0xFFF`) or in raw form with the codec-specific box data as `description`.

**Opus** (Xiph/Google; IETF RFC 6716; 2012) is the mandatory audio codec for WebRTC and the preferred audio codec for WebM. [Source](https://www.rfc-editor.org/rfc/rfc6716) It is a hybrid codec combining the **SILK** vocoder (optimised for speech below 8 kHz) with the **CELT** wideband codec, switching dynamically. Frame sizes range from 2.5 ms to 60 ms; bitrates from 6 kbps to 510 kbps; sample rates up to 48 kHz. Opus is fully royalty-free. `AudioDecoder` uses codec string `opus` (no profile suffix); the initialization packet is carried in the `OpusHead` structure in the `dOps` box or WebM TrackEntry.

**Vorbis** (Xiph.org; 2000) is the predecessor to Opus, royalty-free, and the original WebM audio codec. It is carried in the WebM `A_VORBIS` track codec ID. Vorbis is considered legacy for new deployments — Opus supersedes it — but remains in existing WebM files. `AudioDecoder` supports Vorbis with codec string `vorbis`; the three Vorbis header packets (identification, comment, setup) must be concatenated and passed as `description`.

---

#### Image Codecs and Containers (ImageDecoder)

`ImageDecoder` (part of WebCodecs) handles still images with full metadata preservation. Unlike `VideoDecoder`, it handles container parsing itself — you pass the entire image blob, not individual frames. The image world has both **codecs** (compression algorithms) and **containers** (file formats that wrap the codec output):

**JPEG** (ISO/IEC 10918; 1992) — the universal lossy still-image codec. DCT-based 8×8 block transform, Huffman entropy coding, 8-bit per channel, no alpha channel, no HDR. Ubiquitous: every camera, every browser, every OS has a JPEG decoder. `image/jpeg`. JPEG's successor is JPEG-XL.

**PNG** (ISO 15948; 2003) — lossless, deflate-compressed, supports 8/16-bit per channel and full alpha. The standard for graphics, icons, and screenshots where lossless fidelity matters. `image/png`. No HDR colour space support in the base spec (but HDR metadata via ICC profiles is possible).

**WebP** (Google; 2010) — a dual-mode codec: lossy WebP uses VP8 intra-frame coding (same DCT as VP8 video), lossless WebP uses a palette-based predictor with LZ77+Huffman. Supports animation (replacing animated GIF), alpha in both modes. 25–35% smaller than JPEG at equivalent quality. `image/webp`. Supported in all modern browsers.

**HEIF** (High Efficiency Image File Format; ISO/IEC 23008-12; 2015) is a **container**, not a codec. [Source](https://www.iso.org/standard/83102.html) It is an ISOBMFF-derived file format that can store any codec's intra frames as still images or image sequences:
- **HEIC** (`.heic`) — HEIF container + HEVC intra frames; Apple's default camera format on iPhone; supported natively in macOS/iOS; not supported in Chrome on Linux without a system decoder
- **AVIF** (`.avif`) — HEIF container + AV1 intra frames; royalty-free; the dominant modern still-image format for web use; `image/avif`

The HEIF container adds features the codec itself doesn't have: image collections (burst shots), image sequences (animations), thumbnail storage, depth maps, and auxiliary image items (alpha channel stored as a separate image item). An AVIF file is literally an AV1 Sequence Header OBU + a single AV1 frame (or multiple frames for animation) wrapped in HEIF metadata boxes.

**AVIF** (AOM; 2019): AV1 intra frames in HEIF. Best compression of any browser-supported still-image format — ~50% smaller than JPEG at equivalent quality, ~30% smaller than WebP. Supports HDR (10/12-bit), wide colour gamut (P3, Rec.2020), and lossless. Hardware decode follows AV1 hardware availability (Intel Tiger Lake+, AMD RDNA3+). `image/avif`. Supported Chrome 85+, Firefox 93+, Safari 16+.

**JPEG-XL** (ISO/IEC 18181; 2022) is the next-generation image codec designed to supersede JPEG, WebP, and PNG in a single format. [Source](https://jpeg.org/jpegxl/) Key properties:
- **Lossless JPEG recompression**: can transcode existing JPEG files to JPEG-XL losslessly and recover the original JPEG bitstream exactly — unique among image codecs; enables migration without re-encoding
- **Visually lossless quality** at ~60% of JPEG file size; comparable to AVIF but faster to encode/decode
- **HDR and wide gamut** natively: 32-bit float per channel, any colour space
- **Progressive decode**: a small header enables a rough preview; subsequent passes refine quality
- **Animation**: `image/jxl`

Browser support: Chrome removed its JPEG-XL trial in 2023 (citing insufficient ecosystem traction), then re-added it behind a flag in Chrome 126 (2024). Firefox has it behind `image.jxl.enabled`. Safari added support in Safari 18 (2024). As of mid-2026, JPEG-XL is supported by `ImageDecoder` in Chrome when the flag is enabled; `ImageDecoder` exposes HDR metadata for JPEG-XL via `ImageDecodeResult.image.colorSpace`.

**Image format comparison table:**

| Format | Type | Year | RF | Alpha | Animation | HDR | Compression vs JPEG | Browser support |
|--------|------|------|----|-------|-----------|-----|---------------------|-----------------|
| JPEG | Codec | 1992 | Yes | No | No | No | baseline | Universal |
| PNG | Codec+container | 2003 | Yes | Yes | No | Partial | larger (lossless) | Universal |
| WebP | Codec | 2010 | Yes | Yes | Yes | No | ~30% smaller | Chrome 23+, FF 65+, Safari 14+ |
| AVIF | AV1 in HEIF | 2019 | Yes | Yes | Yes | Yes | ~50% smaller | Chrome 85+, FF 93+, Safari 16+ |
| HEIC | HEVC in HEIF | 2015 | No | Yes | Yes | Yes | ~50% smaller | Safari only (Linux: no) |
| JPEG-XL | Codec | 2022 | Yes | Yes | Yes | Yes | ~40% smaller | Chrome 126+ (flag), Safari 18+, FF (flag) |

---

#### Codec Comparison Table

| Codec | Standard | Year | RF | Bitstream unit | Typical web use | HW decode (Linux) | HW encode (Linux) | Codec string |
|-------|----------|------|----|----------------|-----------------|-------------------|-------------------|--------------|
| H.264 / AVC | ISO 14496-10 | 2003 | Near† | NAL unit | DASH, HLS, WebRTC | Universal (Gen7+/GCN+) | Intel Gen7+, NVENC | `avc1.PPCCLL` |
| VP8 | RFC 6386 | 2010 | Yes | Partition frame | WebRTC (legacy) | Intel Gen6, some ARM | None | `vp8` |
| VP9 | AOM | 2013 | Yes | Superframe | YouTube, WebM | Intel Gen9+, RDNA1+, Pascal+ | Rare | `vp09.PP.LL.BB` |
| H.265 / HEVC | ISO 23008-2 | 2013 | No | NAL unit | Streaming (Safari) | Intel Gen9+ (flag), AMD | VA-API (iHD) | `hvc1.PP.CC.LX` |
| AV1 | AOM | 2018 | Yes | OBU | YouTube, Netflix, WebM | Intel Gen12+, RDNA3+, Ampere+ | Intel DG2+, AMD (buggy) | `av01.P.LLT.BB` |
| AAC-LC | ISO 13818-7 | 1997 | No | ADTS frame | MP4, DASH, HLS | Software only | Software only | `mp4a.40.2` |
| Opus | RFC 6716 | 2012 | Yes | Opus frame | WebRTC, WebM | Software only | Software only | `opus` |
| Vorbis | Xiph | 2000 | Yes | Vorbis packet | WebM (legacy) | Software only | Software only | `vorbis` |
| VP8 (audio: N/A) | | | | | | | | |
| WebP (image) | Google | 2010 | Yes | VP8 intra / custom | Web images | Via AV1/VP8 path | N/A | `image/webp` |
| AVIF (image) | AOM | 2019 | Yes | AV1 intra (HEIF) | Web images (HDR) | Intel Gen12+, RDNA3+ | N/A | `image/avif` |

†H.264 patents expire ~2028 in most jurisdictions; royalty-free for most current deployments under Via LA LLC terms.

**RF** = royalty-free. **HW decode/encode** refers to VA-API hardware paths available in Chrome on Linux as of mid-2026.

| Codec | Profile/level notation | `description` field | Notes |
|-------|----------------------|---------------------|-------|
| H.264 | `avc1.4d0034` — profile `4d`=Main, level `34`=5.2 hex | Binary `avcC` record (SPS+PPS); omit for Annex B | `avc.format: 'annexb'|'avc'` encoder option |
| VP9 | `vp09.00.40.08.00` — profile 0, level 4.0, 8-bit, 4:2:0 | None required | Profile 2 for 10-bit HDR |
| H.265 | `hvc1.1.6.L150.B0` — Main, compat=6, level 5.0 | Binary `hvcC` record (VPS+SPS+PPS) | Chrome requires flag on Linux |
| AV1 | `av01.0.08M.10` — Main, level 4.0, Main tier, 10-bit | Binary `av1C` record (Sequence Header OBU) | `av1C` = 4-byte config + raw OBU |
| Opus | `opus` | Binary `OpusHead` identification header | 8-byte magic + channels + pre-skip + sample rate |
| AAC-LC | `mp4a.40.2` | Binary AudioSpecificConfig (2–5 bytes) | Derived from `esds` box in MP4 |

---

#### What Is Better Than What: Selection Guides

"Better" is always context-dependent. The tables below rank each dimension explicitly.

**Video codec: compression efficiency** (same visual quality, lower bitrate = better):

```
AV1  >  H.265/HEVC  >  VP9  >  H.264  >  VP8
 ~50%        ~40%         ~30%     baseline    ~10% worse than H.264
 vs H.264   vs H.264    vs H.264
```

AV1 delivers the best quality per bit, but its encode is 5–20× slower than H.264 in software (SVT-AV1 narrows this gap). HEVC achieves near-AV1 quality but has royalty costs. VP9 is the royalty-free alternative before AV1 reached hardware support ubiquity.

**Video codec: hardware decode breadth** (more hardware = better real-world support):

```
H.264  >  VP9  >  H.265  >  AV1  >  VP8
(universal)  (Intel Gen9+,  (Intel Gen9+  (Intel Gen12+  (Intel Gen6,
              RDNA1+,        flag in        RDNA3+,        some ARM)
              Pascal+)       Chrome,        Ampere+)
                             AMD RDNA2+)
```

H.264 is the only codec with hardware decode on literally every GPU sold in the last 15 years. AV1 hardware decode is growing fast (all 2022+ GPUs) but absent on anything older.

**Video codec: royalty status**:
```
AV1 = VP9 = VP8 = Opus (royalty-free)  >  H.264 (near-free, ~2028)  >  H.265 (royalties)
```

**Practical video codec decision tree:**
```
Need maximum compatibility (all browsers, all devices)?  → H.264
Need royalty-free + good quality, broad HW decode?       → VP9
Need best compression, royalty-free, 2024+ devices?      → AV1
Need Safari HDR on Apple devices?                        → H.265 (HEVC)
Need WebRTC video?                                       → VP8 (legacy) or VP9/AV1 (modern)
```

**Audio codec ranking:**

| Goal | Best choice | Reason |
|------|-------------|--------|
| Web/WebRTC, royalty-free | **Opus** | Mandatory in WebRTC; best quality at all bitrates; royalty-free |
| MP4/DASH/HLS compatibility | **AAC-LC** | Default audio in every MP4 container; hardware encode everywhere |
| Legacy WebM files | **Vorbis** | Only if reading existing WebM; use Opus for new content |
| Highest quality (archival) | **FLAC** (via Matroska) | Lossless; but not in WebM or browser-native containers |

Opus beats AAC at every bitrate below 128 kbps according to subjective listening tests. Above 192 kbps the difference is inaudible. Use Opus for WebRTC and new WebM; use AAC for DASH/HLS where client device AAC hardware decode matters.

**Container decision tree:**
```
DASH or HLS streaming?                    → fMP4 / CMAF (interoperable)
Browser video, royalty-free codecs?       → WebM (VP9/AV1 + Opus)
Widest compatibility (any codec)?         → MP4 (ISO BMFF)
Archival, complex multi-track, subtitles? → Matroska (MKV)
HLS + MPEG-TS legacy pipeline?            → .ts (only if forced; prefer fMP4)
```

**Streaming protocol decision tree:**
```
Apple devices (iOS, tvOS, macOS Safari)?      → HLS required
Android / Chrome / Linux?                     → DASH preferred (or HLS via hls.js)
Both?                                         → CMAF origin + both manifests (single segment set)
Sub-2-second live latency?                    → LL-HLS or Chunked CMAF DASH
Standard live (< 10 s latency)?               → DASH (6–10 s) or HLS (15–30 s)
Real-time (< 500 ms, bidirectional)?          → WebRTC (SRTP/ICE, not HTTP segments)
```

**DRM decision tree:**
```
Chrome / Firefox / Android?   → Widevine (com.widevine.alpha)
Safari / iOS / Apple TV?      → FairPlay (com.apple.fps)
Edge / Xbox / Tizen?          → PlayReady (com.microsoft.playready)
All of the above?             → Multi-DRM: CENC (one CMAF segment set) + pssh boxes for all three
Linux Chrome specifically?    → Widevine L3 (software CDM); L1 requires TEE hardware not present on PC
Testing only (no real DRM)?   → ClearKey (org.w3.clearkey)
```

---

### 1.8 Why WebCodecs Instead of MSE or WebRTC?

Media Source Extensions feeds segments to the browser's internal decoder, but frames are never exposed to JavaScript — they go directly from the internal decoder to the compositor. WebRTC's `RTCPeerConnection` similarly hides the codec layer, exposing only rendered frames via `<video>` elements. Neither gives frame-level access or allows the application to choose per-frame processing.

#### Do MSE Level 2 and RTCRtpScriptTransform Make WebCodecs Redundant?

A natural question arises: MSE Level 2 added `appendEncodedChunks()` (accepting `EncodedVideoChunk` directly), and `RTCRtpScriptTransform` exposes the encoded bitstream inside a WebRTC pipeline. Don't those two additions close the gap?

They open cracks in the opaque pipelines, but WebCodecs is the toolbox that operates on what flows through those cracks. The gaps they leave are specific:

**`appendEncodedChunks()` does not provide:**
- Decoded frame access — frames still flow from the internal decoder directly to the compositor, never surfacing as JavaScript objects
- Any encoding capability — MSE has no encoder concept
- Freedom from the media timeline — MSE is still governed by a presentation clock; there is no way to decode at an arbitrary application-driven rate

**`RTCRtpScriptTransform` does not provide:**
- Decoded pixel data — `RTCEncodedVideoFrame.data` is the compressed bitstream (NAL units, VP9 superframes, AV1 OBUs), not decoded samples
- Rerouting of decoded output — frames still go through WebRTC's internal decoder into a `<video>` element; they cannot be redirected to `importExternalTexture()` or a Canvas without WebCodecs in the chain
- An encoder with explicit control — the WebRTC encoder is opaque; you cannot force keyframes on demand or switch bitrate modes programmatically

**What WebCodecs uniquely provides:**
1. **Decoded `VideoFrame` as a first-class JS object** — CPU access via `copyTo()`, GPU access via `importExternalTexture()`, `drawImage()`, or `texImage2D()`
2. **No playback clock** — decode on demand, out of order, at application-driven rate (game streaming scrubbing, frame-by-frame ML inference, custom A/V sync)
3. **Any encoded source** — file bytes, WASM-generated bitstreams, network blobs, `RTCEncodedVideoFrame.data` — not limited to HTTP segments or RTP packets
4. **Explicit encoder** — configurable bitrate, latency mode, keyframe forcing, codec-specific options; MSE has no equivalent
5. **Cross-source compositing** — simultaneously decode a file stream and a WebRTC stream, composite both into one WebGPU scene; neither MSE nor WebRTC alone can express this

In practice, the three APIs form a **pipeline, not alternatives**:

```
Camera → RTCPeerConnection → RTCRtpScriptTransform
         (encoded frames extracted)
              ↓ RTCEncodedVideoFrame.data
         VideoDecoder (WebCodecs)
              ↓ VideoFrame
         importExternalTexture() → WebGPU compositor
              ↓ WebGPU rendered output
         VideoEncoder (WebCodecs) → appendEncodedChunks()
                                    → MSE Level 2 → <video> timed playback
```

`RTCRtpScriptTransform` is the *source tap*; `appendEncodedChunks()` is the *downstream sink*; WebCodecs is the *processing stage* between them. This pipeline underpins virtual backgrounds, real-time transcoding, and server-side rendering in browser-based production tools.

WebCodecs is deliberately designed around four principles missing from those APIs:

- **Frame-level access**: Each decoded `VideoFrame` is a JavaScript object wrapping a GPU surface or CPU buffer, accessible via `copyTo()` for CPU data or passable directly to WebGPU's `importExternalTexture()`.
- **Low latency**: The API is non-blocking and queue-based. `decode()` returns immediately; frames are delivered via output callback, not promises, to avoid microtask scheduling overhead.
- **Worker thread operation**: All four interfaces — `VideoDecoder`, `VideoEncoder`, `AudioDecoder`, `AudioEncoder` — are exposed on `DedicatedWorker` (and `SharedWorker` for audio), allowing media work to run off the main thread.
- **Hardware acceleration transparency**: `isConfigSupported()` reveals whether a given codec configuration can be hardware-accelerated before the decoder is created. `MediaCapabilities.decodingInfo()` exposes `powerEfficient` as a hardware-decode proxy.

The W3C WebCodecs specification defines the following primary interfaces [Source](https://www.w3.org/TR/webcodecs/#video-decoder-interface):

```webidl
[Exposed=(Window,DedicatedWorker), SecureContext]
interface VideoDecoder : EventTarget {
  constructor(VideoDecoderInit init);
  readonly attribute CodecState state;
  readonly attribute unsigned long decodeQueueSize;
  attribute EventHandler ondequeue;
  undefined configure(VideoDecoderConfig config);
  undefined decode(EncodedVideoChunk chunk);
  Promise<undefined> flush();
  undefined reset();
  undefined close();
  static Promise<VideoDecoderSupport> isConfigSupported(VideoDecoderConfig config);
};

[Exposed=(Window,DedicatedWorker), SecureContext]
interface VideoEncoder : EventTarget {
  constructor(VideoEncoderInit init);
  readonly attribute CodecState state;
  readonly attribute unsigned long encodeQueueSize;
  undefined configure(VideoEncoderConfig config);
  undefined encode(VideoFrame frame, optional VideoEncoderEncodeOptions options = {});
  Promise<undefined> flush();
  undefined reset();
  undefined close();
  static Promise<VideoEncoderSupport> isConfigSupported(VideoEncoderConfig config);
};
```

Audio counterparts (`AudioDecoder`, `AudioEncoder`) follow the same structural pattern. `ImageDecoder` handles still image formats (JPEG, PNG, AVIF, WebP) with full ICC profile and HDR metadata preservation. This chapter focuses on the video path and its hardware acceleration story on Linux.

---

## 2. VideoDecoder: Chunk Submission, Frame Delivery, and Lifecycle

### 2.1 Codec String Negotiation with `isConfigSupported`

Before creating a decoder, applications should probe hardware support. The `isConfigSupported()` static method performs a lightweight capability query against the GPU process without allocating a full decoder instance:

```javascript
// third_party/blink/renderer/modules/webcodecs/video_decoder.cc (simplified JS usage)
const support = await VideoDecoder.isConfigSupported({
  codec: 'avc1.4d0034',        // H.264 Main Profile, Level 5.2
  codedWidth: 1920,
  codedHeight: 1080,
  hardwareAcceleration: 'prefer-hardware',  // 'no-preference' | 'prefer-hardware' | 'prefer-software'
});
// support.supported: boolean
// support.config: the echoed-back config that is supported (may differ from input)
console.log(support.supported, support.config.codec);
```

The codec string format matters and follows the codec registry [Source](https://www.w3.org/TR/webcodecs-codec-registry/):

| Codec | Example String | Interpretation |
|-------|---------------|----------------|
| H.264 | `avc1.4d0034` | Profile byte `4d` = Main Profile; Level `34` = 5.2 |
| VP9 | `vp09.00.40.08.00` | Profile 0, Level 4.0, 8-bit, 4:2:0 |
| AV1 | `av01.0.08M.08` | Main Profile, Level 4.0, Main Tier, 8-bit |
| HEVC | `hvc1.1.6.L150.B0` | Main Profile, CompatFlags=6, Level 5.0 |
| VP8 | `vp8` | No profile/level suffix required |

Under the hood, `isConfigSupported()` reaches the GPU process via `VideoDecoderBroker`, which queries `media::VideoDecoderConfig` capability through `GpuVideoAcceleratorFactories`. The VA-API capability map is built at GPU process startup (see Section 4) and returned synchronously from the cached `GpuInfo` structure.

### 2.2 Creating a Decoder and Submitting Chunks

```javascript
// Ensure WebCodecs is available (requires SecureContext)
if (!('VideoDecoder' in self) || !self.isSecureContext) {
  throw new Error('WebCodecs requires a secure context (HTTPS or localhost)');
}

const decoder = new VideoDecoder({
  // output callback: called on the GPU thread's output event loop,
  // posted to the JS task queue. Frame MUST be closed when no longer needed.
  output(frame) {
    processFrame(frame);
    frame.close(); // CRITICAL: releases the underlying GPU surface reference
  },
  error(e) {
    console.error('Decoder error:', e.message);
  },
});

// configure() must be called before any decode()
// For H.264, 'description' carries the AVCC extradata (SPS+PPS in binary)
decoder.configure({
  codec: 'avc1.4d0034',
  codedWidth: 1920,
  codedHeight: 1080,
  description: avccExtradata,  // ArrayBuffer; optional for Annex B streams
  optimizeForLatency: true,    // hint to avoid buffering (encoding order ≠ display order)
});

// Submit encoded data as EncodedVideoChunks
// First chunk after configure() MUST be type 'key'
decoder.decode(new EncodedVideoChunk({
  type: 'key',     // 'key' (IDR) or 'delta' (P/B-frame)
  timestamp: 0,    // microseconds; used for VideoFrame.timestamp
  duration: 33333, // microseconds; optional
  data: keyframeBytes,  // ArrayBuffer or BufferSource
}));

// Subsequent delta frames
decoder.decode(new EncodedVideoChunk({
  type: 'delta',
  timestamp: 33333,
  data: deltaFrameBytes,
}));

// flush(): resolves when all pending decode() calls have produced output frames
await decoder.flush();
decoder.close();
```

### 2.3 Backpressure and `decodeQueueSize`

`decode()` is fire-and-forget: it enqueues the chunk and returns immediately. The `decodeQueueSize` attribute reflects how many chunks are queued waiting for the hardware decoder. Applications must implement their own backpressure logic:

```javascript
async function submitChunk(decoder, chunk) {
  // Pause submission if the decoder is backed up
  if (decoder.decodeQueueSize > 5) {
    await new Promise(resolve => {
      // ondequeue fires when an item is dequeued
      decoder.addEventListener('dequeue', resolve, { once: true });
    });
  }
  decoder.decode(chunk);
}
```

Without backpressure management, submitting faster than the hardware decoder can process will cause unbounded memory growth as compressed chunks accumulate in the queue.

### 2.4 Lifecycle: State Machine and Error Handling

`VideoDecoder.state` takes values `"unconfigured"`, `"configured"`, and `"closed"`. The `reset()` method returns the decoder to `"unconfigured"` without closing it, flushing all pending decode operations and dropping their output — essential when seeking (discarding buffered B-frames and resuming from a key frame). The `close()` method releases all resources including the GPU process reference.

The error callback receives a `DOMException`. Errors during `decode()` (e.g., a malformed NAL unit on the hardware path) are reported asynchronously via this callback, not as synchronous throws. After an error callback, the decoder state transitions to `"closed"` and the decoder must be recreated.

`EncodedVideoChunkType.key` vs `delta` has a critical implication: submitting a `delta` chunk when the decoder has not seen a preceding key frame (after `configure()` or `reset()`) produces an error callback from hardware decoders. Software decoders may produce corrupted frames silently. Always verify stream sync-point alignment before calling `decode()`.

### 2.5 VideoFrame Properties

The `VideoFrame` delivered to the output callback carries:

- `timestamp`, `duration` — echoed from the input `EncodedVideoChunk`
- `codedWidth`, `codedHeight` — the dimensions including padding (aligned to macroblock boundaries)
- `displayWidth`, `displayHeight` — the intended display dimensions (may differ due to SAR)
- `visibleRect` — the crop rectangle within the coded frame
- `colorSpace` — a `VideoColorSpace` object with `primaries`, `transfer`, `matrix`, `fullRange`
- `format` — pixel format (`"I420"`, `"NV12"`, `"RGBA"`, `"RGBX"`, etc.) when the frame is CPU-readable; `null` for GPU-only frames

A `VideoFrame` backed by a hardware-decoded GPU surface will have `format === null` until `copyTo()` materialises the pixel data to CPU memory. The GPU-backed frame can be passed directly to `drawImage()`, `texImage2D()`, or `importExternalTexture()` without any CPU-side copy.

---

## 3. VideoEncoder: Parameters, Latency Modes, and Key Frame Control

### 3.1 Configuration

```javascript
const encoder = new VideoEncoder({
  output(chunk, metadata) {
    // metadata.decoderConfig is present on key frames only
    // metadata.decoderConfig.description: AVCC/HVCC binary blob (SPS/PPS for H.264)
    if (metadata?.decoderConfig) {
      storeDecoderConfig(metadata.decoderConfig);
    }
    sendToNetwork(chunk);
  },
  error(e) { console.error(e); },
});

encoder.configure({
  codec: 'avc1.4d0034',
  width: 1280,
  height: 720,
  bitrate: 2_500_000,          // bits per second (average for VBR, target for CBR)
  framerate: 30,
  latencyMode: 'realtime',     // 'realtime' | 'quality' — critical for streaming
  bitrateMode: 'variable',     // 'constant' | 'variable' (default)
  hardwareAcceleration: 'prefer-hardware',
  // H.264-specific options:
  avc: {
    format: 'annexb',          // 'annexb' (start codes) | 'avc' (length-prefixed AVCC)
  },
});
```

### 3.2 `latencyMode` Semantics

The `latencyMode` parameter controls the fundamental encode strategy [Source](https://www.w3.org/TR/webcodecs/#dom-videoencoderconfig-latencymode):

- **`"realtime"`**: The encoder treats the configured bitrate as a hard deadline constraint per frame. It will not buffer frames across a B-frame window, will use I/P-only coding structure, and will drop quality (increase quantizer) rather than exceed a per-frame size budget. This mode is mandatory for live streaming, conferencing, and game capture where late frames are worse than lower-quality frames. VA-API hardware encoders map this to `VA_RC_CBR` or `VA_RC_VBR` with a tight HRD buffer.

- **`"quality"` (default)**: The encoder may buffer multiple frames to enable B-frames, lookahead-based bitrate allocation, and scene-change detection. Output chunks may arrive delayed relative to the input `VideoFrame`. This mode is appropriate for file export or progressive download.

### 3.3 Key Frame Forcing and Encode Options

```javascript
// Per-encode options
const frameIndex = 0;
for await (const videoFrame of frameSource) {
  // Back-pressure: drop frames if encoder is congested
  if (encoder.encodeQueueSize > 2) {
    videoFrame.close();
    continue;
  }

  const forceKeyFrame = (frameIndex % 60 === 0); // force keyframe every 2 seconds at 30fps
  encoder.encode(videoFrame, {
    keyFrame: forceKeyFrame,  // overrides encoder's internal IDR cadence
  });
  videoFrame.close(); // MUST close the frame after encode() — does not take ownership
  frameIndex++;
}

await encoder.flush();
encoder.close();
```

`keyFrame: true` in the encode options is the equivalent of requesting an IDR. Hardware encoders honour this via the `H264EncodeJob::keyframe` flag in `media/gpu/h264_dpb.h` or the analogous field in the VA-API encode delegate. Note that for `latencyMode: "realtime"` the encoder may insert key frames at its own discretion based on scene change detection regardless of this flag.

### 3.4 `bitrateMode` and Codec-Specific Options

`bitrateMode: "constant"` maps to `VA_RC_CBR`; `bitrateMode: "variable"` maps to `VA_RC_VBR`. A proposed `"quantizer"` mode (not yet standardised as of mid-2026) would allow per-frame quantizer control without bitrate management. [Source](https://gist.github.com/Djuffin/3722232679b977058be787be0dff4254)

AV1-specific options (`av1: { bitrateMode }`) and VP9 options (`vp9: { profile }`) follow the codec-specific options dictionary pattern established in the spec. For HEVC, the `hevc: { profile }` dictionary allows selecting Main10 for 10-bit HDR encode.

### 3.5 Hardware Timestamp Access via `VideoFrameMetadata`

When an input `VideoFrame` originates from a camera via `MediaStreamTrackProcessor`, its `VideoFrameMetadata` carries hardware-level timestamps — the capture time from the camera driver, distinct from the presentation timestamp. Encoders that set `timestamp` from `VideoFrameMetadata.captureTime` rather than wall-clock time can achieve tighter synchronisation in conferencing pipelines. Note: `VideoFrameMetadata` is non-standard in some of its more detailed fields and varies across browsers.

---

## 4. The Hardware Acceleration Path on Linux

This section traces the full call path from a JavaScript `VideoDecoder.decode()` call to the VA-API `vaEndPicture()` invocation in the kernel DRM driver. Understanding this path is essential for debugging hardware acceleration failures and for understanding the performance characteristics of the WebCodecs pipeline.

### 4.1 Blink Layer: `VideoDecoder` and `VideoDecoderBroker`

The JavaScript `VideoDecoder` object is implemented in Blink as `blink::VideoDecoder` (`third_party/blink/renderer/modules/webcodecs/video_decoder.h`), a `ScriptWrappable` that holds a `std::unique_ptr<media::VideoDecoder>`. However, the actual `media::VideoDecoder` instance it holds is a `VideoDecoderBroker` — a thin client-side proxy that does not decode anything itself but rather marshals all operations across the GPU process boundary.

```
// Simplified class relationships (from Chromium source tree)
// third_party/blink/renderer/modules/webcodecs/
//   video_decoder.h:     class VideoDecoder : public ScriptWrappable
//   video_decoder_broker.h: class VideoDecoderBroker : public media::VideoDecoder
```

`VideoDecoderBroker` accesses `Platform::Current()->GetGpuFactories()` to obtain a `GpuVideoAcceleratorFactories*`, which it uses to instantiate a `MojoVideoDecoder` — the renderer-side endpoint of the Mojo IPC channel to the GPU process. [Source](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/modules/webcodecs/video_decoder_broker.h)

A key detail is the `GpuFactoriesRetriever` (`third_party/blink/renderer/modules/webcodecs/gpu_factories_retriever.h`): because WebCodecs may run in a `DedicatedWorker` with no `RenderFrame`, the GPU factories must be obtained via a special worker-compatible path. This is what enables off-main-thread decode — a `DedicatedWorker` can call `VideoDecoder.decode()` without touching the main thread.

The hardcoded preferred resolution of `1280×720` in `VideoDecoderBroker` biases hardware decoder selection toward codecs and profiles that the hardware supports at that resolution. This is an implementation heuristic, not a spec requirement. [Source](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/modules/webcodecs/video_decoder_broker.cc)

### 4.2 Mojo IPC: `MojoVideoDecoder` and `FramelessMediaInterfaceProxy`

In a normal HTML `<video>` decode path, the renderer calls `media::mojom::VideoDecoder` Mojo interfaces bound to a specific `RenderFrame`. WebCodecs has no `RenderFrame` when running in a worker — it uses `FramelessMediaInterfaceProxy` instead.

`FramelessMediaInterfaceProxy` routes `media::mojom::InterfaceFactory` requests (for video decoders, audio decoders, etc.) from frameless contexts to the `MediaService` registered in the GPU process. This proxy is shared between WebCodecs, WebRTC's media pipeline, and codec capability probing. [Source](https://source.chromium.org/chromium/chromium/src/+/main:media/mojo/README.md)

On the GPU process side, `MojoVideoDecoderService` receives the Mojo connection and instantiates the platform-specific hardware decoder via `GpuMojoMediaClient` — the GPU process's implementation of `mojom::InterfaceFactory`. For Linux, `GpuMojoMediaClient::CreateVideoDecoder()` creates a `VaapiVideoDecoder`.

The full call path is:

```
JS VideoDecoder.decode(EncodedVideoChunk)
  → blink::VideoDecoder::decode()              [renderer/worker process]
    → VideoDecoderBroker::Decode()             [renderer/worker process]
      → MojoVideoDecoder (client side)         [Mojo IPC pipe]
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ IPC boundary ━━━
      → MojoVideoDecoderService::Decode()      [GPU process]
        → VaapiVideoDecoder::Decode()          [GPU process]
          → VaapiWrapper + VA-API delegates    [GPU process, libva call]
            → libva → iHD driver              [userspace driver]
              → kernel DRM/i915               [kernel]
  Output path (reverse):
              ← VASurface decoded
            ← ExportVASurfaceAsNativePixmapDmaBuf()
          ← gfx::NativePixmapDmaBuf wrapping DMA-BUF fd
        ← SharedImage (SharedImageBackingOzone)
      ← VideoFrame sent back via MojoVideoDecoderService::OnVideoFrameDecoded()
    ← blink::VideoDecoder output callback invoked
  → JS output(frame) callback called
```

### 4.3 VA-API Layer: `VaapiVideoDecoder` and `VaapiWrapper`

`VaapiVideoDecoder` (`media/gpu/vaapi/vaapi_video_decoder.h`) inherits from `VideoDecoderMixin` and implements the hardware decode state machine. Its state transitions are: `kUninitialized → kWaitingForInput → kDecoding → kChangingResolution | kWaitingForOutput | kResetting → kError`. [Source](https://source.chromium.org/chromium/chromium/src/+/main:media/gpu/vaapi/vaapi_video_decoder.h)

The decoder maintains a pool of VA surfaces via `DmabufVideoFramePool`. Each surface is a `VASurfaceID` allocated by `VaapiWrapper::CreateVASurfaceForFrameResource()`, which in turn calls `vaCreateSurfaces()` from libva. The critical lifetime constraint: `allocated_va_surfaces_` (a `flat_map<VASurfaceID, ScopedVASurface>`) keeps each surface alive until all contexts using it are destroyed. This is a libva requirement: `vaDestroySurfaces()` cannot be called while a surface is referenced by a decode context.

Codec-specific decoding logic is handled by delegates:
- `H264VaapiVideoDecoderDelegate` — parses SPS/PPS, manages DPB, calls `vaBeginPicture/vaRenderPicture/vaEndPicture`
- `VP9VaapiVideoDecoderDelegate` — manages reference frame tracking for VP9's probabilistic model
- `AV1VaapiVideoDecoderDelegate` — tile group submission, superframe handling
- `H265VaapiVideoDecoderDelegate` — manages NAL slice parameters for HEVC

`VaapiWrapper` (`media/gpu/vaapi/vaapi_wrapper.h`) is the singleton-per-profile C++ wrapper around libva. Its key export function for the zero-copy path is [Source](https://source.chromium.org/chromium/chromium/src/+/main:media/gpu/vaapi/vaapi_wrapper.h):

```cpp
// media/gpu/vaapi/vaapi_wrapper.h
VaapiStatus::Or<std::unique_ptr<NativePixmapAndSizeInfo>>
ExportVASurfaceAsNativePixmapDmaBuf(const ScopedVASurface& va_surface);
```

This calls `vaExportSurfaceHandle()` with `VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2` (libva 2.1 / VA-API 1.1.0) and `VA_EXPORT_SURFACE_READ_ONLY | VA_EXPORT_SURFACE_SEPARATE_LAYERS`. The `VADRMPRIMESurfaceDescriptor` returned describes up to four DMA-BUF file descriptors (one per NV12 plane for separate-layer export), each with a DRM format modifier for scanout validation.

The `vaExportSurfaceHandle()` function was introduced in **libva 2.1 (VA-API 1.1.0)**, merged November 2017 [Source](https://github.com/intel/libva/commit/51e98b1). It allows export of a decoded surface as a DRM PRIME DMA-BUF without a CPU-side copy, which is the foundational primitive for the entire zero-copy display path.

### 4.4 V4L2 Stateless Backend (ARM/Embedded Only)

On ARM-based Linux systems — ChromeOS-on-ARM, Rockchip RK3568/3588, Allwinner H6, MediaTek MT8195 — Chrome uses a V4L2 stateless backend (`v4l2_video_decode_accelerator`) rather than VA-API. The Linux kernel stabilised the V4L2 stateless H.264 uAPI in **Linux 5.11** (merged by Collabora/Ezequiel Garcia) and V4L2 stateless AV1 uAPI in **Linux 6.5** for Rockchip RK3588 and MediaTek MT8195. [Source](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/ext-ctrls-codec-stateless.rst)

**This backend is not used on desktop x86 Linux.** The VA-API and V4L2 backends are mutually exclusive alternatives targeting different hardware platforms. Desktop Chrome on x86/x86-64 exclusively uses VA-API. Do not conflate the V4L2 kernel version story with the desktop Chrome hardware acceleration timeline.

### 4.5 iHD vs i965 vs AMD: A Critical Driver Distinction

The current `VaapiVideoDecoder` class (replacing the deprecated `VaapiVideoDecodeAccelerator`) has a practical dependency on the **Intel iHD driver** (`iHD_drv_video.so`, `intel-media-driver` package). It uses libva-drm (not libva-x11) and relies on DMA-BUF export semantics that the iHD driver implements reliably. The older `i965` driver (`i965_drv_video.so`, `libva-intel-driver` package) lacks these export semantics and is not supported by the current code path.

AMD GPU hardware decode via VA-API (Mesa Gallium radeonsi VA frontend) requires the `VaapiIgnoreDriverChecks` Chrome feature flag, which bypasses driver allowlist checks. NVIDIA hardware decode requires both `VaapiIgnoreDriverChecks` and `VaapiOnNvidiaGPUs` flags, using the `nvidia-vaapi-driver` userspace shim. Neither AMD nor NVIDIA is supported by default in stable Chrome releases. Chrome 143 enabled hardware decode by default on Wayland, but only for Intel iHD.

---

## 5. VideoFrame and DMA-BUF: Zero-Copy from Decoder to Canvas

### 5.1 `VideoFrameHandle` and `SharedImage`

In Chromium's implementation, a `VideoFrame` object delivered to the WebCodecs output callback is not raw pixel data — it is a handle to a `SharedImage`, which in turn wraps a `gfx::NativePixmapDmaBuf`. The chain is:

```
VASurfaceID (libva)
  → vaExportSurfaceHandle() → DMA-BUF file descriptors
    → gfx::NativePixmapDmaBuf (wraps fds + DRM format modifier)
      → SharedImageBackingOzone (GPU process SharedImage backend)
        → media::VideoFrame (GPU memory storage, backed by SharedImage mailbox)
          → VideoFrameHandle (reference-counted handle in WebCodecs)
            → JS VideoFrame object
```

`SharedImageBackingOzone` manages buffers backed by `gfx::NativePixmapDmaBuf` and can create simultaneous GL (`EGLImage` via `EGL_EXT_image_dma_buf_import`), Vulkan (`VkImage` via `VK_EXT_external_memory_dma_buf`), and overlay representations from a single DMA-BUF. This enables zero-copy across all rendering APIs. [Source](https://source.chromium.org/chromium/chromium/src/+/main:gpu/command_buffer/service/shared_image/ozone_image_backing.h)

`DmabufVideoFramePool` (`media/gpu/chromeos/dmabuf_video_frame_pool.h`) is the pool feeding `VaapiVideoDecoder::CreateSurface()`. The pool pre-allocates a fixed number of DMA-BUF-backed `VideoFrame` objects and recycles them as the JS `VideoFrame.close()` call releases references. Critically, `VideoFrame.close()` does not immediately call `vaDestroySurfaces()` — it decrements the reference count on the `VideoFrameHandle`, and the backing `NativePixmapDmaBuf` is returned to the pool when the count reaches zero.

### 5.2 The Zero-Copy Path in Detail

On Wayland with EGL, the zero-copy decode-to-display path is:

1. `VaapiVideoDecoder` decodes into a `VASurface` backed by a kernel GEM buffer object (on Intel: i915 BO; on AMD: amdgpu BO).
2. `vaExportSurfaceHandle()` exports the GEM BO as a DMA-BUF fd with a DRM format modifier.
3. `SharedImageBackingOzone` wraps the fd in a `gfx::NativePixmapDmaBuf`, creates a `SharedImage` with both GL and overlay representations.
4. The GL representation uses `EGL_EXT_image_dma_buf_import_modifiers` to import the DMA-BUF as an `EGLImage`, bound to a GL texture.
5. The Vulkan representation uses `VK_EXT_external_memory_dma_buf` + `VK_EXT_image_drm_format_modifier` to import as a `VkImage`.
6. When `drawImage(videoFrame, ...)` or `importExternalTexture({ source: videoFrame })` is called, the GPU can sample the YUV data directly from the decoded surface.

No pixel data ever touches CPU memory in this path. The decoded frame goes from the kernel DRM driver directly to the compositor scan-out via the GPU's texture sampler.

### 5.3 Synchronisation: `dma_fence` and `sync_file`

Because the VA-API decoder, the GL texture sampler, and potentially the Wayland KMS overlay all access the same DMA-BUF, synchronisation is critical. Modern DMA-BUF synchronisation uses implicit fencing via the `dma_fence` kernel mechanism. The DRM driver attaches a fence to the BMA-BUF; EGL's `eglWaitSyncKHR` and Vulkan's `VkSemaphore` (via `VK_KHR_external_semaphore_fd`) wait on these fences without requiring CPU-side `vaSyncSurface()` calls.

On the Wayland compositor side, explicit sync (`linux-explicit-synchronization` protocol or the newer `linux-drm-syncobj-v1`) carries these fences across the compositor boundary, allowing the KMS driver to wait for decode completion before scanning out the frame.

### 5.4 Zero-Copy Camera Pipeline

A `VideoFrame` need not come from a decoder. Camera capture via `MediaStreamTrackProcessor` yields `VideoFrame` objects directly from the camera driver's V4L2 output buffers (again as DMA-BUFs on supported hardware). The resulting pipeline is:

```
Camera V4L2 output (DMA-BUF)
  → MediaStreamTrackProcessor → ReadableStream<VideoFrame>
    → JS processing (WebGPU compute, color correction, etc.)
      → VideoEncoder.encode(frame)
        → VaapiVideoEncoder (hardware H.264/AV1 encode)
          → Output chunks → WebRTC / fetch upload
```

Each step can be zero-copy if the GPU-backed `VideoFrame` is not converted to CPU memory by `copyTo()`. The caveat is that `MediaStreamTrackProcessor` was originally documented as worker-only but Chrome exposes it on the main thread as well — a conformance gap to be aware of when targeting multiple browsers.

---

## 6. WebCodecs + WebGL and WebGPU Integration

### 6.1 Drawing VideoFrames to WebGL

The `texImage2D()` overload accepting a `VideoFrame` allows direct upload from a decoded hardware surface to a WebGL texture:

```javascript
// WebGL VideoFrame upload
// third_party/blink/renderer/modules/webgl/webgl_rendering_context_base.cc
// handles VideoFrame as an TexImageSource

const gl = canvas.getContext('webgl2');
const texture = gl.createTexture();
gl.bindTexture(gl.TEXTURE_2D, texture);

// Upload VideoFrame to GL texture
// For hardware-decoded frames, this may use EGL_EXT_image_dma_buf_import internally
// to create an EGLImage from the DMA-BUF — avoiding a CPU copy on Wayland/EGL.
// On X11/GLX, a copy occurs because GLX cannot import DMA-BUF directly.
gl.texImage2D(
  gl.TEXTURE_2D,
  0,                   // mip level
  gl.RGBA,             // internal format
  gl.RGBA,             // format
  gl.UNSIGNED_BYTE,    // type
  videoFrame           // source: VideoFrame (hardware or software)
);

// Note: for YUV hardware frames, Chrome internally performs YUV→RGB conversion
// during the texImage2D call if a direct DMA-BUF import as YUV is not supported
// by the GL extension profile. Use WebGPU importExternalTexture for true YUV sampling.
```

The `WEBGL_webcodecs_video_frame` Khronos proposed extension (not yet in the WebGL registry as a standard as of mid-2026) would enable a 0-copy path: `importVideoFrame(videoFrame)` returns a `WebGLWebCodecsVideoFrameHandle` that locks the frame to a GL texture target until `releaseVideoFrame(handle)` is called, avoiding any copy on EGL/Wayland systems.

### 6.2 WebGPU `importExternalTexture` with `VideoFrame`

The WebGPU path is the highest-performance option for hardware-decoded video:

```javascript
// device: GPUDevice; videoFrame: VideoFrame from VideoDecoder output callback
// third_party/blink/renderer/modules/webgpu/ implements this path

// GPUExternalTexture from VideoFrame — lifetime tied to VideoFrame.close(),
// NOT to the end of the current task (unlike HTMLVideoElement sources)
const externalTexture = device.importExternalTexture({ source: videoFrame });

// Non-standard Chrome developer attribute — indicates zero-copy:
// true  = GPU can sample directly from the decoded DMA-BUF surface
// false = a copy to a GPU-side RGBA texture was required
// console.log('zero copy:', externalTexture.isZeroCopy);

// CRITICAL: do NOT await between importExternalTexture() and draw calls
// For VideoFrame sources, the texture is valid until videoFrame.close(),
// but for HTMLVideoElement sources it expires at the next microtask boundary.
const bindGroup = device.createBindGroup({
  layout: pipeline.getBindGroupLayout(0),
  entries: [{
    binding: 0,
    resource: externalTexture,
  }],
});

const commandEncoder = device.createCommandEncoder();
const renderPass = commandEncoder.beginRenderPass(renderPassDescriptor);
renderPass.setPipeline(pipeline);
renderPass.setBindGroup(0, bindGroup);
renderPass.draw(6); // two triangles for a fullscreen quad
renderPass.end();
device.queue.submit([commandEncoder.finish()]);

// Release the frame — after this, externalTexture is invalid
videoFrame.close();
```

The WGSL shader for sampling a `GPUExternalTexture` uses the `texture_external` type:

```wgsl
// WGSL shader for sampling an external (YUV) video texture
// The textureSampleBaseClampToEdge() requirement prevents edge bleeding
@group(0) @binding(0) var videoTex: texture_external;
@group(0) @binding(1) var samp: sampler;

@fragment
fn fs_main(@location(0) texCoords: vec2<f32>) -> @location(0) vec4<f32> {
  // textureSampleBaseClampToEdge required for texture_external (not textureSample)
  return textureSampleBaseClampToEdge(videoTex, samp, texCoords);
}
```

The `texture_external` type in WGSL is Dawn's representation of a `GPUExternalTexture`. When the backing frame is NV12 (the standard VA-API output format), Dawn internally handles the YUV-to-RGB conversion in the shader using the frame's embedded `VideoColorSpace` metadata. This conversion is performed on the GPU's sampler unit without touching CPU memory.

### 6.3 `GPUExternalTexture` Lifetime: A Critical Difference Between Sources

The lifetime semantics differ between `HTMLVideoElement` and `VideoFrame` sources — a common source of bugs:

| Source | Texture expiry |
|--------|---------------|
| `HTMLVideoElement` | Expires immediately after the current task/microtask (any `await` invalidates it) |
| `VideoFrame` | Expires only when `videoFrame.close()` is explicitly called |

This difference enables pipeline buffering with `VideoFrame` sources: an application can import a texture, submit multiple render passes referencing it, wait for GPU work to complete, and then close the frame — all in an async pipeline. With `HTMLVideoElement` sources, this pattern fails silently because the texture has already expired.

```javascript
// CORRECT: VideoFrame-sourced texture survives across awaits
const frame = await frameQueue.pop(); // VideoFrame from WebCodecs output
const tex = device.importExternalTexture({ source: frame });
// ... encode render commands referencing tex ...
await device.queue.onSubmittedWorkDone(); // GPU work completes
frame.close(); // NOW release — tex has been fully consumed

// BUG: HTMLVideoElement-sourced texture does NOT survive across awaits
const videoTex = device.importExternalTexture({ source: videoElement });
await someAsyncOperation(); // tex is now INVALID — silent bug
renderPass.setBindGroup(0, bindGroup); // uses expired texture
```

### 6.4 `SharedImage` Architecture in Chrome

Under the hood, the `GPUExternalTexture` mechanism in Chrome is implemented via the `SharedImage` subsystem. A `SharedImage` is a GPU-process-owned buffer that multiple backends (GL, Vulkan, overlay, Dawn) can represent simultaneously without copying. For a hardware-decoded frame:

1. `SharedImageBackingOzone` wraps the DMA-BUF as a `SharedImage`.
2. On WebGPU import, Dawn's `ExternalTextureImpl` creates a `DawnSharedImage` representation.
3. Dawn wraps the DMA-BUF as a `VkImage` via `VK_EXT_external_memory_dma_buf`, using the DRM format modifier for layout information.
4. The Tint-compiled WGSL shader then samples this `VkImage` directly.

This is why the `isZeroCopy` non-standard attribute can be `true` on Wayland/Vulkan: the entire path from VA-API decoder to WebGPU shader involves no CPU copy and no format conversion until the shader's built-in YUV sampler.

---

## 7. Codec Support Matrix on Linux

The following table reflects the state of hardware codec support in Chrome on desktop Linux as of mid-2026. "Default on" refers to enabled-by-default status in stable Chrome without flags.

| Codec | Intel iHD (Xe, Arc) | AMD (Mesa Gallium) | NVIDIA (nvidia-vaapi-driver) | Chrome version | Default on |
|-------|--------------------|--------------------|------------------------------|----------------|-----------|
| H.264 decode | HD 4000+ (Gen7+) | RDNA1+ | Decode, requires flags | 108+ | Chrome 143+ (Wayland) |
| VP9 decode | Gen9 Skylake+ | RDNA1+ | Decode, requires flags | 108+ | Chrome 143+ (Wayland) |
| HEVC decode | Gen9+ (8-bit), Gen12+ (10-bit) | RDNA2+ | Decode, requires flags | 108+ | Chrome 143+ (Wayland) |
| AV1 decode | Gen12 Tiger Lake+ | RDNA3+ (RX 7xxx) | Not supported | 108+ | Chrome 143+ (Wayland) |
| H.264 encode | Gen7+ | Limited | Not supported | Requires `AcceleratedVideoEncoder` flag | No |
| VP9 encode | Not via VA-API | Not via VA-API | Not supported | Not supported | No |
| AV1 encode | DG2/Arc, Meteor Lake+ | Broken (DPB bug) | Not supported | Requires `AcceleratedVideoEncoder` flag | No |
| VP8 decode | Gen6 Ivy Bridge+ | Limited | Not supported | 108+ | No |
| AAC audio | Software only (libfdk-aac) | N/A | N/A | Chrome build-dependent | N/A |
| Opus audio | Software only | N/A | N/A | All versions | Yes |

### 7.1 Intel AV1 Decode (Tiger Lake+)

AV1 hardware decode via VA-API requires `VAProfileAV1Profile0` with `VAEntrypointVLD`. This entrypoint is available on Intel Tiger Lake (Gen12) and later using the iHD driver. The `AV1VaapiVideoDecoderDelegate` handles tile group parameter submission. `isConfigSupported({ codec: 'av01.0.08M.08', hardwareAcceleration: 'prefer-hardware' })` will return `supported: true` on qualifying Intel hardware with iHD installed.

### 7.2 AMD AV1 Encode: Known Broken State

AMD Mesa's stateless VA-API AV1 encoder has a bug in its DPB (decoded picture buffer) slot validation: it validates reference frame slots even for keyframes, which causes the Chrome `AV1VaapiVideoEncoderDelegate` to fail on AMD. A fix was proposed in Chromium's `media/gpu/vaapi/av1_vaapi_video_encoder_delegate.cc` (the fix would initialize all `ref_frame_idx` slots to the reconstruct surface on keyframes when the AMD Mesa driver is detected), but the CL was abandoned. As of mid-2026, AV1 hardware encode on AMD Linux in Chrome is unreliable. Intel iHD AV1 encode (on DG2/Arc and Meteor Lake+) is functional and is the recommended path for hardware AV1 encode. [Note: AMD AV1 encode CL status should be re-verified against current Chromium Gerrit before deploying in production.]

### 7.3 Chrome 143 Default-On Scope

Chrome 143's default-on of `AcceleratedVideoDecodeLinuxGL` (enabling `VaapiVideoDecoder` by default) applies specifically to **Wayland sessions**. X11 sessions still require manually enabling the flag via `--enable-features=AcceleratedVideoDecodeLinuxGL`. The additional `AcceleratedVideoDecodeLinuxZeroCopyGL` flag enables the EGL DMA-BUF import path (the true zero-copy path) and is only meaningful on Wayland/EGL — on X11/GLX a CPU copy still occurs because GLX has no DMA-BUF import extension.

### 7.4 Firefox WebCodecs Hardware Acceleration

Firefox 130 enabled WebCodecs on all desktop platforms. However, Firefox's VA-API hardware acceleration on Linux is configured separately and depends on the distribution's `libva` and driver installation. Unlike Chrome, Firefox does not have the iHD-specific allowlisting that Chrome uses, so VA-API support in Firefox on AMD and NVIDIA hardware may be more permissive but also less tested. `powerEfficient` in Firefox's `MediaCapabilities.decodingInfo()` response reflects actual measurement data rather than Chrome's heuristic, but per-codec hardware support coverage is broader in Chrome's Chromium-based implementation.

---

## 8. MediaCapabilities API: Querying Hardware Support

### 8.1 `decodingInfo()` and `encodingInfo()`

`navigator.mediaCapabilities` provides a higher-level query interface complementing `VideoDecoder.isConfigSupported()`:

```javascript
// Query hardware decode capability (type: 'webcodecs' for WebCodecs-specific query)
const result = await navigator.mediaCapabilities.decodingInfo({
  type: 'webcodecs',   // 'file' | 'media-source' | 'webrtc' | 'webcodecs'
  video: {
    contentType: 'video/mp4; codecs="avc1.4d0034"',
    width: 1920,
    height: 1080,
    bitrate: 4_000_000,
    framerate: 30,
  },
});
console.log(result.supported);       // boolean
console.log(result.smooth);          // boolean: heuristic — true if hardware decode likely
console.log(result.powerEfficient);  // boolean: proxy for hardware decode

// Query encode capability
const encodeResult = await navigator.mediaCapabilities.encodingInfo({
  type: 'webrtc',
  video: {
    contentType: 'video/VP9',
    width: 1280,
    height: 720,
    bitrate: 2_000_000,
    framerate: 30,
  },
});
```

The `smooth` and `powerEfficient` flags come with important caveats on Chrome:

- `smooth: true` is returned heuristically: Chrome reports `smooth=true` for any hardware-decoded configuration until per-device playback statistics are accumulated (typically after several decode sessions). It should not be relied on as a definitive hardware-decode confirmation.
- `powerEfficient: true` is a stronger proxy for hardware acceleration: it is only `true` when Chrome's VA-API capability probe indicates hardware decode is available for the configuration.

### 8.2 How Chrome Probes VA-API Capability

At GPU process startup, `VaapiWrapper::GetSupportedDecodeProfiles()` calls `vaQueryConfigEntrypoints()` for each known `VAProfile` and then `vaQuerySurfaceAttributes()` to determine maximum resolutions. The result is cached in the `GpuInfo` structure and serialised back to the browser process, where it is stored in `GpuFeatureInfo` and made accessible to `MediaCapabilitiesProber`. [Source](https://source.chromium.org/chromium/chromium/src/+/main:media/gpu/vaapi/vaapi_wrapper.cc)

The probe is comprehensive: it checks that `VAEntrypointVLD` (decode) is available for H.264, VP9, HEVC, and AV1 profiles, verifies the surface attribute `VA_SURFACE_ATTRIB_MAX_WIDTH/HEIGHT` to determine resolution limits, and stores the result keyed by profile+entrypoint combination. This data drives both `MediaCapabilities.decodingInfo()` responses and the `VideoDecoder.isConfigSupported()` static method.

### 8.3 Practical Capability Detection Pattern

For production applications targeting Linux hardware acceleration:

```javascript
async function detectHardwareDecodeSupport() {
  const codecs = [
    { codec: 'av01.0.08M.08', label: 'AV1' },
    { codec: 'vp09.00.40.08.00', label: 'VP9' },
    { codec: 'avc1.4d0034', label: 'H.264' },
  ];

  for (const { codec, label } of codecs) {
    const support = await VideoDecoder.isConfigSupported({
      codec,
      codedWidth: 1920,
      codedHeight: 1080,
      hardwareAcceleration: 'prefer-hardware',
    });

    const caps = await navigator.mediaCapabilities.decodingInfo({
      type: 'webcodecs',
      video: {
        contentType: `video/webm; codecs="${codec}"`,
        width: 1920, height: 1080, bitrate: 4_000_000, framerate: 30,
      },
    });

    console.log(`${label}: supported=${support.supported}, ` +
                `powerEfficient=${caps.powerEfficient}`);
    // powerEfficient=true is the best available proxy for hardware decode
    if (caps.powerEfficient) return codec; // prefer most efficient codec
  }
  return codecs[codecs.length - 1].codec; // fallback to H.264
}
```

---

## 9. WebRTC Integration: Encoded Transform and Custom Pipelines

### 9.1 WebRTC Encoded Transform API

The **Encoded Transform API** (Baseline 2025, October 2025) allows JavaScript to intercept encoded video and audio in a WebRTC pipeline via `RTCRtpScriptTransform`:

```javascript
// Sender side: transform encoded frames before transmission
const sender = peerConnection.addTrack(track);
sender.transform = new RTCRtpScriptTransform(worker, { operation: 'encrypt' });

// In the worker:
// self.onrtctransform = (event) => {
//   const transformer = event.transformer;
//   transformer.readable.pipeTo(new WritableStream({
//     write(rtcFrame) {
//       // rtcFrame: RTCEncodedVideoFrame
//       // Encrypt frame.data (ArrayBuffer), then enqueue
//     }
//   }));
// };

// Key frame generation (sender side)
sender.transform.port.postMessage('generate-key-frame');
// In worker: await transformer.generateKeyFrame(rid);

// Key frame request (receiver side)
// await transformer.sendKeyFrameRequest();
```

### 9.2 `RTCEncodedVideoFrame` vs `EncodedVideoChunk`: No Direct Interop

`RTCEncodedVideoFrame` is a distinct type from WebCodecs `EncodedVideoChunk`. There is **no direct object-level conversion** between them. To use a WebCodecs `VideoDecoder` or `VideoEncoder` in a WebRTC pipeline, the `data` `ArrayBuffer` must be copied:

```javascript
// Bridge: RTCEncodedVideoFrame → WebCodecs EncodedVideoChunk
function bridgeRtcToWebCodecs(rtcFrame, webCodecsDecoder) {
  // rtcFrame.type: 'key' or 'delta'
  // rtcFrame.timestamp: RTP timestamp (different units from WebCodecs microseconds!)
  const chunk = new EncodedVideoChunk({
    type: rtcFrame.type === 'key' ? 'key' : 'delta',
    // RTP timestamp is in 90kHz clock ticks for video; convert to microseconds:
    timestamp: Math.round(rtcFrame.timestamp / 90 * 1000),
    data: rtcFrame.data,  // ArrayBuffer — this is a copy
  });
  webCodecsDecoder.decode(chunk);
}
```

The timestamp unit difference is a common bug: WebRTC timestamps are in the codec's RTP clock rate (90kHz for video = 90000 ticks/second), while WebCodecs timestamps are in microseconds (1,000,000 units/second). The conversion factor is `rtcTimestamp / 90 * 1000` for 90kHz clocks.

### 9.3 Custom Codec Pipeline with WebCodecs + WebRTC

A practical pattern for inserting a custom codec (e.g., a WASM-based AV1 encoder with perceptual tuning) into a WebRTC pipeline:

```javascript
// Using Encoded Transform to replace WebRTC's built-in encode with WebCodecs H.264

// 1. Disable WebRTC's built-in codec negotiation and send as application/octet-stream
// 2. Use RTCRtpScriptTransform to pass raw camera frames through a WebCodecs encoder
// 3. Send encoded chunks via WebRTC data channel or a custom RTP implementation

// Worker script:
const encoder = new VideoEncoder({ output: sendAsRtpPayload, error: console.error });
encoder.configure({ codec: 'avc1.4d0034', width: 1280, height: 720, bitrate: 2_000_000,
                    latencyMode: 'realtime', hardwareAcceleration: 'prefer-hardware' });

self.onrtctransform = async (event) => {
  // Read unencoded VideoFrame from the sender transform's readable
  for await (const frame of event.transformer.readable) {
    if (frame instanceof VideoFrame) {
      encoder.encode(frame, { keyFrame: shouldForceKey() });
      frame.close();
    }
  }
};
```

Note: this pattern bypasses WebRTC's built-in congestion control and NACK retransmission logic, so applications using it must implement their own reliability layer. It is primarily useful for non-standard codecs or post-processing that the browser's built-in WebRTC codec stack cannot provide.

---

## 10. VideoFrame Canvas Capture and Recording

### 10.1 Capture from `<video>` or `<canvas>`

```javascript
// Capture from HTMLVideoElement — yields frames at the video's natural frame rate
const stream = videoElement.captureStream(30); // optional fps hint
const [track] = stream.getVideoTracks();
const processor = new MediaStreamTrackProcessor({ track });
const reader = processor.readable.getReader();

while (true) {
  const { done, value: frame } = await reader.read();
  if (done) break;
  // frame: VideoFrame from the decoded video source
  encoder.encode(frame, { keyFrame: /* ... */ false });
  frame.close();
}
```

### 10.2 `canvas.captureStream()` for Encoding Composited Frames

`HTMLCanvasElement.captureStream()` returns a `MediaStream` carrying composited frames from a canvas — either a WebGL scene, a 2D canvas, or a WebGPU canvas (via `canvas.getContext('webgpu')`). Combined with `MediaStreamTrackProcessor`, this creates a pipeline from GPU-rendered frames to a WebCodecs encoder:

```javascript
const gpuCanvas = document.createElement('canvas');
const gpuCtx = gpuCanvas.getContext('webgpu');
// ... WebGPU rendering into gpuCtx ...

const captureStream = gpuCanvas.captureStream(60);
const [captureTrack] = captureStream.getVideoTracks();
const captureProcessor = new MediaStreamTrackProcessor({ track: captureTrack });

const encoder = new VideoEncoder({
  output(chunk, metadata) { /* upload chunk */ },
  error(e) { console.error(e); },
});
encoder.configure({ codec: 'av01.0.08M.08', width: 1920, height: 1080,
                    bitrate: 8_000_000, framerate: 60,
                    hardwareAcceleration: 'prefer-hardware' });

const writer = new WritableStream({
  write(frame) {
    encoder.encode(frame, { keyFrame: false });
    frame.close();
  }
});
captureProcessor.readable.pipeTo(writer);
```

### 10.3 MediaRecorder vs WebCodecs for Recording

`MediaRecorder` is the traditional recording API: it accepts a `MediaStream` and produces container-wrapped chunks (WebM, MP4) with codec selection controlled by the browser. WebCodecs provides more control:

| Aspect | MediaRecorder | WebCodecs |
|--------|--------------|-----------|
| Output | Container-wrapped (WebM, MP4) | Raw encoded chunks (no container) |
| Codec control | Browser-chosen (MIME type hint) | Full codec string specification |
| Frame-level access | None | Full (each `EncodedVideoChunk` is accessible) |
| Hardware acceleration | Transparent (browser-managed) | Explicit (`hardwareAcceleration` option) |
| Container muxing | Built-in | Application must use WebAssembly muxer |
| Latency | Higher (container overhead) | Lower |
| API complexity | Low | Higher |

For applications requiring specific codecs (e.g., H.264 Baseline for maximum compatibility), per-frame processing (e.g., encrypting each NAL unit), or integration with custom container formats, WebCodecs is the only viable path. For simple recording to a file, `MediaRecorder` remains simpler.

---

## 11. Debugging WebCodecs and Hardware Acceleration

### 11.1 Chrome Internal Pages

**`chrome://gpu`** is the first stop: the "Video Decode" row in the "Graphics Feature Status" section shows whether `VaapiVideoDecoder` is enabled. The "Video Acceleration Information" section lists all supported codec profiles detected by the VA-API probe at startup.

**`chrome://media-internals`** provides real-time monitoring of all active media pipelines. For a WebCodecs decoder, the `video_decoder` property in the media log should read `VaapiVideoDecoder` for hardware decode or `FFmpegVideoDecoder`/`VpxVideoDecoder` for software fallback. The `has_decryption_config` and `key_system` fields confirm CDM involvement for encrypted streams.

### 11.2 Chrome Command-Line Flags

```bash
# Enable hardware decode on Wayland (default in Chrome 143+, but explicit if needed)
google-chrome \
  --enable-features=AcceleratedVideoDecodeLinuxGL,AcceleratedVideoDecodeLinuxZeroCopyGL \
  --ignore-gpu-blocklist \
  --enable-logging=stderr \
  --v=1

# Enable hardware encode (H.264, AV1 on Intel iHD)
google-chrome \
  --enable-features=AcceleratedVideoDecodeLinuxGL,AcceleratedVideoEncoder \
  --ignore-gpu-blocklist

# For AMD (requires driver allowlist bypass)
google-chrome \
  --enable-features=AcceleratedVideoDecodeLinuxGL,VaapiIgnoreDriverChecks \
  --ignore-gpu-blocklist

# For NVIDIA with nvidia-vaapi-driver
google-chrome \
  --enable-features=AcceleratedVideoDecodeLinuxGL,VaapiIgnoreDriverChecks,VaapiOnNvidiaGPUs \
  --ignore-gpu-blocklist
```

### 11.3 VA-API Debugging

```bash
# 1. Enumerate supported VA-API profiles and entrypoints
vainfo --display drm --device /dev/dri/renderD128

# Expected output for Intel iHD Tiger Lake (AV1 decode):
# VAProfileAV1Profile0            :	VAEntrypointVLD
# VAProfileH264Main               :	VAEntrypointVLD
# VAProfileH264Main               :	VAEntrypointEncSliceLP  (low-power encode)
# VAProfileHEVCMain               :	VAEntrypointVLD
# VAProfileVP9Profile0            :	VAEntrypointVLD

# 2. Enable libva message tracing (produces per-thread log files)
LIBVA_TRACE=/tmp/vaapi_trace \
LIBVA_MESSAGING_LEVEL=2 \
google-chrome --enable-features=AcceleratedVideoDecodeLinuxGL &

# Log files appear as /tmp/vaapi_trace.<pid>.<tid>
# Look for vaBeginPicture/vaRenderPicture/vaEndPicture sequences

# 3. Verify DRM device access
ls -la /dev/dri/renderD*
# GPU process needs read-write access to /dev/dri/renderD128

# 4. Check driver loaded
strings /usr/lib/x86_64-linux-gnu/dri/iHD_drv_video.so | grep "version"
LIBVA_DRIVER_NAME=iHD vainfo  # force specific driver
```

### 11.4 WebCodecs Error Patterns

Common errors and their Linux-specific causes:

| Error | Cause | Fix |
|-------|-------|-----|
| `NotSupportedError: Unsupported configuration` from `isConfigSupported` | No VA-API support for this codec/profile | Install `intel-media-driver` or use `VaapiIgnoreDriverChecks` |
| `EncodingError` in output error callback for AV1 encode on AMD | Mesa AV1 encoder DPB bug | Use software AV1 encode or Intel iHD hardware |
| `VideoDecoder` output callback never fires | `frame.close()` not called on previous frames; decoder stalled | Always call `frame.close()` immediately after use |
| Hardware decode shows in `isConfigSupported` but `chrome://gpu` shows disabled | GPU blocklist overriding | Use `--ignore-gpu-blocklist` for testing |
| `GPUExternalTexture` produces a black frame | Texture used after `videoFrame.close()` | Close frame after GPU work completes |
| `delta` chunk rejected after `reset()` | Missing key frame sync after reset | Submit a key frame (`type: 'key'`) after every `configure()` or `reset()` |

### 11.5 WebCodecs in DevTools

Chrome DevTools (as of Chrome 120+) does not have a dedicated WebCodecs panel, but the **Performance** timeline records `VideoDecoder.decode()` and `VideoEncoder.encode()` calls as tasks on the worker thread. Long `decode()` tasks (>16ms) indicate software fallback — hardware VA-API decode should complete in 2–8ms for 1080p H.264 on modern Intel hardware. The **Media** panel in DevTools (`Ctrl+Shift+I → More tools → Media`) exposes `VideoDecoder` instances and their frame delivery statistics.

---

## Roadmap

### Near-term (6–12 months)

- **Media Source Extensions (MSE) + WebCodecs integration reaching Candidate Recommendation**: The W3C Media Working Group charter targets CR status for the MSE-WebCodecs bridge in Q4 2026, enabling applications to feed WebCodecs-decoded `VideoFrame` objects directly into an `HTMLMediaElement` buffering pipeline for adaptive playback without a separate demuxer. [Source](https://w3c.github.io/charter-media-wg/)
- **VA-API on Wayland stabilisation across distros**: Chromium's `VaapiWrapper` unified the Linux Ozone/Wayland and Ozone/X11 codepaths to use `libva-drm` exclusively; the remaining work is removing the GPU blocklist entries that still disable VA-API on unsigned distro builds, making hardware-accelerated WebCodecs decode the default rather than an opt-in flag. [Source](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/docs/gpu/vaapi.md)
- **AV1 10-bit hardware encoding broadening**: As of mid-2026 only ~8% of sessions support 10-bit AV1 encoding despite 91% hardware decoder coverage for 10-bit AV1; Intel iHD and AMD RADV AV1 encode paths are being hardened to surface 10-bit `VideoEncoder` configurations through `isConfigSupported()`. [Source](https://webcodecsfundamentals.org/datasets/codec-analysis-2026/)
- **V4L2 stateless WebCodecs on mainline Linux desktop**: Chromium issue 372630272 tracks enabling V4L2 stateless decode (currently ChromeOS-only) for desktop Linux; blockers include `VIDIOC_EXPBUF` support in upstream drivers (rkvdec2, hantro) and GBM NV12 buffer allocation with Panfrost. [Source](https://issues.chromium.org/issues/372630272)

### Medium-term (1–3 years)

- **HDR `VideoFrame` metadata and tone-mapping**: W3C webcodecs issue #384 tracks full HDR colour metadata on `VideoFrame` — `colorSpace.matrix`, `primaries`, and `transfer` with HDR10/HLG values — and the corresponding `copyTo()` behaviour for 10-bit and 12-bit pixel formats. Note: no firm timeline for CR-level normative text as of mid-2026. [Source](https://github.com/w3c/webcodecs/issues/384)
- **Encrypted media chunk support in WebCodecs**: The W3C Media Working Group plans to define a standard representation for encrypted `EncodedVideoChunk` / `EncodedAudioChunk` objects, allowing Encrypted Media Extensions (EME) key sessions to decrypt into WebCodecs pipelines rather than the opaque MSE pipeline. [Source](https://w3c.github.io/charter-media-wg/)
- **WebCodecs in Node.js and server-side runtimes**: The `webcodecs-node` project exposes the WebCodecs API to Node.js via native add-ons backed by FFmpeg and platform VA-API; if adopted by the W3C, a non-`SecureContext` server profile could enable server-side transcoding with the same JS API surface. Note: needs verification for standardisation timeline. [Source](https://github.com/Brooooooklyn/webcodecs-node)
- **`ImageDecoder` extension for arbitrary byte-array container formats**: The WICG ImageDecoder proposal aims to make container-coupled image formats (JPEG-XL, AVIF, animated WebP) accessible via a `decode()` → `ImageBitmap` per-frame API, eliminating the need for JS-side container parsers ahead of `VideoDecoder`. [Source](https://discourse.wicg.io/t/proposal-imagedecoder-api-extension-for-webcodecs/4418/)
- **AV2 (Alliance for Open Media next-gen codec) WebCodecs codec registry entry**: AV2 standardisation is in progress at AOM; a WebCodecs codec string registry entry and `VideoDecoder` / `VideoEncoder` configuration surface will follow hardware availability. Note: needs verification on timeline relative to AOM AV2 ratification. [Source](https://en.wikipedia.org/wiki/AV2)

### Long-term

- **Zero-copy `VideoFrame` → WebGPU without `importExternalTexture` lifetime restrictions**: The current `GPUExternalTexture` model requires frames to be closed within the same task; a proposed shared-ownership model based on `VK_EXT_external_memory_dma_buf` persistent import would allow `VideoFrame` GPU memory to live in a `GPUTexture` with standard reference-counted lifetime, removing the need for per-frame re-import round-trips.  Note: needs verification — design discussion only as of mid-2026.
- **Cross-process `VideoFrame` transfer via `SharedArrayBuffer`-backed zero-copy**: Long-term, the `VideoFrame.transfer()` model may extend to Shared Workers and cross-origin isolated iframes using `SharedArrayBuffer` for the CPU-side pixel data, enabling multi-tab video processing pipelines without serialisation overhead. Note: needs verification.
- **Unified stateless codec kernel UAPI and Chromium V4L2 path on ARM Linux**: Upstream kernel work on a unified stateless codec control UAPI (`V4L2_CID_STATELESS_*`) is progressing; once driver coverage is sufficient, Chromium's V4L2 stateless path could become the primary decode mechanism on ARM Linux SBCs and laptops, complementing VA-API on x86. [Source](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/ext-ctrls-codec-stateless.rst)

---

## 12. Integrations

### Chapter 26: VA-API Architecture

The WebCodecs hardware acceleration path on Linux desktop is built entirely on VA-API. [Chapter 26](../part-07a-gpu-apis/) covers the full VA-API architecture: `vaInitialize()`, `vaCreateConfig()`, `vaCreateContext()`, surface management, profile enumeration, and the libva driver model. This chapter focuses on the Chromium-specific wrapping in `VaapiWrapper` and `VaapiVideoDecoder` — the layer between Chrome's media stack and the libva API described in Chapter 26. For `vaExportSurfaceHandle()` internals, the VA-API 1.1.0 specification, and the iHD vs i965 driver distinction at the libva level, see Chapter 26.

### Chapter 33: Chrome GPU Process Architecture

`VaapiVideoDecoder`, `MojoVideoDecoderService`, `GpuMojoMediaClient`, `MediaService`, `FramelessMediaInterfaceProxy`, `SharedImageBackingOzone`, and `DmabufVideoFramePool` all run inside the Chrome GPU process. [Chapter 33](ch33-chromium-gpu-architecture.md) describes the GPU process's overall architecture: the `GpuMain` entry point, `GpuServiceImpl`, the Mojo IPC framework, the `seccomp-BPF` sandbox model, and the `SharedImage` subsystem. This chapter's explanation of how WebCodecs arrives at the GPU process via `VideoDecoderBroker` and `FramelessMediaInterfaceProxy` connects to Chapter 33's description of how the GPU process handles all cross-process GPU resource access.

### Chapter 35: Dawn and WebGPU

`device.importExternalTexture({ source: videoFrame })` is implemented in Dawn, Google's WebGPU implementation. [Chapter 35](ch35-dawn-webgpu.md) covers Dawn's layered architecture, the DawnWire IPC protocol, the Vulkan backend, and the Tint WGSL compiler. The `GPUExternalTexture` mechanism, `VK_EXT_external_memory_dma_buf` import, the `texture_external` WGSL type, and the `textureSampleBaseClampToEdge()` requirement are all Dawn/WebGPU internals described in Chapter 35. This chapter documents only the WebCodecs `VideoFrame` integration surface — the `importExternalTexture()` API boundary.

### Chapter 38: FFmpeg and Software Codec Implementations

When `VaapiVideoDecoder` is unavailable — missing iHD driver, unsupported hardware, non-Intel/non-allowlisted AMD/NVIDIA — Chrome's `VideoDecoderBroker` falls back through `FFmpegVideoDecoder` (H.264, VP9 via libvpx, and others) and `VpxVideoDecoder`. Chrome's build system with `ffmpeg_branding = "Chrome"` unlocks proprietary codec support (H.264, AAC) in FFmpeg. [Chapter 38](../part-09-tooling-contributing/) covers FFmpeg's architecture and Chrome's integration. This chapter notes the fallback; Chapter 38 documents the FFmpeg internals.

---

## 13. References

- W3C WebCodecs Specification: [https://www.w3.org/TR/webcodecs/](https://www.w3.org/TR/webcodecs/)
- W3C WebCodecs Codec Registry: [https://www.w3.org/TR/webcodecs-codec-registry/](https://www.w3.org/TR/webcodecs-codec-registry/)
- Chromium `third_party/blink/renderer/modules/webcodecs/`: [https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/modules/webcodecs/](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/modules/webcodecs/)
- Chromium `VideoDecoderBroker`: [https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/modules/webcodecs/video_decoder_broker.h](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/modules/webcodecs/video_decoder_broker.h)
- Chromium `VaapiVideoDecoder`: [https://source.chromium.org/chromium/chromium/src/+/main:media/gpu/vaapi/vaapi_video_decoder.h](https://source.chromium.org/chromium/chromium/src/+/main:media/gpu/vaapi/vaapi_video_decoder.h)
- Chromium `VaapiWrapper`: [https://source.chromium.org/chromium/chromium/src/+/main:media/gpu/vaapi/vaapi_wrapper.h](https://source.chromium.org/chromium/chromium/src/+/main:media/gpu/vaapi/vaapi_wrapper.h)
- Chromium `SharedImageBackingOzone`: [https://source.chromium.org/chromium/chromium/src/+/main:gpu/command_buffer/service/shared_image/ozone_image_backing.h](https://source.chromium.org/chromium/chromium/src/+/main:gpu/command_buffer/service/shared_image/ozone_image_backing.h)
- Chromium Mojo Media README: [https://source.chromium.org/chromium/chromium/src/+/main:media/mojo/README.md](https://source.chromium.org/chromium/chromium/src/+/main:media/mojo/README.md)
- libva `vaExportSurfaceHandle` — commit `51e98b1` (VA-API 1.1.0): [https://github.com/intel/libva/commit/51e98b1](https://github.com/intel/libva/commit/51e98b1)
- libva API reference (`vaExportSurfaceHandle`, `VADRMPRIMESurfaceDescriptor`): [https://intel.github.io/libva/group__api__core.html](https://intel.github.io/libva/group__api__core.html)
- Mesa `src/gallium/frontends/va/surface.c` (vaExportSurfaceHandle implementation): [https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/gallium/frontends/va/surface.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/gallium/frontends/va/surface.c)
- Linux kernel V4L2 stateless codec controls: [https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/ext-ctrls-codec-stateless.rst](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/ext-ctrls-codec-stateless.rst)
- MDN WebCodecs API: [https://developer.mozilla.org/en-US/docs/Web/API/WebCodecs_API](https://developer.mozilla.org/en-US/docs/Web/API/WebCodecs_API)
- WebCodecs Fundamentals (codec string examples): [https://www.webcodecsfundamentals.org/](https://www.webcodecsfundamentals.org/)
- WebRTC Encoded Transform API (Baseline 2025): [https://www.w3.org/TR/webrtc-encoded-transform/](https://www.w3.org/TR/webrtc-encoded-transform/)
- Dawn WebGPU `importExternalTexture`: [https://dawn.googlesource.com/dawn](https://dawn.googlesource.com/dawn)
- Chrome Platform Status — WebCodecs: [https://chromestatus.com/feature/5669293909868544](https://chromestatus.com/feature/5669293909868544)
- W3C GPU for the Web Working Group — `GPUExternalTexture`: [https://www.w3.org/TR/webgpu/#gpu-external-texture](https://www.w3.org/TR/webgpu/#gpu-external-texture)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
